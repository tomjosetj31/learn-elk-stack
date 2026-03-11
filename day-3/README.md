# Day 3: Elasticsearch Queries & Mappings

## Query DSL Overview

Elasticsearch uses a **JSON-based Query DSL** (Domain Specific Language). Queries live inside a `query` block in your search request.

```
Search Request Structure:
┌─────────────────────────────────────┐
│ GET /index/_search                  │
│ {                                   │
│   "query": { ... },     ← filter   │
│   "sort": [ ... ],      ← ordering │
│   "from": 0,            ← offset   │
│   "size": 10,           ← limit    │
│   "aggs": { ... },      ← bucket   │
│   "_source": ["field1"] ← fields   │
│ }                                   │
└─────────────────────────────────────┘
```

---

## Query Types

### match_all — Return everything

```json
GET /logs/_search
{
  "query": { "match_all": {} }
}
```

### match — Full-text search (analyzed)

```json
GET /logs/_search
{
  "query": {
    "match": {
      "message": "connection timeout"
    }
  }
}
```

### match_phrase — Exact phrase match

```json
GET /logs/_search
{
  "query": {
    "match_phrase": {
      "message": "connection timeout"
    }
  }
}
```

### term — Exact value match (not analyzed)

Use for `keyword`, `boolean`, `number`, `date` fields:

```json
GET /logs/_search
{
  "query": {
    "term": {
      "level": "error"
    }
  }
}
```

### terms — Match any value in a list

```json
GET /logs/_search
{
  "query": {
    "terms": {
      "level": ["error", "warn"]
    }
  }
}
```

### range — Numeric or date range

```json
GET /logs/_search
{
  "query": {
    "range": {
      "@timestamp": {
        "gte": "2024-03-01",
        "lte": "2024-03-11"
      }
    }
  }
}
```

### wildcard — Pattern matching

```json
GET /logs/_search
{
  "query": {
    "wildcard": {
      "service": "api-*"
    }
  }
}
```

### exists — Check if field exists

```json
GET /logs/_search
{
  "query": {
    "exists": {
      "field": "error_code"
    }
  }
}
```

---

## Compound Queries

### bool — Combine multiple queries

```json
GET /logs/_search
{
  "query": {
    "bool": {
      "must": [
        { "match": { "message": "error" } }
      ],
      "filter": [
        { "term": { "service": "api" } },
        { "range": { "@timestamp": { "gte": "now-1h" } } }
      ],
      "must_not": [
        { "term": { "level": "info" } }
      ],
      "should": [
        { "term": { "level": "error" } },
        { "term": { "level": "warn" } }
      ]
    }
  }
}
```

| Clause | Behavior | Affects Score? |
|--------|----------|---------------|
| must | Document MUST match | Yes |
| filter | Document MUST match | No (cached) |
| must_not | Document MUST NOT match | No |
| should | Document SHOULD match (boosts score) | Yes |

---

## Sorting & Pagination

```json
GET /logs/_search
{
  "query": { "match_all": {} },
  "sort": [
    { "@timestamp": { "order": "desc" } },
    { "_score": { "order": "desc" } }
  ],
  "from": 0,
  "size": 20
}
```

---

## Mappings

A **mapping** defines how fields in a document are stored and indexed.

### View an Index Mapping

```bash
GET /logs/_mapping
```

### Common Field Types

| Type | Description | Example |
|------|-------------|---------|
| `text` | Full-text search, analyzed | `"message": "Server error"` |
| `keyword` | Exact match, sorting, aggregations | `"level": "error"` |
| `date` | Date/time values | `"@timestamp": "2024-03-11T10:00:00Z"` |
| `integer` / `long` | Whole numbers | `"status_code": 500` |
| `float` / `double` | Decimal numbers | `"response_time": 0.23` |
| `boolean` | true/false | `"is_error": true` |
| `ip` | IP addresses | `"client_ip": "192.168.1.1"` |
| `geo_point` | Lat/lon coordinates | `"location": {"lat": 40.7, "lon": -74.0}` |
| `object` | Nested JSON object | `"user": {"name": "tom", "role": "admin"}` |

### Create Index with Explicit Mapping

```json
PUT /logs-app
{
  "mappings": {
    "properties": {
      "@timestamp":    { "type": "date" },
      "level":         { "type": "keyword" },
      "message":       { "type": "text" },
      "service":       { "type": "keyword" },
      "response_time": { "type": "float" },
      "status_code":   { "type": "integer" },
      "client_ip":     { "type": "ip" },
      "user": {
        "properties": {
          "name": { "type": "keyword" },
          "id":   { "type": "integer" }
        }
      }
    }
  }
}
```

### Dynamic Mapping

Elasticsearch automatically infers field types if no mapping is defined. This is convenient but can lead to unexpected types — always define explicit mappings in production.

---

## Analyzers

Analyzers process `text` fields at index and search time.

```
Input: "The Quick Brown Fox"
         ↓
   Character Filter (strip HTML, etc.)
         ↓
   Tokenizer  → ["The", "Quick", "Brown", "Fox"]
         ↓
   Token Filters → lowercase, stop words, stemming
         ↓
   Tokens: ["quick", "brown", "fox"]
```

### Built-in Analyzers

| Analyzer | Description |
|----------|-------------|
| `standard` | Default. Lowercases, removes punctuation |
| `simple` | Splits on non-letter characters, lowercases |
| `whitespace` | Splits on whitespace only |
| `keyword` | No analysis — treats entire string as one token |
| `english` | Standard + English stop words + stemming |

### Test an Analyzer

```json
POST _analyze
{
  "analyzer": "standard",
  "text": "The Quick Brown Fox jumps over the lazy dog!"
}
```

### Custom Analyzer

```json
PUT /my-index
{
  "settings": {
    "analysis": {
      "analyzer": {
        "my_analyzer": {
          "type": "custom",
          "tokenizer": "standard",
          "filter": ["lowercase", "stop", "snowball"]
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "message": {
        "type": "text",
        "analyzer": "my_analyzer"
      }
    }
  }
}
```

---

## Aggregations

Aggregations allow you to compute statistics over your data (like SQL GROUP BY + COUNT/SUM/AVG).

### Terms Aggregation (group by field)

```json
GET /logs/_search
{
  "size": 0,
  "aggs": {
    "by_level": {
      "terms": {
        "field": "level",
        "size": 10
      }
    }
  }
}
```

Response:
```json
{
  "aggregations": {
    "by_level": {
      "buckets": [
        { "key": "info",  "doc_count": 1523 },
        { "key": "error", "doc_count": 342  },
        { "key": "warn",  "doc_count": 89   }
      ]
    }
  }
}
```

### Date Histogram

```json
GET /logs/_search
{
  "size": 0,
  "aggs": {
    "errors_over_time": {
      "date_histogram": {
        "field": "@timestamp",
        "calendar_interval": "1h"
      }
    }
  }
}
```

### Metric Aggregations

```json
GET /logs/_search
{
  "size": 0,
  "aggs": {
    "avg_response_time": { "avg":   { "field": "response_time" } },
    "max_response_time": { "max":   { "field": "response_time" } },
    "min_response_time": { "min":   { "field": "response_time" } },
    "total_requests":    { "value_count": { "field": "response_time" } }
  }
}
```

---

## Exercises

1. Create an index `server-logs` with explicit mappings for: `@timestamp` (date), `level` (keyword), `message` (text), `service` (keyword), `status_code` (integer), `response_time` (float)
2. Insert 10 sample log documents with varied levels and services
3. Query: find all `error` level logs from the last 24 hours
4. Query: find logs where message contains "timeout" AND service is "api"
5. Aggregation: count logs grouped by `level`
6. Aggregation: calculate average `response_time` per `service`
7. Test the `standard` analyzer with a sample log message

---

## Summary

- **match** = full-text search; **term** = exact match; **range** = numeric/date filters
- **bool** queries combine multiple conditions with must/filter/should/must_not
- **Mappings** define field types — always use explicit mappings in production
- **text** fields are analyzed (tokenized); **keyword** fields are not
- **Aggregations** are like SQL GROUP BY — great for dashboards

---

**Previous:** [Day 2 - Elasticsearch Basics](../day-2/README.md) | **Next:** [Day 4 - Logstash](../day-4/README.md)
