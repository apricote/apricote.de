---
author: "Julian Tölle"
title: "Building a Kubernetes cloud-controller-manager"
date: "2023-09-06"
summary: |
  A tutorial series to showcase the steps needed to build a Kubernetes Cloud Controller Manager.
  Intended to be your guide into better Kubernetes-compatibility for your own Infrastructure-as-a-Service Cloud.
description: |
  This tutorial series tries to showcase the steps needed to build a first iteration of a Kubernetes Cloud Controller Manager.
  It can be adapted to any Cloud Provider and may be your guide into better Kubernetes-compatibility for your own Infrastructure-as-a-Service Cloud.
tags: ["kubernetes"]
ShowBreadCrumbs: true
cover:
  image: cover.webp
  responsiveImages: true
  caption: dall-e with prompt "ship stearing towards clouds, a captain behind the steering wheel, cartoon, wide angle"
---

![dall-e with prompt 'ship stearing towards clouds, a captain behind the steering wheel, cartoon, wide angle'](./cover.webp)

{{< collapse summary="Disclaimers" openByDefault=true >}}
##### Employer

I [work for Hetzner Cloud]({{< ref "/work" >}}) and this project was done during working hours as part of the _Lab Days_.
During Lab Days, employees can choose to work on topics unrelated to their work.

##### Production Readiness

The code written in this series is not ready for production.
You should not actually use the code I provide for your Kubernetes Cluster on Hetzner Cloud.
Please use the official [`hcloud-cloud-controller-manager`](https://github.com/hetznercloud/hcloud-cloud-controller-manager) instead.
{{</ collapse >}}

## Introduction

### What is a Cloud Controller Manager?

[Source](https://kubernetes.io/docs/concepts/architecture/cloud-controller/)

- Controllers: Loop that reconciles reality with the desired state 
- Controller Manager: Binary that runs a bunch of controllers
- Cloud Controller Manager: Part of the Kubernetes Control Plane, responsible for exposing Cloud Provider functionality inside the Kubernetes cluster.
- Controllers in CCMs:
  - Node Metadata
    - Match Kubernetes `Node` to Cloud API Server (/Instance/Droplet/...) and set `spec.providerID`
    - Label Kubernetes `Nodes` with metadata of the Node, including the [Instance Type](https://kubernetes.io/docs/reference/labels-annotations-taints/#nodekubernetesioinstance-type), Topology ([Region](https://kubernetes.io/docs/reference/labels-annotations-taints/#topologykubernetesioregion), [Zone](https://kubernetes.io/docs/reference/labels-annotations-taints/#topologykubernetesiozone))
    - Figure out the Network addresses of the `Node` and set them in `status.addresses`.
  - Node Lifecycle
    - The `kubelet` is usually responsible for creating the `Node` object in Kubernetes on its first startup.
    - This does not work well for the removal of the node, as the `kubelet` might not get a chance to do this. Instead, the Node Lifecycle controller regularly checks if the Node was deleted in the Cloud API, and if it was it also deletes the `Node` object in Kubernetes.
  - Service
    - Watches `Service` objects with `type: LoadBalancer`. Creates Cloud Load Balancers and configures them according to the `Service` and `Node` objects
  - Route
    - In Kubernetes, every Pod gets an IP address. This IP address needs to be available from every other pods network (by default). This is usually implemented as an Overlay Network through your CNI like Calico or Cilium.
    - If you are already using some kind of Private Networking functionality of your Cloud Provider, then you can use this instead to get rid of the additional Overlay Network and let the Cloud Provider handle the connectivity.
    - This is implemented by configuring Routes in the Private Network Routing Table that send all traffic to the `Nodes` `spec.podCIDR` to the `Nodes` private IP.

### What is the Hetzner Cloud?

[Source](https://hetzner.com/cloud)

- IaaS with: Servers, Load Balancers, Private Networks

### Parts

This project got way longer than I expected, so I split everything up into 5 parts:
