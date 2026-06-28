---
# =============================================================================
# FreshRSS on Kubernetes — README
# =============================================================================
# Self-hosted RSS aggregator deployed on k3s using ArgoCD
# =============================================================================

## Overview

FreshRSS is a self-hosted RSS feed aggregator. This deployment brings it from
Docker to your k3s cluster using ArgoCD's app of apps pattern.

**Key Features:**
- ✅ Persistent storage via Longhorn
- ✅ Automatic HTTPS via cert-manager
- ✅ Accessible via Envoy Gateway
- ✅ Easy backup/restore with Velero
- ✅ Automated deployment with ArgoCD

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
      │  Service (freshrss:80)          │
      │  Namespace: freshrss            │
      └────────────────┬────────────────┘
                       │
                       ↓
      ┌──────────────────────────────────────────┐
      │  Deployment (freshrss)                   │
      │  Image: freshrss/freshrss:latest         │
      │  Replicas: 1                             │
      │  Namespace: freshrss                     │
      └────────────────┬─────────────────────────┘
                       │
                       ↓ (Mount)
      ┌──────────────────────────────────────────┐
      │  PVC (freshrss-data, 10Gi)               │
      │  StorageClass: longhorn                  │
      │  Mount Path: /var/www/FreshRSS/data      │
      └──────────────────────────────────────────┘
```

---

## Files Structure

```
argocd/apps/
└── freshrss-app.yaml          # ArgoCD Application definition

infrastructure/freshrss/
├── deployment.yaml            # PVC + Deployment + Service
├── rbac.yaml                  # ServiceAccount
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

# Check Kubernetes resources
kubectl get pods -n freshrss
kubectl get svc -n freshrss
kubectl get pvc -n freshrss

# Check pod logs
kubectl logs -n freshrss deployment/freshrss -f

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
```

### 2. Wait for Pod Ready
```bash
kubectl wait --for=condition=ready pod -l app=freshrss -n freshrss --timeout=300s
```

### 3. Access the Web Interface
```
URL: https://freshrss.k8s.darktang3nt.cloud

Note: First access will show a setup screen
- Create admin account
- Configure database (SQLite is pre-configured)
- Add feeds
```

### 4. Add Admin Credentials (Optional)

For future automated setup, you can create a Kubernetes Secret:
```bash
kubectl create secret generic freshrss-credentials \
  -n freshrss \
  --from-literal=admin-password='your-strong-password'
```

Then uncomment the environment variables in `deployment.yaml`.

---

## Data Migration from Docker

**See [MIGRATION.md](MIGRATION.md) for detailed step-by-step instructions.**

Quick summary:
1. Backup FreshRSS data from Docker container
2. Deploy k3s application via ArgoCD
3. Restore backup into Kubernetes PVC
4. Restart pod and verify

---

## Backup & Restore

### Manual Backup

```bash
# Backup to local machine
kubectl exec -n freshrss deployment/freshrss -- tar -cz /var/www/FreshRSS/data | \
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

Current configuration:
- **CPU Request**: 100m
- **CPU Limit**: 500m
- **Memory Request**: 128Mi
- **Memory Limit**: 512Mi
- **Storage**: 10Gi (Longhorn PVC)

Adjust in `deployment.yaml` if needed based on feed volume.

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
kubectl describe pvc -n freshrss freshrss-data

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

---

## Updating FreshRSS

To update the image version:

1. Edit `infrastructure/freshrss/deployment.yaml`
2. Change `image: freshrss/freshrss:latest` to `freshrss/freshrss:1.x.x`
3. Commit and push to Git
4. ArgoCD will auto-sync and restart the pod

```bash
# Monitor the rollout
kubectl rollout status deployment/freshrss -n freshrss -w
```

---

## Uninstalling

```bash
# Delete via ArgoCD
argocd app delete freshrss

# Or delete via kubectl
kubectl delete application freshrss -n argocd

# This will delete the entire namespace and all data (if finalizers configured)
# To keep data, remove the namespace finalizer first
kubectl delete finalizer application/freshrss -n argocd
```

---

## Security Considerations

1. **Database Password**: Use a strong password and store in Vault/Secret
2. **HTTPS**: Enforced via cert-manager + Envoy Gateway
3. **Network Policy**: Consider adding NetworkPolicy to restrict ingress
4. **RBAC**: ServiceAccount has minimal permissions
5. **Data Encryption**: Enable Longhorn encryption in production

---

## Related Documentation

- [FreshRSS Official Docs](https://www.freshrss.org/)
- [Longhorn Storage](../longhorn/)
- [Envoy Gateway](../envoy-gateway/)
- [cert-manager](../cert-manager/)
- [ArgoCD App of Apps](../argocd/)

---

## Support

For issues or questions:
1. Check pod logs: `kubectl logs -n freshrss deployment/freshrss`
2. Review MIGRATION.md for migration-specific issues
3. Check [FreshRSS Issues](https://github.com/FreshRSS/FreshRSS/issues)
4. Review k3s/Kubernetes documentation for cluster issues
