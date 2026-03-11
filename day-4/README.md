# Day 4: Logstash

## What is Logstash?

Logstash is a **server-side data processing pipeline** that ingests data from multiple sources, transforms it, and then sends it to a destination (usually Elasticsearch).

```
┌───────────────────────────────────────────────────────────────┐
│                    LOGSTASH PIPELINE                          │
│                                                               │
│   ┌──────────┐    ┌──────────┐    ┌──────────┐               │
│   │  INPUT   │───▶│  FILTER  │───▶│  OUTPUT  │               │
│   └──────────┘    └──────────┘    └──────────┘               │
│                                                               │
│   Where does       Transform,      Where does                 │
│   data come        parse, enrich   data go?                   │
│   from?            data                                       │
└───────────────────────────────────────────────────────────────┘
```

---

## Logstash Configuration

Logstash config files use a custom DSL and are placed in `/etc/logstash/conf.d/`.

### Basic Structure

```ruby
input {
  # Where to read data from
}

filter {
  # How to transform data
}

output {
  # Where to send data
}
```

---

## Input Plugins

### File Input

```ruby
input {
  file {
    path => "/var/log/nginx/access.log"
    start_position => "beginning"
    sincedb_path => "/dev/null"
    type => "nginx"
  }
}
```

### Beats Input (receive from Filebeat)

```ruby
input {
  beats {
    port => 5044
  }
}
```

### Stdin (for testing)

```ruby
input {
  stdin {
    codec => "line"
  }
}
```

### Syslog Input

```ruby
input {
  syslog {
    port => 514
    type => "syslog"
  }
}
```

### HTTP Input

```ruby
input {
  http {
    port => 8080
    codec => "json"
  }
}
```

---

## Filter Plugins

### Grok — Parse unstructured text

Grok uses predefined patterns to extract fields from raw text.

```ruby
filter {
  grok {
    match => {
      "message" => "%{COMBINEDAPACHELOG}"
    }
  }
}
```

Custom pattern:
```ruby
filter {
  grok {
    match => {
      "message" => "%{TIMESTAMP_ISO8601:timestamp} %{LOGLEVEL:level} %{GREEDYDATA:log_message}"
    }
  }
}
```

Common Grok patterns:
| Pattern | Matches |
|---------|---------|
| `%{IP:client}` | IP address |
| `%{NUMBER:bytes}` | Integer number |
| `%{WORD:method}` | Single word |
| `%{TIMESTAMP_ISO8601:ts}` | ISO8601 timestamp |
| `%{URIPATH:path}` | URL path |
| `%{GREEDYDATA:rest}` | Everything remaining |

### Date — Parse timestamps

```ruby
filter {
  date {
    match => ["timestamp", "dd/MMM/yyyy:HH:mm:ss Z"]
    target => "@timestamp"
  }
}
```

### Mutate — Modify fields

```ruby
filter {
  mutate {
    rename  => { "host"  => "hostname" }
    add_field => { "env" => "production" }
    remove_field => ["message", "beat"]
    convert => { "bytes" => "integer" }
    uppercase => ["level"]
    lowercase => ["service"]
  }
}
```

### GeoIP — Enrich with location data

```ruby
filter {
  geoip {
    source => "client_ip"
    target => "geoip"
  }
}
```

### JSON — Parse JSON strings

```ruby
filter {
  json {
    source => "message"
    target => "parsed"
  }
}
```

### Conditionals

```ruby
filter {
  if [level] == "error" {
    mutate {
      add_tag => ["alert"]
    }
  }

  if [status] >= 500 {
    mutate {
      add_field => { "is_server_error" => true }
    }
  }

  if "nginx" in [tags] {
    grok {
      match => { "message" => "%{COMBINEDAPACHELOG}" }
    }
  }
}
```

---

## Output Plugins

### Elasticsearch Output

```ruby
output {
  elasticsearch {
    hosts => ["http://localhost:9200"]
    index => "logs-%{+YYYY.MM.dd}"
    # With authentication:
    # user => "elastic"
    # password => "changeme"
  }
}
```

### Stdout (for debugging)

```ruby
output {
  stdout {
    codec => rubydebug
  }
}
```

### File Output

```ruby
output {
  file {
    path => "/tmp/logstash-output.log"
    codec => "json_lines"
  }
}
```

### Multiple Outputs

```ruby
output {
  elasticsearch {
    hosts => ["http://localhost:9200"]
    index => "logs-%{+YYYY.MM.dd}"
  }

  if [level] == "error" {
    file {
      path => "/var/log/errors.log"
    }
  }
}
```

---

## Complete Example: Nginx Log Pipeline

```ruby
# /etc/logstash/conf.d/nginx.conf

input {
  file {
    path => "/var/log/nginx/access.log"
    start_position => "beginning"
    type => "nginx-access"
  }
}

filter {
  if [type] == "nginx-access" {
    grok {
      match => { "message" => "%{COMBINEDAPACHELOG}" }
    }

    date {
      match => ["timestamp", "dd/MMM/yyyy:HH:mm:ss Z"]
      target => "@timestamp"
      remove_field => ["timestamp"]
    }

    mutate {
      convert => {
        "bytes"    => "integer"
        "response" => "integer"
      }
      remove_field => ["host", "message"]
    }

    geoip {
      source => "clientip"
    }

    if "_grokparsefailure" in [tags] {
      drop {}
    }
  }
}

output {
  elasticsearch {
    hosts => ["http://localhost:9200"]
    index => "nginx-logs-%{+YYYY.MM.dd}"
  }
  stdout { codec => rubydebug }
}
```

---

## Running Logstash

```bash
# Test config
logstash -f /etc/logstash/conf.d/nginx.conf --config.test_and_exit

# Run with config
logstash -f /etc/logstash/conf.d/nginx.conf

# Run with hot reload
logstash -f /etc/logstash/conf.d/ --config.reload.automatic

# Docker
docker run -it --rm \
  -v $(pwd)/pipeline:/usr/share/logstash/pipeline \
  --net elk \
  docker.elastic.co/logstash/logstash:8.12.0
```

---

## Logstash Monitoring API

```bash
# Node info
curl http://localhost:9600/_node

# Pipeline stats
curl http://localhost:9600/_node/stats/pipelines

# JVM stats
curl http://localhost:9600/_node/stats/jvm
```

---

## Multiple Pipelines

```yaml
# /etc/logstash/pipelines.yml
- pipeline.id: nginx
  path.config: "/etc/logstash/conf.d/nginx.conf"
  pipeline.workers: 2

- pipeline.id: app-logs
  path.config: "/etc/logstash/conf.d/app.conf"
  pipeline.workers: 4
```

---

## Exercises

1. Write a Logstash config that reads from stdin and outputs to stdout (debugging mode)
2. Write a pipeline to parse this log line with Grok:
   ```
   2024-03-11 10:30:00 ERROR api Connection to database timed out after 30s
   ```
3. Add a date filter to set `@timestamp` from the parsed timestamp
4. Add a mutate filter to add a field `environment: "production"`
5. Write a full pipeline that reads an Nginx access log and sends to Elasticsearch

---

## Summary

- Logstash pipeline: **Input → Filter → Output**
- **Grok** is the most important filter — it parses raw text into structured fields
- Always test your config with `--config.test_and_exit`
- Use **stdout with rubydebug codec** for debugging
- **Conditionals** let you apply filters selectively based on field values

---

**Previous:** [Day 3 - Queries & Mappings](../day-3/README.md) | **Next:** [Day 5 - Kibana](../day-5/README.md)
