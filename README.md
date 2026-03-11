# Learn ELK Stack - 10-Day Guide

A comprehensive, beginner-friendly ELK Stack learning path. Each day covers a specific topic with detailed explanations, diagrams, examples, and hands-on exercises.

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│   ███████╗██╗     ██╗  ██╗                                      │
│   ██╔════╝██║     ██║ ██╔╝                                      │
│   █████╗  ██║     █████╔╝                                       │
│   ██╔══╝  ██║     ██╔═██╗                                       │
│   ███████╗███████╗██║  ██╗                                      │
│   ╚══════╝╚══════╝╚═╝  ╚═╝                                      │
│                                                                 │
│   ELK Stack Learning Path                                       │
│   Elasticsearch · Logstash · Kibana · Beats                     │
│   From Zero to Hero in 10 Days                                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## 📚 Course Structure

### Week 1: Foundations

| Day | Topic | Description |
|-----|-------|-------------|
| [Day 1](./day-1/README.md) | **Introduction to ELK Stack** | What is ELK, architecture overview, use cases |
| [Day 2](./day-2/README.md) | **Elasticsearch Basics** | Indices, documents, shards, CRUD operations |
| [Day 3](./day-3/README.md) | **Elasticsearch Queries & Mappings** | Query DSL, mappings, analyzers, aggregations |
| [Day 4](./day-4/README.md) | **Logstash** | Pipelines, inputs, filters, outputs, Grok patterns |
| [Day 5](./day-5/README.md) | **Kibana** | Dashboards, Discover, Visualizations, Lens |

### Week 2: Advanced Topics

| Day | Topic | Description |
|-----|-------|-------------|
| [Day 6](./day-6/README.md) | **Beats** | Filebeat, Metricbeat, Heartbeat, Packetbeat |
| [Day 7](./day-7/README.md) | **Index Management & ILM** | Index lifecycle, rollover, snapshots, data tiers |
| [Day 8](./day-8/README.md) | **Security & Authentication** | TLS, role-based access control, API keys |
| [Day 9](./day-9/README.md) | **ELK on Kubernetes** | Elastic Cloud on Kubernetes (ECK), Helm charts |
| [Day 10](./day-10/README.md) | **Troubleshooting & Best Practices** | Common issues, performance tuning, production tips |

### 📋 Quick Reference

| Resource | Description |
|----------|-------------|
| [ELK Cheat Sheet](./cheatsheet.md) | Complete command and query reference for daily use |

---

## 🎯 Learning Objectives

By the end of this course, you will be able to:

- ✅ Understand ELK Stack architecture and how the components work together
- ✅ Install and configure Elasticsearch, Logstash, Kibana, and Beats
- ✅ Index, search, and aggregate data in Elasticsearch
- ✅ Build Logstash pipelines to parse and transform log data
- ✅ Create Kibana dashboards and visualizations
- ✅ Ship logs and metrics with Beats agents
- ✅ Manage index lifecycle policies
- ✅ Secure the ELK Stack with TLS and RBAC
- ✅ Deploy ELK on Kubernetes using ECK
- ✅ Troubleshoot common issues and tune for production

---

## 🛠 Prerequisites

Before starting, you should have:

- Basic understanding of Linux command line
- Familiarity with JSON
- Basic networking knowledge (HTTP, REST APIs)
- Docker installed (for local setup)

### Setting Up a Local ELK Stack

```bash
# Option 1: Docker Compose (recommended for learning)
curl -L https://raw.githubusercontent.com/elastic/start-local/main/start-local.sh | bash

# Option 2: Manual Docker setup
docker network create elk

docker run -d --name elasticsearch \
  --net elk \
  -p 9200:9200 \
  -e "discovery.type=single-node" \
  -e "xpack.security.enabled=false" \
  docker.elastic.co/elasticsearch/elasticsearch:8.12.0

docker run -d --name kibana \
  --net elk \
  -p 5601:5601 \
  -e "ELASTICSEARCH_HOSTS=http://elasticsearch:9200" \
  docker.elastic.co/kibana/kibana:8.12.0

# Verify Elasticsearch is running
curl http://localhost:9200

# Kibana: open http://localhost:5601 in your browser
```

---

## 📖 How to Use This Guide

### For Self-Paced Learning

1. **Follow in order** - Each day builds on previous concepts
2. **Read the theory** - Understand the "why" before the "how"
3. **Type the examples** - Don't just copy-paste, type them out
4. **Experiment** - Modify queries and see what changes
5. **Use the cheat sheet** - Reference it during practice

### Daily Study Plan

```
┌─────────────────────────────────────────────────────────────────┐
│                    RECOMMENDED DAILY SCHEDULE                    │
│                                                                  │
│   📖 30 min - Read the day's content                            │
│   💻 30 min - Follow along with examples                        │
│   🧪 30 min - Experiment and practice                           │
│   📝 15 min - Review and take notes                             │
│                                                                  │
│   Total: ~2 hours per day                                        │
└─────────────────────────────────────────────────────────────────┘
```

---

## 📋 Quick Reference

### Essential Commands

```bash
# Check Elasticsearch health
curl http://localhost:9200/_cluster/health?pretty

# List all indices
curl http://localhost:9200/_cat/indices?v

# Index a document
curl -X POST http://localhost:9200/my-index/_doc \
  -H 'Content-Type: application/json' \
  -d '{"message": "hello elk", "level": "info"}'

# Search
curl http://localhost:9200/my-index/_search?pretty

# Check Logstash pipeline status
curl http://localhost:9600/_node/stats/pipelines?pretty

# Filebeat test config
filebeat test config -e

# Filebeat test output
filebeat test output
```

### ELK Stack Ports

| Service | Default Port | Description |
|---------|-------------|-------------|
| Elasticsearch | 9200 | HTTP REST API |
| Elasticsearch | 9300 | Node transport (cluster) |
| Kibana | 5601 | Web UI |
| Logstash | 5044 | Beats input |
| Logstash | 9600 | Monitoring API |
| Filebeat | - | Agent (no server port) |

---

## 🔗 Additional Resources

### Official Documentation
- [Elasticsearch Docs](https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html)
- [Logstash Docs](https://www.elastic.co/guide/en/logstash/current/index.html)
- [Kibana Docs](https://www.elastic.co/guide/en/kibana/current/index.html)
- [Beats Docs](https://www.elastic.co/guide/en/beats/libbeat/current/index.html)

### Interactive Learning
- [Elastic Training](https://www.elastic.co/training/)
- [Elastic Community](https://discuss.elastic.co/)

### Certification
- [Elastic Certified Engineer (ECE)](https://www.elastic.co/training/elastic-certified-engineer-exam)
- [Elastic Certified Analyst (ECA)](https://www.elastic.co/training/elastic-certified-analyst-exam)

---

## 📁 Repository Structure

```
learn-elk-stack/
├── README.md                 # This file
├── cheatsheet.md             # ELK Quick Reference
├── day-1/README.md           # Introduction to ELK Stack
├── day-2/README.md           # Elasticsearch Basics
├── day-3/README.md           # Queries & Mappings
├── day-4/README.md           # Logstash
├── day-5/README.md           # Kibana
├── day-6/README.md           # Beats
├── day-7/README.md           # Index Management & ILM
├── day-8/README.md           # Security & Authentication
├── day-9/README.md           # ELK on Kubernetes
└── day-10/README.md          # Troubleshooting & Best Practices
```

---

## 🚀 Getting Started

Ready to begin? Start with [Day 1: Introduction to ELK Stack](./day-1/README.md)!

---

## 📝 Notes

- All examples use Elasticsearch 8.x
- REST API examples use `curl` — you can also use Kibana Dev Tools
- Docker Compose is recommended for local practice
- Commands are tested on Elasticsearch 8.12+

---

**Happy Learning!**

*If you find this helpful, consider sharing it with your team!*
