---
layout: post
title: Canary Deployments with Gloo Function Gateway
date: 2019-02-19T15:05:59Z
description: Introduction to Canary releases with Solo.io Gloo.
tags:
- Gloo
- Canary
---

<figure class="aligncenter">
    <img src="https://images.unsplash.com/photo-1544215901-f855c2b93434?ixlib=rb-1.2.1&ixid=eyJhcHBfaWQiOjEyMDd9&auto=format&fit=crop&w=2126&q=80"/>
    <figcaption>Photo by <a href="https://unsplash.com/@zabit" target="_blank">Zab Consulting</a>.</figcaption>
</figure>

This is the 2nd post in my 3 part series on doing Canary Releases with [Solo.io](https://solo.io) [Gloo](https://gloo.solo.io).

* In part 1, [Routing with Gloo Function Gateway]({{ site.baseurl }}{% post_url 2019-02-19-function-routing-with-gloo %}),
you learned how to setup [Gloo](https://gloo.solo.io), and using Gloo to setup some initial function level routing rules.
* In part 2, [Canary Deployments with Gloo Function Gateway]({{ site.baseurl }}{% post_url 2019-02-19-canary-deployments-with-solo %}),
you learned how to setup a conditional routing rule that routed requests to the new version of our service only when
a request header was present with the correct value.
* In part 3, [Canary Deployments with Gloo Function Gateway using Weighted Destinations]({{ site.baseurl }}{% post_url 2019-03-12-canary-deployments-with-weighted-routes %}),
we used weighted destinations to route a percentage of request traffic to individual upstream services.

This post expands on the [Function Routing with Gloo]({{ site.baseurl }}{% post_url 2019-02-19-function-routing-with-gloo %})
post to show you how to do a Canary release of a new version of a function. [Gloo is a function gateway](https://medium.com/solo-io/announcing-gloo-the-function-gateway-3f0860ef6600)
that gives users a number of benefits including sophisticated function level routing, and deep service discovery with
introspection of OpenAPI (Swagger) definitions, gRPC reflection, Lambda discovery and more. This post will show a simple
example of Gloo discovering 2 different deployments of a service, and setting up some routes. The route rules will use the
presence of a request header `x-canary:true` to influence runtime routing to either version 1 or version 2 of our function.
Then once we're happy with our new version, we will update the route so all requests now go to version 2 of our
service. All without changing or even redeploying our 2 services. But first, let's set some context...

# Background

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
your production environment you can't be positive everything will work as expected. So having a way to release a new
version into production concurrently with the existing version(s) with some way to route traffic can be helpful. Ideally,
we'd like to route most traffic to existing, known to work version, and have a way for some (test) requests go to the
new version. Once you're feeling comfortable that your new version is working like you expect, then and only then, do you start
routing most/all requests to the new version, and then eventually decommission the original service.

Being able to change request routes **without** needing to change or redeploy your code, I think, is very helpful in
building confidence that your code is ready for production. That is, if you need to change your code or use a code based
feature flag, then your exercising different code paths and/or changing deployed configuration settings. I feel its
better if you can deploy your service, code and configurations, all ready for production, and use an external mechanism
to manage request routing.

Gloo uses [Envoy](https://www.envoyproxy.io/), which is a super high performance service proxy, to do the request routing.
In this example, we'll use a request header to influence the routing, though we could also use other variables like the
IP range of the requestor to drive routing decisions. That is, if requests are coming from specific test machines we can
route them to our new version. Lots more information on how Gloo and Envoy works can be found on the [Solo.io](https://solo.io)
website. On to the example...

This post assumes you've already run thru the [Function Routing with Gloo]({{ site.baseurl }}{% post_url 2019-02-19-function-routing-with-gloo %})
post, and that you've already got a Kubernetes environment setup with Gloo. If not, please refer back to that post for
setup instructions and the basics of `VirtualServices` and `Routes` with Gloo.

All of the Kubernetes manifests are located at <https://github.com/scranton/gloo-canary-example>. I'd suggest you clone
that repo locally to make it easier to try these example yourself. All command examples assume your in the top level
director of that repo.

# Review

In the previous post, we had a single service `petstore-v1`, and we setup Gloo to route requests to its `findPets`
REST function. Let's test that its still working as expected. Remember we need to get Gloo's proxy url by calling the
`glooctl proxy url` command, and then we can make requests against that with the `/findPets` route that we previously
setup. If still working correctly we should get 2 Pets back.

```shell
export PROXY_URL=$(glooctl proxy url)
curl ${PROXY_URL}/findPets
```

```json
[{"id":1,"name":"Dog","status":"available"},{"id":2,"name":"Cat","status":"pending"}]
```

# Canary Routing

Now let's deploy a version 2 of our service, and let's setup a canary route for the `findPets` function. That is, by
default we'll route to version 1 of the function, and if there is a request header `x-canary:true` set, we'll route that
request to version 2 of our function.

## Install and verify petstore version 2 example service

Let's first deploy version 2 of our petstore service. This version has been modified to return 3 pets.

```shell
kubectl apply -f petstore-v2.yaml
```

{% github_sample_ref /scranton/gloo-canary-example/master/petstore-v2.yaml %}
{% highlight yaml %}
{% github_sample /scranton/gloo-canary-example/master/petstore-v2.yaml %}
{% endhighlight %}

Verify its setup right

```shell
kubectl get services --namespace default
```

```shell
NAME          TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
kubernetes    ClusterIP   10.96.0.1       <none>        443/TCP    22h
petstore-v1   ClusterIP   10.110.99.86    <none>        8080/TCP   33m
petstore-v2   ClusterIP   10.109.91.120   <none>        8080/TCP   6s
```

Now let's setup a port forward to see if it works. When we do a `GET` against `/api/pets` we should get back 3 pets.

```shell
kubectl port-forward services/petstore-v2 8080:8080
```

And in a different terminal, run the following to see if we get back 3 pets for version 2 of our service.

```shell
curl localhost:8080/api/pets
```

```json
[{"id":1,"name":"Dog","status":"v2"},{"id":2,"name":"Cat","status":"v2"},{"id":3,"name":"Parrot","status":"v2"}]
```

You should kill all port forwarding as we'll use Gloo to proxy future tests.

## Setup Canary Route

Let's setup a new function route rule for petstore version 2 `findPets` function that depends on the presence of the
`x-canary:true` request header.

```shell
glooctl add route \
   --name coalmine \
   --path-prefix /findPets \
   --dest-name default-petstore-v2-8080 \
   --rest-function-name findPets \
   --header x-canary=true
```

Default routing should still go to petstore version 1, and return only 2 pets.

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

Here's the complete YAML for our `coalmine` virtual service that you could `kubectl apply` if you wanted to recreate

{% github_sample_ref /scranton/gloo-canary-example/master/coalmine-virtual-service-part-2-header.yaml %}
{% highlight yaml %}
{% github_sample /scranton/gloo-canary-example/master/coalmine-virtual-service-part-2-header.yaml %}
{% endhighlight %}

The part of the virtual service manifest that is specifying the header based routing is highlighted as follows.

{% highlight yaml %}
{% github_sample /scranton/gloo-canary-example/master/coalmine-virtual-service-part-2-header.yaml 12 18 %}
{% endhighlight %}

### Make version 2 the default for all requests

Once we're feeling good about version 2 of our function, we can make the default call to `/findPets` go to version 2.
Note that will Gloo as your function gateway, you do not have to route all function requests to version 2 of the petstore
service. In this example, we're only routing requests for the `findPets` function to version 2. All other requests are
going to version 1 of petstore. This partial routing may not always work for all services; this post is showing that
Gloo makes this level of granularity possible when it helps you more fine tune your application upgrading decisions. For
example, this may make sense if you want to patch a critical bug but are not ready to role out other breaking changes in
a new service version.

The easiest way to make change the routing rules to route all requests to version 2 `findPets` is by applying a YAML
file. You can use the `glooctl` command line tool to add and remove routes, but it takes several calls.

{% github_sample_ref /scranton/gloo-canary-example/master/coalmine-virtual-service-part-2-v2.yaml %}
{% highlight yaml %}
{% github_sample /scranton/gloo-canary-example/master/coalmine-virtual-service-part-2-v2.yaml %}
{% endhighlight %}

# Summary

This post has shown you have to leverage the Gloo function gateway to do a Canary Release of a new version of a function,
and allow you to do very granular function level routing to validate your new function is working correctly. Then it showed
changing routing rules so all traffic goes to the new version. All without redeploying either of the 2 service
implementations. In this post we used the presence of a request header to influence function routing; we could also have
done routing based on IP range of incoming request or other variables. This hopefully shows you the power and flexibility
that Gloo function gateway can provide you in your journey to microservices and service mesh.