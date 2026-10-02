# Part 32: Prometheus Monitoring
## Steps 301-310: Kubernetes Observability Stack

---

## 📖 บทนำ

Prometheus เป็น monitoring system และ time-series database สำหรับ Kubernetes ecosystem รวมกับ Grafana สร้าง observability stack ที่สมบูรณ์

---

## Step 301: Prometheus Architecture

```
Architecture:
┌─────────────────────────────────────────────────┐
│                   Prometheus Server              │
│  ┌──────────┐  ┌───────────┐  ┌─────────────┐  │
│  │  Scraper  │  │  TSDB     │  │  Rule Eval  │  │
│  └──────────┘  └───────────┘  └─────────────┘  │
│  ┌──────────────────────────────────────────┐   │
│  │          HTTP API / PromQL               │   │
│  └──────────────────────────────────────────┘   │
└─────────────────────────────────────────────────┘
         │                        │
    ┌────▼────┐              ┌────▼────┐
    │  Jobs/  │              │  Alert  │
    │ Targets │              │Manager  │
    └─────────┘              └─────────┘
```

```yaml
# kube-prometheus-stack values.yaml
prometheus:
  prometheusSpec:
    retention: 30d
    retentionSize: 50GB
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: fast-ssd
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 100Gi
    resources:
      requests:
        cpu: 500m
        memory: 2Gi
      limits:
        cpu: 2000m
        memory: 8Gi
    externalLabels:
      cluster: prod-sea
      region: ap-southeast-1

grafana:
  enabled: true
  persistence:
    enabled: true
    size: 10Gi

alertmanager:
  alertmanagerSpec:
    storage:
      volumeClaimTemplate:
        spec:
          storageClassName: standard
          resources:
            requests:
              storage: 10Gi
```

---

## Step 302: ServiceMonitor

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: my-app
  namespace: monitoring
  labels:
    release: kube-prometheus-stack
spec:
  namespaceSelector:
    matchNames:
      - production
  selector:
    matchLabels:
      app: my-app
      metrics: enabled
  endpoints:
    - port: metrics
      interval: 30s
      scrapeTimeout: 10s
      path: /metrics
      relabelings:
        - sourceLabels: [__meta_kubernetes_pod_name]
          targetLabel: pod
        - sourceLabels: [__meta_kubernetes_namespace]
          targetLabel: namespace
      metricRelabelings:
        - sourceLabels: [__name__]
          regex: "go_gc_.*"
          action: drop

---
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: my-app-pods
  namespace: monitoring
spec:
  namespaceSelector:
    matchNames:
      - production
  selector:
    matchLabels:
      app: my-app
  podMetricsEndpoints:
    - port: metrics
      interval: 15s
```

---

## Step 303: PrometheusRule

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: my-app-alerts
  namespace: monitoring
  labels:
    release: kube-prometheus-stack
spec:
  groups:
    - name: my-app.rules
      interval: 30s
      rules:
        - record: job:my_app_requests:rate5m
          expr: |
            sum by (job, status_code) (
              rate(http_requests_total[5m])
            )

        - alert: MyAppHighErrorRate
          expr: |
            (
              sum(rate(http_requests_total{job="my-app", status_code=~"5.."}[5m]))
              /
              sum(rate(http_requests_total{job="my-app"}[5m]))
            ) > 0.05
          for: 5m
          labels:
            severity: critical
            team: backend
          annotations:
            summary: "High error rate on {{ $labels.job }}"
            description: "Error rate is {{ $value | humanizePercentage }}"
            runbook: "https://wiki.company.com/runbooks/high-error-rate"

        - alert: MyAppHighLatency
          expr: |
            histogram_quantile(0.99,
              sum by (le, job) (
                rate(http_request_duration_seconds_bucket{job="my-app"}[5m])
              )
            ) > 1.0
          for: 10m
          labels:
            severity: warning
          annotations:
            summary: "P99 latency > 1s on {{ $labels.job }}"

        - alert: PodCrashLooping
          expr: |
            rate(kube_pod_container_status_restarts_total[15m]) * 60 * 5 > 5
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "Pod {{ $labels.namespace }}/{{ $labels.pod }} is crash looping"
```

---

## Step 304: Alertmanager

```yaml
apiVersion: monitoring.coreos.com/v1alpha1
kind: AlertmanagerConfig
metadata:
  name: my-app-alerts
  namespace: production
spec:
  route:
    groupBy: ["alertname", "cluster", "service"]
    groupWait: 30s
    groupInterval: 5m
    repeatInterval: 4h
    receiver: slack-notifications
    routes:
      - matchers:
          - name: severity
            value: critical
        receiver: pagerduty
        repeatInterval: 1h

  receivers:
    - name: slack-notifications
      slackConfigs:
        - apiURL:
            key: slackWebhookUrl
            name: alertmanager-secrets
          channel: "#alerts"
          sendResolved: true
          title: "[{{ .Status | toUpper }}] {{ .CommonLabels.alertname }}"
          text: |
            {{ range .Alerts }}
            *Alert:* {{ .Annotations.summary }}
            *Details:* {{ .Annotations.description }}
            {{ end }}

    - name: pagerduty
      pagerdutyConfigs:
        - serviceKey:
            key: pagerdutyKey
            name: alertmanager-secrets
          description: "{{ .CommonAnnotations.summary }}"
```

---

## Step 305: PromQL

```promql
# Request rate
rate(http_requests_total{job="my-app"}[5m])

# Aggregation
sum by (service) (rate(http_requests_total[5m]))

# P95 latency
histogram_quantile(0.95, 
  sum by (le) (
    rate(http_request_duration_seconds_bucket{job="my-app"}[5m])
  )
)

# Error rate percentage
100 * (
  sum(rate(http_requests_total{status_code=~"5.."}[5m]))
  /
  sum(rate(http_requests_total[5m]))
)

# CPU throttling
rate(container_cpu_cfs_throttled_seconds_total[5m])

# Node CPU utilization
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# SLO: 99th percentile requests under 500ms
(
  sum(rate(http_request_duration_seconds_bucket{le="0.5"}[5m]))
  /
  sum(rate(http_request_duration_seconds_count[5m]))
) * 100
```

---

## Step 306: Grafana Dashboard

```json
{
  "title": "My App Overview",
  "uid": "my-app-overview",
  "panels": [
    {
      "title": "Request Rate",
      "type": "timeseries",
      "targets": [{
        "expr": "sum by (status_code) (rate(http_requests_total{job=\"my-app\"}[5m]))",
        "legendFormat": "{{ status_code }}"
      }]
    },
    {
      "title": "P95 Latency",
      "type": "gauge",
      "targets": [{
        "expr": "histogram_quantile(0.95, sum by (le) (rate(http_request_duration_seconds_bucket{job=\"my-app\"}[5m])))"
      }],
      "fieldConfig": {
        "defaults": {
          "unit": "s",
          "thresholds": {
            "steps": [
              {"color": "green", "value": 0},
              {"color": "yellow", "value": 0.5},
              {"color": "red", "value": 1.0}
            ]
          }
        }
      }
    }
  ]
}
```

---

## Step 307: Thanos

```yaml
# Thanos Sidecar
containers:
  - name: thanos-sidecar
    image: quay.io/thanos/thanos:v0.32.0
    args:
      - sidecar
      - --tsdb.path=/prometheus
      - --prometheus.url=http://localhost:9090
      - --grpc-address=0.0.0.0:10901
      - --objstore.config-file=/etc/thanos/objstore.yaml

---
# objstore config (S3)
apiVersion: v1
kind: Secret
metadata:
  name: thanos-objstore-config
stringData:
  objstore.yaml: |
    type: S3
    config:
      bucket: thanos-metrics-prod
      endpoint: s3.ap-southeast-1.amazonaws.com
      region: ap-southeast-1
```

---

## Step 308-309: App Instrumentation

```python
# Python prometheus_client
from prometheus_client import Counter, Histogram, Gauge, start_http_server
import time

REQUEST_COUNT = Counter(
    'http_requests_total',
    'Total HTTP requests',
    ['method', 'endpoint', 'status_code']
)

REQUEST_DURATION = Histogram(
    'http_request_duration_seconds',
    'HTTP request duration',
    ['method', 'endpoint'],
    buckets=[0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0]
)

ACTIVE_REQUESTS = Gauge(
    'http_active_requests',
    'Active HTTP requests in flight'
)

def instrumented_handler(method, endpoint, handler):
    ACTIVE_REQUESTS.inc()
    start = time.time()
    try:
        response = handler()
        REQUEST_COUNT.labels(
            method=method,
            endpoint=endpoint,
            status_code=response.status_code
        ).inc()
        return response
    except Exception:
        REQUEST_COUNT.labels(method=method, endpoint=endpoint, status_code=500).inc()
        raise
    finally:
        duration = time.time() - start
        REQUEST_DURATION.labels(method=method, endpoint=endpoint).observe(duration)
        ACTIVE_REQUESTS.dec()

start_http_server(9090)
```

---

## Step 310: Workshop

```yaml
# SLO Alert
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: slo-alerts
  namespace: monitoring
  labels:
    release: kube-prometheus-stack
spec:
  groups:
    - name: slo
      rules:
        - alert: SLOViolation
          expr: |
            (
              1 - (
                sum(rate(http_requests_total{status_code!~"5.."}[5m]))
                / sum(rate(http_requests_total[5m]))
              )
            ) > 0.01
          for: 5m
          labels:
            severity: critical
          annotations:
            summary: "SLO violation: error rate > 1%"
```

---

## 📊 สรุป Part 32

| Component | หน้าที่ |
|-----------|----------|
| Prometheus | Metrics collection, storage |
| ServiceMonitor | กำหนด scrape targets |
| PrometheusRule | Alert + recording rules |
| Alertmanager | Alert routing + notifications |
| Grafana | Visualization |
| Thanos | Long-term metrics storage |

---

## 🔗 ต่อไป
- [Part 33: Logging Stack](./part-33-logging.md)

---
*Part 32 | Steps 301-310 | ระดับสูง*
