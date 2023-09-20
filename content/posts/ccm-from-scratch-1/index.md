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

I like to make an empty initial commit, makes the occasional
rebase of the first few commits so much easier.
git commit --allow-empty -m "feat: init repo"

# Optional
git remote add git@github.com:apricote/ccm-from-scratch.git
git push
```

### `main.go`

> Source: https://github.com/kubernetes/cloud-provider/tree/v0.28.2/sample

Thankfully, `k8s.io/cloud-provider` provides an example entrypoint to get us started with out CCM. Lets review the code to make sure we understand what is happening:

```go
package main

// [Imports]

// The main function is called as the entrypoint of our Go binary.
func main() {
    // k8s.io/cloud-provider has an elaborate way to read the configuration from flags.
    // I found this very tedious to debug, but at least we do not have to do this ourselves.
    ccmOptions, err := options.NewCloudControllerManagerOptions()
    if err != nil {
        klog.Fatalf("unable to initialize command options: %v", err)
    }
    
    // Can be used to add custom flags to the binary, we dont need this.
    fss := cliflag.NamedFlagSets{}
    
    // This initializes the CLI command. The arguments are interesting, so lets take a closer look:
    command := app.NewCloudControllerManagerCommand(
        // The options we initialized earlier.
        ccmOptions,
        // This is a custom function that needs to return a [cloudprovider.Interface],
        // we will get to this soon.
        cloudInitializer,
        // This defines which controllers are started, if wanted,
        // one can add additional controller loops heroe
        app.DefaultInitFuncConstructors,
        // Kubernetes v1.28 renamed the controllers to more sensible names, this
        // map makes it so that old command-line arguments (--controllers) still work
        names.CCMControllerAliases(),
        // Our optional additional flags
        fss,
        // A [<-chan struct{}] that can be used to shut down the CCM on demand,
        // we do not need this. 
        wait.NeverStop,
    )
    // Actually run the command to completion.
    code := cli.Run(command)
    os.Exit(code)
}
```

Now, this does not compile right now. We use the undefined `cloudInitalizer` method. The method signature we need to implement is `(*config.CompletedConfig) cloudprovider.Interface`. The sample code includes this method, but I found it overly complex for our small CCM, so we will implement it ourselves. Lets take a closer look at the interface we need to return in the next section.




### `cloudprovider.Interface`

> Source: https://github.com/kubernetes/cloud-provider/blob/v0.28.2/cloud.go#L42-L69

This is the entrypoint into the functionality we can (and will!) implement for our CCM. The interface includes an initializer, two cloud-provider metadata methods and a bunch of Getter functions to other interfaces that implement the actual functionality.

Before we can write the `cloudInitializer` method from above, lets prepare a struct that implements the interface:

```go
package ccm
// ccm/cloud.go

import (
	cloudprovider "k8s.io/cloud-provider"
)

type CloudProvider struct {}

// Let's try to assign our struct to a var of the interface we try to implement.
// This way we get nice feedback from our IDE if we break the contract.
var _ cloudprovider.Interface = CloudProvider{}


// Can be used to setup our own controllers or goroutines that need to talk to
//Kubernetes. Not needed for our implementation, so we will leave it empty.
func (c CloudProvider) Initialize(clientBuilder cloudprovider.ControllerClientBuilder, stop <-chan struct{}) {}

// An arbitrary name for our provider. We will use "ccm-from-scratch" for this example.
func (c CloudProvider) ProviderName() string { return "ccm-from-scratch" }

// I have not yet figured out what Cluster ID is actually supposed to do. We can disable it.
func (c CloudProvider) HasClusterID() bool { return false }

// The following methods all expose additional interfaces. This allows us to
// modularly choose what we want to support. As long as we return "false" as the
// second argument, the associated functionality is disabled for our CCM.
// We will get to implementing [InstancesV2], [LoadBalancer] & [Routes] in later parts of this series.
func (c CloudProvider) LoadBalancer() (cloudprovider.LoadBalancer, bool) { return nil, false }
func (c CloudProvider) Instances()    (cloudprovider.Instances,    bool) { return nil, false }
func (c CloudProvider) InstancesV2()  (cloudprovider.InstancesV2,  bool) { return nil, false }
func (c CloudProvider) Zones()        (cloudprovider.Zones,        bool) { return nil, false }
func (c CloudProvider) Clusters()     (cloudprovider.Clusters,     bool) { return nil, false }
func (c CloudProvider) Routes()       (cloudprovider.Routes,       bool) { return nil, false }
```

Now that we have a struct implementing the interface, lets create our `cloudInitializer` method. We will actually do this in the same file as our struct:

```go
// ccm/cloud.go

// Just create a new struct for now, we will add some more stuff to this later.
func NewCloudProvider(_ *config.CompletedConfig) cloudprovider.Interface {
	return CloudProvider{}
}
```

And now we can pass this method in our `main()`:

```go
package main
// main.go
import (
	// Existing imports
	
	"github.com/apricote/ccm-from-scratch/ccm"
)

func main() {
	// [ other code ]
    // This initializes the CLI command. The arguments are interesting, so lets take a closer look:
	command := app.NewCloudControllerManagerCommand(
        ccmOptions,
		
		// This was previously "cloudInitializer"
        ccm.NewCloudProvider,
		//
		
        app.DefaultInitFuncConstructors,
        names.CCMControllerAliases(),
        fss,
        wait.NeverStop,
    )
}
```

### Starting our CCM

We now have all the pieces in place to try to run our program.

For now, we can use `go run` to test our binary. The next part of this series will implement a proper development environment.

```shell
$ go run . 
TODO figure out actual arguments necessary and output
```

## (Part 2, Sidequest) Development Environment with Terraform, k3s and Skaffold

> Goal: Get an environment to deploy our binary to, to test any controllers we implement

- If this topic does not interest you, you can safely skip ahead! While we use what we built in the following parts, you can still understand what is going on without reading this article.

## (Part 3) Controller: Node

> Goal: Implement the Node controller with all features, deploy & verify

## (Part 4) Controller: Service (Load Balancers)

> Goal: Setup Load Balancers so traffic reaches our cluster, set up an ingress in the dev env that uses the LB

## (Part 5) Controller: Route

> Goal: Use the Hetzner Cloud Networks instead of Overlay Networks for Pod CIDR connectivity