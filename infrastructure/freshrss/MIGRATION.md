---
# =============================================================================
# FreshRSS Data Migration Guide
# =============================================================================
# Migrate FreshRSS from Docker to Kubernetes cluster
#
# Prerequisites:
# - Running Docker instance of FreshRSS with existing data
# - k3s cluster with ArgoCD installed
# - kubectl configured to access your k3s cluster
# - freshrss-app.yaml deployed to ArgoCD
# =============================================================================

## Step 1: Deploy FreshRSS to Kubernetes

First, ensure the FreshRSS application is created in ArgoCD:

```bash
# Verify the FreshRSS namespace and pod are running
kubectl get ns freshrss
kubectl get pods -n freshrss
```

Wait for the deployment to be ready before proceeding.

---

## Step 2: Backup Docker Data

Backup your existing FreshRSS data from Docker:

```bash
# Option A: Using docker exec (if container is running)
docker exec freshrss_container_name tar -czf - /var/www/FreshRSS/data > freshrss-backup.tar.gz

# Option B: Using docker cp (copy from stopped/running container)
docker cp freshrss_container_name:/var/www/FreshRSS/data ./freshrss-data-backup

# Option C: From volume mount (if you know the mount path)
tar -czf freshrss-backup.tar.gz /path/to/docker/volume/data
```

**Example using docker-compose** (if you have it):
```bash
docker-compose -f docker-compose.yml exec freshrss tar -czf - /var/www/FreshRSS/data > freshrss-backup.tar.gz
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

Connect to a cluster node and extract the data into the PVC:

```bash
# SSH into a cluster node
ssh root@k3s-node

# Find the actual PVC mount path on the node
# PVCs backed by Longhorn are typically in /var/lib/rancher/k3s/data/

# First, identify the PVC
kubectl get pvc -n freshrss

# Get the PVC details to find the backing volume
kubectl describe pvc freshrss-data -n freshrss | grep 'Volume:'

# Longhorn volumes are typically located at:
# /var/lib/rancher/k3s/data/longhorn/volumes/<volume-id>/replicas/<replica-id>/volume.meta

# For local access, you can mount the PVC's backing volume
# Option 1: If using a local node, directly extract to the volume path
cd /var/lib/rancher/k3s/data/longhorn/volumes/
ls -la

# Find the correct volume directory and extract
tar -xzf /tmp/freshrss-backup.tar.gz -C /var/lib/rancher/k3s/data/longhorn/volumes/<your-volume-id>/
```

### Alternative: Using kubectl exec (Recommended)

```bash
# Copy backup into the pod
kubectl cp ./freshrss-backup.tar.gz freshrss/freshrss:/tmp/backup.tar.gz

# Connect to pod and restore
kubectl exec -it -n freshrss deployment/freshrss -- sh

# Inside the pod:
cd /var/www/FreshRSS/data
tar -xzf /tmp/backup.tar.gz --strip-components=4
chown -R 33:33 .
exit
```

---

## Step 5: Restart the Pod

Force a restart to reload the data:

```bash
kubectl rollout restart deployment/freshrss -n freshrss

# Verify it's running
kubectl get pods -n freshrss -w
```

---

## Step 6: Verify Data Migration

```bash
# Check the logs
kubectl logs -n freshrss deployment/freshrss -f

# Verify the data was restored
kubectl exec -n freshrss deployment/freshrss -- ls -la /var/www/FreshRSS/data/

# Access FreshRSS via the HTTPRoute URL: https://freshrss.k8s.darktang3nt.cloud
```

---

## Step 7: Validate Feeds and Settings

1. Open your browser to `https://freshrss.k8s.darktang3nt.cloud`
2. Log in with your credentials
3. Verify all feeds are present and updating
4. Check that all your settings and read/unread status are preserved

---

## Troubleshooting

### Issue: Data folder is empty after restore
```bash
# Check PVC is properly mounted
kubectl describe pod -n freshrss deployment/freshrss

# Verify data directory permissions
kubectl exec -n freshrss deployment/freshrss -- ls -la /var/www/FreshRSS/
```

### Issue: Pod won't start after data restore
```bash
# Check pod logs for errors
kubectl logs -n freshrss deployment/freshrss --previous

# Verify file ownership (should be 33:33 for www-data)
kubectl exec -n freshrss deployment/freshrss -- stat /var/www/FreshRSS/data
```

### Issue: Database corruption
If the restored database is corrupted:
```bash
# Remove the corrupted database and let FreshRSS reinitialize
kubectl exec -it -n freshrss deployment/freshrss -- rm /var/www/FreshRSS/data/db.sqlite

# Then restart
kubectl rollout restart deployment/freshrss -n freshrss

# You'll need to re-add feeds, but data directory will be preserved
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
kubectl delete application freshrss -n argocd
```

---

## Notes

- **Storage**: The k3s deployment uses Longhorn for reliable persistent storage
- **Backup Strategy**: Consider setting up regular backups with Velero using MinIO backend
- **DNS**: Ensure `freshrss.k8s.darktang3nt.cloud` resolves to your Envoy Gateway IP
- **SSL/TLS**: Handled by cert-manager (configured in your Envoy Gateway setup)

---

## FreshRSS SQLite Database Location

After migration, the database should be at:
- Kubernetes: `/var/www/FreshRSS/data/db.sqlite`
- Original Docker: `/var/www/FreshRSS/data/db.sqlite`

If you want to export feeds as OPML before migration:
```bash
# From Docker container
docker exec freshrss_container_name curl -u admin:password http://localhost/api/opml.php > feeds.opml

# Import into k8s instance via web UI or API
curl -X POST -F 'opml_file=@feeds.opml' \
  -u admin:password \
  https://freshrss.k8s.darktang3nt.cloud/api/
```
