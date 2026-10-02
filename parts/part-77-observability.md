# Part 77: Kubernetes Observability
## Steps 731-740: Prometheus, Grafana, Tracing, Logging

---

## Step 731: Observability Pillars Overview

```
Three Pillars of Observability:

1. Metrics (Prometheus/Thanos):
   - Numerical time-series data
   - Aggregatable, low-cardinality
   - RED method: Rate, Errors, Duration
   - USE method: Utilization, Saturation, Errors

2. Logs (Loki/FluentBit/Elasticsearch):
   - Events with context
   - High-cardinality OK (per-request)
   - Structured (JSON) preferred

3. Traces (Jaeger/Tempo/Zipkin):
   - Request flow across services
   - Latency breakdown per hop
   - Distributed context propagation

OpenTelemetry (OTel):
  - Vendor-neutral standard
  - SDK for Go, Java, Python, JS, etc.
  - Collector: receive -> process -> export
  - OTLP protocol (gRPC/HTTP)

Four Golden Signals (Google SRE):
  - Latency: time to serve requests
  - Traffic: requests per second
  - Errors: error rate
  - Saturation: resource utilization
```

---

## Step 732: Prometheus Stack Setup

```yaml
# kube-prometheus-stack Helm values
alertmanager:
  enabled: true
  alertmanagerSpec:
    retention: 120h
    storage:
      volumeClaimTemplate:
        spec:
          resources:
            requests:
              storage: 10Gi

prometheus:
  prometheusSpec:
    retention: 30d
    retentionSize: 45GB
    storageSpec:
      volumeClaimTemplate:
        spec:
          resources:
            requests:
              storage: 50Gi
    serviceMonitorSelectorNilUsesHelmValues: false
    podMonitorSelectorNilUsesHelmValues: false
    externalLabels:
      cluster: production-us-east-1
      environment: production
    thanos:
      baseImage: quay.io/thanos/thanos
      version: v0.32.0

grafana:
  enabled: true
  adminPassword: changeme
  ingress:
    enabled: true
    hosts: [grafana.example.com]
  dashboardProviders:
    dashboardproviders.yaml:
      apiVersion: 1
      providers:
        - name: default
          folder: Kubernetes
          type: file
          options:
            path: /var/lib/grafana/dashboards
  dashboards:
    default:
      k8s-cluster:
        gnetId: 7249
        datasource: Prometheus
```

---

## Step 733: ServiceMonitor and PodMonitor

```yaml
# ServiceMonitor: scrape a Service's pods
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: myapp
  namespace: monitoring
  labels:
    app: myapp
spec:
  namespaceSelector:
    matchNames:
      - production
  selector:
    matchLabels:
      app: myapp
  endpoints:
    - port: metrics
      interval: 30s
      path: /metrics
      scrapeTimeout: 10s
      relabelings:
        - sourceLabels: [__meta_kubernetes_pod_name]
          targetLabel: pod
        - sourceLabels: [__meta_kubernetes_namespace]
          targetLabel: namespace

---
# PodMonitor: scrape pods directly
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: batch-jobs
  namespace: monitoring
spec:
  namespaceSelector:
    matchNames:
      - batch
  selector:
    matchLabels:
      monitoring: "true"
  podMetricsEndpoints:
    - port: metrics
      interval: 60s

---
# Service with metrics port
apiVersion: v1
kind: Service
metadata:
  name: myapp
  namespace: production
  labels:
    app: myapp
spec:
  selector:
    app: myapp
  ports:
    - name: http
      port: 80
      targetPort: 8080
    - name: metrics
      port: 9090
      targetPort: 9090
```

---

## Step 734: PrometheusRule - Alerts and Recording Rules

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: myapp-rules
  namespace: monitoring
  labels:
    prometheus: kube-prometheus
spec:
  groups:
    - name: myapp.recording
      interval: 30s
      rules:
        - record: job:http_requests:rate5m
          expr: sum(rate(http_requests_total[5m])) by (job, status_code)
        
        - record: job:http_errors:rate5m
          expr: |
            sum(rate(http_requests_total{status_code=~"5.."}[5m])) by (job)
            /
            sum(rate(http_requests_total[5m])) by (job)
    
    - name: myapp.alerts
      rules:
        - alert: HighErrorRate
          expr: job:http_errors:rate5m > 0.05
          for: 5m
          annotations:
            summary: "Error rate > 5% for {{ $labels.job }}"
            description: "Current error rate: {{ $value | humanizePercentage }}"
            runbook_url: "https://wiki.example.com/runbooks/high-error-rate"
          labels:
            severity: critical
            team: platform
        
        - alert: SlowResponses
          expr: |
            histogram_quantile(0.99,
              sum(rate(http_request_duration_seconds_bucket[5m])) by (le, job)
            ) > 2
          for: 10m
          annotations:
            summary: "P99 latency > 2s for {{ $labels.job }}"
          labels:
            severity: warning

        - alert: SLOBudgetBurning
          expr: |
            (
              sum(rate(http_requests_total{status_code=~"5.."}[1h]))
              /
              sum(rate(http_requests_total[1h]))
            ) > 0.001
          for: 5m
          labels:
            severity: critical
          annotations:
            summary: "SLO budget burning: error rate > 0.1%"
```

---

## Step 735: Alertmanager Configuration

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: alertmanager-kube-prometheus-stack-alertmanager
  namespace: monitoring
type: Opaque
stringData:
  alertmanager.yaml: |
    global:
      smtp_smarthost: 'smtp.gmail.com:587'
      smtp_from: 'alerts@example.com'
      slack_api_url: 'https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK'
    
    route:
      group_by: ['alertname', 'cluster', 'namespace']
      group_wait: 30s
      group_interval: 5m
      repeat_interval: 4h
      receiver: slack-default
      routes:
        - match:
            severity: critical
          receiver: pagerduty-critical
          continue: true
        
        - match_re:
            alertname: "^(PVC|Postgres|StatefulSet).*"
          receiver: slack-dba
        
        - match:
            severity: warning
          receiver: slack-default
          mute_time_intervals:
            - nights-and-weekends
    
    receivers:
      - name: slack-default
        slack_configs:
          - channel: '#alerts'
            title: '{{ .GroupLabels.alertname }}'
            text: '{{ range .Alerts }}{{ .Annotations.summary }}{{ end }}'
            send_resolved: true
      
      - name: pagerduty-critical
        pagerduty_configs:
          - service_key: 'YOUR_PAGERDUTY_KEY'
            severity: '{{ .Labels.severity }}'
      
      - name: slack-dba
        slack_configs:
          - channel: '#dba-alerts'
            send_resolved: true
    
    mute_time_intervals:
      - name: nights-and-weekends
        time_intervals:
          - weekdays: [saturday, sunday]
          - times:
              - start_time: 18:00
                end_time: 08:00
```

---

## Step 736: Distributed Tracing with Jaeger/Tempo

```yaml
# Grafana Tempo Helm values (scalable tracing)
distributor:
  receivers:
    otlp:
      protocols:
        grpc:
          endpoint: 0.0.0.0:4317
        http:
          endpoint: 0.0.0.0:4318
    jaeger:
      protocols:
        grpc: {}
        thrift_http: {}

storage:
  trace:
    backend: s3
    s3:
      bucket: my-tempo-bucket
      region: us-east-1

---
# OpenTelemetry Collector: receive, process, export
apiVersion: v1
kind: ConfigMap
metadata:
  name: otel-collector-config
  namespace: observability
data:
  config.yaml: |
    receivers:
      otlp:
        protocols:
          grpc:
            endpoint: 0.0.0.0:4317
          http:
            endpoint: 0.0.0.0:4318
    
    processors:
      batch:
        timeout: 10s
        send_batch_size: 1000
      
      memory_limiter:
        limit_mib: 512
        spike_limit_mib: 128
      
      resource:
        attributes:
          - key: deployment.environment
            value: production
            action: upsert
    
    exporters:
      otlp/tempo:
        endpoint: tempo.observability.svc:4317
        tls:
          insecure: true
      
      prometheus:
        endpoint: "0.0.0.0:8889"
    
    service:
      pipelines:
        traces:
          receivers: [otlp]
          processors: [memory_limiter, batch, resource]
          exporters: [otlp/tempo]
        metrics:
          receivers: [otlp]
          processors: [batch]
          exporters: [prometheus]
```

---

## Step 737: Structured Logging with Loki

```yaml
# FluentBit ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluent-bit-config
  namespace: logging
data:
  fluent-bit.conf: |
    [SERVICE]
        Flush         5
        Log_Level     info
        Parsers_File  parsers.conf

    [INPUT]
        Name              tail
        Path              /var/log/containers/*.log
        multiline.parser  docker, cri
        Tag               kube.*
        Refresh_Interval  5
        Mem_Buf_Limit     50MB
        Skip_Long_Lines   On

    [FILTER]
        Name                kubernetes
        Match               kube.*
        Kube_URL            https://kubernetes.default.svc:443
        Kube_CA_File        /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        Kube_Token_File     /var/run/secrets/kubernetes.io/serviceaccount/token
        Merge_Log           On
        Keep_Log            Off
        K8S-Logging.Parser  On
        K8S-Logging.Exclude On

    [FILTER]
        Name    grep
        Match   kube.*
        Exclude $kubernetes['namespace_name'] kube-system

    [OUTPUT]
        Name        loki
        Match       kube.*
        Host        loki.logging.svc
        Port        3100
        Labels      job=fluentbit, namespace=$kubernetes['namespace_name'], pod=$kubernetes['pod_name']
        Auto_Kubernetes_Labels On

  parsers.conf: |
    [PARSER]
        Name        json
        Format      json
        Time_Key    time
        Time_Format %Y-%m-%dT%H:%M:%S.%L

---
# Loki Helm values
loki:
  enabled: true
  persistence:
    enabled: true
    size: 50Gi
  config:
    schema_config:
      configs:
        - from: "2024-01-01"
          store: boltdb-shipper
          object_store: s3
          schema: v12
          index:
            prefix: index_
            period: 24h
    storage_config:
      aws:
        s3: s3://my-loki-bucket/
        region: us-east-1
    limits_config:
      retention_period: 30d
      ingestion_rate_mb: 16
      max_streams_per_user: 10000
```

---

## Step 738: OpenTelemetry Auto-Instrumentation

```yaml
# OTel Operator: auto-instrument applications
apiVersion: opentelemetry.io/v1alpha1
kind: Instrumentation
metadata:
  name: auto-instrumentation
  namespace: production
spec:
  exporter:
    endpoint: http://otel-collector.observability.svc:4317
  propagators:
    - tracecontext
    - baggage
    - b3
  sampler:
    type: parentbased_traceidratio
    argument: "0.1"
  java:
    image: ghcr.io/open-telemetry/opentelemetry-operator/autoinstrumentation-java:1.30.0
  python:
    image: ghcr.io/open-telemetry/opentelemetry-operator/autoinstrumentation-python:0.41b0
  nodejs:
    image: ghcr.io/open-telemetry/opentelemetry-operator/autoinstrumentation-nodejs:0.41.1

---
# Enable auto-instrumentation per deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: java-service
  namespace: production
spec:
  template:
    metadata:
      annotations:
        instrumentation.opentelemetry.io/inject-java: "true"
    spec:
      containers:
        - name: app
          image: myapp-java:1.0
          env:
            - name: OTEL_SERVICE_NAME
              value: java-service
            - name: OTEL_RESOURCE_ATTRIBUTES
              value: "team=platform,environment=production"
```

---

## Step 739: Grafana Dashboards as Code

```yaml
# Grafana dashboard ConfigMap (auto-loaded via sidecar)
apiVersion: v1
kind: ConfigMap
metadata:
  name: myapp-dashboard
  namespace: monitoring
  labels:
    grafana_dashboard: "1"
data:
  myapp-dashboard.json: |
    {
      "title": "MyApp Overview",
      "uid": "myapp-overview",
      "panels": [
        {
          "title": "Request Rate",
          "type": "graph",
          "targets": [
            {
              "expr": "sum(rate(http_requests_total[5m])) by (status_code)",
              "legendFormat": "{{ status_code }}"
            }
          ],
          "gridPos": {"h": 8, "w": 12, "x": 0, "y": 0}
        },
        {
          "title": "P99 Latency",
          "type": "graph",
          "targets": [
            {
              "expr": "histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))",
              "legendFormat": "p99"
            }
          ],
          "gridPos": {"h": 8, "w": 12, "x": 12, "y": 0}
        },
        {
          "title": "Error Rate",
          "type": "stat",
          "targets": [
            {
              "expr": "sum(rate(http_requests_total{status_code=~\"5..\"}[5m])) / sum(rate(http_requests_total[5m]))"
            }
          ],
          "fieldConfig": {
            "defaults": {
              "unit": "percentunit",
              "thresholds": {
                "steps": [
                  {"color": "green", "value": 0},
                  {"color": "yellow", "value": 0.01},
                  {"color": "red", "value": 0.05}
                ]
              }
            }
          },
          "gridPos": {"h": 4, "w": 6, "x": 0, "y": 8}
        }
      ]
    }
```

---

## Step 740: Workshop - Observability Stack Checklist

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: observability-checklist
  namespace: monitoring
data:
  checklist.yaml: |
    metrics:
      - kube-prometheus-stack deployed
      - ServiceMonitors for all services
      - PrometheusRules with SLO alerts
      - Grafana dashboards: cluster, namespace, app
      - Alertmanager routes to Slack/PD
      - Thanos for long-term storage (optional)
    
    logging:
      - FluentBit DaemonSet collecting all pod logs
      - Loki receiving and indexing
      - Grafana Loki datasource configured
      - Log retention policy set (30d recommended)
      - Structured JSON logging in apps
    
    tracing:
      - OTel Collector deployed
      - Jaeger or Tempo backend
      - Apps instrumented (OTel auto or manual)
      - Grafana Tempo/Jaeger datasource
      - Trace-metric correlation (exemplars)
    
    slos:
      - Define SLIs for each service (error rate, latency)
      - Set SLO targets (99.9%, 99.5%, etc.)
      - Create error budget alerts
      - Monthly SLO review process

---
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: observability-health
  namespace: monitoring
spec:
  groups:
    - name: observability
      rules:
        - alert: PrometheusTargetMissing
          expr: up == 0
          for: 5m
          annotations:
            summary: "Prometheus target {{ $labels.job }} is down"
          labels:
            severity: critical

        - alert: FluentBitDown
          expr: |
            absent(kube_daemonset_status_number_ready{daemonset="fluent-bit",namespace="logging"})
            or
            kube_daemonset_status_number_ready{daemonset="fluent-bit",namespace="logging"}
            < kube_daemonset_status_desired_number_scheduled{daemonset="fluent-bit",namespace="logging"}
          for: 5m
          annotations:
            summary: "FluentBit not running on all nodes"
          labels:
            severity: warning
```

---

## 📊 สรุป Part 77

| Component | Tool | Storage | Protocol |
|-----------|------|---------|----------|
| Metrics | Prometheus | TSDB / S3 (Thanos) | PromQL |
| Logs | Loki + FluentBit | S3 | LogQL |
| Traces | Jaeger / Tempo | Memory / S3 | OTLP |
| Dashboards | Grafana | Postgres | - |
| Alerts | Alertmanager | In-memory | Webhook |

---

## 🔗 ต่อไป
- [Part 78: Multi-Tenancy and Resource Management](./part-78-multi-tenancy.md)

---
*Part 77 | Steps 731-740 | Observability | Educational Content*
