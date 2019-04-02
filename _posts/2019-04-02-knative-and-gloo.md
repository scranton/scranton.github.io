---
layout: post
title: Automating your Services with Knative and Solo.io Gloo
date: 2019-04-02T19:28:31Z
description: Learn how Knative and Solo.io Gloo work together to support on demand code delivery in Kubernetes.
tags:
- Gloo
- Knative
- Kubernetes
---

[Knative](https://github.com/knative/) is talked about a great deal, especially around how its capabilities can
help provide more standard building blocks on top of Kubernetes for building microservices and serverless like services,
e.g., scale to zero, and scale on demand. Knative high level has three capability areas: building, serving, and eventing.
This post will provide some examples around Knative Build and Knative Serving with Solo.io Gloo.

Knative Serving initially included all of Istio only to use a small fraction of its capabilities around being a
Kubernetes cluster ingress. Recently the Knative team added [Solo.io](https://solo.io) [Gloo](https://gloo.solo.io) as
an alternative to Istio. More details are available in [Gloo, Knative and the future of Serverless](https://medium.com/solo-io/gloo-knative-and-the-future-of-serverless-902b89f15c2c)
and [Gloo, by Solo.io, is the first alternative to Istio on Knative](https://medium.com/solo-io/gloo-by-solo-io-is-the-first-alternative-to-istio-on-knative-324753586f3a).

This post shows a quick example of Knative Building, Knative Serving, and Gloo integration.

All of the Kubernetes Manifests are located in the following GitHub repository
<https://github.com/scranton/helloworld-knative>. I encourage you to fork that repository to help you try these examples
yourself.

# Setup

These instructions assume you are running on a clean, recent [minikube](https://kubernetes.io/docs/setup/minikube/)
install locally, and that you also have `kubectl` available locally.

## Install Gloo

On Mac or Linux, the quickest option is to use [Homebrew](https://bash.sh/). Full Gloo
install instructions at [Gloo documentation](https://gloo.solo.io/installation/install_glooctl/).

```shell
brew install glooctl
```

Then assuming you've got a running `minikube`, and `kubectl` set up against that `minikube` instance, i.e., `kubectl
config current-context` returns `minikube`, run the following to install Gloo with Knative Serving.

```shell
glooctl install knative
```

# Deploy existing example image

I've already built this example, and have hosted the image publicly in my [Docker Hub repository](https://hub.docker.com/r/scottcranton/helloworld-go).
To use Knative to serve up this existing image, you need to do the following command.

```shell
kubectl apply --filename service.yaml
```

Verify the domain URL for the service. It should be `helloworld-go.default.example.com`.

```shell
kubectl get kservice helloworld-go \
  --namespace default \
  --output=custom-columns=NAME:.metadata.name,DOMAIN:.status.domain
```

And call the service. Note: the `curl --connect-to` option is only required when calling locally against minikube as
that option will add the correct host and sni headers to the request, and send the request to the host and port pair
returned from `glooctl proxy address`.

```shell
curl --connect-to helloworld-go.default.example.com:80:$(glooctl proxy address --name clusteringress-proxy) http://helloworld-go.default.example.com
```

To clean up, delete the resources.

```shell
kubectl delete --filename service.yaml
```

# Build locally, and deploy using Knative Serving

Run `docker build` with your Docker Hub username.

```shell
docker build -t ${DOCKER_USERNAME}/helloworld-go .
docker push ${DOCKER_USERNAME}/helloworld-go
```

Deploy the service. Again, make sure you updated username in `service.yaml` file, i.e., replace image reference
`docker.io/scottcranton/helloworld-go` with your Docker Hub username.

```shell
kubectl apply --filename service.yaml
```

Verify domain URL for service. Should be `helloworld-go.default.example.com`.

```shell
kubectl get kservice helloworld-go \
  --namespace default \
  --output=custom-columns=NAME:.metadata.name,DOMAIN:.status.domain
```

And test your service.

```shell
curl --connect-to helloworld-go.default.example.com:80:$(glooctl proxy address --name clusteringress-proxy) http://helloworld-go.default.example.com
```

To clean up, delete the resources.

```shell
kubectl delete --filename service.yaml
```

# Build using Knative Build, and deploy using Knative Serving

To install Knative Build, do the following. I'm using the `kaniko` build template, so you'll also need to install that
as well.

```shell
kubectl apply \
  --filename https://github.com/knative/build/releases/download/v0.4.0/build.yaml

kubectl apply \
  --filename https://raw.githubusercontent.com/knative/build-templates/master/kaniko/kaniko.yaml
```

To verify the Knative Build install, do the following.

```shell
kubectl get pods --namespace knative-build
```

I'd encourage forking my example GitHub repository <https://github.com/scranton/helloworld-knative>, so you can push
code changes and see them in your environment.

Create a Kubernetes secret for your Docker Hub account that will allow Knative build to push your image. You also need to annotate the secret to indicate it's for Docker. More details in [Guiding credential selection](https://www.knative.dev/docs/build/auth/#guiding-credential-selection).

```shell
kubectl create secret generic basic-user-pass \
  --type="kubernetes.io/basic-auth" \
  --from-literal=username=${DOCKER_USERNAME} \
  --from-literal=password=${DOCKER_PASSWORD}

kubectl annotate secret basic-user-pass \
  build.knative.dev/docker-0=https://index.docker.io/v1/
```

It should result in a secret like the following.

```shell
kubectl describe secret basic-user-pass

Name:         basic-user-pass
Namespace:    default
Labels:       <none>
Annotations:  build.knative.dev/docker-0: https://index.docker.io/v1/

Type:  kubernetes.io/basic-auth

Data
====
username:  12 bytes
password:  24 bytes
```

Verify that `serviceaccount.yaml` references your secret.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: build-bot
secrets:
  - name: basic-user-pass
```

Update `service-build.yaml` with your GitHub and Docker usernames. This manifest will use Knative Build to create an
image using the `kaniko-build` build template and deploy the service using Knative Serving with Gloo.

```yaml
apiVersion: serving.knative.dev/v1alpha1
kind: Service
metadata:
  name: helloworld-go
  namespace: default
spec:
  runLatest:
    configuration:
      build:
        apiVersion: build.knative.dev/v1alpha1
        kind: Build
        metadata:
          name: kaniko-build
        spec:
          serviceAccountName: build-bot
          source:
            git:
              url: https://github.com/{ GitHub username }/helloworld-knative
              revision: master
          template:
            name: kaniko
            arguments:
              - name: IMAGE
                value: docker.io/{ Docker Hub username }/helloworld-go
          timeout: 10m
      revisionTemplate:
        spec:
          container:
            image: docker.io/{ Docker Hub username }/helloworld-go
            imagePullPolicy: Always
            env:
              - name: TARGET
                value: "Go Sample v1"
```

To deploy, apply the manifests.

```shell
kubectl apply \
  --filename serviceaccount.yaml \
  --filename service-build.yaml
```

Then you can watch the build and deployment happening.

```shell
kubectl get pods --watch
```

Once you see all the `helloworld-go-0000x-deployment-....` pods are ready, then you can Ctrl+C to escape the watch, and
then test your deployment.

Verify the domain URL for service. Should be `helloworld-go.default.example.com`.

```shell
kubectl get kservice helloworld-go \
  --namespace default \
  --output=custom-columns=NAME:.metadata.name,DOMAIN:.status.domain
```

And test your service.

```shell
curl --connect-to helloworld-go.default.example.com:80:$(glooctl proxy address --name clusteringress-proxy) http://helloworld-go.default.example.com
```

## Cleanup

```shell
kubectl delete \
  --filename serviceaccount.yaml \
  --filename service-build.yaml

kubectl delete secret basic-user-pass
```

# Summary

Hopefully, this post gave you a taste for how Gloo and Knative can work together to provide you with a way to build and
deploy your services on demand into Kubernetes.

# See Also

* <https://github.com/knative/docs/blob/master/install/getting-started-knative-app.md>
* <https://github.com/knative/docs>
* <https://gloo.solo.io/getting_started/kubernetes/gloo_with_knative/>
