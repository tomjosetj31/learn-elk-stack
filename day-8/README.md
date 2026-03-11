# Day 8: Security & Authentication

## Why Security Matters

By default, older versions of Elasticsearch had no security. In Elasticsearch 8.x, **security is enabled by default**. Securing your ELK stack is critical to prevent unauthorized access to your logs and data.

```
┌───────────────────────────────────────────────────────────────┐
│                  ELK SECURITY LAYERS                          │
│                                                               │
│   1. Transport Layer Security (TLS/SSL)                       │
│      └── Encrypt communication between nodes and clients     │
│                                                               │
│   2. Authentication                                           │
│      └── Verify who is making the request                    │
│          (native users, LDAP, SAML, OIDC, API keys)         │
│                                                               │
│   3. Authorization (RBAC)                                     │
│      └── Control what authenticated users can do             │
│          (roles, role mappings)                              │
│                                                               │
│   4. Audit Logging                                            │
│      └── Track who did what and when                         │
└───────────────────────────────────────────────────────────────┘
```

---

## Built-in Users

Elasticsearch ships with built-in users:

| Username | Purpose |
|----------|---------|
| `elastic` | Superuser — full admin access |
| `kibana_system` | Kibana to connect to Elasticsearch |
| `logstash_system` | Logstash to connect to Elasticsearch |
| `beats_system` | Beats to connect to Elasticsearch |
| `apm_system` | APM server to connect to Elasticsearch |
| `remote_monitoring_user` | Metricbeat monitoring |

### Set Built-in User Passwords

```bash
# Interactive setup
elasticsearch-setup-passwords interactive

# Or auto-generate
elasticsearch-setup-passwords auto
```

---

## TLS Configuration

### Generate TLS Certificates

```bash
# Generate CA and node certificates using elasticsearch-certutil
elasticsearch-certutil ca
elasticsearch-certutil cert --ca elastic-stack-ca.p12

# Or generate PEM format
elasticsearch-certutil ca --pem
elasticsearch-certutil cert --ca-cert ca.crt --ca-key ca.key --pem
```

### elasticsearch.yml TLS Settings

```yaml
# Transport layer (node-to-node)
xpack.security.transport.ssl.enabled: true
xpack.security.transport.ssl.keystore.path: elastic-certificates.p12
xpack.security.transport.ssl.truststore.path: elastic-certificates.p12

# HTTP layer (client-to-node)
xpack.security.http.ssl.enabled: true
xpack.security.http.ssl.keystore.path: elasticsearch.p12
```

### kibana.yml TLS Settings

```yaml
elasticsearch.hosts: ["https://localhost:9200"]
elasticsearch.ssl.certificateAuthorities: ["/etc/kibana/certs/ca.crt"]
elasticsearch.username: "kibana_system"
elasticsearch.password: "your-password"

# Kibana server TLS
server.ssl.enabled: true
server.ssl.certificate: /etc/kibana/certs/kibana.crt
server.ssl.key: /etc/kibana/certs/kibana.key
```

---

## Role-Based Access Control (RBAC)

### Built-in Roles

| Role | Access Level |
|------|-------------|
| `superuser` | Full access to everything |
| `kibana_admin` | All Kibana features |
| `viewer` | Read-only in Kibana |
| `editor` | Read/write in Kibana |
| `monitoring_user` | View monitoring data |
| `beats_admin` | Manage Beats config |
| `logstash_admin` | Manage Logstash pipelines |

### Create a Custom Role

```json
POST _security/role/logs-reader
{
  "cluster": ["monitor"],
  "indices": [
    {
      "names": ["logs-*", "filebeat-*"],
      "privileges": ["read", "view_index_metadata"],
      "field_security": {
        "grant": ["@timestamp", "level", "message", "service"]
      }
    }
  ],
  "applications": [
    {
      "application": "kibana-.kibana",
      "privileges": ["feature_discover.read", "feature_dashboard.read"],
      "resources": ["*"]
    }
  ]
}
```

### Create a User

```json
POST _security/user/john
{
  "password": "secure-password-123!",
  "roles": ["logs-reader"],
  "full_name": "John Smith",
  "email": "john@example.com",
  "metadata": {
    "team": "ops"
  }
}
```

### View Users and Roles

```bash
GET _security/user
GET _security/user/john
GET _security/role
GET _security/role/logs-reader
```

---

## API Keys

API keys are a secure way to authenticate API calls without using usernames/passwords.

### Create an API Key

```json
POST _security/api_key
{
  "name": "filebeat-key",
  "expiration": "30d",
  "role_descriptors": {
    "filebeat-writer": {
      "cluster": ["monitor", "manage_index_templates"],
      "indices": [
        {
          "names": ["filebeat-*"],
          "privileges": ["create_index", "create", "index", "write"]
        }
      ]
    }
  }
}
```

Response:
```json
{
  "id": "abc123",
  "name": "filebeat-key",
  "api_key": "xyz789",
  "encoded": "base64encodedvalue=="
}
```

### Use an API Key

```bash
# In Authorization header
curl -H "Authorization: ApiKey base64encodedvalue==" \
  http://localhost:9200/_cat/indices
```

### Filebeat with API Key

```yaml
output.elasticsearch:
  hosts: ["https://localhost:9200"]
  api_key: "id:api_key_value"
```

---

## Document-Level Security

Restrict which documents a user can see:

```json
POST _security/role/sales-only
{
  "indices": [
    {
      "names": ["orders-*"],
      "privileges": ["read"],
      "query": {
        "term": { "department": "sales" }
      }
    }
  ]
}
```

## Field-Level Security

Restrict which fields a user can see:

```json
POST _security/role/limited-view
{
  "indices": [
    {
      "names": ["logs-*"],
      "privileges": ["read"],
      "field_security": {
        "grant": ["@timestamp", "level", "message"],
        "except": ["client_ip", "user_data"]
      }
    }
  ]
}
```

---

## Audit Logging

Track security events:

```yaml
# elasticsearch.yml
xpack.security.audit.enabled: true
xpack.security.audit.logfile.events.include:
  - authentication_success
  - authentication_failed
  - access_denied
  - connection_denied
```

---

## Enable Security (Docker)

```yaml
# docker-compose.yml
version: '3'
services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.12.0
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=true
      - xpack.security.http.ssl.enabled=false  # Use HTTP for simplicity in dev
      - ELASTIC_PASSWORD=changeme
    ports:
      - "9200:9200"

  kibana:
    image: docker.elastic.co/kibana/kibana:8.12.0
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
      - ELASTICSEARCH_USERNAME=kibana_system
      - ELASTICSEARCH_PASSWORD=changeme
    ports:
      - "5601:5601"
    depends_on:
      - elasticsearch
```

---

## Exercises

1. Enable security on your local Elasticsearch (set `ELASTIC_PASSWORD`)
2. Access Elasticsearch with credentials: `curl -u elastic:changeme http://localhost:9200`
3. Create a user `devuser` with password
4. Create a role `dev-logs-reader` that can only read `logs-*` indices
5. Assign the role to `devuser`
6. Verify the user can read logs but cannot create/delete indices
7. Generate an API key and use it to query Elasticsearch
8. List all users and roles in your cluster

---

## Summary

- Elasticsearch 8.x has security **enabled by default**
- **TLS** encrypts all communication (HTTP and transport layers)
- **RBAC** controls what users can do with roles and privileges
- **API keys** are preferred over passwords for service-to-service auth
- **Document/field-level security** provides fine-grained data access control
- **Audit logging** tracks all security events

---

**Previous:** [Day 7 - Index Management & ILM](../day-7/README.md) | **Next:** [Day 9 - ELK on Kubernetes](../day-9/README.md)
