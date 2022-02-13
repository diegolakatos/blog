---
title: "It's not always DNS, having fun with proxies, AWS IAM Roles and a pinch of Terraform"
date: 2022-02-12T12:33:53Z
Description: ""
Tags: []
Categories: []
DisableComments: false
draft: true
---
While ago while working in a pipeline that was going to be used to create resources on AWS I faced a strange situation that led me to an interesting troubleshooting path.

The scenario was quite simple, an EC2 instance acting as a [gitlab-runner](https://docs.gitlab.com/runner/) executing a docker container that would download some terraform code from our git repository and run `terraform apply`. The EC2 instance had an IAM role attached that with all the necessary permissions to create the resources defined on the Terraform code.

When everything was ready to go the pipeline was triggered and as soon `terraform apply` started to run...

*Note: all snippets and examples below were created on my own AWS account and using my own code. I tried to recreate my thought process and actions as best as I could.*
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
That's weird terraform is receiving a 404 when calling the instance metadata, that is used to retrieve information about roles attached to the instance and other useful information, it can be accessed on the address *http://169.254.169.254*.

I double-checked the Terraform's provider, and its configuration couldn't be simpler.
{{< highlight hcl "linenos=false,linenostart=1" >}}
provider "aws"{
region = "us-east-1"
}
{{< / highlight >}}
Time to troubleshoot, I decided to first verify if everything was working on the instance before moving to the docker container, so I ssh into the instance and run some commands to check the connectivity with the metadata service and also to make sure that the role was correctly configured:

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

Everything looks good so let me try to use terraform to create something on aws:
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
That also worked, maybe is something wrong with the code that is going through the pipeline, to pull the code from the repository I had to configure a proxy using the `HTTP_PROXY` and `HTTP_PROXY` variables.
{{< highlight bash "linenos=false,linenostart=1" >}}
[ec2-user@ip-172-31-67-102 ~]$ export HTTPS_PROXY=172.31.94.221:3128
[ec2-user@ip-172-31-67-102 ~]$ export HTTP_PROXY=172.31.94.221:3128
{{< / highlight >}}
Note that the pipeline uses the proxy to download the code
I cloned the repository and tried to run `terraform apply` again:
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
That's not what I expected.... Let's run `curl` and `aws s3 ls` again:
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
The `aws s3 ls` command took some seconds before failing, while `curl` responded almost immediately.
Let's investigate a little further:

*I omitted some lines of the following outputs for clarity*
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
We can see that the AWS CLI is trying to retrieve credentials following the precedence order explained on the [aws documentation](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-quickstart.html) and receives a 404, same error that is reported by Terraform, when tries to retrieve the iam-role, we can see that the AWS CLI is using the proxy that was previously configured and how about `curl`?
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
Looks like it's not using the proxy after some thinking I remembered that curl don't use the variables `HTTP_PROXY` and `HTTPS_PROXY` in uppercase (more details can be found [here](https://everything.curl.dev/usingcurl/proxies/env)), so I set it to lowercase and tried again:
{{< highlight bash "linenos=false,linenostart=1" >}}
[ec2-user@ip-172-31-67-102 ~]$export http_proxy=172.31.94.221:3128
[ec2-user@ip-172-31-67-102 ~]$ export https_proxy=172.31.94.221:3128
[ec2-user@ip-172-31-67-102 ~]$ curl http://169.254.169.254/latest/meta-data/iam/info -vv
* Uses proxy env variable http_proxy == '172.31.94.221:3128'
*   Trying 172.31.94.221:3128...
* connect to 172.31.94.221 port 3128 failed: Connection timed out
* Failed to connect to 172.31.94.221 port 3128 after 131016 ms: Connection timed out
* Closing connection 0
curl: (28) Failed to connect to 172.31.94.221 port 3128 after 131016 ms: Connection timed out
{{< / highlight >}}
Success!! I mean... fail =)

Now we know that the proxy could be the cause of the problem, I don't want to unset the proxy because I'll need it later in my pipeline so let's define some address that we don't want to go through the proxy and try again:

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

Finally it is working as was intended.

It's always amazing how simple things can lead to strange behaviours, and of course all of that could be avoided if I read the [aws documentation](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-proxy.html)
![Debug](/images/debug.jpeg)