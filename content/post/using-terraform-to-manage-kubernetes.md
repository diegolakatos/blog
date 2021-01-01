---
title: "Using Terraform to manage Kubernetes"
date: 2021-01-01T00:00:00Z
draft: false
DisableComments: false
Description: "This post shows how to use Terraform to manage Kubernetes clusters"
Tags: [Terraform, Kubernetes, IaC, Containers]
Categories: [IaC]
---
When working with kubernetes the standard way to create objects such as pods, deployments and services is to create a yaml manifest file that describes the desired state and then use ```kubectl``` to create/update the object.
Althought, straightforward this process might create a barrier for developers and operations teams that now have to mantain a new codebase in a new "language". YAML is easy to read but troubleshooting it might be difficult due the need for correct identation and the errors that are exibithed by ```kubectl``` might not be easily understood.
If a team is already using Terraform to create and manage the infrastructure it makes sense to use it to create the objects inside kubernetes as well, this brings some advantages such as:

- A single workflow to administer both the underlying infrastructure and the cluster itself
- Easly templating without the need to use tools like ``helm``,
- Lifecycle management without the need to check the Kubernetes' API
- Terraform is capable of understanding the relationship between resources, if a resource has dependencies that failed to create Terraform will understand this and don't create the resource or if you delete an object that has dependencies the dependencies will also be deleted.

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

It's also possible to create more elaborated structures, in the next example we are going to explore the dependency graph that Terraform uses to manage the resources, for this we are going to create 3 objects: one namespace, one deployment and one service:

{{< highlight hcl "linenos=table,linenostart=1" >}}
resource "kubernetes_namespace" "webapplication-namespace" {
  metadata {
    annotations = {
      name = "webapplication"
    }

    labels = {
      team = "42"
    }

    name = "webapplication"
  }
}

resource "kubernetes_deployment" "webapplication-deploy" {
  metadata {
    name = "webapplication"
    labels = {
      team = "42"
      app  = "frontend"
    }
    namespace = kubernetes_namespace.webapplication-namespace.id
  }

  spec {
    replicas = 3

    selector {
      match_labels = {
        team = "42"
        app  = "frontend"
      }
    }

    template {
      metadata {
        labels = {
          team = "42"
          app  = "frontend"
        }
      }

      spec {
        container {
          image = "nginx:1.19.4-alpine"
          name  = "frontend-app"

        }
      }
    }
  }
}

resource "kubernetes_service" "webapplication-service" {
  metadata {
    name = "webapplication-service"
    labels = {
      team = "42"
      app  = "frontend"
    }
  }
  spec {
    selector = {
      app = "kubernetes_deployment.webapplication-deploy.metadata.labels.app"
    }
    port {
      port        = 8080
      target_port = 80
    }

    type = "NodePort"
  }
  depends_on = [kubernetes_namespace.webapplication-namespace]
}
{{< / highlight >}}

Since all resources have the same labels we can use the ```kubectl``` command to inspect the state of the cluster before and after the ``terraform apply``:

{{< highlight bash >}}
$ kubectl get all --all-namespaces -l team=42
No resources found

$ kubectl get ns
NAME              STATUS   AGE
default           Active   160d
kube-node-lease   Active   160d
kube-public       Active   160d
kube-system       Active   160d
{{< / highlight >}}

now to create the resources:

{{< highlight bash >}}
$ terraform apply -auto-approve
kubernetes_pod.pod: Refreshing state... [id=default/pod]
kubernetes_namespace.webapplication-namespace: Creating...
kubernetes_namespace.webapplication-namespace: Creation complete after 0s [id=webapplication]
kubernetes_service.webapplication-service: Creating...
kubernetes_service.webapplication-service: Creation complete after 0s [id=default/webapplication-service]
kubernetes_deployment.webapplication-deploy: Creating...
kubernetes_deployment.webapplication-deploy: Creation complete after 4s [id=webapplication/webapplication]

Apply complete! Resources: 3 added, 0 changed, 0 destroyed.
{{< / highlight >}}

To confirm the creation:
{{< highlight bash >}}
$ kubectl get all --all-namespaces -l team=42
NAMESPACE        NAME                                  READY   STATUS    RESTARTS   AGE
webapplication   pod/webapplication-7f8849d748-pp4hf   1/1     Running   0          64s
webapplication   pod/webapplication-7f8849d748-q5xn2   1/1     Running   0          64s
webapplication   pod/webapplication-7f8849d748-qftfb   1/1     Running   0          64s

NAMESPACE   NAME                             TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
default     service/webapplication-service   NodePort   10.106.237.85   <none>        8080:31770/TCP   65s

NAMESPACE        NAME                             READY   UP-TO-DATE   AVAILABLE   AGE
webapplication   deployment.apps/webapplication   3/3     3            3           64s

NAMESPACE        NAME                                        DESIRED   CURRENT   READY   AGE
webapplication   replicaset.apps/webapplication-7f8849d748   3         3         3       64s

$ kubectl get ns
NAME              STATUS   AGE
default           Active   160d
kube-node-lease   Active   160d
kube-public       Active   160d
kube-system       Active   160d
webapplication    Active   106s
{{< / highlight >}}

As mentioned before Terraform is able to understand the relations between different resources, this will allow us to completely delete all the resources that we created with a single command even in cases where kubernetes would not delete by itself.
Suppose that we want to delete the webapplication namespace and all the resources that have any kind of association with it, if we run  ``kubectl delete ns webapplication`` command the namespace and the deployment will be deleted, however the service that was created in another namespace won't be affected:

{{< highlight bash >}}
$ kubectl get all --all-namespaces -l team=42
NAMESPACE        NAME                                  READY   STATUS    RESTARTS   AGE
webapplication   pod/webapplication-7f8849d748-5hs7v   1/1     Running   0          7m57s
webapplication   pod/webapplication-7f8849d748-nbqxf   1/1     Running   0          7m57s
webapplication   pod/webapplication-7f8849d748-t6wc2   1/1     Running   0          7m57s

NAMESPACE   NAME                             TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
default     service/webapplication-service   NodePort   10.106.212.15   <none>        8080:30498/TCP   8m8s

NAMESPACE        NAME                             READY   UP-TO-DATE   AVAILABLE   AGE
webapplication   deployment.apps/webapplication   3/3     3            3           8m8s

NAMESPACE        NAME                                        DESIRED   CURRENT   READY   AGE
webapplication   replicaset.apps/webapplication-7f8849d748   3         3         3       7m57s

$ kubectl delete ns webapplication
namespace "webapplication" deleted

$ kubectl get all --all-namespaces -l team=42
NAMESPACE   NAME                             TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
default     service/webapplication-service   NodePort   10.106.212.15   <none>        8080:30498/TCP   8m34s
{{< / highlight >}}

But on our Terraform code we explicit dependence between the namespace and the service, therefore it's possible to delete everything in one command:

{{< highlight bash >}}
$ kubectl get all --all-namespaces -l team=42
NAMESPACE        NAME                                  READY   STATUS    RESTARTS   AGE
webapplication   pod/webapplication-7f8849d748-5hs7v   1/1     Running   0          7m57s
webapplication   pod/webapplication-7f8849d748-nbqxf   1/1     Running   0          7m57s
webapplication   pod/webapplication-7f8849d748-t6wc2   1/1     Running   0          7m57s

NAMESPACE   NAME                             TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
default     service/webapplication-service   NodePort   10.106.212.15   <none>        8080:30498/TCP   8m8s

NAMESPACE        NAME                             READY   UP-TO-DATE   AVAILABLE   AGE
webapplication   deployment.apps/webapplication   3/3     3            3           8m8s

NAMESPACE        NAME                                        DESIRED   CURRENT   READY   AGE
webapplication   replicaset.apps/webapplication-7f8849d748   3         3         3       7m57s

$ terraform destroy -target=kubernetes_namespace.webapplication-namespace
kubernetes_namespace.webapplication-namespace: Refreshing state... [id=webapplication]

An execution plan has been generated and is shown below.
Resource actions are indicated with the following symbols:
  - destroy

Terraform will perform the following actions:

  # kubernetes_deployment.webapplication-deploy will be destroyed
  - resource "kubernetes_deployment" "webapplication-deploy" {
      - id               = "webapplication/webapplication" -> null
      - wait_for_rollout = true -> null

      - metadata {
          - generation       = 1 -> null
          - labels           = {
              - "app"  = "frontend"
              - "team" = "42"
            } -> null
          - name             = "webapplication" -> null
          - namespace        = "webapplication" -> null
          - resource_version = "1742605" -> null
          - self_link        = "/apis/apps/v1/namespaces/webapplication/deployments/webapplication" -> null
          - uid              = "c11ffac3-fe6c-49c6-9955-f7e9acbd6e87" -> null
        }

      - spec {
          - min_ready_seconds         = 0 -> null
          - paused                    = false -> null
          - progress_deadline_seconds = 600 -> null
          - replicas                  = 3 -> null
          - revision_history_limit    = 10 -> null

          - selector {
              - match_labels = {
                  - "app"  = "frontend"
                  - "team" = "42"
                } -> null
            }

          - strategy {
              - type = "RollingUpdate" -> null

              - rolling_update {
                  - max_surge       = "25%" -> null
                  - max_unavailable = "25%" -> null
                }
            }

          - template {
              - metadata {
                  - generation = 0 -> null
                  - labels     = {
                      - "app"  = "frontend"
                      - "team" = "42"
                    } -> null
                }

              - spec {
                  - active_deadline_seconds          = 0 -> null
                  - automount_service_account_token  = false -> null
                  - dns_policy                       = "ClusterFirst" -> null
                  - enable_service_links             = true -> null
                  - host_ipc                         = false -> null
                  - host_network                     = false -> null
                  - host_pid                         = false -> null
                  - restart_policy                   = "Always" -> null
                  - share_process_namespace          = false -> null
                  - termination_grace_period_seconds = 30 -> null

                  - container {
                      - image                      = "nginx:1.19.4-alpine" -> null
                      - image_pull_policy          = "IfNotPresent" -> null
                      - name                       = "frontend-app" -> null
                      - stdin                      = false -> null
                      - stdin_once                 = false -> null
                      - termination_message_path   = "/dev/termination-log" -> null
                      - termination_message_policy = "File" -> null
                      - tty                        = false -> null

                      - resources {
                        }
                    }
                }
            }
        }
    }

  # kubernetes_namespace.webapplication-namespace will be destroyed
  - resource "kubernetes_namespace" "webapplication-namespace" {
      - id = "webapplication" -> null

      - metadata {
          - annotations      = {
              - "name" = "webapplication"
            } -> null
          - generation       = 0 -> null
          - labels           = {
              - "team" = "42"
            } -> null
          - name             = "webapplication" -> null
          - resource_version = "1742545" -> null
          - self_link        = "/api/v1/namespaces/webapplication" -> null
          - uid              = "0d0e53a4-8d2d-4b31-8557-af680a54629a" -> null
        }
    }

  # kubernetes_service.webapplication-service will be destroyed
  - resource "kubernetes_service" "webapplication-service" {
      - id                    = "default/webapplication-service" -> null
      - load_balancer_ingress = [] -> null

      - metadata {
          - annotations      = {} -> null
          - generation       = 0 -> null
          - labels           = {
              - "app"  = "frontend"
              - "team" = "42"
            } -> null
          - name             = "webapplication-service" -> null
          - namespace        = "default" -> null
          - resource_version = "1742550" -> null
          - self_link        = "/api/v1/namespaces/default/services/webapplication-service" -> null
          - uid              = "5d35f7aa-f825-4e00-aec8-a76f13d80ce2" -> null
        }

      - spec {
          - cluster_ip                  = "10.152.183.63" -> null
          - external_ips                = [] -> null
          - external_traffic_policy     = "Cluster" -> null
          - health_check_node_port      = 0 -> null
          - load_balancer_source_ranges = [] -> null
          - publish_not_ready_addresses = false -> null
          - selector                    = {
              - "app" = "kubernetes_deployment.webapplication-deploy.metadata.labels.app"
            } -> null
          - session_affinity            = "None" -> null
          - type                        = "NodePort" -> null

          - port {
              - node_port   = 30127 -> null
              - port        = 8080 -> null
              - protocol    = "TCP" -> null
              - target_port = "80" -> null
            }
        }
    }

Plan: 0 to add, 0 to change, 3 to destroy.


Warning: Resource targeting is in effect

You are creating a plan with the -target option, which means that the result
of this plan may not represent all of the changes requested by the current
configuration.

The -target option is not for routine use, and is provided only for
exceptional situations such as recovering from errors or mistakes, or when
Terraform specifically suggests to use it as part of an error message.

Do you really want to destroy all resources?
  Terraform will destroy all your managed infrastructure, as shown above.
  There is no undo. Only 'yes' will be accepted to confirm.

  Enter a value: yes

kubernetes_service.webapplication-service: Destroying... [id=default/webapplication-service]
kubernetes_service.webapplication-service: Destruction complete after 0s
kubernetes_deployment.webapplication-deploy: Destroying... [id=webapplication/webapplication]
kubernetes_deployment.webapplication-deploy: Destruction complete after 0s
kubernetes_namespace.webapplication-namespace: Destroying... [id=webapplication]
kubernetes_namespace.webapplication-namespace: Still destroying... [id=webapplication, 10s elapsed]
kubernetes_namespace.webapplication-namespace: Destruction complete after 13s

Warning: Applied changes may be incomplete

The plan was created with the -target option in effect, so some changes
requested in the configuration may have been ignored and the output values may
not be fully updated. Run the following command to verify that no other
changes are pending:
    terraform plan

Note that the -target option is not suitable for routine use, and is provided
only for exceptional situations such as recovering from errors or mistakes, or
when Terraform specifically suggests to use it as part of an error message.


Destroy complete! Resources: 3 destroyed.

$ kubectl get all --all-namespaces -l team=42
No resources found

{{< / highlight >}}

As we could see in this post Terraform offers a low friction way to perform day-by-day operations on a Kubernetes Cluster and helps to simplify the workflow of the development and operations teams.
All the code that is shown in this post can be found at https://github.com/diegolakatos/blog-posts/tree/main/terraform-kubernetes

Hope you enjoyed this post.

Cheers