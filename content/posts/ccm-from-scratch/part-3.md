---
author: "Julian Tölle"
title: "Part 3: Implementing Controller: Node"
date: "2023-09-06"
description: |
  Tutorial on building a Kubernetes cloud-controller-manager from Scratch.
  In this post we outline the background and scope of the tutorial and setup the boilerplate.
summary: |
  Tutorial on building a Kubernetes cloud-controller-manager from Scratch.
  In this post we outline the background and scope of the tutorial and setup the boilerplate.
tags: ["kubernetes"]
ShowToc: true
ShowReadingTime: true
ShowBreadCrumbs: true
---

This is part 3 out of 5 of the series [Building a Kubernetes cloud-controller-manager]({{< ref "." >}}).


> Goal: Implement the Node controller with all features, deploy & verify

Now that our side-quest is finished, we can finally go back to the part why you are all here: Implementing the controllers of our cloud-controller-manager.

This part focuses on the Node controller. To [recap](TODO Link to part 1) its functionality:
- _Node_
    - Match Kubernetes `Node` to Cloud API Server (/Instance/Droplet/...) and set `spec.providerID`
    - Label Kubernetes `Nodes` with metadata of the Node, including the [Instance Type](https://kubernetes.io/docs/reference/labels-annotations-taints/#nodekubernetesioinstance-type), Topology ([Region](https://kubernetes.io/docs/reference/labels-annotations-taints/#topologykubernetesioregion), [Zone](https://kubernetes.io/docs/reference/labels-annotations-taints/#topologykubernetesiozone))
    - Figure out the Network addresses of the `Node` and set them in `status.addresses`.
- _Node Lifecycle_
    - The `kubelet` is usually responsible for creating the `Node` object in Kubernetes on its first startup.
    - This does not work well for the removal of the node, as the `kubelet` might not get a chance to do this. Instead, the Node Lifecycle controller regularly checks if the Node was deleted in the Cloud API, and if it was it also deletes the `Node` object in Kubernetes.

Luck has is, `k8s.io/cloud-provider` already implements most of the logic, so we can focus on the integration with our Cloud Provider (Hetzner Cloud in my case).
Similar to the general `Cloud` interface from [Part 1](TODO LINK), this post focuses on the [InstancesV2](https://pkg.go.dev/k8s.io/cloud-provider@v0.28.2#InstancesV2) interface.

Lets start by stubbing out the interface in a new file:

```go
package ccm
// ccm/instances_v2.go

import (
  "context"
  
  cloudprovider "k8s.io/cloud-provider"
	corev1 "k8s.io/api/core/v1"
)

type InstancesV2 struct {}

// Let's try to assign our struct to a var of the interface we try to implement.
// This way we get nice feedback from our IDE if we break the contract.
var _ cloudprovider.InstancesV2 = InstancesV2{}

func (i *InstancesV2) InstanceExists(ctx context.Context, node *corev1.Node) (bool, error) { 
	// We return true for now, to avoid CCM removing any Nodes from the cluster.
	return true, nil
}

func (i *InstancesV2) InstanceShutdown(ctx context.Context, node *corev1.Node) (bool, error) {
	return false, nil
}

func (i *InstancesV2) InstanceMetadata(ctx context.Context, node *corev1.Node) (*cloudprovider.InstanceMetadata, error) {
	return nil, nil
}
```

### Creating our Cloud Provider Client

Before we can make any requests to our Cloud Provider API, we need to read some credentials and create the client.
As mentioned in Part 1, we are going to use [hcloud-go](https://github.com/hetznercloud/hcloud-go), but you can use whatever client you like for your case.
Because we also need the client in other controllers, we will create it in `ccm.NewCloudProvider()` and save it to the `CloudProvider` struct.

```go
package ccm
// ccm/cloud.go

import (
	"os"
	
	"github.com/hetznercloud/hcloud-go/hcloud"
)

type CloudProvider struct {
	client *hcloud.Client
}

func NewCloudProvider(_ *config.CompletedConfig) cloudprovider.Interface {
	// In Part 2 we setup a Kubernetes Secret with our Hetzner Cloud API token, that is mounted as
	// the environment variable HCLOUD_TOKEN in our Pod.
	token := os.Getenv("HCLOUD_TOKEN")
	
	client := hcloud.NewClient(
    hcloud.WithToken(token),
	  // Setting a custom user agent
	  hcloud.WithApplication("ccm-from-scratch", ""),
  )
	
	
  return CloudProvider{client: client}
}
```

We can now initialize the `InstancesV2` struct and pass the client to it:

```go
package ccm

// ccm/instances_v2.go
type InstancesV2 struct {
	client *hcloud.Client
}

// ccm/cloud.go
func (c CloudProvider) InstancesV2() (cloudprovider.InstancesV2, bool) {
  return InstancesV2{client: c.client}, true
}
```

When we start our program now, it should detect that the InstancesV2 interface is supported and start the Node & NodeLifecycle controllers.

### InstanceMetadata

// TODO: Start with everything inline, refactor it out in the next section

### Refactoring

### InstanceExists

### InstanceShutdown

### Testing (in prod!)