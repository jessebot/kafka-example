# Kubeneters, GitOps, and Kafka, OH MY!
Just a quick example of how to set up a test k8s environment, using ArgoCD to test Kafka :)

## Installation of quick k8s cluster (KIND)
You can install this (kind)[https://kind.sigs.k8s.io/] with (brew)[https://brew.sh/] on Mac or Linux :D

```bash
brew install kind
```

*Note: You'll need (docker)[https://www.docker.com/get-started/] already installed and running.*

Then you can create a quick small cluster with the below command. It will create a cluster called kind, and it will have one node, but it will be fast, like no more than a few minutes. 

```bash
$ kind create cluster
Creating cluster "kind" ...
 ✓ Ensuring node image (kindest/node:v1.21.1) 🖼
 ✓ Preparing nodes 📦
 ✓ Writing configuration 📜
 ✓ Starting control-plane 🕹️
 ✓ Installing CNI 🔌
 ✓ Installing StorageClass 💾
Set kubectl context to "kind-kind"
You can now use your cluster with:

kubectl cluster-info --context kind-kind

Not sure what to do next? 😅  Check out https://kind.sigs.k8s.io/docs/user/quick-start/
```

*Note: make sure you have kubectl installed, and you can install that with brew as well*

You'll want to follow their advice and  get some important info with:
``` bash
$ kubectl cluster-info --context kind-kind
Kubernetes control plane is running at https://127.0.0.1:64067
CoreDNS is running at https://127.0.0.1:64067/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

$ kind get clusters
kind

$ kg namespaces
NAME                 STATUS   AGE
default              Active   27m
kube-node-lease      Active   27m
kube-public          Active   27m
kube-system          Active   27m
local-path-storage   Active   27m
```

Now that we've verified we have a local k8s cluster, let's get argo up and running! First we'll need helm (`brew install helm`), which is like a package manager for k8s. Then, if you want this to be repeatable, you can clone this repo because you'll need to create the `Chart.yaml` and `values.yaml` in `charts/argo`. Then you can run the following helm commands:

```bash
$ helm repo add argo https://argoproj.github.io/argo-helm
"argo" has been added to your repositories

$ helm dep update charts/argo/
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "argo" chart repository
Update Complete. ⎈Happy Helming!⎈
Saving 1 charts
Downloading argo-cd from repo https://argoproj.github.io/argo-helm
Deleting outdated charts
```

Then you should have ArgoCD on your Kind cluster :D

Special thanks to:
- https://www.arthurkoziel.com/setting-up-argocd-with-helm/
