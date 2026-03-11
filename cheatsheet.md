# ELK Stack Cheat Sheet

## Elasticsearch REST API

### Cluster

```bash
# Health
GET _cluster/health?pretty
GET _cluster/stats?pretty
GET _cluster/pending_tasks

# Nodes
GET _cat/nodes?v&h=name,heap.percent,cpu,disk.used_percent
GET _nodes/stats?pretty
GET _nodes/hot_threads

# Allocation
GET _cluster/allocation/explain
GET _cat/allocation?v
GET _cat/shards?v

# Settings
GET _cluster/settings?pretty
PUT _cluster/settings
{ "persistent": { "key": "value" } }
```

### Indices

```bash
# List
GET _cat/indices?v
GET _cat/indices?v&h=index,health,docs.count,store.size&s=store.size:desc

# Create
PUT /my-index
{ "settings": { "number_of_shards": 1, "number_of_replicas": 1 } }

# Delete
DELETE /my-index

# Stats
GET /my-index/_stats
GET /my-index/_mapping
GET /my-index/_settings

# Refresh / Flush
POST /my-index/_refresh
POST /my-index/_flush

# Update settings
PUT /my-index/_settings
{ "number_of_replicas": 0 }

# Forcemerge
POST /my-index/_forcemerge?max_num_segments=1
```

### Documents

```bash
# Create
POST /my-index/_doc
{ "field": "value" }

PUT /my-index/_doc/1
{ "field": "value" }

# Read
GET /my-index/_doc/1

# Update (partial)
POST /my-index/_update/1
{ "doc": { "field": "new-value" } }

# Delete
DELETE /my-index/_doc/1

# Bulk
POST _bulk
{ "index": { "_index": "my-index" } }
{ "field": "value" }
{ "delete": { "_index": "my-index", "_id": "1" } }
```

### Search

```bash
# Match all
GET /my-index/_search
{ "query": { "match_all": {} }, "size": 10 }

# Full-text search
GET /my-index/_search
{ "query": { "match": { "message": "error timeout" } } }

# Exact match
GET /my-index/_search
{ "query": { "term": { "level": "error" } } }

# Multiple values
GET /my-index/_search
{ "query": { "terms": { "level": ["error", "warn"] } } }

# Range
GET /my-index/_search
{ "query": { "range": { "@timestamp": { "gte": "now-1h" } } } }

# Bool query
GET /my-index/_search
{
  "query": {
    "bool": {
      "must":     [ { "match": { "message": "error" } } ],
      "filter":   [ { "term":  { "service": "api" } } ],
      "must_not": [ { "term":  { "level": "debug" } } ],
      "should":   [ { "term":  { "level": "error" } } ]
    }
  }
}

# With sorting and pagination
GET /my-index/_search
{
  "query": { "match_all": {} },
  "sort": [ { "@timestamp": "desc" } ],
  "from": 0,
  "size": 20,
  "_source": ["@timestamp", "level", "message"]
}
```

### Aggregations

```bash
# Count by field
GET /my-index/_search
{
  "size": 0,
  "aggs": {
    "by_level": { "terms": { "field": "level", "size": 10 } }
  }
}

# Date histogram
GET /my-index/_search
{
  "size": 0,
  "aggs": {
    "over_time": {
      "date_histogram": { "field": "@timestamp", "calendar_interval": "1h" }
    }
  }
}

# Metrics
GET /my-index/_search
{
  "size": 0,
  "aggs": {
    "avg_rt": { "avg": { "field": "response_time" } },
    "max_rt": { "max": { "field": "response_time" } },
    "p99_rt": { "percentiles": { "field": "response_time", "percents": [95, 99] } }
  }
}
```

---

## Security

```bash
# Users
GET _security/user
POST _security/user/myuser
{ "password": "pass", "roles": ["viewer"] }
DELETE _security/user/myuser

# Roles
GET _security/role
POST _security/role/my-role
{ "indices": [{ "names": ["logs-*"], "privileges": ["read"] }] }

# API Keys
POST _security/api_key
{ "name": "my-key", "expiration": "30d" }

GET _security/api_key?name=my-key
DELETE _security/api_key
{ "name": "my-key" }
```

---

## ILM

```bash
# Policies
GET _ilm/policy
PUT _ilm/policy/my-policy
{ "policy": { "phases": { "hot": { ... }, "delete": { "min_age": "30d", "actions": { "delete": {} } } } } }

# Index ILM status
GET /my-index/_ilm/explain

# Move index to next phase
POST /my-index/_ilm/move
{ "current_step": { "phase": "hot", "action": "rollover", "name": "check-rollover-ready" }, "next_step": { "phase": "warm" } }
```

---

## Index Templates

```bash
GET _index_template
PUT _index_template/my-template
{
  "index_patterns": ["logs-*"],
  "data_stream": {},
  "template": {
    "settings": { "number_of_shards": 1 },
    "mappings": { "properties": { "@timestamp": { "type": "date" } } }
  }
}
DELETE _index_template/my-template
```

---

## Snapshots

```bash
# Register repo
PUT _snapshot/my-repo
{ "type": "fs", "settings": { "location": "/mnt/backups" } }

# Create snapshot
PUT _snapshot/my-repo/snap-1
{ "indices": "logs-*", "ignore_unavailable": true }

# List snapshots
GET _snapshot/my-repo/_all

# Restore
POST _snapshot/my-repo/snap-1/_restore
{ "indices": "logs-000001" }

# Delete
DELETE _snapshot/my-repo/snap-1
```

---

## Logstash

```bash
# Test config
logstash -f pipeline.conf --config.test_and_exit

# Run
logstash -f pipeline.conf

# Hot reload
logstash -f conf.d/ --config.reload.automatic

# API
curl http://localhost:9600/_node
curl http://localhost:9600/_node/stats/pipelines
curl http://localhost:9600/_node/stats/jvm
```

### Grok Patterns

```
%{IP:client_ip}
%{NUMBER:status_code:int}
%{WORD:http_method}
%{URIPATH:request_path}
%{QUOTEDSTRING:user_agent}
%{TIMESTAMP_ISO8601:timestamp}
%{COMBINEDAPACHELOG}                    # Full Apache/Nginx access log
%{SYSLOGLINE}                           # Syslog format
%{GREEDYDATA:rest}                      # Match everything remaining
```

---

## Filebeat

```bash
# Test
filebeat test config -e
filebeat test output

# Run
filebeat -e

# Setup dashboards
filebeat setup --dashboards

# Modules
filebeat modules list
filebeat modules enable nginx mysql
filebeat modules disable nginx
```

### filebeat.yml Quick Reference

```yaml
filebeat.inputs:
  - type: filestream
    paths: ["/var/log/*.log"]

output.elasticsearch:
  hosts: ["http://localhost:9200"]
  username: "elastic"
  password: "changeme"

# OR
output.logstash:
  hosts: ["localhost:5044"]

processors:
  - add_host_metadata: ~
  - add_cloud_metadata: ~
```

---

## Kibana Query Language (KQL)

```kql
level: "error"                    # Exact match
message: *timeout*                # Wildcard
response_time > 1.5               # Numeric range
NOT level: "info"                 # Negation
level: "error" AND service: "api" # AND
level: "error" OR level: "warn"   # OR
(level: "error" OR level: "warn") AND service: "api"  # Grouped
@timestamp > "2024-03-01"         # Date filter
error_code: *                     # Field exists
```

---

## Docker Quick Start

```bash
# Create network
docker network create elk

# Elasticsearch
docker run -d --name elasticsearch --net elk \
  -p 9200:9200 \
  -e "discovery.type=single-node" \
  -e "xpack.security.enabled=false" \
  docker.elastic.co/elasticsearch/elasticsearch:8.12.0

# Kibana
docker run -d --name kibana --net elk \
  -p 5601:5601 \
  -e "ELASTICSEARCH_HOSTS=http://elasticsearch:9200" \
  docker.elastic.co/kibana/kibana:8.12.0

# Logstash
docker run -d --name logstash --net elk \
  -v $(pwd)/pipeline:/usr/share/logstash/pipeline \
  docker.elastic.co/logstash/logstash:8.12.0

# Filebeat
docker run -d --name filebeat --net elk \
  --user root \
  -v $(pwd)/filebeat.yml:/usr/share/filebeat/filebeat.yml \
  -v /var/log:/var/log:ro \
  docker.elastic.co/beats/filebeat:8.12.0
```

---

## Default Ports

| Service | Port | Protocol |
|---------|------|---------|
| Elasticsearch HTTP | 9200 | HTTP/HTTPS |
| Elasticsearch Transport | 9300 | TCP |
| Kibana | 5601 | HTTP/HTTPS |
| Logstash Beats input | 5044 | TCP |
| Logstash HTTP input | 8080 | HTTP |
| Logstash Monitoring API | 9600 | HTTP |

---

## Troubleshooting Quick Reference

```bash
# Cluster not green
GET _cluster/health
GET _cluster/allocation/explain

# Disk space
GET _cat/allocation?v
GET _cat/indices?v&h=index,store.size&s=store.size:desc

# Performance
GET _cat/nodes?v&h=name,heap.percent,cpu,load_1m
GET _nodes/hot_threads

# Tasks
GET _tasks?detailed=true

# Pending cluster tasks
GET _cluster/pending_tasks

# Index recovery
GET /my-index/_recovery?pretty

# Slow logs
PUT /my-index/_settings
{ "index.search.slowlog.threshold.query.warn": "5s" }
```
