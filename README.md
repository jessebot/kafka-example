# Kubernetes, ArgoCD, Vault, and Kafka
Just a quick example of how to set up a test Kuberentes (k8s) environment with KIND, and then using ArgoCD as our Continuous Delivery to test deploy a Kafka helm chart. Uses Hashicorp's Vault for secret storage :)

# Tech stack and Dependencies
## Package Managers
- [brew](https://brew.sh/) - Missing package manager for Mac (also supports Linux)
  - Also got some scripts for Debian distros
- [helm2](https://helm.sh/docs/intro/quickstart/) installs k8s apps (mostly a bunch of k8s yamls)
- [Docker](https://www.docker.com/get-started/) - for the containers
- [Kubernetes](https://kubernetes.io/docs/concepts/overview/what-is-kubernetes/) - [THE CLUSTER](media/peridot.png) we use to scale containers :3
  - [KIND](https://kind.sigs.k8s.io/) - Tool to spin up mini k8s cluster locally
- [Vault](https://github.com/hashicorp/vault) - Open source secret management from Hashicorp
- [ArgoCD](https://argo-cd.readthedocs.io/en/stable/) - Continuous Delivery for k8s, from within k8s
- [Argo CLI tool](https://argo-cd.readthedocs.io/en/stable/cli_installation/) - this will let you use ArgoCD without the web interface
- [ArgoCD Vault Plugin](https://argocd-vault-plugin.readthedocs.io/en/stable/installation/) - ArgoCD with Vault
  - [Kustomize](https://kustomize.io/) - kubernetes native config managment to install argo with vault
- [Kafka](https://kafka.apache.org/intro) - Handles real-time data feeds at scale

## Installation for Pre-Reqs
To get your enviornment set up, check out this [repo](https://github.com/jessebot/argo-vault-example) to learn how to install kind and ArgoCD :)

# Getting Started with Kafka
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

## Add kafka to ArgoCD

Create an SSH key pair, and then paste the public key into the deploy key section of your GitHub repo.

Make sure you have argocd cli installed, and then you can run (with `~/id_rsa_argo_deploy` being your newly created SSH key pair from the previous step):
```bash
$ argocd repo add git@github.com:jessebot/argo-kafka-example/charts/kafka --insecure-ignore-host-key --ssh-private-key-path ~/id_rsa_argo_deploy
```
