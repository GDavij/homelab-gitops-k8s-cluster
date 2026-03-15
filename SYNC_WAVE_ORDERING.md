# ArgoCD Sync Wave Ordering

## Overview
This document describes the sync wave ordering for the GitOps-managed Kubernetes cluster, ensuring proper dependency resolution and preventing race conditions.

## Sync Wave Order

### Wave 0: Namespaces & Base Infrastructure
- All namespaces (security, external-secrets, ai-services, ci-cd, database, stack, storage, dev-env)
- Storage provisioner (local-path)
- Vault service account

### Wave 1: Transit Vault Deployment
- **Transit Vault deployment** (first Vault component)
- Transit Vault configmap and service

### Wave 2: Transit Vault Initialization
- **Transit init job** (initializes and unseals Transit Vault)
- **Vault init RBAC** (ServiceAccount, Role, RoleBinding)
- Transit Vault is now ready for auto-unseal

### Wave 3: Main Vault Deployment
- **Main Vault deployment** (uses Transit Vault for auto-unseal)
- Main Vault configmap, service, and storage
- Traefik ingress controller (RBAC, service, deployment)

### Wave 4: Infrastructure Services
- Storage, services, configmaps for databases and applications
- Network policies

### Wave 5: Main Vault Initialization
- **Vault init job** (initializes Main Vault, seeds secrets, creates ESO token)
- Main Vault is now fully operational

### Wave 6: External Secrets Operator Integration
- **ClusterSecretStore** (connects to Vault using ESO token)
- Vault ingress

### Wave 7: ExternalSecrets
- All ExternalSecrets (coder, drone, gitea, minio, langfuse, rabbitmq, postgres, dev-env)
- Secrets are now available in Kubernetes

### Wave 8: Application Deployments
- All application deployments (after secrets are available)
- Databases, CI/CD tools, AI services, observability stack

## Critical Dependencies

### Transit Vault → Main Vault
```
Transit Vault (wave 1) 
  → Transit Init Job (wave 2, initializes and unseals Transit Vault)
  → Main Vault (wave 3, auto-unseals using Transit Vault)
```

**Why this order is critical:**
- Main Vault uses Transit Vault for auto-unseal
- Transit Vault must be initialized and unsealed BEFORE Main Vault starts
- If Main Vault starts before Transit Vault is ready, it will fail with "Vault is sealed" error

### Vault → ExternalSecrets → Applications
```
Main Vault (wave 3)
  → Vault Init Job (wave 5, seeds secrets and creates ESO token)
  → ClusterSecretStore (wave 6)
  → ExternalSecrets (wave 7)
  → Application Deployments (wave 8)
```

## Key Changes

1. **Split vault-init-job into two jobs:**
   - `transit-init-job` (wave 2): Initializes and unseals Transit Vault
   - `vault-init-job` (wave 5): Initializes Main Vault and seeds secrets

2. **Moved Main Vault deployment to wave 3** (after Transit Vault is initialized)

3. **Moved all deployments to wave 8** (after secrets are available)

## Troubleshooting

### Error: "Vault is sealed"
**Cause:** Main Vault is trying to auto-unseal from Transit Vault, but Transit Vault is still sealed.

**Solution:** Ensure Transit Vault is initialized and unsealed before Main Vault starts:
1. Transit Vault deployment (wave 1)
2. Transit init job (wave 2)
3. Main Vault deployment (wave 3)

### Error: ExternalSecret cannot connect to Vault
**Cause:** ClusterSecretStore or ExternalSecrets are created before Vault is ready.

**Solution:** Ensure proper ordering:
1. Vault init job creates ESO token (wave 5)
2. ClusterSecretStore uses ESO token (wave 6)
3. ExternalSecrets use ClusterSecretStore (wave 7)

## Validation

To validate the sync wave ordering:
```bash
cd homelab-gitops-k8s-cluster
grep -r "argocd.argoproj.io/sync-wave" --include="*.yml" | sort -t'"' -k2 -n
```

## References

- [ArgoCD Sync Waves](https://argo-cd.readthedocs.io/en/stable/user-guide/sync-waves/)
- [HashiCorp Vault Auto-Unseal](https://developer.hashicorp.com/vault/docs/concepts/seal#auto-unseal)
- [External Secrets Operator](https://external-secrets.io/)
