---
title: "Using Terraform to manage Kubernetes"
date: 2020-11-15T12:31:10Z
draft: false
---
When working with kubernetes the standard way to create objects such as pods, deployments and services is to create a yaml manifest file that describes the desired state and then use ```kubectl``` to create the object.
Althought, straightforward this process might create a barrier for developers and operations teams that now have to mantain a new codebase in a new "language". YAML is easy to read but troubleshooting it might be difficult due the need for correct identation and the errors that are exibithed by ```kubectl``` might not be easily understood.
If a team is already using Terraform to create and manage the infrastructure it makes sense to use it to create the objects inside kubernetes as well, this brings some advantages such as:

- A single workflow to administer both the underlying infrastructure and the cluster itself
- Easly templating without the need to use tools like ``helm``, 
- Lifecycle management without the need to check the Kubernetes' API
- Terraform is capable of understanding the relationship between resources so if yo



{{< highlight hcl "linenos=table,linenostart=1" >}}
terraform {
  required_providers {
    kubernetes = {
      source = "hashicorp/kubernetes"
    }
  }
}

provider "kubernetes" {}
{{< / highlight >}}

Then let's create a pod


{{< highlight hcl "linenos=table,linenostart=1" >}}
resource "kubernetes_pod" "pod" {
  metadata {
    name = "pod"
  }
  spec {
    container {
      image = "nginx:1.19.4-alpine"
      name  = "nginx"
    }

  }

}
{{< / highlight >}}