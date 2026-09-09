# homelab-main

Kubernetes manifests for homelab cluster with sane defaults, deployed via ArgoCD

## Bootstrap

Prerequisites: k3s, helm or connection to kubernetes cluster
Clone the repo, and run the following commands:

1. Install ArgoCD:

```bash
helm install argocd argo-cd --repo https://argoproj.github.io/argo-helm --namespace argocd --create-namespace --set 'configs.params.server\.insecure=true'
```

> **Note:** Change `kjkszpj` in the helm command above (ArgoCD admin password) and in `bootstrap.yaml` (cluster database password) before running.
>
> To retrieve a secret value:
> ```bash
> kubectl -n <namespace> get secret <secret-name> -o jsonpath='{.data.<key>}' | base64 -d; echo
> ```
> Useful secrets: `argocd/argocd-initial-admin-secret .data.password`, `apps/cluster-secret .data.password`

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
> **Note** In order for k3s containerd to access nvidia containerd runtime and schedule GPU workloads, a k3s containerd config is needed. Configuration: https://github.com/milanpanic2/nixos-config/blob/main/modules/k3s.nix contains containerdConfigTemplate

**vLLM:**

Change the `--model` arg in `vllm.yaml` to any HuggingFace model ID (e.g. `Qwen/Qwen3.6-27B-AWQ`).
For gated models (Gemma, Llama), uncomment and configure the Hugging Face token secret in `vllm.yaml`.
Or create yourself.

```bash
kubectl apply -f ~/cluster/manifests/vllm/vllm.yaml
kubectl delete -f ~/cluster/manifests/vllm/vllm.yaml
# If renaming resources (e.g. Service), delete first then apply:
kubectl delete -f ~/cluster/manifests/vllm/vllm.yaml && kubectl apply -f ~/cluster/manifests/vllm/vllm.yaml
```

**Open WebUI** (chat interface, connects to vLLM):
```bash
kubectl apply -f ~/cluster/manifests/vllm/open-webui.yaml
kubectl delete -f ~/cluster/manifests/vllm/open-webui.yaml
```

## Garage (S3 object storage)

Secrets: `rpc_secret` and `admin_token` are injected from the
`cluster-secret` (bootstrap) via the `GARAGE_RPC_SECRET` / `GARAGE_ADMIN_TOKEN` env
vars. Rotate them in `bootstrap.yaml` (`garage_rpc_secret`, `garage_admin_token`).

### Ports

| Port | Purpose   | On the k8s Service? | Notes |
|------|-----------|---------------------|-------|
| 3900 | S3 API    | ✅ yes              | Object storage endpoint — connect S3 clients here |
| 3902 | Web       | ✅ yes              | Static website serving (routes by `Host` header) |
| 3901 | RPC       | ❌ (single node)    | Inter-node cluster traffic; add a headless Service when scaling to multi-node |
| 3903 | Admin API | ✅ (ClusterIP only) | Management; used by garage-webui. Not exposed externally |

### First-time setup (run ONCE per cluster)

Garage needs a storage layout assigned before it accepts data. This is a one-time
imperative step — the layout persists on the PVC, so you never run it again (survives
restarts/upgrades). After the pod is `Running`:

```bash
# assign this node 10G of capacity, then commit the layout
NODE=$(kubectl -n garage exec garage-0 -- /garage status | awk 'NR>2{print $1; exit}')
kubectl -n garage exec garage-0 -- /garage layout assign -z default -c 10G "$NODE"
kubectl -n garage exec garage-0 -- /garage layout apply --version 1
```

### Connecting
  `http://garage.garage.svc.cluster.local:3900`, region `garage`.

Editing `garage.toml` (the ConfigMap) does **not** restart the pod — reload with
`kubectl -n garage rollout restart statefulset garage`.

### garage-webui (management UI)

`garage-webui/garage-webui.yaml` runs [`khairul169/garage-webui`](https://github.com/khairul169/garage-webui)
in the `garage` namespace, talking to the admin API (`:3903`) and S3 (`:3900`) over the
internal Service. Only the UI is exposed, via ingress at **`https://garage.homelab.com`**.
The ArgoCD app is `argocd-apps/garage-webui.yaml` (sync-wave 3).

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

# assign persistent volume to garage layout
kubectl -n garage exec -it garage-0 -- /garage status          # note the node ID
kubectl -n garage exec -it garage-0 -- /garage layout assign -z home -c 10G <node-id>
kubectl -n garage exec -it garage-0 -- /garage layout apply --version 1

# setup garage-rpc-secret for distributed node communication
# needs openssl installed to generate password
kubectl -n garage create secret generic garage-rpc-secret --from-literal=rpcSecret=$(openssl rand -hex 32)
# on nixos:
kubectl -n garage create secret generic garage-rpc-secret --from-literal=rpcSecret=$(head -c 32 /dev/urandom | od -An -tx1 | tr -d ' \n')

# delete secret example
kubectl -n garage delete secret garage-rpc-secret
```

## Sync wave order

- **Wave -1**: Namespaces (databases, forgejo, apps)
- **Wave 0**: MetalLB, cert-manager, Reflector, CNPG, Strimzi
- **Wave 1**: MetalLB config, cert-manager config
- **Wave 2**: PostgreSQL, Forgejo, Kafka, Garage
- **Wave 3**: Auth service, garage-webui


