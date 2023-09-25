# Get started with Argo rollouts, istio, flagger and canary deployments

Get a working container with the following command:

Lets create a Kubernetes cluster to play with using miniKube

```bash
minikube start --cpus=4 --memory=8192
minikube addons enable ingress
```

Enable some addons:

```bash
minikube addons enable ingress
minikube addons enable dashboard
minikube addons enable metrics-server
```

Start the dashboard:

```bash
minikube dashboard
# http://127.0.0.1:35587/api/v1/namespaces/kubernetes-dashboard/services/http:kubernetes-dashboard:/proxy/#/workloads?namespace=default
```

# Getting Started with Istio

Firstly, I like to do most of my work in containers so everything is reproducable <br/>
and my machine remains clean.

## Start a container to work in:

```bash
docker run -it --rm --net host --name minikube-access-container \
-e KUBECONFIG=/home/arsene/.kube/config \
-e MINIKUBE_HOME=/home/arsene/.minikube \
-v /var/run/docker.sock:/var/run/docker.sock \
-v ${HOME}/.kube/:/home/arsene/.kube/ \
-v ${HOME}/.minikube/:/home/arsene/.minikube/ \
-v ${PWD}:/work \
-w /work alpine sh
```

## Update the package repository and install necessary dependencies and utilities.

Let us create a command folder accessible from across the container

```bash
mkdir -p /cmd
```

## Install common utilities

```bash
# Install common utilities and beautify the terminal
apk update
apk add --no-cache docker curl wget py-pip python3-dev libffi-dev openssl-dev gcc libc-dev make  zip bash openssl mongodb-tools git docker-compose zsh vim nano unzip npm jq kustomize
# Install zsh for a cool looking terminal with plugins auto-suggestions and syntax-highlighting
sh -c "$(curl -fsSL https://raw.github.com/robbyrussell/oh-my-zsh/master/tools/install.sh)"

## Clone the zsh-autosuggestions repository into $ZSH_CUSTOM/plugins
git clone https://github.com/zsh-users/zsh-autosuggestions.git $ZSH_CUSTOM/plugins/zsh-autosuggestions
git clone https://github.com/zsh-users/zsh-syntax-highlighting.git $ZSH_CUSTOM/plugins/zsh-syntax-highlighting
sed -i.bak 's/plugins=(git)/plugins=(git zsh-autosuggestions zsh-syntax-highlighting)/' ~/.zshrc

# Check docker-compose version
docker-compose --version
apk update

# Install kubectl
curl -LO https://storage.googleapis.com/kubernetes-release/release/`curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt`/bin/linux/amd64/kubectl
chmod +x ./kubectl
mv ./kubectl /usr/local/bin/kubectl

# Install helm
curl -LO https://get.helm.sh/helm-v3.7.2-linux-amd64.tar.gz
tar -C /tmp/ -zxvf helm-v3.7.2-linux-amd64.tar.gz
rm helm-v3.7.2-linux-amd64.tar.gz
mv /tmp/linux-amd64/helm /usr/local/bin/helm
chmod +x /usr/local/bin/helm

# Get Terraform

curl -o /tmp/terraform.zip -LO https://releases.hashicorp.com/terraform/1.5.5/terraform_1.5.5_linux_amd64.zip
unzip /tmp/terraform.zip
chmod +x terraform && mv terraform /usr/local/bin/
terraform

# Install kind to access the cluster
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.20.0/kind-linux-amd64
chmod +x ./kind
mv ./kind /usr/local/bin/kind

# 4. Install ArgoCD CLI

curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
install -m 555 argocd-linux-amd64 /usr/bin/argocd
rm argocd-linux-amd64

# Install argo-rollouts plugin
curl -LO https://github.com/argoproj/argo-rollouts/releases/latest/download/kubectl-argo-rollouts-linux-amd64
chmod +x kubectl-argo-rollouts-linux-amd64
mv kubectl-argo-rollouts-linux-amd64 /usr/local/bin/kubectl-argo-rollouts



curl -L https://istio.io/downloadIstio | ISTIO_VERSION=1.19.0 TARGET_ARCH=x86_64 sh -
mv istio-1.19.0/bin/istioctl /usr/local/bin/
chmod +x /usr/local/bin/istioctl
mv istio-1.19.0 ~

# Test cluster access
kubectl get nodes
# NAME STATUS ROLES AGE VERSION
# vault-control-plane Ready control-plane,master 37s v1.21.1
```

Deleting in linux is a dangerous operation, let us create a script to confirm before deleting

```bash
cat << 'EOF' > /cmd/confirm_rm_rf.sh
#!/bin/bash

# Check first argument for recursive delete flags
case $1 in
  -r|-rf)
    printf "Do you really wanna delete (yes/no) \n===>: "
    # Reading the input from terminal
    read -r answer
    if [ "$answer" == "yes" ]; then
      sudo rm -rf "$@"
    else
      printf "You didn't confirm!\nExiting, no action taken!\n"
    fi
    ;;
  *)
    # If no "-r" or "-rf", proceed without asking
    rm -i "$@"
    ;;
esac

EOF
chmod +x /cmd/confirm_rm_rf.sh
cat /cmd/confirm_rm_rf.sh


# ---
cat << 'EOF' >> ~/.zshrc
source $ZSH/oh-my-zsh.sh
source $ZSH_CUSTOM/plugins/zsh-autosuggestions
source $ZSH_CUSTOM/plugins/zsh-syntax-highlighting

export PATH="$PATH:/cmd"
alias rm="confirm_rm_rf.sh"

EOF
cat ~/.zshrc

# To apply the changes, the auto-suggestions and syntax-highlighting plugins must be sourced:
source ~/.zshrc
zsh
```

## Prerequisites

#### ArgoCD Installation

1. First, install ArgoCD by running the following commands:

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Here’s a brief synopsis of what was actually installed

- Custom resource definitions (CRDs): You’ve now installed the necessary resources ArgoCD needs.
- Service accounts: We now also have the various ServiceAccounts for our Argo CD components. Which includes the ApplicationSet controller, the Notifications controller, and more.
- Roles and Cluster Roles: These roles will provide the cluster-level permissions needed for Argo CD to interact with various Kubernetes resources and perform operations within the cluster.
- Role Bindings and Cluster Role Bindings: This will associate the ServiceAccount’s with their respective Role or Cluster Role.
- Config Maps: We’ve also installed the config data for the various Argo CD components as well.
- Secrets: These secrets store sensitive information required by the Argo CD components and resources, which ensuring the handling of credentials, API keys, and other confidential data necessary to operate our cluster.
- Deployments and Stateful Sets: These are the running instances of the specific Argo CD components we will use for this workshop.
- Network policies: These policies establish specific network connectivity rules for our Argo CD components, ensuring that we have secure communication within the Kubernetes cluster.

Now that we have Argo CD up and running, let’s check the status of the pods in our cluster, run the command:

```bash
# Get all pods in our cluster
kubectl get pods -A
```

Get the password for the ArgoCD UI

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

Use the Argo CD CLI to enable port forwarding and login to the UI (note: we’re going to be adding that --port-forward-namespace flag to every CLI command we run):

```bash
argocd --port-forward --port-forward-namespace argocd login localhost:8080
```

This is where you will need the admin password we decoded in the steps above. Once we have logged in successfully we’re going to create an app which will be a part of our Argo CD list of managed applications:

```bash
argocd --port-forward-namespace argocd repo add "https://github.com/argocon22Workshop/ArgoCDRollouts"
```

This is the same repo we have specified as our workshop project listed above in the Prerequisites section. The next thing we need to do is create an app with argocd app create command:

```bash
argocd --port-forward-namespace argocd app sync argo-rollouts
```
