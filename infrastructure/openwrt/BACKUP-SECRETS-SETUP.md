---
# =============================================================================
# OpenWrt Backup — Credentials
# =============================================================================
# IMPORTANT: These secrets must be created MANUALLY before the CronJob syncs.
# Never commit real secret values to Git.
#
# ssh-privatekey: dedicated key for the backup CronJob only (not the human
#   admin key) — public half is appended to /etc/dropbear/authorized_keys
#   on the router, alongside the existing admin key, not replacing it.
#
# minio-credentials: scoped MinIO user limited to the openwrt-backups bucket
#   only via an attached policy — NOT the MinIO root credentials, so a
#   compromised router can't reach Velero/Longhorn/Loki/Tempo data.
#
#   kubectl create secret generic openwrt-backup-ssh-key \
#     --from-file=id_ed25519=<path-to-private-key> \
#     -n monitoring
#
#   kubectl create secret generic openwrt-backup-minio-credentials \
#     --from-literal=access-key=<scoped-access-key> \
#     --from-literal=secret-key=<scoped-secret-key> \
#     -n monitoring
# =============================================================================
