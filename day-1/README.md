# Day 1: Introduction to ELK Stack

## What is the ELK Stack?

The **ELK Stack** is a collection of three open-source tools — **Elasticsearch**, **Logstash**, and **Kibana** — built and maintained by Elastic. Together they form a powerful platform for **centralized logging, search, and analytics**.

The stack is often extended with **Beats** (lightweight data shippers), making it the **Elastic Stack**.

```
┌─────────────────────────────────────────────────────────────────────┐
│                        ELK STACK OVERVIEW                           │
│                                                                     │
│   [Your Apps / Servers]                                             │
│          │                                                          │
│          ▼                                                          │
│   ┌─────────────┐    ┌──────────────┐                               │
│   │   Beats     │    │   Logstash   │  ← Collect & Parse logs       │
│   │ (Filebeat,  │───▶│  (optional)  │                               │
│   │ Metricbeat) │    └──────┬───────┘                               │
│   └─────────────┘           │                                       │
│                             ▼                                       │
│                   ┌──────────────────┐                              │
│                   │  Elasticsearch   │  ← Store & Search            │
│                   │  (search engine) │                              │
│                   └────────┬─────────┘                              │
│                            │                                        │
│                            ▼                                        │
│                   ┌──────────────────┐                              │
│                   │     Kibana       │  ← Visualize & Explore       │
│                   │  (web UI)        │                              │
│                   └──────────────────┘                              │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Core Components

### 1. Elasticsearch
- **What**: Distributed search and analytics engine
- **Built on**: Apache Lucene
- **Data format**: JSON documents
- **Query language**: Query DSL (JSON-based)
- **Key features**: Full-text search, real-time indexing, horizontal scaling

### 2. Logstash
- **What**: Server-side data processing pipeline
- **Role**: Ingest → Parse → Transform → Output
- **Inputs**: Files, syslog, Beats, Kafka, HTTP, and more
- **Filters**: Grok, mutate, date, GeoIP, and more
- **Outputs**: Elasticsearch, S3, Kafka, and more

### 3. Kibana
- **What**: Web-based UI for Elasticsearch
- **Features**: Discover, Dashboards, Visualizations, Maps, Alerting
- **Accessed at**: `http://localhost:5601`

### 4. Beats
- **What**: Lightweight single-purpose data shippers
- **Types**:
  - **Filebeat** — Log files
  - **Metricbeat** — System/service metrics
  - **Heartbeat** — Uptime monitoring
  - **Packetbeat** — Network data
  - **Auditbeat** — Audit framework data

---

## Why Use ELK Stack?

| Problem | ELK Solution |
|---------|-------------|
| Logs scattered across many servers | Centralize all logs in one place |
| Hard to search logs | Full-text search across billions of documents |
| No visibility into application health | Real-time dashboards and alerting |
| Compliance & audit requirements | Long-term log retention with ILM |
| Security monitoring | SIEM capabilities with Elastic Security |

---

## Common Use Cases

1. **Log aggregation** — Collect logs from all microservices into one place
2. **Application Performance Monitoring (APM)** — Trace requests across services
3. **Security Information & Event Management (SIEM)** — Detect threats
4. **Infrastructure monitoring** — CPU, memory, disk, network metrics
5. **Business analytics** — Analyze user behavior, sales data, etc.

---

## Key Concepts

### Index
An index is like a database table. It stores related documents.
```
Example: logs-nginx-2024.03.11
         └── Index pattern: logs-nginx-*
```

### Document
A document is a single JSON record stored in an index.
```json
{
  "@timestamp": "2024-03-11T10:30:00Z",
  "level": "error",
  "service": "api",
  "message": "Connection timeout to database"
}
```

### Shard
Elasticsearch splits indices into shards for horizontal scaling. Each shard is a self-contained Lucene index.

```
Index: logs-2024
├── Shard 0 (primary) → Node 1
├── Shard 1 (primary) → Node 2
├── Shard 0 (replica) → Node 2
└── Shard 1 (replica) → Node 1
```

### Cluster & Nodes
- **Cluster**: A group of Elasticsearch nodes working together
- **Node types**: Master, Data, Ingest, Coordinating

---

## Data Flow

```
Log File / App
     │
     ▼
Filebeat (tail log file)
     │
     ▼
Logstash (parse, enrich, transform)
     │  ← Grok pattern: parse Apache log format
     │  ← Add GeoIP from IP address
     │  ← Parse timestamp
     ▼
Elasticsearch (index & store)
     │
     ▼
Kibana (search, visualize, alert)
```

---

## Hands-On: Start Your First ELK Setup

### Step 1: Pull Docker images

```bash
docker pull docker.elastic.co/elasticsearch/elasticsearch:8.12.0
docker pull docker.elastic.co/kibana/kibana:8.12.0
```

### Step 2: Start Elasticsearch

```bash
docker network create elk

docker run -d \
  --name elasticsearch \
  --net elk \
  -p 9200:9200 \
  -e "discovery.type=single-node" \
  -e "xpack.security.enabled=false" \
  docker.elastic.co/elasticsearch/elasticsearch:8.12.0
```

### Step 3: Verify Elasticsearch

```bash
curl http://localhost:9200
```

Expected output:
```json
{
  "name" : "elasticsearch",
  "cluster_name" : "docker-cluster",
  "version" : {
    "number" : "8.12.0",
    ...
  },
  "tagline" : "You Know, for Search"
}
```

### Step 4: Start Kibana

```bash
docker run -d \
  --name kibana \
  --net elk \
  -p 5601:5601 \
  -e "ELASTICSEARCH_HOSTS=http://elasticsearch:9200" \
  docker.elastic.co/kibana/kibana:8.12.0
```

### Step 5: Access Kibana

Open your browser at `http://localhost:5601`

---

## Exercises

1. Start Elasticsearch and Kibana using Docker as shown above
2. Verify Elasticsearch responds at `http://localhost:9200`
3. Explore the Kibana home page at `http://localhost:5601`
4. Open Kibana **Dev Tools** (Menu > Management > Dev Tools) and run:
   ```
   GET /
   GET _cluster/health
   GET _cat/nodes?v
   ```

---

## Summary

| Component | Role | Port |
|-----------|------|------|
| Elasticsearch | Store & search data | 9200 |
| Logstash | Parse & transform data | 5044, 9600 |
| Kibana | Visualize & explore | 5601 |
| Filebeat | Ship log files | - |
| Metricbeat | Ship metrics | - |

---

**Next:** [Day 2 - Elasticsearch Basics](../day-2/README.md)
