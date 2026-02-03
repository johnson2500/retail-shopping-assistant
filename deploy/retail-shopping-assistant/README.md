# Retail Shopping Assistant Helm Chart

This Helm chart deploys the NVIDIA Retail Shopping Assistant on OpenShift.

## Prerequisites

- OpenShift 4.12 or later
- Helm 3.x
- `oc` CLI configured with cluster access
- NVIDIA NGC API key

## Quick Start

### 1. Create a namespace

```bash
oc new-project retail-shopping
```

### 2. Create the NVIDIA API key secret

```bash
oc create secret generic nvidia-api-keys \
  --from-literal=ngc-api-key='<YOUR_NGC_API_KEY>' \
  --from-literal=llm-api-key='<YOUR_NGC_API_KEY>' \
  --from-literal=embed-api-key='<YOUR_NGC_API_KEY>' \
  --from-literal=nvidia-api-key='<YOUR_NGC_API_KEY>'
```

### 3. Build and push application images

```bash
# Create BuildConfigs and build images
oc new-build --name=chain-server --binary --strategy=docker
oc start-build chain-server --from-dir=./chain_server --follow

oc new-build --name=catalog-retriever --binary --strategy=docker
oc start-build catalog-retriever --from-dir=./catalog_retriever --follow

oc new-build --name=memory-retriever --binary --strategy=docker
oc start-build memory-retriever --from-dir=./memory_retriever --follow

oc new-build --name=rails --binary --strategy=docker
oc start-build rails --from-dir=./guardrails --follow

oc new-build --name=frontend --binary --strategy=docker
oc start-build frontend --from-dir=./ui --follow
```

### 4. Install the chart

```bash
helm install retail-shopping ./deploy/retail-shopping-assistant \
  --namespace retail-shopping \
  --set secrets.create=false
```

Or with values file:

```bash
helm install retail-shopping ./deploy/retail-shopping-assistant \
  --namespace retail-shopping \
  -f my-values.yaml
```

## Configuration

### Key Values

| Parameter | Description | Default |
|-----------|-------------|---------|
| `secrets.ngcApiKey` | NVIDIA NGC API key | `""` |
| `nim.useCloudNims` | Use NVIDIA API Catalog | `true` |
| `nim.llmModel` | LLM model name | `meta/llama-3.1-70b-instruct` |
| `chainServer.replicaCount` | Chain Server replicas | `2` |
| `milvus.useOperator` | Use Milvus Operator | `false` |
| `postgresql.enabled` | Deploy PostgreSQL | `false` |
| `route.enabled` | Create OpenShift Route | `true` |

### Using Cloud NIMs (Default)

The chart defaults to using NVIDIA API Catalog for LLM and embedding services. This requires:

1. An active NGC API key with API Catalog access
2. Network access to `integrate.api.nvidia.com`

```yaml
nim:
  useCloudNims: true
  cloudEndpoint: "https://integrate.api.nvidia.com/v1"
```

### Using Local NIMs

For air-gapped or on-premise deployments:

1. Install NVIDIA GPU Operator
2. Deploy local NIM services
3. Update values:

```yaml
nim:
  useCloudNims: false
chainServer:
  env:
    CONFIG_OVERRIDE: ""  # Use default config
```

### Using PostgreSQL (Recommended for Production)

SQLite is used by default but is not recommended for production. Enable PostgreSQL:

```yaml
postgresql:
  enabled: true
secrets:
  postgres:
    enabled: true
    password: "your-secure-password"
memoryRetriever:
  database:
    usePostgres: true
    postgresUrl: "postgresql://memoryuser:your-secure-password@retail-shopping-assistant-postgresql:5432/memorydb"
```

### Using Milvus Operator

For production deployments, use the Milvus Operator instead of standalone Milvus:

```bash
# Install Milvus Operator first
helm repo add milvus-operator https://zilliztech.github.io/milvus-operator/
helm install milvus-operator milvus-operator/milvus-operator
```

```yaml
milvus:
  useOperator: true
```

### Using External Milvus (Zilliz Cloud)

```yaml
milvus:
  enabled: true
  external:
    enabled: true
    endpoint: "https://your-instance.zillizcloud.com:19530"
etcd:
  enabled: false
minio:
  enabled: false
```

## Upgrading

```bash
helm upgrade retail-shopping ./deploy/retail-shopping-assistant \
  --namespace retail-shopping \
  -f my-values.yaml
```

## Uninstalling

```bash
helm uninstall retail-shopping --namespace retail-shopping

# Clean up PVCs (optional)
oc delete pvc -l app.kubernetes.io/instance=retail-shopping
```

## Troubleshooting

### Pods not starting

Check pod status and logs:

```bash
oc get pods
oc describe pod <pod-name>
oc logs <pod-name>
```

### Image pull errors

Ensure ImagePullSecrets are configured:

```bash
oc create secret docker-registry nvidia-registry \
  --docker-server=nvcr.io \
  --docker-username='$oauthtoken' \
  --docker-password='<NGC_API_KEY>' \
  --docker-email=your-email@example.com

oc secrets link default nvidia-registry --for=pull
```

### Milvus issues

If using standalone Milvus and encountering issues, consider using the Milvus Operator or Zilliz Cloud instead.

### Network connectivity

Test internal connectivity:

```bash
oc exec -it <chain-server-pod> -- curl http://<catalog-retriever-service>:8010/health
```

## Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                    OpenShift Route (TLS)                            │
└─────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│                         Nginx Reverse Proxy                          │
└─────────────────────────────────────────────────────────────────────┘
                          │                    │
              ┌───────────┘                    └───────────┐
              ▼                                            ▼
┌─────────────────────────┐              ┌─────────────────────────────┐
│   Frontend (React UI)   │              │     Chain Server (FastAPI)  │
└─────────────────────────┘              └─────────────────────────────┘
                                                      │
                    ┌─────────────────────────────────┼─────────────────────────────────┐
                    │                                 │                                 │
                    ▼                                 ▼                                 ▼
     ┌──────────────────────────┐    ┌──────────────────────────┐    ┌──────────────────────────┐
     │   Catalog Retriever      │    │   Memory Retriever       │    │   Guardrails (Rails)     │
     └──────────────────────────┘    └──────────────────────────┘    └──────────────────────────┘
                    │                                                          │
                    ▼                                                          ▼
     ┌──────────────────────────┐                            ┌──────────────────────────────────┐
     │   Milvus Vector DB       │                            │    NVIDIA NIM Services           │
     └──────────────────────────┘                            │    (Cloud or Local)              │
                    │                                        └──────────────────────────────────┘
          ┌────────┴────────┐
          ▼                 ▼
┌─────────────────┐  ┌─────────────────┐
│   etcd          │  │   MinIO         │
└─────────────────┘  └─────────────────┘
```

## License

Copyright (c) 2024-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
Apache-2.0
