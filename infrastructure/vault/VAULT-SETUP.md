# Vault Setup Guide — Manual Post-Install Steps

Vault cannot be fully automated on first install. These steps must be run
ONCE after ArgoCD deploys the Vault Helm chart.

---

## Prerequisites

Before ArgoCD syncs vault-app.yaml, create the AWS KMS secret manually:

```bash
# Create AWS KMS credentials secret
# Replace values with your actual AWS credentials
kubectl create secret generic vault-kms-credentials \
  --from-literal=AWS_ACCESS_KEY_ID=<YOUR_ACCESS_KEY> \
  --from-literal=AWS_SECRET_ACCESS_KEY=<YOUR_SECRET_KEY> \
  --from-literal=KMS_KEY_ID=<YOUR_KMS_KEY_ID> \
  --from-literal=AWS_REGION=us-east-1 \
  -n security

# Verify secret created
kubectl get secret vault-kms-credentials -n security
```

> This is the ONLY secret ever stored outside Vault.
> Everything else goes INTO Vault and out via ESO.

---

## Step 1 — Verify Vault Pod is Running

```bash
kubectl get pods -n security
# vault-0   0/1   Running   0   2m
# Status 0/1 is normal — Vault starts sealed, not ready until initialized
```

---

## Step 2 — Initialize Vault

```bash
# exec into vault pod
kubectl exec -it vault-0 -n security -- vault operator init \
  -key-shares=5 \
  -key-threshold=3 \
  -format=json > vault-init.json

# CRITICAL: Save vault-init.json somewhere safe (password manager, printed)
# This file contains:
#   - 5 unseal keys (need 3 of 5 to unseal manually if KMS fails)
#   - root token (initial admin access)
# NEVER commit this file to Git
cat vault-init.json
```

With AWS KMS auto-unseal, Vault unseals automatically after init.
Verify:

```bash
kubectl exec -it vault-0 -n security -- vault status
# Initialized: true
# Sealed:      false   ← KMS auto-unsealed ✅
```

---

## Step 3 — Login with Root Token

```bash
# get root token from init output
ROOT_TOKEN=$(cat vault-init.json | jq -r '.root_token')

# login
kubectl exec -it vault-0 -n security -- \
  vault login $ROOT_TOKEN
```

---

## Step 4 — Enable Audit Logging

```bash
kubectl exec -it vault-0 -n security -- \
  vault audit enable file file_path=/vault/audit/vault-audit.log

# verify
kubectl exec -it vault-0 -n security -- vault audit list
```

Audit logs ship to Loki via Promtail (configured when monitoring is deployed).

---

## Step 5 — Enable KV v2 Secret Engine

```bash
kubectl exec -it vault-0 -n security -- \
  vault secrets enable -path=secret kv-v2

# verify
kubectl exec -it vault-0 -n security -- vault secrets list
```

---

## Step 6 — Enable Kubernetes Auth Method

```bash
kubectl exec -it vault-0 -n security -- \
  vault auth enable kubernetes

# configure k8s auth
kubectl exec -it vault-0 -n security -- \
  vault write auth/kubernetes/config \
    kubernetes_host="https://kubernetes.default.svc:443"

# verify
kubectl exec -it vault-0 -n security -- vault auth list
```

---

## Step 7 — Create Namespaced Policies

```bash
# cert-manager policy — read Cloudflare token only
kubectl exec -it vault-0 -n security -- vault policy write cert-manager - <<EOF
path "secret/data/cloudflare/*" {
  capabilities = ["read"]
}
EOF

# Create k8s auth role for cert-manager
kubectl exec -it vault-0 -n security -- \
  vault write auth/kubernetes/role/cert-manager \
    bound_service_account_names=cert-manager \
    bound_service_account_namespaces=cert-manager \
    policies=cert-manager \
    ttl=1h
```

---

## Step 8 — Store Initial Secrets in Vault

```bash
# Cloudflare API token (used by cert-manager for DNS-01)
kubectl exec -it vault-0 -n security -- \
  vault kv put secret/cloudflare/api-token \
    api-token=<YOUR_CLOUDFLARE_TOKEN>

# verify
kubectl exec -it vault-0 -n security -- \
  vault kv get secret/cloudflare/api-token
```

---

## Step 9 — Revoke Root Token

After setup is complete, revoke the root token for security.
Generate a new one only when needed via unseal keys.

```bash
kubectl exec -it vault-0 -n security -- \
  vault token revoke $ROOT_TOKEN
```

---

## Verification

```bash
# Vault status
kubectl exec -it vault-0 -n security -- vault status

# List secret engines
kubectl exec -it vault-0 -n security -- vault secrets list

# List auth methods
kubectl exec -it vault-0 -n security -- vault auth list

# List policies
kubectl exec -it vault-0 -n security -- vault policy list

# Access UI
# vault.k8s.darktang3nt.cloud (via Twingate)
```

---

## Auto-Unseal Verification

Test that AWS KMS auto-unseal works:

```bash
# delete vault pod — simulates reboot
kubectl delete pod vault-0 -n security

# watch it come back
kubectl get pods -n security -w

# check it auto-unsealed (Sealed: false)
kubectl exec -it vault-0 -n security -- vault status
```

If `Sealed: false` after restart without manual unseal — KMS is working ✅
