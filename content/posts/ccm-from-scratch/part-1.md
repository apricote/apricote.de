---
author: "Julian Tölle"
title: "Part 1: Setting up the Boilerplate"
date: "2023-09-06"
description: |
  Goal: Get a compilable binary running k/cloud-provider with no controllers yet
summary: |
  Goal: Get a compilable binary running k/cloud-provider with no controllers yet

tags: ["kubernetes"]
ShowToc: true
ShowReadingTime: true
ShowBreadCrumbs: true
---

This is part 1 out of 5 of the series [Building a Kubernetes cloud-controller-manager]({{< ref "." >}}).

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
