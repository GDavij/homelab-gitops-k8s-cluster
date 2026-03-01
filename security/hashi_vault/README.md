# HashiCorp Vault

Vault is deployed separately via Ansible (not managed by ArgoCD).
See: `ansible_automation/roles/kubernetes_cni_workstations/tasks/vault.yml`

## ESO Integration

The `ClusterSecretStore` connects ESO to Vault at `http://vault.security.svc:8200`.
Authentication uses a scoped ESO token (policy: `eso-reader`, read-only on `secret/data/*`).

## Vault KV Secret Paths

After unsealing Vault, populate these paths. Generate strong passwords with `openssl rand -base64 24`.

```bash
# Database — all per-service passwords + full connection URLs
vault kv put secret/database/postgres \
  POSTGRES_PASSWORD="<generated>" \
  CODER_DB_PASSWORD="<generated>" \
  CODER_DB_URL="postgres://coder:<CODER_DB_PASSWORD>@postgres.database.svc:5432/coder?sslmode=disable" \
  MLFLOW_DB_PASSWORD="<generated>" \
  MLFLOW_DB_URL="postgresql://mlflow:<MLFLOW_DB_PASSWORD>@postgres.database.svc:5432/mlflow" \
  LANGFUSE_DB_PASSWORD="<generated>" \
  LANGFUSE_DB_URL="postgresql://langfuse:<LANGFUSE_DB_PASSWORD>@postgres.database.svc:5432/langfuse" \
  GITEA_DB_PASSWORD="<generated>" \
  DRONE_DB_PASSWORD="<generated>" \
  DRONE_DB_URL="postgres://drone:<DRONE_DB_PASSWORD>@postgres.database.svc:5432/drone?sslmode=disable"

# Storage
vault kv put secret/storage/minio \
  MINIO_ROOT_USER="minioadmin" \
  MINIO_ROOT_PASSWORD="<generated>"

# Observability
vault kv put secret/observability/mlflow \
  AWS_ACCESS_KEY_ID="minioadmin" \
  AWS_SECRET_ACCESS_KEY="<same-as-minio-password>"

vault kv put secret/observability/langfuse \
  NEXTAUTH_SECRET="$(openssl rand -base64 32)" \
  SALT="$(openssl rand -base64 32)"

# CI/CD
vault kv put secret/ci-cd/drone \
  DRONE_RPC_SECRET="$(openssl rand -hex 16)" \
  DRONE_GITEA_CLIENT_ID="<from-gitea-oauth>" \
  DRONE_GITEA_CLIENT_SECRET="<from-gitea-oauth>"

vault kv put secret/ci-cd/coder \
  CODER_PG_CONNECTION_URL="postgres://coder:<CODER_DB_PASSWORD>@postgres.database.svc:5432/coder?sslmode=disable"
```

## Policy (minimal)

```bash
vault policy write eso-reader - <<EOF
path "secret/data/*" {
  capabilities = ["read"]
}
path "secret/metadata/*" {
  capabilities = ["read", "list"]
}
EOF
```
