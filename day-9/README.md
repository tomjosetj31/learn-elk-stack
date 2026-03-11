# Day 9: ELK on Kubernetes

## Running ELK on Kubernetes

There are two main approaches to running Elasticsearch on Kubernetes:

1. **ECK (Elastic Cloud on Kubernetes)** — Official Elastic operator (recommended)
2. **Helm charts** — Community-maintained, more manual control

```
┌────────────────────────────────────────────────────────────────┐
│                  ECK ARCHITECTURE                              │
│                                                                │
│  Kubernetes Cluster                                            │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                                                          │  │
│  │  ECK Operator Pod (controller)                          │  │
│  │       │                                                  │  │
│  │       ▼  Watches CRDs                                    │  │
│  │  ┌────────────┐   ┌────────────┐   ┌──────────────────┐  │  │
│  │  │Elasticsearch│  │  Kibana    │   │    Logstash      │  │  │
│  │  │StatefulSet │   │Deployment  │   │   Deployment     │  │  │
│  │  └────────────┘   └────────────┘   └──────────────────┘  │  │
│  │                                                          │  │
│  │  ┌────────────┐                                          │  │
│  │  │  Filebeat  │  (DaemonSet — runs on every node)        │  │
│  │  │ DaemonSet  │                                          │  │
│  │  └────────────┘                                          │  │
│  └──────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────┘
```

---

## ECK — Elastic Cloud on Kubernetes

### Install ECK Operator

```bash
# Install CRDs and operator
kubectl create -f https://download.elastic.co/downloads/eck/2.11.1/crds.yaml
kubectl apply -f https://download.elastic.co/downloads/eck/2.11.1/operator.yaml

# Monitor operator logs
kubectl -n elastic-system logs -f statefulset.apps/elastic-operator
```

### Deploy Elasticsearch

```yaml
# elasticsearch.yaml
apiVersion: elasticsearch.k8s.elastic.co/v1
kind: Elasticsearch
metadata:
  name: quickstart
  namespace: elastic
spec:
  version: 8.12.0
  nodeSets:
    - name: default
      count: 3
      config:
        node.store.allow_mmap: false
      podTemplate:
        spec:
          containers:
            - name: elasticsearch
              resources:
                requests:
                  memory: 2Gi
                  cpu: 500m
                limits:
                  memory: 2Gi
                  cpu: 2
      volumeClaimTemplates:
        - metadata:
            name: elasticsearch-data
          spec:
            accessModes:
              - ReadWriteOnce
            resources:
              requests:
                storage: 10Gi
            storageClassName: standard
```

```bash
kubectl apply -f elasticsearch.yaml

# Check status
kubectl get elasticsearch -n elastic
kubectl get pods -n elastic

# Get default elastic password
kubectl get secret quickstart-es-elastic-user \
  -n elastic -o jsonpath='{.data.elastic}' | base64 --decode
```

### Deploy Kibana

```yaml
# kibana.yaml
apiVersion: kibana.k8s.elastic.co/v1
kind: Kibana
metadata:
  name: quickstart
  namespace: elastic
spec:
  version: 8.12.0
  count: 1
  elasticsearchRef:
    name: quickstart    # Reference to the Elasticsearch resource
  podTemplate:
    spec:
      containers:
        - name: kibana
          resources:
            requests:
              memory: 1Gi
              cpu: 500m
            limits:
              memory: 1Gi
              cpu: 1
```

```bash
kubectl apply -f kibana.yaml

# Check status
kubectl get kibana -n elastic

# Port-forward to access Kibana locally
kubectl port-forward service/quickstart-kb-http 5601 -n elastic
# Open: https://localhost:5601
```

### Deploy Filebeat (DaemonSet)

```yaml
# filebeat.yaml
apiVersion: beat.k8s.elastic.co/v1beta1
kind: Beat
metadata:
  name: quickstart
  namespace: elastic
spec:
  type: filebeat
  version: 8.12.0
  elasticsearchRef:
    name: quickstart
  kibanaRef:
    name: quickstart
  config:
    filebeat.inputs:
      - type: container
        paths:
          - /var/log/containers/*.log
    processors:
      - add_kubernetes_metadata:
          host: ${NODE_NAME}
          matchers:
            - logs_path:
                logs_path: "/var/log/containers/"
  daemonSet:
    podTemplate:
      spec:
        serviceAccountName: filebeat
        automountServiceAccountToken: true
        hostNetwork: true
        dnsPolicy: ClusterFirstWithHostNet
        containers:
          - name: filebeat
            securityContext:
              runAsUser: 0
            volumeMounts:
              - name: varlogcontainers
                mountPath: /var/log/containers
              - name: varlogpods
                mountPath: /var/log/pods
              - name: varlibdockercontainers
                mountPath: /var/lib/docker/containers
        volumes:
          - name: varlogcontainers
            hostPath:
              path: /var/log/containers
          - name: varlogpods
            hostPath:
              path: /var/log/pods
          - name: varlibdockercontainers
            hostPath:
              path: /var/lib/docker/containers
```

### RBAC for Filebeat

```yaml
# filebeat-rbac.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: filebeat
  namespace: elastic
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: filebeat
rules:
  - apiGroups: [""]
    resources:
      - nodes
      - namespaces
      - events
      - pods
      - services
    verbs: ["get", "list", "watch"]
  - apiGroups: ["apps"]
    resources:
      - replicasets
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: filebeat
subjects:
  - kind: ServiceAccount
    name: filebeat
    namespace: elastic
roleRef:
  kind: ClusterRole
  name: filebeat
  apiGroup: rbac.authorization.k8s.io
```

---

## Helm Charts (Alternative)

```bash
# Add Elastic Helm repo
helm repo add elastic https://helm.elastic.co
helm repo update

# Install Elasticsearch
helm install elasticsearch elastic/elasticsearch \
  --namespace elastic \
  --create-namespace \
  --set replicas=1 \
  --set resources.requests.memory=512Mi \
  --set resources.limits.memory=1Gi

# Install Kibana
helm install kibana elastic/kibana \
  --namespace elastic \
  --set elasticsearchHosts=http://elasticsearch-master:9200

# Install Filebeat
helm install filebeat elastic/filebeat \
  --namespace elastic

# Check status
helm list -n elastic
kubectl get all -n elastic
```

---

## ECK Useful Commands

```bash
# Watch all Elastic resources
kubectl get elastic -n elastic

# Get all ECK resources
kubectl get elasticsearch,kibana,beat,logstash -n elastic

# Describe Elasticsearch cluster
kubectl describe elasticsearch quickstart -n elastic

# Get Elasticsearch logs
kubectl logs -f -n elastic \
  $(kubectl get pod -n elastic -l elasticsearch.k8s.elastic.co/cluster-name=quickstart \
    -o jsonpath='{.items[0].metadata.name}')

# Scale Elasticsearch
kubectl patch elasticsearch quickstart -n elastic \
  --type merge \
  --patch '{"spec":{"nodeSets":[{"name":"default","count":5}]}}'

# Get elastic user password
kubectl get secret quickstart-es-elastic-user \
  -n elastic -o go-template='{{.data.elastic | base64decode}}'
```

---

## Production Considerations

### Node Roles

```yaml
nodeSets:
  - name: master
    count: 3
    config:
      node.roles: ["master"]

  - name: data-hot
    count: 3
    config:
      node.roles: ["data_hot", "data_content", "ingest"]

  - name: data-warm
    count: 2
    config:
      node.roles: ["data_warm"]

  - name: coordinating
    count: 2
    config:
      node.roles: []
```

### Resource Recommendations

| Component | Minimum | Recommended |
|-----------|---------|-------------|
| Elasticsearch (each node) | 2GB RAM | 8-16GB RAM |
| Kibana | 1GB RAM | 2-4GB RAM |
| Logstash | 1GB RAM | 2-4GB RAM |
| Filebeat | 100MB RAM | 256MB RAM |

### Storage

```yaml
volumeClaimTemplates:
  - metadata:
      name: elasticsearch-data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: "fast-ssd"    # Use SSD for hot tier
      resources:
        requests:
          storage: 100Gi
```

---

## Exercises

1. Install ECK operator in your Kubernetes cluster (minikube or kind)
2. Deploy a single-node Elasticsearch cluster with ECK
3. Deploy Kibana and connect it to Elasticsearch
4. Port-forward Kibana and access it in your browser
5. Deploy Filebeat as a DaemonSet to collect container logs
6. Verify container logs appear in Kibana Discover

---

## Summary

- **ECK** is the official, recommended way to run Elastic Stack on Kubernetes
- ECK uses **Custom Resource Definitions (CRDs)** to manage Elastic components
- Filebeat runs as a **DaemonSet** to collect logs from every node
- **RBAC** is required for Filebeat to access Kubernetes metadata
- For production: separate node roles (master, data-hot, data-warm, coordinating)
- Use fast SSD storage for hot tier nodes

---

**Previous:** [Day 8 - Security & Authentication](../day-8/README.md) | **Next:** [Day 10 - Troubleshooting & Best Practices](../day-10/README.md)
