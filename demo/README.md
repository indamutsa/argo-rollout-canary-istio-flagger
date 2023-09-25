Argo Rollouts Plugin installation

```sh
curl -LO https://github.com/argoproj/argo-rollouts/releases/latest/download/kubectl-argo-rollouts-linux-amd64
chmod +x ./kubectl-argo-rollouts-linux-amd64
mv ./kubectl-argo-rollouts-linux-amd64 /usr/local/bin/kubectl-argo-rollouts
```

Verify the installation

```sh
kubectl argo rollouts version
```

Install Kustomize

```sh
curl --silent --location --remote-name "https://github.com/kubernetes-sigs/kustomize/releases/download/kustomize/v3.2.3/kustomize_kustomize.v3.2.3_linux_amd64"
chmod a+x kustomize_kustomize.v3.2.3_linux_amd64
mv kustomize_kustomize.v3.2.3_linux_amd64 /usr/local/bin/kustomize
```

Verify the install with:

```sh
kustomize version
```

To run an example:

1. Apply the manifests of one of the examples:

```bash
kustomize build demo | kubectl apply -f -
```

2. Watch the rollout or experiment using the argo rollouts kubectl plugin:

```bash
kubectl argo rollouts get rollout <ROLLOUT-NAME> --watch
kubectl argo rollouts get experiment <EXPERIMENT-NAME> --watch
```

3. For rollouts, trigger an update by setting the image of a new color to run:

```bash
kubectl argo rollouts set image <ROLLOUT-NAME> "*=argoproj/rollouts-demo:yellow"
```

watch the rollout status

```bash
watch -n 2 'kubectl get virtualservice -n rollouts-demo-istio istio-rollout-vsvc -o yaml | tail -n 10'

kubectl argo rollouts promote istio-rollout -n rollouts-demo-istio
kubectl argo rollouts get rollout istio-rollout -n rollouts-demo-istio --watch
kubectl argo rollouts get experiment istio-rollout -n rollouts-demo-istio --watch

kubectl delete -f rollout.yaml -n rollouts-demo-istio
kubectl apply -f rollout.yaml -n rollouts-demo-istio

```

We don't have to get inside the container, simply execute everything from outside using this comand:

```bash
docker exec -- minikube-access-container ls
# or
docker exec minikube-access-container ls
```

Let us inspect the labels:

```bash
kubectl get pods -n rollouts-demo-istio --show-labels
```

<!-- https://www.youtube.com/watch?v=hIL0E2gLkf8&t=190s -->
