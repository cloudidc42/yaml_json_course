# Part 15: Observability Stack YAML
## Steps 141-150: Prometheus, Grafana, Loki, Jaeger

---

## Step 141: Observability Stack Overview

```
Observability = Logs + Metrics + Traces

Components:
  Metrics:    Prometheus + Grafana
  Logs:       Loki + Promtail/Fluent Bit + Grafana
  Traces:     Jaeger/Tempo + OpenTelemetry
  Alerts:     Alertmanager
  Dashboards: Grafana

Install via kube-prometheus-stack Helm Chart (all-in-one):
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install monitoring prometheus-community/kube-prometheus-stack \
  -n monitoring --create-namespace \
  -f monitoring-values.yaml
```

---

## Step 142: Prometheus Configuration

```yaml
# monitoring-values.yaml สำหรับ kube-prometheus-stack
prometheus:
  prometheusSpec:
    retention: 30d
    retentionSize: "50GB"
    
    resources:
      requests:
        cpu: 500m
        memory: 2Gi
      limits:
        cpu: 2
        memory: 8Gi
    
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: fast-ssd
          resources:
            requests:
              storage: 100Gi
    
    additionalScrapeConfigs:
      - job_name: 'myapp'
        kubernetes_sd_configs:
          - role: pod
        relabel_configs:
          - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
            action: keep
            regex: "true"
          - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
            action: replace
            target_label: __metrics_path__
            regex: (.+)
          - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
            action: replace
            regex: ([^:]+)(?::\d+)?;(\d+)
            replacement: $1:$2
            target_label: __address__
    
    externalLabels:
      cluster: production
      environment: prod
    
    remoteWrite:
      - url: https://prometheus-remote.example.com/api/v1/write
        basicAuth:
          username:
            name: remote-write-auth
            key: username
          password:
            name: remote-write-auth
            key: password

alertmanager:
  alertmanagerSpec:
    storage:
      volumeClaimTemplate:
        spec:
          storageClassName: fast-ssd
          resources:
            requests:
              storage: 10Gi
    
    config:
      global:
        resolve_timeout: 5m
        slack_api_url: 'https://hooks.slack.com/services/xxx'
      
      route:
        group_by: ['alertname', 'cluster', 'service']
        group_wait: 30s
        group_interval: 5m
        repeat_interval: 4h
        receiver: 'slack-critical'
        routes:
          - match:
              severity: critical
            receiver: 'pagerduty'
          - match:
              severity: warning
            receiver: 'slack-warnings'
      
      receivers:
        - name: 'slack-critical'
          slack_configs:
            - channel: '#alerts-critical'
              title: '{{ template "slack.title" . }}'
              text: '{{ template "slack.text" . }}'
              color: '{{ if eq .Status "firing" }}danger{{ else }}good{{ end }}'
        
        - name: 'pagerduty'
          pagerduty_configs:
            - service_key: <pagerduty-key>
              description: '{{ range .Alerts }}{{ .Annotations.summary }}\n{{ end }}'
        
        - name: 'slack-warnings'
          slack_configs:
            - channel: '#alerts-warnings'

grafana:
  adminPassword: "change-me-in-production"
  persistence:
    enabled: true
    storageClassName: fast-ssd
    size: 10Gi
  
  grafana.ini:
    server:
      domain: grafana.example.com
      root_url: https://grafana.example.com
    auth.github:
      enabled: true
      client_id: <github-client-id>
      client_secret: <github-client-secret>
      allowed_organizations: my-org
  
  dashboardProviders:
    dashboardproviders.yaml:
      apiVersion: 1
      providers:
        - name: 'default'
          orgId: 1
          folder: ''
          type: file
          options:
            path: /var/lib/grafana/dashboards
```

---

## Step 143: PrometheusRule (Alerts)

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: myapp-alerts
  namespace: production
  labels:
    release: monitoring
spec:
  groups:
    - name: myapp.availability
      interval: 30s
      rules:
        - alert: MyAppDown
          expr: |
            up{job="myapp"} == 0
          for: 2m
          labels:
            severity: critical
            team: backend
          annotations:
            summary: "MyApp instance {{ $labels.instance }} is down"
            description: "MyApp has been unreachable for 2 minutes"
            runbook: "https://wiki.example.com/runbooks/myapp-down"
        
        - alert: MyAppHighErrorRate
          expr: |
            sum(rate(http_requests_total{job="myapp", status=~"5.."}[5m])) /
            sum(rate(http_requests_total{job="myapp"}[5m])) > 0.05
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "High error rate: {{ $value | humanizePercentage }}"
        
        - alert: MyAppHighLatency
          expr: |
            histogram_quantile(0.95, 
              sum(rate(http_request_duration_seconds_bucket{job="myapp"}[5m])) by (le)
            ) > 1.0
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "p95 latency is {{ $value }}s (threshold: 1s)"
    
    - name: myapp.resources
      rules:
        - alert: MyAppHighCPU
          expr: |
            sum(rate(container_cpu_usage_seconds_total{namespace="production", container="myapp"}[5m])) /
            sum(kube_pod_container_resource_limits{namespace="production", container="myapp", resource="cpu"}) > 0.8
          for: 10m
          labels:
            severity: warning
          annotations:
            summary: "CPU usage is at {{ $value | humanizePercentage }}"
        
        - alert: MyAppHighMemory
          expr: |
            sum(container_memory_usage_bytes{namespace="production", container="myapp"}) /
            sum(kube_pod_container_resource_limits{namespace="production", container="myapp", resource="memory"}) > 0.9
          for: 5m
          labels:
            severity: critical
          annotations:
            summary: "Memory usage is at {{ $value | humanizePercentage }}"
    
    - name: slo
      rules:
        - record: myapp:availability:rate5m
          expr: |
            1 - (
              sum(rate(http_requests_total{job="myapp", status=~"5.."}[5m])) /
              sum(rate(http_requests_total{job="myapp"}[5m]))
            )
        
        - alert: SLOBudgetBurning
          expr: |
            myapp:availability:rate5m < 0.999
          for: 1m
          labels:
            severity: critical
          annotations:
            summary: "SLO budget burning: {{ $value | humanizePercentage }} availability"
```

---

## Step 144: ServiceMonitor

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: myapp-monitor
  namespace: production
  labels:
    release: monitoring
spec:
  selector:
    matchLabels:
      app: myapp
  
  namespaceSelector:
    matchNames:
      - production
  
  endpoints:
    - port: metrics
      path: /metrics
      interval: 30s
      scrapeTimeout: 10s
      
      bearerTokenSecret:
        name: metrics-token
        key: token
      
      tlsConfig:
        insecureSkipVerify: false
        caFile: /etc/prometheus/certs/ca.crt
      
      metricRelabelings:
        - sourceLabels: [__name__]
          regex: 'go_.*'
          action: drop
        
        - sourceLabels: [__name__]
          regex: 'http_request_duration_seconds_bucket'
          action: keep

---
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: myapp-pod-monitor
  namespace: production
spec:
  selector:
    matchLabels:
      app: myapp
  podMetricsEndpoints:
    - port: metrics
      path: /metrics
      interval: 30s
```

---

## Step 145: Loki Stack (Logging)

```yaml
# loki-values.yaml
loki:
  enabled: true
  isDefault: true
  
  config:
    auth_enabled: false
    
    ingester:
      chunk_idle_period: 3m
      chunk_retain_period: 1m
      max_transfer_retries: 0
      chunk_encoding: snappy
    
    limits_config:
      enforce_metric_name: false
      reject_old_samples: true
      reject_old_samples_max_age: 168h
      max_query_length: 721h
      ingestion_rate_mb: 16
      ingestion_burst_size_mb: 24
    
    schema_config:
      configs:
        - from: 2020-10-24
          store: boltdb-shipper
          object_store: s3
          schema: v11
          index:
            prefix: index_
            period: 24h
    
    storage_config:
      boltdb_shipper:
        active_index_directory: /data/loki/index
        cache_location: /data/loki/boltdb-cache
        cache_ttl: 24h
        shared_store: s3
      
      aws:
        s3: s3://us-east-1/loki-logs
        s3forcepathstyle: false
    
    compactor:
      working_directory: /data/loki/boltdb-shipper-compactor
      shared_store: s3
      compaction_interval: 10m
      retention_enabled: true
      retention_delete_delay: 2h
    
    table_manager:
      retention_deletes_enabled: true
      retention_period: 720h

promtail:
  enabled: true
  config:
    clients:
      - url: http://loki:3100/loki/api/v1/push
    
    scrape_configs:
      - job_name: kubernetes-pods
        pipeline_stages:
          - cri: {}
          
          - json:
              expressions:
                level: level
                msg: message
                timestamp: time
          
          - labels:
              level:
          
          - match:
              selector: '{level="error"}'
              stages:
                - metrics:
                    error_total:
                      type: Counter
                      description: total errors
        
        kubernetes_sd_configs:
          - role: pod
        
        relabel_configs:
          - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
            action: drop
            regex: "false"
          - source_labels: [__meta_kubernetes_namespace]
            target_label: namespace
          - source_labels: [__meta_kubernetes_pod_name]
            target_label: pod
          - source_labels: [__meta_kubernetes_container_name]
            target_label: container
```

---

## Step 146: OpenTelemetry Collector

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: otel-collector
  namespace: monitoring
spec:
  replicas: 2
  selector:
    matchLabels:
      app: otel-collector
  template:
    spec:
      containers:
        - name: collector
          image: otel/opentelemetry-collector-contrib:latest
          ports:
            - containerPort: 4317   # OTLP gRPC
            - containerPort: 4318   # OTLP HTTP
            - containerPort: 9411   # Zipkin
            - containerPort: 14268  # Jaeger HTTP
          volumeMounts:
            - name: config
              mountPath: /etc/otel
      volumes:
        - name: config
          configMap:
            name: otel-config

---
apiVersion: v1
kind: ConfigMap
metadata:
  name: otel-config
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
      
      jaeger:
        protocols:
          thrift_http:
            endpoint: 0.0.0.0:14268
      
      zipkin:
        endpoint: 0.0.0.0:9411
    
    processors:
      batch:
        timeout: 1s
        send_batch_size: 1024
      
      memory_limiter:
        limit_mib: 512
        spike_limit_mib: 128
        check_interval: 5s
      
      k8sattributes:
        extract:
          metadata:
            - k8s.namespace.name
            - k8s.pod.name
            - k8s.deployment.name
        pod_association:
          - sources:
              - from: resource_attribute
                name: k8s.pod.ip
    
    exporters:
      jaeger:
        endpoint: jaeger-collector:14250
        tls:
          insecure: true
      
      prometheus:
        endpoint: "0.0.0.0:8889"
      
      loki:
        endpoint: http://loki:3100/loki/api/v1/push
    
    service:
      pipelines:
        traces:
          receivers: [otlp, jaeger, zipkin]
          processors: [memory_limiter, k8sattributes, batch]
          exporters: [jaeger]
        
        metrics:
          receivers: [otlp]
          processors: [batch]
          exporters: [prometheus]
        
        logs:
          receivers: [otlp]
          processors: [batch]
          exporters: [loki]
```

---

## Step 147: Jaeger Tracing

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jaeger
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: jaeger
  template:
    spec:
      containers:
        - name: jaeger
          image: jaegertracing/all-in-one:1.52
          env:
            - name: COLLECTOR_ZIPKIN_HOST_PORT
              value: ":9411"
            - name: SPAN_STORAGE_TYPE
              value: elasticsearch
            - name: ES_SERVER_URLS
              value: http://elasticsearch:9200
          ports:
            - containerPort: 16686  # UI
            - containerPort: 14268  # Collector HTTP
            - containerPort: 14250  # Collector gRPC
            - containerPort: 9411   # Zipkin

---
# Python Tracing ด้วย OpenTelemetry
apiVersion: v1
kind: ConfigMap
metadata:
  name: python-tracing-example
data:
  app.py: |
    from opentelemetry import trace
    from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
    from opentelemetry.sdk.trace import TracerProvider
    from opentelemetry.sdk.trace.export import BatchSpanProcessor
    from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor
    from opentelemetry.instrumentation.sqlalchemy import SQLAlchemyInstrumentor
    
    provider = TracerProvider()
    exporter = OTLPSpanExporter(endpoint="http://otel-collector:4317")
    provider.add_span_processor(BatchSpanProcessor(exporter))
    trace.set_tracer_provider(provider)
    
    tracer = trace.get_tracer(__name__)
    
    FastAPIInstrumentor.instrument_app(app)
    SQLAlchemyInstrumentor().instrument(engine=engine)
    
    @app.get("/users/{user_id}")
    async def get_user(user_id: str):
        with tracer.start_as_current_span("get_user") as span:
            span.set_attribute("user.id", user_id)
            
            with tracer.start_as_current_span("db_query"):
                user = await db.get_user(user_id)
            
            if not user:
                span.set_status(trace.StatusCode.ERROR, "User not found")
                raise HTTPException(404)
            
            return user
```

---

## Step 148: Grafana Dashboards as Code

```yaml
# Deploy Grafana dashboard via ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: myapp-dashboard
  namespace: monitoring
  labels:
    grafana_dashboard: "1"
data:
  myapp-overview.json: |
    {
      "title": "MyApp Overview",
      "uid": "myapp-overview",
      "panels": [
        {
          "id": 1,
          "title": "Request Rate",
          "type": "stat",
          "gridPos": {"h": 4, "w": 6, "x": 0, "y": 0},
          "targets": [
            {
              "expr": "sum(rate(http_requests_total{job='myapp'}[5m]))",
              "legendFormat": "req/s"
            }
          ]
        },
        {
          "id": 2,
          "title": "Error Rate",
          "type": "timeseries",
          "gridPos": {"h": 8, "w": 12, "x": 6, "y": 0},
          "targets": [
            {
              "expr": "sum(rate(http_requests_total{job='myapp', status=~'5..'}[5m])) / sum(rate(http_requests_total{job='myapp'}[5m])) * 100",
              "legendFormat": "Error %"
            }
          ]
        }
      ]
    }
```

---

## Step 149: Log Aggregation Patterns

```yaml
# Fluent Bit as DaemonSet
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluent-bit
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: fluent-bit
  template:
    spec:
      tolerations:
        - key: node-role.kubernetes.io/control-plane
          operator: Exists
          effect: NoSchedule
      
      serviceAccountName: fluent-bit
      
      containers:
        - name: fluent-bit
          image: fluent/fluent-bit:2.2
          volumeMounts:
            - name: varlog
              mountPath: /var/log
              readOnly: true
            - name: varlibdockercontainers
              mountPath: /var/lib/docker/containers
              readOnly: true
            - name: config
              mountPath: /fluent-bit/etc
      
      volumes:
        - name: varlog
          hostPath:
            path: /var/log
        - name: varlibdockercontainers
          hostPath:
            path: /var/lib/docker/containers
        - name: config
          configMap:
            name: fluent-bit-config

---
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluent-bit-config
  namespace: monitoring
data:
  fluent-bit.conf: |
    [SERVICE]
        Flush         1
        Log_Level     info
        Daemon        off
        Parsers_File  parsers.conf
    
    [INPUT]
        Name              tail
        Tag               kube.*
        Path              /var/log/containers/*.log
        Parser            cri
        DB                /var/log/flb_kube.db
        Mem_Buf_Limit     50MB
        Skip_Long_Lines   On
        Refresh_Interval  10
    
    [FILTER]
        Name                kubernetes
        Match               kube.*
        Kube_URL            https://kubernetes.default.svc:443
        Kube_CA_File        /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        Kube_Token_File     /var/run/secrets/kubernetes.io/serviceaccount/token
        Merge_Log           On
        K8S-Logging.Parser  On
        K8S-Logging.Exclude Off
    
    [OUTPUT]
        Name        loki
        Match       kube.*
        Host        loki
        Port        3100
        Labels      job=kubernetes,namespace=$kubernetes['namespace_name'],pod=$kubernetes['pod_name']
        Label_Keys  $level,$severity
        Auto_Kubernetes_Labels On
  
  parsers.conf: |
    [PARSER]
        Name   json
        Format json
        Time_Key time
        Time_Format %Y-%m-%dT%H:%M:%S.%L
    
    [PARSER]
        Name        cri
        Format      regex
        Regex       ^(?<time>[^ ]+) (?<stream>stdout|stderr) (?<logtag>[^ ]*) (?<log>.*)$
        Time_Key    time
        Time_Format %Y-%m-%dT%H:%M:%S.%LZ
```

---

## Step 150: SLI/SLO/SLA Implementation

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: slo-rules
  namespace: production
spec:
  groups:
    - name: slo.availability
      rules:
        - alert: SLOAvailabilityBurnRateFast
          expr: |
            (
              sum(rate(http_requests_total{job="myapp", status=~"5.."}[1h])) /
              sum(rate(http_requests_total{job="myapp"}[1h]))
            ) > (14.4 * 0.001)
          for: 5m
          labels:
            severity: critical
            slo: availability
          annotations:
            summary: "Fast burn rate: consuming 99.9% SLO budget"
            description: "At this rate, the SLO budget will be exhausted in 1 hour"
        
        - alert: SLOAvailabilityBurnRateSlow
          expr: |
            (
              sum(rate(http_requests_total{job="myapp", status=~"5.."}[6h])) /
              sum(rate(http_requests_total{job="myapp"}[6h]))
            ) > (6 * 0.001)
          for: 30m
          labels:
            severity: warning
            slo: availability
          annotations:
            summary: "Slow burn rate: consuming SLO budget"
    
    - name: slo.latency
      rules:
        - record: slo:myapp:http_request_duration:p95
          expr: |
            histogram_quantile(0.95,
              sum(rate(http_request_duration_seconds_bucket{job="myapp"}[5m])) by (le)
            )
        
        - alert: SLOLatencyBreach
          expr: slo:myapp:http_request_duration:p95 > 0.5
          for: 10m
          labels:
            severity: warning
          annotations:
            summary: "p95 latency {{ $value }}s exceeds 500ms SLO"
    
    - name: slo.error_budget
      rules:
        - record: slo:myapp:error_budget_remaining:ratio
          expr: |
            1 - (
              sum_over_time(
                (sum(rate(http_requests_total{job="myapp", status=~"5.."}[1h])) /
                sum(rate(http_requests_total{job="myapp"}[1h])))[30d:1h]
              ) / (30 * 24)
            ) / 0.001
```

---

## 📊 สรุป Part 15

| Component | ฟังก์ชัน | Technology |
|-----------|---------|----------|
| Metrics Collection | Scrape metrics | Prometheus |
| Alerting | Send alerts | Alertmanager |
| Visualization | Dashboards | Grafana |
| Log Collection | Aggregate logs | Fluent Bit/Promtail |
| Log Storage | Store logs | Loki |
| Distributed Tracing | Trace requests | Jaeger/Tempo |
| Trace Collection | Collect spans | OpenTelemetry |

---
*Part 15 | Steps 141-150 | ระดับกลาง/สูง*
