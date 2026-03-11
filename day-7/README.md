# Day 7: Index Management & ILM

## Why Index Management Matters

In production, log data grows continuously. Without a plan, you'll run out of disk space and your cluster will slow down. **Index Lifecycle Management (ILM)** automates the transition of indices through phases to control cost and performance.

```
┌────────────────────────────────────────────────────────────────┐
│                  INDEX LIFECYCLE PHASES                        │
│                                                                │
│  Hot Phase        Warm Phase     Cold Phase     Delete Phase   │
│  ┌──────────┐    ┌──────────┐   ┌──────────┐   ┌──────────┐   │
│  │ Active   │───▶│ Read-    │──▶│ Rarely   │──▶│ Index    │   │
│  │ writes & │    │ only,    │   │ accessed │   │ deleted  │   │
│  │ reads    │    │ optimized│   │ (frozen) │   │          │   │
│  │          │    │          │   │          │   │          │   │
│  │ Fast SSD │    │ Normal   │   │ Cold     │   │          │   │
│  │ nodes    │    │ disk     │   │ storage  │   │          │   │
│  └──────────┘    └──────────┘   └──────────┘   └──────────┘   │
│                                                                │
│  Day 0-7         Day 7-30       Day 30-90       Day 90+        │
└────────────────────────────────────────────────────────────────┘
```

---

## ILM Phases

| Phase | Purpose | Typical Actions |
|-------|---------|-----------------|
| **Hot** | Active indexing and search | Rollover, set priority |
| **Warm** | Less frequent querying | Shrink, force merge, reduce replicas |
| **Cold** | Rarely accessed, read-only | Freeze, move to cold tier |
| **Frozen** | Very rarely accessed | Fully frozen, minimal resources |
| **Delete** | Remove old data | Delete index |

---

## Creating an ILM Policy

```json
PUT _ilm/policy/logs-policy
{
  "policy": {
    "phases": {
      "hot": {
        "min_age": "0ms",
        "actions": {
          "rollover": {
            "max_age": "7d",
            "max_size": "50gb",
            "max_docs": 10000000
          },
          "set_priority": {
            "priority": 100
          }
        }
      },
      "warm": {
        "min_age": "7d",
        "actions": {
          "shrink": {
            "number_of_shards": 1
          },
          "forcemerge": {
            "max_num_segments": 1
          },
          "set_priority": {
            "priority": 50
          }
        }
      },
      "cold": {
        "min_age": "30d",
        "actions": {
          "set_priority": {
            "priority": 0
          }
        }
      },
      "delete": {
        "min_age": "90d",
        "actions": {
          "delete": {}
        }
      }
    }
  }
}
```

---

## Rollover

**Rollover** creates a new index when the current one meets a condition (size, age, or document count). This prevents indices from growing too large.

```
logs-000001  →  logs-000002  →  logs-000003
(full)           (full)           (current)
```

Rollover requires a **write alias** pointing to the active index.

---

## Data Streams (Modern Approach)

Data streams are the modern way to manage time-series log data. They automatically handle rollover and use ILM policies behind the scenes.

```json
// 1. Create component template for mappings
PUT _component_template/logs-mappings
{
  "template": {
    "mappings": {
      "properties": {
        "@timestamp": { "type": "date" },
        "level":      { "type": "keyword" },
        "message":    { "type": "text" },
        "service":    { "type": "keyword" }
      }
    }
  }
}

// 2. Create component template for settings
PUT _component_template/logs-settings
{
  "template": {
    "settings": {
      "index.lifecycle.name": "logs-policy",
      "number_of_shards": 1,
      "number_of_replicas": 1
    }
  }
}

// 3. Create index template for the data stream
PUT _index_template/logs-template
{
  "index_patterns": ["logs-*"],
  "data_stream": {},
  "composed_of": ["logs-mappings", "logs-settings"],
  "priority": 500
}

// 4. Index documents — data stream is auto-created
POST logs-myapp/_doc
{
  "@timestamp": "2024-03-11T10:30:00Z",
  "level": "info",
  "message": "Server started",
  "service": "api"
}
```

---

## View ILM Status

```bash
# View all ILM policies
GET _ilm/policy

# View a specific policy
GET _ilm/policy/logs-policy

# Check ILM status of an index
GET /logs-000001/_ilm/explain

# View all data streams
GET _data_stream

# View data stream stats
GET _data_stream/logs-myapp/_stats
```

---

## Snapshots & Backups

Snapshots are point-in-time backups of your Elasticsearch indices.

### Register a Snapshot Repository

```json
PUT _snapshot/my-backups
{
  "type": "fs",
  "settings": {
    "location": "/mount/backups",
    "compress": true
  }
}
```

For cloud storage:
```json
PUT _snapshot/s3-backups
{
  "type": "s3",
  "settings": {
    "bucket": "my-es-backups",
    "region": "us-east-1"
  }
}
```

### Create a Snapshot

```json
PUT _snapshot/my-backups/snapshot-2024-03-11
{
  "indices": "logs-*",
  "ignore_unavailable": true,
  "include_global_state": false
}
```

### Restore a Snapshot

```json
POST _snapshot/my-backups/snapshot-2024-03-11/_restore
{
  "indices": "logs-000001",
  "rename_pattern": "(.+)",
  "rename_replacement": "restored-$1"
}
```

### Snapshot Lifecycle Management (SLM)

Automate snapshot creation:

```json
PUT _slm/policy/daily-snapshots
{
  "schedule": "0 30 1 * * ?",  // 1:30 AM daily
  "name": "<daily-snap-{now/d}>",
  "repository": "my-backups",
  "config": {
    "indices": ["logs-*"],
    "ignore_unavailable": true
  },
  "retention": {
    "expire_after": "30d",
    "min_count": 5,
    "max_count": 30
  }
}
```

---

## Index Templates

Reusable settings/mappings applied automatically to new indices.

```json
PUT _index_template/nginx-logs-template
{
  "index_patterns": ["nginx-logs-*"],
  "template": {
    "settings": {
      "number_of_shards": 1,
      "number_of_replicas": 1,
      "index.lifecycle.name": "logs-policy",
      "index.lifecycle.rollover_alias": "nginx-logs"
    },
    "mappings": {
      "properties": {
        "@timestamp":  { "type": "date" },
        "clientip":    { "type": "ip" },
        "status":      { "type": "integer" },
        "bytes":       { "type": "long" },
        "verb":        { "type": "keyword" },
        "request":     { "type": "keyword" },
        "geoip": {
          "properties": {
            "location": { "type": "geo_point" }
          }
        }
      }
    }
  },
  "priority": 200
}
```

---

## Shard Management Best Practices

```
┌────────────────────────────────────────────────────────┐
│             SHARD SIZING GUIDELINES                    │
│                                                        │
│  • Aim for 10-50 GB per shard                         │
│  • Avoid too many small shards (<1 GB)                │
│  • Max ~20 shards per GB of heap memory               │
│  • Use rollover to control shard size                 │
│  • Use shrink in warm phase to reduce shard count     │
└────────────────────────────────────────────────────────┘
```

```bash
# Check shard sizes
GET _cat/shards/logs-*?v&h=index,shard,prirep,state,docs,store

# Check number of shards per node
GET _cat/allocation?v
```

---

## Exercises

1. Create an ILM policy `app-logs-policy` that:
   - Hot: rollover at 1GB or 7 days
   - Warm: shrink to 1 shard after 7 days
   - Delete: after 30 days
2. Create an index template that uses this ILM policy for `app-logs-*`
3. Create a data stream for `logs-myservice`
4. Index 5 documents into the data stream
5. Check the ILM status of the backing index
6. Register a local filesystem snapshot repository and create a snapshot

---

## Summary

- **ILM** automates moving indices through Hot → Warm → Cold → Delete phases
- **Rollover** creates new indices when size/age/doc limits are reached
- **Data Streams** are the modern approach for time-series data
- **Snapshots** provide point-in-time backups for disaster recovery
- **SLM** automates snapshot creation on a schedule
- Keep shards between 10-50 GB for best performance

---

**Previous:** [Day 6 - Beats](../day-6/README.md) | **Next:** [Day 8 - Security & Authentication](../day-8/README.md)
