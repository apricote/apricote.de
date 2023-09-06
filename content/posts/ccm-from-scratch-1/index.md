---
author: "Julian Tölle"
title: "Building a Kubernetes cloud-controller-manager"
date: "2023-09-06"
description: |
  Tutorial on building a Kubernetes cloud-controller-manager from Scratch.
  In this post we outline the background and scope of the tutorial and setup the boilerplate.
tags: ["kubernetes"]
ShowToc: true
cover:
  image: cover.webp
  responsiveImages: true
  caption: dall-e with prompt "ship stearing towards clouds, a captain behind the steering wheel, cartoon, wide angle"
---

### Disclaimers

#### Work

I [work for Hetzner Cloud](../../work.md) and this project was done during working hours as part of the _Lab Days_.
During Lab Days, employees can choose to work on topics unrelated to their work.

#### Production Readiness

The code written in this series is not ready for production.
You should not actually use the code I provide for your Kubernetes Cluster on Hetzner Cloud.
Please use the official [`hcloud-cloud-controller-manager`](https://github.com/hetznercloud/hcloud-cloud-controller-manager) instead.

## Scope

The series tries to showcase the steps needed to build a first iteration of a Kubernetes Cloud Controller Manager.
It can be adapted to any Cloud Provider and may be your guide into better Kubernetes-compatibility for your own Infrastructure-as-a-Service Cloud.

## Introduction

### What is a Cloud Controller Manager?

> Source: https://kubernetes.io/docs/concepts/architecture/cloud-controller/

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

> Source: https://hetzner.com/cloud

- IaaS with: Servers, Load Balancers, Private Networks

## (Part 1) Setting up the Boilerplate

> Goal: Get a compilable binary running k/cloud-provider with no controllers yet

### Dependencies

As all ~~good~~ software does, we [stand on the shoulders of giants](https://xkcd.com/2347/). Let us review the two main dependencies that our code will use:

#### `k8s.io/cloud-provider`

> Repository: https://github.com/kubernetes/cloud-provider

Official Library by Kubernetes Project to implement cloud-controller-managers.
Historically part of `kubelet` but extracted a long time ago to build out-of-tree CCMs.
Handles communication with Kubernetes, CCM implementors only need to implement a narrowly-scoped interface for each functionality.

> This series will use Kubernetes `v1.28` throughout.
> If you use a different version, you may need to adjust your code.
> The `k8s.io/cloud-provider` release series associated with Kubernetes `v1.28` is `v0.28.x`.

#### `github.com/hetznercloud/hcloud-go`

> Repository: https://github.com/hetznercloud/hcloud-go

Official Go SDK by Hetzner Cloud, currently maintained by my team.
Exposes all functionality of the Hetzner Cloud API in (mostly) idiomatic Go.

### Repo initialization

```shell
mkdir ccm-from-scratch
cd ccm-from-scratch
git init
git commit --allow-empty -m "feat: init repo"

# Optional
git remote add git@github.com:apricote/ccm-from-scratch.git
git push
```

### `main.go`

> Source: https://github.com/kubernetes/cloud-provider/tree/v0.28.1/sample


## (Part 2) Development Environment with Terraform, k3s and Skaffold

> Goal: Get an environment to deploy our binary to, to test any controllers we implement

## (Part 3) Controller: Node

> Goal: Implement the Node controller with all features, deploy & verify

## (Part 4) Controller: Service (Load Balancers)

> Goal: Setup Load Balancers so traffic reaches our cluster, set up an ingress in the dev env that uses the LB

## (Part 5) Controller: Route

> Goal: Use the Hetzner Cloud Networks instead of Overlay Networks for Pod CIDR connectivity