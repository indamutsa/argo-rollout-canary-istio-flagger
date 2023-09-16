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
apk add --no-cache docker curl wget py-pip python3-dev libffi-dev openssl-dev gcc libc-dev make  zip bash openssl mongodb-tools git docker-compose zsh vim nano unzip npm
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

### Step 1: Install ArgoCD and Rollouts

**Objective:**  
The aim of this step is to get ArgoCD installed on your Kubernetes cluster. ArgoCD will help you manage deployments and keep your Kubernetes manifests in sync with the ones in your Git repository. After installing ArgoCD, we'll also install Argo Rollouts, which extends Kubernetes to add extra deployment strategies like Canary and Blue-Green deployments.

#### ArgoCD Installation

1. First, install ArgoCD by running the following commands:

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

2. Once the pods are up and running, you can port-forward the ArgoCD API server to access the ArgoCD UI on your local machine:

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

3. Open your browser and go to `https://localhost:8080`. The default username is `admin`, and the password can be retrieved it using:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

#### Argo Rollouts Installation

1. To install Argo Rollouts, you can use the following command:

```bash
kubectl create namespace argo-rollouts
kubectl apply -n argo-rollouts -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml
```

Verify that the Argo Rollouts controller is running:

```bash
kubectl -n argo-rollouts get pods
```

You've now installed both ArgoCD and Argo Rollouts, setting the stage for a more flexible deployment strategy, such as Canary deployments.

### Step 2: Install Istio

Install Istio

```bash
istioctl install --set profile=demo -y
```

is already installed. But let us enable the Istio sidecar injection for the default namespace:

```bash
kubectl label namespace default istio-injection=enabled

# Check if the default namespace has the label
kubectl get namespace default -o jsonpath='{.metadata.labels}'
```

### Step 3: Apply BookInfo Kubernetes App

**Objective:**  
The goal of this step is to deploy the initial version of the BookInfo application. This sample app, which comes with Istio, is composed of multiple microservices and makes for a good demonstration of Istio's capabilities. The initial deployment will act as the 'stable' version for the canary deployment process.

1. **Navigate to the BookInfo Sample**:  
   Navigate to the Istio package directory and find the BookInfo sample:

```bash
 cp -r ~/istio/samples/bookinfo/ .
```

2. **Deploy the BookInfo App**:  
   Apply the Kubernetes manifests for the BookInfo application. Make sure you're working in the namespace where Istio injection is enabled (as per Step 2):

```bash
 kubectl apply -f bookinfo/platform/kube/bookinfo.yaml
```

3. **Wait for Pods to be Running**:

Get the app labels for the running pods:

```bash
kubectl get pods --show-labels
```

```bash
kubectl wait --for=condition=Ready --timeout=300s pods -l app=productpage
kubectl wait --for=condition=Ready --timeout=300s pods -l app=details
kubectl wait --for=condition=Ready --timeout=300s pods -l app=ratings
kubectl wait --for=condition=Ready --timeout=300s pods -l app=reviews
```

### Step 4: Configure Istio Ingress, Gateway, and VirtualService

**Objective:**  
The aim here is to set up Istio ingress, gateway, and virtual service resources. These Istio components manage how external traffic gets routed to the services in your Kubernetes cluster.

1. **Apply Istio Gateway**:  
   To control ingress traffic for your application, you'll need to define an Istio Gateway:

```bash
kubectl apply -f bookinfo/networking/bookinfo-gateway.yaml
```

2. **Apply Virtual Services**:  
   Virtual services in Istio enable fine-grained traffic routing. Apply the virtual service definitions that route traffic through the gateway to your application:

```bash
kubectl apply -f bookinfo/networking/virtual-service-all-v1.yaml
```

3. **Confirm Configuration**:  
   After applying the gateway and virtual services, verify they are correctly set up:

```bash
kubectl get gateway
kubectl get virtualservices
```

4. **Test External Access**:  
   At this point, you should be able to access the BookInfo app via the Istio ingress gateway. Find the external IP for the ingress:

```bash
kubectl get svc istio-ingressgateway -n istio-system
```

Then, access the app using the obtained external IP and the configured port. If you've set it up using standard HTTP, you can navigate to it using your browser.

Great, let's integrate Git with ArgoCD for better management of our application lifecycle. We'll also define the Argo Rollout manifests for a canary deployment.

### Step 5: Integrate Git with ArgoCD and Create Rollout Manifests

**Objective:**  
The aim here is to configure ArgoCD to sync with a Git repository where your application's manifests are stored. We'll use GitHub for this example and create a repository for our Argo Rollout configuration.

Let us first create the repository on GitHub. Create this outside the container.

```bash
# Initialize the repository
git init

# Start the creation of the repository using the GitHub CLI
gh repo create
```
