---
author: "Julian Tölle"
title: "Building a Kubernetes cloud-controller-manager"
date: "2023-09-06"
description: |
  Tutorial on building a Kubernetes cloud-controller-manager from Scratch.
  In this post we outline the background and scope of the tutorial and setup the boilerplate.
tags: ["kubernetes"]
ShowToc: true
ShowReadingTime: true
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

Lets start by listing the requirements for our development environment:
- Single command to start the environment
- Working Kubernetes Cluster
- Running on Hetzner Cloud servers (as we target this environment with our cloud-controller-manager, you can substitute in your own cloud provider)
- Some way to deploy & debug our code

I have chosen the following technologies to implement this:

- Terraform Infrastructure-as-Code
  - Using the [`terraform-provider-hcloud`](https://github.com/hetznercloud/terraform-provider-hcloud) to manage the Hetzner Cloud resources
  - Using the `kubernetes` & `helm` providers to add the required resources to the cluster
- k3s through k3sup for the Kubernetes cluster
- Skaffold for the deployment & debugging

These tools should be installed locally:
- k3sup
- skaffold
- helm
- kubectl
- terraform

### Terraform

Terraform is an Infrastructure-as-Code tool. It allows us to define our infrastructure in a declarative way and then apply the changes to our cloud provider. If you have never used it before, I would recommend checking out some of their [tutorials](https://developer.hashicorp.com/terraform/tutorials) to get you started.

I will only `apply` our configuration at the end, to keep this post short. If you are developing this, I would recommend running `terraform plan` & `terraform apply` in between the steps to make sure everything so far works as expected.

Let us create our main terraform file:

```hcl
# dev-env.tf
terraform {
  # In this block we will later define all providers we use.
  required_providers {}
}
```

#### SSH Key

As we try to keep the development environment mostly self-contained, we start by creating an SSH Key that we can use to connect to our servers:

```hcl
terraform {
  required_providers {
    tls = {
      source  = "hashicorp/tls"
      version = "4.0.4"
    }
    local = {
      source  = "hashicorp/local"
      version = "2.4.0"
    }
  }
}


# This will generate a new random private key and save it to the Terraform state
resource "tls_private_key" "ssh" {
  algorithm = "ED25519"
}

# This writes out the private key to a file in the current directory.
# We can use this to connect to our servers through SSH later.
resource "local_sensitive_file" "ssh" {
  content  = tls_private_key.ssh.private_key_openssh
  filename = "${path.module}/.dev-env.ssh"
}
```

Now is a good time to add some Terraform specific files to our `.gitignore`:

```gitignore
# .gitignore

# Directory created by terraform for caches & internal state
.terraform
# The terraform state
terraform.tfstate
# Terraform makes a backup of the state before any operation
terraform.tfstate.backup

# Our private key we wrote in local_sensitive_file.ssh
.dev-env.ssh

# We will use this file in the next step to load credentials for the hcloud provider
credentials.auto.tfvars

# We will save the kubeconfig to this path in a later step
kubeconfig.yaml
```

#### Hetzner Cloud Resources

We will need:
- An SSH Key
- A Network & Subnet (for the `Routes` controller)
- A Control Plane Server
- 0 to X Worker Servers

To access the Hetzner Cloud API, we need to configure the provider with a token:

```hcl
terraform {
  required_providers {
    # Add to the others
    hcloud = {
      source  = "hetznercloud/hcloud"
      version = "1.42.1"
    }
  }
}

variable "hcloud_token" {
  type      = string
  sensitive = true
}

# Configure the Hetzner Cloud Provider
provider "hcloud" {
  token = var.hcloud_token
}
```

Now we can create create a new project in the [Clound Console](https://console.hetzner.cloud) and [generate a new API Token](https://docs.hetzner.com/cloud/api/getting-started/generating-api-token). This token will be saved in `credentials.auto.tfvars`. Because the file has the suffix `auto.tfvars`, Terraform will automatically use it.

```terraform
# credentials.auto.tfvars
hcloud_token = "YOUR_TOKEN"
```

Next, create the SSH Key:

```terraform
resource "hcloud_ssh_key" "default" {
  name       = "ccm-from-scratch dev env"
  # Using the public key from the SSH Key we generated earlier
  public_key = tls_private_key.ssh.public_key_openssh
}
```

Now we can create the Network & Subnet:

```terraform
resource "hcloud_network" "cluster" {
  name = "ccm-from-scratch"
  ip_range = "10.0.0.0/8"
}

resource "hcloud_network_subnet" "cluster" {
  ip_range     = "10.0.0.0/24"
  network_id   = hcloud_network.cluster.id
  network_zone = "eu-central"
  type         = "cloud"
}
```

And finally create the servers and add them to the network:

```hcl
resource "hcloud_server" "control" {
  name        = "ccm-from-scratch-control"
  server_type = "cpx11"
  location    = "fsn1"
  image       = "ubuntu-22.04"
  # Pass in the ssh key
  ssh_keys    = [hcloud_ssh_key.default.id]

  connection {
    host        = self.ipv4_address
    user        = "root"
    type        = "ssh"
    private_key = tls_private_key.ssh.private_key_openssh
  }

  # This makes sure that the server is fully initialized before we install the Kubernetes cluster
  provisioner "remote-exec" {
    inline = ["cloud-init status --wait"]
  }
}

resource "hcloud_server_network" "control" {
  server_id   = hcloud_server.control.id
  subnet_id  = hcloud_network_subnet.cluster.id
}
```

As we want to be able to scale the number of worker nodes, we will use a `count` to create multiple servers, besides that and the name prefix, the configuration are the same:

```terraform
variable "worker_count" {
  type = number
  default = 0
}

resource "hcloud_server" "worker" {
  count = var.worker_count

  name        = "ccm-from-scratch-worker-${count.index}"
  server_type = "cpx11"
  location    = "fsn1"
  image       = "ubuntu-22.04"
  ssh_keys    = [hcloud_ssh_key.default.id]
}

resource "hcloud_server_network" "control" {
  for_each = hcloud_server.worker
  
  server_id   = each.id
  subnet_id  = hcloud_network_subnet.cluster.id
}
```

#### Kubernetes Cluster

As mentioned earlier, we will use `k3sup` to install `k3s` and other dependencies on the servers and join them to the Cluster.

We will use the `null_resource` and a local provisioner to run these commands locally as soon as the servers are ready. The control plane server needs a different command from the worker nodes, so we will use two resources.

```terraform
terraform {
  required_providers {
    null = {
      source = "hashicorp/null"
      version = "3.2.1"
    }
  }
}

# We need to define some additional configuration values for the Kubernetes cluster
locals {
  # The CIDR range for the Pods, must be included in the range of the
  # network (10.0.0.0/8) but must not overlap with the Subnet (10.0.0.0/24)
  cluster_cidr = "10.244.0.0/16"
  
  # Path to write the kubeconfig to, should be the same as the path in our gitignore from earlier.
  # path.module is the directory the `dev-env.tf` file is in.
  kubeconfig_path = "${path.module}/kubeconfig.yaml"
  
  # As mentioned in the first article, we use Kubernetes v1.28
  k3s_channel = "v1.28"
}

resource "null_resource" "k3sup_control" {
  triggers = {
    # When these triggers change, the resource will be recreated and the provisioner runs again.
    id = hcloud_server.control.id
    # This is added to make sure that the server is attached to the network before we continue
    ip = hcloud_server_network.control.ip
  }
  
  provisioner "local-exec" {
    # This will run k3sup on the local machine. Lets look at some of the flags:
    # --ssh-key and --ip specify how to connect to the server
    # --k3s-channel specifies which version of k3s to install
    # --k3s-extra-args allows us to pass additional arguments to the k3s installer:
    #   --disable-cloud-controller disables the built-in cloud-controller-manager, as we will use our own
    #   --cluster-cidr specifies the CIDR range for the Pods, which is relevant for the Routes controller
    #   --kubelet-arg cloud-provider=external makes sure that the kubelet expects a cloud-controller-manager and taints nodes accordingly
    #   --disable=XYZ disables a bunch of built-in addons that we do not need for our tests
    #   --flannel-backend=none disables the built-in CNI Flannel. We will install Cilium later, as it works better with the Routes controller
    #   --node-external-ip and --node-ip specify the external and internal IP of the node
    # --local-path specifies where k3sup will save the kubeconfig to access the cluster
    command = <<EOT
      k3sup install --print-config=false \
        --ssh-key "${local_sensitive_file.ssh.filename}" \
        --ip "${hcloud_server.control.ipv4_address}" \
        --k3s-channel "${local.k3s_channel}" \
        --k3s-extra-args "--disable-cloud-controller --cluster-cidr ${local.cluster_cidr} --kubelet-arg cloud-provider=external --disable=traefik --disable=servicelb --flannel-backend=none --disable=local-storage --node-external-ip ${hcloud_server.control.ipv4_address} --node-ip ${hcloud_server_network.control.ip}" \
        --local-path "${local.kubeconfig_path}"
    EOT
  }
}
```

The workers use `k3sup join` instead of `k3sup install`, and they need to wait for the control plane to finish installing before they can join:

```terraform
resource "null_resource" "k3sup_worker" {
  for_each = hcloud_server.worker
  
  triggers = {
    # Same as before
    id = each.value.id.id
    ip = hcloud_server_network.worker[each.key].ip

    # If the control-plane server changed, we will also need to re-join our workers to the new cluster.
    # In addition this adds a dependency on the fully initialized control-plane server
    control_id = null_resource.k3sup_control.id
  }
  
  provisioner "local-exec" {
    # Again we will run k3sup locally, this time with less flags. The only new flag is:
    # --server-ip specifies the IP of the control-plane server
    command = <<EOT
      k3sup join \
        --ssh-key "${local_sensitive_file.ssh.filename}" \
        --ip "${self.ipv4_address}" \
        --server-ip "${hcloud_server.control.ipv4_address}" \
        --k3s-channel "${local.k3s_channel}" \
        --k3s-extra-args "--kubelet-arg cloud-provider=external --node-external-ip ${each.value.ipv4_address} --node-ip ${hcloud_server_network.worker[each.key].ip}" \
      EOT
  }
}
```

#### Kubernetes Addons

Nice! We now have a working Kubernetes Cluster, so let's deploy our CNI solution Cilium to it. We are going to use the Cilium Helm Chart & and the `helm` Provider for Terraform to install it:


```terraform
terraform {
  required_providers {
    # Add to the others
    helm = {
      source = "hashicorp/helm"
      version = "2.11.0"
    }
  }
}

# As with the hcloud provider, we need to configure Helm and tell it how to connect to the cluster.
# k3sup wrote the config to a local file, and luckily the provider supports reading this BUT we need to make sure that the file is actually written / the Cluster is initialized before we can use it. Terraform has the `depends_on` we can use the local terraform provider to read it:
# TODO, finalize description above, maybe make actual text?
data "local_sensitive_file" "kubeconfig" {
  # Kubeconfig is only written after control-plane is initialized finished
  depends_on = [null_resource.control]

  filename = local.kubeconfig_path
}

provider "helm" {
  kubernetes {
    # And now we pass the 
    config_path = data.local_sensitive_file.kubeconfig.filename
  }
}

```



## (Part 3) Controller: Node

> Goal: Implement the Node controller with all features, deploy & verify

## (Part 4) Controller: Service (Load Balancers)

> Goal: Setup Load Balancers so traffic reaches our cluster, set up an ingress in the dev env that uses the LB

## (Part 5) Controller: Route

> Goal: Use the Hetzner Cloud Networks instead of Overlay Networks for Pod CIDR connectivity