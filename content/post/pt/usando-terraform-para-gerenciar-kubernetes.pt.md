---
title: "Usando Terraform Para Gerenciar Kubernetes"
date: 2020-12-19T11:16:44Z
draft: true
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

Conforme mencionado antes, o Terraform é capaz de entender as relações entre diferentes recursos, isso nos permitirá excluir completamente todos os recursos que criamos com um único comando, mesmo nos casos em que o kubernetes não excluiria por si mesmo.
Suponha que desejamos excluir todos os recursos que acabamos de criar usando o  ``kubectl``, se excluirmos o *namespace*, o *deployment* também será excluída porque um *deployment* não pode existir sem o *namespace*, entretanto o serviço não depende da existencia do namespace, portanto não seria excluído:

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

Como

In our case we have a declared dependency between the service that exposes our application and the namespace
