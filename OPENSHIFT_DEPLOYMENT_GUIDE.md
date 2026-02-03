# OpenShift AI Deployment Guide - Retail Shopping Assistant

## Table of Contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Architecture Summary](#architecture-summary)
- [Service Components](#service-components)
- [OpenShift Compatibility Assessment](#openshift-compatibility-assessment)
- [Required Changes for OpenShift](#required-changes-for-openshift)
- [Step-by-Step Deployment](#step-by-step-deployment)
- [Hardware Requirements](#hardware-requirements)
- [External Dependencies](#external-dependencies)
- [Deployment Strategy Recommendations](#deployment-strategy-recommendations)
- [Security Considerations](#security-considerations)
- [Configuration Requirements](#configuration-requirements)
- [Known Limitations](#known-limitations)
- [Migration Checklist](#migration-checklist)
- [Troubleshooting](#troubleshooting)

---

## Overview

The Retail Shopping Assistant is an AI-powered multi-agent application built with LangGraph for agent orchestration. It provides intelligent product search, shopping cart management, visual search capabilities, and conversational AI interactions.

This document outlines the requirements and considerations for deploying this application on OpenShift (Red Hat OpenShift Container Platform or OpenShift AI platform).

### What This Application Does

- **Product Search**: Text and image-based product discovery using vector embeddings
- **Shopping Cart**: Persistent cart management with add/remove/view operations
- **AI Conversation**: Natural language interactions powered by NVIDIA LLMs
- **Content Safety**: Input/output filtering via NeMo Guardrails
- **Visual Search**: Upload an image to find similar products

---

## Prerequisites

Before deploying to OpenShift, ensure you have:

### Required Accounts & Access

- [ ] **NVIDIA NGC Account** with API key access (get one at [ngc.nvidia.com](https://ngc.nvidia.com))
- [ ] **OpenShift Cluster** with admin or project-level access
- [ ] **OpenShift CLI (`oc`)** installed and configured

### Cluster Requirements

- OpenShift 4.12 or later
- Storage provisioner configured (for PersistentVolumeClaims)
- Network policies allowing internal pod communication
- (Optional) NVIDIA GPU Operator if deploying local NIMs

---

## Architecture Summary

The application follows a microservices architecture with the following components:

```
┌─────────────────────────────────────────────────────────────────────┐
│                         External Access                              │
│                    (OpenShift Route/Ingress)                        │
└─────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│                         Nginx Reverse Proxy                          │
│                    (nginx:1.25.3-alpine - Port 80)                  │
└─────────────────────────────────────────────────────────────────────┘
                          │                    │
              ┌───────────┘                    └───────────┐
              ▼                                            ▼
┌─────────────────────────┐              ┌─────────────────────────────┐
│   Frontend (React UI)   │              │     Chain Server (FastAPI)  │
│   Port 3000             │              │     Port 8009               │
│   Node.js 21            │              │     Python 3.11             │
└─────────────────────────┘              └─────────────────────────────┘
                                                      │
                    ┌─────────────────────────────────┼─────────────────────────────────┐
                    │                                 │                                 │
                    ▼                                 ▼                                 ▼
     ┌──────────────────────────┐    ┌──────────────────────────┐    ┌──────────────────────────┐
     │   Catalog Retriever      │    │   Memory Retriever       │    │   Guardrails (NeMo)      │
     │   Port 8010              │    │   Port 8011              │    │   Port 8012              │
     │   Python 3.11            │    │   Python 3.11            │    │   Python 3.11            │
     └──────────────────────────┘    └──────────────────────────┘    └──────────────────────────┘
                    │                         │                                 │
                    │                         │                                 │
                    ▼                         │                                 ▼
     ┌──────────────────────────┐             │              ┌──────────────────────────────────┐
     │   Milvus Vector DB       │◄────────────┘              │    NVIDIA NIM Services           │
     │   Port 19530             │                            │    (Local or Cloud API)          │
     └──────────────────────────┘                            │    - LLM (Llama 3.1 70B)         │
                    │                                        │    - Embeddings (NV-EmbedQA)     │
          ┌────────┴────────┐                                │    - Image (NV-CLIP)             │
          ▼                 ▼                                │    - Content Safety              │
┌─────────────────┐  ┌─────────────────┐                     │    - Topic Control               │
│   etcd          │  │   MinIO         │                     └──────────────────────────────────┘
│   Port 2379     │  │   Port 9000     │
└─────────────────┘  └─────────────────┘
```

**Data Flow Notes:**
- **Catalog Retriever** connects to Milvus for vector similarity search
- **Memory Retriever** uses local SQLite (`context.db`) for user context and cart storage (does NOT use Milvus)
- **Chain Server** orchestrates all backend services and connects to NVIDIA NIM services for LLM inference
- **Guardrails** connects to NVIDIA NIM services for content safety checks

---

## Service Components

### 1. Chain Server (Main API)

| Property | Value |
|----------|-------|
| **Image Base** | `python:3.11-slim` |
| **Port** | 8009 |
| **Framework** | FastAPI with Uvicorn |
| **Key Dependencies** | LangGraph 1.0.5, LangChain 1.2.1, OpenAI SDK 1.97.0, Pydantic 2.11.7 |
| **Environment Variables** | `LLM_API_KEY`, `CONFIG_OVERRIDE`, `CATALOG_RETRIEVER_URL`, `MEMORY_RETRIEVER_URL`, `RAILS_URL` |
| **Volume Mounts** | `/app/shared` (configs and data) |
| **Config Path** | `/app/shared/configs/chain_server/config.yaml` |

**Key Features:**
- Multi-agent orchestration with LangGraph (Planner, Cart, Retriever, Chatter, Summary agents)
- Streaming responses via Server-Sent Events (SSE) at `/query/stream`
- Timing endpoint for performance analysis at `/query/timing`
- Health check endpoint at `/health`

**Default Service URLs (set in Dockerfile):**
```
CATALOG_RETRIEVER_URL=http://catalog-retriever:8010
MEMORY_RETRIEVER_URL=http://memory-retriever:8011
RAILS_URL=http://rails:8012
```

### 2. Catalog Retriever

| Property | Value |
|----------|-------|
| **Image Base** | `python:3.11-slim` |
| **Port** | 8010 |
| **Framework** | FastAPI with Uvicorn |
| **Key Dependencies** | LangChain-Milvus 0.3.3, PyMilvus 2.6.0, Pandas 2.3.1, Pillow 11.3.0, OpenAI SDK 1.97.0 |
| **Environment Variables** | `EMBED_API_KEY`, `MILVUS_HOST`, `MILVUS_PORT`, `CONFIG_OVERRIDE` |
| **Volume Mounts** | `/app/shared` (configs, data, images) |
| **Config Path** | `/app/shared/configs/catalog_retriever/config.yaml` |

**Key Features:**
- Text-based product search via `/query/text`
- Image + text search via `/query/image`
- Integration with Milvus vector database
- Loads product catalog from `/app/shared/data/products_extended.csv` on startup
- Uses NVIDIA NV-EmbedQA-E5-v5 for text embeddings
- Uses NVIDIA NV-CLIP for image embeddings

### 3. Memory Retriever

| Property | Value |
|----------|-------|
| **Image Base** | `python:3.11-slim` |
| **Port** | 8011 |
| **Framework** | FastAPI with Uvicorn |
| **Key Dependencies** | SQLAlchemy 2.0.41, Pydantic 2.11.7, FastAPI 0.116.1 |
| **Environment Variables** | None required (Milvus env vars in docker-compose are unused) |
| **Volume Mounts** | `/app/shared` (configs) |
| **Database** | SQLite (local file `./context.db` in working directory) |

**Key Features:**
- User session context management via `/user/{user_id}/context/*` endpoints
- Shopping cart persistence via `/user/{user_id}/cart/*` endpoints
- Health check at `/health`

**API Endpoints:**
- `GET /user/{user_id}/context` - Get conversation context
- `POST /user/{user_id}/context/add` - Append to context
- `POST /user/{user_id}/context/replace` - Replace context
- `GET /user/{user_id}/cart` - Get cart contents
- `POST /user/{user_id}/cart/add` - Add item to cart
- `POST /user/{user_id}/cart/remove` - Remove item from cart
- `POST /user/{user_id}/cart/clear` - Clear entire cart

⚠️ **CRITICAL OpenShift Note:** SQLite writes to local filesystem (`./context.db`). This has several implications:
1. Data is lost if the pod is rescheduled to a different node
2. Multiple replicas cannot share the same database
3. **Recommendation:** Replace with PostgreSQL for production (see [Required Changes](#6-memory-retriever---replace-sqlite-with-postgresql))

### 4. Guardrails Service

| Property | Value |
|----------|-------|
| **Image Base** | `python:3.11-slim` |
| **Port** | 8012 |
| **Framework** | FastAPI with NeMo Guardrails |
| **Key Dependencies** | nemoguardrails==0.19.0, langchain-nvidia-ai-endpoints==1.0.1, OpenAI SDK 1.97.0 |
| **Environment Variables** | `NVIDIA_API_KEY`, `CONFIG_OVERRIDE` |
| **Volume Mounts** | `/app/shared` (for configs/rails) |
| **Config Path** | `/app/shared/configs/rails/config-build.yaml` (when using cloud NIMs) |

**Key Features:**
- Input content safety checks via `/rail/input/check`
- Output content safety checks via `/rail/output/check`
- Timing endpoints via `/rail/input/timing` and `/rail/output/timing`
- Uses NVIDIA NeMo Guardrails framework

**Build Notes:**
The Dockerfile installs system dependencies (`gcc`, `g++`, `python3-dev`) required for NeMo Guardrails compilation.

### 5. Frontend (React UI)

| Property | Value |
|----------|-------|
| **Image Base** | `nvcr.io/nvidia/base/ubuntu:22.04_20240212` (current) |
| **Node.js Version** | 21 (installed via nodesource) |
| **Port** | 3000 |
| **Framework** | React 18 with TypeScript |
| **Key Dependencies** | Material-UI, TailwindCSS, DOMPurify |
| **Volume Mounts** | `/app/shared` (symlinked to `/app/public/images`) |

**Key Features:**
- Responsive chat interface with streaming support
- Image upload for visual search (drag & drop or file picker)
- Guardrails toggle in UI
- Message download functionality
- Chat reset functionality

**Build Notes:**
- The Dockerfile creates a symlink: `/app/shared/images` → `/app/public/images`
- Product images must be available in the shared volume

⚠️ **OpenShift Note:** The current base image from `nvcr.io` requires registry authentication. For simplified deployment, consider replacing with `node:21-slim` (see [Required Changes](#2-frontend-dockerfile-replace-nvrcio-base-image)).

### 6. Nginx Reverse Proxy

| Property | Value |
|----------|-------|
| **Image** | `nginx:1.25.3-alpine` |
| **Port** | 80 (exposed externally as 3000) |
| **Configuration** | Custom `nginx.conf` mounted at `/etc/nginx/nginx.conf` |

**Routing Configuration:**
```
Location /      → http://frontend:3000  (React UI)
Location /api/  → http://chain-server:8009 (Backend API)
```

**Key Features:**
- Reverse proxy for all external traffic
- WebSocket upgrade support for streaming responses
- Proxy buffering disabled for real-time SSE streaming
- HTTP/1.1 with Connection upgrade headers

**OpenShift Note:** In OpenShift, you may replace Nginx with an OpenShift Route or Ingress pointing directly to the frontend service, with path-based routing configured for `/api/` to chain-server.

### 7. Milvus Vector Database

| Property | Value |
|----------|-------|
| **Image** | `milvusdb/milvus:v2.4.13-hotfix` |
| **Ports** | 19530 (gRPC), 9091 (metrics/health) |
| **Dependencies** | etcd (metadata), MinIO (object storage) |
| **Storage** | `/var/lib/milvus` |
| **Health Check** | `curl http://localhost:9091/healthz` |

**Environment Variables:**
```
ETCD_ENDPOINTS=http://etcd:2379
MINIO_ADDRESS=minio:9000
MINIO_ACCESS_KEY=minioadmin
MINIO_SECRET_KEY=minioadmin
COMMON_STORAGETYPE=local
```

**Collections Created:**
- `shopping_advisor_text_db` - Text embeddings (1024 dimensions)
- `shopping_advisor_image_db` - Image embeddings (1024 dimensions)

⚠️ **CRITICAL OpenShift Issue:** Requires `security_opt: seccomp:unconfined` which is **NOT ALLOWED** in default OpenShift. See [Milvus Deployment Options](#3-milvus-deployment-options) for alternatives.

### 8. etcd (Milvus Metadata)

| Property | Value |
|----------|-------|
| **Image** | `quay.io/coreos/etcd:v3.5.5` |
| **Port** | 2379 |
| **Storage** | `/etcd` |

### 9. MinIO (Milvus Storage)

| Property | Value |
|----------|-------|
| **Image** | `minio/minio:RELEASE.2023-03-20T20-16-18Z` |
| **Ports** | 9000 (API), 9001 (Console) |
| **Storage** | `/minio_data` |
| **Credentials** | `minioadmin/minioadmin` (default) |

⚠️ **OpenShift Note:** Default credentials must be changed for production.

---

## OpenShift Compatibility Assessment

### ✅ Compatible Components

| Component | Status | Notes |
|-----------|--------|-------|
| Chain Server | ✅ Compatible | Standard Python container |
| Catalog Retriever | ✅ Compatible | Standard Python container |
| Memory Retriever | ⚠️ Needs Changes | SQLite → PostgreSQL recommended |
| Guardrails | ✅ Compatible | Standard Python container |
| Frontend | ⚠️ Needs Changes | nvcr.io registry auth required |
| Nginx | ✅ Compatible | Standard nginx image |
| etcd | ✅ Compatible | Available on quay.io |
| MinIO | ✅ Compatible | Consider OpenShift ODF instead |

### ❌ Components Requiring Significant Changes

| Component | Issue | Severity | Resolution |
|-----------|-------|----------|------------|
| **Milvus** | `seccomp:unconfined` | 🔴 Critical | Use Milvus Operator or external managed service |
| **NVIDIA NIMs (Local)** | GPU scheduling, shm_size, privileged mode | 🔴 Critical | Use cloud NIMs or NVIDIA GPU Operator |
| **All containers** | Root user execution | 🟡 Medium | Modify Dockerfiles for non-root users |

### OpenShift Security Context Constraints (SCC) Analysis

The following features from the current Docker Compose setup are **NOT ALLOWED** in default OpenShift:

```yaml
# ❌ NOT ALLOWED - Milvus container
security_opt:
  - seccomp:unconfined

# ❌ NOT ALLOWED - NIM containers
shm_size: "16gb"
user: "${UID:-1000}"  # May conflict with random UID assignment

# ❌ NOT ALLOWED - GPU device mapping (use NVIDIA GPU Operator instead)
deploy:
  resources:
    reservations:
      devices:
        - driver: nvidia
          device_ids: ['0', '1']
          capabilities: [gpu]
```

---

## Required Changes for OpenShift

### 1. Dockerfile Modifications (Non-Root Users)

All Dockerfiles need to be updated to run as non-root users:

```dockerfile
# Example modification for chain_server/Dockerfile
FROM python:3.11-slim

# Create non-root user
RUN useradd -m -u 1001 appuser

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY ./src ./app

# Change ownership and switch user
RUN chown -R appuser:appuser /app
USER appuser

ENV PYTHONPATH=/app

EXPOSE 8009

CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8009"]
```

### 2. Frontend Dockerfile (Replace nvcr.io Base Image)

```dockerfile
# Replace nvcr.io image with standard Node.js image
FROM node:21-slim

WORKDIR /app

COPY package*.json ./
RUN npm install

COPY . .

# Create symlink for shared images
RUN mkdir -p /app/public/images

# Run as non-root
USER node

EXPOSE 3000

CMD ["npm", "run", "start"]
```

### 3. Milvus Deployment Options

**Option A: Milvus Operator (Recommended)**
```yaml
apiVersion: milvus.io/v1beta1
kind: Milvus
metadata:
  name: retail-milvus
  namespace: retail-shopping
spec:
  mode: standalone
  dependencies:
    storage:
      external: true
      type: S3
      endpoint: minio:9000
      secretRef: minio-credentials
    etcd:
      external: true
      endpoints:
        - etcd:2379
```

**Option B: Use Managed Milvus (Zilliz Cloud)**

Update config to point to external Milvus instance:
```yaml
db_port: "https://your-zilliz-instance.zillizcloud.com:19530"
```

### 4. NVIDIA NIM Deployment

**Option A: Cloud NIMs (Recommended for OpenShift)**

Set `CONFIG_OVERRIDE=config-build.yaml` to use NVIDIA API Catalog:

```yaml
# chain_server config-build.yaml
llm_port: "https://integrate.api.nvidia.com/v1"
llm_name: "meta/llama-3.1-70b-instruct"

# catalog_retriever config-build.yaml
text_embed_port: "https://integrate.api.nvidia.com/v1"
image_embed_port: "https://integrate.api.nvidia.com/v1"

# guardrails config-build.yaml
models:
  - type: content_safety
    parameters:
      base_url: https://integrate.api.nvidia.com/v1
  - type: topic_control
    parameters:
      base_url: https://integrate.api.nvidia.com/v1
```

**Option B: Local NIMs with NVIDIA GPU Operator**

Prerequisites:
1. Install NVIDIA GPU Operator on OpenShift
2. Configure Node Feature Discovery (NFD)
3. Create GPU-enabled machine sets

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: llama-nim
spec:
  containers:
  - name: llama
    image: nvcr.io/nim/meta/llama-3.1-70b-instruct:1.10.1
    resources:
      limits:
        nvidia.com/gpu: 2
    env:
    - name: NGC_API_KEY
      valueFrom:
        secretKeyRef:
          name: nvidia-api-keys
          key: ngc-api-key
    volumeMounts:
    - name: nim-cache
      mountPath: /opt/nim/.cache
    - name: dshm
      mountPath: /dev/shm
  volumes:
  - name: nim-cache
    persistentVolumeClaim:
      claimName: nim-cache-pvc
  - name: dshm
    emptyDir:
      medium: Memory
      sizeLimit: 16Gi
```

### 5. Persistent Volume Claims

Replace local volume mounts with PVCs:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: shared-data-pvc
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 10Gi
  storageClassName: ocs-storagecluster-cephfs  # OpenShift Data Foundation

---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: milvus-data-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 50Gi
  storageClassName: ocs-storagecluster-ceph-rbd
```

### 6. Memory Retriever - Replace SQLite with PostgreSQL

```python
# Update memory_retriever/src/main.py
import os

DATABASE_URL = os.environ.get(
    "DATABASE_URL",
    "postgresql://user:password@postgres:5432/memorydb"
)
engine = create_engine(DATABASE_URL)
```

Deploy PostgreSQL:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
spec:
  template:
    spec:
      containers:
      - name: postgres
        image: postgres:15
        env:
        - name: POSTGRES_DB
          value: memorydb
        - name: POSTGRES_USER
          valueFrom:
            secretKeyRef:
              name: postgres-credentials
              key: username
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-credentials
              key: password
```

---

## Step-by-Step Deployment

This section provides a complete walkthrough for deploying the Retail Shopping Assistant to OpenShift using **Cloud NIMs** (recommended for most deployments).

### Step 1: Create the Project Namespace

```bash
# Create a new project (namespace)
oc new-project retail-shopping

# Verify you're in the correct project
oc project retail-shopping
```

### Step 2: Create Secrets

```bash
# Create NVIDIA API key secret
oc create secret generic nvidia-api-keys \
  --from-literal=ngc-api-key='<YOUR_NGC_API_KEY>' \
  --from-literal=llm-api-key='<YOUR_NGC_API_KEY>' \
  --from-literal=embed-api-key='<YOUR_NGC_API_KEY>'

# Create MinIO credentials secret
oc create secret generic minio-credentials \
  --from-literal=access-key='minioadmin' \
  --from-literal=secret-key='<STRONG_PASSWORD_HERE>'

# (Optional) Create PostgreSQL credentials if using PostgreSQL
oc create secret generic postgres-credentials \
  --from-literal=username='memoryuser' \
  --from-literal=password='<STRONG_PASSWORD_HERE>'
```

### Step 3: Create ImagePullSecret for nvcr.io (if using NVIDIA base images)

```bash
oc create secret docker-registry nvidia-registry \
  --docker-server=nvcr.io \
  --docker-username='$oauthtoken' \
  --docker-password='<YOUR_NGC_API_KEY>' \
  --docker-email=your-email@example.com

# Link the secret to the default service account
oc secrets link default nvidia-registry --for=pull
oc secrets link builder nvidia-registry
```

### Step 4: Create PersistentVolumeClaims

**Note:** The `shared-data-pvc` must use `ReadWriteMany` (RWX) access mode since multiple pods need to read the shared configs, data, and images. This typically requires:
- OpenShift Data Foundation (ODF/OCS) with CephFS
- NFS-based storage
- Azure Files, AWS EFS, or similar cloud-native RWX storage

```yaml
# Save as pvcs.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: shared-data-pvc
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 10Gi
  # Uncomment and set your storage class for RWX support
  # storageClassName: ocs-storagecluster-cephfs
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: milvus-data-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: etcd-data-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: minio-data-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

```bash
oc apply -f pvcs.yaml
```

### Step 4b: Populate Shared Data Volume

The shared volume needs the product catalog, images, and configuration files. Create a temporary pod to copy data:

```yaml
# Save as data-loader.yaml
apiVersion: v1
kind: Pod
metadata:
  name: data-loader
spec:
  containers:
  - name: loader
    image: busybox
    command: ["sleep", "3600"]
    volumeMounts:
    - name: shared-data
      mountPath: /data
  volumes:
  - name: shared-data
    persistentVolumeClaim:
      claimName: shared-data-pvc
  restartPolicy: Never
```

```bash
# Create the data loader pod
oc apply -f data-loader.yaml
oc wait --for=condition=Ready pod/data-loader

# Copy the shared directory contents to the PVC
oc cp shared/configs data-loader:/data/configs
oc cp shared/data data-loader:/data/data
oc cp shared/images data-loader:/data/images

# Verify the data was copied
oc exec data-loader -- ls -la /data/
oc exec data-loader -- ls -la /data/data/
oc exec data-loader -- ls -la /data/images/ | head -20

# Delete the data loader pod
oc delete pod data-loader
```

**Contents of the shared volume:**
```
/data/
├── configs/
│   ├── chain_server/
│   │   ├── config.yaml
│   │   └── config-build.yaml
│   ├── catalog_retriever/
│   │   ├── config.yaml
│   │   └── config-build.yaml
│   └── rails/
│       └── config-build.yaml
├── data/
│   └── products_extended.csv
└── images/
    └── [product images - ~200 JPG files]
```

### Step 5: Create ConfigMaps

The configuration files contain detailed prompts and settings. The easiest approach is to create ConfigMaps directly from the existing config files in the repository:

```bash
# Create ConfigMaps from existing config files
oc create configmap chain-server-config \
  --from-file=config.yaml=shared/configs/chain_server/config.yaml \
  --from-file=config-build.yaml=shared/configs/chain_server/config-build.yaml

oc create configmap catalog-retriever-config \
  --from-file=config.yaml=shared/configs/catalog_retriever/config.yaml \
  --from-file=config-build.yaml=shared/configs/catalog_retriever/config-build.yaml

oc create configmap rails-config \
  --from-file=config-build.yaml=shared/configs/rails/config-build.yaml

oc create configmap nginx-config \
  --from-file=nginx.conf=nginx.conf
```

Alternatively, you can create ConfigMaps with YAML for more control:

```yaml
# Save as configmaps.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: catalog-retriever-config
data:
  config-build.yaml: |
    text_embed_port: "https://integrate.api.nvidia.com/v1"
    text_model_name: "nvidia/nv-embedqa-e5-v5"
    image_embed_port: "https://integrate.api.nvidia.com/v1"
    image_model_name: "nvidia/nvclip"
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: rails-config
data:
  config-build.yaml: |
    models:
      - type: content_safety
        parameters:
          base_url: https://integrate.api.nvidia.com/v1
      - type: topic_control
        parameters:
          base_url: https://integrate.api.nvidia.com/v1
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
data:
  nginx.conf: |
    events {
        worker_connections 1024;
    }
    http {
        upstream frontend {
            server frontend:3000;
        }
        upstream backend {
            server chain-server:8009;
        }
        server {
            listen 80;
            location / {
                proxy_pass http://frontend;
                proxy_set_header Host $host;
                proxy_set_header X-Real-IP $remote_addr;
                proxy_http_version 1.1;
                proxy_set_header Upgrade $http_upgrade;
                proxy_set_header Connection "upgrade";
            }
            location /api/ {
                proxy_pass http://backend/;
                proxy_set_header Host $host;
                proxy_buffering off;
                proxy_cache off;
                proxy_http_version 1.1;
                proxy_set_header Upgrade $http_upgrade;
                proxy_set_header Connection "upgrade";
            }
        }
    }
```

```bash
oc apply -f configmaps.yaml
```

**Important:** The chain-server requires the full `config.yaml` with routing and chatter prompts. Copy the contents of `shared/configs/chain_server/config.yaml` from the repository.

### Step 6: Deploy Infrastructure Services

Deploy etcd, MinIO, and Milvus (or use Milvus Operator):

```yaml
# Save as infrastructure.yaml
# etcd Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: etcd
spec:
  replicas: 1
  selector:
    matchLabels:
      app: etcd
  template:
    metadata:
      labels:
        app: etcd
    spec:
      containers:
      - name: etcd
        image: quay.io/coreos/etcd:v3.5.5
        command:
          - etcd
          - --advertise-client-urls=http://etcd:2379
          - --listen-client-urls=http://0.0.0.0:2379
          - --data-dir=/etcd
        env:
        - name: ETCD_AUTO_COMPACTION_MODE
          value: "revision"
        - name: ETCD_AUTO_COMPACTION_RETENTION
          value: "1000"
        - name: ETCD_QUOTA_BACKEND_BYTES
          value: "4294967296"
        ports:
        - containerPort: 2379
        volumeMounts:
        - name: etcd-data
          mountPath: /etcd
      volumes:
      - name: etcd-data
        persistentVolumeClaim:
          claimName: etcd-data-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: etcd
spec:
  selector:
    app: etcd
  ports:
  - port: 2379
    targetPort: 2379
---
# MinIO Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: minio
spec:
  replicas: 1
  selector:
    matchLabels:
      app: minio
  template:
    metadata:
      labels:
        app: minio
    spec:
      containers:
      - name: minio
        image: minio/minio:RELEASE.2023-03-20T20-16-18Z
        command:
          - minio
          - server
          - /minio_data
          - --console-address
          - ":9001"
        env:
        - name: MINIO_ROOT_USER
          valueFrom:
            secretKeyRef:
              name: minio-credentials
              key: access-key
        - name: MINIO_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: minio-credentials
              key: secret-key
        ports:
        - containerPort: 9000
        - containerPort: 9001
        volumeMounts:
        - name: minio-data
          mountPath: /minio_data
      volumes:
      - name: minio-data
        persistentVolumeClaim:
          claimName: minio-data-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: minio
spec:
  selector:
    app: minio
  ports:
  - name: api
    port: 9000
    targetPort: 9000
  - name: console
    port: 9001
    targetPort: 9001
```

```bash
oc apply -f infrastructure.yaml
```

### Step 7: Deploy Milvus (Using Milvus Operator - Recommended)

```bash
# Install Milvus Operator
helm repo add milvus-operator https://zilliztech.github.io/milvus-operator/
helm install milvus-operator milvus-operator/milvus-operator
```

```yaml
# Save as milvus-cr.yaml
apiVersion: milvus.io/v1beta1
kind: Milvus
metadata:
  name: milvus
spec:
  mode: standalone
  dependencies:
    etcd:
      external: true
      endpoints:
        - etcd:2379
    storage:
      external: true
      type: S3
      endpoint: minio:9000
      secretRef: minio-credentials
```

```bash
oc apply -f milvus-cr.yaml
```

### Step 8: Build and Deploy Application Services

```bash
# Create BuildConfigs for each service
# Chain Server
oc new-build --name=chain-server --binary --strategy=docker
oc start-build chain-server --from-dir=./chain_server --follow

# Catalog Retriever  
oc new-build --name=catalog-retriever --binary --strategy=docker
oc start-build catalog-retriever --from-dir=./catalog_retriever --follow

# Memory Retriever
oc new-build --name=memory-retriever --binary --strategy=docker
oc start-build memory-retriever --from-dir=./memory_retriever --follow

# Guardrails
oc new-build --name=rails --binary --strategy=docker
oc start-build rails --from-dir=./guardrails --follow

# Frontend (after modifying Dockerfile for non-root)
oc new-build --name=frontend --binary --strategy=docker
oc start-build frontend --from-dir=./ui --follow
```

### Step 9: Deploy Application Services

See the [Sample OpenShift Deployment Manifest](#sample-openshift-deployment-manifest) section for complete deployment YAML files.

### Step 10: Create Services and Routes

```yaml
# Save as services.yaml
apiVersion: v1
kind: Service
metadata:
  name: chain-server
spec:
  selector:
    app: chain-server
  ports:
  - port: 8009
    targetPort: 8009
---
apiVersion: v1
kind: Service
metadata:
  name: catalog-retriever
spec:
  selector:
    app: catalog-retriever
  ports:
  - port: 8010
    targetPort: 8010
---
apiVersion: v1
kind: Service
metadata:
  name: memory-retriever
spec:
  selector:
    app: memory-retriever
  ports:
  - port: 8011
    targetPort: 8011
---
apiVersion: v1
kind: Service
metadata:
  name: rails
spec:
  selector:
    app: rails
  ports:
  - port: 8012
    targetPort: 8012
---
apiVersion: v1
kind: Service
metadata:
  name: frontend
spec:
  selector:
    app: frontend
  ports:
  - port: 3000
    targetPort: 3000
---
apiVersion: v1
kind: Service
metadata:
  name: nginx
spec:
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
```

```bash
oc apply -f services.yaml

# Create external route
oc expose service nginx --name=retail-shopping-assistant
oc get route retail-shopping-assistant
```

### Step 11: Verify Deployment

```bash
# Check all pods are running
oc get pods

# Check service health endpoints
oc exec -it $(oc get pod -l app=chain-server -o name | head -1) -- curl localhost:8009/health
oc exec -it $(oc get pod -l app=catalog-retriever -o name | head -1) -- curl localhost:8010/health
oc exec -it $(oc get pod -l app=memory-retriever -o name | head -1) -- curl localhost:8011/health

# Get the application URL
echo "Application URL: https://$(oc get route retail-shopping-assistant -o jsonpath='{.spec.host}')"
```

---

## Hardware Requirements

### Minimum Requirements (Cloud NIMs)

| Resource | Requirement |
|----------|-------------|
| CPU | 8 cores |
| RAM | 32 GB |
| Storage | 50 GB |
| GPU | Not required |
| Network | Stable internet for API calls |

### Recommended Requirements (Local NIMs)

| Resource | Requirement |
|----------|-------------|
| CPU | 16+ cores |
| RAM | 128 GB |
| Storage | 100 GB+ SSD |
| GPU | 4x NVIDIA H100 (80GB) or 4x A100 (80GB) |
| Network | High-speed internal network |

### GPU Distribution (Local NIM Deployment)

Based on the `docker-compose-nim-local.yaml` configuration:

| NIM Service | Image | GPU Assignment | Shared Memory |
|-------------|-------|----------------|---------------|
| Llama 3.1 70B | `nvcr.io/nim/meta/llama-3.1-70b-instruct:1.10.1` | GPU 0, GPU 1 | 16 GB |
| NV-EmbedQA-E5-v5 | `nvcr.io/nim/nvidia/nv-embedqa-e5-v5:1.6` | GPU 2 | Default |
| NV-CLIP | `nvcr.io/nim/nvidia/nvclip:2.0.0` | GPU 2 (shared) | Default |
| Content Safety | `nvcr.io/nim/nvidia/llama-3.1-nemoguard-8b-content-safety:1.10.1` | GPU 2 (shared) | Default |
| Topic Control | `nvcr.io/nim/nvidia/llama-3.1-nemoguard-8b-topic-control:1.0.0` | GPU 3 | Default |

**Note:** Content Safety and Topic Control NIMs use `NIM_KVCACHE_PERCENT=.4` to limit memory usage when sharing GPUs.

---

## External Dependencies

### Required API Keys & Credentials

| Secret Name | Environment Variable | Purpose | Used By |
|-------------|---------------------|---------|---------|
| `NGC_API_KEY` | `NGC_API_KEY` | NVIDIA NGC authentication | All local NIM services |
| `LLM_API_KEY` | `LLM_API_KEY` | LLM API authentication | Chain Server |
| `EMBED_API_KEY` | `EMBED_API_KEY` | Embedding API authentication | Catalog Retriever |
| `NVIDIA_API_KEY` | `NVIDIA_API_KEY` | Guardrails API authentication | Guardrails Service |

**Note:** When using Cloud NIMs (NVIDIA API Catalog), all API keys can be the same NGC API key. The environment variables have different names for flexibility in using different keys for different services.

### Container Registries

| Registry | Images | Authentication |
|----------|--------|----------------|
| `nvcr.io` | NVIDIA NIMs, NVIDIA base images | NGC API Key |
| `docker.io` | Python, Nginx base images | Optional |
| `quay.io` | etcd | None required |
| `milvusdb` | Milvus | None required |
| `minio` | MinIO | None required |

### Create ImagePullSecrets for nvcr.io

```bash
oc create secret docker-registry nvidia-registry \
  --docker-server=nvcr.io \
  --docker-username='$oauthtoken' \
  --docker-password='<NGC_API_KEY>' \
  --docker-email=your-email@example.com

oc secrets link default nvidia-registry --for=pull
oc secrets link builder nvidia-registry
```

---

## Deployment Strategy Recommendations

### Strategy 1: Cloud-First (Recommended)

**Best for:** Quick deployment, reduced operational complexity

1. Use NVIDIA API Catalog for all NIM services
2. Deploy application services on OpenShift
3. Use managed PostgreSQL (e.g., Amazon RDS, Azure Database)
4. Use managed Milvus (Zilliz Cloud) or Milvus Operator

**Pros:**
- No GPU infrastructure required
- Faster time to deployment
- Reduced operational overhead

**Cons:**
- Ongoing cloud API costs
- Network latency
- Data privacy considerations

### Strategy 2: Hybrid (Balanced)

**Best for:** Production workloads with cost optimization

1. Deploy critical services (Chain Server, Retrievers, Frontend) on OpenShift
2. Use NVIDIA GPU Operator for local NIM deployment
3. Deploy Milvus using the Milvus Operator
4. Use OpenShift Data Foundation for storage

### Strategy 3: Fully On-Premise

**Best for:** Air-gapped environments, strict data privacy requirements

1. Full local NIM deployment with NVIDIA GPU Operator
2. All services containerized on OpenShift
3. OpenShift Data Foundation for all storage
4. May require privileged SCC for some components

---

## Security Considerations

### Secrets Management

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: nvidia-api-keys
type: Opaque
stringData:
  ngc-api-key: "<your-ngc-api-key>"
  llm-api-key: "<your-llm-api-key>"
  embed-api-key: "<your-embed-api-key>"
  rail-api-key: "<your-rail-api-key>"
```

### Network Policies

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: retail-shopping-network-policy
spec:
  podSelector:
    matchLabels:
      app: retail-shopping
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: openshift-ingress
  egress:
  - to:
    - namespaceSelector: {}  # Allow internal communication
  - to:
    - ipBlock:
        cidr: 0.0.0.0/0  # Allow external API calls
    ports:
    - protocol: TCP
      port: 443
```

### Pod Security Standards

Apply restricted security context:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: chain-server
spec:
  securityContext:
    runAsNonRoot: true
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: chain-server
    securityContext:
      allowPrivilegeEscalation: false
      capabilities:
        drop:
        - ALL
      readOnlyRootFilesystem: true
```

---

## Configuration Requirements

### ConfigMaps

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: chain-server-config
data:
  config.yaml: |
    llm_port: "https://integrate.api.nvidia.com/v1"
    llm_name: "meta/llama-3.1-70b-instruct"
    retriever_port: "http://catalog-retriever:8010"
    memory_port: "http://memory-retriever:8011"
    rails_port: "http://rails:8012"
    memory_length: 16384
    top_k_retrieve: 4
    multimodal: true
    categories:
      - bag
      - sunglasses
      - dress
      - skirt
      - top blouse sweater
      - shoes
      - earrings
      - bracelet
      - necklace
```

### Environment Variables Summary

| Service | Variable | Required | Default | Notes |
|---------|----------|----------|---------|-------|
| chain-server | `LLM_API_KEY` | Yes | - | NGC API key for LLM access |
| chain-server | `CONFIG_OVERRIDE` | No | - | Set to `config-build.yaml` for cloud NIMs |
| chain-server | `CATALOG_RETRIEVER_URL` | No | `http://catalog-retriever:8010` | Set in Dockerfile |
| chain-server | `MEMORY_RETRIEVER_URL` | No | `http://memory-retriever:8011` | Set in Dockerfile |
| chain-server | `RAILS_URL` | No | `http://rails:8012` | Set in Dockerfile |
| catalog-retriever | `EMBED_API_KEY` | Yes | - | NGC API key for embedding access |
| catalog-retriever | `MILVUS_HOST` | Yes | - | Milvus service hostname |
| catalog-retriever | `MILVUS_PORT` | Yes | - | Typically `19530` |
| catalog-retriever | `CONFIG_OVERRIDE` | No | - | Set to `config-build.yaml` for cloud NIMs |
| memory-retriever | `DATABASE_URL` | No (Yes if PostgreSQL) | `sqlite:///./context.db` | Override for PostgreSQL |
| guardrails | `NVIDIA_API_KEY` | Yes | - | NGC API key for guardrails |
| guardrails | `CONFIG_OVERRIDE` | No | - | Set to `config-build.yaml` for cloud NIMs |

---

## Known Limitations

### Current Limitations in OpenShift

| Limitation | Impact | Workaround |
|------------|--------|------------|
| Milvus requires `seccomp:unconfined` | Cannot deploy standalone Milvus | Use Milvus Operator or Zilliz Cloud |
| NIM containers require elevated privileges | Complex GPU scheduling | Use NVIDIA GPU Operator or Cloud NIMs |
| SQLite in Memory Retriever | Not suitable for HA deployments | Migrate to PostgreSQL |
| Local volume mounts | Not portable across nodes | Use PVCs with RWX storage |
| nvcr.io base image in frontend | Registry authentication required | Rebuild with standard Node.js image |
| Hardcoded localhost references | May break in container networks | Use service names via environment variables |

### Feature Limitations

| Feature | Limitation | Notes |
|---------|------------|-------|
| Image Upload | 10MB max size | Configure in `ui/src/config/config.ts` |
| Supported Categories | Fixed list | Update in `config.yaml` if expanding |
| Concurrent Users | Limited by resource allocation | Scale pods horizontally |

---

## Migration Checklist

### Pre-Migration

- [ ] Obtain NVIDIA NGC API key with appropriate entitlements
- [ ] Verify OpenShift cluster has sufficient resources
- [ ] Install NVIDIA GPU Operator (if using local NIMs)
- [ ] Configure OpenShift Data Foundation or alternative storage
- [ ] Create namespace and set up RBAC

### Container Registry Setup

- [ ] Create ImagePullSecret for nvcr.io
- [ ] Push custom images to internal registry (optional)
- [ ] Verify all base images are accessible

### Application Migration

- [ ] Modify Dockerfiles for non-root execution
- [ ] Replace SQLite with PostgreSQL in Memory Retriever
- [ ] Create Kubernetes manifests (Deployments, Services, ConfigMaps)
- [ ] Configure PersistentVolumeClaims
- [ ] Create Secrets for API keys

### Database Setup

- [ ] Deploy PostgreSQL (or use managed service)
- [ ] Deploy Milvus using Milvus Operator (or use Zilliz Cloud)
- [ ] Configure etcd (if using Milvus Operator, this is handled automatically)
- [ ] Configure MinIO or use OpenShift Data Foundation S3

### Networking

- [ ] Create Services for each component
- [ ] Configure Routes/Ingress for external access
- [ ] Apply NetworkPolicies
- [ ] Verify inter-service communication

### Testing

- [ ] Verify health check endpoints
- [ ] Test product search functionality
- [ ] Test image upload and visual search
- [ ] Test shopping cart operations
- [ ] Verify guardrails are functioning
- [ ] Load test with expected user volume

### Production Readiness

- [ ] Configure horizontal pod autoscaling
- [ ] Set up monitoring and alerting
- [ ] Configure log aggregation
- [ ] Document backup and recovery procedures
- [ ] Establish incident response runbooks

---

## Sample OpenShift Deployment Manifest

```yaml
# chain-server-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: chain-server
  labels:
    app: retail-shopping
    component: chain-server
spec:
  replicas: 2
  selector:
    matchLabels:
      app: retail-shopping
      component: chain-server
  template:
    metadata:
      labels:
        app: retail-shopping
        component: chain-server
    spec:
      securityContext:
        runAsNonRoot: true
        seccompProfile:
          type: RuntimeDefault
      containers:
      - name: chain-server
        image: image-registry.openshift-image-registry.svc:5000/retail-shopping/chain-server:latest
        ports:
        - containerPort: 8009
        env:
        - name: LLM_API_KEY
          valueFrom:
            secretKeyRef:
              name: nvidia-api-keys
              key: llm-api-key
        - name: CONFIG_OVERRIDE
          value: "config-build.yaml"
        - name: CATALOG_RETRIEVER_URL
          value: "http://catalog-retriever:8010"
        - name: MEMORY_RETRIEVER_URL
          value: "http://memory-retriever:8011"
        volumeMounts:
        - name: shared-config
          mountPath: /app/shared/configs
        - name: shared-data
          mountPath: /app/shared/data
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
          limits:
            memory: "2Gi"
            cpu: "2000m"
        livenessProbe:
          httpGet:
            path: /health
            port: 8009
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /health
            port: 8009
          initialDelaySeconds: 5
          periodSeconds: 5
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            drop:
            - ALL
      volumes:
      - name: shared-config
        configMap:
          name: chain-server-config
      - name: shared-data
        persistentVolumeClaim:
          claimName: shared-data-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: chain-server
spec:
  selector:
    app: retail-shopping
    component: chain-server
  ports:
  - port: 8009
    targetPort: 8009
```

---

## Troubleshooting

### Common Issues and Solutions

#### 1. Pods Failing to Start with "CrashLoopBackOff"

**Check logs:**
```bash
oc logs <pod-name>
oc describe pod <pod-name>
```

**Common causes:**
- Missing environment variables (API keys)
- Config files not mounted correctly
- Services not yet available (dependency ordering)

#### 2. Milvus Fails to Start

**Issue:** `seccomp:unconfined` not allowed

**Solution:** Use the Milvus Operator instead of standalone deployment, or use an external managed Milvus (Zilliz Cloud).

#### 3. Frontend Cannot Connect to Backend

**Check nginx routing:**
```bash
oc exec -it <nginx-pod> -- cat /etc/nginx/nginx.conf
oc logs <nginx-pod>
```

**Verify services are reachable:**
```bash
oc exec -it <nginx-pod> -- curl http://chain-server:8009/health
oc exec -it <nginx-pod> -- curl http://frontend:3000
```

#### 4. Catalog Retriever Cannot Connect to Milvus

**Check Milvus health:**
```bash
oc exec -it <milvus-pod> -- curl localhost:9091/healthz
```

**Verify network connectivity:**
```bash
oc exec -it <catalog-retriever-pod> -- curl http://milvus:19530
```

#### 5. Images Not Displaying

**Issue:** Product images return 404

**Solution:** Ensure the shared volume is properly mounted and contains the images directory:
```bash
oc exec -it <frontend-pod> -- ls -la /app/public/images
oc exec -it <frontend-pod> -- ls -la /app/shared/images
```

#### 6. Streaming Responses Not Working

**Check nginx proxy buffering:**
Ensure nginx config includes:
```
proxy_buffering off;
proxy_cache off;
```

**Check OpenShift Route timeout:**
```bash
oc annotate route retail-shopping-assistant --overwrite haproxy.router.openshift.io/timeout=5m
```

#### 7. Permission Denied Errors

**Issue:** Container runs as non-root but files are owned by root

**Solution:** Ensure PVCs have correct permissions:
```bash
oc exec -it <pod-name> -- ls -la /app/shared
```

Consider using SecurityContextConstraints or init containers to fix permissions.

### Viewing Logs

```bash
# All pods
oc logs -l app=retail-shopping --all-containers

# Specific service
oc logs -f deployment/chain-server
oc logs -f deployment/catalog-retriever
oc logs -f deployment/memory-retriever
oc logs -f deployment/rails
oc logs -f deployment/frontend

# Previous container (after crash)
oc logs <pod-name> --previous
```

### Health Check Endpoints

| Service | Endpoint | Expected Response |
|---------|----------|-------------------|
| Chain Server | `/health` | `{"status": "healthy", ...}` |
| Catalog Retriever | `/health` | `{"status": "healthy", ...}` |
| Memory Retriever | `/health` | `{"status": "healthy", ...}` |
| Guardrails | N/A | Service responds to POST requests |
| Milvus | `:9091/healthz` | `OK` |
| MinIO | `:9000/minio/health/live` | HTTP 200 |
| etcd | `etcdctl endpoint health` | `is healthy` |

---

## References

- [NVIDIA GPU Operator Documentation](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/index.html)
- [OpenShift Security Context Constraints](https://docs.openshift.com/container-platform/latest/authentication/managing-security-context-constraints.html)
- [Milvus Operator for Kubernetes](https://milvus.io/docs/install_cluster-milvusoperator.md)
- [NeMo Guardrails Documentation](https://github.com/NVIDIA/NeMo-Guardrails)
- [NVIDIA NGC Catalog](https://catalog.ngc.nvidia.com/)
- [NVIDIA API Catalog](https://build.nvidia.com/) - Cloud-hosted NIMs

---

## Document Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-01-23 | Auto-generated | Initial assessment |
| 1.1 | 2026-01-27 | - | Added Prerequisites, Step-by-Step Deployment, and Troubleshooting sections; corrected service details and environment variables; updated configuration examples to match actual codebase |

---

*This document provides guidance for deploying the retail-shopping-assistant to OpenShift. Always verify configuration details against the actual codebase as the application evolves.*
