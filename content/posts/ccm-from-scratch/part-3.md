---
author: "Julian Tölle"
title: "Part 3: Implementing Controller: Node"
date: "2023-09-06"
description: |
  Goal: Implement the Node controller with all features, deploy & verify.
summary: |
  Implement the Node controller with all features, deploy & verify.
tags: ["kubernetes"]
ShowToc: true
ShowReadingTime: true
ShowBreadCrumbs: true
---

This is part 3 out of 5 of the series [Building a Kubernetes cloud-controller-manager]({{< ref "." >}}).

Now that our side-quest is finished, we can finally go back to the part why you are all here: Implementing the controllers of our cloud-controller-manager.

This part focuses on the Node controller. To [recap]({{< ref ".#what-is-a-cloud-controller-manager" >}}) its functionality:
- _Node_
    - Match Kubernetes `Node` to Cloud API Server (/VM/Instance/Droplet/...) and set [`spec.providerID`][ref-node-spec]
    - Label Kubernetes `Nodes` with the [Instance Type][label-instance-type] and Topology info ([Region][label-region], [Zone][label-zone])
    - Figure out the Network addresses of the `Node` and set them in `status.addresses`.
    - Mark `Node` as initialized by removing the [`node.cloudprovider.kubernetes.io/uninitialized`][taint-uninitialized] taint.
- _Node Lifecycle_
  
  The `kubelet` is usually responsible for creating the `Node` object in Kubernetes on its first startup.
  This does not work well for the removal of the node, as the `kubelet` might not get a chance to do this. Instead, the Node Lifecycle controller regularly checks if the Node was deleted in the Cloud API, and if it was it also deletes the `Node` object in Kubernetes.

[ref-node-spec]: https://kubernetes.io/docs/reference/kubernetes-api/cluster-resources/node-v1/#NodeSpec
[label-instance-type]: https://kubernetes.io/docs/reference/labels-annotations-taints/#nodekubernetesioinstance-type
[label-region]: https://kubernetes.io/docs/reference/labels-annotations-taints/#topologykubernetesioregion
[label-zone]: https://kubernetes.io/docs/reference/labels-annotations-taints/#topologykubernetesiozone
[taint-uninitialized]: https://kubernetes.io/docs/reference/labels-annotations-taints/#node-cloudprovider-kubernetes-io-uninitialized

Luck has is, `k8s.io/cloud-provider` already implements most of the logic, so we can focus on the integration with our Cloud Provider (Hetzner Cloud in my case).
Similar to the general `cloudprovider.Interface` from [Part 1]({{< ref "part-1.md#cloudproviderinterface" >}}), this post focuses on the [`cloudprovider.InstancesV2`](https://pkg.go.dev/k8s.io/cloud-provider@v0.28.2#InstancesV2) interface.

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
// This way we get nice feedback from the compiler (and the IDE) if we break the contract.
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
As mentioned in Part 1, we are going to use [hcloud-go](https://github.com/hetznercloud/hcloud-go) here, but you should use a client for your API.
Because we also need the client in other controllers, we will create it in `ccm.NewCloudProvider()` and save it to the `CloudProvider` struct.

We also need to parse and the save the ID of the network that our nodes use to communicate with each other. We will use this later to figure out the internal IP of the node.

```go {hl_lines=[13,14,"18-35"]}
package ccm
// ccm/cloud.go

import (
	"fmt"
	"os"
	"strconv"
	
	"github.com/hetznercloud/hcloud-go/v2/hcloud"
)

type CloudProvider struct {
	client    *hcloud.Client
	networkID int64
}

func NewCloudProvider(_ *config.CompletedConfig) cloudprovider.Interface {
	// In Part 2 we set up a Kubernetes Secret with our Hetzner Cloud API token.
	// We mounted it as the environment variable HCLOUD_TOKEN into our Pod.
	token := os.Getenv("HCLOUD_TOKEN")
	
	client := hcloud.NewClient(
	  hcloud.WithToken(token),
	  // Setting a custom user agent
	  hcloud.WithApplication("ccm-from-scratch", ""),
	)
	
	// Also from Part 2
	networkID, err := strconv.ParseInt(os.Getenv("HCLOUD_NETWORK_ID"), 10, 64)
	if err != nil {
		// TODO: Return error?
		panic(fmt.Errorf("failed to parse HCLOUD_NETWORK_ID: %w", err))
	}
	
	return CloudProvider{client: client, networkID: networkID}
}
```

We can now initialize the `InstancesV2` struct and pass the client to it:

```go {hl_lines=[5,6,11]}
package ccm
// ccm/instances_v2.go

type InstancesV2 struct {
	client    *hcloud.Client
	networkID int64
}

// ccm/cloud.go
func (c CloudProvider) InstancesV2() (cloudprovider.InstancesV2, bool) {
	return InstancesV2{client: c.client, networkID: c.networkID}, true
}
```

When we start our program now, it should detect that the `InstancesV2` interface is supported and start the _Node_ & _NodeLifecycle_ controllers.


### Matching Up Nodes

For all methods in this interface, we need to ask the Cloud API about details of our server. For this, we need to somehow match the Kubernetes `Node` with its Cloud API server equivalent.

On the first run, we need to use some heuristic to make this match. All methods in the interface receive the `Node` as a parameter, so we could use any of its fields, like the Name, IP, Labels or Annotations. To keep this simple, we are going to use the Node name and expect that the server in the Hetzner Cloud API has the same name. 

In `InstanceMetadata` we return a `ProviderID`, which is written to `Node.spec.providerID` by `k/cloud-provider`. After the node was initialized by us, we can therefore use the ID we previously saved, and do not need to keep using the name heuristic. We are going to use the format `ccm-from-scratch://$ID` for our `ProviderID`. By using a unique prefix, we can avoid issues where users accidentally install multiple cloud-providers into their cluster.

### InstanceMetadata

The `InstanceMetadata` method is called by the _Node_ controller when the Node is initialized for the first time.

To get started, we need to get the Server from the API, using the method described in the [previous section](#matching-up-nodes). Matching up by name is pretty easy:

```go
package ccm

// ccm/instances_v2.go

import (
	"fmt"
)

const (
	// A constant for the ProviderID. If you want, you can also update `CloudProvider.ProviderName` to use it.
	providerName = "ccm-from-scratch"
)

func (i *InstancesV2) InstanceMetadata(ctx context.Context, node *corev1.Node) (*cloudprovider.InstanceMetadata, error) {
	var server *hcloud.Server
	var err error

	if node.Spec.ProviderID == "" {
	  // If no ProviderID was set yet, we use the name to get the right server
	  // Luckily this is pretty easy with hcloud-go:
	  server, _, err = i.client.Server.GetByName(ctx, node.Name)
	  if err != nil {
		  return nil, err
	  }
	} else {
	  // If the Node was previously initialized, it should have a ProviderID set.
	  // We can parse it to get the ID and then use that to get the server from the API.
	  providerID, found := strings.CutPrefix(node.Spec.ProviderID, fmt.Sprintf("%s://", providerName))
	  if !found {
		  return nil, fmt.Errorf("ProviderID does not follow expected format: %s", node.Spec.ProviderID)
	  }

	  id, err := strconv.ParseInt(providerID, 10, 64)
	  if err != nil {
		  return nil, fmt.Errorf("unable to parse ProviderID to integer: %w", err)
	  }
	  
	  server, _, err = i.client.Server.GetByID(ctx, id)
	  if err != nil {
		  return nil, err
	  }
	}
	
	if server == nil {
	  return nil, errors.New("server not found")
	}
	
	return nil, nil
}
```

Now that we have the server, we can start filling out the `InstanceMetadata` struct. It looks like this:

```go
package cloudprovider

// Source: TODO
type InstanceMetadata struct {
	ProviderID string
	InstanceType string
	NodeAddresses []corev1.NodeAddress
	Zone string
	Region string
}
```

We already discussed `ProviderID`, the values for `InstanceType` is also quite easy, we are going to use the Server Type from the Hetzner Cloud API). `Zone` and `Region` are a bit more open ended, and I recommend you read the docs for their respective labels ([Region][label-region], [Zone][label-zone]). In short, the Zone is smaller and Regions span multiple zones. For the Hetzner Cloud API, we are going to use the Network Zone (eg. `eu-central`) as the Region, and the Location (eg. `fsn1`) as the Zone. For your own API, you should evaluate what makes the most sense based on the availability guarantees of your Zones/Regions/DCs/... and the needs of your customers.

The `NodeAddresses` is a bit more complicated. We need to list all the addresses of the server along with their respective type (`NodeHostName`, `NodeExternalIP`, `NodeInternalIP`). We will put this into a separate function to keep the `InstanceMetadata` method clean.

```go
package ccm

// ccm/instances_v2.go
func getNodeAddresses(server *hcloud.Server, networkID int64) []corev1.NodeAddress {
	// No matter what, we always have the hostname, which equals
	// the name of the server in Hetzner Cloud default images.
	addresses := []corev1.NodeAddress{
		{
			Type:    corev1.NodeHostName,
			Address: server.Name,
		},
	}

	// If the server has a public IPv4 address, we add it to the list.
	// Disabling the public IPv4 is possible in Hetzner Cloud API, so
	// we need to check for it.
	if !server.PublicNet.IPv4.IsUnspecified() {
		addresses = append(addresses, corev1.NodeAddress{
			Type:    corev1.NodeExternalIP,
			Address: server.PublicNet.IPv4.IP.String(),
		})
	}
	
	// For simplicity, we only support IPv4 here, but you can easily
	// extend this for IPv6 or Dualstack clusters. 

	// The server might be attached to multiple Networks, so we
	// need to iterate through them to find the one we are
	// looking for, based on the configured network ID.
	for _, privNet := range server.PrivateNet {
		if privNet.Network.ID != networkID {
			continue
		}

		if !privNet.IP.IsUnspecified() {
			addresses = append(addresses, corev1.NodeAddress{
				Type:    corev1.NodeInternalIP,
				Address: privNet.IP.String(),
			})
		}
	}

	return addresses
}

func (i *InstancesV2) InstanceMetadata(ctx context.Context, node *corev1.Node) (*cloudprovider.InstanceMetadata, error) {
	var server *hcloud.Server 
	// Skipping the previous code for brevity

	return &cloudprovider.InstanceMetadata{
		ProviderID:    fmt.Sprintf("%s://%d", providerName, server.ID),
		InstanceType:  server.ServerType.Name,
		NodeAddresses: getNodeAddresses(server, i.networkID),
		Zone:          server.Datacenter.Location.Name,
		Region:        string(server.Datacenter.Location.NetworkZone),
	}, nil
}
```

### Refactoring

Now that we have the basic functionality in, I would like to extract the code to parse the `ProviderID` and get the node into their own functions. This will make them reusable for the other two methods in the Interface.

Lets start with `getProviderID`:

```go {hl_lines=["4-15","30-33"]}
package ccm
// ccm/instances_v2.go

func getProviderID(node *v1.Node) (int64, error) {
	providerID, found := strings.CutPrefix(node.Spec.ProviderID, fmt.Sprintf("%s://", providerName))
	if !found {
		return 0, fmt.Errorf("ProviderID does not follow expected format: %s", node.Spec.ProviderID)
	}

	id, err := strconv.ParseInt(providerID, 10, 64)
	if err != nil {
		return 0, fmt.Errorf("unable to parse ProviderID to integer: %w", err)
	}
	return id, nil
}


func (i *InstancesV2) InstanceMetadata(ctx context.Context, node *corev1.Node) (*cloudprovider.InstanceMetadata, error) {
	var server *hcloud.Server
	var err error

	if node.Spec.ProviderID == "" {
		// If no ProviderID was set yet, we use the name to get the right server
		// Luckily this is pretty easy with hcloud-go:
		server, _, err = i.client.Server.GetByName(ctx, node.Name)
		if err != nil {
			return nil, err
		}
	} else {
		id, err := getProviderID(node)
		if err != nil {
			return nil, err
		}

		server, _, err = i.client.Server.GetByID(ctx, id)
		if err != nil {
			return nil, err
		}
	}

	if server == nil {
		return nil, errors.New("server not found")
	}

	return &cloudprovider.InstanceMetadata{
		ProviderID:    fmt.Sprintf("%s://%d", providerName, server.ID),
		InstanceType:  server.ServerType.Name,
		NodeAddresses: getNodeAddresses(server, i.networkID),
		Zone:          server.Datacenter.Location.Name,
		Region:        string(server.Datacenter.Location.NetworkZone),
	}, nil
}
```

Next we move out the code to get the server into `getServerForNode`:

```go {hl_lines=["4-32","35-38"]}
package ccm
// ccm/instances_v2.go

func (i InstancesV2) getServerForNode(ctx context.Context, node *v1.Node) (*hcloud.Server, error) {
	var server *hcloud.Server
	var err error

	if node.Spec.ProviderID == "" {
		// If no ProviderID was set yet, we use the name to get the right server
		// Luckily this is pretty easy with hcloud-go:
		server, _, err = i.client.Server.GetByName(ctx, node.Name)
		if err != nil {
			return nil, err
		}
	} else {
		id, err := getProviderID(node)
		if err != nil {
			return nil, err
		}

		server, _, err = i.client.Server.GetByID(ctx, id)
		if err != nil {
			return nil, err
		}
	}

	if server == nil {
		return nil, errors.New("server not found")
	}
	
	return server, nil
}

func (i *InstancesV2) InstanceMetadata(ctx context.Context, node *corev1.Node) (*cloudprovider.InstanceMetadata, error) {
	server, err := i.getServerForNode(ctx, node)
	if err != nil {
		return nil, err
	}

	return &cloudprovider.InstanceMetadata{
		ProviderID:    fmt.Sprintf("%s://%d", providerName, server.ID),
		InstanceType:  server.ServerType.Name,
		NodeAddresses: getNodeAddresses(server, i.networkID),
		Zone:          server.Datacenter.Location.Name,
		Region:        string(server.Datacenter.Location.NetworkZone),
	}, nil
}
```

Now that is looking better! One last thing I want to do is to replace the inline error (`errors.New("server not found")`) with a predefined error, which makes implementing the next method way easier.

```go {hl_lines=[5,32]}
package ccm
// ccm/instances_v2.go

var (
	errServerNotFound = errors.New("server not found")
)

func (i InstancesV2) getServerForNode(ctx context.Context, node *v1.Node) (*hcloud.Server, error) {
	var server *hcloud.Server
	var err error

	if node.Spec.ProviderID == "" {
		// If no ProviderID was set yet, we use the name to get the right server
		// Luckily this is pretty easy with hcloud-go:
		server, _, err = i.client.Server.GetByName(ctx, node.Name)
		if err != nil {
			return nil, err
		}
	} else {
		id, err := getProviderID(node)
		if err != nil {
			return nil, err
		}

		server, _, err = i.client.Server.GetByID(ctx, id)
		if err != nil {
			return nil, err
		}
	}

	if server == nil {
		return nil, errServerNotFound
	}

	return server, nil
}
```


### InstanceExists

`InstanceExists` is being called by the _NodeLifecycle_ controller. Based on the return value the `Node` will be deleted. It is probably the easiest out of the three in this interface, as we only need to return `true` if a server matching the `Node` exists in the API, and `false` otherwise.

```go
package ccm
// ccm/instances_v2.go

func (i InstancesV2) InstanceExists(ctx context.Context, node *corev1.Node) (bool, error) {
	// We get the server using the method from the previous section
	_, err := i.getServerForNode(ctx, node)
	if err != nil {
		// If we get back our predefined error, we know that
		// the server does not exist and return false
		if errors.Is(err, errServerNotFound) {
			return false, nil
		}

		return false, err
	}

	// If we get here, the server exists and we return true.
	return true, nil
}
```

And thats it. Pretty simple.

### InstanceShutdown

`InstaceShutdown` is also being called by the _NodeLifecyle_ controller. Based on the return value, the `Node` will get the taint [`node.cloudprovider.kubernetes.io/shutdown`][taint-shutdown], the taint is removed if we return true. Similar to `InstanceExists`, this is pretty easy to implement. For the Hetzner Cloud API, we just need to check if the Server is in status `"off"`.

```go
package ccm
// ccm/instances_v2.go

func (i *InstancesV2) InstanceShutdown(ctx context.Context, node *corev1.Node) (bool, error) {
	server, err := i.getServerForNode(ctx, node)
	if err != nil {
		return false, err
	}

	return server.Status == hcloud.ServerStatusOff, nil
}
```

[taint-shutdown]: https://kubernetes.io/docs/reference/labels-annotations-taints/#node-cloudprovider-kubernetes-io-shutdown

### Testing (in prod!)

And thats it for the `InstancesV2` interface. We can now start our project again and test if it works as expected:

TODO: Test init&metadata

TODO: Test Shutdown

TODO: Test delete