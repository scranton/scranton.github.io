---
layout: post
title: Kubernetes Ingress Control using Gloo
date: 2019-03-28T16:37:03Z
description: Setting up Gloo to be your Kubernetes Ingress controller
tags:
- Gloo
- Ingress
- Kubernetes
---

Kubernetes is great and makes it easier to create and manage highly distributed applications. A challenge then is how do you share your great Kubernetes hosted applications with the rest of the world. Many lean towards [Kubernetes Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/) objects and this article will show you how to use the open source Solo.io [Gloo](https://gloo.solo.io) to fill this need.

![Gloo as Ingress](/assets/gloo_as_ingress.png)

[Gloo is a function gateway](https://medium.com/solo-io/announcing-gloo-the-function-gateway-3f0860ef6600) that gives users a number of benefits including sophisticated function level routing, and deep service discovery with introspection of OpenAPI (Swagger) definitions, gRPC reflection, Lambda discovery and more. Gloo can act as an Ingress Controller, that is, by routing Kubernetes external traffic to Kubernetes cluster hosted services based on the path routing rules defined in an Ingress Object. I’m a big believer in showing technology through examples, so let’s quickly run through an example to show you what's possible.

# Prerequisites

This example assumes your running on a local [minikube](https://kubernetes.io/docs/setup/minikube/) instance, and that you also have [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/) also running. You can run this same example on your favorite cloud provider managed Kubernetes cluster, you’d just need to make a few tweaks. You’ll also need [Gloo](https://gloo.solo.io) installed. Let’s use [Homebrew](https://brew.sh/) to set all of this up for us.

```shell
brew update
brew cask install minikube
brew install kubectl glooctl curl

minikube start
kubectl config current-context
glooctl install ingress
```

It will take a few minutes to download and install everything to your local machine, and get everything started. Assuming it all went well, the command (`kubectl`) should have returned `minikube` meaning that your local minikube cluster is running and your Kubernetes cluster config is correctly set up for minikube.

One more thing before we dive into Ingress objects, let’s set up an example service deployed on Kubernetes that we can reference.

```shell
kubectl apply \
  --filename https://raw.githubusercontent.com/solo-io/gloo/master/example/petstore/petstore.yaml
```

# Setting up an Ingress to our example Petstore

Let’s set up a basic Ingress object that routes all HTTP traffic to our petstore service. To make this a little more interesting and challenging, and who doesn’t like a good tech challenge, let’s also configure a host domain, which will require a little extra `curl` magic to call correctly on our local Kubernetes cluster. The following Ingress definition will route all requests to `http://gloo.example.com` to our petstore services listening on port 8080 within our cluster. The petstore service provides some REST functions listening on the query path `/api/pets` that will return JSON for the inventory of pets in our (small) store.

If you are trying this example in a public cloud Kubernetes instance, you’ll most likely need to configure a Cloud Load Balancer. Make sure you configure that Load Balancer for the `service/ingress-proxy` running in the `gloo-system` namespace.

The important details are:

* Annotation `kubernetes.io/ingress.class: gloo` which is the standard way to mark an Ingress as handled by a specific Ingress controller, i.e., Gloo. This requirement will go away soon as we add ability for Gloo to be the cluster default Ingress controller.
* Path wildcard `/.*` to indicate to route all traffic to our petstore service.

```yaml
cat <<EOF | kubectl apply --filename -
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
 name: petstore-ingress
 annotations:
    kubernetes.io/ingress.class: gloo
spec:
  rules:
  - host: gloo.example.com
    http:
      paths:
      - path: /.*
        backend:
          serviceName: petstore
          servicePort: 8080
EOF
```

We can validate that Kubernetes created our Ingress correctly by the following command

```shell
kubectl get ingress petstore-ingress

NAME               HOSTS              ADDRESS   PORTS   AGE
petstore-ingress   gloo.example.com             80      14h
```

To test we’ll use `curl` to call our local cluster. Like I said earlier, by defining a `host: gloo.example.com` in our Ingress, we need to do a little more to call this without doing things with DNS or our local /etc/hosts file. I’m going to use the recent `curl --connect-to` options, and you can read more about that at the [curl man pages](https://curl.haxx.se/docs/manpage.html#--connect-to).

The glooctl command-line tool helps us get the local host IP and port for the proxy with the `glooctl proxy url --name <ingress name>` command. It returns a full URL with protocol prefix that we’ll need to strip off to use with curl. The following two commands will do a little bit of command line magic to make this example work on your local machine.

```shell
export PROXY_HOST_PORT=$(glooctl proxy url --name ingress-proxy --port http | sed -n -e 's/^.*:\/\///p')

curl --connect-to gloo.example.com:80:${PROXY_HOST_PORT} http://gloo.example.com/api/pets
```

Which should return the following JSON

```json
[{"id":1,"name":"Dog","status":"available"},{"id":2,"name":"Cat","status":"pending"}]
```

# TLS Configuration

These days, most want to use TLS to secure your communications. Gloo Ingress can act as a TLS terminator, and we’ll quickly run through what that set up would look like.

Any Kubernetes Ingress doing TLS will need a Kubernetes TLS secret created, so let’s create a self-signed certificate we can use for our example `gloo.example.com` domain. The following two commands will create a certificate, and create a TLS secret named `my-tls-secret` in minikube.

```shell
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout my_key.key -out my_cert.cert -subj "/CN=gloo.example.com/O=gloo.example.com"

kubectl create secret tls my-tls-secret --key my_key.key --cert my_cert.cert
```

Now let’s update our Ingress object with the needed TLS configuration. Important that the TLS host and the rules host match, and the `secretName` matches the name of the Kubernetes secret deployed previously.

```yaml
cat <<EOF | kubectl apply --filename -
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: petstore-ingress
  annotations:
    kubernetes.io/ingress.class: gloo
spec:
  tls:
  - hosts:
    - gloo.example.com
    secretName: my-tls-secret
  rules:
  - host: gloo.example.com
    http:
      paths:
      - path: /.*
        backend:
          serviceName: petstore
          servicePort: 8080
EOF
```

If all went well we should have changed our petstore to now be listening to `https://gloo.example.com`. Let’s try it, again using our curl magic, which we need to both resolve the host and port as well as to validate our certificate. Notice that we’re asking glooctl for `--port https` this time, and we’re curling `https://gloo.example.com` on port 443.

```shell
export PROXY_HOST_PORT=$(glooctl proxy url --name ingress-proxy --port https | sed -n -e 's/^.*:\/\///p')

curl --cacert my_cert.cert --connect-to gloo.example.com:443:${PROXY_HOST_PORT} https://gloo.example.com/api/pets
```

# Next Steps

This was a quick tour of how Gloo can act as your Kubernetes Ingress controller making very minimal changes to your existing Kubernetes manifests. Please try it out and let us know what you think at our [community Slack channel](https://slack.solo.io/).

If you're interested in powering up you Gloo superpowers, try Gloo in gateway mode `glooctl install gateway`, which unlocks a set of Kubernetes CRDs (Custom Resources) that give you a more standard and powerful way of doing far more advanced traffic shifting, rate limiting, and more without the annotation smell in your Kubernetes cluster. Check out these other articles for more details on Gloo’s extra powers.

* [Routing with Gloo Function Gateway]({{ site.baseurl }}{% post_url 2019-02-19-function-routing-with-gloo %})
* [Canary Deployments with Gloo Function Gateway]({{ site.baseurl }}{% post_url 2019-02-19-canary-deployments-with-solo %})
* [Canary Deployments with Gloo Function Gateway using Weighted Destinations]({{ site.baseurl }}{% post_url 2019-03-12-canary-deployments-with-weighted-routes %})
