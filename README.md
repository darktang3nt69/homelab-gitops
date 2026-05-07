# homelab-gitops

GitOps repository for the homelab Kubernetes cluster running on thor (Acer Predator PH315-51).

## Stack

| Component | Purpose |
|---|---|
| k3s + embedded etcd | Kubernetes distribution |
| Cilium | CNI (eBPF) |
| MetalLB | LoadBalancer IP pool |
| cert-manager | TLS certificates (Let's Encrypt DNS-01) |
| Longhorn | Block storage |
| Envoy Gateway | Ingress |
| ArgoCD | GitOps controller |

## Structure

```
bootstrap/          ← one-time ArgoCD repo connection
argocd/apps/        ← ArgoCD Application manifests (App of Apps)
infrastructure/     ← Helm chart values + configs per component
```

## How It Works

1. ArgoCD watches this repo (main branch)
2. `argocd/apps/root-app.yaml` is the App of Apps — manages all other ArgoCD apps
3. Every folder in `infrastructure/` has its own ArgoCD Application
4. Push to Git → ArgoCD detects diff → syncs to cluster

## Deploy Order

```
MetalLB → cert-manager → Longhorn → Envoy Gateway → Vault → apps
```

## Node

- **thor** — `192.168.1.9` — control plane + worker (media workloads)
- **orangepi** — TBD — worker (general workloads, Phase 2)
