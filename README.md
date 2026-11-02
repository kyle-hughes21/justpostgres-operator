# JustPostgres Operator

**"Stop paying for SaaS. Just use PostgreSQL."**

A Kubernetes operator that makes PostgreSQL your entire backend — replacing Redis, Elasticsearch, MongoDB, Kafka, Firebase, and more with native PostgreSQL extensions.

## Philosophy

> **Stop paying for SaaS tools. I built a $0 tech stack with just PostgreSQL.**

Modern applications need multiple data models: caching, search, documents, time-series, queues, real-time, auth. The default answer is a pile of SaaS: Redis Cloud, Elastic Cloud, MongoDB Atlas, Confluent, Firebase, Supabase, Upstash...

But PostgreSQL has native extensions for **all of it**:
- **Caching** → `pg_redis` (Redis protocol), `pg_cron` (scheduled jobs)
- **Search** → `pg_search` / ParadeDB (Elasticsearch API), `pg_trgm` (trigram)
- **Documents** → `FerretDB` (MongoDB wire protocol), `jsonb` + GIN
- **Time-series** → `TimescaleDB` (hypertables, continuous aggregates)
- **Queues** → `pg_queue` / `pgmq` (Kafka-like), `pg_notify` (pub/sub)
- **Real-time** → `pg_notify` + `LISTEN/NOTIFY`, Supabase Realtime compatible
- **Auth** → `pgsodium` (crypto), `pgjwt` (JWT), row-level security
- **Vectors/AI** → `pgvector` (embeddings, RAG)
- **Analytics** → `pg_analytics` / `pg_duckdb` (OLAP)

One cluster. One backup. One upgrade. One mental model.

## Quick Start

```bash
# Install CRDs
kubectl apply -f https://github.com/kyle-hughes21/justpostgres-operator/releases/latest/download/crds.yaml

# Install operator
kubectl apply -f https://github.com/kyle-hughes21/justpostgres-operator/releases/latest/download/operator.yaml

# Create your cluster
cat <<EOF | kubectl apply -f -
apiVersion: justpostgres.io/v1
kind: JustPostgresCluster
metadata:
  name: my-backend
spec:
  postgres:
    version: "16"
    replicas: 3
  extensions:
    cache: true          # pg_redis
    search: true         # pg_search / ParadeDB
    documents: true      # FerretDB (MongoDB API)
    timeseries: true     # TimescaleDB
    queue: true          # pgmq
    realtime: true       # pg_notify + Supabase Realtime
    vectors: true        # pgvector
  connectionPool:
    pgbouncer:
      replicas: 3
  backup:
    schedule: "0 2 * * *"
    s3:
      bucket: "my-backups"
EOF
```

## What You Replace

| SaaS Category | SaaS Examples | JustPostgres Extension |
|---------------|---------------|------------------------|
| **Caching** | Redis Cloud, Upstash | `pg_redis`, `pg_cron` |
| **Search** | Elastic Cloud, Algolia, Meilisearch | `pg_search` (ParadeDB), `pg_trgm` |
| **Documents** | MongoDB Atlas, Firebase, DynamoDB | `FerretDB` (MongoDB wire protocol) |
| **Time-series** | InfluxDB, Timescale Cloud | `TimescaleDB` (hypertables) |
| **Queues** | Kafka, RabbitMQ, SQS, Upstash | `pgmq`, `pg_notify` |
| **Real-time** | Pusher, Ably, Supabase Realtime | `pg_notify` + Realtime server |
| **Auth** | Auth0, Clerk, Firebase Auth | `pgsodium`, `pgjwt`, RLS |
| **Vectors/RAG** | Pinecone, Weaviate, Chroma | `pgvector` |
| **Analytics** | Snowflake, BigQuery, ClickHouse | `pg_analytics`, `pg_duckdb` |

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     JustPostgresCluster                          │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │              Primary PostgreSQL (Writer)                   │  │
│  │  ┌─────────┐ ┌──────────┐ ┌─────────┐ ┌──────────────┐   │  │
│  │  │ Core    │ │ Timescale│ │FerretDB │ │ pg_redis/    │   │  │
│  │  │ PG 16   │ │ 2.16     │ │ (Mongo) │ │ pg_search    │   │  │
│  │  └─────────┘ └──────────┘ └─────────┘ └──────────────┘   │  │
│  └───────────────────────────────────────────────────────────┘  │
│                           │                                      │
│          ┌────────────────┼────────────────┐                    │
│          ▼                ▼                ▼                    │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐            │
│  │ Replica 1   │  │ Replica 2   │  │ Replica 3   │  (Sync/Async)│
│  │ (Read)      │  │ (Read)      │  │ (Read)      │              │
│  └─────────────┘  └─────────────┘  └─────────────┘            │
│                           │                                      │
│          ┌────────────────┼────────────────┐                    │
│          ▼                ▼                ▼                    │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                    PgBouncer Pool (3x)                     │  │
│  │  Transaction pooling • Prepared statements • TLS termination│  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                              │
         ┌────────────────────┼────────────────────┐
         ▼                    ▼                    ▼
┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
│  WAL-G /        │ │   PITR          │ │  Cross-engine   │
│  Barman Cloud   │ │  (Point-in-Time │ │  Query          │
│  Backup         │ │   Recovery)     │ │  Federation     │
└─────────────────┘ └─────────────────┘ └─────────────────┘
```

## CRD Specification

```yaml
apiVersion: justpostgres.io/v1
kind: JustPostgresCluster
metadata:
  name: my-app
spec:
  postgres:
    version: "16"
    replicas: 3
    resources:
      requests:
        memory: "8Gi"
        cpu: "4"
      limits:
        memory: "16Gi"
        cpu: "8"
    storage:
      size: "500Gi"
      class: "fast-ssd"
  
  extensions:
    cache: true              # pg_redis - Redis protocol
    search: true             # pg_search - Elasticsearch API
    documents: true          # FerretDB - MongoDB API
    timeseries: true         # TimescaleDB - hypertables
    queue: true              # pgmq - Kafka-like queues
    realtime: true           # pg_notify - Supabase Realtime
    vectors: true            # pgvector - embeddings/RAG
    analytics: true          # pg_duckdb - OLAP
  
  connectionPool:
    pgbouncer:
      replicas: 3
      poolMode: "transaction"
      maxClientConn: 10000
  
  backup:
    schedule: "0 2 * * *"
    retention: "30d"
    storage:
      s3:
        bucket: "justpostgres-backups"
        prefix: "my-app"
    pitr: true
    encryption: true
  
  monitoring:
    prometheus: true
    grafanaDashboards: true
```

## Operator Features

| Feature | Status |
|---------|--------|
| **Provisioning** | ✅ Automated bootstrap, TLS, secrets |
| **HA & Failover** | ✅ Patroni, <30s failover |
| **Replication** | ✅ Streaming + logical, cascading |
| **Connection Pooling** | ✅ PgBouncer sidecars, TLS termination |
| **Backup & PITR** | ✅ WAL-G/Barman, S3/GCS/Azure, encryption |
| **Extension Management** | ✅ Version compatibility matrix, auto-upgrade |
| **Observability** | ✅ Prometheus exporter, Grafana dashboards |
| **Security** | ✅ TLS everywhere, mTLS, Vault integration |
| **Upgrades** | ✅ In-place minor, blue-green major |
| **Multi-tenancy** | ✅ Schema isolation, quotas |

## Why This Demonstrates Kubernetes Depth

1. **Stateful workload mastery** — PostgreSQL HA is one of the hardest stateful problems on K8s
2. **Extension ecosystem as first-class** — Managing 10+ extensions with version compatibility
3. **Multi-model unification** — Solving "polyglot persistence" at infrastructure level
4. **Full Day 2 operations** — Backup, PITR, failover, upgrades, scaling
5. **Plugin architecture** — Clean framework for adding new extensions without core changes

## Project Status

**Pre-Alpha** — Core reconciliation loop working. Extension framework designed. Seeking contributors.

## Links

- **Source**: [github.com/kyle-hughes21/justpostgres-operator](https://github.com/kyle-hughes21/justpostgres-operator)
- **Design Doc**: [docs/DESIGN.md](https://github.com/kyle-hughes21/justpostgres-operator/blob/main/docs/DESIGN.md)
- **Extension SDK**: [docs/EXTENSIONS.md](https://github.com/kyle-hughes21/justpostgres-operator/blob/main/docs/EXTENSIONS.md)