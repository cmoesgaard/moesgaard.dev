---
title: Adventures with k8s
date: 2023-09-13T09:04:30.231Z
---
In the interest of self-flagellation and modernizing my self-hosting setup I've begun moving everything I have to a Kubernetes cluster.

My goal is to have a fairly robust GitOps-based k8s setup, where I no longer have to SSH to a server and perform a number of manual tasks to update and maintain my services. There are of course a *lot* of things to potentially add to such a setup, and this project will probably keep me busy for the next long while. But let's start small.

For now, I've settled on running a single-node [k3s](https://k3s.io/) cluster on a VM. For this I've chosen a [Hetzner CX31](https://www.hetzner.com/cloud) VPS.

My initial goal is as follows:

* Get k3s running on my VM
* Install [Flux](https://fluxcd.io/) and have it monitor my `home-ops` repo. 
* Set up ingress and LetsEncrypt certificates via [Traefik](https://traefik.io/) and [cert-manager](https://cert-manager.io/).
* Install [Weave GitOps](https://www.weave.works/product/gitops/), to have a neat GUI for managing Flux.
* Have Weave be reachable through an external hostname over HTTPS.

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
        --branch=main \
        --path=./kubernetes/flux \
        --personal
```

Checking `flux stats` will show us that everything looks to be in order.

```shell
❯ flux stats
RECONCILERS          	RUNNING	FAILING	SUSPENDED	STORAGE
GitRepository        	1      	0      	0        	40.9 KiB 	
OCIRepository        	0      	0      	0        	-        	
HelmRepository       	0      	0      	0        	-	
HelmChart            	0      	0      	0        	-	
Bucket               	0      	0      	0        	-        	
Kustomization        	1      	0      	0        	-        	
HelmRelease          	0      	0      	0        	-        	
Alert                	0      	0      	0        	-        	
Provider             	0      	0      	0        	-        	
Receiver             	0      	0      	0        	-        	
ImageUpdateAutomation	0      	0      	0        	-        	
ImagePolicy          	0      	0      	0        	-        	
ImageRepository      	0      	0      	0        	-        	
```

From now on, everything merged to the `main` branch in the repo will now be reconciled by Flux.

### Secret management

With GitOps we get a lovely paper trail regarding changes to our cluster, with every update or change corresponding to a git commit. We'd like to also have a way to store the various secrets we need as part of this, but without exposing every password and API token to the world. For this, we'll use [SOPS](https://github.com/getsops/sops). SOPS allows us to store our secrets as encrypted files, so that they can be decrypted and read by us and by Flux in the cluster, but not by third parties snooping around my repository. Flux supports SOPS for decrypting secrets [out of the box](https://fluxcd.io/flux/guides/mozilla-sops/).

To make this work, we'll need to perform _another_ manual step. We'll need to generate a set of encryption keys using `age`:

```shell
❯ age-keygen -o age.agekey
Public key: age1vt4lmr873png75lskfhz9ymh29wvnr3gydgzea6w8r7wp4al54lsdhw08a
```

And then store the private key as a secret in the cluster.

```shell
❯ cat age.agekey |
kubectl create secret generic sops-age \
--namespace=flux-system \
--from-file=age.agekey=/dev/stdin
```

To allow SOPS to decrypt the files, the `age.agekey` must be stored in `~/.config/sops/age` as `keys.txt` (or added to this file if it exists)

We only want SOPS to encrypt specific values inside our files, so we add a `.sops.yaml` file to the root of the repo containing the following, substituting the appropriate public key:

```yaml
---
creation_rules:
  - path_regex: kubernetes/.*\.yaml
    encrypted_regex: "^(data|stringData)$"
    age: age1vt4lmr873png75lskfhz9ymh29wvnr3gydgzea6w8r7wp4al54lsdhw08a
```

SOPS can now be used in the following way to encrypt secrets in-place:

```shell
❯ sops -i -e cluster-secrets.yaml
```
Turning a secret like this:

```yaml
---
apiVersion: v1
kind: Secret
metadata:
    name: cluster-secrets
    namespace: flux-system
stringData:
    PASSWORD: hunter2
```

Into this:

```yaml
---
apiVersion: v1
kind: Secret
metadata:
    name: cluster-secrets
    namespace: flux-system
stringData:
    PASSWORD: ENC[AES256_GCM,data:VYLeI8Ft,iv:A2Ph7+1qOwcfofRt5HLYCfSMQRnQKVz9fn6Jbm9ENO4=,tag:FH8V2kYjNLRZ7mSywKkVKA==,type:str]
sops:
    kms: []
    gcp_kms: []
    azure_kv: []
    hc_vault: []
    age:
        - recipient: age1vt4lmr873png75lskfhz9ymh29wvnr3gydgzea6w8r7wp4al54lsdhw08a
          enc: |
            -----BEGIN AGE ENCRYPTED FILE-----
            YWdlLWVuY3J5cHRpb24ub3JnL3YxCi0+IFgyNTUxOSB5WnY3UEt0bm5HektpUFh6
            NU1ZbzdmTWRialFVeVZEZ01GdW44SUE5OUFrCndWdDR4djE3bjVUZHFwNmozSmZL
            R0xheXcwSE9sMmpmVHZPZmNhaFNuaXMKLS0tIGtJLzVXRlZMR2ZERGY1VzZLUzQ0
            Z1lxbXJNazdFN2I2MlBZbEg0dzRVNncKoT1oK/yakATXRaEHaeuDKvjL+vnxm/IC
            wYdzbRJnPU2q/ssueGhIT2vjynniTMC8yoKJLEU/nn2/z7kPLokEsg==
            -----END AGE ENCRYPTED FILE-----
    lastmodified: "2023-09-13T11:21:32Z"
    mac: ENC[AES256_GCM,data:p3rVoUXsqWbV6K6dKXtCTDcFvw3I9wijPNwJ1hauivoobjlDEbxtW/jksZaqWnlUH0K1EI37P9Gq49wDT8piyeiIVLZ+KvHeiP24/0JXf9i0+Tk2edgT6TIz2Z5jCP84flcFNWu0IT91eoYQqt1Gu5Np33517QJUC6J8jPY0C6k=,iv:4EPEAgSgCFXdtucSxznvORXYVy91LEAaXAIC+NTKeHA=,tag:J0R6gg5E9RSpPkx9CNc4tA==,type:str]
    pgp: []
    encrypted_regex: ^(data|stringData)$
    version: 3.7.3
```

Opening the above file with `sops cluster-secrets.yaml` will then allow us to modify the decrypted secret.

Finally, when the `Secret` above is created in the cluster, we add the following snippet to the relevant `Kustomization`, to tell Flux to decrypt the secret using SOPS and the private key we stored previously:

```yaml
  ...
  decryption:
    provider: sops
    secretRef:
      name: sops-age
  ...
```

