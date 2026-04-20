# 🔍 ELK Stack — Complete Monitoring Guide
### Elasticsearch · Logstash · Kibana · Beats · KQL · EQL

> Full-stack log management and observability platform — from zero to production-ready deployment with parsing pipelines, dashboards, alerting, and index lifecycle management.

---

## 📋 Table of Contents

- [Architecture Overview](#architecture-overview)
- [Prerequisites](#prerequisites)
- [Quick Start — Docker Compose](#quick-start--docker-compose)
- [Component Details](#component-details)
- [Logstash Pipeline Reference](#logstash-pipeline-reference)
- [Beats Configuration](#beats-configuration)
- [Query Reference — KQL & EQL](#query-reference--kql--eql)
- [Elasticsearch DSL](#elasticsearch-dsl)
- [Kibana Dashboards & Alerting](#kibana-dashboards--alerting)
- [Index Lifecycle Management](#index-lifecycle-management)
- [Security — TLS & RBAC](#security--tls--rbac)
- [Kubernetes Deployment (ECK)](#kubernetes-deployment-eck)
- [Troubleshooting](#troubleshooting)
- [Production Checklist](#production-checklist)

---

## 🏗️ Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Your Infrastructure                         │
│                                                                     │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────────┐    │
│  │Filebeat  │   │Metricbeat│   │Heartbeat │   │ Elastic Agent│    │
│  │(logs)    │   │(metrics) │   │(uptime)  │   │ (unified)    │    │
│  └────┬─────┘   └────┬─────┘   └────┬─────┘   └──────┬───────┘    │
│       │              │              │                 │            │
└───────┼──────────────┼──────────────┼─────────────────┼────────────┘
        │              │              │                 │
        ▼              │              │                 │
  ┌──────────┐         │              │                 │
  │Logstash  │◄────────┘              │                 │
  │:5044     │   (complex parsing)    │                 │
  │ Input    │                        │                 │
  │ Filter   │                        │                 │
  │ Output   │                        │                 │
  └────┬─────┘                        │                 │
       │                              │                 │
       └──────────────────────────────┼─────────────────┘
                                      │
                                      ▼
                          ┌───────────────────────┐
                          │    Elasticsearch       │
                          │      :9200             │
                          │  ┌─────────────────┐  │
                          │  │  Index: logs-*  │  │
                          │  │  Index: metrics-│  │
                          │  │  Index: apm-*   │  │
                          │  └─────────────────┘  │
                          └──────────┬────────────┘
                                     │
                                     ▼
                          ┌───────────────────────┐
                          │       Kibana           │
                          │       :5601            │
                          │  Discover │ Dashboards │
                          │  Alerts   │ SIEM       │
                          └───────────────────────┘
```

### Two Common Data Pipelines

```
Pipeline A (Full Processing — with Logstash):
Beats → Logstash:5044 → parse/enrich/filter → Elasticsearch:9200 → Kibana

Pipeline B (Direct — lightweight):
Beats → Elasticsearch:9200 (with Ingest Pipelines) → Kibana
```

> **Use Pipeline A** when you need complex parsing, enrichment (GeoIP, threat intel), or routing to multiple outputs.
> **Use Pipeline B** when forwarding simple structured logs to save resources.

---

## ✅ Prerequisites

| Requirement | Detail |
|-------------|--------|
| Docker + Compose | `sudo apt install -y docker.io docker-compose-plugin` |
| RAM | 4GB minimum. Elasticsearch needs 2GB heap alone. |
| Disk | `50GB+` — logs accumulate fast. Plan for volume × retention × 1.5 |
| OS setting | `vm.max_map_count=262144` — **required** or Elasticsearch refuses to start |

```bash
# Required before starting Elasticsearch — run this first
sudo sysctl -w vm.max_map_count=262144

# Make it permanent
echo 'vm.max_map_count=262144' | sudo tee -a /etc/sysctl.conf
```

---

## 🚀 Quick Start — Docker Compose

### 1. Clone and configure

```bash
git clone <this-repo>
cd elk-stack

# Set your passwords in .env
cp .env.example .env
```

### 2. Start the full stack

```bash
docker compose up -d

# Watch Elasticsearch start (takes ~60 seconds)
docker compose logs -f elasticsearch

# Verify all containers are healthy
docker compose ps
```

### 3. Verify Elasticsearch is up

```bash
curl -u elastic:changeme http://localhost:9200/_cluster/health?pretty
# "status" should be "green" or "yellow" (yellow = single node, no replicas)
```

### 4. Access Kibana

Open `http://YOUR-IP:5601` → login: `elastic` / `changeme`

---

## 📦 Component Details

### Port Reference

| Service | Port | Protocol | Purpose |
|---------|------|----------|---------|
| Elasticsearch | `9200` | HTTP | REST API — indexing, searching, management |
| Elasticsearch | `9300` | TCP | Inter-node transport |
| Kibana | `5601` | HTTP | Web UI + REST API |
| Logstash | `5044` | Beats | Beats input (Lumberjack protocol) |
| Logstash | `9600` | HTTP | Logstash monitoring API |
| APM Server | `8200` | HTTP | APM agent data ingestion |
| Fleet Server | `8220` | HTTP | Elastic Agent management |

### Elasticsearch Concepts

| Concept | Explanation |
|---------|-------------|
| Index | Container for documents. Like a database table. Named: `app-logs-2026.04.20` |
| Document | A JSON object in an index. Has a unique `_id`. |
| Shard | Physical Lucene index. An index splits into N shards across nodes for scale. |
| Replica | Copy of a shard on a different node. High availability + read scaling. |
| Mapping | Schema: defines field types — `keyword`, `text`, `date`, `integer`, `geo_point` |
| Inverted Index | Lucene structure: word → list of documents. Makes full-text search fast. |

---

## 🐳 Docker Compose Configuration

### `docker-compose.yml`

```yaml
version: '3.8'

services:

  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.13.0
    container_name: elasticsearch
    environment:
      - node.name=es01
      - cluster.name=elk-demo
      - discovery.type=single-node
      - ELASTIC_PASSWORD=changeme
      - xpack.security.enabled=true
      - xpack.security.http.ssl.enabled=false   # disable for demo
      - ES_JAVA_OPTS=-Xms1g -Xmx1g
    volumes:
      - esdata:/usr/share/elasticsearch/data
    ports:
      - '9200:9200'
    healthcheck:
      test: ['CMD-SHELL', 'curl -s -u elastic:changeme http://localhost:9200/_cluster/health | grep -q -e green -e yellow']
      interval: 30s
      retries: 10
    networks: [elk]

  kibana:
    image: docker.elastic.co/kibana/kibana:8.13.0
    container_name: kibana
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
      - ELASTICSEARCH_USERNAME=kibana_system
      - ELASTICSEARCH_PASSWORD=changeme
    ports:
      - '5601:5601'
    depends_on:
      elasticsearch:
        condition: service_healthy
    networks: [elk]

  logstash:
    image: docker.elastic.co/logstash/logstash:8.13.0
    container_name: logstash
    volumes:
      - ./logstash/pipeline:/usr/share/logstash/pipeline
      - ./logstash/config/logstash.yml:/usr/share/logstash/config/logstash.yml
    ports:
      - '5044:5044'
      - '9600:9600'
    environment:
      - LS_JAVA_OPTS=-Xms512m -Xmx512m
    depends_on:
      elasticsearch:
        condition: service_healthy
    networks: [elk]

  filebeat:
    image: docker.elastic.co/beats/filebeat:8.13.0
    container_name: filebeat
    user: root
    volumes:
      - ./filebeat/filebeat.yml:/usr/share/filebeat/filebeat.yml:ro
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      - /var/log:/var/log:ro
      - filebeatdata:/usr/share/filebeat/data   # registry — tracks file positions
    depends_on: [logstash]
    networks: [elk]

volumes:
  esdata:
  filebeatdata:

networks:
  elk:
    driver: bridge
```

---

## 🔧 Logstash Pipeline Reference

### Pipeline Structure

```
INPUT → (persistent queue) → FILTER → OUTPUT
```

### `logstash/pipeline/main.conf` — Parse Nginx Logs

```ruby
input {
  beats {
    port => 5044
  }
}

filter {
  if [log][file][path] =~ /nginx/ {

    # Parse combined log format
    grok {
      match => { "message" => "%{COMBINEDAPACHELOG}" }
    }

    # Convert response code to integer
    mutate {
      convert => { "response" => "integer" }
    }

    # Add GeoIP from client IP
    geoip {
      source => "clientip"
      target => "geoip"
    }

    # Parse timestamp
    date {
      match => ["timestamp", "dd/MMM/yyyy:HH:mm:ss Z"]
      target => "@timestamp"
    }

    # Remove raw message after parsing
    mutate { remove_field => ["message"] }
  }

  # Drop health check noise
  if [request] == "/health" {
    drop {}
  }
}

output {
  elasticsearch {
    hosts    => ["http://elasticsearch:9200"]
    user     => "elastic"
    password => "changeme"
    index    => "nginx-logs-%{+YYYY.MM.dd}"
  }
}
```

### Common Logstash Filters

```ruby
# Rename fields
mutate { rename => { "host" => "hostname" } }

# Add fields
mutate { add_field => { "environment" => "production" } }

# Remove fields (save storage)
mutate { remove_field => ["agent", "ecs", "input"] }

# Convert types
mutate { convert => { "status_code" => "integer", "bytes" => "float" } }

# Uppercase/lowercase
mutate { uppercase => ["http_method"], lowercase => ["log_level"] }

# Drop events
if [request] =~ /^\/health/ { drop {} }

# Tag parsing failures
if "_grokparsefailure" in [tags] { drop {} }
```

> **Tip:** Test grok patterns at [grokdebugger.com](https://grokdebugger.com) before deploying.

---

## 📡 Beats Configuration

### Filebeat — `filebeat/filebeat.yml`

```yaml
filebeat.inputs:
  - type: log
    enabled: true
    paths:
      - /var/log/nginx/*.log
      - /var/log/app/*.log
    fields:
      env: production
      service: web
    fields_under_root: true
    multiline:                     # handle Java stack traces
      pattern: '^[[:space:]]'
      negate: false
      match: after

  - type: container                # Docker container logs
    paths:
      - /var/lib/docker/containers/*/*.log
    processors:
      - add_docker_metadata: ~     # adds container name, image, labels

output.logstash:
  hosts: ['logstash:5044']

processors:
  - add_host_metadata: ~           # hostname, IP, OS
  - add_cloud_metadata: ~          # cloud provider, region, instance type
```

### Metricbeat — `metricbeat/metricbeat.yml`

```yaml
metricbeat.modules:
  - module: system
    metricsets: [cpu, memory, network, diskio, filesystem]
    period: 15s
    cpu.metrics: [percentages, normalized_percentages]

  - module: docker
    metricsets: [container, cpu, memory, network]
    hosts: ['unix:///var/run/docker.sock']
    period: 15s

output.elasticsearch:
  hosts: ['http://elasticsearch:9200']
  username: elastic
  password: changeme

setup.kibana:
  host: 'http://kibana:5601'

setup.dashboards.enabled: true    # auto-import Metricbeat dashboards
```

### Filebeat useful commands

```bash
# Test config
filebeat test config -e

# Test connectivity to output
filebeat test output -e

# Run once (debug mode)
filebeat -e -d "*"
```

---

## 🔍 Query Reference — KQL & EQL

### KQL (Kibana Query Language) — used in Discover and Dashboards

```kql
# Free text
error
"connection refused"

# Field match
log.level: ERROR
http.response.status_code: 500
service.name: "payment-service"

# Comparison
http.response.status_code >= 400
event.duration < 1000

# Boolean
log.level: ERROR AND service.name: payment
log.level: ERROR OR log.level: CRITICAL
NOT http.response.status_code: 200
log.level: ERROR AND NOT service.name: healthcheck

# Wildcard
host.name: web-*
url.path: /api/*
message: *timeout*

# Field exists / does not exist
error.message: *
NOT error.message: *

# Range
http.response.status_code >= 500 AND http.response.status_code < 600

# Nested fields
host.os.type: linux
container.labels.app: nginx
```

### EQL (Event Query Language) — used in Security / SIEM

```eql
# Basic event match
process where process.name == "cmd.exe"
network where destination.port == 22
authentication where event.outcome == "failure"

# Multiple conditions
process where process.name == "powershell.exe"
  and process.args : "*-EncodedCommand*"
  and user.name != "SYSTEM"

# SEQUENCE — detect attack chains
# Failed login followed by success (brute force then access)
sequence by user.name with maxspan=5m
  [authentication where event.outcome == "failure"]
  [authentication where event.outcome == "success"]

# Process spawn then network (C2 beacon pattern)
sequence by host.id with maxspan=30s
  [process where event.type == "start" and process.name == "notepad.exe"]
  [network where destination.port in (80, 443, 8080)]

# 3 failed logins from same IP (brute force detection)
sequence by source.ip with maxspan=1m
  [authentication where event.outcome == "failure"] with runs=3

# Aggregation — top source IPs
network where true
| stats request_count = count() by source.ip
| sort request_count desc
| head 10
```

---

## 📊 Elasticsearch DSL

```json
// Match all
GET logs-*/_search
{ "query": { "match_all": {} }, "size": 10, "sort": [{ "@timestamp": { "order": "desc" } }] }

// Full text search
GET logs-*/_search
{ "query": { "match": { "message": "connection refused" } } }

// Bool query — combine AND/OR/NOT
GET logs-*/_search
{
  "query": {
    "bool": {
      "must":     [{ "match": { "log.level": "ERROR" } },
                   { "range": { "@timestamp": { "gte": "now-1h" } } }],
      "must_not": [{ "match": { "service.name": "healthcheck" } }],
      "should":   [{ "match": { "message": "timeout" } },
                   { "match": { "message": "refused" } }],
      "minimum_should_match": 1
    }
  }
}

// Aggregation — error count by service
GET logs-*/_search
{
  "size": 0,
  "query": { "match": { "log.level": "ERROR" } },
  "aggs": {
    "errors_by_service": {
      "terms": { "field": "service.name", "size": 10 }
    }
  }
}

// Date histogram — errors over time
GET logs-*/_search
{
  "size": 0,
  "aggs": {
    "over_time": {
      "date_histogram": { "field": "@timestamp", "fixed_interval": "1h" },
      "aggs": { "by_level": { "terms": { "field": "log.level" } } }
    }
  }
}
```

### Management Commands

```bash
# Cluster health
GET /_cluster/health?pretty

# List indices sorted by size
GET /_cat/indices?v&s=store.size:desc

# Node stats
GET /_cat/nodes?v

# Create index with mapping
PUT /app-logs
{
  "settings": { "number_of_shards": 1, "number_of_replicas": 0 },
  "mappings": {
    "properties": {
      "@timestamp":   { "type": "date" },
      "log.level":    { "type": "keyword" },
      "message":      { "type": "text" },
      "service.name": { "type": "keyword" },
      "http.response.status_code": { "type": "integer" },
      "geoip.location": { "type": "geo_point" }
    }
  }
}

# Delete old indices
DELETE /nginx-logs-2026.01.*
```

---

## 📈 Kibana Dashboards & Alerting

### Create Data View (Index Pattern)

1. Stack Management → Data Views → Create data view
2. Index pattern: `logs-*` (wildcard matches all daily indices)
3. Timestamp field: `@timestamp`
4. Save → go to Discover

### Key Panels to Build

| Panel | Visualization | KQL Filter | Field |
|-------|--------------|-----------|-------|
| Error Rate Over Time | Line chart (Lens) | `log.level: ERROR` | `@timestamp` + count |
| HTTP Status Distribution | Donut chart | `http.response.status_code: *` | `status_code` terms |
| Top 10 Errors | Data table | `log.level: ERROR` | `message.keyword` terms |
| Geo Map | Maps (geo_point) | none | `geoip.location` |
| Active Errors Count | Metric/Stat | `log.level: ERROR` | count |

### Kibana Alerting Rule (Threshold)

1. Stack Management → Rules → Create rule
2. Rule type: **Elasticsearch query**
3. Index: `logs-*`, Time field: `@timestamp`
4. KQL: `log.level: ERROR`
5. Threshold: count > 100 in last 5 minutes
6. Check every: 1 minute
7. Action → Webhook → POST with body:

```json
{
  "alert": "{{alertName}}",
  "value": "{{context.value}}",
  "threshold": "{{context.threshold}}",
  "timestamp": "{{date}}",
  "kibana_url": "{{kibanaBaseUrl}}/app/discover"
}
```

---

## 🔄 Index Lifecycle Management

```json
// Create ILM policy
PUT /_ilm/policy/logs-policy
{
  "policy": {
    "phases": {
      "hot": {
        "actions": {
          "rollover": {
            "max_size": "50gb",
            "max_age":  "1d",
            "max_docs": 5000000
          }
        }
      },
      "warm": {
        "min_age": "3d",
        "actions": {
          "forcemerge": { "max_num_segments": 1 },
          "shrink":     { "number_of_shards": 1 }
        }
      },
      "delete": {
        "min_age": "30d",
        "actions": { "delete": {} }
      }
    }
  }
}

// Apply via index template
PUT /_index_template/nginx-logs-template
{
  "index_patterns": ["nginx-logs-*"],
  "template": {
    "settings": {
      "number_of_shards": 1,
      "number_of_replicas": 1,
      "index.lifecycle.name": "logs-policy",
      "index.lifecycle.rollover_alias": "nginx-logs"
    }
  }
}
```

| Phase | Trigger | Action |
|-------|---------|--------|
| Hot | Always | Active writes, rollover at 50GB / 1 day |
| Warm | 3 days old | Read-only, shrink, force merge |
| Delete | 30 days old | Delete index |

---

## 🔐 Security — TLS & RBAC

### Built-in Users

| User | Purpose |
|------|---------|
| `elastic` | Superuser — change password immediately |
| `kibana_system` | Used by Kibana to connect to ES |
| `logstash_system` | Used by Logstash monitoring |
| `beats_system` | Used by Beats monitoring |

```bash
# Reset elastic password
curl -X POST 'http://localhost:9200/_security/user/elastic/_password' \
  -H 'Content-Type: application/json' \
  -u elastic:current_password \
  -d '{ "password": "new_strong_password" }'
```

### Create Read-Only Role

```json
PUT /_security/role/log-reader
{
  "indices": [{
    "names": ["logs-*", "nginx-logs-*"],
    "privileges": ["read", "view_index_metadata"]
  }],
  "cluster": ["monitor"]
}

PUT /_security/user/alice
{
  "password": "secure_password",
  "roles": ["log-reader"],
  "full_name": "Alice Dev"
}
```

---

## ☸️ Kubernetes Deployment (ECK)

```bash
# Install ECK operator
kubectl create -f https://download.elastic.co/downloads/eck/2.12.1/crds.yaml
kubectl apply  -f https://download.elastic.co/downloads/eck/2.12.1/operator.yaml

# Watch operator logs
kubectl -n elastic-system logs -f statefulset.apps/elastic-operator
```

```yaml
# elasticsearch.yaml
apiVersion: elasticsearch.k8s.elastic.co/v1
kind: Elasticsearch
metadata:
  name: quickstart
spec:
  version: 8.13.0
  nodeSets:
  - name: default
    count: 1
    config:
      node.store.allow_mmap: false   # required for Kind/Docker
    volumeClaimTemplates:
    - metadata:
        name: elasticsearch-data
      spec:
        accessModes: [ReadWriteOnce]
        resources:
          requests:
            storage: 5Gi
---
# kibana.yaml
apiVersion: kibana.k8s.elastic.co/v1
kind: Kibana
metadata:
  name: quickstart
spec:
  version: 8.13.0
  count: 1
  elasticsearchRef:
    name: quickstart   # auto-wires to the ES cluster
```

```bash
# Apply
kubectl apply -f elasticsearch.yaml -f kibana.yaml

# Get elastic password
kubectl get secret quickstart-es-elastic-user \
  -o jsonpath='{.data.elastic}' | base64 --decode

# Access Kibana
kubectl port-forward svc/quickstart-kb-http 5601:5601
# Visit https://localhost:5601
```

---

## 🔧 Troubleshooting

| Symptom | Where to look | Fix |
|---------|--------------|-----|
| ES won't start | `docker logs elasticsearch` | `vm.max_map_count` too low — `sysctl -w vm.max_map_count=262144` |
| Yellow cluster health | `GET /_cluster/health?pretty` | Single node can't have replicas — `PUT /index/_settings {"number_of_replicas":0}` |
| `_grokparsefailure` tag | Discover — filter by tag | Test pattern at grokdebugger.com |
| Filebeat not shipping | `filebeat test output -e` | Check Logstash port 5044 reachable, check firewall |
| Kibana can't connect | `docker logs kibana` | Verify `ELASTICSEARCH_HOSTS` env var, ES health on 9200 |
| Index not in Kibana | Stack Management → Data Views | Create data view with matching pattern + `@timestamp` |
| High JVM heap | `GET /_nodes/stats/jvm` | Increase `-Xmx`, add nodes, enable ILM to delete old data |
| Slow queries | `GET /_nodes/hot_threads` | Too many shards — optimise mappings, use `keyword` not `text` for exact-match |

```bash
# Essential debug commands
GET /_cluster/health?pretty
GET /_cat/indices?v&s=store.size:desc
GET /_cat/nodes?v
GET /_nodes/stats/jvm,indices
GET /_cluster/pending_tasks
docker compose logs -f elasticsearch
docker compose logs -f logstash
filebeat test config -e
filebeat test output -e
```

---

## ✅ Production Checklist

### Elasticsearch
- [ ] `vm.max_map_count=262144` set permanently in `/etc/sysctl.conf`
- [ ] Heap: `-Xms` and `-Xmx` set to same value, max 50% RAM, never > 31GB
- [ ] Swap disabled: `sudo swapoff -a`
- [ ] Dedicated data volume mounted for `/var/lib/elasticsearch`
- [ ] `number_of_replicas >= 1` for all production indices
- [ ] ILM policy applied — delete phase configured
- [ ] All built-in user passwords changed

### Logstash
- [ ] Persistent queue enabled: `queue.type: persisted`
- [ ] Dead letter queue enabled: `dead_letter_queue.enable: true`
- [ ] Workers tuned to CPU cores: `pipeline.workers: <cpu_count>`

### Filebeat
- [ ] Registry directory (`/var/lib/filebeat/registry`) **never deleted**
- [ ] `close_inactive` set to release file handles
- [ ] Config tested: `filebeat test config -e`

### Security
- [ ] TLS enabled on HTTP and transport layers
- [ ] Kibana behind reverse proxy (Nginx/Traefik) with TLS
- [ ] RBAC roles created — no apps using `elastic` superuser

### Observability
- [ ] Stack Monitoring enabled (Kibana → Stack Monitoring)
- [ ] Alerts created for: error rate spike, disk usage > 80%, JVM heap > 85%
- [ ] Dashboards exported to JSON and committed to Git

---

## 📁 Repository Structure

```
.
├── README.md
├── docker-compose.yml
├── .env.example
├── elasticsearch/
│   └── elasticsearch.yml
├── logstash/
│   ├── config/
│   │   └── logstash.yml
│   └── pipeline/
│       ├── nginx.conf
│       └── app-logs.conf
├── filebeat/
│   └── filebeat.yml
├── metricbeat/
│   └── metricbeat.yml
└── kibana/
    └── dashboards/
        └── nginx-dashboard.json
```

---

## 🔗 References

- [Elastic Stack documentation](https://www.elastic.co/guide/index.html)
- [Elasticsearch DSL reference](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl.html)
- [KQL reference](https://www.elastic.co/guide/en/kibana/current/kuery-query.html)
- [EQL reference](https://www.elastic.co/guide/en/elasticsearch/reference/current/eql.html)
- [Logstash filter plugins](https://www.elastic.co/guide/en/logstash/current/filter-plugins.html)
- [ECK operator](https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-quickstart.html)
- [Grok Debugger](https://grokdebugger.com)
- [ILM documentation](https://www.elastic.co/guide/en/elasticsearch/reference/current/index-lifecycle-management.html)

---

> Covers **ELK Stack 8.x** — security (TLS + authentication) is enabled by default from 8.0 onwards.
