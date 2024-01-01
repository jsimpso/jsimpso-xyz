+++
title = "Flux Rapid Bootstrapping"
date = 2023-12-12
[taxonomies]
tags = ["flux", "ci-cd"]
+++

# Flux: Rapid Bootstrap for testing

## Why

One of the tools I use to manage my home infrastructure is Flux. Lately I've been looking at setting up an automated system to test recovery scenarios, and part of that involves getting a new cluster up and running with Flux.

##  Let's do it

### Step 1: Prepare Kubernetes

First of all,  we'll need a kubernetes cluster to get Flux running in. Any cluster will do, I find the simplest way to get one up and running is with [MicroK8s](https://microk8s.io/).

From my example Ubuntu 22.04 system:

```
sudo snap install microk8s --classic
sudo microk8s status --wait-ready
sudo microk8s enable dns
```

Done! We now have a simple single-node cluster (as much as one node can constitute a cluster) ready to go.
While I'm here, I'll get the kubeconfig and put it in the expected place, so that flux can access it easily:
```
mkdir $HOME/.kube
sudo microk8s config > $HOME/.kube/config
```

### Step 2: Bootstrap Flux

Firstly, we need the flux CLI. The pre-built binaries are available on GitHub, you can get those manually via the [releases](https://github.com/fluxcd/flux2/releases) page. Or (assuming your platform is `linux_amd64` like mine):
```bash
# Download and extract directly to $HOME/bin. I keep $HOME/bin in my $PATH, adjust as required if you don't!
curl -s https://api.github.com/repos/fluxcd/flux2/releases/latest | jq '.assets | .[]|select(.name | contains("linux_amd64")) | .browser_download_url' | xargs wget -O - | tar -xzvf - -C $HOME/bin/
```

Verify:
```bash
jsimpso@demo:~$ $HOME/bin/flux -v
flux version 2.1.2
```

Success! Now we can bootstrap the cluster. I'm going to have flux use a public GitHub repository, and I want it to create the repo during bootstrapping. So per the [documentation](https://fluxcd.io/flux/installation/bootstrap/github/) I've created a GitHub PAT with "repo" permissions, and exported it as `GITHUB_TOKEN`.
```bash
flux bootstrap github \
  --token-auth \
  --owner=jsimpso \
  --repository=flux-demo \
  --branch=main \
  --path=clusters/microk8s \
  --personal \
  --private=false
```

A couple of minutes later, I see:
```
✔ helm-controller: deployment ready
✔ kustomize-controller: deployment ready
✔ notification-controller: deployment ready
✔ source-controller: deployment ready
✔ all components are healthy
```

And I can confirm flux has created the repo: 
```bash
jsimpso@demo:~$ git clone https://github.com/jsimpso/flux-demo/
Cloning into 'flux-demo'...
remote: Enumerating objects: 13, done.
remote: Counting objects: 100% (13/13), done.
remote: Compressing objects: 100% (6/6), done.
remote: Total 13 (delta 0), reused 13 (delta 0), pack-reused 0
Receiving objects: 100% (13/13), 36.00 KiB | 752.00 KiB/s, done.
jsimpso@demo:~$ cd flux-demo/
jsimpso@demo:~$/flux-demo$ tree
.
└── clusters
    └── microk8s
        └── flux-system
            ├── gotk-components.yaml
            ├── gotk-sync.yaml
            └── kustomization.yaml

4 directories, 3 files

```
