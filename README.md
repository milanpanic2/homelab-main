# homelab-main

Kubernetes manifests for homelab, deployed via ArgoCD.

## Bootstrap

1. Install k3s and ArgoCD (on NixOS use `k3s.nix`, otherwise install manually)
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
