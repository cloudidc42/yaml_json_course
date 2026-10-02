# Part 92: Observability and SRE Practices
## Steps 881-890: Prometheus, Grafana, Tracing, SLOs, Incident Management

---

## Step 881: Observability Overview

```
Observability Pillars (The Three + One):

1. Metrics (Prometheus + Grafana)
   - Quantitative measurements over time
   - CPU, memory, request rates, error rates
   - Aggregatable, low cardinality
   - Alert on thresholds

2. Logs (Loki + Grafana)
   - Structured event records
   - JSON format: timestamp, level, message, context
   - High cardinality: per-request tracing
   - Correlate with trace IDs

3. Traces (Tempo/Jaeger + Grafana)
   - Distributed request flows
   - Span: single operation within a trace
   - Trace: entire request journey
   - Identify latency bottlenecks

4. Profiles (Parca/Pyroscope)
   - Continuous profiling: CPU flame graphs
   - Memory allocation tracking
   - Identify hot code paths

SRE Core Concepts:
  SLI: Service Level Indicator (what to measure)
    - Request success rate
    - P99 latency
    - Availability

  SLO: Service Level Objective (target)
    - 99.9% success rate over 30 days
    - P99 latency < 500ms

  Error Budget:
    - SLO 99.9% = 43.2 min/month allowed downtime
    - Budget burns rate = actual vs allowed
    - Freeze deployments when budget exhausted

Golden Signals (Google SRE):
  Latency, Traffic, Errors, Saturation (LTES)
```

---

## Step 882: Prometheus Stack Setup

```yaml
# kube-prometheus-stack Helm values
prometheus:
  prometheusSpec:
    replicas: 2
    retention: 30d
    retentionSize: 50GB
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: gp3
          resources:
            requests:
              storage: 100Gi
    serviceMonitorSelectorNilUsesHelmValues: false
    podMonitorSelectorNilUsesHelmValues: false
    remoteWrite:
      - url: http://thanos-receive.monitoring.svc:19291/api/v1/receive

alertmanager:
  alertmanagerSpec:
    replicas: 2
  config:
    route:
      group_wait: 30s
      group_interval: 5m
      repeat_interval: 12h
      receiver: slack-critical
      routes:
        - match:
            severity: critical
          receiver: pagerduty-critical
        - match:
            severity: warning
          receiver: slack-warning
    receivers:
      - name: pagerduty-critical
        pagerduty_configs:
          - service_key: $PAGERDUTY_KEY
      - name: slack-warning
        slack_configs:
          - api_url: $SLACK_WEBHOOK
            channel: "#alerts"

---
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: myapp
  namespace: production
spec:
  selector:
    matchLabels:
      app: myapp
  endpoints:
    - port: metrics
      path: /metrics
      interval: 30s
      scrapeTimeout: 10s
  namespaceSelector:
    matchNames: [production]
```

---

## Step 883: SLO with PrometheusRule

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: slo-myapp
  namespace: monitoring
spec:
  groups:
    - name: slo.myapp
      rules:
        - record: job:http_requests:rate5m
          expr: |
            sum(rate(http_requests_total{job="myapp",code!~"5.."}[5m]))
            /
            sum(rate(http_requests_total{job="myapp"}[5m]))
        
        # Fast burn: 14.4x burn rate for 1h (page immediately)
        - alert: SLOBudgetBurnFast
          expr: |
            (
              sum(rate(http_requests_total{job="myapp",code=~"5.."}[1h]))
              /
              sum(rate(http_requests_total{job="myapp"}[1h]))
            ) > 14.4 * (1 - 0.999)
          for: 2m
          annotations:
            summary: "myapp SLO error budget burning fast (14.4x)"
          labels:
            severity: critical
        
        # Slow burn: 6x burn rate for 6h (warn)
        - alert: SLOBudgetBurnSlow
          expr: |
            (
              sum(rate(http_requests_total{job="myapp",code=~"5.."}[6h]))
              /
              sum(rate(http_requests_total{job="myapp"}[6h]))
            ) > 6 * (1 - 0.999)
          for: 15m
          annotations:
            summary: "myapp SLO error budget burning at 6x rate"
          labels:
            severity: warning
        
        # Error budget remaining
        - record: job:error_budget_remaining:percent
          expr: |
            1 - (
              sum(increase(http_requests_total{job="myapp",code=~"5.."}[30d]))
              /
              sum(increase(http_requests_total{job="myapp"}[30d]))
              / (1 - 0.999)
            )
```

---

## Step 884: Grafana Dashboards

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: myapp-dashboard
  namespace: monitoring
  labels:
    grafana_dashboard: "1"
data:
  myapp.json: |
    {
      "title": "MyApp Production",
      "uid": "myapp-prod",
      "tags": ["production", "myapp"],
      "panels": [
        {
          "title": "Request Rate",
          "type": "graph",
          "targets": [{"expr": "sum(rate(http_requests_total{job=\"myapp\"}[5m])) by (code)"}]
        },
        {
          "title": "P99 Latency",
          "type": "graph",
          "targets": [{"expr": "histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket{job=\"myapp\"}[5m])) by (le))"}]
        },
        {
          "title": "Error Budget",
          "type": "stat",
          "targets": [{"expr": "job:error_budget_remaining:percent{job=\"myapp\"} * 100"}]
        }
      ]
    }
```

---

## Step 885: Distributed Tracing with Tempo

```yaml
# OpenTelemetry Collector
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
      jaeger:
        protocols:
          thrift_http:
            endpoint: 0.0.0.0:14268
    
    processors:
      batch:
        timeout: 1s
        send_batch_size: 1024
      memory_limiter:
        limit_mib: 512
    
    exporters:
      otlp:
        endpoint: tempo.monitoring.svc:4317
        tls:
          insecure: true
      prometheus:
        endpoint: 0.0.0.0:8889
    
    service:
      pipelines:
        traces:
          receivers: [otlp, jaeger]
          processors: [memory_limiter, batch]
          exporters: [otlp]
        metrics:
          receivers: [otlp]
          processors: [batch]
          exporters: [prometheus]
```

---

## Step 886: Log Aggregation with Loki

```yaml
# Loki values.yaml
loki:
  auth_enabled: false
  storage:
    type: s3
    s3:
      bucketnames: mycompany-loki
      region: us-east-1
  limits_config:
    retention_period: 30d
    ingestion_rate_mb: 16
    ingestion_burst_size_mb: 32
  compactor:
    retention_enabled: true

---
# Promtail config
apiVersion: v1
kind: ConfigMap
metadata:
  name: promtail-config
  namespace: monitoring
data:
  config.yml: |
    server:
      http_listen_port: 9080
    clients:
      - url: http://loki.monitoring.svc:3100/loki/api/v1/push
    scrape_configs:
      - job_name: kubernetes-pods
        kubernetes_sd_configs:
          - role: pod
        pipeline_stages:
          - docker: {}
          - json:
              expressions:
                level: level
                trace_id: trace_id
        relabel_configs:
          - source_labels: [__meta_kubernetes_namespace]
            target_label: namespace
          - source_labels: [__meta_kubernetes_pod_name]
            target_label: pod
          - source_labels: [__meta_kubernetes_pod_label_app]
            target_label: app
```

---

## Step 887: SLO with Sloth

```yaml
apiVersion: sloth.slok.dev/v1
kind: PrometheusServiceLevel
metadata:
  name: myapp-slo
  namespace: monitoring
spec:
  service: myapp
  labels:
    team: platform
    tier: "1"
  slos:
    - name: requests-availability
      objective: 99.9
      description: "99.9% of requests must succeed"
      sli:
        events:
          errorQuery: |
            sum(rate(http_requests_total{job="myapp",code=~"5.."}[{{.window}}]))
          totalQuery: |
            sum(rate(http_requests_total{job="myapp"}[{{.window}}]))
      alerting:
        name: MyAppHighErrorRate
        labels:
          team: platform
        page_alert:
          labels:
            severity: critical
        ticket_alert:
          labels:
            severity: warning
    
    - name: requests-latency
      objective: 99.0
      description: "99% of requests must complete under 500ms"
      sli:
        events:
          errorQuery: |
            sum(rate(http_request_duration_seconds_bucket{job="myapp",le="0.5"}[{{.window}}]))
          totalQuery: |
            sum(rate(http_request_duration_seconds_count{job="myapp"}[{{.window}}]))
```

---

## Step 888: Incident Management

```yaml
# Alertmanager PagerDuty receiver
receivers:
  - name: pagerduty-critical
    pagerduty_configs:
      - routing_key: $PAGERDUTY_INTEGRATION_KEY
        description: "{{ .CommonAnnotations.summary }}"
        client: "Alertmanager"
        details:
          firing: "{{ .Alerts.Firing | len }}"
          namespace: "{{ .CommonLabels.namespace }}"
          cluster: production
        severity: critical

---
apiVersion: v1
kind: ConfigMap
metadata:
  name: runbooks
  namespace: monitoring
data:
  HighErrorRate.md: |
    # High Error Rate Runbook
    
    ## Immediate Actions
    1. Check pod logs:
       kubectl logs -l app=myapp -n production --tail=100
    
    2. Check recent deployments:
       kubectl rollout history deployment/myapp -n production
    
    3. If recent deploy is bad, rollback:
       kubectl rollout undo deployment/myapp -n production
    
    4. Check database connectivity:
       kubectl exec -n production deploy/myapp -- nc -z postgres 5432
    
    ## Escalation
    - After 15 min: escalate to on-call engineer
    - After 30 min: escalate to engineering manager
```

---

## Step 889: Chaos Engineering

```yaml
# Chaos Mesh: pod kill
apiVersion: chaos-mesh.org/v1alpha1
kind: PodChaos
metadata:
  name: pod-kill-test
  namespace: testing
spec:
  action: pod-kill
  mode: one
  selector:
    namespaces: [production]
    labelSelectors:
      app: myapp
  scheduler:
    cron: "@every 30m"

---
# NetworkChaos: latency injection
apiVersion: chaos-mesh.org/v1alpha1
kind: NetworkChaos
metadata:
  name: network-delay-test
  namespace: testing
spec:
  action: delay
  mode: all
  selector:
    namespaces: [production]
    labelSelectors:
      app: api-service
  delay:
    latency: 100ms
    correlation: "25"
    jitter: 10ms
  direction: to
  target:
    selector:
      namespaces: [databases]
    mode: all
  duration: 5m

---
# StressChaos: CPU stress
apiVersion: chaos-mesh.org/v1alpha1
kind: StressChaos
metadata:
  name: cpu-stress-test
  namespace: testing
spec:
  mode: one
  selector:
    namespaces: [production]
    labelSelectors:
      app: myapp
  stressors:
    cpu:
      workers: 4
      load: 80
  duration: 10m
```

---

## Step 890: Workshop - SRE Monitoring

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: sre-alerts
  namespace: monitoring
spec:
  groups:
    - name: sre
      rules:
        - alert: ErrorBudgetBurnCritical
          expr: |
            job:error_budget_remaining:percent < 0.25
          for: 0m
          annotations:
            summary: "Error budget < 25% remaining for {{ $labels.job }}"
          labels:
            severity: critical

        - alert: ErrorBudgetExhausted
          expr: |
            job:error_budget_remaining:percent <= 0
          for: 0m
          annotations:
            summary: "Error budget exhausted for {{ $labels.job }} - freeze deployments"
          labels:
            severity: critical

        - alert: SLOWindowExpiry
          expr: |
            (time() - on(job) group_left slo_window_start_time) > 30 * 24 * 3600
          for: 0m
          annotations:
            summary: "SLO measurement window reset for {{ $labels.job }}"
          labels:
            severity: info
```

---

## 📊 สรุป Part 92

| Pillar | Tool | Use Case |
|--------|------|----------|
| Metrics | Prometheus + Grafana | Alerting, dashboards |
| Logs | Loki + Promtail | Log search, correlation |
| Traces | Tempo + OTel | Request flow, latency |
| SLO | Sloth + PrometheusRule | Error budget tracking |
| Incident | PagerDuty + Runbooks | Escalation, remediation |
| Chaos | Chaos Mesh | Resilience testing |

---

## 🔗 ต่อไป
- [Part 93: Multi-Cluster and Federation](./part-93-multi-cluster.md)

---
*Part 92 | Steps 881-890 | Observability and SRE Practices | Educational Content*
