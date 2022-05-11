# Kubernetes, ArgoCD, and Kafka
Just a quick example of how to set up a test Kuberentes (k8s) environment with KIND, and then using ArgoCD as our Continuous Delivery to test deploy a Kafka helm chart :) I'm writing this in mid-May 2022, and I make no promises for people trying to do legacy work, or proceed in the future. If it's not mid-May 2022, ymmv.

If you'd like, you can Learn more about the following before proceeding:
- [brew](https://brew.sh/) - Missing package manager for Mac (also supports Linux)
- [docker](https://www.docker.com/get-started/) - Containerization tooling
- [Kubernetes](https://kubernetes.io/docs/concepts/overview/what-is-kubernetes/) - [THE CLUSTER](media/peridot.png) we use to scale containers :3
- [KIND](https://kind.sigs.k8s.io/) - Tool to spin up mini k8s cluster
- [helm](https://helm.sh/docs/intro/quickstart/) - This is a package manager for kube apps (mostly a bunch of k8s yamls)
- [ArgoCD](https://argo-cd.readthedocs.io/en/stable/) - Continuous Delivery for k8s apps (4.6.0)
- [Kafka](https://kafka.apache.org/intro) - Handles real-time data feeds at scale

## Installation of quick k8s cluster (KIND)
You can install docker, kubernetes, kind, and helm with brew on Mac or Linux :D Check the links in the above section to learn more.

### brew (On MacOS/Linux)
```bash
brew install docker
brew install kubernetes
brew install kubectl
brew install kind
brew install helm
```

### apt (On Debian/Ubuntu)
```zsh
# install docker/kubectl deps
sudo apt-get -y install \
	apt-transport-https \
	ca-certificates \
	curl \
	gnupg-agent \
	software-properties-common

# download the gpg keys we need 
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
curl https://baltocdn.com/helm/signing.asc | sudo apt-key add -

# Add the repositories 
# Docker requires the appropriate ubuntu flavor for docker, in this case "focal".
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu focal stable"
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
echo "deb https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list

# update the package cache 
sudo apt-get -y update

# actually install something :D
sudo apt-get -y install \
	docker-ce \
	docker-ce-cli \
	containerd.io \
	kubectl \
	helm

# create a group for docker, and add the user to it for sudoless docker
sudo apt-get -y update
groupadd docker
sudo usermod -aG docker $USER
newgrp docker

#install KIND
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.13.0/kind-linux-amd64
chmod +x ./kind
mv ./kind /usr/bin/kind
```

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

You'll want to follow kind advice and get some important info with, as well as verify the cluster is good to go:
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

Now that we've verified we have a local k8s cluster, let's get Argo up and running!

## Installing ArgoCD

First we'll need helm (`brew install helm`, if you haven't already). Then, if you want this to be repeatable, you can clone this repo because you'll need to create the `Chart.yaml` and `values.yaml` in `charts/argo`. You can update your `version` parameter in `charts/argo/Chart.yaml` to the `version` param you see in [this repo](https://github.com/argoproj/argo-helm/blob/master/charts/argo-cd/Chart.yaml), at whatever time in the future that you're working on this. If you don't verify that version you will end up like me, half way down this article... :facepalm:

Then you can run the following helm commands:

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

The next thing you need to do do is install the chart with:

```bash
$ helm install argo-cd charts/argo/
```

### How to fix crd-install issue (Skip if no issue on `helm install`)
*Why and How*
You would see this:
```bash
$ helm install argo-cd charts/argo/
manifest_sorter.go:192: info: skipping unknown hook: "crd-install"
manifest_sorter.go:192: info: skipping unknown hook: "crd-install"
NAME: argo-cd
LAST DEPLOYED: Wed May 11 10:53:55 2022
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
```

You need those CRDs, or the pod will just crash loop, and then when you try to follow the next logical step of port-forwarding to test the frontend, you'll get something like this, when you actually test it:
```bash
$ kubectl port-forward svc/argo-cd-argocd-server 8080:443
Forwarding from 127.0.0.1:8080 -> 8080
Forwarding from [::1]:8080 -> 8080
Handling connection for 8080
```

Which seems fine, but when you go to http://localhost:8080 in your browser you'll see this in stdout in your terminal:
```
E0511 11:07:43.094956   46063 portforward.go:406] an error occurred forwarding 8080 -> 8080: error forwarding port 8080 to pod 53c2b12a3c748bb2c9acd763ed898c5261227ca4b359c047ec264608cbc67058, uid : failed to execute portforward in network namespace "/var/run/netns/cni-84865981-c6a2-6e6d-1ce1-336602591e41": failed to connect to localhost:8080 inside namespace "53c2b12a3c748bb2c9acd763ed898c5261227ca4b359c047ec264608cbc67058", IPv4: dial tcp4 127.0.0.1:8080: connect: connection refused IPv6 dial tcp6 [::1]:8080: connect: connection refused
E0511 11:07:43.095553   46063 portforward.go:234] lost connection to pod
Handling connection for 8080
E0511 11:07:43.096354   46063 portforward.go:346] error creating error stream for port 8080 -> 8080: EOF
```

This happens because you're using an older version of argoCD, and is apparently because of [this issue](https://github.com/bitnami/charts/issues/7972) and is fixed by [this](https://github.com/helm/helm/issues/6930), so you can just update your version.

*Fix*
Update `version` of your `charts/argo/Chart.yaml` to at least 4.6.0 (cause that's what worked for me :D)

Then you'll need to rerun the dep update:
```bash
helm dep update charts/argo/
```

Followed by uninstalling, and then reinstalling:
```bash
$ helm uninstall argo-cd
release "argo-cd" uninstalled
```

### Resume here after CRD issue detour
Now, for the perfect installation of our dreams:
```bash
$ helm install argo-cd charts/argo/
NAME: argo-cd
LAST DEPLOYED: Wed May 11 14:52:59 2022
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
```
:chef-kiss:

## Argo via the GUI
You'll need to test out the front end, but before you can do that, you need to do some port forwarding:
```bash
$ kubectl port-forward svc/argo-cd-argocd-server 8080:443
Forwarding from 127.0.0.1:8080 -> 8080
Forwarding from [::1]:8080 -> 8080
Handling connection for 8080
Handling connection for 8080
Handling connection for 8080
Handling connection for 8080
```

SUCCESS, we now get this in the browser:

<img src="media/argo_screenshot_2022-05-11_15.36.20.png" alt="Screenshot of the self-hosted-k8s ArgoCD login page in firefox" width="500"/>

The default username is admin. The password is auto-generated and we can get it with:
```bash
kubectl get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

## Getting Started with Kafka
Kafka will also be installed via helm, but this time with Argo. You'll want to have a git repo for kafka for this next part. In this case, I'm using this repo, but only to keep this managable as a tutorial.

Add the Bitnami helm repo:
```
$ helm repo add bitnami https://charts.bitnami.com/bitnami
```

Make sure you have the `charts/kafka/Chart.yaml` for this next part. Then update your chart:
```
$ helm dep update charts/kafka/
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "bitnami" chart repository
Update Complete. ⎈Happy Helming!⎈
Saving 1 charts
Downloading kafka from repo https://charts.bitnami.com/bitnami
Deleting outdated charts
```

That should put you to the point where you can now push this repo, and have argo sync it.

## Prepare kafka repo
For sake of simplicity and future automation, you can use the [github cli](https://cli.github.com/) to create a repo:
```
$ gh repo create argo-and-kafka-example
```

After it's created, since it's going to be public, we can connect to the repo using HTTPS, but if you plan on doing this with a private repo, you'll need to generate an SSH key pair to use as a deploy key in Github.

## Add kafka to ArgoCD

### GUI
Will fill this in later

### CLI

