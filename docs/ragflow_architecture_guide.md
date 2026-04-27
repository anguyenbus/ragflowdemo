# RAGFlow Architecture and Services Guide

## Overview

RAGFlow is a distributed RAG (Retrieval-Augmented Generation) engine that requires multiple supporting services to function. This guide explains the service architecture, required components, and how RAGFlow integrates with external APIs.

---

## Core Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              RAGFLOW SYSTEM                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐                  │
│  │   Frontend   │    │  API Server  │    │ Task Executor│                  │
│  │   (React)    │◀──▶│   (Python)   │◀──▶│   (Workers)  │                  │
│  │   Port: 80   │    │   Port: 9380 │    │              │                  │
│  └──────────────┘    └──────────────┘    └──────────────┘                  │
│                                │                                            │
│                                │                                            │
│         ┌──────────────────────┼──────────────────────┐                    │
│         │                      │                      │                    │
│         ▼                      ▼                      ▼                    │
│  ┌──────────┐          ┌──────────┐           ┌───────────┐               │
│  │   MySQL  │          │   Redis  │           │  MinIO/S3 │               │
│  │  (Meta)  │          │  (Cache) │           │(Storage)  │               │
│  └──────────┘          └──────────┘           └───────────┘               │
│         │                      │                      │                    │
│         └──────────────────────┴──────────────────────┘                    │
│                                │                                            │
│                                ▼                                            │
│                    ┌───────────────────┐                                     │
│                    │  Vector Database  │                                     │
│                    │ (Elasticsearch/   │                                     │
│                    │  Infinity/        │                                     │
│                    │  OceanBase)       │                                     │
│                    └───────────────────┘                                     │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
                                │
                                │ API Keys
                                ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         EXTERNAL LLM SERVICES                                │
├─────────────────────────────────────────────────────────────────────────────┤
│  OpenAI | Anthropic | ZhipuAI | BAAI | Local Models | Ollama                │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Required Services

### 1. Vector Database (Document Storage)

RAGFlow stores document chunks and embeddings in a vector database for similarity search.

| Option | Description | Port | Use Case |
|--------|-------------|------|----------|
| **Elasticsearch** (default) | Full-text + vector search | 9200 | General purpose |
| **Infinity** | Specialized vector DB | 23817 | High performance |
| **OceanBase** | Distributed SQL + vector | 2881 | Enterprise scale |
| **OpenSearch** | Elasticsearch alternative | 9201 | AWS compatibility |
| **SeekDB** | Lightweight vector DB | 2881 | Small deployments |

#### Configuration (.env)
```bash
DOC_ENGINE=elasticsearch  # or infinity, oceanbase, opensearch, seekdb
ES_HOST=es01
ES_PORT=1200
ELASTIC_PASSWORD=infini_rag_flow
```

#### Service Configuration (service_conf.yaml)
```yaml
es:
  hosts: 'http://${ES_HOST:-es01}:9200'
  username: '${ES_USER:-elastic}'
  password: '${ELASTIC_PASSWORD:-infini_rag_flow}'
```

---

### 2. MySQL (Metadata Storage)

Stores application metadata, user information, dataset configurations, and model references.

#### Configuration
```yaml
mysql:
  name: '${MYSQL_DBNAME:-rag_flow}'
  user: '${MYSQL_USER:-root}'
  password: '${MYSQL_PASSWORD:-infini_rag_flow}'
  host: '${MYSQL_HOST:-mysql}'
  port: ${MYSQL_PORT:-3306}
  max_connections: 900
```

#### Key Tables
- `user` - User accounts
- `tenant` - Tenant/organization data
- `dataset` - Knowledge base configurations
- `document` - Uploaded file metadata
- `chunk` - Parsed document chunks
- `tenant_llm` - Registered LLM models
- `dialog` - Chat conversations

---

### 3. MinIO (Object Storage)

Stores uploaded files, parsed results, and generated artifacts.

#### Configuration
```yaml
minio:
  user: '${MINIO_USER:-rag_flow}'
  password: '${MINIO_PASSWORD:-infini_rag_flow}'
  host: '${MINIO_HOST:-minio}:9000'
  bucket: '${MINIO_BUCKET:-}'
  prefix_path: '${MINIO_PREFIX_PATH:-}'
```

#### Buckets
- `ragflow` - Main storage bucket
- `sandbox-artifacts` - Charts and generated files

#### Alternative Storage
RAGFlow also supports:
- **AWS S3** - via `s3` configuration
- **Alibaba OSS** - via `oss` configuration
- **Azure Blob** - via `azure` configuration
- **OpenDAL** - Unified storage interface

---

### 4. Redis (Cache and Queue)

Caches frequently accessed data and manages task queues.

#### Configuration
```yaml
redis:
  db: 1
  username: '${REDIS_USERNAME:-}'
  password: '${REDIS_PASSWORD:-infini_rag_flow}'
  host: '${REDIS_HOST:-redis}:6379'
```

#### Uses
- Session caching
- Task queue management
- Temporary storage for parsing results
- Rate limiting

---

### 5. LLM Providers (External Services)

RAGFlow integrates with external LLM services for embeddings, chat, and reranking.

#### Supported Providers

| Provider | Factory Name | Models |
|----------|--------------|--------|
| **OpenAI** | `OpenAI` | gpt-4o, gpt-4o-mini, text-embedding-3-small |
| **Anthropic** | `Anthropic` | claude-3-opus, claude-3-sonnet |
| **ZhipuAI** | `ZhipuAI` | glm-4, glm-4-flash |
| **BAAI** | `BAAI` | bge-m3, bge-reranker-v2 |
| **Ollama** | `Ollama` | Local models |
| **LocalAI** | `LocalAI` | Self-hosted |
| **Xinference** | `Xinference` | Distributed inference |
| **VolcEngine** | `VolcEngine` | Doubao models |

---

## API Key Configuration

### Default Model Configuration

The `user_default_llm` section in `service_conf.yaml` defines default models for new users:

```yaml
user_default_llm:
  factory: 'OpenAI'
  api_key: 'sk-proj-...'                    # Your API key
  base_url: 'https://api.openai.com/v1'     # API endpoint
  default_models:
    chat_model:
      name: 'gpt-4o'
      factory: 'OpenAI'
      api_key: 'sk-proj-...'
    embedding_model:
      name: 'text-embedding-3-small'
      factory: 'OpenAI'
      api_key: 'sk-proj-...'
```

### Per-User Model Configuration

Users can configure their own models through the web UI:

1. Navigate to **Settings** → **Model Providers**
2. Add a new model provider:
   - Select factory (e.g., OpenAI)
   - Enter API key
   - Configure base URL (if using proxy)
3. Select models for:
   - **Chat** - Conversational AI
   - **Embedding** - Vector generation
   - **Rerank** - Result re-ranking
   - **ASR** - Speech-to-text
   - **Image2Text** - Vision tasks

### API Key Storage

- Stored in MySQL `tenant_llm` table
- Encrypted in database
- Never exposed in API responses
- Can be rotated per provider

---

## Service Communication Flow

### Document Upload Flow

```
1. User uploads file via Frontend
                │
                ▼
2. API Server receives file
                │
                ├──▶ Store original in MinIO
                │
                ▼
3. Create document record in MySQL
                │
                ▼
4. Enqueue parsing task (Redis)
                │
                ▼
5. Task Executor picks up task
                │
                ├──▶ Download from MinIO
                │
                ├──▶ Parse with selected parser
                │
                ├──▶ Generate embeddings (LLM API)
                │
                ├──▶ Store chunks in Vector DB
                │
                ▼
6. Update status in MySQL
                │
                ▼
7. Frontend polls for completion
```

### Chat/Retrieval Flow

```
1. User sends question via Frontend
                │
                ▼
2. API Server receives request
                │
                ├──▶ Retrieve relevant chunks (Vector DB)
                │
                ├──▶ Rerank if configured (LLM API)
                │
                ├──▶ Build context prompt
                │
                ▼
3. Send to LLM for generation (LLM API)
                │
                ▼
4. Stream response to Frontend
                │
                └──▶ Save conversation to MySQL
```

---

## Docker Compose Services

### Service List

```bash
# Core Services
ragflow-server     # Main API server (Python)
ragflow-go         # Go backend (optional)
mysql              # MySQL database
redis              # Redis cache
minio              # Object storage

# Vector Database (one selected via DOC_ENGINE)
es01               # Elasticsearch (default)
infinity           # Infinity vector DB
oceanbase          # OceanBase
opensearch01       # OpenSearch
seekdb             # SeekDB

# Optional Services
tei-cpu/tei-gpu    # Text Embeddings Inference (local models)
sandbox-executor-manager  # Code execution sandbox
kibana             # Elasticsearch UI
```

### Starting Services

```bash
cd /path/to/ragflow/docker

# Copy and edit environment
cp .env .env.local
vim .env.local

# Start with default profile (Elasticsearch + CPU)
docker-compose up -d

# Start with specific vector DB
DOC_ENGINE=infinity docker-compose up -d

# Start with GPU support
DEVICE=gpu docker-compose up -d

# Start with TEI (local embeddings)
# Uncomment in .env:
# COMPOSE_PROFILES=${COMPOSE_PROFILES},tei-cpu
```

---

## API Key Sources

### 1. Environment Variables

For deployment configuration:

```bash
# .env file
OPENAI_API_KEY=sk-proj-...
ANTHROPIC_API_KEY=sk-ant-...
ZHIPU_API_KEY=your_key_here
```

### 2. service_conf.yaml

For default models:

```yaml
user_default_llm:
  factory: 'OpenAI'
  api_key: '${OPENAI_API_KEY:-default_key}'
```

### 3. Web UI

For per-user configuration:
- Login as user
- Go to Settings → Model Providers
- Add API key for each provider

---

## Security Considerations

### API Key Management

1. **Never commit API keys to git** - Use `.env` files (gitignored)
2. **Rotate keys regularly** - Update via web UI or database
3. **Use separate keys per environment** - Dev, staging, production
4. **Monitor usage** - Track API costs per tenant

### Internal Service Communication

```yaml
# Services communicate internally via Docker network
# Internal ports:
es01:         9200   (Elasticsearch)
mysql:        3306   (MySQL)
redis:        6379   (Redis)
minio:        9000   (MinIO API)
ragflow:      9380   (RAGFlow API)

# Exposed ports (for external access):
ES_PORT=1200
MINIO_PORT=9000
MINIO_CONSOLE_PORT=9001
SVR_WEB_HTTP_PORT=80
SVR_HTTP_PORT=9380
```

### Database Passwords

Change default passwords before production deployment:

```bash
# .env
MYSQL_PASSWORD=strong_random_password
MINIO_PASSWORD=strong_random_password
ELASTIC_PASSWORD=strong_random_password
REDIS_PASSWORD=strong_random_password
```

---

## Port Reference

| Service | Internal Port | Exposed Port | Protocol |
|---------|---------------|--------------|----------|
| **RAGFlow API** | 9380 | 9380 | HTTP |
| **RAGFlow Admin** | 9381 | 9381 | HTTP |
| **RAGFlow Web** | 80 | 80 | HTTP |
| **Elasticsearch** | 9200 | 1200 | HTTP |
| **MySQL** | 3306 | 3306 | TCP |
| **Redis** | 6379 | 6382 | TCP |
| **MinIO API** | 9000 | 9000 | HTTP |
| **MinIO Console** | 9001 | 9001 | HTTP |
| **Kibana** | 5601 | 6601 | HTTP |
| **Infinity** | 23820 | 23820 | HTTP |
| **TEI** | 80 | 6381 | HTTP |

---

## Troubleshooting

### Service Connection Issues

```bash
# Check if services are running
docker-compose ps

# View service logs
docker-compose logs ragflow-server
docker-compose logs mysql

# Test Elasticsearch connection
curl http://localhost:1200/_cluster/health

# Test MySQL connection
docker exec -it ragflow-mysql mysql -uroot -p
```

### API Key Not Working

1. Verify key in web UI Settings → Model Providers
2. Check `tenant_llm` table in MySQL
3. Verify LLM provider status
4. Check RAGFlow logs for authentication errors

### Vector Database Issues

```bash
# Elasticsearch
curl -u elastic:password http://localhost:1200/_cat/indices

# Reindex if needed
docker exec es01 curl -X DELETE "localhost:9200/ragflow_*"
```

---

## Quick Reference Files

| File | Purpose |
|------|---------|
| `docker/.env` | Environment variables |
| `docker/service_conf.yaml` | Service configuration |
| `docker/docker-compose.yml` | Service definitions |
| `docker/docker-compose-base.yml` | Base services |
| `conf/service_config.yaml` | Runtime config |

---

## Summary

| Component | Required | Alternative |
|-----------|----------|-------------|
| Vector DB | ✅ Yes | Elasticsearch, Infinity, OceanBase, OpenSearch, SeekDB |
| MySQL | ✅ Yes | - |
| Redis | ✅ Yes | - |
| MinIO | ✅ Yes | S3, OSS, Azure, OpenDAL |
| LLM API | ✅ Yes | OpenAI, Anthropic, Zhipu, Local |
| TEI | ❌ Optional | For local embeddings only |
| GPU | ❌ Optional | CPU mode available |
