---
layout: post
title: Canary Deployments with Gloo Function Gateway using Weighted Destinations
date: 2019-03-12T14:34:04Z
description: Introduction to Canary releases with Solo.io Gloo, part II
tags:
- Gloo
- Canary
---

<figure class="aligncenter">
    <img src="https://images.unsplash.com/photo-1527248200634-b8bc0b8b28ae?ixlib=rb-1.2.1&ixid=eyJhcHBfaWQiOjEyMDd9&auto=format&fit=crop&w=1939&q=80"/>
    <figcaption>Photo by <a href="https://unsplash.com/@theformfitness" target="_blank">Form</a>.</figcaption>
</figure>

This is the 3rd post in my 3 part series on doing Canary Releases with [Solo.io](https://solo.io) [Gloo](https://gloo.solo.io).

* In part 1, [Routing with Gloo Function Gateway]({{ site.baseurl }}{% post_url 2019-02-19-function-routing-with-gloo %}),
you learned how to setup [Gloo](https://gloo.solo.io), and using Gloo to setup some initial function level routing rules.
* In part 2, [Canary Deployments with Gloo Function Gateway]({{ site.baseurl }}{% post_url 2019-02-19-canary-deployments-with-solo %}),
you learned how to setup a conditional routing rule that routed requests to the new version of our service only when
a request header was present with the correct value.
* In part 3, [Canary Deployments with Gloo Function Gateway using Weighted Destinations]({{ site.baseurl }}{% post_url 2019-03-12-canary-deployments-with-weighted-routes %}),
we used weighted destinations to route a percentage of request traffic to individual upstream services.

This post will show a different way of doing Canary release by using weighted routes to send a fraction of the request
traffic to the new version (the canary). For example, you could initially route 5% of your request traffic to your new
version to validate that its working correctly in production without risking too much if your new version fails. As you
gain confidence in your new version, you can route more and more traffic to it until you cut over completely, i.e. 100%
to new version, and decommission the old version.

All of the Kubernetes manifests are located at <https://github.com/scranton/gloo-canary-example>. I'd suggest you clone
that repo locally to make it easier to try these example yourself. All command examples assume you are in the top level
directory of that repo.

## Review

Quickly reviewing the previous 2 posts, we learned that Gloo can help with function level routing, and that routing can
be used as part of a Canary Release process, that is slowly testing a new version of our service in an environment. In
the last post, we used Gloo to create a special routing rule to our new version that only forwarded on requests that
included a specific request header. That allows us to deploy our new service into production, while only allowing
request traffic from specific clients, i.e. clients that know to set that specific request header. Once we got
confident that our new version was working as expected, we then changed the Gloo routing rules so that all request
traffic went to the new service. This is a great way to validate that our new deployment is correctly configured in our
environment **before** sending any important traffic to it.

In this post, we're going to expand on that approach with a more sophisticated pattern - weighted routes. With this
capability we can route a percentage of the request traffic to one or more functions. This enhances our previous header
based approach as we can now validate that our new service can handle a managed load of traffic, and as we gain
confidence we can route higher loads to the new version till its handling 100% of the request traffic. If at any point,
we see errors we can either rollback 100% of traffic to the original working version OR debug our service to better
understand why it started to have problems handling a faction of our target load, which in theory should help us fix
our new service version quicker.

You can always combine both the header routing and weighted destination routing, and other routing options Gloo provides.

## Setup

This post assumes you've already run thru the [Canary Deployments with Gloo Function Gateway]({{ site.baseurl }}{% post_url 2019-02-19-canary-deployments-with-solo %})
post, and that you've already got a Kubernetes environment setup with Gloo. If not, please refer back to that post for
setup instructions and the basics of `VirtualServices` and `Routes` with Gloo.

By the end of that post, we had 100% of `findPets` function traffic going to our `petstore-v2` service, and the other
functions going to the original `petstore-v1`. Let's validate our services before we make any changes.

```shell
export PROXY_URL=$(glooctl proxy url)
curl ${PROXY_URL}/findPets
```

The call to `findPets` should have been routed to `petstore-v2`, which should return the following result.

```json
[{"id":1,"name":"Dog","status":"v2"},{"id":2,"name":"Cat","status":"v2"},{"id":3,"name":"Parrot","status":"v2"}]
```

And calls to `findPetWithId` should route to `petstore-v1`, which only has 2 pets (Dog & Cat) each with a status of
`available` and `pending` respectively (versus status of `v2` for `petstore-v2` responses).

```shell
curl ${PROXY_URL}/findPetWithId/1
 ```
 
 ```json
{"id":1,"name":"Dog","status":"available"}
```
 
```shell
curl ${PROXY_URL}/findPetWithId/2
 ```
 
 ```json
{"id":2,"name":"Cat","status":"pending"}
```

```shell
curl ${PROXY_URL}/findPetWithId/3
 ```
 
 ```json
{"code":404,"message":"not found: pet 3"}                                                                                                                                                                                             
```

So let's play with doing a Canary Release with weighted destinations to migrate the `findPetWithId` function.

## Setting up Weighted Destinations in Gloo

Let's start by looking at our existing, virtual service `coalmine`

```shell
kubectl get virtualservice coalmine --namespace gloo-system --output yaml
```

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
        prefix: /findPets
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
        prefix: /findPetWithId
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

To create a weighted destination, we need to change the `routeAction` from `single` to `multi`
and provide 2+ `destination`, which are `destinationSpec` with `weight`. For example, to route
10% of request traffic to `findPetWithId` to `petstore-v2` and the remaining 90% to `petstore-v1`.

```shell
kubectl apply -f coalmine-virtual-service-part-3-weighted.yaml
```

Here's the relevant part of the virtual service manifest showing the weighted destination spec.

{% github_sample_ref /scranton/gloo-canary-example/master/coalmine-virtual-service-part-3-weighted.yaml %}
{% highlight yaml %}
{% github_sample /scranton/gloo-canary-example/master/coalmine-virtual-service-part-3-weighted.yaml 25 50 %}
{% endhighlight %}

Let's run a shell loop to test, remember that `petstore-v2` responses have a `status` field of `v2`. The following
command will call our function 20 times, and we should see ~2 responses (~10%) return with `"status":"v2"`.

```shell
COUNTER=0
while [ $COUNTER -lt 20 ]; do
    curl ${PROXY_URL}/findPetWithId/1
    let COUNTER=COUNTER+1
done
```

```json
{"id":1,"name":"Dog","status":"available"}
{"id":1,"name":"Dog","status":"available"}
{"id":1,"name":"Dog","status":"available"}
{"id":1,"name":"Dog","status":"available"}
{"id":1,"name":"Dog","status":"available"}
{"id":1,"name":"Dog","status":"available"}
{"id":1,"name":"Dog","status":"available"}
{"id":1,"name":"Dog","status":"available"}
{"id":1,"name":"Dog","status":"v2"}
{"id":1,"name":"Dog","status":"available"}
{"id":1,"name":"Dog","status":"available"}
{"id":1,"name":"Dog","status":"available"}
{"id":1,"name":"Dog","status":"available"}
{"id":1,"name":"Dog","status":"v2"}
{"id":1,"name":"Dog","status":"available"}
{"id":1,"name":"Dog","status":"available"}
{"id":1,"name":"Dog","status":"available"}
{"id":1,"name":"Dog","status":"available"}
{"id":1,"name":"Dog","status":"available"}
{"id":1,"name":"Dog","status":"available"}
```

Now if we want to increase the traffic to our new version, we just need to update the `weight` attibutes in the 2
`destination` objects. Gloo sums all of the `weight` values within a given weighted destination route, and routes the
respective percentage to each destination. So if we set both route weights to `1` then each route would get `1/2` or 50%
of the request traffic. I'd recommend setting the values with a sum of 100 so they look like percentages for greater
readability. The following example will update our routes to do 50/50 traffic split.

```shell
kubectl apply -f coalmine-virtual-service-part-3-weighted-50-50.yaml
```

{% github_sample_ref /scranton/gloo-canary-example/master/coalmine-virtual-service-part-3-weighted-50-50.yaml %}
{% highlight yaml %}
{% github_sample /scranton/gloo-canary-example/master/coalmine-virtual-service-part-3-weighted-50-50.yaml 25 50 %}
{% endhighlight %}

And if we run our test loop again, we should see about 10 of the 20 requests returning `"status":"v2`.

```shell
COUNTER=0
while [ $COUNTER -lt 20 ]; do
    curl ${PROXY_URL}/findPetWithId/1
    let COUNTER=COUNTER+1
done
```

```json
{"id":1,"name":"Dog","status":"v2"}
{"id":1,"name":"Dog","status":"v2"}
{"id":1,"name":"Dog","status":"v2"}
{"id":1,"name":"Dog","status":"available"}
{"id":1,"name":"Dog","status":"v2"}
{"id":1,"name":"Dog","status":"available"}
{"id":1,"name":"Dog","status":"available"}
{"id":1,"name":"Dog","status":"v2"}
{"id":1,"name":"Dog","status":"v2"}
{"id":1,"name":"Dog","status":"available"}
{"id":1,"name":"Dog","status":"v2"}
{"id":1,"name":"Dog","status":"v2"}
{"id":1,"name":"Dog","status":"available"}
{"id":1,"name":"Dog","status":"available"}
{"id":1,"name":"Dog","status":"v2"}
{"id":1,"name":"Dog","status":"v2"}
{"id":1,"name":"Dog","status":"available"}
{"id":1,"name":"Dog","status":"available"}
{"id":1,"name":"Dog","status":"available"}
{"id":1,"name":"Dog","status":"v2"}
```

## Summary

This series has hopefully given you all a taste of how [Solo.io](https://solo.io) [Gloo](https://gloo.solo.io) can help
you create more interesting applications, and also enhance your application delivery approaches. These posts have shown
how to do function level request routing, and how you can enhance those routing rules by requiring presence of request
headers and doing managed load balancing by specifying the percentage of traffic going to individual upstream
destinations. Gloo supports many more options, and I hope you'll continue your journey by going to <https://gloo.solo.io>
to learn more. 