---
author: "Julian Tölle"
title: "Part 2: Development Environment with Terraform, k3s and Skaffold"
date: "2023-09-06"
description: |
  [SideQuest] Goal: Get an environment to deploy our binary to, to test any controllers we implement
summary: |
  [SideQuest] Goal: Get an environment to deploy our binary to, to test any controllers we implement
tags: ["kubernetes"]
ShowToc: true
ShowReadingTime: true
ShowBreadCrumbs: true
---

> If this topic does not interest you, you can safely skip ahead! While we use what we built in the following parts, you can still understand what is going on without reading this article.

This is part 2 out of 5 of the series [Building a Kubernetes cloud-controller-manager]({{< ref "." >}}).

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

Nice! We now have a working Kubernetes Cluster, so let's deploy our CNI solution Cilium to it. We are going to use the Cilium Helm Chart & and the `helm` Provider for Terraform to install it.

As with the hcloud provider, we need to configure Helm and tell it how to connect to the cluster.

`k3sup` wrote the config to a local file, and luckily the provider supports reading this _but_ we need to make sure that the file is actually written / the Cluster is initialized _before_ we can use it.
Terraform has the `depends_on` meta-argument, but this does not work on `providers`.
Instead we will create an intermediary `local_sensitive_file` resource that has `depends_on` specified, and use its output `filename` in the Helm provider config.

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

data "local_sensitive_file" "kubeconfig" {
  # Kubeconfig is only written after control-plane is initialized finished
  depends_on = [null_resource.control]

  filename = local.kubeconfig_path
}

provider "helm" {
  kubernetes {
    # And now we pass the filename from the data source
    config_path = data.local_sensitive_file.kubeconfig.filename
  }
}
```

With the provider configured, we can install Cilium:

```terraform
resource "helm_release" "cilium" {
  name       = "cilium"
  chart      = "cilium"
  repository = "https://helm.cilium.io"
  namespace  = "kube-system"
  version    = "1.13.1"

  set {
    name  = "ipam.mode"
    value = "kubernetes"
  }
}
```

For our own purposes, we will need to create one more resource: a `Secret` containing the Hetzner Cloud API Token. We will use this later in the `cloud-controller-manager` to access the API.

We will use the `kubernetes` provider this time, which is configured similar to the `helm` provider:

```terraform
provider "kubernetes" {
  config_path = data.local_sensitive_file.kubeconfig.filename
}

resource "kubernetes_secret_v1" "hcloud_token" {
  metadata {
    name      = "hcloud"
    namespace = "kube-system"
  }

  data = {
    token = var.hcloud_token
    network = hcloud_network.cluster.id
  }
}
```

#### Applying the configuration

All the resources we will need (for now) exist, so let's apply the configuration:

```shell
$ terraform init
$ terraform apply
// TODO Output
```

### Kubernetes Manifests

To deploy our application, we need to write some YAML.
I will not spend too much time explaining this, as its not the focus of the tutorial.
We will need a `Deployment` and some RBAC rules to enable access to the Kubernetes API.

```yaml
# deploy.yaml
kind: Deployment
apiVersion: apps/v1
metadata:
  name: ccm-from-scratch
  namespace: kube-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ccm-from-scratch
  template:
    metadata:
      labels:
        app: ccm-from-scratch
    spec:
      hostNetwork: true
      serviceAccountName: cloud-controller-manager
      tolerations:
        - effect: NoSchedule
          key: node.cloudprovider.kubernetes.io/uninitialized
          operator: Exists

      containers:
        - name: ccm-from-scratch
          # This points to my DockerHub repository, you can use your own
          image: apricote/ccm-from-scratch:latest
          args: []
          env:
            - name: HCLOUD_TOKEN
              valueFrom:
                secretKeyRef:
                  key: token
                  name: hcloud

---
kind: ServiceAccount
apiVersion: v1
metadata:
  name: cloud-controller-manager
  namespace: kube-system
---

kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: cloud-controller-manager
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  # This binds us to the all-powerful cluster-admin role. In a real-world
  # scenario, you would want to restrict this to a custom ClusterRole that only
  # has access to the required paths.
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: cloud-controller-manager
    namespace: kube-system
```

### Skaffold

[Skaffold](https://skaffold.dev/) is a tool to automate the deployment of applications to Kubernetes.
It can watch the local filesystem for changes and automatically re-deploy the application with an updated image.


#### Config

Skaffold requires a configuration file. We need to specify which image to build & how our application should be deployed.

For Image building we will use [ko](https://ko.build/).
It produces very small images very quickly, and we do not need to write an _artisanal_ Dockerfile.
The `ko` builder by default builds & containerizes the `main.go` in the project root.

For deployment we use the `kubectl` deployer and point it at the `deploy.yaml` from the last section.

```yaml
# skaffold.yaml
apiVersion: skaffold/v4beta7
kind: Config
metadata:
  name: ccm-from-scratch
build:
  artifacts:
      # Must match the image you used in deploy.yaml
    - image: apricote/ccm-from-scratch
      ko: {}
deploy:
  kubectl:
    manifests:
      - deploy.yaml
```

TODO: SKAFFOLD_DEFAULT_REPO?

#### Babies first deploy

Lets try to deploy our application:

```shell
$ export KUBECONFIG=kubeconfig.yaml
$ export SKAFFOLD_DEFAULT_REPO? TODO
$ skaffold run
... TODO Output
```

> If you are using Docker Hub the first deployment will probably fail, Docker Hub sets new images to "private" by default.
> You need to configure an `ImagePullSecret` or make the image public in the Docker Hub settings.
> Afterward you can retry the `skaffold run` step and it should work.

If everything worked, you should now have a running `ccm-from-scratch` deployment in the `kube-system` namespace and `skaffold` should show some logs from the application.

```shell
$ kubectl get pods -n kube-system -l app=ccm-from-scratch
TODO output
```