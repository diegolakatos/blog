---
date : '2026-02-12T16:10:09Z'
draft : true
title : 'Migrating From K3s to Talos Linux Pt.1'
Description : "How I migrate a single node k3s cluster to a multinode cluster using Talos Linux"
DisableComments : "true"
Tags : "[Kubernetes, Containers, HomeLab, Linux]"
Categories : [IaC]
language : en
---
I'm a big fan of homelabing and self-hosting. As an IT professional I think it is important to have a place where I can test new things and learn, specially those things that are not directly related with your current job. I also like to keep my personal stuff private and don't rely on big companies that constantly rug-pulls its customers, enshification is here to stay and will only get worse, without mentioning using our data to train AI models.

In my homelab most of the services where I keep valuable data, such as photos, has its unique virtual machine that may use containers to run the application (using a docker-compose file) or the application is installed using another method.
All services have its own certificate generated using Let's Encrypt and are accessible through Tailscale.
For services that are stateless or doesn't hold very important data I currently use a single node k3s "cluster". K3s is a Kubernetes distribuition that is lightweight and optimized for edge-computing, it works really well and I really recommended it for anyone that want to use Kubernetes without all the hasle of setting a more complex cluster, keeping in mind that k3s can be used in a multi-node deployment as well.

While I'm a big fan of the K.I.S.S philosophy at my work I like to play with scenarios that are not so simple to learn new things, Talos Linux is one of those things. Talos is a distribution made for running Kubernetes clusters, it is very minimal and secure by design, its configuration is done by using config files and the `talosctl` cli tool (ssh is not installed), that means that I can store all my configuration in a git repository, enabling a gitops approach and faster rebuild times if necessary.

In my current scenario my k3s node exposes the services using the built-in load balancer and traefik ingress controller, the applications are deployed using ArgoCD. The applications that needs to write on the disk can do that by using hostPath (which is not ideal but works well enough in this case), monitoring is made using a combination of Uptime Kuma and netdata.

The goals of this "project" are:
- Have a high available cluster using Talos Linux
- Proper monitoring using Prometheus + Loki, which will be used to monitor my entire homelab
- Use Longhorn as a CSI
- Replace Ingress with Gateway API
- Migrate all applications running on k3s
- Keep everything working as before

In this first post I will demonstrate how I installed Talos on my Proxmox cluster.

![1](/images/talos-pt1/1.png)