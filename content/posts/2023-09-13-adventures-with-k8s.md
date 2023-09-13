---
title: Adventures with k8s
date: 2023-09-13T09:04:30.231Z
---
In the interest of self-flagellation and modernizing my self-hosting setup I've begun moving everything I have to a Kubernetes cluster.

My goal is to have a fairly robust GitOps-based k8s setup, where I no longer have to SSH to a server and perform a number of manual tasks to update and maintain my services. There are of course a _lot_ of things to potentially add to such a setup, and this project will probably keep me busy for the next long while. But let's start small.

For now, I've settled on running a single-node [k3s](https://k3s.io/) cluster on a VM. For this I've chosen a [Hetzner CX31](https://www.hetzner.com/cloud) VPS.

My initial goal is as follows:

- Get k3s running on my VM
- Install [Flux](https://fluxcd.io/) and have it monitor my `home-ops` repo. 
- Set up ingress and LetsEncrypt certificates via [Traefik](https://traefik.io/) and [cert-manager](https://cert-manager.io/).
- Install [Weave GitOps](https://www.weave.works/product/gitops/), to have a neat GUI for managing Flux.
- Have Weave be reachable through an external hostname over HTTPS.

This should serve as a good, solid PoC for running my own cluster with GitOps.

## Installing k3s
The VM has been provisioned and I'm ready to install k3s. K3s deploys its own instance of Traefik by default, so we'll have to disable that as we want to control that part ourselves. The entire install process can be accomplished with this scary `curl`/`sh` oneliner:
```shell
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="server --disable traefik" sh -
```

Once the install is complete, a `kubeconfig` file will have been generated for us, that allows us to reach and manage the cluster from the outside, located at `/etc/rancher/k3s/k3s.yaml`. We just need to `scp` this back home to our own machine and change the IP to point to the external IP of our VM.

```shell
❯ kubectl get pods -A
NAMESPACE      NAME                                       READY   STATUS    RESTARTS   AGE
kube-system    local-path-provisioner-957fdf8bc-knfmv     1/1     Running   0          25h
kube-system    coredns-77ccd57875-pvpbb                   1/1     Running   0          25h
kube-system    metrics-server-648b5df564-lnhws            1/1     Running   0          25h
```

It works. 🎈

Fow now (until we get around to setting up some monitoring of the VM) we shouldn't need to access the VM directly again.

## Installing Flux
We're using Flux for GitOps. Flux will monitor a git repository and reconcile whatever is defined there with what is running in the cluster.  For this we'll naturally need a git repository. I've decided to just start from scratch with this, taking inspiration from the [Flux recommended ways](https://fluxcd.io/flux/guides/repository-structure/) of structuring a repo.

Flux is able to manage itself and its own configuration, but this presents us with a chicken-and-egg problem. Before this runs automatically there are a few manual steps we need to perform.  Enter `flux bootstrap`.

Flux has a CLI that can be used to manage Flux in a cluster, and perform [initial bootstrapping](https://fluxcd.io/flux/installation/bootstrap/gitlab/). The following command will install the necessary controllers and CRDs in the cluster, commit the corresponding manifests to git, set up Flux to monitor the repository and then finally wait for Flux to reconcile its own state.

```shell
❯ flux bootstrap gitlab \
        --owner=cmoesgaard \
        --repository=home-ops \
        --branch=flux-test \
        --path=./kubernetes/flux \
        --personal
```

- Bootstrap Flux
- Generate `age` key and create secret in cluster by hand