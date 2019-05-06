---
layout: post
title: SuperGloo to the Rescue! Making it easier to write applications for Service Meshes.
date: 2019-05-06T17:51:24Z
description: How SuperGloo abstracts Service Meshes to allow applications to run on one or more Service Meshes like a superhero.
tags:
- SuperGloo
- Service Mesh
- Kubernetes
---

![SuperGloo](/assets/supergloo.png "SuperGloo"){:style="width:50%;"}

<!--
<figure class="aligncenter">
    <img src="https://images.unsplash.com/photo-1531907700752-62799b2a3e84?ixlib=rb-1.2.1&auto=format&fit=crop&w=1350&q=80"/>
    <figcaption>Photo by <a href="https://unsplash.com/@tkristin" target="_blank">TK Hammonds</a>.</figcaption>
</figure>
-->

# Overview

SuperGloo is a new open source project that helps with managing service meshes at scale. SuperGloo provides
an opinionated abstraction layer that simplifies the installation, management, and operations of one or more service
meshes like [Istio](https://istio.io/), [AWS App Mesh](https://aws.amazon.com/app-mesh/), [Linkerd](https://linkerd.io/)
and [HashiCorp Consul](https://www.consul.io/). Supports on-site, in the cloud or any combination you need.

![SuperGloo Architecture](/assets/supergloo_architecture.png "SuperGloo Architecture"){:style="padding-top: 50px; padding-bottom: 50px;"}

There are a growing number of articles on why SuperGloo, like [Christian Posta's](https://blog.christianposta.com/)
["Solo.io Streamlines Service Mesh and Serverless Adoption for Enterprises in Google Cloud"](https://medium.com/solo-io/solo-io-streamlines-service-mesh-and-serverless-adoption-for-enterprises-in-google-cloud-c93b8aa04d81).
This article is going to focus on how SuperGloo can help software packages like [Weaveworks Flagger](https://github.com/weaveworks/flagger)
work on multiple Services Meshes like Istio and AWS App Mesh that both support traffic shifting.

Flagger is a cool open source project that automates the promotion of Canary deployments of your Kubernetes services.
You associate a Flagger Canary Kubernetes custom resource (CRD) with your deployment and Flagger follows your defined
rules for helping roll out a new version. It detects when a new version of your service has deployed, instantiating your
new version in parallel to your existing version, slowly shifting request traffic between the two versions, and using
your defined Prometheus metric health checks to determine if Flagger should continue moving more traffic to the new
version or roll back to the old version. Since a Canary CRD is a YAML file, this provides you a
declarative way to ensure that all of your service upgrades follow your prescribed sophisticated rollout strategy and
complements GitOps pipelines used in [Weave Flux](https://github.com/weaveworks/flux) and [JenkinsX](https://jenkins-x.io/).

More information on what Canary deployments and Traffic Shifting are is in the following posts. [Gloo](https://gloo.solo.io)
uses the same underlying data plane technology - [Envoy](https://www.envoyproxy.io/) - as Istio to provide traffic shifting capabilities used by
Flagger and Knative. Gloo is an API/Function gateway and not a full Service Mesh, so Gloo can be used in use cases that
do not require all of the power, and weight, of full service mesh implementations.

* [Canary Deployments with Gloo Function Gateway using Weighted Destinations]({{ site.baseurl }}{% post_url 2019-03-12-canary-deployments-with-weighted-routes %})
* [Automating your Services with Knative and Solo.io Gloo]({{ site.baseurl }}{% post_url 2019-04-02-knative-and-gloo %})
* [Jenkins X Serverless Apps](https://jenkins-x.io/developing/knative/)

This article quickly runs through setting up the Flagger `podinfo` example application on SuperGloo with Istio so
you all can see what's involved and try yourselves if you like.

## Install Kubernetes and Helm

The first step on our journey is to get a basic local Kubernetes cluster running. My friend [Christian Hernandez](https://twitter.com/christianh814)
clued me in on [kind (Kubernetes IN Docker)](https://kind.sigs.k8s.io/) from the Kubernetes sig-testing team. It's a fast,
lightweight way to spin up/down a local cluster assuming you have a locally running copy of Docker, e.g., Docker Desktop.
This example works equally well in `minikube` if you prefer. The following code does the basics you need for most
Kubernetes clusters.

* Creates a `kind` cluster with one control plane node and one worker node
* Configures the `KUBECONFIG` as `kind` creates a separate kubeconfig file for each `kind` cluster
* Installs Helm and Tiller with a service account for Tiller

{% gist 798ced2125eedb4a93c2df65526a9359 create_cluster.sh %}

## Install SuperGloo and SuperGloo installs and configures Istio

Here's where the magic happens so let's spend a little time teasing out all the things that are happening due to these
few lines of code.

{% gist 798ced2125eedb4a93c2df65526a9359 install_supergloo.sh %}

The first command `supergloo init` installs SuperGloo into your Kubernetes cluster that is equivalent to using Helm
to install SuperGloo.

{% gist 798ced2125eedb4a93c2df65526a9359 install_supergloo_helm.sh %}

The second command `kubectl --namespace supergloo-system rollout status deployment/supergloo --watch=true` is a hack
to wait till the SuperGloo deployment is fully deployed and running. It's similar to using the `--wait` option on a
Helm install.

The `supergloo install istio ...` command declares a custom resource and the SuperGloo controller installs and
configures Istio as declared. In this case, we are installing Istio version 1.0.6 with Istio's Prometheus installation and
with Istio deploying sidecars in all pods within namespaces with the label `istio-injection=enabled`, i.e., [Istio's
default behavior for auto-injecting sidecars](kubectl --namespace supergloo-system rollout status deployment/supergloo --watch=true
). This imperative `supergloo install istio` command creates the following manifest that you could `kubectl apply`
if you prefer. Refer to the full [`Install`](https://supergloo.solo.io/v1/github.com/solo-io/supergloo/api/v1/install.proto.sk#istioinstall)
specification for more details.

{% gist 798ced2125eedb4a93c2df65526a9359 supergloo-install.yaml %}

## Install Flagger

The following install Flagger and its dependent parts. The following is a quick summary of installing Flagger. More
details at [Flagger Doc Site](https://docs.flagger.app/).

1. Add a reference to Flagger helm repo
2. Wait for Tiller to be fully running. Only an issue for quick scripts that create Kubernetes clusters from scratch
3. Create a cluster role binding that allows Flagger to modify SuperGloo/Istio resources
4. Install core Flagger referencing Istio's provided Prometheus and telling Flagger that SuperGloo is the mesh controller
5. Install Flagger's Grafana dashboards which are not used as part of this demo
6. Install Flagger's LoadTester which can help generate test traffic during a Canary deployment if there is not enough
user traffic

{% gist 798ced2125eedb4a93c2df65526a9359 install_flagger.sh %}

## Install Flagger example application

The example application [`podinfo`](https://github.com/stefanprodan/k8s-podinfo) is a simple golang web application.
It is instrumented with Prometheus so we can tell if it's performing well (or not) that helps with our Canary
deployment to validate that the new version is handling incoming traffic. The example application also has hooks
to allow you to generate faults if you want to explicitly fail a deployment to see how the Flagger Canary handles that
situation. Full details on the options around the Flagger example application are [here](https://docs.flagger.app/usage/progressive-delivery).
The following is the summary of installation steps.

1. Install a `test` namespace, the example Kubernetes Deployment manifest and an (optional) horizontal pod autoscaler
2. Deploy the Canary policy for the example application. More details on that in a moment
3. Wait for the Canary controller to report that it's fully ready, which means Istio and Flagger are fully deployed and
running

{% gist 798ced2125eedb4a93c2df65526a9359 install_example.sh %}

The Canary manifest has a target reference that associates it with the podinfo deployment. The Canary analysis says that
for every `interval` (1 minute) Flagger increment by `stepWeight` (10%) more request traffic to the new version up to
`maxWeight` (50%) as long as the `metrics` stay within the defined healthy ranges. If more than `threshold` (5) health
checks fail, then rollback to 100% of traffic to the old version and delete the new version deployment. There is also
an optional section to allow the Flagger loadtester to generate additional traffic to help validate the new Canary
version, i.e., hard to know if the new version works if it has not handled any requests.

{% gist 798ced2125eedb4a93c2df65526a9359 podinfo-canary.yaml %}

## Deploy a new image version and watch the Canary deployment

First, we check on the currently deployed image version and print that out to help us verify if the test updates
the service like we expect; should be `quay.io/stefanprodan/podinfo:1.4.0`. Then to help make the changes more visible,
we trigger a background process to update the image version to `quay.io/stefanprodan/podinfo:1.4.1` after a five second
delay. We then loop and print out the change events for podinfo to see the traffic weight changes until the Canary
reports Success. You'd need to change this loop if you want to try introducing errors to see the Canary rollback. Lastly,
we'll print out the image version which should be `quay.io/stefanprodan/podinfo:1.4.1` is everything succeeded.

{% gist 798ced2125eedb4a93c2df65526a9359 update_version.sh %}

## Cleanup Kubernetes

The final step is to clean up the Kubernetes cluster, which in our case means to delete the kind cluster by running
`kind delete cluster` and unsetting the `KUBECONFIG` environment variable.

## Everything

Here's an [Asciinema](https://asciinema.org/) screen recording of the whole example script running, and afterward you
can see the whole script if you wanted to try yourself. The Asciinema recorder speeds up any long running commands,
that is, if a command takes more than two seconds to execute the playback delays up to two seconds. This speedup reduces
the run time from 15+ minutes to around two minutes.

<script id="asciicast-hUwE6wzhmEkw2KIK74dtNiIfd" src="https://asciinema.org/a/hUwE6wzhmEkw2KIK74dtNiIfd.js" data-rows="32" data-cols="120" async></script>

{% gist 798ced2125eedb4a93c2df65526a9359 run_flagger_demo.sh %}

# Summary

Hopefully, this example gave you a taste of how SuperGloo supports a Canary deployment engine like Flagger. Before
SuperGloo you'd either need to learn how to install all of Istio yourself or be constrained to using a managed Istio or
App Mesh installation from GKE or AWS respectively. Those are good managed offerings, but they do limit your choices
to the versions and configurations they currently support.

SuperGloo provides a great abstraction and management layer to help applications leverage one or more Service Meshes
without needing to get deep into the weeds of the huge API surface area of any one of the meshes like Istio or
App Mesh. SuperGloo makes it easy for applications to use just what they need of the underlying meshes. That helps
with adoption on Service Meshes based on the feedback I've heard, that is, many are currently experimenting with
Istio or App Mesh or Linkerd for just on capability, typically traffic shifting or mutual TLS, and they're finding it
difficult to manage and configure the whole mesh even though they aren't using those other capabilities. SuperGloo to
the rescue to help make it easier to use just the parts of Service Meshes that add value today, and allow you to add
more as you need it including mixing and matching multiple service meshes easily to get the biggest return on your
investment.

I strongly encourage you to learn more yourself as its fun to learn new technology, especially tech that helps you solve
complex challenges and accelerates your ability to deploy larger and more sophisticated systems.

* [SuperGloo](httsp://supergloo.solo.io)
* [Flagger](https://docs.flagger.app/)
* [Istio](https://istio.io)
