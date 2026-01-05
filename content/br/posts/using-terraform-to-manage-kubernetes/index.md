---
title: "Usando Terraform Para Gerenciar Kubernetes"
date: 2021-01-01T00:00:00Z
draft: false
DisableComments: true
Description: "Este post mostra como usar o Terraform para gerenciar clusters Kubernetes"
Tags: [Terraform, Kubernetes, IaC, Containers]
Categories: [IaC]
language: pt-br
---
Ao trabalhar com kubernetes, a maneira padrão de criar objetos tais quais *pods*, *deployments* e *services* é criar um arquivo de manifesto, normalmente em yaml, que descreva o estado desejado e, em seguida, usar ```kubectl``` para realizar a criação ou atualização o objeto.

Apesar de tudo, esse processo simples pode criar uma barreira para desenvolvedores e equipes de operações que agora precisam manter uma nova base de código em uma nova "linguagem". O YAML é fácil de ler, mas solucionar o problema pode ser difícil devido à necessidade de identificação correta e os erros exibidos pelo ```kubectl``` podem não ser facilmente compreendidos.
Se uma equipe já está usando o Terraform para criar e gerenciar a infraestrutura, faz sentido usá-lo para criar os objetos dentro do kubernetes, isso traz algumas vantagens, como:

- Um único fluxo de trabalho para administrar a infraestrutura subjacente e o próprio cluster
- Modelagem fácil sem a necessidade de usar ferramentas como o ``helm``,
- Gerenciamento do ciclo de vida sem a necessidade de verificar a API do Kubernetes
- O Terraform é capaz de entender a relação entre recursos, se um recurso tiver dependências que não conseguiram criar o Terraform entenderá isso e não criar o recurso ou se você excluir um objeto que possui dependências, as dependências também serão excluídas.

Feitas todas essas considerações, vamos começar a trabalhar. A primeira etapa é configurar o *provider* do kubernetes:

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

Agora, para criar um *pod*, podemos usar o seguinte arquivo:

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

Para aplicá-lo, basta executar:

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

Também é possível criar estruturas mais elaboradas, no próximo exemplo vamos explorar o grafo de dependências que o Terraform usa para gerenciar os recursos, para isso vamos criar 3 objetos: um namespace, um *deployment* e um *service*:

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

{{< / highlight >}}

Como todos os recursos têm os mesmos rótulos, podemos usar o comando ```kubectl``` para inspecionar o estado do cluster antes e depois do ``terraform apply``

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

agora para criar os objetos:

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

Para confirmar a criação:
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

Como mencionado antes, o Terraform é capaz de entender as relações entre diferentes recursos, isso nos permitirá excluir completamente todos os recursos que criamos com um único comando, mesmo nos casos em que o kubernetes não o faria.
Suponha que desejemos excluir o namespace do aplicativo da web e todos os recursos que têm qualquer tipo de associação com ele, se executarmos o comando ``kubectl delete ns webapplication`` o namespace e a implantação serão excluídos, no entanto, o serviço que foi criado em outro namespace não será afetado:

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

Mas em nosso código Terraform, temos uma dependência explícita  entre o *namespace* e o *service*, portanto, é possível excluir tudo em um comando:
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

Como pudemos ver nesta postagem, o Terraform oferece uma maneira simples de realizar operações diárias em um cluster do Kubernetes e ajuda a simplificar o fluxo de trabalho das equipes de desenvolvimento e operações.
Todos os códigos apresentados nesse post podem ser encontrados em https://github.com/diegolakatos/blog-posts/tree/main/terraform-kubernetes


Espero que você tenha gostado desse post.

Abraços
