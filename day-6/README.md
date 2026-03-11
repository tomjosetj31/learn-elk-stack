# Day 6: Beats

## What are Beats?

**Beats** are lightweight, single-purpose data shippers installed directly on servers. They collect data and send it to Elasticsearch (directly) or Logstash (for further processing).

```
┌──────────────────────────────────────────────────────────────────┐
│                     BEATS ECOSYSTEM                              │
│                                                                  │
│   ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌──────────┐  │
│   │ Filebeat   │  │ Metricbeat │  │ Heartbeat  │  │Packetbeat│  │
│   │ (log files)│  │ (metrics)  │  │ (uptime)   │  │(network) │  │
│   └─────┬──────┘  └─────┬──────┘  └─────┬──────┘  └────┬─────┘  │
│         └───────────────┴───────────────┴───────────────┘        │
│                                   │                              │
│                          ┌────────▼────────┐                     │
│                          │   Logstash      │  (optional)         │
│                          └────────┬────────┘                     │
│                                   │                              │
│                          ┌────────▼────────┐                     │
│                          │  Elasticsearch  │                     │
│                          └─────────────────┘                     │
└──────────────────────────────────────────────────────────────────┘
```

---

## Filebeat

### What it does
Tails log files and ships them to Elasticsearch or Logstash.

### Install

```bash
# Linux (DEB)
curl -L -O https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-8.12.0-amd64.deb
dpkg -i filebeat-8.12.0-amd64.deb

# macOS
brew install filebeat

# Docker
docker run -d \
  --name filebeat \
  --net elk \
  --user root \
  -v $(pwd)/filebeat.yml:/usr/share/filebeat/filebeat.yml \
  -v /var/log:/var/log:ro \
  docker.elastic.co/beats/filebeat:8.12.0
```

### filebeat.yml Configuration

```yaml
filebeat.inputs:
  - type: filestream
    id: nginx-access
    enabled: true
    paths:
      - /var/log/nginx/access.log
    fields:
      log_type: nginx-access
    fields_under_root: true

  - type: filestream
    id: app-logs
    enabled: true
    paths:
      - /var/log/myapp/*.log
    multiline:
      pattern: '^\d{4}-\d{2}-\d{2}'  # Line starts with date
      negate: true
      match: after  # Append non-matching lines to previous

processors:
  - add_host_metadata: ~
  - add_cloud_metadata: ~

output.elasticsearch:
  hosts: ["http://localhost:9200"]
  index: "filebeat-%{[agent.version]}-%{+yyyy.MM.dd}"

# OR send to Logstash
# output.logstash:
#   hosts: ["localhost:5044"]

setup.kibana:
  host: "http://localhost:5601"
```

### Common Commands

```bash
# Test config
filebeat test config -e

# Test output connectivity
filebeat test output

# Run Filebeat
filebeat -e

# Setup Kibana dashboards
filebeat setup --dashboards

# Setup index template
filebeat setup --index-management

# List available modules
filebeat modules list

# Enable a module
filebeat modules enable nginx

# Run with a module
filebeat -e -modules=nginx
```

### Filebeat Modules

Modules provide pre-built config for common log formats:

```bash
# Enable nginx module
filebeat modules enable nginx

# Configure the module
vi /etc/filebeat/modules.d/nginx.yml
```

```yaml
# nginx.yml
- module: nginx
  access:
    enabled: true
    var.paths: ["/var/log/nginx/access.log"]
  error:
    enabled: true
    var.paths: ["/var/log/nginx/error.log"]
```

Available modules: `nginx`, `apache`, `mysql`, `postgresql`, `redis`, `kafka`, `elasticsearch`, `kibana`, `logstash`, `system`, `auditd`, `aws`, `gcp`, and many more.

---

## Metricbeat

### What it does
Collects system and service metrics (CPU, memory, disk, network, Docker, Kubernetes, etc.).

### metricbeat.yml Configuration

```yaml
metricbeat.config.modules:
  path: ${path.config}/modules.d/*.yml
  reload.enabled: true

metricbeat.modules:
  - module: system
    metricsets:
      - cpu
      - load
      - memory
      - network
      - process
      - process_summary
      - filesystem
      - diskio
    enabled: true
    period: 10s
    processes: ['.*']

  - module: docker
    metricsets:
      - container
      - cpu
      - diskio
      - memory
      - network
    hosts: ["unix:///var/run/docker.sock"]
    period: 10s

output.elasticsearch:
  hosts: ["http://localhost:9200"]

setup.kibana:
  host: "http://localhost:5601"
```

### Common Metricbeat Modules

| Module | What it monitors |
|--------|-----------------|
| `system` | CPU, memory, disk, network |
| `docker` | Container stats |
| `kubernetes` | Pod/node/namespace metrics |
| `nginx` | Nginx stub status |
| `mysql` | MySQL performance metrics |
| `redis` | Redis server stats |
| `elasticsearch` | Elasticsearch cluster stats |
| `aws` | CloudWatch metrics |

---

## Heartbeat

### What it does
Monitors uptime — checks if services/hosts are reachable.

### heartbeat.yml Configuration

```yaml
heartbeat.monitors:
  - type: http
    id: my-api
    name: "My API"
    urls:
      - "http://api.example.com/health"
    schedule: "@every 30s"
    check.response.status: [200]

  - type: tcp
    id: db-check
    name: "Database"
    hosts: ["database.example.com:5432"]
    schedule: "@every 10s"

  - type: icmp
    id: ping-check
    name: "Server Ping"
    hosts:
      - "192.168.1.10"
    schedule: "@every 60s"

output.elasticsearch:
  hosts: ["http://localhost:9200"]
```

---

## Packetbeat

### What it does
Analyzes network traffic in real-time — captures packets and decodes protocols.

### Supported Protocols
- HTTP/HTTPS
- MySQL
- PostgreSQL
- Redis
- MongoDB
- DNS
- AMQP (RabbitMQ)
- Cassandra

### packetbeat.yml Configuration

```yaml
packetbeat.interfaces.device: any

packetbeat.protocols:
  - type: http
    ports: [80, 8080, 443, 8443]

  - type: mysql
    ports: [3306]

  - type: dns
    ports: [53]

output.elasticsearch:
  hosts: ["http://localhost:9200"]
```

---

## Beats vs Logstash

| Feature | Beats | Logstash |
|---------|-------|---------|
| Resource usage | Very lightweight | Heavy (JVM-based) |
| Data processing | Basic | Rich (Grok, GeoIP, etc.) |
| Plugins | Limited | 200+ plugins |
| Use case | Data shipping | Data transformation |
| Typical setup | Every server | Centralized |

**Common pattern**: Beats (ship) → Logstash (parse) → Elasticsearch (store)

---

## Exercises

1. Install Filebeat and configure it to read `/var/log/syslog` (or `/var/log/system.log` on macOS)
2. Send logs directly to Elasticsearch
3. Verify logs appear in Kibana (create a data view for `filebeat-*`)
4. Enable the `system` Metricbeat module and collect CPU/memory metrics
5. Set up Heartbeat to monitor `http://localhost:9200` every 30 seconds
6. View Heartbeat results in Kibana Uptime app

---

## Summary

- **Filebeat**: Ship log files — most commonly used Beat
- **Metricbeat**: Collect system and service metrics
- **Heartbeat**: Monitor service uptime
- **Packetbeat**: Capture and analyze network traffic
- Beats are lightweight and run on every server
- Use Logstash between Beats and Elasticsearch when you need heavy log transformation

---

**Previous:** [Day 5 - Kibana](../day-5/README.md) | **Next:** [Day 7 - Index Management & ILM](../day-7/README.md)
