# Part 33: Logging Stack
## Steps 311-320: Centralized Logging ใน Kubernetes

---

## 📖 บทนำ

Centralized logging เป็นส่วนสำคัญของ observability ใน Kubernetes รวมถึง log collection, aggregation, storage และ analysis

---

## Step 311: Logging Architecture

```
Logging Architecture:
┌─────────────┐    ┌─────────────┐    ┌──────────────┐
│  Container  │    │  Fluentbit  │    │   Loki /     │
│    Logs     │───▶│  DaemonSet  │───▶│ Elasticsearch│
└─────────────┘    └─────────────┘    └──────▬───────┘
                                              │
                                      ┌──────▼───────┐
                                      │    Grafana    │
                                      └───────────────┘

Strategies:
1. Sidecar: log-shipper container ใน same pod
2. Node-level: DaemonSet ที่ collect จาก /var/log/containers
3. Application: เขียน logs ไป external service โดยตรง
```

---

## Step 312: Fluent Bit DaemonSet

```yaml
# fluent-bit-values.yaml
config:
  service: |
    [SERVICE]
        Daemon Off
        Flush 1
        Log_Level info
        HTTP_Server On
        HTTP_Listen 0.0.0.0
        HTTP_Port 2020

  inputs: |
    [INPUT]
        Name tail
        Path /var/log/containers/*.log
        multiline.parser docker, cri
        Tag kube.*
        Mem_Buf_Limit 50MB
        Skip_Long_Lines On
        DB /run/fluent-bit/flb_kube.db

  filters: |
    [FILTER]
        Name kubernetes
        Match kube.*
        Merge_Log On
        Keep_Log Off
        K8S-Logging.Parser On
        K8S-Logging.Exclude On

    [FILTER]
        Name modify
        Match kube.*
        Add cluster prod-sea
        Add environment production

    [FILTER]
        Name grep
        Match kube.*
        Exclude log level=debug

  outputs: |
    [OUTPUT]
        Name loki
        Match kube.*
        Host loki.monitoring.svc.cluster.local
        Port 3100
        Labels job=fluent-bit, namespace=$kubernetes['namespace_name'], pod=$kubernetes['pod_name']
        Auto_Kubernetes_Labels On

tolerations:
  - key: node-role.kubernetes.io/control-plane
    operator: Exists
    effect: NoSchedule

resources:
  requests:
    cpu: 50m
    memory: 64Mi
  limits:
    cpu: 200m
    memory: 256Mi
```

---

## Step 313: Loki

```yaml
# loki-values.yaml
loki:
  auth_enabled: false
  
  limits_config:
    retention_period: 744h
    ingestion_rate_mb: 16
    ingestion_burst_size_mb: 32
    max_streams_per_user: 10000
  
  storage:
    type: s3
    s3:
      endpoint: s3.ap-southeast-1.amazonaws.com
      region: ap-southeast-1
      bucketnames: my-loki-logs
  
  schemaConfig:
    configs:
      - from: "2024-01-01"
        store: tsdb
        object_store: s3
        schema: v13
        index:
          prefix: loki_index_
          period: 24h
  
  compactor:
    retention_enabled: true
    retention_delete_delay: 2h
    delete_request_store: s3
```

---

## Step 314: LogQL

```logql
# เบื้องต้น
{namespace="production"}
{namespace="production", app="my-app"}

# Filter
{namespace="production"} |= "ERROR"
{namespace="production"} |~ "timeout|connection refused"
{namespace="production"} != "healthcheck"

# JSON parsing
{namespace="production"} | json
{namespace="production"} | json | level="error"

# Metric queries
rate({namespace="production", app="my-app"}[5m])
rate({namespace="production"} |= "ERROR" [5m])
sum by (app) (count_over_time({namespace="production"}[5m]))

# Pattern extraction
{namespace="production"} 
  | pattern `<level> <_> <message>`
  | level = "ERROR"

# Logfmt
{namespace="production"} | logfmt | duration > 1s
```

---

## Step 315: Fluentd (EFK Stack)

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluentd-config
  namespace: logging
data:
  fluent.conf: |
    <source>
      @type tail
      path /var/log/containers/*.log
      pos_file /var/log/fluentd-containers.log.pos
      tag kubernetes.*
      <parse>
        @type json
        time_key time
      </parse>
    </source>
    
    <filter kubernetes.**>
      @type kubernetes_metadata
      kubernetes_url "https://#{ENV['KUBERNETES_SERVICE_HOST']}:#{ENV['KUBERNETES_SERVICE_PORT']}/api"
      ca_file /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
    </filter>
    
    <filter kubernetes.**>
      @type record_transformer
      <record>
        cluster_name "#{ENV['CLUSTER_NAME']}"
      </record>
    </filter>
    
    <match kubernetes.**>
      @type elasticsearch
      host "#{ENV['ELASTICSEARCH_HOST']}"
      port "#{ENV['ELASTICSEARCH_PORT']}"
      scheme https
      index_name kubernetes-%Y.%m.%d
      <buffer>
        @type file
        path /var/log/fluentd-buffers/kubernetes.buffer
        flush_interval 5s
        retry_max_interval 30
        chunk_limit_size 2M
      </buffer>
    </match>
```

---

## Step 316: Grafana Loki Integration

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-datasources
  namespace: monitoring
  labels:
    grafana_datasource: "1"
data:
  datasources.yaml: |
    apiVersion: 1
    datasources:
      - name: Loki
        type: loki
        url: http://loki:3100
        access: proxy
        jsonData:
          maxLines: 1000
          derivedFields:
            - datasourceUid: jaeger
              matcherRegex: "traceID=(\\w+)"
              name: TraceID
              url: "$${__value.raw}"
      
      - name: Prometheus
        type: prometheus
        url: http://prometheus-operated:9090
        isDefault: true
```

---

## Step 317: Structured Logging

```python
import logging
import json
import sys
from datetime import datetime

class JSONFormatter(logging.Formatter):
    def format(self, record):
        log_data = {
            "timestamp": datetime.utcnow().isoformat() + "Z",
            "level": record.levelname.lower(),
            "message": record.getMessage(),
            "logger": record.name,
            "module": record.module,
            "line": record.lineno,
        }
        if hasattr(record, 'request_id'):
            log_data['request_id'] = record.request_id
        if hasattr(record, 'duration_ms'):
            log_data['duration_ms'] = record.duration_ms
        if record.exc_info:
            log_data['exception'] = self.formatException(record.exc_info)
        return json.dumps(log_data)

def setup_logging():
    handler = logging.StreamHandler(sys.stdout)
    handler.setFormatter(JSONFormatter())
    root_logger = logging.getLogger()
    root_logger.addHandler(handler)
    root_logger.setLevel(logging.INFO)
    return root_logger
```

---

## Step 318: Log Retention

```yaml
# Elasticsearch ILM Policy
apiVersion: v1
kind: ConfigMap
metadata:
  name: elasticsearch-ilm-policy
data:
  policy.json: |
    {
      "policy": {
        "phases": {
          "hot": {
            "actions": {
              "rollover": {
                "max_primary_shard_size": "50gb",
                "max_age": "1d"
              }
            }
          },
          "warm": {
            "min_age": "7d",
            "actions": {
              "shrink": {"number_of_shards": 1},
              "forcemerge": {"max_num_segments": 1}
            }
          },
          "delete": {
            "min_age": "90d",
            "actions": {"delete": {}}
          }
        }
      }
    }
```

---

## Step 319: OpenTelemetry Collector

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: otel-collector-config
  namespace: monitoring
data:
  config.yaml: |
    receivers:
      otlp:
        protocols:
          grpc:
            endpoint: 0.0.0.0:4317
          http:
            endpoint: 0.0.0.0:4318
      
      filelog:
        include:
          - /var/log/containers/*.log
        operators:
          - type: json_parser

    processors:
      batch:
        send_batch_size: 10000
        timeout: 10s
      resource:
        attributes:
          - key: cluster
            value: "prod-sea"
            action: insert
      memory_limiter:
        check_interval: 1s
        limit_mib: 1000

    exporters:
      loki:
        endpoint: http://loki:3100/loki/api/v1/push
      jaeger:
        endpoint: http://jaeger-collector:14250
      prometheus:
        endpoint: "0.0.0.0:8889"

    service:
      pipelines:
        logs:
          receivers: [otlp, filelog]
          processors: [batch, resource, memory_limiter]
          exporters: [loki]
        traces:
          receivers: [otlp]
          processors: [batch, resource]
          exporters: [jaeger]
```

---

## Step 320: Workshop

```bash
# PLG Stack
# helm install loki grafana/loki-stack \
#   --namespace monitoring \
#   --set grafana.enabled=true \
#   --set prometheus.enabled=true

# Fluent Bit
# helm install fluent-bit fluent/fluent-bit \
#   --namespace logging --create-namespace \
#   -f fluent-bit-values.yaml

# LogQL query
# curl -G http://loki:3100/loki/api/v1/query \
#   --data-urlencode 'query={namespace="production"} |= "ERROR"' \
#   --data-urlencode 'limit=50'
```

---

## 📊 สรุป Part 33

| Component | หน้าที่ |
|-----------|----------|
| Fluent Bit | Lightweight log collector |
| Loki | Log storage + label index |
| LogQL | Loki query language |
| Elasticsearch | Full-text search storage |
| OpenTelemetry | Unified signals |

---

## 🔗 ต่อไป
- [Part 34: Distributed Tracing](./part-34-tracing.md)

---
*Part 33 | Steps 311-320 | ระดับสูง*
