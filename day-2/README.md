# Day 2: Elasticsearch Basics

## What is Elasticsearch?

Elasticsearch is a **distributed, RESTful search and analytics engine**. It stores data as JSON documents and exposes a rich REST API for indexing, searching, and managing data.

```
┌──────────────────────────────────────────────────────────────┐
│                  ELASTICSEARCH CONCEPTS                       │
│                                                              │
│   SQL World          Elasticsearch World                     │
│   ──────────         ─────────────────                       │
│   Database      ←→   Index                                   │
│   Table         ←→   (removed in ES 7+, was "Type")         │
│   Row           ←→   Document                                │
│   Column        ←→   Field                                   │
│   Schema        ←→   Mapping                                 │
│   Primary Key   ←→   _id                                     │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## REST API Basics

Elasticsearch uses standard HTTP verbs:

| HTTP Verb | Action |
|-----------|--------|
| GET | Read data |
| POST | Create data (auto-generated ID) |
| PUT | Create/update data (specify ID) |
| DELETE | Delete data |

Base URL: `http://localhost:9200`

---

## Working with Indices

### Create an Index

```bash
PUT /my-index
```

With settings:
```json
PUT /my-index
{
  "settings": {
    "number_of_shards": 1,
    "number_of_replicas": 0
  }
}
```

### List All Indices

```bash
GET _cat/indices?v
```

### Delete an Index

```bash
DELETE /my-index
```

### Check Index Info

```bash
GET /my-index
```

---

## CRUD Operations

### Create a Document (POST — auto ID)

```json
POST /my-index/_doc
{
  "title": "Learning Elasticsearch",
  "author": "DevOps Engineer",
  "tags": ["elk", "search", "devops"],
  "published": true,
  "timestamp": "2024-03-11T10:00:00Z"
}
```

### Create a Document (PUT — specify ID)

```json
PUT /my-index/_doc/1
{
  "title": "Learning Elasticsearch",
  "author": "DevOps Engineer"
}
```

### Read a Document

```bash
GET /my-index/_doc/1
```

Response:
```json
{
  "_index": "my-index",
  "_id": "1",
  "_version": 1,
  "found": true,
  "_source": {
    "title": "Learning Elasticsearch",
    "author": "DevOps Engineer"
  }
}
```

### Update a Document (partial update)

```json
POST /my-index/_update/1
{
  "doc": {
    "author": "Senior DevOps Engineer"
  }
}
```

### Replace a Document (full replace)

```json
PUT /my-index/_doc/1
{
  "title": "Learning Elasticsearch",
  "author": "Senior DevOps Engineer",
  "updated": true
}
```

### Delete a Document

```bash
DELETE /my-index/_doc/1
```

---

## Bulk API

Index multiple documents in one request:

```json
POST _bulk
{ "index": { "_index": "logs", "_id": "1" } }
{ "level": "info", "message": "Server started", "service": "api" }
{ "index": { "_index": "logs", "_id": "2" } }
{ "level": "error", "message": "DB connection failed", "service": "api" }
{ "index": { "_index": "logs", "_id": "3" } }
{ "level": "warn", "message": "High memory usage", "service": "worker" }
```

---

## Basic Search

### Search All Documents

```json
GET /my-index/_search
{
  "query": {
    "match_all": {}
  }
}
```

### Simple Match Query

```json
GET /logs/_search
{
  "query": {
    "match": {
      "message": "connection failed"
    }
  }
}
```

### Search Response Structure

```json
{
  "took": 5,               // Time in milliseconds
  "timed_out": false,
  "hits": {
    "total": {
      "value": 3,          // Total matching documents
      "relation": "eq"
    },
    "max_score": 1.5,
    "hits": [              // Array of matching documents
      {
        "_index": "logs",
        "_id": "2",
        "_score": 1.5,
        "_source": {
          "level": "error",
          "message": "DB connection failed"
        }
      }
    ]
  }
}
```

---

## Shards & Replicas

```
┌────────────────────────────────────────────────────────┐
│                  3-Node Cluster                        │
│                                                        │
│  Node 1           Node 2           Node 3             │
│  ┌─────────┐      ┌─────────┐      ┌─────────┐        │
│  │ Shard 0 │      │ Shard 1 │      │ Shard 2 │        │
│  │(primary)│      │(primary)│      │(primary)│        │
│  ├─────────┤      ├─────────┤      ├─────────┤        │
│  │ Shard 1 │      │ Shard 2 │      │ Shard 0 │        │
│  │(replica)│      │(replica)│      │(replica)│        │
│  └─────────┘      └─────────┘      └─────────┘        │
└────────────────────────────────────────────────────────┘
```

- **Primary shard**: Holds original data, handles write operations
- **Replica shard**: Copy of primary, handles read operations + provides HA
- Default: 1 primary shard, 1 replica (ES 7+)

---

## Cluster Health

```bash
GET _cluster/health
```

| Status | Meaning |
|--------|---------|
| green | All shards assigned and healthy |
| yellow | All primaries assigned, some replicas missing |
| red | Some primary shards not assigned |

```bash
# Detailed node info
GET _cat/nodes?v

# Shard allocation
GET _cat/shards?v

# Cluster stats
GET _cluster/stats?pretty
```

---

## Index Templates

Define default settings/mappings for new indices matching a pattern:

```json
PUT _index_template/logs-template
{
  "index_patterns": ["logs-*"],
  "template": {
    "settings": {
      "number_of_shards": 1,
      "number_of_replicas": 1
    },
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
```

---

## Exercises

1. Create an index called `products` with 1 shard, 0 replicas
2. Add 5 product documents with fields: `name`, `price`, `category`, `in_stock`
3. Retrieve a document by ID
4. Update the price of one product
5. Delete one product
6. Use the Bulk API to add 3 more products in one request
7. Check the cluster health and node info

---

## Summary

- Elasticsearch stores data as JSON **documents** inside **indices**
- Full REST API: GET, POST, PUT, DELETE
- Documents have an `_id`, `_index`, `_score`, and `_source`
- **Shards** distribute data; **replicas** provide high availability
- Use **index templates** to auto-apply settings to new indices

---

**Previous:** [Day 1 - Introduction](../day-1/README.md) | **Next:** [Day 3 - Queries & Mappings](../day-3/README.md)
