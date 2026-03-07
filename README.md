# homelab-gitops-k8s-cluster

GitOps repository for the **Monster Homelab Kubernetes Cluster** — a single-node bare-metal cluster
managed via ArgoCD. All manifests here are declarative and applied automatically by ArgoCD on push.

---

## Table of Contents

1. [Infrastructure Overview](#infrastructure-overview)
2. [Network & DNS Architecture](#network--dns-architecture)
3. [TLS & mTLS Architecture](#tls--mtls-architecture)
4. [Namespaces](#namespaces)
5. [Service Directory](#service-directory)
   - [CI/CD](#cicd)
   - [AI Services](#ai-services)
   - [Databases](#databases)
   - [Stack (Messaging & Cache)](#stack-messaging--cache)
   - [Storage](#storage)
   - [Observability](#observability)
   - [Security](#security)
6. [Ingress Reference Table](#ingress-reference-table)
7. [Secrets Management](#secrets-management)
8. [Persistent Storage](#persistent-storage)
9. [Accessing Services (mTLS Client Setup)](#accessing-services-mtls-client-setup)
10. [Repository Structure](#repository-structure)
11. [Sync Wave Order](#sync-wave-order)

---

## Infrastructure Overview

| Component         | Detail                                          |
|-------------------|-------------------------------------------------|
| **Node**          | `monster-workstation` · `192.168.1.50`          |
| **Kubernetes**    | Single-node bare-metal                          |
| **CNI**           | Cilium (eBPF, network policy, Hubble observability) |
| **Ingress**       | Traefik v3 (NodePort `:30443` → `websecure`)    |
| **GitOps**        | ArgoCD (continuous sync from this repo)         |
| **Secrets**       | External Secrets Operator → HashiCorp Vault KV v2 |
| **Storage**       | Longhorn (`longhorn-ssd` / `longhorn-hdd`)      |
| **GPU**           | NVIDIA (passed through to `ai-services`)        |
| **Gateway/DNS**   | `192.168.1.77` (AdGuard + gateway node)         |

---

## Network & DNS Architecture

```
Client (browser / CLI)
        │
        │  *.svc.k8s.monster.lab
        ▼
  AdGuard Home (192.168.1.77)
        │  Wildcard DNS rewrite → 192.168.1.77
        ▼
  Traefik NodePort (192.168.1.50:30443)
        │  SNI-based routing
        ▼
  Kubernetes Service → Pod
```

**DNS wildcard rule** (configured in AdGuard):
```
*.svc.k8s.monster.lab  →  192.168.1.77
```
Traefik on the gateway forwards to `192.168.1.50:30443` (the workstation node), where the actual
K8s Traefik ingress controller routes to in-cluster services based on the `Host()` header.

CoreDNS is **not** modified — all external DNS resolution goes through AdGuard.

---

## TLS & mTLS Architecture

All exposed services use TLS terminated at Traefik. Services are protected by one of two
`TLSOption` policies:

| Policy Name        | `clientAuthType`          | Client Cert Required | Used By |
|--------------------|---------------------------|----------------------|---------|
| `mtls-strict`      | `RequireAndVerifyClientCert` | **Yes** — must present a cert signed by internal CA | Most services |
| `no-client-cert`   | `NoClientCert`            | No                   | Coder, Postgres TCP |

**Certificates:**
- `monster-server-cert` — Wildcard server certificate (`*.svc.k8s.monster.lab`) mounted on Traefik
- `monster-ca-trust` — Internal CA bundle (`ca.crt`) for client cert verification + pod CA trust

**To access mTLS-protected services**, your HTTP client must present a client certificate signed
by the internal Monster CA. See [Accessing Services](#accessing-services-mtls-client-setup).

---

## Namespaces

| Namespace      | Purpose                                      |
|----------------|----------------------------------------------|
| `ai-services`  | GPU-accelerated AI workloads (Ollama, ComfyUI) |
| `argocd`       | ArgoCD GitOps controller                     |
| `ci-cd`        | Developer tooling (Coder, Gitea, Drone)      |
| `database`     | Persistent data stores (Postgres, Qdrant, MongoDB) |
| `external-secrets` | External Secrets Operator             |
| `observability`| LLM/ML observability (Langfuse)              |
| `security`     | HashiCorp Vault                              |
| `stack`        | Application infrastructure (RabbitMQ, Redis) |
| `storage`      | Object storage (MinIO)                       |

---

## Service Directory

### CI/CD

#### Coder
Remote development environment platform — provision workspaces backed by K8s.

| Field         | Value                                          |
|---------------|------------------------------------------------|
| **URL**       | `https://coder.svc.k8s.monster.lab`            |
| **Namespace** | `ci-cd`                                        |
| **Image**     | `ghcr.io/coder/coder:v2.29.7`                  |
| **TLS Mode**  | `no-client-cert` (no mTLS — browser accessible) |
| **Port**      | `3000`                                         |
| **Replicas**  | 1                                              |
| **Resources** | req: 100m CPU / 256Mi RAM · lim: 500m CPU / 512Mi RAM |
| **Database**  | PostgreSQL (`coder` DB via `CODER_DB_PASSWORD`) |
| **Notes**     | `SSL_CERT_FILE` set to internal CA so Coder can reach `coder.svc.k8s.monster.lab`; uses `hostAliases` to resolve the URL inside-pod |

---

#### Gitea
Self-hosted Git service. Drone CI OAuth app is registered here.

| Field         | Value                                          |
|---------------|------------------------------------------------|
| **URL**       | `https://gitea.svc.k8s.monster.lab`            |
| **Namespace** | `ci-cd`                                        |
| **Image**     | `gitea/gitea:1.25-rootless`                    |
| **TLS Mode**  | `mtls-strict` 🔒                               |
| **Ports**     | `3000` (HTTP) · `22` (SSH, internal)           |
| **Replicas**  | 1                                              |
| **Resources** | req: 100m CPU / 128Mi RAM · lim: 500m CPU / 512Mi RAM |
| **Database**  | PostgreSQL (`gitea` DB via `GITEA_DB_PASSWORD`) |
| **Storage**   | 5Gi `longhorn-hdd` (repositories, LFS, avatars) |
| **Config**    | Injected via ConfigMap → `app.ini` init container |

---

#### Drone CI
CI/CD pipeline runner integrated with Gitea via OAuth.

| Field         | Value                                          |
|---------------|------------------------------------------------|
| **URL**       | `https://drone.svc.k8s.monster.lab`            |
| **Namespace** | `ci-cd`                                        |
| **Image**     | `drone/drone:2.27.1`                           |
| **TLS Mode**  | `mtls-strict` 🔒                               |
| **Port**      | `80`                                           |
| **Replicas**  | 1                                              |
| **Resources** | req: 50m CPU / 64Mi RAM · lim: 250m CPU / 256Mi RAM |
| **Database**  | PostgreSQL (`drone` DB via `DRONE_DB_PASSWORD`) |
| **Secrets**   | `DRONE_RPC_SECRET`, `DRONE_GITEA_CLIENT_ID/SECRET` from Vault |

---

### AI Services

#### Ollama
LLM inference server. Models served over REST API.

| Field         | Value                                          |
|---------------|------------------------------------------------|
| **URL**       | `https://ollama.svc.k8s.monster.lab`           |
| **Namespace** | `ai-services`                                  |
| **Image**     | `ollama/ollama:0.17.5`                         |
| **TLS Mode**  | `mtls-strict` 🔒                               |
| **Port**      | `11434`                                        |
| **Replicas**  | 1                                              |
| **Resources** | req: 500m CPU / 2Gi RAM · lim: 4 CPU / 8Gi RAM |
| **Storage**   | 50Gi `longhorn-hdd` (model weights)            |
| **GPU**       | NVIDIA GPU passthrough enabled                 |
| **API**       | `POST /api/generate` · `POST /api/chat` · `GET /api/tags` |

---

#### ComfyUI
Node-based Stable Diffusion / image generation UI.

| Field         | Value                                          |
|---------------|------------------------------------------------|
| **URL**       | `https://comfyui.svc.k8s.monster.lab`          |
| **Namespace** | `ai-services`                                  |
| **Image**     | `yanwk/comfyui-boot:cu128-megapak` (CUDA 12.8) |
| **TLS Mode**  | `mtls-strict` 🔒                               |
| **Port**      | `8188`                                         |
| **Replicas**  | 1                                              |
| **Resources** | req: 500m CPU / 2Gi RAM · lim: 4 CPU / 8Gi RAM |
| **Storage**   | 30Gi `longhorn-hdd` (models, outputs, workflows) |
| **GPU**       | NVIDIA GPU passthrough enabled                 |

---

### Databases

#### PostgreSQL
Central relational database. Serves multiple applications via per-app databases and users.

| Field         | Value                                          |
|---------------|------------------------------------------------|
| **External URL** | `postgres.svc.k8s.monster.lab:443` (TCP TLS passthrough) |
| **Connection** | `psql "sslmode=verify-ca host=postgres.svc.k8s.monster.lab port=443 ..."` |
| **Namespace** | `database`                                     |
| **Image**     | `postgres:18-alpine3.23`                       |
| **TLS Mode**  | `no-client-cert` (IngressRouteTCP — raw TCP passthrough) |
| **Port**      | `5432`                                         |
| **Replicas**  | 1                                              |
| **Resources** | req: 250m CPU / 512Mi RAM · lim: 1 CPU / 1536Mi RAM |
| **Storage**   | 10Gi `longhorn-hdd`                            |
| **Databases** | `postgres` (admin) · `coder` · `gitea` · `drone` · `langfuse` · `mlflow` |
| **Init**      | ConfigMap-backed init SQL creates per-app DBs on first boot |

---

#### Qdrant
High-performance vector database for AI/ML embeddings.

| Field         | Value                                          |
|---------------|------------------------------------------------|
| **UI / REST** | `https://qdrant.svc.k8s.monster.lab`           |
| **Dashboard** | `https://qdrant.svc.k8s.monster.lab/dashboard` |
| **REST alias**| `https://api-qdrant.svc.k8s.monster.lab`       |
| **Namespace** | `database`                                     |
| **Image**     | `qdrant/qdrant:v1.17-unprivileged`             |
| **TLS Mode**  | `mtls-strict` 🔒                               |
| **Ports**     | `6333` (HTTP/REST + Dashboard) · `6334` (gRPC, internal) |
| **Replicas**  | 1                                              |
| **Resources** | req: 100m CPU / 256Mi RAM · lim: 500m CPU / 512Mi RAM |
| **Storage**   | 5Gi `longhorn-ssd` (fast NVMe for vector index) |
| **Probes**    | `/readyz` (ready) · `/healthz` (live)          |

---

#### MongoDB
Document database for unstructured / semi-structured workloads.

| Field         | Value                                          |
|---------------|------------------------------------------------|
| **URL**       | Internal only (`mongodb.database.svc:27017`)   |
| **Namespace** | `database`                                     |
| **Image**     | `mongo:8.0`                                    |
| **Port**      | `27017` (internal cluster only)                |
| **Replicas**  | 1                                              |
| **Resources** | req: 100m CPU / 256Mi RAM · lim: 500m CPU / 512Mi RAM |
| **Storage**   | 5Gi `longhorn-hdd`                             |
| **HPA**       | Horizontal Pod Autoscaler configured           |
| **Notes**     | No external ingress — accessed internally via service DNS |

---

### Stack (Messaging & Cache)

#### RabbitMQ
AMQP message broker with management UI.

| Field         | Value                                          |
|---------------|------------------------------------------------|
| **URL**       | `https://rabbitmq.svc.k8s.monster.lab`         |
| **Namespace** | `stack`                                        |
| **Image**     | `rabbitmq:3.13-management-alpine`              |
| **TLS Mode**  | `mtls-strict` 🔒                               |
| **Ports**     | `5672` (AMQP, internal) · `15672` (Management UI, exposed) |
| **Replicas**  | 1                                              |
| **Resources** | req: 50m CPU / 128Mi RAM · lim: 500m CPU / 384Mi RAM |
| **Storage**   | 2Gi `longhorn-hdd` (message persistence / mnesia) |
| **Credentials** | `RABBITMQ_DEFAULT_USER/PASS` from Vault → `rabbitmq-credentials` secret |
| **Config**    | `rabbitmq.conf` via ConfigMap                  |

---

#### Redis
In-memory data store — used as cache/session store by Langfuse.

| Field         | Value                                          |
|---------------|------------------------------------------------|
| **URL**       | Internal only (`redis.stack.svc:6379`)         |
| **Namespace** | `stack`                                        |
| **Image**     | `redis/redis-stack-server:7.4.0-v8`           |
| **Port**      | `6379` (internal cluster only)                 |
| **Replicas**  | 1                                              |
| **Resources** | req: 50m CPU / 128Mi RAM · lim: 500m CPU / 384Mi RAM |
| **Notes**     | No external ingress — Redis Stack includes RediSearch, RedisJSON |

---

### Storage

#### MinIO
S3-compatible object storage — used by Langfuse, MLflow, and general artifact storage.

| Field          | Value                                          |
|----------------|------------------------------------------------|
| **Console UI** | `https://minio.svc.k8s.monster.lab`            |
| **S3 API**     | `https://s3.svc.k8s.monster.lab`               |
| **Namespace**  | `storage`                                      |
| **Image**      | `minio/minio:RELEASE.2025-09-07T16-13-09Z`     |
| **TLS Mode**   | `mtls-strict` 🔒                               |
| **Ports**      | `9001` (Console) · `9000` (S3 API)             |
| **Replicas**   | 1                                              |
| **Resources**  | req: 50m CPU / 128Mi RAM · lim: 500m CPU / 512Mi RAM |
| **Storage**    | 50Gi `longhorn-hdd`                            |
| **Credentials**| `MINIO_ROOT_USER/PASSWORD` from Vault → `minio-credentials` secret |
| **S3 endpoint**| `https://s3.svc.k8s.monster.lab` (for SDK/CLI `--endpoint-url`) |

---

### Observability

#### Langfuse
LLM observability and prompt management platform (traces, evals, datasets).

| Field         | Value                                          |
|---------------|------------------------------------------------|
| **URL**       | `https://langfuse.svc.k8s.monster.lab`         |
| **Namespace** | `observability`                                |
| **TLS Mode**  | `mtls-strict` 🔒                               |
| **Replicas**  | 1                                              |
| **Resources** | web: req 50m/256Mi · lim 500m/1Gi · worker: req 100m/256Mi · lim 500m/1Gi |
| **Database**  | PostgreSQL (`langfuse` DB via `LANGFUSE_DB_PASSWORD`) |
| **Cache**     | Redis (`redis.stack.svc:6379`)                 |
| **Storage**   | MinIO (`s3.svc.k8s.monster.lab`)               |
| **Secrets**   | `NEXTAUTH_SECRET`, `SALT`, MinIO creds from Vault |
| **Deploy**    | Managed as ArgoCD `Application` YAML (`application.yml`) |

---

### Security

#### HashiCorp Vault
Secrets management backend. All application credentials are stored here and synced to K8s secrets
by the External Secrets Operator.

| Field         | Value                                          |
|---------------|------------------------------------------------|
| **URL**       | `https://vault.svc.k8s.monster.lab`            |
| **Namespace** | `security`                                     |
| **Port**      | `8200`                                         |
| **KV Engine** | `secret/` (KV v2)                              |
| **Auth**      | Token auth (ESO uses a long-lived token stored in `vault-token` secret) |

**Vault Secret Paths:**

| Path                             | Keys                                                    | Consumers |
|----------------------------------|---------------------------------------------------------|-----------|
| `secret/data/storage/minio`      | `MINIO_ROOT_USER`, `MINIO_ROOT_PASSWORD`                | MinIO, Langfuse |
| `secret/data/database/postgres`  | `POSTGRES_PASSWORD`, `CODER_DB_PASSWORD`, `GITEA_DB_PASSWORD`, `DRONE_DB_PASSWORD`, `LANGFUSE_DB_PASSWORD`, `MLFLOW_DB_PASSWORD` | Postgres, all app DBs |
| `secret/data/stack/rabbitmq`     | `RABBITMQ_DEFAULT_USER`, `RABBITMQ_DEFAULT_PASS`        | RabbitMQ |
| `secret/data/observability/langfuse` | `NEXTAUTH_SECRET`, `SALT`                           | Langfuse |
| `secret/data/ci/drone`           | `DRONE_RPC_SECRET`, `DRONE_GITEA_CLIENT_ID`, `DRONE_GITEA_CLIENT_SECRET` | Drone |

#### ArgoCD
GitOps controller. Watches this repository and reconciles cluster state.

| Field         | Value                                          |
|---------------|------------------------------------------------|
| **URL**       | `https://argocd-server.svc.k8s.monster.lab`    |
| **Namespace** | `argocd`                                       |
| **Ingress**   | `IngressRouteTCP` (TLS passthrough, SNI)        |
| **Notes**     | Deployed by Ansible (`kubernetes_core_workstations` role), not managed in this repo |

---

## Ingress Reference Table

> 🔒 = mTLS required (client certificate from internal Monster CA)

| Service       | URL                                              | Protocol    | Backend Port | TLS Policy       |
|---------------|--------------------------------------------------|-------------|-------------|------------------|
| ArgoCD        | `argocd-server.svc.k8s.monster.lab`              | HTTPS (TCP) | 443         | Passthrough      |
| Coder         | `coder.svc.k8s.monster.lab`                      | HTTPS       | 3000        | `no-client-cert` |
| ComfyUI       | `comfyui.svc.k8s.monster.lab`                    | HTTPS       | 8188        | `mtls-strict` 🔒 |
| Drone CI      | `drone.svc.k8s.monster.lab`                      | HTTPS       | 80          | `mtls-strict` 🔒 |
| Gitea         | `gitea.svc.k8s.monster.lab`                      | HTTPS       | 3000        | `mtls-strict` 🔒 |
| Langfuse      | `langfuse.svc.k8s.monster.lab`                   | HTTPS       | 3000        | `mtls-strict` 🔒 |
| MinIO Console | `minio.svc.k8s.monster.lab`                      | HTTPS       | 9001        | `mtls-strict` 🔒 |
| MinIO S3 API  | `s3.svc.k8s.monster.lab`                         | HTTPS       | 9000        | `mtls-strict` 🔒 |
| Ollama        | `ollama.svc.k8s.monster.lab`                     | HTTPS       | 11434       | `mtls-strict` 🔒 |
| PostgreSQL    | `postgres.svc.k8s.monster.lab:443`               | TCP/TLS     | 5432        | `no-client-cert` |
| Qdrant UI/API | `qdrant.svc.k8s.monster.lab`                     | HTTPS       | 6333        | `mtls-strict` 🔒 |
| Qdrant API alt| `api-qdrant.svc.k8s.monster.lab`                 | HTTPS       | 6333        | `mtls-strict` 🔒 |
| RabbitMQ Mgmt | `rabbitmq.svc.k8s.monster.lab`                   | HTTPS       | 15672       | `mtls-strict` 🔒 |
| Vault         | `vault.svc.k8s.monster.lab`                      | HTTPS       | 8200        | —               |

**Internal-only (no ingress):**
- MongoDB: `mongodb.database.svc:27017`
- Redis: `redis.stack.svc:6379`
- RabbitMQ AMQP: `rabbitmq.stack.svc:5672`
- Qdrant gRPC: `qdrant.database.svc:6334`

---

## Secrets Management

All secrets follow a two-tier pattern:

```
HashiCorp Vault (KV v2)
        │
        │  ExternalSecret CR (polling interval: 1h)
        ▼
External Secrets Operator
        │  reconciles
        ▼
Kubernetes Secret (e.g. minio-credentials, postgres-credentials)
        │
        │  envFrom / secretKeyRef
        ▼
Application Pod
```

The `ClusterSecretStore` (`vault-backend`) connects to Vault at
`http://vault.security.svc:8200` using a token from the `vault-token` secret
in the `external-secrets` namespace.

**To add a new secret:**
1. Write the value to Vault: `vault kv put secret/your/path KEY=value`
2. Create an `ExternalSecret` manifest referencing `vault-backend` as the store
3. ArgoCD syncs the `ExternalSecret`; ESO creates the K8s `Secret`

---

## Persistent Storage

All volumes use [Longhorn](https://longhorn.io) with two storage classes:

| StorageClass    | Backing Medium | Use Case                  |
|-----------------|----------------|---------------------------|
| `longhorn-ssd`  | NVMe SSD       | Latency-sensitive (Qdrant vectors) |
| `longhorn-hdd`  | HDD            | Bulk storage (models, DB data, repos) |

| Service       | PVC Name         | Size   | StorageClass    |
|---------------|------------------|--------|-----------------|
| ComfyUI       | `comfyui-data`   | 30Gi   | `longhorn-hdd`  |
| Gitea         | `gitea-data`     | 5Gi    | `longhorn-hdd`  |
| MinIO         | `minio-data`     | 50Gi   | `longhorn-hdd`  |
| MongoDB       | `mongodb-data`   | 5Gi    | `longhorn-hdd`  |
| Ollama        | `ollama-models`  | 50Gi   | `longhorn-hdd`  |
| PostgreSQL    | `postgres-data`  | 10Gi   | `longhorn-hdd`  |
| Qdrant        | `qdrant-data`    | 5Gi    | `longhorn-ssd`  |
| RabbitMQ      | `rabbitmq-data`  | 2Gi    | `longhorn-hdd`  |

---

## Accessing Services (mTLS Client Setup)

Most services require a client certificate signed by the internal Monster CA.

### Browser (Firefox / Chrome)
1. Obtain the client cert+key (`client.crt` + `client.key`) from the PKI admin (Ansible `pki` role)
2. Bundle to PKCS12: `openssl pkcs12 -export -in client.crt -inkey client.key -out client.p12`
3. Import `client.p12` into browser certificate store
4. Import `ca.crt` as a trusted CA authority

### curl
```bash
curl --cert client.crt --key client.key --cacert ca.crt \
  https://qdrant.svc.k8s.monster.lab/dashboard
```

### Python / httpx
```python
import httpx

client = httpx.Client(
    cert=("client.crt", "client.key"),
    verify="ca.crt"
)
resp = client.get("https://qdrant.svc.k8s.monster.lab/collections")
```

### MinIO CLI (`mc`)
```bash
mc alias set homelab https://s3.svc.k8s.monster.lab \
  "$MINIO_ROOT_USER" "$MINIO_ROOT_PASSWORD" \
  --api S3v4 \
  --config-dir ~/.mc \
  --insecure  # or add CA via system trust store
```

### PostgreSQL
```bash
psql "host=postgres.svc.k8s.monster.lab port=443 \
      sslmode=require dbname=coder user=coder \
      password=$CODER_DB_PASSWORD"
```
> Postgres TCP is exposed via TLS passthrough — `sslmode=require` is sufficient,
> no client cert needed.

### Ollama API
```bash
curl --cert client.crt --key client.key --cacert ca.crt \
  https://ollama.svc.k8s.monster.lab/api/tags
```

---

## Repository Structure

```
homelab-gitops-k8s-cluster/
├── ai_services/
│   ├── comfyui/          # Stable Diffusion UI (CUDA 12.8)
│   └── ollama/           # LLM inference server
├── apps/                 # Reserved — general apps
├── ci_cd/
│   ├── coder/            # Remote dev environments
│   ├── drone/            # CI/CD pipelines
│   └── gitea/            # Self-hosted Git
├── database/
│   ├── mongodb/          # Document store
│   ├── postgres/         # Relational DB (shared, multi-tenant)
│   └── qdrant/           # Vector database
├── dev_env/              # Coder workspace templates / namespace config
├── observability/
│   ├── langfuse/         # LLM observability (ArgoCD Application)
│   ├── mlflow/           # ML experiment tracking
│   └── otel/             # OpenTelemetry collector
├── security/
│   ├── external_secrets_operator/
│   └── hashi_vault/      # ClusterSecretStore definition
├── stack/
│   ├── rabbitmq/         # Message broker
│   └── redis/            # Cache / session store
└── storage/
    └── minio/            # S3-compatible object storage
```

---

## Sync Wave Order

ArgoCD sync waves control dependency ordering. The pattern used across manifests:

| Wave | Resources                                 |
|------|-------------------------------------------|
| `0`  | Namespaces, CRDs                          |
| `1`  | Vault, ESO ClusterSecretStore, PVCs       |
| `2`  | ExternalSecrets (secrets must exist before pods) |
| `3`  | Deployments, Services, Ingresses          |
| `4`  | HPA, NetworkPolicies                      |

> Tip: If a deployment fails with `CreateContainerConfigError`, check that the
> ExternalSecret in wave `2` has successfully synced (`kubectl get externalsecret -n <ns>`).

---

*Last updated: 2026-03-06 · Managed by: ArgoCD · Source of truth: this repo*
