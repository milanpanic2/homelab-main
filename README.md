# homelab-main

Kubernetes manifests for homelab cluster with sane defaults, deployed via ArgoCD

## Bootstrap

Prerequisites: k3s, helm or connection to kubernetes cluster
Clone the repo, and run the following commands:

1. Install ArgoCD:

```bash
helm install argocd argo-cd --repo https://argoproj.github.io/argo-helm --namespace argocd --create-namespace --set 'configs.params.server\.insecure=true'
```

2. Edit `bootstrap.yaml` and change the password in the secret
3. Apply the bootstrap:

```bash
kubectl apply -f bootstrap.yaml
```

ArgoCD takes over and deploys everything via sync waves.

## Sync wave order

- **Wave -1**: Namespaces (databases, forgejo, apps)
- **Wave 0**: MetalLB, cert-manager, Reflector, CNPG, Strimzi
- **Wave 1**: MetalLB config, cert-manager config
- **Wave 2**: PostgreSQL, Forgejo, Kafka
- **Wave 3**: Auth service
