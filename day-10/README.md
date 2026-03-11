# Day 10: Troubleshooting & Best Practices

## Common Elasticsearch Issues

### Cluster Health is Red or Yellow

```bash
# Check cluster health
GET _cluster/health?pretty

# Find unassigned shards
GET _cat/shards?h=index,shard,prirep,state,unassigned.reason&s=state

# Get allocation explanation
GET _cluster/allocation/explain
{
  "index": "my-index",
  "shard": 0,
  "primary": true
}
```

Common causes:
| Issue | Fix |
|-------|-----|
| Disk too full | Free up disk space or add nodes |
| Not enough nodes for replicas | Reduce replica count or add nodes |
| Node left cluster | Investigate node, it will rejoin |
| Corrupted data | Restore from snapshot |

```json
// Temporary fix: reduce replicas to 0 (not for production)
PUT /my-index/_settings
{
  "number_of_replicas": 0
}

// Re-enable allocation (if disabled)
PUT _cluster/settings
{
  "persistent": {
    "cluster.routing.allocation.enable": "all"
  }
}
```

---

### Disk Watermark Exceeded

Elasticsearch stops allocating shards when disk is too full.

```bash
# Check disk usage
GET _cat/allocation?v

# Check watermark settings
GET _cluster/settings?include_defaults=true&filter_path=*.cluster.routing.allocation.disk
```

```json
// Temporarily raise watermarks (NOT for permanent use)
PUT _cluster/settings
{
  "transient": {
    "cluster.routing.allocation.disk.watermark.low":  "95%",
    "cluster.routing.allocation.disk.watermark.high": "97%",
    "cluster.routing.allocation.disk.watermark.flood_stage": "99%"
  }
}
```

---

### High JVM Heap Usage

Elasticsearch uses the JVM. High heap = slow GC = slow cluster.

```bash
# Check JVM heap
GET _nodes/stats/jvm?pretty
GET _cat/nodes?v&h=name,heap.percent,heap.current,heap.max,ram.percent
```

Causes and fixes:
- Too many shards → reduce shard count with ILM/rollover
- Field data cache too large → limit with `indices.fielddata.cache.size`
- Too many aggregations → use `keyword` fields for aggs instead of `text`
- Set heap to 50% of RAM (max 31GB):
  ```bash
  # jvm.options
  -Xms4g
  -Xmx4g
  ```

---

### Slow Queries

```bash
# Enable slow log (index level)
PUT /my-index/_settings
{
  "index.search.slowlog.threshold.query.warn":  "10s",
  "index.search.slowlog.threshold.query.info":  "5s",
  "index.search.slowlog.threshold.fetch.warn":  "1s",
  "index.search.slowlog.threshold.fetch.info":  "800ms"
}

# Enable slow log (cluster level)
PUT _cluster/settings
{
  "persistent": {
    "cluster.search.slowlog.threshold.query.warn": "10s"
  }
}
```

Query optimization tips:
- Use `filter` context instead of `query` where possible (no scoring, cached)
- Avoid `wildcard` on large indices
- Use `keyword` fields for exact-match queries
- Limit `from+size` pagination — use `search_after` for deep pagination
- Avoid script-based queries in hot paths

---

## Common Logstash Issues

### Logstash Not Parsing Logs

```bash
# Test config
logstash -f pipeline.conf --config.test_and_exit

# Run with debug output
logstash -f pipeline.conf --log.level=debug

# Test Grok pattern
# Use: https://grokdebug.herokuapp.com
# Or in Kibana: Dev Tools > Grok Debugger
```

### Logstash Falling Behind (high latency)

```bash
# Check pipeline stats
curl http://localhost:9600/_node/stats/pipelines | jq .

# Look for: pipeline.events.out vs pipeline.events.in
```

Fixes:
- Increase `pipeline.workers` in `logstash.yml`
- Increase `pipeline.batch.size`
- Add more Logstash nodes
- Use Kafka as a buffer between Beats and Logstash

---

## Common Filebeat Issues

### Filebeat Not Shipping Logs

```bash
# Test config
filebeat test config -e

# Test connectivity
filebeat test output

# Run with verbose logging
filebeat -e -d "*"

# Check registry (tracks file positions)
cat /var/lib/filebeat/registry/filebeat/log.json
```

Common fixes:
- Check file permissions (`filebeat` must be able to read the log file)
- Verify `path` in filebeat.yml matches actual log location
- Check that output host/port is correct and reachable
- Reset registry if files were rotated and Filebeat lost track

---

## Common Kibana Issues

### Kibana Can't Connect to Elasticsearch

```bash
# Check Kibana logs
journalctl -u kibana -f

# Verify Elasticsearch is reachable from Kibana host
curl -u elastic:changeme http://elasticsearch-host:9200
```

Common fixes:
- Verify `elasticsearch.hosts` in `kibana.yml`
- Check `kibana_system` user password
- Check network/firewall between Kibana and Elasticsearch
- Verify TLS certificates if using HTTPS

### Kibana Showing "No results"

- Check the time picker — logs may be outside the selected range
- Verify the data view (index pattern) matches the actual index name
- Check if data was indexed to a different index than expected

---

## Performance Best Practices

### Elasticsearch

```
┌────────────────────────────────────────────────────────────┐
│              ELASTICSEARCH BEST PRACTICES                  │
│                                                            │
│  Indexing Performance:                                     │
│  • Use bulk API (not individual documents)                │
│  • Set refresh_interval to 30s or more for heavy writes   │
│  • Disable replicas during bulk loads, re-enable after    │
│  • Use auto-generated document IDs (POST, not PUT)        │
│                                                            │
│  Search Performance:                                       │
│  • Use filter context for non-scoring queries             │
│  • Pre-warm frequently-used cache with warmers            │
│  • Limit result size (avoid returning millions of docs)   │
│  • Use date math for time-based index routing             │
│                                                            │
│  Cluster Stability:                                        │
│  • Keep heap at 50% of RAM, max 31GB                      │
│  • Use dedicated master nodes in production               │
│  • Monitor shard count (< 20 per GB of heap)             │
│  • Enable slow logs to identify problematic queries       │
└────────────────────────────────────────────────────────────┘
```

### Mapping Best Practices

```json
// Good mapping
{
  "mappings": {
    "dynamic": "strict",       // Reject unknown fields
    "properties": {
      "level":   { "type": "keyword" },
      "message": {
        "type": "text",
        "fields": {
          "raw": { "type": "keyword", "ignore_above": 256 }
        }
      },
      "@timestamp": { "type": "date" },
      "request_id": {
        "type": "keyword",
        "doc_values": false    // Disable if not aggregating
      }
    }
  }
}
```

---

## Monitoring the ELK Stack

### Built-in Stack Monitoring

1. Enable monitoring in `elasticsearch.yml`:
   ```yaml
   xpack.monitoring.collection.enabled: true
   ```
2. In Kibana: **Stack Management** > **Stack Monitoring**

### Key Metrics to Monitor

| Metric | Warning Threshold |
|--------|------------------|
| JVM heap used | > 75% |
| CPU usage | > 80% |
| Disk usage | > 80% |
| Index rate (docs/s) | Sudden drop |
| Search latency | > 500ms |
| Rejected threads | > 0 |
| Unassigned shards | > 0 |

---

## Useful Diagnostic Commands

```bash
# Full cluster diagnostics
GET _cluster/health?pretty
GET _cluster/stats?pretty
GET _nodes/stats?pretty
GET _cat/nodes?v&h=name,heap.percent,cpu,load_1m,disk.used_percent
GET _cat/indices?v&h=index,health,docs.count,store.size
GET _cat/shards?v&h=index,shard,prirep,state,docs,store,node

# Index diagnostics
GET /my-index/_stats
GET /my-index/_segments
GET /my-index/_recovery

# Task management
GET _tasks?detailed=true
GET _tasks?actions=*search*

# Pending tasks
GET _cluster/pending_tasks

# Hot threads (CPU profiling)
GET _nodes/hot_threads
```

---

## Production Checklist

Before going to production, verify:

- [ ] Security enabled (TLS + authentication)
- [ ] Dedicated master nodes (3 or 5)
- [ ] Heap set to 50% of RAM (max 31GB)
- [ ] ILM policy configured for log retention
- [ ] Snapshot/backup strategy in place
- [ ] Monitoring dashboards set up
- [ ] Alerting rules configured
- [ ] Index templates defined with explicit mappings
- [ ] `dynamic: "strict"` in mappings to prevent mapping explosion
- [ ] Log retention policy meets compliance requirements
- [ ] Firewall rules restrict Elasticsearch port (9200, 9300)

---

## Exercises

1. Simulate a cluster health issue by stopping an Elasticsearch node and investigate
2. Use `GET _cluster/allocation/explain` to diagnose unassigned shards
3. Enable slow query logging on an index and run a slow search
4. Check JVM heap usage and node stats
5. Identify which indices are using the most disk
6. Set up Stack Monitoring in Kibana

---

## Summary

| Issue | First Command to Run |
|-------|---------------------|
| Red/yellow health | `GET _cluster/allocation/explain` |
| Disk too full | `GET _cat/allocation?v` |
| High heap | `GET _cat/nodes?v&h=heap.percent` |
| Slow queries | Enable slow logs, check `_tasks` |
| Logstash lag | `curl :9600/_node/stats/pipelines` |
| Filebeat not working | `filebeat test config && filebeat test output` |
| Kibana down | Check `kibana.yml` and Elasticsearch connectivity |

---

**Previous:** [Day 9 - ELK on Kubernetes](../day-9/README.md)

---

**Congratulations! You've completed the ELK Stack learning path!**

Review the [ELK Cheat Sheet](../cheatsheet.md) for a quick reference to the most important commands and concepts.
