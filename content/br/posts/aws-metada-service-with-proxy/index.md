---
title: "Nem sempre é DNS, diversão com proxies, AWS IAM roles e uma pitada de Terraform"
date: 2022-02-12T12:33:53Z
Description: "A historia de uma investigação"
draft: false
DisableComments: true
Tags: [Terraform, AWS, IaC, Linux]
Categories: [IaC]
language: pt-br
---
Há um tempo atrás, enquanto trabalhava em um pipeline que seria usado para criar recursos na AWS, enfrentei uma situação inusitada que me levou a uma investigação bastante interessante.

O cenário era bem simples, uma instância do EC2 atuando como um [gitlab-runner](https://docs.gitlab.com/runner/) executando um contêiner Docker que baixaria um código Terraform do nosso repositório e executaria `terraform apply`. A instância do EC2 tinha uma IAM role anexada com todas as permissões necessárias para criar os recursos definidos no código do Terraform.

Quando tudo estava pronto, o pipeline foi acionado e assim que o `terraform apply` começou a ser executado...

*Observação: todos os trechos e exemplos abaixo foram criados em minha própria conta da AWS e usando meu próprio código. Tentei recriar meu processo de pensamento e ações da melhor maneira possível.*

{{< highlight bash "linenos=false,linenostart=1" >}}
│ Error: error configuring Terraform AWS Provider: no valid credential sources for Terraform AWS Provider found.
│
│ Please see https://registry.terraform.io/providers/hashicorp/aws
│ for more information about providing credentials.
│
│ Error: no EC2 IMDS role found, operation error ec2imds: GetMetadata, http response error StatusCode: 404,
| request to EC2 IMDS failed
│
│
│   with provider["registry.terraform.io/hashicorp/aws"],
│   on s3.tf line 1, in provider "aws":
│    1: provider "aws"{
│
{{< / highlight >}}
Que estranho! O terraform está recebendo um 404 ao acessar os metadados da instância, que são usados para recuperar informações sobre a instância, como por exemplo a role anexada a ela, os metadado pode ser acessado no endereço *http://169.254.169.254*.
Verifiquei novamente o provedor do Terraform e sua configuração não poderia ser mais simples.
{{< highlight hcl "linenos=false,linenostart=1" >}}
provider "aws"{
region = "us-east-1"
}
{{< / highlight >}}
Na hora de investigar, decidi primeiro verificar se tudo estava funcionando na instância antes de passar para o contêiner docker, então eu acessei a instância e executei alguns comandos para verificar a conectividade com o serviço de metadados e também para garantir que a role estava configurada corretamente:
{{< highlight bash "linenos=false,linenostart=1" >}}
[ec2-user@ip-172-31-67-102 ~]$ curl http://169.254.169.254/latest/meta-data
ami-id
ami-launch-index
ami-manifest-path
block-device-mapping/
events/
hostname
iam/
identity-credentials/
instance-action
instance-id
instance-life-cycle
instance-type
local-hostname
local-ipv4
mac
metrics/
network/
placement/
profile
public-hostname
public-ipv4
public-keys/
reservation-id
security-groups
services/
[ec2-user@ip-172-31-67-102 ~]$ aws s3 ls
2021-10-29 10:04:16 lakatos-cassandra
2021-03-29 16:40:08 terraform-lakatos-state-3
{{< / highlight >}}

Tudo parece bom, então deixe-me tentar usar o terraform para criar algo na AWS:
{{< highlight hcl "linenos=false,linenostart=1" >}}
provider "aws"{
region = "us-east-1"
}

resource "aws_s3_bucket" "b" {
  bucket = "lakatos-my-tf-test-bucket"
}
{{< / highlight >}}
{{< highlight bash "linenos=false,linenostart=1" >}}
[ec2-user@ip-172-31-67-102 ~]$ terraform apply --auto-approve

Terraform used the selected providers to generate the following execution plan. Resource actions are indicated
with the following symbols:
  + create

Terraform will perform the following actions:

  # aws_s3_bucket.b will be created
  + resource "aws_s3_bucket" "b" {
      + acceleration_status                  = (known after apply)
      + acl                                  = (known after apply)
      + arn                                  = (known after apply)
      + bucket                               = "lakatos-my-tf-test-bucket"
      + bucket_domain_name                   = (known after apply)
      + bucket_regional_domain_name          = (known after apply)
      + cors_rule                            = (known after apply)
      + force_destroy                        = false
      + grant                                = (known after apply)
      + hosted_zone_id                       = (known after apply)
      + id                                   = (known after apply)
      + lifecycle_rule                       = (known after apply)
      + logging                              = (known after apply)
      + policy                               = (known after apply)
      + region                               = (known after apply)
      + replication_configuration            = (known after apply)
      + request_payer                        = (known after apply)
      + server_side_encryption_configuration = (known after apply)
      + tags_all                             = (known after apply)
      + versioning                           = (known after apply)
      + website                              = (known after apply)
      + website_domain                       = (known after apply)
      + website_endpoint                     = (known after apply)

      + object_lock_configuration {
          + object_lock_enabled = (known after apply)
          + rule                = (known after apply)
        }
    }

Plan: 1 to add, 0 to change, 0 to destroy.
aws_s3_bucket.b: Creating...
aws_s3_bucket.b: Creation complete after 1s [id=lakatos-my-tf-test-bucket]

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
[ec2-user@ip-172-31-67-102 ~]$ aws s3 ls
2021-10-29 10:04:16 lakatos-cassandra
2022-02-13 13:06:02 lakatos-my-tf-test-bucket
2021-03-29 16:40:08 terraform-lakatos-state-3
{{< / highlight >}}
Isso também funcionou, talvez seja algo errado com o código que está passando pelo pipeline, para puxar o código do repositório tive que configurar um proxy usando as variáveis `HTTP_PROXY` e `HTTP_PROXY`.
{{< highlight bash "linenos=false,linenostart=1" >}}
[ec2-user@ip-172-31-67-102 ~]$ export HTTPS_PROXY=172.31.94.221:3128
[ec2-user@ip-172-31-67-102 ~]$ export HTTP_PROXY=172.31.94.221:3128
{{< / highlight >}}
O pipeline também foi configurado para usar proxy, caso contrário não seria possível baixar o código do repositório.
Eu clonei o repositório e tentei executar o `terraform apply` novamente:
{{< highlight bash "linenos=false,linenostart=1" >}}
│ Error: error configuring Terraform AWS Provider: no valid credential sources for Terraform AWS Provider found.
│
│ Please see https://registry.terraform.io/providers/hashicorp/aws
│ for more information about providing credentials.
│
│ Error: no EC2 IMDS role found, operation error ec2imds: GetMetadata, http response error StatusCode: 404,
| request to EC2 IMDS failed
│
│
│   with provider["registry.terraform.io/hashicorp/aws"],
│   on s3.tf line 1, in provider "aws":
│    1: provider "aws"{
│
{{< / highlight >}}
Não era isso que eu esperava... Vamos rodar `curl` e `aws s3 ls` novamente:
{{< highlight bash "linenos=false,linenostart=1" >}}
[ec2-user@ip-172-31-67-102 ~]$ curl http://169.254.169.254/latest/meta-data
ami-id
ami-launch-index
ami-manifest-path
block-device-mapping/
events/
hostname
iam/
identity-credentials/
instance-action
instance-id
instance-life-cycle
instance-type
local-hostname
local-ipv4
mac
metrics/
network/
placement/
profile
public-hostname
public-ipv4
public-keys/
reservation-id
security-groups
services/
[ec2-user@ip-172-31-67-102 ~]$ aws s3 ls
Unable to locate credentials. You can configure credentials by running "aws configure".
{{< / highlight >}}

Isso é estranho... vamos investigar um pouco mais:

*Eu omiti algumas linhas das seguintes para maior clareza*
{{< highlight bash "linenos=false,linenostart=1" >}}
[ec2-user@ip-172-31-67-102 ~]$ aws s3 ls --debug

2022-02-13 18:14:21,107 - MainThread - botocore.credentials - DEBUG - Looking for credentials via: env
2022-02-13 18:14:21,107 - MainThread - botocore.credentials - DEBUG - Looking for credentials via: assume-role
2022-02-13 18:14:21,107 - MainThread - botocore.credentials - DEBUG - Looking for credentials via: assume-role-with-web-identity
2022-02-13 18:14:21,107 - MainThread - botocore.credentials - DEBUG - Looking for credentials via: sso
2022-02-13 18:14:21,107 - MainThread - botocore.credentials - DEBUG - Looking for credentials via: shared-credentials-file
2022-02-13 18:14:21,107 - MainThread - botocore.credentials - DEBUG - Looking for credentials via: custom-process
2022-02-13 18:14:21,107 - MainThread - botocore.credentials - DEBUG - Looking for credentials via: config-file
2022-02-13 18:14:21,107 - MainThread - botocore.credentials - DEBUG - Looking for credentials via: ec2-credentials-file
2022-02-13 18:14:21,108 - MainThread - botocore.credentials - DEBUG - Looking for credentials via: boto-config
2022-02-13 18:14:21,108 - MainThread - botocore.credentials - DEBUG - Looking for credentials via: container-role
2022-02-13 18:14:21,108 - MainThread - botocore.credentials - DEBUG - Looking for credentials via: iam-role
2022-02-13 18:14:21,109 - MainThread - urllib3.connectionpool - DEBUG - Starting new HTTP connection (1): 172.31.94.221:3128
2022-02-13 18:14:21,112 - MainThread - urllib3.connectionpool - DEBUG - http://172.31.94.221:3128 "PUT http://169.254.169.254/latest/api/token HTTP/1.1" 403 0
2022-02-13 18:14:21,114 - MainThread - urllib3.connectionpool - DEBUG - http://172.31.94.221:3128 "GET http://169.254.169.254/latest/meta-data/iam/security-credentials/ HTTP/1.1" 404 339
2022-02-13 18:14:21,115 - MainThread - botocore.utils - DEBUG - Metadata service returned non-200 response with status code of 404 for url: http://169.254.169.254/latest/meta-data/iam/security-credentials/, content body: <?xml version="1.0" encoding="iso-8859-1"?>

{{< / highlight >}}
Podemos ver que a AWS CLI está tentando recuperar credenciais seguindo a ordem de precedência explicada na [documentação da AWS](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-quickstart.html) e recebe um 404, mesmo erro que é reportado pelo Terraform, ao tentar recuperar o iam-role, podemos ver que a AWS CLI está usando o proxy que foi configurado anteriormente e que tal `curl`?
{{< highlight bash "linenos=false,linenostart=1" >}}
[ec2-user@ip-172-31-67-102 ~]$ curl http://169.254.169.254/latest/meta-data/iam/info -v
*   Trying 169.254.169.254:80...
* Connected to 169.254.169.254 (169.254.169.254) port 80 (#0)
> GET /latest/meta-data/iam/info HTTP/1.1
> Host: 169.254.169.254
> User-Agent: curl/7.79.1
> Accept: */*
>
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< Content-Type: text/plain
< Accept-Ranges: none
< Last-Modified: Sun, 13 Feb 2022 13:17:17 GMT
< Content-Length: 200
< Date: Sun, 13 Feb 2022 13:37:57 GMT
< Server: EC2ws
< Connection: close
<
{
  "Code" : "Success",
  "LastUpdated" : "2022-02-13T13:17:17Z",
  "InstanceProfileArn" : "arn:aws:iam::12345678901011:instance-profile/S3AdminAccess",
  "InstanceProfileId" : "AIPAVMQT32F2Q43IXV2TC"
* Closing connection 0
}
{{< / highlight >}}
Parece que não está usando o proxy depois de pensar um pouco, lembrei que o curl não usa as variáveis `HTTP_PROXY` e `HTTPS_PROXY` em maiúsculas (mais detalhes podem ser encontrados [aqui](https://everything.curl.dev/usingcurl /proxies/env)), então configurei para minúsculas e tentei novamente:
{{< highlight bash "linenos=false,linenostart=1" >}}
[ec2-user@ip-172-31-67-102 ~]$ curl http://169.254.169.254/latest/meta-data/iam/info -v
* Uses proxy env variable http_proxy == '172.31.94.221:3128'
*   Trying 172.31.94.221:3128...
* Connected to 172.31.94.221 (172.31.94.221) port 3128 (#0)
> GET http://169.254.169.254/latest/meta-data/iam/info HTTP/1.1
> Host: 169.254.169.254
> User-Agent: curl/7.79.1
> Accept: */*
> Proxy-Connection: Keep-Alive
>
* Mark bundle as not supporting multiuse
< HTTP/1.1 404 Not Found
< Content-Type: text/html
< Content-Length: 339
< Date: Sun, 13 Feb 2022 18:16:12 GMT
< Server: EC2ws
< X-Cache: MISS from SquidProxy
< X-Cache-Lookup: MISS from SquidProxy:3128
< Via: 1.1 SquidProxy (squid/4.10)
< Connection: keep-alive
<
<?xml version="1.0" encoding="iso-8859-1"?>
<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Transitional//EN"
		 "http://www.w3.org/TR/xhtml1/DTD/xhtml1-transitional.dtd">
<html xmlns="http://www.w3.org/1999/xhtml" xml:lang="en" lang="en">
 <head>
  <title>404 - Not Found</title>
 </head>
 <body>
  <h1>404 - Not Found</h1>
 </body>
</html>
* Connection #0 to host 172.31.94.221 left intact

{{< / highlight >}}
Sucesso!! Quer dizer... falha =)

Agora sabemos que o proxy pode ser a causa do problema, não quero desconfigurar o proxy porque vou precisar dele mais tarde no meu pipeline, em vez disso, posso apenas definir os endereços que quero acessar sem passar pelo proxy:

{{< highlight bash "linenos=false,linenostart=1" >}}
[ec2-user@ip-172-31-67-102 ~]$ export NO_PROXY=169.254.169.254
[ec2-user@ip-172-31-67-102 ~]$ export no_proxy=169.254.169.254
[ec2-user@ip-172-31-67-102 ~]$ curl http://169.254.169.254/latest/meta-data/iam/info
{
  "Code" : "Success",
  "LastUpdated" : "2022-02-13T13:17:17Z",
  "InstanceProfileArn" : "arn:aws:iam::12345678901011:instance-profile/S3AdminAccess",
  "InstanceProfileId" : "AIPAVMQT32F2Q43IXV2TC"
}
[ec2-user@ip-172-31-67-102 ~]$ aws s3 ls
2021-10-29 10:04:16 lakatos-cassandra
2021-03-29 16:40:08 terraform-lakatos-state-3

ec2-user@ip-172-31-67-102 ~]$ terraform apply --auto-approve

Terraform used the selected providers to generate the following execution plan. Resource actions are indicated
with the following symbols:
  + create

Terraform will perform the following actions:

  # aws_s3_bucket.b will be created
  + resource "aws_s3_bucket" "b" {
      + acceleration_status                  = (known after apply)
      + acl                                  = (known after apply)
      + arn                                  = (known after apply)
      + bucket                               = "lakatos-my-tf-test-bucket"
      + bucket_domain_name                   = (known after apply)
      + bucket_regional_domain_name          = (known after apply)
      + cors_rule                            = (known after apply)
      + force_destroy                        = false
      + grant                                = (known after apply)
      + hosted_zone_id                       = (known after apply)
      + id                                   = (known after apply)
      + lifecycle_rule                       = (known after apply)
      + logging                              = (known after apply)
      + policy                               = (known after apply)
      + region                               = (known after apply)
      + replication_configuration            = (known after apply)
      + request_payer                        = (known after apply)
      + server_side_encryption_configuration = (known after apply)
      + tags_all                             = (known after apply)
      + versioning                           = (known after apply)
      + website                              = (known after apply)
      + website_domain                       = (known after apply)
      + website_endpoint                     = (known after apply)

      + object_lock_configuration {
          + object_lock_enabled = (known after apply)
          + rule                = (known after apply)
        }
    }

Plan: 1 to add, 0 to change, 0 to destroy.
aws_s3_bucket.b: Creating...
aws_s3_bucket.b: Creation complete after 1s [id=lakatos-my-tf-test-bucket]

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
[ec2-user@ip-172-31-67-102 ~]$ aws s3 ls
2021-10-29 10:04:16 lakatos-cassandra
2022-02-13 13:06:02 lakatos-my-tf-test-bucket
2021-03-29 16:40:08 terraform-lakatos-state-3
{{< / highlight >}}

Finalmente está funcionando!!

É sempre incrível como coisas simples podem levar a comportamentos estranhos e, claro, tudo isso poderia ser evitado se eu lesse a [documentação da AWS](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-proxy.html)
![Debug](/images/debug.jpeg)