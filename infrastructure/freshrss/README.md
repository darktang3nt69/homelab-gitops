---
# =============================================================================
# FreshRSS on Kubernetes — README
# =============================================================================
# Self-hosted RSS aggregator deployed on k3s using Helm + ArgoCD
# =============================================================================

## Overview

FreshRSS is a self-hosted RSS feed aggregator. This deployment brings it from
Docker to your k3s cluster using:
- **Helm Chart**: k8s-home-lab/freshrss (v7.2.1)
- **Image**: ghcr.io/linuxserver/freshrss:1.28.1
- **Deployment**: ArgoCD app of apps pattern

**Key Features:**
- ✅ Persistent storage via Longhorn (10Gi)
- ✅ Automatic HTTPS via cert-manager
- ✅ Accessible via Envoy Gateway
- ✅ Easy backup/restore with Velero
- ✅ Automated deployment with Helm + ArgoCD
- ✅ Non-root container (UID: 568)
- ✅ Security scanning + Kyverno policies compliant

---

## Architecture

```
┌─────────────────────────────────────────────────────┐
│         Envoy Gateway (ingress-envoy)               │
│  freshrss.k8s.darktang3nt.cloud (HTTPS via cert-m) │
└────────────────────────┬────────────────────────────┘
                         │
                         ↓
       ┌─────────────────────────────────┐
       │   HTTPRoute (freshrss)          │
       │   Namespace: freshrss           │
       └────────────────┬────────────────┘
                        │
                        ↓
      ┌─────────────────────────────────┐
      │  Service (freshrss)             │
      │  (Created by Helm Chart)        │
      └────────────────┬────────────────┘
                       │
                       ↓
      ┌──────────────────────────────────────────┐
      │  Deployment (freshrss)                   │
      │  Image: linuxserver/freshrss:1.28.1      │
      │  Replicas: 1                             │
      │  (Created by Helm Chart)                 │
      └────────────────┬─────────────────────────┘
                       │
                       ↓ (Mount)
      ┌──────────────────────────────────────────┐
      │  PVC (freshrss-config, 10Gi)             │
      │  StorageClass: longhorn                  │
      │  Mount Path: /config                     │
      │  (Created by Helm Chart)                 │
      └──────────────────────────────────────────┘
```

---

## Files Structure

```
argocd/apps/
└── freshrss-app.yaml          # ArgoCD Application (Helm-based)

infrastructure/freshrss/
├── deployment.yaml            # Documentation (Helm manages deployment)
├── rbac.yaml                  # ServiceAccount (optional)
├── httproute.yaml             # Envoy Gateway ingress
├── MIGRATION.md               # Data migration guide
└── README.md                  # This file
```

---

## Deployment Status

Check deployment status:

```bash
# Check if ArgoCD recognizes the app
argocd app list | grep freshrss
# or
kubectl get application freshrss -n argocd

# Check Kubernetes resources (all created by Helm)
kubectl get pods -n freshrss
kubectl get svc -n freshrss
kubectl get pvc -n freshrss

# Check pod logs
kubectl logs -n freshrss deployment/freshrss -f

# Check Helm release
helm list -n freshrss

# Check HTTPRoute
kubectl get httproute -n freshrss
```

---

## First-Time Setup

### 1. ArgoCD Auto-Deploy

Once the `freshrss-app.yaml` is in the Git repo:
```bash
# ArgoCD will automatically detect and deploy (if sync is enabled)
# Or manually sync:
argocd app sync freshrss

# Watch the deployment progress
argocd app wait freshrss
```

### 2. Wait for Pod Ready
```bash
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=freshrss -n freshrss --timeout=300s
```

### 3. Access the Web Interface
```
URL: https://freshrss.k8s.darktang3nt.cloud

Note: First access will show a setup screen
- Create admin account
- Configure database (SQLite is pre-configured in /config)
- Add feeds
```

---

## Configuration

All configuration is managed via Helm values in [argocd/apps/freshrss-app.yaml](../../apps/freshrss-app.yaml):

| Setting | Value | Notes |
|---------|-------|-------|
| Image | ghcr.io/linuxserver/freshrss:1.28.1 | Pinned version (Kyverno compliant) |
| Storage | 10Gi Longhorn | Persistent configuration |
| User ID | 568 | Non-root user (linuxserver standard) |
| Timezone | Asia/Kolkata | Configurable via Helm values |
| CPU Limit | 500m | Adjustable per workload |
| Memory Limit | 512Mi | Adjustable per workload |

To modify configuration, edit the `helm.values` section in `argocd/apps/freshrss-app.yaml`.

---

## Data Migration from Docker

**See [MIGRATION.md](MIGRATION.md) for detailed step-by-step instructions.**

Quick summary:
1. Backup FreshRSS data from Docker container
2. Deploy k3s application via ArgoCD (Helm chart)
3. Restore backup into Kubernetes PVC at `/config`
4. Restart pod and verify

---

## Backup & Restore

### Manual Backup

```bash
# Backup to local machine
kubectl exec -n freshrss deployment/freshrss -- tar -cz /config | \
  tar -xz -O > freshrss-backup-$(date +%Y%m%d).tar.gz

# Or backup using Velero
velero backup create freshrss-backup --include-namespaces freshrss
```

### Manual Restore

```bash
# Restore from backup (via kubectl cp)
kubectl cp ./freshrss-backup.tar.gz freshrss/freshrss:/tmp/backup.tar.gz
kubectl exec -it freshrss/freshrss -- tar -xzf /tmp/backup.tar.gz -C /
```

### Automated Backups with Velero

```yaml
# Create a Velero scheduled backup
apiVersion: velero.io/v1
kind: Schedule
metadata:
  name: freshrss-daily
  namespace: velero
spec:
  schedule: "0 2 * * *"  # 2 AM daily
  template:
    includedNamespaces:
      - freshrss
    storageLocation: default
```

---

## Resource Limits

Current Helm configuration:
- **CPU Request**: 100m
- **CPU Limit**: 500m
- **Memory Request**: 128Mi
- **Memory Limit**: 512Mi
- **Storage**: 10Gi (Longhorn PVC)

Adjust in `argocd/apps/freshrss-app.yaml` under `resources` section if needed based on feed volume.

---

## Troubleshooting

### Pod won't start
```bash
kubectl describe pod -n freshrss deployment/freshrss
kubectl logs -n freshrss deployment/freshrss -f
```

### Storage issues
```bash
# Check PVC status
kubectl describe pvc -n freshrss freshrss-config

# Check Longhorn volumes
kubectl get volumes -n longhorn-system
kubectl describe volume -n longhorn-system <volume-name>
```

### Access issues
```bash
# Test Service connectivity
kubectl run -it --rm debug --image=curlimages/curl --restart=Never -- \
  curl -v http://freshrss.freshrss.svc.cluster.local

# Check HTTPRoute
kubectl get httproute -n freshrss -o yaml
```

### DNS not resolving
```bash
# Verify CoreDNS is working
kubectl get pods -n kube-system | grep coredns

# Test DNS from pod
kubectl exec -it -n freshrss deployment/freshrss -- nslookup freshrss.freshrss.svc.cluster.local
```

### Helm chart issues
```bash
# Check Helm release status
helm status freshrss -n freshrss

# Get Helm release values
helm get values freshrss -n freshrss

# Rollback to previous version if needed
helm rollback freshrss -n freshrss
```

---

## Updating FreshRSS

### Update via ArgoCD

To update the image version:

1. Edit `argocd/apps/freshrss-app.yaml`
2. Change version under `helm.values.image.tag`
3. Commit and push to Git
4. ArgoCD will auto-sync and restart the pod

```bash
# Monitor the rollout
kubectl rollout status deployment/freshrss -n freshrss -w
```

### Update Helm Chart Version

To use a newer Helm chart version:

1. Edit `targetRevision` in `argocd/apps/freshrss-app.yaml`
2. Check chart changelog at: https://artifacthub.io/packages/helm/k8s-home-lab-repo/freshrss

---

## Uninstalling

```bash
# Delete via ArgoCD
argocd app delete freshrss

# Or delete via kubectl
kubectl delete application freshrss -n argocd

# This will delete the entire namespace and PVC (if finalizers configured)
# To preserve data, remove the namespace finalizer first
kubectl patch finalizer application/freshrss -n argocd
```

---

## Security Considerations

1. **Non-Root User**: Runs as UID 568 (linuxserver standard)
2. **Image**: Scanned and pinned to specific version (1.28.1)
3. **HTTPS**: Enforced via cert-manager + Envoy Gateway
4. **Network Policy**: Consider adding NetworkPolicy to restrict ingress
5. **RBAC**: Minimal ServiceAccount privileges
6. **Data Encryption**: Enable Longhorn encryption in production
7. **Kyverno Policies**: Compliant with:
   - ✅ disallow-latest-tag
   - ✅ require-non-root-user
   - ✅ restrict-capabilities

---

## Related Documentation

- [FreshRSS Official Docs](https://www.freshrss.org/)
- [Helm Chart (k8s-home-lab)](https://artifacthub.io/packages/helm/k8s-home-lab-repo/freshrss)
- [Longhorn Storage](../longhorn/)
- [Envoy Gateway](../envoy-gateway/)
- [cert-manager](../cert-manager/)
- [ArgoCD App of Apps](../argocd/)

---

## Support

For issues or questions:
1. Check pod logs: `kubectl logs -n freshrss deployment/freshrss`
2. Review MIGRATION.md for migration-specific issues
3. Check Helm chart issues: https://github.com/k8s-home-lab/helm-charts/issues
4. Check [FreshRSS Issues](https://github.com/FreshRSS/FreshRSS/issues)
5. Review k3s/Kubernetes documentation for cluster issues
