# homelab-main

Kubernetes manifests for homelab cluster with sane defaults, deployed via ArgoCD

## Bootstrap

Prerequisites: k3s, helm or connection to kubernetes cluster
Clone the repo, and run the following commands:

1. Install ArgoCD:

```bash
helm install argocd argo-cd --repo https://argoproj.github.io/argo-helm --namespace argocd --create-namespace --set 'configs.params.server\.insecure=true' --set 'configs.secret.argocdServerAdminPassword=kjkszpj'
```

> **Note:** Change `kjkszpj` in the helm command above (ArgoCD admin password) and in `bootstrap.yaml` (cluster database password) before running.

2. Edit `bootstrap.yaml` and change the password in the secret
3. Apply the bootstrap:

```bash
kubectl apply -f bootstrap.yaml
```

ArgoCD takes over and deploys everything via sync waves.

### Changing passwords after deployment

**ArgoCD admin password:**

```bash
helm upgrade argocd argo-cd --repo https://argoproj.github.io/argo-helm --namespace argocd --set 'configs.params.server\.insecure=true' --set 'configs.secret.argocdServerAdminPassword=NEW_PASSWORD'
```

**Cluster secret:**

```bash
kubectl -n apps patch secret cluster-secret -p '{"stringData": {"password": "NEW_PASSWORD"}}'
```

Reflector propagates the change to `forgejo`, `databases`, and `monitoring`. Restart pods in those namespaces to pick up the new value:

```bash
kubectl rollout restart deployment -n apps --all
kubectl rollout restart deployment -n forgejo --all
kubectl rollout restart deployment -n databases --all
kubectl rollout restart deployment -n monitoring --all
```

## Adding a namespace

```bash
kubectl create namespace <namespace>
```

## Adding a secret to a namespace

```bash
kubectl create secret generic <secret-name> -n <namespace> --from-literal=<key>=<value>
```

To add or update a key in an existing secret:

```bash
kubectl patch secret <secret-name> -n <namespace> -p '{"stringData": {"<key>": "<value>"}}'
```
## Manual deployments (not managed by ArgoCD)

**NVIDIA device plugin:**
```bash
kubectl apply -f ~/cluster/manifests/nvidia-device-plugin/device-plugin.yaml
kubectl delete -f ~/cluster/manifests/nvidia-device-plugin/device-plugin.yaml
```

**vLLM:**

Change the `--model` arg in `vllm.yaml` to any HuggingFace model ID (e.g. `Qwen/Qwen3.6-27B-AWQ`).
For gated models (Gemma, Llama), uncomment and configure the Hugging Face token secret in `vllm.yaml`.
Or create yourself.

```bash
kubectl apply -f ~/cluster/manifests/vllm/vllm.yaml
kubectl delete -f ~/cluster/manifests/vllm/vllm.yaml
```

**Open WebUI** (chat interface, connects to vLLM):
```bash
kubectl apply -f ~/cluster/manifests/vllm/open-webui.yaml
kubectl delete -f ~/cluster/manifests/vllm/open-webui.yaml
```

## Useful commands

```bash
# Force ArgoCD to re-read the git repo and sync
kubectl annotate application root -n argocd argocd.argoproj.io/refresh=hard --overwrite

# Restart a deployment (instead of deleting pods)
kubectl rollout restart deployment <name> -n <namespace>

# Restart all deployments in a namespace
kubectl rollout restart deployment -n <namespace> --all

# Check sync status of all ArgoCD applications
kubectl get applications -n argocd

# If an ArgoCD app is stuck in a failed sync, delete it — the root app will recreate it
kubectl delete application <name> -n argocd

# View logs for a pod
kubectl logs -f deployment/<name> -n <namespace>

# Get all pods across all namespaces
kubectl get pods -A

# View events in a namespace (sorted by time)
kubectl get events -n <namespace> --sort-by='.lastTimestamp'

# View events for a specific pod
kubectl describe pod <name> -n <namespace>
```

## Sync wave order

- **Wave -1**: Namespaces (databases, forgejo, apps)
- **Wave 0**: MetalLB, cert-manager, Reflector, CNPG, Strimzi
- **Wave 1**: MetalLB config, cert-manager config
- **Wave 2**: PostgreSQL, Forgejo, Kafka
- **Wave 3**: Auth service
