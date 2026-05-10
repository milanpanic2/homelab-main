# homelab-main
Kubernetes manifests with sane defaults for homelab

  ArgoCD deploys from GitHub (wave order):
  - Wave 0: MetalLB, cert-manager, Kong, Reflector, CNPG, Strimzi (Helm charts — operators/infra)
  - Wave 1: MetalLB config, cert-manager config, PostgreSQL clusters, Keycloak namespace, Forgejo namespace (raw manifests —
   depends on operators)
  - Wave 2: Keycloak, Forgejo, Kafka, vscode-https (apps — depends on databases)
