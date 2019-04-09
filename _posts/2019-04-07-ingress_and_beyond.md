---
layout: post
title: Kubernetes Ingress Past, Present, and Future
date: 2019-04-07T13:29:21Z
description: Learn how Knative and Solo.io Gloo work together to support on demand code delivery in Kubernetes.
tags:
- Gloo
- Ingress
- Kubernetes
---

<figure class="aligncenter">
    <img src="/assets/reflect.jpeg"/>
    <figcaption>Photo by <a href="https://unsplash.com/@lukeporter" target="_blank">Luke Porter</a>.</figcaption>
</figure>

# Overview

This post was inspired by listening to the February 19, 2019, [Kubernetes Podcast](https://kubernetespodcast.com/),
["Ingress, with Tim Hockin."](https://kubernetespodcast.com/episode/041-ingress/) The Kubernetes Podcast is turning out
to be a very well done podcast overall, and well worth the listen. In the Ingress episode, the podcasters interview
Tim Hockin who's one of the original Kubernetes co-founders, a team lead on the Kubernetes predecessor Borg/Omega,
and is still very active within the Kubernetes community such as chairing the [Kubernetes Network Special Interest Group](https://github.com/kubernetes/community/tree/master/sig-network)
that currently own the Ingress resource specification. Tim talks in the podcast about the history of the
[Kubernetes Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/), current developments around
Ingress, and proposed futures. This talk inspired me to reflect on both Ingress Controllers (realizes the implementation of
Ingress manifest) and Ingress the concept (allow client outside the Kubernetes cluster to access services running in
the Kubernetes cluster).

## So what's a Kubernetes Ingress?

To paraphrase from the [Kubernetes Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/#what-is-ingress)
documentation, Ingress is an L7 network service that exposes HTTP(S) routes from outside to inside a Kubernetes cluster.
A Kubernetes cluster may have one or more [Ingress Controllers](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/)
running, and each controller manages service reachability, load balancing, TLS/SSL termination, and other services for
that controller's associated routes.

![Gloo as Ingress](/assets/gloo_as_ingress.png)

Each Ingress manifest includes an annotation that indicates which Ingress controller should manage that Ingress resource.
For example, to have [Solo.io](https://solo.io) [Gloo](https://gloo.solo.io) manage a specific Ingress resource, you
would specify the following. Note the included annotation `kubernetes.io/ingress.class: gloo`.

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: gloo
  labels:
    chart: jsonplaceholder-v0.1.0
  name: jsonplaceholder-jsonplaceholder
  namespace: default
spec:
  rules:
  - host: gloo.example.com
    http:
      paths:
      - path: /.*
        backend:
          serviceName: jsonplaceholder-jsonplaceholder
          servicePort: 8080
```

## Ingress Challenges

Ingress has existed as a beta extension since Kubernetes 1.1, and it's proven to be a lowest common denominator API. For
example, the NGNIX community Ingress Controller is used by many in production, but that NGNIX Ingress controller
requires the use of many [NGNIX specific Ingress Annotations](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/)
for all but the simplest use cases. The current Kubernetes Ingress resource specification has many limitations like that all
referenced services and secrets MUST be in the same namespace as the Ingress, i.e., no cross namespace referencing.
And there have been long debates about how exactly to interpret the `path` attribute; is it a regular expression like
the documentation implies OR is it a path prefix like some controllers like NGNIX implement. These challenges have made
it, in practice, difficult to have an Ingress manifest that is portable across implementations. The current Ingress
manifest has also proven difficult to round trip sync with [Custom Resources (CRD)](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/)
which is unfortunate as CRDs are proving to be a beneficial way to extend Kubernetes.

## What's Next for Ingress?

In the podcast, Tim Hockin says given how many are using the current beta Ingress spec in production, there is a push
to move the existing Ingress spec to GA status, and then start work on a next-generation specification, either an Ingress v2 or
breaking up Ingress across multiple CRDs. Tim mentions how the Kubernetes community is looking at several
[Envoy](https://www.envoyproxy.io/) based Ingress implementations for inspiration for the next generation of Ingress. For
example, [Heptio Contour](https://github.com/heptio/contour) has created a very interesting, and implementation
neutral CRD called [Ingress Route](https://github.com/heptio/contour/blob/master/docs/ingressroute.md). An Ingress Route
looks to address the governance challenges with Ingress, for example, if a company wants to expose a
`/eng` route path there are many challenges with the current Ingress model as you can have conflicting Ingress
manifests for the route `/eng`. Ingress Route provides a way to create governance and delegation such as cluster admins
can define a virtual host `/eng` and delegate implementation explicitly to the `eng` namespace, and this prevents others
from overriding that route path.

The [Istio](https://istio.io/) community, also based on Envoy like Heptio Contour, are also defining Ingress CRDs.

It will be fascinating to see how Ingress evolves in the not too distant future.

Related reading: [API Gateways are going through an identity crisis](https://medium.com/solo-io/api-gateways-are-going-through-an-identity-crisis-d1d833a313d7).

## Demo Time

I find it helpful to see working code to help make concepts more real, so let's run through a few examples of Ingress and
beyond.

For this example, I'm going to use a Kubernetes service created from <https://jsonplaceholder.typicode.com/>, which
provides a quick set of REST APIs that provide different JSON output that can be helpful for testing. It's based on
a Node.js [json-server](https://github.com/typicode/json-server) - it's very cool and worth looking at independently. I
forked the original GitHub [jsonplaceholder repository](https://github.com/typicode/jsonplaceholder), ran [`draft create`](https://draft.sh/)
on the project, and made a couple of tweaks to the generated [Helm](https://helm.sh/) chart. [Draft](https://draft.sh/)
is a super fast and easy way to bootstrap existing code into Kubernetes. I'm running all of this example locally using
[minikube](https://kubernetes.io/docs/setup/minikube/).

The jsonplaceholder service comes with six common resources each of which returns several JSON objects. For this
example, we'll be getting the first user resource at `/users/1`.

* `/posts`	100 posts
* `/comments`	500 comments
* `/albums`	100 albums
* `/photos`	5000 photos
* `/todos`	200 todos
* `/users`	10 users

Following is a script to try this example yourself, and there's also an [asciinema](https://asciinema.org/) playback so you can see
what it looks like running on my machine. We'll unpack what's happening following the playback.

```shell
# Install tooling
brew update
brew cask install minikube
brew install kubernetes-cli \
  kubernetes-helm \
  azure/draft/draft \
  glooctl

# Create and set up local Kubernetes Cluster
minikube start
helm init
draft init
glooctl install ingress

# Draft runs better locally if you configure
# against minikube docker daemon
eval $(minikube docker-env)

# Get and run the example
git clone https://github.com/scranton/jsonplaceholder.git
cd jsonplaceholder
draft up

# Validate all is running
kubectl get all --namespace default
kubectl get all --namespace gloo-system
kubectl get ingress --namespace default
curl --header "Host: gloo.example.com" \
  $(glooctl proxy url --name ingress-proxy)/users/1
```

<script id="asciicast-JjLna1ONZiGmQeYrJhP4oNcS5" src="https://asciinema.org/a/JjLna1ONZiGmQeYrJhP4oNcS5.js" data-speed="1.5" data-rows="32" data-cols="80" async></script>

## What Happened?

We installed local tooling (you can check respective websites for full install details)

* [minikube](https://kubernetes.io/docs/setup/minikube/)
* [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/)
* [Helm](https://helm.sh/)
* [Draft](https://draft.sh/)
* [glooctl](https://gloo.solo.io)

We then started up a local Kubernetes cluster (minikube) and initialized Helm and Draft. We also installed
[Gloo ingress](https://gloo.solo.io/user_guides/basic_ingress/) into our local cluster.

We then `git clone` our example and used `draft up` to build and deploy it to our cluster. Let's spend a minute on
what happened in this step. I originally forked the `jsonplaceholder` GitHub repository and ran `draft create` against
its code. Draft autodetects the source code language, in this case, Node.js, and creates both a `Dockerfile` that builds
our example application into an image container and creates a default Helm chart. I then made a few minor tweaks to the
Helm chart to enable its Ingress. Let's look at that Ingress manifest. The main changes are the addition of the
`ingress.class: gloo` annotation to mark this Ingress for Gloo's Ingress Controller. And the host is set to
`gloo.example.com`, which is why our curl statement set `curl --header "Host: gloo.example.com"`.

<figure>
{% highlight yaml %}
{% raw %}
{{- if .Values.ingress.enabled -}}
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: {{ template "fullname" . }}
  labels:
    chart: "{{ .Chart.Name }}-{{ .Chart.Version | replace "+" "_" }}"
  annotations:
    kubernetes.io/ingress.class: {{ .Values.ingress.class }}
spec:
  rules:
  - host: {{ .Values.ingress.basedomain }}
    http:
      paths:
      - path: /.*
        backend:
          serviceName: {{ template "fullname" . }}
          servicePort: {{ .Values.service.externalPort }}
{{- end -}}
{% endraw %}
{% endhighlight %}
  <figcaption>charts/template/ingress.yaml</figcaption>
</figure>

For more examples of using Gloo as an basic Ingress controller you can check out
[Kubernetes Ingress Control using Gloo](https://scott.cranton.com/kubernetes-ingress-controlling-with-gloo.html).

You may have also noticed the call to `$(glooctl proxy url --name ingress-proxy)` in the curl command. This is needed
when you're running in a local environment like minikube and you need to get the host IP and port of the Gloo proxy
server. When Gloo is deployed to a Cloud Provider like Google or AWS, then those environments would associate a static IP and
allow port 80 (or port 443 for HTTPS) to be used, and that static IP would be registered with a DNS server, i.e., 
when Gloo is deployed to a cloud-managed Kubernetes you could do `curl http://gloo.example.com/users/1`.

## Ingress Example Challenges

Let's say we wanted to remap the exiting `/users/1` to `/people/1` as users are people too. That becomes tricky with
Ingress manifests as we can set up a second rule for `/people`, but we need to rewrite that path to `/users` before
sending to our service as it doesn't know how to handle requests for `/people`. If you were using the NGNIX ingress, you
could add another annotation `nginx.ingress.kubernetes.io/rewrite-target: /`, but now we're adding implementation
specific annotations, that is, the nginx annotation won't be recognized by other Ingress Controllers. And annotations
are a flat name space so adding lots of annotations can get quite messy, which is part of why
[Custom Resources (CRD)](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/) was
created. Let's see what the original route, and our path re-writing route, would look like in a CRD based Ingress: Gloo.

## Gloo Virtual Services

Gloo uses a concept called [Virtual Service](https://gloo.solo.io/introduction/concepts#virtual-services) that is
derived from similar ideas in Istio and Envoy and is conceptually equivalent to an Ingress resource. Easiest to show
you the equivalent of the example Ingress we've created so far in a Gloo Virtual Service.

```yaml
apiVersion: gateway.solo.io/v1
kind: VirtualService
metadata:
  name: default
  namespace: gloo-system
spec:
  virtualHost:
    domains:
    - gloo.example.com
    routes:
    - matcher:
        prefix: /
      routeAction:
        single:
          upstream:
            name: default-jsonplaceholder-jsonplaceholder-8080
            namespace: gloo-system
```

You'll notice that it looks very similar to the Ingress we had previously created with a few subtle changes. The path
specifier is `prefix: /` which is generally what people intend, i.e., if the beginning of the request message path
matches the route path specifier than apply the route actions. If we wanted to exactly match the previous Ingress, we could use
`regex: /.*` instead. Virtual Services allow you to specify paths by: prefix, exact, and regular expression. You can
also see that instead of `backend:` with `serviceName` and `servicePort`, a Virtual Service has a `routeAction` that
delegates to a `single` `upstream`. Gloo upstreams are auto-discovered and can refer to Kubernetes Services AND
REST/gRPC function, cloud functions like AWS Lambda and Google Functions, and other external to Kubernetes services.

More details on Gloo at:

* [Routing with Gloo Function Gateway](https://medium.com/solo-io/routing-with-gloo-function-gateway-301765bb103e)
* [5 minutes with Gloo - The Anatomy of a VirtualService](https://medium.com/solo-io/5-minutes-with-gloo-the-anatomy-of-a-virtualservice-4deb4cfc558e)

Let's go back to our example, and update our Virtual Service to do the path rewrite we wanted, i.e., `/people` => `/users`

```yaml
apiVersion: gateway.solo.io/v1
kind: VirtualService
metadata:
  name: default
  namespace: gloo-system
spec:
  virtualHost:
    domains:
    - gloo.example.com
    routes:
    - matcher:
        prefix: /people
      routeAction:
        single:
          upstream:
            name: default-jsonplaceholder-jsonplaceholder-8080
            namespace: gloo-system
      routePlugins:
        prefixRewrite:
          prefixRewrite: /users
    - matcher:
        prefix: /
      routeAction:
        single:
          upstream:
            name: default-jsonplaceholder-jsonplaceholder-8080
            namespace: gloo-system
```

We've added a second route matcher, just like adding a second route path in an Ingress, and specified `prefix: /people`.
This will match all requests that start with `/people`, and all other calls to the `gloo.example.com` domain will be
handled by the other route matcher. We also added a `routePlugins` section that will rewrite the request path to `/users` such
that our service will now correctly handle our request. [Route Plugins](https://gloo.solo.io/user_guides/advanced_route_plugins/)
allow you to perform many operations on both the request to the upstream service and the response back from the upstream
service. Best shown with an example, so for our new `/people` route let's also transform the response to both add
a new header `x-test-phone` with a value from the response body, and let's transform the response body to return a
couple of fields: name, and the address/street and address/city.

{% highlight yaml %}
{% raw %}
piVersion: gateway.solo.io/v1
kind: VirtualService
metadata:
  creationTimestamp: "2019-04-08T21:43:45Z"
  generation: 1
  name: default
  namespace: gloo-system
  resourceVersion: "772"
  selfLink: /apis/gateway.solo.io/v1/namespaces/gloo-system/virtualservices/default
  uid: 6267ee31-5a47-11e9-bc30-867df7be8a8a
spec:
  virtualHost:
    domains:
    - gloo.example.com
    routes:
    - matcher:
        prefix: /people
      routeAction:
        single:
          upstream:
            name: default-jsonplaceholder-jsonplaceholder-8080
            namespace: gloo-system
      routePlugins:
        prefixRewrite:
          prefixRewrite: /users
        transformations:
          responseTransformation:
            transformation_template:
              body:
                text: '{ "name": "{{ name }}", "address":
                  { "street": "{{ address.street }}",
                    "city": "{{ address.city }}" } }'
              headers:
                x-test-phone:
                  text: '{{ phone }}'
    - matcher:
        prefix: /
      routeAction:
        single:
          upstream:
            name: default-jsonplaceholder-jsonplaceholder-8080
            namespace: gloo-system
{% endraw %}
{% endhighlight %}

Let's see what that looks like. My example GitHub repository already included
the full Gloo Virtual Service we just examined. We need to configure Gloo for
`gateway` which means adding another proxy to handle Virtual Services in
addition to Ingress resources. We'll use `draft up` to ensure our example is
fully deployed including the full Virtual Service, and then we'll call both
`/users/1` and `/people/1` to see the differences.

```shell
# Install Gloo and update example
glooctl install gateway
draft up

# Call service
curl --verbose --header "Host: gloo.example.com" \
  $(glooctl proxy url --name gateway-proxy)/users/1

curl --verbose --header "Host: gloo.example.com" \
  $(glooctl proxy url --name gateway-proxy)/people/1
```

<script id="asciicast-5DXyvz6bOmd6zTjayKJ4kjZlB" src="https://asciinema.org/a/5DXyvz6bOmd6zTjayKJ4kjZlB.js" data-speed="1.5" data-rows="32" data-cols="80" async></script>

<img src="/assets/mind_blown.png" alt="Mind Blown" style="width: 75px;" class="aligncenter"/>

Ok, well not that mind-blowing if you've used other L7 networking products or done other integration work, but still
pretty cool relative to standard Ingress objects. Gloo is using Inja Templates to process the JSON response body.
More details in the [Gloo documentation](https://gloo.solo.io/user_guides/advanced_route_plugins#transformation_template).

# Summary

In this article, we touched on some of the history and difficulties with the existing Kubernetes Ingress resources. Ingress
resources continue to play a role within Kubernetes deployments despite the many challenges that annotation-based
extensions have. Kubernetes Custom Resources (CRDs) was created to address some of those extension challenges and
can provide a cleaner way to extend Kubernetes as you saw in the Gloo Ingress and Gateway examples. I'm a big believer
in the potential of Envoy based solutions as are others in the Istio and Contour communities, and it will be exciting
to see how the Kubernetes community decides to evolve Ingress after they finally move the existing
resource spec to GA status.