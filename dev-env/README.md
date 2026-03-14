# Dev Environments

Shared infrastructure for development workspaces running inside the cluster.

## What's Provided

Every dev pod in the `dev-env` namespace gets access to:

| Service | Internal DNS | Port |
|---------|-------------|------|
| PostgreSQL | `postgres.database.svc` | 5432 |
| Qdrant | `qdrant.database.svc` | 6333 (HTTP), 6334 (gRPC) |
| Redis Stack | `redis-stack.stack.svc` | 6379 |
| RabbitMQ | `rabbitmq.stack.svc` | 5672 (AMQP), 15672 (UI) |
| MinIO (S3) | `minio.storage.svc` | 9000 (API), 9001 (Console) |
| Ollama | `ollama.ai-services.svc` | 11434 |
| MLflow | `mlflow.observability.svc` | 5000 |
| Langfuse | `langfuse.observability.svc` | 3000 |
| OTEL Collector | `otel-collector.observability.svc` | 4317 (gRPC), 4318 (HTTP) |
| Vault | `vault.security.svc` | 8200 |

## Usage in a Dev Pod

```yaml
spec:
  containers:
    - name: dev
      envFrom:
        # All service endpoints as env vars
        - configMapRef:
            name: devenv-service-endpoints
```

Then in your code:

```python
import os

# Postgres — build connection string from env vars
pg_url = f"postgresql://{os.environ['POSTGRES_USER']}:{os.environ['POSTGRES_PASSWORD']}@{os.environ['POSTGRES_HOST']}:{os.environ['POSTGRES_PORT']}/mydb"

# Qdrant
from qdrant_client import QdrantClient
client = QdrantClient(host=os.environ["QDRANT_HOST"], port=int(os.environ["QDRANT_HTTP_PORT"]))

# Redis
import redis
r = redis.Redis(host=os.environ["REDIS_HOST"], port=int(os.environ["REDIS_PORT"]))

# MinIO
import boto3
s3 = boto3.client("s3",
    endpoint_url=os.environ["MINIO_ENDPOINT"],
    aws_access_key_id=os.environ["MINIO_ROOT_USER"],
    aws_secret_access_key=os.environ["MINIO_ROOT_PASSWORD"])

# Ollama
import httpx
resp = httpx.get(f"{os.environ['OLLAMA_HOST']}/api/tags")

# Vault
import hvac
vault = hvac.Client(url=os.environ["VAULT_ADDR"])
```

## Network Access

NetworkPolicies on Postgres, Redis, and MinIO explicitly allow traffic from `dev-env`.
Qdrant, RabbitMQ, Ollama, and observability services have no NetworkPolicy restrictions.
