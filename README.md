# homelab-main

GitOps manifests for a single-node k3s homelab. ArgoCD watches this repo and applies
everything in sync-wave order. The only imperative steps are the initial bootstrap and
Garage's one-time setup.

## Contents

- [Architecture](#architecture)
- [Bootstrap](#bootstrap)
- [Garage](#garage)
- [Operations](#operations)
  - [ArgoCD](#argocd)
  - [Restart a workload](#restart-a-workload)
  - [Resize or relocate a volume (local-path)](#resize-or-relocate-a-volume-local-path)
  - [Rotating the DB password](#rotating-the-db-password)
  - [Secrets](#secrets)
  - [Manual deployments (outside ArgoCD)](#manual-deployments-outside-argocd)
- [Debugging (k9s)](#debugging-k9s)
- [Reference — sync waves](#reference--sync-waves)

## Architecture

- **k3s** on a single NixOS host.
- **ArgoCD**, app-of-apps pattern: `bootstrap.yaml` installs a `root` Application that
  deploys everything under `argocd-apps/`.
- **Secrets**: `bootstrap.yaml` seeds `cluster-secret`; Reflector mirrors it into the
  namespaces that consume it. Garage's secrets are generated in-cluster, never committed.
- **Storage**: `local-path`, backed by the host's `/home` partition.

## Bootstrap

Requires a running k3s cluster with `kubectl` and `helm`.

1. Install ArgoCD and set the admin password:

```bash
helm install argocd argo-cd \
  --repo https://argoproj.github.io/argo-helm \
  --namespace argocd --create-namespace \
  --set 'configs.params.server\.insecure=true' \
  --set 'configs.secret.argocdServerAdminPassword=CHANGE_ME'
```

2. Set the DB password in `bootstrap.yaml` (`stringData.password`).
3. Apply it:

```bash
kubectl apply -f bootstrap.yaml
```

ArgoCD deploys the rest by sync wave. Complete the Garage setup before using object storage.

## Garage

S3 object storage. Secrets and layout persist on the PVC, so setup runs once per cluster.
`garage-0` stays in `CreateContainerConfigError` until `garage-secrets` exists.

| Port | Purpose   | Notes |
|------|-----------|-------|
| 3900 | S3 API    | Main endpoint — S3 clients connect here |
| 3901 | RPC       | Inter-node only; unused on a single node |
| 3902 | Web       | Static site serving, routed by `Host` header |
| 3903 | Admin API | Management; used by garage-webui |

**1. Secrets** — random values in `garage-secrets`:

```bash
# openssl
kubectl -n garage create secret generic garage-secrets \
  --from-literal=garage_rpc_secret=$(openssl rand -hex 32) \
  --from-literal=garage_admin_token=$(openssl rand -hex 32)

# NixOS (no openssl)
kubectl -n garage create secret generic garage-secrets \
  --from-literal=garage_rpc_secret=$(head -c32 /dev/urandom | od -An -tx1 | tr -d ' \n') \
  --from-literal=garage_admin_token=$(head -c32 /dev/urandom | od -An -tx1 | tr -d ' \n')
```

**2. Layout** — once `garage-0` is Running:

```bash
NODE=$(kubectl -n garage exec garage-0 -- /garage status | awk 'NR>2{print $1; exit}')
kubectl -n garage exec garage-0 -- /garage layout assign -z home -c 50G "$NODE"
kubectl -n garage exec garage-0 -- /garage layout apply --version 1
```

**3. App key and bucket**:

```bash
kubectl -n garage exec garage-0 -- /garage key create app-key
kubectl -n garage exec garage-0 -- /garage key allow --create-bucket app-key
kubectl -n garage exec garage-0 -- /garage key info --show-secret app-key
```

`key info` prints the Key ID and Secret key. Write them into the secret the app reads
(`access_key` ← Key ID, `secret_key` ← Secret key):

```bash
kubectl -n apps patch secret garage-secret -p '{"stringData": {"access_key": "<Key ID>", "secret_key": "<Secret key>"}}'
```

Restart the consuming app afterward so it reloads the credentials.

**Operations**:

```bash
kubectl -n garage exec garage-0 -- /garage status                    # node + capacity
kubectl -n garage exec garage-0 -- /garage key list
kubectl -n garage exec garage-0 -- /garage bucket list
kubectl -n garage exec garage-0 -- /garage bucket alias <id> <name>  # expose in garage-webui
```

In-cluster endpoint: `http://garage.garage.svc.cluster.local:3900`, region `garage`.
Editing the `garage.toml` ConfigMap does not restart the pod — reload with
`kubectl -n garage rollout restart statefulset garage`. The management UI
(garage-webui, `argocd-apps/garage-webui.yaml`) is served at `https://garage.homelab.com`.

## Operations

### ArgoCD

```bash
# did a pushed change apply? which app is unhappy? (Sync + Health)
kubectl get applications -n argocd

# stop waiting ~3 min for the poll — re-read git now
kubectl annotate application root -n argocd argocd.argoproj.io/refresh=hard --overwrite

# app wedged in a failed sync — delete it, root recreates it clean from git
kubectl delete application <name> -n argocd
```

Get the current admin password:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d; echo
```

Change the admin password:

```bash
helm upgrade argocd argo-cd \
  --repo https://argoproj.github.io/argo-helm --namespace argocd \
  --set 'configs.params.server\.insecure=true' \
  --set 'configs.secret.argocdServerAdminPassword=NEW_PASSWORD'
```

### Restart a workload

```bash
# pick up a changed Secret/ConfigMap, or clear a stuck pod (rolling, no data loss)
kubectl rollout restart deployment <name> -n <namespace>
kubectl rollout restart statefulset <name> -n <namespace>

# all deployments in a namespace — e.g. after a shared secret rotated
kubectl rollout restart deployment -n <namespace> --all
```

### Resize or relocate a volume (local-path)

`local-path` volumes cannot be expanded in place — resizing or moving one means
recreating it. Back up first unless the data is disposable. Deleting a PVC triggers the
Delete reclaim, which removes the old directory automatically; new volumes follow the
current `--default-local-storage-path`. The recreate differs by workload:

```bash
# CNPG cluster — delete it, the operator rebuilds from the manifest
kubectl delete cluster <name> -n <namespace>

# StatefulSet — volumeClaimTemplates are immutable, so drop the object (pods/PVCs keep
# running), let ArgoCD recreate it; delete the PVC too to get a fresh/larger volume
kubectl delete statefulset <name> -n <namespace> --cascade=orphan
kubectl delete pvc <pvc-name> -n <namespace>

# Deployment with a PVC — scale down, delete the PVC, scale back up
kubectl scale deployment <name> -n <namespace> --replicas=0
kubectl delete pvc <pvc-name> -n <namespace>
kubectl scale deployment <name> -n <namespace> --replicas=1
```

Self-heal may recreate the workload and re-adopt the old PVC before the delete completes —
a silent failure that leaves the old path and size. Verify after rebuild; if re-adopted,
delete the new PVC and pod and repeat.

```bash
kubectl get pv -o custom-columns='PVC:.spec.claimRef.name,SIZE:.spec.capacity.storage,PATH:.spec.local.path' | grep <name>
# expect /home/k3s-storage/... at the new size
```

### Rotating the DB password

`cluster-secret` feeds two independent things: the Postgres role (CNPG rewrites it from
the secret) and every client that reads it. Patching the secret alone is not enough —
CNPG updates the role and Reflector propagates the copies, but running pods keep the old
value in their connection pools until restarted.

```bash
kubectl -n apps patch secret cluster-secret -p '{"stringData": {"password": "NEW_PASSWORD"}}'
kubectl rollout restart deployment -n databases --all
kubectl rollout restart deployment -n forgejo --all
kubectl rollout restart deployment -n monitoring --all
```

### Secrets

```bash
kubectl create namespace <namespace>
kubectl create secret generic <name> -n <namespace> --from-literal=<key>=<value>
kubectl patch secret <name> -n <namespace> -p '{"stringData": {"<key>": "<value>"}}'

# read a value (stored base64)
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d; echo
kubectl -n apps   get secret cluster-secret            -o jsonpath='{.data.password}' | base64 -d; echo
```

### Manual deployments (outside ArgoCD)

NVIDIA device plugin — GPU scheduling; containerd runtime config lives in the NixOS `k3s.nix`:

```bash
kubectl apply  -f ~/cluster/manifests/nvidia-device-plugin/device-plugin.yaml
```

vLLM — set `--model` in `vllm.yaml` to a HuggingFace model id; gated models need the HF
token secret uncommented:

```bash
kubectl apply  -f ~/cluster/manifests/vllm/vllm.yaml
kubectl delete -f ~/cluster/manifests/vllm/vllm.yaml && kubectl apply -f ~/cluster/manifests/vllm/vllm.yaml  # after a rename
```

Open WebUI — chat UI against vLLM:

```bash
kubectl apply -f ~/cluster/manifests/vllm/open-webui.yaml
```

## Debugging (k9s)

k9s is the primary tool; the kubectl one-liners below are for one-offs and scripting.

| Key | Action |
|-----|--------|
| `:pods` `⏎`   | switch resource type — also `:deploy`, `:sts`, `:svc`, `:applications`, `:events`, `:pvc`, `:secrets` |
| `0` / `/text` | all namespaces / filter the list |
| `⏎` / `esc`   | drill in (Deployment → its pods) / back |
| `l` / `d`     | logs / describe (config + an Events list) |
| `y` / `s`     | full YAML / shell into the container |

### Diagnosing

**Pod won't start or crashes** — `d`, read the Events: `CreateContainerConfigError`
(missing secret/configmap), `ImagePullBackOff` (bad image/tag), `OOMKilled` (hit its
memory limit), `FailedScheduling` (no capacity). Then `l` for the app's own output.

**Deployment** — `d` shows rollout status and conditions (`ProgressDeadlineExceeded` =
pods never went Ready). Drill in (`⏎`) to its pods and diagnose the failing one.

**ArgoCD Application** — `:applications`, then `d` on the app: **Sync** (does it match
git?) and **Health**, plus the sync error. OutOfSync + Degraded usually means a bad
manifest; a wedged app gets the delete-and-recreate from Operations.

### kubectl one-liners

```bash
# logs — plain, follow live, or the previous crashed container
kubectl logs <pod> -n <namespace>
kubectl logs -f deployment/<name> -n <namespace>
kubectl logs <pod> -n <namespace> --previous

# why a container died — exit code + reason (OOMKilled, Error, ...)
kubectl -n <namespace> get pod <pod> -o jsonpath='{.status.containerStatuses[0].lastState}{"\n"}'

# describe (config + Events at the bottom)
kubectl describe pod <name> -n <namespace>

# events in a namespace, newest last
kubectl get events -n <namespace> --sort-by='.lastTimestamp'

# every pod, all namespaces
kubectl get pods -A
```

## Reference — sync waves

| Wave | Contents |
|------|----------|
| -1   | namespaces |
| 0    | operators — cert-manager, CNPG, MetalLB, Reflector, Strimzi |
| 1    | configs + DB — argocd-config, cert-manager-config, forgejo-config, metallb-config, traefik, postgres |
| 2    | forgejo, garage, kafka, searxng, valkey |
| 3    | apps — auth (Keycloak), crw, garage-webui, llm-agents, open-webui, web-scraper; monitoring — alloy, grafana, loki, prometheus, tempo |
| 4    | monitoring-config |
</content>
