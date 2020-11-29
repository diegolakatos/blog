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
- Terraform is capable of understanding the relationship between resources, if a resource has dependencies that failed to create Terraform will understand this and don't create the resource

With all these considerations done let's start working. The first step is to configure the kubernetes provider:

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

Now to create  a pod we can use the following file:


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

To apply it just run:

{{< highlight bash "linenos=table,linenostart=1">}}
$ terraform apply

Plan: 1 to add, 0 to change, 0 to destroy.

Do you want to perform these actions?
  Terraform will perform the actions described above.
  Only 'yes' will be accepted to approve.

  Enter a value: yes

kubernetes_pod.pod: Creating...
kubernetes_pod.pod: Creation complete after 2s [id=default/pod]

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.

{{< / highlight >}}

