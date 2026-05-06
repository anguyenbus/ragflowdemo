# Using AWS OpenSearch with RAGFlow

## Overview

RAGFlow supports AWS OpenSearch Service as a vector database alternative to self-hosted Elasticsearch. This guide explains how to configure RAGFlow to use AWS OpenSearch for document storage, vector search, and retrieval.

---

## Why AWS OpenSearch?

| Feature | AWS OpenSearch | Self-Hosted Elasticsearch |
|---------|----------------|---------------------------|
| **Management** | Fully managed | Manual setup & maintenance |
| **Scaling** | Auto-scaling | Manual scaling |
| **High Availability** | Built-in multi-AZ | Manual configuration |
| **Security** | AWS IAM, VPC, encryption | Manual setup |
| **Pricing** | Pay-as-you-go | Fixed infrastructure cost |
| **Maintenance** | AWS handled | Self-managed |
| **Version Support** | OpenSearch 1.x-2.x | Any version |

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         RAGFLOW + AWS OPENSEARCH                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌──────────────┐                                                            │
│  │   RAGFlow    │                                                            │
│  │  (Docker)    │                                                            │
│  │              │                                                            │
│  │  - API       │                                                            │
│  │  - Parser    │                                                            │
│  │  - Workers   │                                                            │
│  └──────┬───────┘                                                            │
│         │                                                                    │
│         │ HTTPS (Vector Search API)                                          │
│         │                                                                    │
│         ▼                                                                    │
│  ┌────────────────────────────────────┐                                      │
│  │     AWS OpenSearch Domain          │                                      │
│  │  ┌─────────────────────────────┐   │                                      │
│  │  │  Vector Index (k-NN)        │   │                                      │
│  │  │  - Document chunks          │   │                                      │
│  │  │  - Embeddings (1536 dim)    │   │                                      │
│  │  │  - Full-text search         │   │                                      │
│  │  │  - Aggregations             │   │                                      │
│  │  └─────────────────────────────┘   │                                      │
│  │                                    │                                      │
│  │  - Fine-grained access control     │                                      │
│  │  - Encryption at rest/in transit   │                                      │
│  │  - Snapshot to S3                  │                                      │
│  └────────────────────────────────────┘                                      │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Prerequisites

### AWS Requirements

1. **AWS Account** with appropriate permissions
2. **OpenSearch Domain** (version 2.x required)
3. **VPC Configuration** (recommended for production)
4. **S3 Bucket** for snapshots (optional but recommended)

### OpenSearch Domain Requirements

| Requirement | Minimum | Recommended |
|-------------|---------|-------------|
| **Version** | 2.x | Latest 2.x |
| **Instance Type** | t3.small.search | r6g.large.search |
| **Instance Count** | 1 | 3+ (multi-AZ) |
| **EBS Volume** | 10GB GP2 | 100GB GP3 |
| **Memory** | 2GB | 16GB+ |

### RAGFlow Requirements

- RAGFlow v0.25.0+
- Docker or Kubernetes deployment
- Network connectivity to AWS OpenSearch

---

## Step 1: Create AWS OpenSearch Domain

### Option A: AWS Console

1. Navigate to **OpenSearch Service** in AWS Console
2. Click **Create domain**
3. Configure:
   ```
   Domain name: ragflow-prod
   Engine version: OpenSearch 2.x
   ```
4. **Network**:
   - For development: Public access
   - For production: VPC access
5. **Data nodes**:
   ```
   Instance type: r6g.large.search
   Number of nodes: 3
   Storage: 100GB GP3
   ```
6. **Security**:
   - Fine-grained access control: **Enabled**
   - Create master user: `ragflow_admin`
   - HTTPS required: **Yes**
7. **Access policy** (if public):
   ```json
   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Effect": "Allow",
         "Principal": {
           "AWS": "*"
         },
         "Action": "es:*",
         "Resource": "arn:aws:es:REGION:ACCOUNT:domain/DOMAIN_NAME/*"
       }
     ]
   }
   ```
8. Create domain

### Option B: AWS CLI

```bash
aws opensearch create-domain \
  --domain-name ragflow-prod \
  --engine-version OpenSearch_2.11 \
  --cluster-config \
    InstanceType=r6g.large.search, \
    InstanceCount=3, \
    DedicatedMasterEnabled=false, \
    ZoneAwarenessEnabled=true, \
    ZoneAwarenessConfig={AvailabilityZoneCount=3} \
  --ebs-options \
    EBSEnabled=true, \
    VolumeType=gp3, \
    VolumeSize=100 \
  --access-policies '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {"AWS": "*"},
      "Action": "es:*",
      "Resource": "arn:aws:es:REGION:ACCOUNT:domain/ragflow-prod/*"
    }]
  }' \
  --advanced-security-options \
    Enabled=true, \
    InternalUserDatabaseEnabled=true, \
    MasterUserOptions={MasterUserName=ragflow_admin,MasterUserPassword=YOUR_PASSWORD}
```

### Option C: Terraform

```hcl
resource "aws_opensearch_domain" "ragflow" {
  domain_name    = "ragflow-prod"
  engine_version = "OpenSearch_2.11"

  cluster_config {
    instance_type          = "r6g.large.search"
    instance_count         = 3
    zone_awareness_enabled = true
    availability_zone_count = 3
  }

  ebs_options {
    ebs_enabled = true
    volume_type = "gp3"
    volume_size = 100
  }

  advanced_security_options {
    enabled                        = true
    internal_user_database_enabled = true
    master_user_options {
      master_user_name     = "ragflow_admin"
      master_user_password = var.opensearch_password
    }
  }

  encrypt_at_rest {
    enabled = true
  }

  node_to_node_encryption {
    enabled = true
  }

  domain_endpoint_options {
    enforce_https = true
    tls_security_policy = "Policy-Min-TLS-1-2-2019-07"
  }
}
```

---

## Step 2: Configure Network Access

### For Development (Public Endpoint)

1. Note the domain endpoint from AWS Console
2. Format: `https://search-XXXXXX.region.amazonaws.com`

### For Production (VPC Access)

1. Create VPC endpoint for OpenSearch
2. Configure security groups:
   ```bash
   # Allow RAGFlow containers to access OpenSearch
   aws ec2 authorize-security-group-ingress \
     --group-id sg-XXXXX \
     --protocol tcp \
     --port 443 \
     --source-group sg-RAGFLOW
   ```

3. Update DNS resolution for VPC endpoint

---

## Step 3: Configure RAGFlow

### Method A: Docker Environment Variables

Edit `/path/to/ragflow/docker/.env`:

```bash
# Use OpenSearch as document engine
DOC_ENGINE=opensearch

# Remove/Disable Elasticsearch variables
# ES_HOST=es01
# ES_PORT=1200

# OpenSearch Configuration
OS_HOST=search-xxxxx.us-east-1.es.amazonaws.com
OS_PORT=443
OS_USER=ragflow_admin
OPENSEARCH_PASSWORD=your_secure_password

# For HTTPS (AWS OpenSearch requires HTTPS)
OS_HTTPS=true
OS_VERIFY_CERTS=true

# Update profiles (remove elasticsearch)
COMPOSE_PROFILES=opensearch,cpu
```

### Method B: service_conf.yaml

Edit `/path/to/ragflow/docker/service_conf.yaml.template`:

```yaml
# Comment out or remove Elasticsearch config
# es:
#   hosts: 'http://${ES_HOST:-es01}:9200'
#   username: '${ES_USER:-elastic}'
#   password: '${ELASTIC_PASSWORD:-infini_rag_flow}'

# Configure OpenSearch for AWS
os:
  hosts: 'https://${OS_HOST:-search-xxxxx.region.es.amazonaws.com}:443'
  username: '${OS_USER:-ragflow_admin}'
  password: '${OPENSEARCH_PASSWORD:-your_password}'
  # Optional: SSL verification
  verify_certs: ${OS_VERIFY_CERTS:-true}

# Other configurations remain the same
mysql:
  name: '${MYSQL_DBNAME:-rag_flow}'
  # ... rest of config
```

### Method C: Kubernetes Helm Values

```yaml
# values.yaml
opensearch:
  enabled: false  # Don't deploy local OpenSearch

externalOpenSearch:
  enabled: true
  host: "search-xxxxx.us-east-1.es.amazonaws.com"
  port: 443
  username: "ragflow_admin"
  password: "your_password"
  protocol: "https"
  verifyCerts: true
```

---

## Step 4: Update Docker Compose

### Disable Local OpenSearch/Elasticsearch

Edit `/path/to/ragflow/docker/docker-compose.yml`:

```yaml
# Comment out or remove local OpenSearch/Elasticsearch services
# services:
#   es01:
#     image: ...
#     ...
#   opensearch01:
#     image: ...
#     ...

# Keep only RAGFlow services
services:
  ragflow-server:
    depends_on:
      - mysql
      - redis
      - minio
    # ... rest of config
```

### Start RAGFlow

```bash
cd /path/to/ragflow/docker

# Set environment variables
export DOC_ENGINE=opensearch
export OS_HOST=search-xxxxx.region.amazonaws.com
export OS_USER=ragflow_admin
export OPENSEARCH_PASSWORD=your_password

# Start services (excluding local OpenSearch)
docker-compose up -d ragflow-server ragflow-go mysql redis minio
```

---

## Step 5: Verify Connection

### Check RAGFlow Logs

```bash
docker-compose logs ragflow-server | grep -i opensearch
```

Expected output:
```
ragflow-server | Use OpenSearch https://search-xxxxx.region.amazonaws.com as the doc engine.
ragflow-server | OpenSearch https://search-xxxxx.region.amazonaws.com is healthy.
```

### Test OpenSearch Connection

```bash
# From RAGFlow container
docker-compose exec ragflow-server bash -c "
  curl -u ragflow_admin:password \
    https://search-xxxxx.region.amazonaws.com/_cluster/health
"

# Expected response
{
  "cluster_name" : "123456789012:ragflow-prod",
  "status" : "green",
  "number_of_nodes" : 3,
  ...
}
```

### Check Index Creation

After uploading a document:

```bash
curl -u ragflow_admin:password \
  https://search-xxxxx.region.amazonaws.com/_cat/indices?v
```

---

## SSL/TLS Configuration

### AWS Certificate Management

AWS OpenSearch uses AWS-managed certificates. The endpoint URL includes the domain:

```
https://search-XXXXX.us-east-1.es.amazonaws.com
```

### RAGFlow SSL Settings

For production, ensure SSL verification is enabled:

```python
# In opensearch_conn.py (or via config)
self.os = OpenSearch(
    hosts,
    http_auth=(username, password),
    verify_certs=True,  # Enable for production
    ssl_show_warn=False,
    timeout=600
)
```

### CA Bundle (if needed)

For self-signed certificates (development only):

```bash
# Mount CA bundle to container
docker-compose.yml:
  ragflow-server:
    volumes:
      - ./ca-bundle.crt:/usr/local/share/ca-certificates/ca-bundle.crt:ro
    environment:
      - REQUESTS_CA_BUNDLE=/usr/local/share/ca-certificates/ca-bundle.crt
```

---

## Authentication Options

### Basic Authentication (Internal User Database)

Default method for AWS OpenSearch:

```yaml
os:
  hosts: 'https://search-xxxxx.region.amazonaws.com'
  username: 'ragflow_admin'
  password: 'your_password'
```

### IAM Authentication (Recommended for Production)

1. Create IAM role for RAGFlow
2. Attach policy with OpenSearch permissions:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "es:ESHttp*"
      ],
      "Resource": "arn:aws:es:REGION:ACCOUNT:domain/ragflow-prod/*"
    }
  ]
}
```

3. Configure RAGFlow to use IAM credentials (requires SDK modification)

### Fine-Grained Access Control

Create limited user for RAGFlow:

```bash
# Via OpenSearch Dashboard
# Create user: ragflow_app
# Permissions:
#   - index_patterns: ragflow*
#   - permissions: indices_all
```

---

## Vector Search Configuration

### Enable k-NN in OpenSearch

OpenSearch 2.x includes k-NN plugin by default. Verify:

```bash
curl -u user:pass https://domain/_cat/plugins?v
```

Should show:
```
opensearch-knn  2.x.x
```

### Index Mapping

RAGFlow automatically creates indices with k-NN configuration:

```json
{
  "settings": {
    "index": {
      "knn": true,
      "knn.algo_param.ef_search": 100
    }
  },
  "mappings": {
    "properties": {
      "q_1536_vec": {
        "type": "knn_vector",
        "dimension": 1536,
        "method": {
          "name": "hnsw",
          "space_type": "l2",
          "engine": "nmslib",
          "parameters": {
            "ef_construction": 128,
            "m": 24
          }
        }
      }
    }
  }
}
```

---

## Performance Optimization

### Instance Sizing Guide

| Documents | Embeddings | Recommended Instance |
|-----------|------------|---------------------|
| < 10K | < 1M | t3.small.search |
| 10K-100K | 1M-10M | r6g.large.search |
| 100K-1M | 10M-100M | r6g.2xlarge.search |
| > 1M | > 100M | r6g.8xlarge.search |

### Storage Optimization

```bash
# Force merge index (reduce segments)
curl -X POST -u user:pass \
  "https://domain/ragflow/_forcemerge?max_num_segments=1"

# Refresh interval (default 1s)
curl -X PUT -u user:pass \
  "https://domain/ragflow/_settings" \
  -H 'Content-Type: application/json' \
  -d '{"index": {"refresh_interval": "30s"}}'
```

### Query Optimization

```yaml
# In RAGFlow service_conf.yaml
parser_config:
  chunk_token_num: 512
  retrieval:
    vector_similarity_weight: 0.7
    topn: 20
```

---

## Backup and Recovery

### Automated Snapshots to S3

1. Create S3 bucket for snapshots
2. Register snapshot repository:

```bash
curl -X PUT -u user:pass \
  "https://domain/_snapshot/ragflow-backup" \
  -H 'Content-Type: application/json' \
  -d '{
    "type": "s3",
    "settings": {
      "bucket": "ragflow-opensearch-backups",
      "region": "us-east-1",
      "role_arn": "arn:aws:iam::ACCOUNT:role/OpenSearchSnapshotRole"
    }
  }'
```

3. Create snapshot:

```bash
curl -X PUT -u user:pass \
  "https://domain/_snapshot/ragflow-backup/snapshot-1?wait_for_completion=true"
```

### Restore from Snapshot

```bash
curl -X POST -u user:pass \
  "https://domain/_snapshot/ragflow-backup/snapshot-1/_restore" \
  -H 'Content-Type: application/json' \
  -d '{
    "indices": "ragflow*",
    "ignore_unavailable": true,
    "include_global_state": false
  }'
```

---

## Monitoring

### CloudWatch Metrics

AWS OpenSearch automatically publishes metrics to CloudWatch:

- `ClusterStatus.red`
- `ClusterStatus.yellow`
- `Nodes`
- `CPUUtilization`
- `FreeStorageSpace`
- `IndexingLatency`

### OpenSearch Dashboard

Access dashboard at:
```
https://search-xxxxx.region.amazonaws.com/_dashboards/
```

---

## Troubleshooting

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| "Connection timeout" | Security group blocking | Add ingress rule for port 443 |
| "Authentication failed" | Wrong credentials | Verify username/password |
| "Certificate verify failed" | Self-signed cert | Set `verify_certs=false` (dev only) |
| "Index not found" | First run | Auto-created on first document |
| "403 Forbidden" | IAM permissions missing | Add IAM policy for OpenSearch |

### Debug Commands

```bash
# Check RAGFlow OpenSearch configuration
docker-compose exec ragflow-server env | grep OS_

# Test connectivity
docker-compose exec ragflow-server \
  curl -v https://search-xxxxx.region.amazonaws.com/_cluster/health

# Check RAGFlow logs
docker-compose logs -f ragflow-server | grep -i opensearch
```

### Enable Debug Logging

```python
# In RAGFlow settings
import logging
logging.getLogger('ragflow.opensearch_conn').setLevel(logging.DEBUG)
```

---

## Cost Optimization

### Development Environment

- Use t3.small.search instances
- Single data node
- 10GB GP2 storage
- Estimated: ~$30/month

### Production Environment

- Use r6g.large.search (Graviton - better price/performance)
- 3 data nodes with multi-AZ
- 100GB GP3 storage
- Estimated: ~$300/month

### Cost Saving Tips

1. Use reserved instances for long-running deployments
2. Enable ultrawarm storage for old indices
3. Delete unused indices
4. Use GP3 instead of GP2 for storage
5. Consider domain sharing for multiple environments

---

## Security Best Practices

1. **Use VPC endpoints** for production
2. **Enable fine-grained access control**
3. **Rotate credentials regularly**
4. **Enable encryption at rest and in transit**
5. **Use IAM roles instead of static credentials**
6. **Restrict access via security groups**
7. **Enable CloudTrail logging**
8. **Regular security audits**

---

## Migration from Self-Hosted Elasticsearch

### Export from Existing

```bash
# Export data from self-hosted Elasticsearch
curl -X POST "localhost:9200/_snapshot/backup/snapshot_1?wait_for_completion=true"

# Copy to S3
aws s3 cp /backup/ s3://ragflow-migration/ --recursive
```

### Import to AWS OpenSearch

```bash
# Register S3 as snapshot repository
curl -X PUT "https://domain/_snapshot/migration" \
  -d '{
    "type": "s3",
    "settings": {
      "bucket": "ragflow-migration",
      "region": "us-east-1"
    }
  }'

# Restore
curl -X POST "https://domain/_snapshot/migration/snapshot_1/_restore"
```

---

## Summary

| Task | Command/Config |
|------|----------------|
| Create domain | AWS Console / CLI / Terraform |
| Set DOC_ENGINE | `DOC_ENGINE=opensearch` |
| Configure host | `OS_HOST=search-xxxxx.region.amazonaws.com` |
| Set credentials | `OS_USER`, `OPENSEARCH_PASSWORD` |
| Enable SSL | `OS_HTTPS=true`, `OS_VERIFY_CERTS=true` |
| Start RAGFlow | `docker-compose up -d` |
| Verify connection | Check logs + `_cluster/health` API |
