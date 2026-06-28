---
# =============================================================================
# FreshRSS Data Migration Guide
# =============================================================================
# Migrate FreshRSS from Docker to Kubernetes cluster (Helm-based)
#
# Prerequisites:
# - Running Docker instance of FreshRSS with existing data
# - k3s cluster with ArgoCD installed
# - kubectl configured to access your k3s cluster
# - freshrss-app.yaml deployed to ArgoCD (Helm chart v7.2.1)
# =============================================================================

## Step 1: Deploy FreshRSS to Kubernetes

First, ensure the FreshRSS application is created in ArgoCD:

```bash
# Verify the FreshRSS namespace and pod are running
kubectl get ns freshrss
kubectl get pods -n freshrss

# Check ArgoCD app status
argocd app list | grep freshrss
argocd app info freshrss
```

Wait for the deployment to be ready before proceeding.

---

## Step 2: Backup Docker Data

Backup your existing FreshRSS data from Docker.

**Important:** Identify where your FreshRSS config/database is stored:
- **Linuxserver/FreshRSS**: typically at `/config` in container
- **Official FreshRSS**: typically at `/var/www/FreshRSS/data`

### Option A: Using docker exec (if container is running)

```bash
# For linuxserver/freshrss
docker exec freshrss_container_name tar -czf - /config > freshrss-backup.tar.gz

# OR for official freshrss image
docker exec freshrss_container_name tar -czf - /var/www/FreshRSS/data > freshrss-backup.tar.gz
```

### Option B: Using docker cp (copy from stopped/running container)

```bash
# Create temporary backup
mkdir -p ./freshrss-backup-temp
docker cp freshrss_container_name:/config ./freshrss-backup-temp/
tar -czf freshrss-backup.tar.gz ./freshrss-backup-temp/
rm -rf ./freshrss-backup-temp
```

### Option C: From volume mount (if you know the mount path)

```bash
# For Linuxserver FreshRSS with volume at /mnt/freshrss
tar -czf freshrss-backup.tar.gz /mnt/freshrss

# Find volumes:
docker volume ls | grep freshrss
docker volume inspect freshrss_config
```

**Example using docker-compose**:
```bash
docker-compose -f docker-compose.yml exec freshrss tar -czf - /config > freshrss-backup.tar.gz
```

---

## Step 3: Copy Backup to Cluster Node

Copy the backup to one of your cluster nodes:

```bash
# Copy to the cluster node (adjust IP/path as needed)
scp freshrss-backup.tar.gz root@k3s-node:/tmp/

# Or use your preferred method (rsync, sftp, etc.)
```

---

## Step 4: Restore Data to PVC

### Method 1: Using kubectl cp (Recommended)

```bash
# Copy backup into the pod
kubectl cp ./freshrss-backup.tar.gz freshrss/freshrss:/tmp/backup.tar.gz

# Connect to pod and restore
kubectl exec -it -n freshrss deployment/freshrss -- sh

# Inside the pod:
cd /config
tar -xzf /tmp/backup.tar.gz --strip-components=1
ls -la  # Verify files are restored
exit
```

### Method 2: Direct volume mount access

```bash
# SSH into a cluster node
ssh root@k3s-node

# Find the actual PVC mount path on the node
# Longhorn PVCs are typically in /var/lib/rancher/k3s/data/

# Get the PVC details
kubectl get pvc -n freshrss freshrss-config -o jsonpath='{.spec.volumeName}'

# Get the volume mount point
kubectl describe pv <volume-name> | grep path

# Extract directly to the volume
cd /var/lib/rancher/k3s/data/longhorn/volumes/<volume-id>/
tar -xzf /tmp/freshrss-backup.tar.gz --strip-components=1
```

### Method 3: Using a temporary job

```yaml
# Create a temporary pod to restore data
kubectl run -n freshrss restore-job \
  --image=busybox:1.36 \
  --rm -it \
  --overrides='{"spec": {"serviceAccountName": "freshrss"}}' \
  -- sh

# Inside the pod:
# Copy backup and extract (you'd need to mount the backup volume first)
```

---

## Step 5: Verify Restore and Set Permissions

```bash
# Verify data was extracted
kubectl exec -n freshrss deployment/freshrss -- ls -la /config

# The files should be owned by UID 568 (linuxserver default)
# Verify ownership if needed
kubectl exec -n freshrss deployment/freshrss -- stat /config
```

---

## Step 6: Restart the Pod

Force a restart to reload the data:

```bash
kubectl rollout restart deployment/freshrss -n freshrss

# Verify it's running
kubectl get pods -n freshrss -w

# Wait for ready state
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=freshrss -n freshrss --timeout=300s
```

---

## Step 7: Verify Data Migration

```bash
# Check the logs
kubectl logs -n freshrss deployment/freshrss -f

# Verify the data was restored
kubectl exec -n freshrss deployment/freshrss -- ls -la /config

# Check for FreshRSS database/config files
# Should contain: data.php, db.sqlite (or similar database file)
kubectl exec -n freshrss deployment/freshrss -- find /config -type f
```

---

## Step 8: Validate Feeds and Settings

1. Open your browser to `https://freshrss.k8s.darktang3nt.cloud`
2. **Important**: You may need to log in again or reconfigure if the database was not properly restored
3. Verify all feeds are present and updating
4. Check that all your settings and read/unread status are preserved
5. Verify stored credentials for feed authentication (may need to re-enter)

---

## Data Layout Reference

### Kubernetes (Helm-based, linuxserver/freshrss)

```
/config/                              # Mount point in container
├── data/                             # Runtime data
├── db.sqlite                         # SQLite database
├── installs.php                      # Feed configuration
└── config.php                        # Application config
```

### Docker (Original structure to backup)

**Linuxserver version:**
```
/config/                              # Inside container
├── data/
├── db.sqlite
└── installs.php
```

**Official version:**
```
/var/www/FreshRSS/data/              # Inside container
├── db.sqlite
└── installs.php
```

---

## Troubleshooting

### Issue: "No such file or directory" when running kubectl exec

```bash
# Wait for pod to be fully ready
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=freshrss -n freshrss

# If still failing, check pod status
kubectl describe pod -n freshrss deployment/freshrss
kubectl logs -n freshrss deployment/freshrss
```

### Issue: Data folder appears empty after restore

```bash
# Check PVC is properly mounted
kubectl describe pod -n freshrss deployment/freshrss

# Verify the restore operation
kubectl exec -n freshrss deployment/freshrss -- tar -tzf /tmp/backup.tar.gz | head

# Re-extract if needed
kubectl exec -it -n freshrss deployment/freshrss -- sh
cd /config
rm -rf * 
tar -xzf /tmp/backup.tar.gz --strip-components=1
exit
```

### Issue: Pod won't start after data restore

```bash
# Check pod logs for errors
kubectl logs -n freshrss deployment/freshrss --previous

# Verify file ownership (should be UID 568 for linuxserver)
kubectl exec -n freshrss deployment/freshrss -- stat /config

# Check for database corruption
kubectl exec -n freshrss deployment/freshrss -- sqlite3 /config/db.sqlite ".tables"
```

### Issue: Can't log in after restore

```bash
# Database structure may have changed between versions
# Check if you have the latest FreshRSS migration scripts
kubectl exec -n freshrss deployment/freshrss -- ls -la /config/

# You may need to re-add feeds if the database is incompatible
# But user preferences/accounts should persist
```

### Issue: "Connection refused" to freshrss.k8s.darktang3nt.cloud

```bash
# Verify HTTPRoute is correct
kubectl get httproute -n freshrss -o yaml

# Test internal connectivity
kubectl run -it --rm debug --image=curlimages/curl --restart=Never -- \
  curl -v http://freshrss.freshrss.svc.cluster.local

# Check DNS resolution
kubectl run -it --rm debug --image=busybox:1.36 --restart=Never -- \
  nslookup freshrss.k8s.darktang3nt.cloud
```

---

## Rollback Plan

If something goes wrong, you can quickly fall back to Docker:

```bash
# Stop the k8s FreshRSS
kubectl delete deployment freshrss -n freshrss

# Resume Docker container
docker start freshrss_container_name

# Once verified, delete k8s app from ArgoCD
argocd app delete freshrss

# Or via kubectl
kubectl delete application freshrss -n argocd
```

---

## Data Verification Checklist

- [ ] Backup created from Docker
- [ ] Backup copied to cluster
- [ ] Kubernetes pod is running
- [ ] Data restored to `/config` directory
- [ ] Pod restarted and running
- [ ] Can access `freshrss.k8s.darktang3nt.cloud` in browser
- [ ] Feeds are visible and updating
- [ ] User accounts and preferences intact
- [ ] Feed credentials re-entered if needed

---

## Post-Migration Steps

1. **Test Thoroughly**: Verify all feeds are working
2. **Monitor Logs**: Watch for any errors: `kubectl logs -f -n freshrss deployment/freshrss`
3. **Set Up Backups**: Configure Velero scheduled backups for Kubernetes
4. **Clean Up Docker**: Once fully verified, you can safely remove Docker FreshRSS
5. **Document Custom Config**: Note any custom settings or modifications

```bash
# Recommended: Create a Velero scheduled backup
kubectl apply -f - << EOF
apiVersion: velero.io/v1
kind: Schedule
metadata:
  name: freshrss-daily
  namespace: velero
spec:
  schedule: "0 2 * * *"
  template:
    includedNamespaces:
      - freshrss
    storageLocation: default
EOF
```

---

## Notes

- **Image**: Deployment uses `ghcr.io/linuxserver/freshrss:1.28.1` (Helm-managed)
- **Storage**: Uses Longhorn for reliable persistent storage (10Gi PVC)
- **Configuration Directory**: `/config` in container (vs `/var/www/FreshRSS/data` in official image)
- **Database**: SQLite stored in `/config/db.sqlite`
- **User ID**: Runs as UID 568 (linuxserver standard, not 33/www-data)
- **DNS**: Ensure `freshrss.k8s.darktang3nt.cloud` resolves to your Envoy Gateway IP
- **SSL/TLS**: Handled by cert-manager (configured in Envoy Gateway setup)

---

## Quick Reference: Container Paths

| Component | Path in Container |
|-----------|-------------------|
| Config Directory | `/config` |
| Database | `/config/db.sqlite` |
| Feed Config | `/config/installs.php` |
| Web Root | `/app/www` |
| User ID | 568 (linuxserver) |
| Group ID | 568 (linuxserver) |
