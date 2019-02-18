---
layout: post
title: Canary Deployments with Solo.io Gloo
date: 2019-02-17 09:29
---

<figure class="aligncenter">
    <img src="https://images.unsplash.com/photo-1544215901-f855c2b93434?ixlib=rb-1.2.1&ixid=eyJhcHBfaWQiOjEyMDd9&auto=format&fit=crop&w=2126&q=80" />
    <figcaption>Photo by <a href="https://unsplash.com/@zabit" target="_blank">Zab Consulting</a>.</figcaption>
</figure>

This post walk thru how to use the open source [Solo.io](https://solo.io) [Gloo](https://github.com/solo-io/gloo) project
to do a Canary release of a new version of a function. Gloo is a microservices gateway that gives users a number of
benefits including sophisticated function level routing, and deep service discovery with introspection of OpenAPI (Swagger)
definations. This post will show a simple example of Gloo discovering 2 different deployments of a service, and setting
up some routes. Initial route will use the presence of a request header `x-canary:true` to influence runtime routing to
either version 1 or version 2 of our service, and then assuming we're happy with our new version, we update the route
so all requests now go to version 2 of our service. All without changing or even redploying our 2 services. But first,
let's set some context...

## Background

<figure class='quote'>
  <blockquote>
    <p>Canary release is a technique to reduce the risk of introducing a new software version in production by slowly
    rolling out the change to a small subset of users before rolling it out to the entire infrastructure and making it
    available to everybody.</p>
  </blockquote>
  <figcaption class='quote-source'>
    <span class='quote-author'>Danilo Sato</span>
    <cite class='quote-title'><a href='https://martinfowler.com/bliki/CanaryRelease.html'>Canary Release</a></cite>
  </figcaption>
</figure>

The idea of a Canary release is that no matter how much testing you do on a new implementation, until you deploy it into
your production environment you can't be postive everything will work as expected. So having a way to release a new
version into production concurrently with the existing version(s) with some way to route traffic can be helpful. Ideally,
we'd like to route most traffic to existing, known to work version, and have a way for some requests go to the new version.
Once you're feeling comfordable that your new version is working like you expect, then and only then, do you start
routing more/all requests to the new version, and then eventually decommision the original service.

Being able to change request routes **without** needing to change or redploy your code, I think, is very helpful in
building confidence that your code is ready for production. That is, if you need to change your code or use a code based
feature flag, then your exercising different code paths and/or changing configuration settings. I feel its better if you
can deploy your service, code and configurations, all ready for production, and use an external mechansim to manage
request routing.

Gloo uses [Evnoy](https://www.envoyproxy.io/), which is a super high performance service proxy, to do the request routing.
In this example, we'll use a request header to influence the routing, though we could also use other variables like the
IP range of the requestor to drive routing decisions. That is, if requests are coming from specific test machines we can
route them to our new version. Lots more information on how Gloo and Envoy works can be found on the [Solo.io](https://solo.io)
website. On to the example...

I'm assuming you're already running a Kubernetes installation. I'm assuming you're using [minikube](https://kubernetes.io/docs/setup/minikube/)
for this post though any recent Kubernetes installation should work as long as you have [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/)
setup and configured correctly for your Kubernetes installation.

## Setup

### Setup Example Services

1. Let's install both versions of our services. These services are based on the [g-swagger petstore example](https://github.com/go-swagger/go-swagger/tree/master/examples/2.0/petstore).
The only difference between these 2 versions is that version 2 starts with 3 pets versus version 1 with only 2 pets. This
works for the purposes of this post. I'm including YAML contents within this post for ease.

   ```yaml
   cat <<EOF | kubectl apply -f -
   ---
   # petstore-v1
   apiVersion: v1
   kind: Service
   metadata:
     name: petstore-v1
     namespace: default
     labels:
       sevice: petstore-v1
   spec:
     type: ClusterIP
     ports:
     - name: http
       port: 8080
       targetPort: 8080
       protocol: TCP
     selector:
       app: petstore-v1
   ---
   apiVersion: extensions/v1beta1
   kind: Deployment
   metadata:
     name: petstore-v1
     namespace: default
     labels:
       app: petstore-v1
   spec:
     replicas: 1
     template:
       metadata:
         labels:
           app: petstore-v1
       spec:
         containers:
         - image: scottcranton/petstore:v1
           name: petstore-v1
           ports:
           - containerPort: 8080
   ---
   # petstore-v2
   apiVersion: v1
   kind: Service
   metadata:
     name: petstore-v2
     namespace: default
     labels:
       sevice: petstore-v2
   spec:
     type: ClusterIP
     ports:
     - name: http
       port: 8080
       targetPort: 8080
       protocol: TCP
     selector:
       app: petstore-v2
   ---
   apiVersion: extensions/v1beta1
   kind: Deployment
   metadata:
     name: petstore-v2
     namespace: default
     labels:
       app: petstore-v2
   spec:
     replicas: 1
     template:
       metadata:
         labels:
           app: petstore-v2
       spec:
         containers:
         - image: scottcranton/petstore:v2
           name: petstore-v2
           ports:
           - containerPort: 8080
   EOF
   ```

   Both versions were installed into the `default` namespace, so we can look there to see if there installed correctly.

   ```shell
   kubectl get all --namespace default
   ```

   ```shell
   NAME                               READY   STATUS    RESTARTS   AGE
   pod/petstore-v1-7876bd5bf8-skn5f   1/1     Running   0          6m45s
   pod/petstore-v2-684bdfbcf5-z9kh8   1/1     Running   0          5m43s

   NAME                  TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
   service/kubernetes    ClusterIP   10.96.0.1        <none>        443/TCP    2d20h
   service/petstore-v1   ClusterIP   10.97.195.104    <none>        8080/TCP   6m45s
   service/petstore-v2   ClusterIP   10.102.155.172   <none>        8080/TCP   5m25s

   NAME                          READY   UP-TO-DATE   AVAILABLE   AGE
   deployment.apps/petstore-v1   1/1     1            1           6m45s
   deployment.apps/petstore-v2   1/1     1            1           5m43s

   NAME                                     DESIRED   CURRENT   READY   AGE
   replicaset.apps/petstore-v1-7876bd5bf8   1         1         1       6m45s
   replicaset.apps/petstore-v2-684bdfbcf5   1         1         1       5m43s
   ```

1. Let's test our services. They're both setup to expose on port `8080`, and both will return the list of all pets for
`GET` requests on the query path `/api/pets`. The easiest way to test is to `port-forward` the pods so we can access
locally.

   ```shell
   kubectl get pods --namespace default
   ```

   ```shell
   NAME                           READY   STATUS    RESTARTS   AGE
   petstore-v1-7876bd5bf8-skn5f   1/1     Running   0          13m
   petstore-v2-684bdfbcf5-z9kh8   1/1     Running   0          12m
   ```

   We'll need the pod names for the port forwarding. Let's test the first version. Make sure the pod names match the
   ones from your system. This will foward port 8080 from the service running in your Kubernetes installation to your
   local machine, i.e. `localhost:8080`.

   ```shell
   kubectl port-portforward petstore-v1-7876bd5bf8-skn5f 8080:8080
   ```

   In a seperate terminal, run the following.

   ```shell
   curl localhost:8080/api/pets
   ```

   And hopefully you will see the following. Version 1 of our petstore should only have 2 pets: a Dog, and a Cat.

   ```json
   [{"id":1,"name":"Dog","status":"available"},{"id":2,"name":"Cat","status":"pending"}]
   ```

   You can also get the Swagger spec as well.

   ```shell
   curl localhost:8080/swagger.json
   ```

   You can now kill the port forward command, and repeat the above instructions for `petstore-v2`. Make sure you use the
   pod name for `petstore-v2` for your installation; they change each time you deploy. Version 2 of our service has a
   big upgrade of **3** pets: Dog, Cat, and a Parrot. You'll also notice that the status is `v2` for version 2 as well.

   ```json
   [{"id":1,"name":"Dog","status":"v2"},{"id":2,"name":"Cat","status":"v2"},{"id":3,"name":"Parrot","status":"v2"}]
   ```

   You can kill all the port forwards. Now we'll setup Gloo...

### Setup Gloo

1. Let's setup the Gloo command line utility. Full instructions are at the [Gloo doc site](https://gloo.solo.io/installation/).
Here are the quick setup instructions.

   Setup the `glooctl` command line tool. This makes installation, upgrade, and operations of Gloo easier. I encourage
   you all to look at the [install script](https://run.solo.io/gloo/install) *before* running some random shell script
   off the Internet.

   ```shell
   curl -sL https://run.solo.io/gloo/install | sh
   ```

   This sets up `glooctl` in your `$HOME/.gloo/bin` directory, and the installation will have you add that path to your
   `PATH`.

1. Now let's install Gloo into your Kubernetes installation.

   ```shell
   glooctl install gateway
   ```

   Pretty easy, eh? Let's verify that its installed and running correctly. Gloo by default creates and installs into the
   `gloo-system` namespece, so let's look at everything running there.

   ```shell
   kubectl get all --namespace gloo-system
   ```

   And the output should look something like the following.

   ```shell
   NAME                                READY   STATUS    RESTARTS   AGE
   pod/discovery-7d87b5c69d-t45mr      1/1     Running   0          13s
   pod/gateway-7459bf6bcc-x72tl        1/1     Running   0          12s
   pod/gateway-proxy-6c97c6c74-gdnf8   1/1     Running   0          12s
   pod/gloo-77f695fbb9-68lcw           1/1     Running   0          13s

   NAME                    TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
   service/gateway-proxy   LoadBalancer   10.97.60.144     <pending>     80:32685/TCP,443:32668/TCP   13s
   service/gloo            ClusterIP      10.108.141.196   <none>        9977/TCP                     13s

   NAME                            READY   UP-TO-DATE   AVAILABLE   AGE
   deployment.apps/discovery       1/1     1            1           13s
   deployment.apps/gateway         1/1     1            1           13s
   deployment.apps/gateway-proxy   1/1     1            1           12s
   deployment.apps/gloo            1/1     1            1           13s

   NAME                                      DESIRED   CURRENT   READY   AGE
   replicaset.apps/discovery-7d87b5c69d      1         1         1       13s
   replicaset.apps/gateway-7459bf6bcc        1         1         1       12s
   replicaset.apps/gateway-proxy-6c97c6c74   1         1         1       12s
   replicaset.apps/gloo-77f695fbb9           1         1         1       13s
   ```

## Upstreams

Before we get into routing, let's talk a little about the concept of `Upstreams`. Upstreams are the services that Gloo has discovered automatically. Let's look at the upstreams that Gloo has discovered in our Kubernetes cluster.

```shell
glooctl get upstreams
```

You may see some different entries than what follows. It depends on your Kubernetes cluster, and what is running currently.

```shell
+-------------------------------+------------+----------+------------------------------+
|           UPSTREAM            |    TYPE    |  STATUS  |           DETAILS            |
+-------------------------------+------------+----------+------------------------------+
| default-kubernetes-443        | Kubernetes | Accepted | svc name:      kubernetes    |
|                               |            |          | svc namespace: default       |
|                               |            |          | port:          443           |
|                               |            |          |                              |
| default-petstore-v1-8080      | Kubernetes | Accepted | svc name:      petstore-v1   |
|                               |            |          | svc namespace: default       |
|                               |            |          | port:          8080          |
|                               |            |          | REST service:                |
|                               |            |          | functions:                   |
|                               |            |          | - addPet                     |
|                               |            |          | - deletePet                  |
|                               |            |          | - findPetById                |
|                               |            |          | - findPets                   |
|                               |            |          |                              |
| default-petstore-v2-8080      | Kubernetes | Accepted | svc name:      petstore-v2   |
|                               |            |          | svc namespace: default       |
|                               |            |          | port:          8080          |
|                               |            |          | REST service:                |
|                               |            |          | functions:                   |
|                               |            |          | - addPet                     |
|                               |            |          | - deletePet                  |
|                               |            |          | - findPetById                |
|                               |            |          | - findPets                   |
|                               |            |          |                              |
| gloo-system-gateway-proxy-443 | Kubernetes | Accepted | svc name:      gateway-proxy |
|                               |            |          | svc namespace: gloo-system   |
|                               |            |          | port:          443           |
|                               |            |          |                              |
| gloo-system-gateway-proxy-80  | Kubernetes | Accepted | svc name:      gateway-proxy |
|                               |            |          | svc namespace: gloo-system   |
|                               |            |          | port:          80            |
|                               |            |          |                              |
| gloo-system-gloo-9977         | Kubernetes | Accepted | svc name:      gloo          |
|                               |            |          | svc namespace: gloo-system   |
|                               |            |          | port:          9977          |
|                               |            |          |                              |
| kube-system-kube-dns-53       | Kubernetes | Accepted | svc name:      kube-dns      |
|                               |            |          | svc namespace: kube-system   |
|                               |            |          | port:          53            |
|                               |            |          |                              |
+-------------------------------+------------+----------+------------------------------+
```

Notice that our 2 petstore services - `default-petstore-v1-8080` and `default-petstore-v2-8080` are different in
that their details are also listing 4 REST service functions. This is because Gloo can autodetect OpenAPI / Swagger
definations. This allows Gloo to route to individual functions versus what most traditional API Gateways do in
only letting you route to host:port granular services. Let's see that in action.

Let's look a little closer. The `glooctl` command let's us output the full details in yaml or json.

```shell
glooctl get upstream default-petstore-v1-8080 -o yaml
```

```yaml
---
discoveryMetadata: {}
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","kind":"Service","metadata":{"annotations":{},"labels":{"sevice":"petstore-v1"},"name":"petstore-v1","namespace":"default"},"spec":{"ports":[{"name":"http","port":8080,"protocol":"TCP"}],"selector":{"app":"petstore-v1"}}}
  labels:
    discovered_by: kubernetesplugin
    sevice: petstore-v1
  name: default-petstore-v1-8080
  namespace: gloo-system
  resourceVersion: "129285"
status:
  reportedBy: gloo
  state: Accepted
upstreamSpec:
  kube:
    selector:
      app: petstore-v1
    serviceName: petstore-v1
    serviceNamespace: default
    servicePort: 8080
    serviceSpec:
      rest:
        swaggerInfo:
          url: http://petstore-v1.default.svc.cluster.local:8080/swagger.json
        transformations:
          addPet:
            body:
              text: '{"id": {{ default(id, "") }},"name": "{{ default(name, "")}}","tag":
                "{{ default(tag, "")}}"}'
            headers:
              :method:
                text: POST
              :path:
                text: /api/pets
              content-type:
                text: application/json
          deletePet:
            headers:
              :method:
                text: DELETE
              :path:
                text: /api/pets/{{ default(id, "") }}
              content-type:
                text: application/json
          findPetById:
            body: {}
            headers:
              :method:
                text: GET
              :path:
                text: /api/pets/{{ default(id, "") }}
              content-length:
                text: "0"
              content-type: {}
              transfer-encoding: {}
          findPets:
            body: {}
            headers:
              :method:
                text: GET
              :path:
                text: /api/pets?tags={{default(tags, "")}}&limit={{default(limit,
                  "")}}
              content-length:
                text: "0"
              content-type: {}
              transfer-encoding: {}
```

For example, we see that the `findPets` REST function is looking for requests on `/api/pets`, and `findPetsById` is looking for requests on `/api/pets/{id}` where `{id}` is the id number of the single pet who's details are to be returned.

## Simple Routing

Gloo acts like an [Kubernetes Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/), which means it
can allow requests from external to the Kubernetes cluster to access services running inside the cluster. Gloo uses a
concept called `Virtual Service` to setup routes to Kubernetes hosted services.

This post will show you how to configure Gloo using the commandline tools, and I'll explain a little of what's happening
with each command. I'll also include the YAML at the end of each step if you'd prefer to work in a purely declarative
fashion (versus imperative commands).

1. Setup `Virtual Service`. This gives us a place to setup a set of related routes. This won't do much till we create some routes in the next steps.

   ```shell
   glooctl create virtualservice --name coalmine
   ```

   Here's the YAML that will create the same resource. Note that by default the `glooctl` command creates resources in the `gloo-system` namespace.

   ```yaml
   ---
   displayName: coalmine
   metadata:
     name: coalmine
     namespace: gloo-system
   virtualHost:
     domains:
     - '*'
     name: gloo-system.coalmine
   ```

1. Create a route for all traffic to go to version 1 of our service.

   ```shell
   glooctl add route \
      --name coalmine \
      --path-prefix /petstore \
      --dest-name default-petstore-v1-8080 \
      --prefix-rewrite /api/pets/
   ```

   This sets up a simple ingress route so that all requests going to the Gloo Proxy `/` URL are redirected to 
   the `default-petstore-v1-8080` service `/api/pets`. Let's test it.

   To get the Gloo proxy host and port number (remember that Gloo is acting like a Kubernetes Ingress), we need to
   call `glooctl proxy url`. Then let's call the route path.

   ```shell
   export PROXY_URL=$(glooctl proxy url)
   curl ${PROXY_URL}/petstore
   ```

   And we should see the same results as when we called the port forwarded service.

   ```json
   [{"id":1,"name":"Dog","status":"available"},{"id":2,"name":"Cat","status":"pending"}]
   ```

## Function Routing

Wouldn't it be better if we could just route to the named REST function versus having to know the specifics of
the query path (i.e. `/api/pets`) the service is expecting? Gloo can help us with that. Let's setup a route to
`findPets` REST function.

```shell
glooctl add route \
   --name coalmine \
   --path-prefix /findPets \
   --dest-name default-petstore-v1-8080 \
   --rest-function-name findPets
```

And test it.

```shell
curl ${PROXY_URL}/findPets
```

We should see the same results as the `/petstore`.

If we want to route to a function with parameters, we can do that too.

```shell
glooctl add route \
   --name coalmine \
   --path-prefix /findPetWithId/ \
   --dest-name default-petstore-v1-8080 \
   --rest-function-name findPetById \
   --rest-parameters ':path=/findPetWithId/{id}'
```

Let's look up the details for the pet with id 1

```shell
curl ${PROXU_URL}/findPetWithId/1
```

```json
{"id":1,"name":"Dog","status":"available"}
```

## Canary Routing

Now let's deploy a version 2 of our service, and let's setup a canary route for the `findPets` function. That is,
by default we'll route to version 1 of the function, and if there is a request header `x-canary:true` set, we'll route that request to version 2 of our service.

```shell
glooctl add route \
   --name coalmine \
   --path-prefix /findPets \
   --dest-name default-petstore-v2-8080 \
   --rest-function-name findPets \
   --header x-canary=true
```

Default routing should go to petstore version 1, and return only 2 pets.

```shell
curl ${PROXY_URL}/findPets
```

```json
[{"id":1,"name":"Dog","status":"available"},{"id":2,"name":"Cat","status":"pending"}]
```

If we make a request with the `x-canary:true` set, it should route to petstore version 2, and return 3 pets.

```shell
curl -H "x-canary:true" ${PROXY_URL}/findPets
```

```json
[{"id":1,"name":"Dog","status":"v2"},{"id":2,"name":"Cat","status":"v2"},{"id":3,"name":"Parrot","status":"v2"}]
```

Just to verify, let's set the header to a different value, e.g. `x-canary:false` to see that it routes to petstore v1

```shell
curl -H "x-canary:false" ${PROXY_URL}/findPets
```

```json
[{"id":1,"name":"Dog","status":"available"},{"id":2,"name":"Cat","status":"pending"}]
```

Once we're feeling good about version 2, we can make the default call to `/findPets` go to version 2. The easiest
way to make this change is through applying a YAML file.

```yaml
cat <<EOF | kubectl apply -f -
apiVersion: gateway.solo.io/v1
kind: VirtualService
metadata:
  name: coalmine
  namespace: gloo-system
spec:
  displayName: coalmine
  virtualHost:
    domains:
    - '*'
    name: gloo-system.coalmine
    routes:
    - matcher:
        prefix: /findPetWithId/
      routeAction:
        single:
          destinationSpec:
            rest:
              functionName: findPetById
              parameters:
                headers:
                  :path: /findPetWithId/{id}
          upstream:
            name: default-petstore-v1-8080
            namespace: gloo-system
    - matcher:
        exact: /findPets
      routeAction:
        single:
          destinationSpec:
            rest:
              functionName: findPets
              parameters: {}
          upstream:
            name: default-petstore-v2-8080
            namespace: gloo-system
    - matcher:
        prefix: /petstore
      routeAction:
        single:
          upstream:
            name: default-petstore-v1-8080
            namespace: gloo-system
      routePlugins:
        prefixRewrite:
          prefixRewrite: /api/pets/
EOF
```
