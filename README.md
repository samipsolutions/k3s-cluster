# k3s cluster backed by Flux (archived due to colo costs)

Highly opinionated template for deploying a single [k3s](https://k3s.io) cluster with [Ansible](https://www.ansible.com) and [Terraform](https://www.terraform.io) backed by [Flux](https://toolkit.fluxcd.io/) and [SOPS](https://toolkit.fluxcd.io/guides/mozilla-sops/).

The purpose here is to showcase how you can deploy an entire Kubernetes cluster and show it off to the world using the [GitOps](https://www.weave.works/blog/what-is-gitops-really) tool [Flux](https://toolkit.fluxcd.io/). When completed, your Git repository will be driving the state of your Kubernetes cluster. In addition with the help of the [Ansible](https://github.com/ansible-collections/community.sops), [Terraform](https://github.com/carlpett/terraform-provider-sops) and [Flux](https://toolkit.fluxcd.io/guides/mozilla-sops/) SOPS integrations you'll be able to commit [Age](https://github.com/FiloSottile/age) encrypted secrets to your public repo.

## Overview

- [Introduction](https://github.com/onedr0p/flux-cluster-template#-introduction)
- [Prerequisites](https://github.com/onedr0p/flux-cluster-template#-prerequisites)
- [Repository structure](https://github.com/onedr0p/flux-cluster-template#-repository-structure)
- [Lets go!](https://github.com/onedr0p/flux-cluster-template#-lets-go)
- [Post installation](https://github.com/onedr0p/flux-cluster-template#-post-installation)
- [Troubleshooting](https://github.com/onedr0p/flux-cluster-template#-troubleshooting)
- [What's next](https://github.com/onedr0p/flux-cluster-template#-whats-next)
- [Thanks](https://github.com/onedr0p/flux-cluster-template#-thanks)

## 👋 Introduction

The following components will be installed in your [k3s](https://k3s.io/) cluster by default. Most are only included to get a minimum viable cluster up and running.

- [flux](https://toolkit.fluxcd.io/) - GitOps operator for managing Kubernetes clusters from a Git repository
- [metallb](https://metallb.universe.tf/) - Load balancer for Kubernetes services
- [cert-manager](https://cert-manager.io/) - Operator to request SSL certificates and store them as Kubernetes resources
- [external-dns](https://github.com/kubernetes-sigs/external-dns) - Operator to publish DNS records to Cloudflare (and other providers) based on Kubernetes ingresses
- [ingress-nginx](https://kubernetes.github.io/ingress-nginx/) - Kubernetes ingress controller used for a HTTP reverse proxy of Kubernetes ingresses
- [local-path-provisioner](https://github.com/rancher/local-path-provisioner) - provision persistent local storage with Kubernetes

_Additional applications include [hajimari](https://github.com/toboshii/hajimari), [error-pages](https://github.com/tarampampam/error-pages), [echo-server](https://github.com/Ealenn/Echo-Server), [system-upgrade-controller](https://github.com/rancher/system-upgrade-controller), [reloader](https://github.com/stakater/Reloader), and [kured](https://github.com/weaveworks/kured)_

For provisioning the following tools will be used:

- [Fedora 36 Server](https://getfedora.org/en/server/download/) - Universal operating system that supports running all kinds of home related workloads in Kubernetes and has a faster release cycle
- [Ansible](https://www.ansible.com) - Provision Fedora Server and install k3s

## 📝 Prerequisites

**Note:** _This template has not been tested on cloud providers like AWS EC2, Hetzner, Scaleway etc... Those cloud offerings probably have a better way of provsioning a Kubernetes cluster and it's advisable to use those instead of the Ansible playbooks included here. This repository can still be tweaked for the GitOps/Flux portion if there's a cluster working in one those environments._

### 📚 Reading material

- [Organizing Cluster Access Using kubeconfig Files](https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/)

### 💻 Systems

- One or more nodes with a fresh install of [Fedora Server 36](https://getfedora.org/en/server/download/).
  - These nodes can be ARM64/AMD64 bare metal or VMs.
  - An odd number of control plane nodes, greater than or equal to 3 is required if deploying more than one control plane node.
- A [Cloudflare](https://www.cloudflare.com/) account with a domain, this will be managed by Terraform and external-dns. You can [register new domains](https://www.cloudflare.com/products/registrar/) directly thru Cloudflare.
- Some experience in debugging problems and a positive attitude ;)

📍 It is recommended to have 3 master nodes for a highly available control plane.

### 🔧 Workstation Tools

1. Install the **most recent versions** of the following CLI tools on your workstation, if you are using [Homebrew](https://brew.sh/) on MacOS or Linux skip to steps 3 and 4.

    * Required: [age](https://github.com/FiloSottile/age), [ansible](https://www.ansible.com), [flux](https://toolkit.fluxcd.io/), [go-task](https://github.com/go-task/task), [ipcalc](http://jodies.de/ipcalc), [jq](https://stedolan.github.io/jq/), [kubectl](https://kubernetes.io/docs/tasks/tools/), [pre-commit](https://github.com/pre-commit/pre-commit), [sops](https://github.com/mozilla/sops), [terraform](https://www.terraform.io), [yq v4](https://github.com/mikefarah/yq)

    * Recommended: [direnv](https://github.com/direnv/direnv), [helm](https://helm.sh/), [kustomize](https://github.com/kubernetes-sigs/kustomize), [prettier](https://github.com/prettier/prettier), [stern](https://github.com/stern/stern), [yamllint](https://github.com/adrienverge/yamllint)

2. This guide heavily relies on [go-task](https://github.com/go-task/task) as a framework for setting things up. It is advised to learn and understand the commands it is running under the hood.

3. Install [go-task](https://github.com/go-task/task) via Brew

    ```sh
    brew install go-task/tap/go-task
    ```

4. Install workstation dependencies via Brew

    ```sh
    task init
    ```

### ⚠️ pre-commit

It is advisable to install [pre-commit](https://pre-commit.com/) and the pre-commit hooks that come with this repository.
[sops-pre-commit](https://github.com/k8s-at-home/sops-pre-commit) will check to make sure you are not committing non-encrypted Kubernetes secrets to your repository.

1. Enable Pre-Commit

    ```sh
    task precommit:init
    ```

2. Update Pre-Commit, though it will occasionally make mistakes, so verify its results.

    ```sh
    task precommit:update
    ```

## 📂 Repository structure

The Git repository contains the following directories under `cluster` and are ordered below by how Flux will apply them.

```sh
📁 cluster      # k8s cluster defined as code
├─📁 flux       # flux, gitops operator, loaded before everything
├─📁 crds       # custom resources, loaded before 📁 core and 📁 apps
├─📁 charts     # helm repos, loaded before 📁 core and 📁 apps
├─📁 config     # cluster config, loaded before 📁 core and 📁 apps
├─📁 core       # crucial apps, namespaced dir tree, loaded before 📁 apps
└─📁 apps       # regular apps, namespaced dir tree, loaded last
```

## 🤝 Thanks

Big shout out to all the authors and contributors to the projects that we are using in this repository.

Community member [@Whazor](https://github.com/whazor) created [this website](https://whazor.github.io/k8s-at-home-search/) as a creative way to search Helm Releases across GitHub. You may use it as a means to get ideas on how to configure an applications' Helm values.
