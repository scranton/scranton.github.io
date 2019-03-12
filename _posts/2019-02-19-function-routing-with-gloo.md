---
layout: post
title: Routing with Gloo Function Gateway
date: 2019-02-19T14:26:47Z
description: Introduction to function level routing with Solo.io Gloo.
tags:
- Gloo
---

<figure class="aligncenter">
    <img src="https://images.unsplash.com/photo-1495592822108-9e6261896da8?ixlib=rb-1.2.1&ixid=eyJhcHBfaWQiOjEyMDd9&auto=format&fit=crop&w=2250&q=80"/>
    <figcaption>Photo by <a href="https://unsplash.com/@pietrozj" target="_blank">Pietro Jeng</a>.</figcaption>
</figure>

This is the 1st post in my 3 part series on doing Canary Releases with [Solo.io](https://solo.io) [Gloo](https://gloo.solo.io).

* In part 1, [Routing with Gloo Function Gateway]({{ site.baseurl }}{% post_url 2019-02-19-function-routing-with-gloo %}),
you learned how to setup [Gloo](https://gloo.solo.io), and using Gloo to setup some initial function level routing rules.
* In part 2, [Canary Deployments with Gloo Function Gateway]({{ site.baseurl }}{% post_url 2019-02-19-canary-deployments-with-solo %}),
you learned how to setup a conditional routing rule that routed requests to the new version of our service only when
a request header was present with the correct value.
* In part 3, [Canary Deployments with Gloo Function Gateway using Weighted Destinations]({{ site.baseurl }}{% post_url 2019-03-12-canary-deployments-with-weighted-routes %}),
we used weighted destinations to route a percentage of request traffic to individual upstream services.

This post introduces you to how to use the open source [Solo.io](https://solo.io) [Gloo](https://github.com/solo-io/gloo)
project to help you to route traffic to your Kubernetes hosted services. [Gloo is a function gateway](https://medium.com/solo-io/announcing-gloo-the-function-gateway-3f0860ef6600) that gives users a number of benefits including sophisticated
function level routing, and deep service discovery with introspection of OpenAPI (Swagger) definitions, gRPC reflection,
Lambda discovery and more. This post will start with a simple example of Gloo discovering and routing requests to a
service that exposes REST functions. Later posts will build on this initial example to do highlight ever more complex
scenarios.

# Prerequisites

I'm assuming you're already running a Kubernetes installation. I'm assuming you're using [minikube](https://kubernetes.io/docs/setup/minikube/)
for this post though any recent Kubernetes installation should work as long as you have [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/)
setup and configured correctly for your Kubernetes installation.

# Setup

## Setup example service

All of the Kubernetes manifests are located at <https://github.com/scranton/gloo-canary-example>. I'd suggest you clone
that repo locally to make it easier to try these example yourself. All command examples assume you are in the top level
directory of that repo.

Let's start by installing an example service that exposes 4 REST functions. This service is based on the
[go-swagger petstore example](https://github.com/go-swagger/go-swagger/tree/master/examples/2.0/petstore).

```shell
kubectl apply -f petstore-v1.yaml
```

{% github_sample_ref /scranton/gloo-canary-example/master/petstore-v1.yaml %}
{% highlight yaml %}
{% github_sample /scranton/gloo-canary-example/master/petstore-v1.yaml %}
{% endhighlight %}

We've installing this service into the `default` namespace, so we can look there to see if it's installed correctly.

```shell
kubectl get all --namespace default
```

```
NAME                              READY   STATUS    RESTARTS   AGE
pod/petstore-v1-986747fc8-6hn9p   1/1     Running   0          16s

NAME                  TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
service/kubernetes    ClusterIP   10.96.0.1      <none>        443/TCP    22h
service/petstore-v1   ClusterIP   10.110.99.86   <none>        8080/TCP   17s

NAME                          READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/petstore-v1   1/1     1            1           16s

NAME                                    DESIRED   CURRENT   READY   AGE
replicaset.apps/petstore-v1-986747fc8   1         1         1       16s
```

Let's test our service to make sure it installed correctly. This service is setup to expose on port `8080`, and will
return the list of all pets for `GET` requests on the query path `/api/pets`. The easiest way to test is to
`port-forward` the service so we can access it locally. We'll need the service name for the port forwarding. Make sure
the service name matches the ones from your system. This will forward port 8080 from the service running in your Kubernetes
installation to your local machine, i.e. `localhost:8080`.

Get the service names for your installation

```shell
kubectl get service --namespace default
```

```shell
NAME          TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
kubernetes    ClusterIP   10.96.0.1      <none>        443/TCP    22h
petstore-v1   ClusterIP   10.110.99.86   <none>        8080/TCP   42s
```

Setup the port forwarding

```shell
kubectl port-forward service/petstore-v1 8080:8080
```

In a separate terminal, run the following. The petstore function should return 2 pets: Dog and Cat.

```shell
curl localhost:8080/api/pets
```

```json
[{"id":1,"name":"Dog","status":"available"},{"id":2,"name":"Cat","status":"pending"}]
```

You can also get the Swagger spec as well.

```shell
curl localhost:8080/swagger.json
```

You can kill all the port forwards. Now we'll setup Gloo...

## Setup Gloo

Let's setup the Gloo command line utility. Full instructions are at the [Gloo doc site](https://gloo.solo.io/installation/).
Here are the quick setup instructions.

Setup the `glooctl` command line tool. This makes installation, upgrade, and operations of Gloo easier. Full installation
instructions are located on the <https://gloo.solo.io> site.
 
If you're a Mac or Linux Homebrew user, I'd recommend installing as follows.

```shell
brew install glooctl
```

Now let's install Gloo into your Kubernetes installation.

```shell
glooctl install gateway
```

Pretty easy, eh? Let's verify that its installed and running correctly. Gloo by default creates and installs into the
`gloo-system` namespace, so let's look at everything running there.

```shell
kubectl get all --namespace gloo-system
```

And the output should look something like the following.

```
NAME                                 READY   STATUS    RESTARTS   AGE
pod/discovery-66c865f9bc-h6v8f       1/1     Running   0          22h
pod/gateway-777cf4486c-8mzj5         1/1     Running   0          22h
pod/gateway-proxy-5f58774ccc-rcmdv   1/1     Running   0          22h
pod/gloo-5c6c4466f-ptc8v             1/1     Running   0          22h

NAME                    TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
service/gateway-proxy   LoadBalancer   10.97.13.246    <pending>     80:31333/TCP,443:32470/TCP   22h
service/gloo            ClusterIP      10.104.80.219   <none>        9977/TCP                     22h

NAME                            READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/discovery       1/1     1            1           22h
deployment.apps/gateway         1/1     1            1           22h
deployment.apps/gateway-proxy   1/1     1            1           22h
deployment.apps/gloo            1/1     1            1           22h

NAME                                       DESIRED   CURRENT   READY   AGE
replicaset.apps/discovery-66c865f9bc       1         1         1       22h
replicaset.apps/gateway-777cf4486c         1         1         1       22h
replicaset.apps/gateway-proxy-5f58774ccc   1         1         1       22h
replicaset.apps/gloo-5c6c4466f             1         1         1       22h
```

# Routing

## Upstreams

Before we get into routing, let's talk a little about the concept of `Upstreams`. Upstreams are the services that Gloo
has discovered automatically. Let's look at the upstreams that Gloo has discovered in our Kubernetes cluster.

```shell
glooctl get upstreams
```

You may see some different entries than what follows. It depends on your Kubernetes cluster, and what is running currently.

```
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

Notice that our petstore service `default-petstore-v1-8080` is different from the other upstreams in that its details is
listing 4 REST service functions: `addPet`, `deletePet`, `findPetByID`, and `findPets`. This is because Gloo can auto-detect
OpenAPI / Swagger definitions. This allows Gloo to route to individual functions versus what most traditional API
Gateways do in only letting you route only to host:port granular services. Let's see that in action.

Let's look a little closer at our petstore upstream. The `glooctl` command let's us output the full details in yaml or
json.

```shell
glooctl get upstream default-petstore-v1-8080 --output yaml
```

{% highlight yaml %}
{% github_sample /scranton/gloo-canary-example/master/default-petstore-v1-8080-upstream.yaml %}
{% endhighlight %}

Here we see that the `findPets` REST function is looking for requests on `/api/pets`, and `findPetsById` is
looking for requests on `/api/pets/{id}` where `{id}` is the id number of the single pet who's details are to be returned.

### Basic Routing

Gloo acts like an (better) [Kubernetes Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/), which
means it can allow requests from external to the Kubernetes cluster to access services running inside the cluster. Gloo
uses a concept called `VirtualService` to setup routes to Kubernetes hosted services.

This post will show you how to configure Gloo using the command line tools, and I'll explain a little of what's happening
with each command. I'll also include the YAML at the end of each step if you'd prefer to work in a purely declarative
fashion (versus imperative commands).

Setup `VirtualService`. This gives us a place to define a set of related routes. This won't do much till we create
some routes in the next steps.

```shell
glooctl create virtualservice --name coalmine
```

Here's the YAML that will create the same resource as the `glooctl` command we just ran. Note that by default the
`glooctl` command creates resources in the `gloo-system` namespace.

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
```

Create a route for all traffic to go to our service.

```shell
glooctl add route \
  --name coalmine \
  --path-prefix /petstore \
  --dest-name default-petstore-v1-8080 \
  --prefix-rewrite /api/pets
```

This sets up a simple ingress route so that all requests going to the Gloo Proxy `/` URL are redirected to the
`default-petstore-v1-8080` service `/api/pets`. Let's test it. To get the Gloo proxy host and port number (remember
that Gloo is acting like a Kubernetes Ingress), we need to call `glooctl proxy url`. Then let's call the route path.

```shell
export PROXY_URL=$(glooctl proxy url)
curl ${PROXY_URL}/petstore
```

And we should see the same results as when we called the port forwarded service.

```json
[{"id":1,"name":"Dog","status":"available"},{"id":2,"name":"Cat","status":"pending"}]
```

Here's the full YAML for the coal virtual service created so far.

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
        prefix: /petstore
      routeAction:
        single:
          upstream:
            name: default-petstore-v1-8080
            namespace: gloo-system
      routePlugins:
        prefixRewrite:
          prefixRewrite: /api/pets
```

## Function Routing

Would it be better if we could just route to the named REST function versus having to know the specifics of the query
path (i.e. `/api/pets`) the service is expecting? Gloo can help us with that. Let's setup a route to `findPets` REST
function.

```shell
glooctl add route \
   --name coalmine \
   --path-prefix /findPets \
   --dest-name default-petstore-v1-8080 \
   --rest-function-name findPets
```

And test it. We should see the same results as the request to `/petstore` as both those examples were exercising the
`findPets` REST function in the petstore service. This also shows that Gloo allows you to create multiple routing rules
for the same REST functions, if you want.

```shell
curl ${PROXY_URL}/findPets
```

If we want to route to a function with parameters, we can do that too by telling Gloo how to find the `id` parameter.
In this case, it happens to be a path parameter, but it could come from other parts of the request.

**Note**: We're about to create a route with a **different** name `findPetWithId` than the function name `findPetById`
it is routing to. Gloo allows you to setup routing rules for any prefix path to any function name.


```shell
glooctl add route \
   --name coalmine \
   --path-prefix /findPetWithId \
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

And the pet with id 2

```shell
curl ${PROXY_URL}/findPetWithId/2
```

```json
{"id":2,"name":"Cat","status":"pending"}
```

Here's the complete YAML you could apply to get the same virtual service setup that we just did. To recreate this
virtual service you could just `kubectl apply` the following YAML.

{% github_sample_ref /scranton/gloo-canary-example/master/coalmine-virtual-service-part-1.yaml %}
{% highlight yaml %}
{% github_sample /scranton/gloo-canary-example/master/coalmine-virtual-service-part-1.yaml %}
{% endhighlight %}

# Summary

This post is just the beginning of our function gateway journey with Gloo. Hopefully its given you a taste of some
more sophisticated function level routing options that are available to you. I'll try to follow up with more posts on
even more options available to you.