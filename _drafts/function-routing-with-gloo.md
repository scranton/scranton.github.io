---
layout: post
title: Routing with Gloo Function Gateway
date: 2019-02-18 09:29
---

<figure class="aligncenter">
    <img src="https://images.unsplash.com/photo-1495592822108-9e6261896da8?ixlib=rb-1.2.1&ixid=eyJhcHBfaWQiOjEyMDd9&auto=format&fit=crop&w=2250&q=80" />
    <figcaption>Photo by <a href="https://unsplash.com/@pietrozj" target="_blank">Pietro Jeng</a>.</figcaption>
</figure>

This post introduces you to how to use the open source [Solo.io](https://solo.io) [Gloo](https://github.com/solo-io/gloo)
project to help us to route traffic to our Kubernetes hosted services. [Gloo is a function gateway](https://medium.com/solo-io/announcing-gloo-the-function-gateway-3f0860ef6600) that gives users a number of benefits including sophisticated
function level routing, and deep service discovery with introspection of OpenAPI (Swagger) definitions, gRPC reflection,
Lambda discovery and more. This post will start with a simple example of Gloo discovering and routing requests to a
service that exposes REST functions. Later posts will build on this initial example to do highlight ever more complex
scenarios.

## Prerequisites

I'm assuming you're already running a Kubernetes installation. I'm assuming you're using [minikube](https://kubernetes.io/docs/setup/minikube/)
for this post though any recent Kubernetes installation should work as long as you have [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/)
setup and configured correctly for your Kubernetes installation.

## Setup

### Setup Example Services

1. Let's start by installing an example service that exposes 4 REST functions. This service is based on the
[go-swagger petstore example](https://github.com/go-swagger/go-swagger/tree/master/examples/2.0/petstore). I'm including
YAML contents within this post for ease.

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
     selector:
       matchLabels:
         app: petstore-v1
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
   EOF
   ```

   We've installing this service into the `default` namespace, so we can look there to see if there installed correctly.

   ```shell
   kubectl get all --namespace default
   ```

   ```shell
   NAME                               READY   STATUS    RESTARTS   AGE
   pod/petstore-v1-7876bd5bf8-skn5f   1/1     Running   0          6m45s

   NAME                  TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
   service/kubernetes    ClusterIP   10.96.0.1        <none>        443/TCP    2d20h
   service/petstore-v1   ClusterIP   10.97.195.104    <none>        8080/TCP   6m45s

   NAME                          READY   UP-TO-DATE   AVAILABLE   AGE
   deployment.apps/petstore-v1   1/1     1            1           6m45s

   NAME                                     DESIRED   CURRENT   READY   AGE
   replicaset.apps/petstore-v1-7876bd5bf8   1         1         1       6m45s
   ```

1. Let's test our service to make sure it installed correctly. This service is setup to expose on port `8080`, and will
return the list of all pets for `GET` requests on the query path `/api/pets`. The easiest way to test is to
`port-forward` the pod so we can access it locally.

   ```shell
   kubectl get pods --namespace default
   ```

   ```shell
   NAME                           READY   STATUS    RESTARTS   AGE
   petstore-v1-7876bd5bf8-skn5f   1/1     Running   0          13m
   ```

   We'll need the pod name for the port forwarding. Make sure the pod name matches the ones from your system. This will
   forward port 8080 from the service running in your Kubernetes installation to your local machine, i.e. `localhost:8080`.

   ```shell
   kubectl port-forward petstore-v1-7876bd5bf8-skn5f 8080:8080
   ```

   In a separate terminal, run the following.

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
   `gloo-system` namespace, so let's look at everything running there.

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

## Routing

### Upstreams

Before we get into routing, let's talk a little about the concept of `Upstreams`. Upstreams are the services that Gloo
has discovered automatically. Let's look at the upstreams that Gloo has discovered in our Kubernetes cluster.

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

Notice that our petstore service `default-petstore-v1-8080` is different from the other upstreams in
that its details is listing 4 REST service functions. This is because Gloo can autodetect OpenAPI / Swagger
definitions. This allows Gloo to route to individual functions versus what most traditional API Gateways do in
only letting you route to host:port granular services. Let's see that in action.

Let's look a little closer at our petstore upstream. The `glooctl` command let's us output the full details in yaml or
json.

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

For example, we see that the `findPets` REST function is looking for requests on `/api/pets`, and `findPetsById` is
looking for requests on `/api/pets/{id}` where `{id}` is the id number of the single pet who's details are to be returned.

## Basic Routing

Gloo acts like an [Kubernetes Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/), which means it
can allow requests from external to the Kubernetes cluster to access services running inside the cluster. Gloo uses a
concept called `VirtualService` to setup routes to Kubernetes hosted services.

This post will show you how to configure Gloo using the command line tools, and I'll explain a little of what's happening
with each command. I'll also include the YAML at the end of each step if you'd prefer to work in a purely declarative
fashion (versus imperative commands).

1. Setup `VirtualService`. This gives us a place to define a set of related routes. This won't do much till we create
some routes in the next steps.

   ```shell
   glooctl create virtualservice --name coalmine
   ```

   Here's the YAML that will create the same resource. Note that by default the `glooctl` command creates resources in
   the `gloo-system` namespace.

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

1. Create a route for all traffic to go to our service.

   ```shell
   glooctl add route \
      --name coalmine \
      --path-prefix /petstore \
      --dest-name default-petstore-v1-8080 \
      --prefix-rewrite /api/pets/
   ```

   This sets up a simple ingress route so that all requests going to the Gloo Proxy `/` URL are redirected to the
   `default-petstore-v1-8080` service `/api/pets`. Let's test it.

   To get the Gloo proxy host and port number (remember that Gloo is acting like a Kubernetes Ingress), we need to call
   `glooctl proxy url`. Then let's call the route path.

   ```shell
   export PROXY_URL=$(glooctl proxy url)
   curl ${PROXY_URL}/petstore
   ```

   And we should see the same results as when we called the port forwarded service.

   ```json
   [{"id":1,"name":"Dog","status":"available"},{"id":2,"name":"Cat","status":"pending"}]
   ```

## Function Routing

Wouldn't it be better if we could just route to the named REST function versus having to know the specifics of the query
path (i.e. `/api/pets`) the service is expecting? Gloo can help us with that. Let's setup a route to `findPets` REST
function.

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

If we want to route to a function with parameters, we can do that too by telling Gloo how to find the `id` parameter.
In this case, it happens to be a path parameter, but it could come from other parts of the request:

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
curl ${PROXY_URL}/findPetWithId/1
```

```json
{"id":1,"name":"Dog","status":"available"}
```

Here's the complete YAML you could apply to get the same virtual service setup that we just did.

```yaml
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
            name: default-petstore-v1-8080
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
```

## Summary

This post is just the beginning of our function gateway journey with Gloo. Hopefully its given you a taste of some
more sophisticated function level routing options that are available to you. I'll try to follow up with more posts on
even more options available to you.