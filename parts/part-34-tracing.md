# Part 34: Distributed Tracing
## Steps 321-330: Jaeger, Zipkin, OpenTelemetry

---

## 📖 บทนำ

Distributed tracing ช่วย trace การ request ผ่าน microservices หลายตัว ทำให้เห็น latency bottleneck และ error propagation

---

## Step 321: Tracing Concepts

```
Distributed Trace:
Request ────────────────────────────────────────────────▶

[Span: API Gateway     |────────────────────| ]
  [Span: Auth Service  |──|                       ]
  [Span: User Service     |────────|              ]
    [Span: DB Query         |─|                   ]
  [Span: Order Service         |──────|           ]
    [Span: Payment API            |───|           ]

Terms:
- Trace: collection ของ spans
- Span: unit ของ work (ชื่อ, start time, duration, tags)
- TraceID: unique ID สำหรับ trace ทั้งหมด
- SpanID: unique ID สำหรับ span
- Parent SpanID: span ที่ spawn span นี้
```

---

## Step 322: Jaeger

```yaml
# Jaeger All-In-One (development)
apiVersion: jaegertracing.io/v1
kind: Jaeger
metadata:
  name: jaeger
  namespace: monitoring
spec:
  strategy: allInOne
  allInOne:
    image: jaegertracing/all-in-one:1.50
  storage:
    type: memory
    options:
      memory:
        max-traces: 100000

---
# Production: Jaeger with Elasticsearch
apiVersion: jaegertracing.io/v1
kind: Jaeger
metadata:
  name: jaeger-production
  namespace: monitoring
spec:
  strategy: production
  collector:
    replicas: 2
    resources:
      requests:
        cpu: 500m
        memory: 512Mi
  query:
    replicas: 2
  storage:
    type: elasticsearch
    options:
      es:
        server-urls: https://elasticsearch:9200
        tls.enabled: true
    secretName: jaeger-elasticsearch-secret
  ingress:
    enabled: true
    hosts:
      - tracing.example.com
```

---

## Step 323: OpenTelemetry Instrumentation

```python
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.sdk.resources import Resource
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor
from opentelemetry.instrumentation.httpx import HTTPXClientInstrumentor
from opentelemetry.instrumentation.sqlalchemy import SQLAlchemyInstrumentor

def setup_tracing(service_name: str, otlp_endpoint: str):
    resource = Resource.create({
        "service.name": service_name,
        "service.version": "1.0.0",
        "deployment.environment": "production",
    })
    provider = TracerProvider(resource=resource)
    exporter = OTLPSpanExporter(endpoint=otlp_endpoint, insecure=True)
    processor = BatchSpanProcessor(exporter, max_queue_size=2048)
    provider.add_span_processor(processor)
    trace.set_tracer_provider(provider)
    return trace.get_tracer(service_name)

# Auto-instrumentation
app = fastapi.FastAPI()
FastAPIInstrumentor.instrument_app(app)
HTTPXClientInstrumentor().instrument()
SQLAlchemyInstrumentor().instrument()

# Manual spans
tracer = trace.get_tracer(__name__)

@app.get("/orders/{order_id}")
async def get_order(order_id: str):
    with tracer.start_as_current_span("get_order") as span:
        span.set_attribute("order.id", order_id)
        
        with tracer.start_as_current_span("db.query") as db_span:
            db_span.set_attribute("db.system", "postgresql")
            db_span.set_attribute("db.statement", "SELECT * FROM orders WHERE id = $1")
            order = await db.fetch_order(order_id)
        
        if not order:
            span.set_status(trace.StatusCode.ERROR, "Order not found")
            raise HTTPException(404, "Order not found")
        
        return order
```

---

## Step 324: Context Propagation

```python
import httpx
from opentelemetry.propagate import inject, extract

async def call_payment_service(order_id: str, amount: float):
    headers = {}
    inject(headers)  # เพิ่ม traceparent header
    
    async with httpx.AsyncClient() as client:
        response = await client.post(
            "http://payment-service/charge",
            json={"order_id": order_id, "amount": amount},
            headers=headers
        )
    return response.json()

def process_event(event_headers: dict):
    ctx = extract(event_headers)
    with tracer.start_as_current_span("process_event", context=ctx) as span:
        span.set_attribute("event.type", "order_created")
        do_processing()
```

---

## Step 325: Istio Tracing

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: istio
  namespace: istio-system
data:
  mesh: |
    enableTracing: true
    defaultConfig:
      tracing:
        sampling: 1.0
        zipkin:
          address: zipkin.istio-system:9411

---
apiVersion: telemetry.istio.io/v1alpha1
kind: Telemetry
metadata:
  name: tracing-config
  namespace: production
spec:
  tracing:
    - providers:
        - name: jaeger
      randomSamplingPercentage: 5.0
      customTags:
        env:
          literal:
            value: production
        version:
          header:
            name: x-app-version
            defaultValue: unknown
```

---

## Step 326: Sampling Strategies

```yaml
# OTel Collector - tail-based sampling
processors:
  tail_sampling:
    decision_wait: 10s
    num_traces: 100000
    policies:
      # เก็บทุก error trace
      - name: errors-policy
        type: status_code
        status_code:
          status_codes: [ERROR]
      
      # เก็บ slow traces (> 500ms)
      - name: slow-traces
        type: latency
        latency:
          threshold_ms: 500
      
      # Sample 1%
      - name: probabilistic-policy
        type: probabilistic
        probabilistic:
          sampling_percentage: 1
      
      # เก็บ traces ของ critical services
      - name: important-services
        type: string_attribute
        string_attribute:
          key: service.name
          values:
            - payment-service
            - auth-service
```

---

## Step 327: Trace Analysis

```python
import httpx
from datetime import datetime, timedelta

JAEGER_URL = "http://jaeger-query:16686"

async def get_slow_traces(service: str, min_duration_ms: int = 500, limit: int = 100):
    end_time = datetime.utcnow()
    start_time = end_time - timedelta(hours=1)
    
    async with httpx.AsyncClient() as client:
        response = await client.get(
            f"{JAEGER_URL}/api/traces",
            params={
                "service": service,
                "minDuration": f"{min_duration_ms}ms",
                "start": int(start_time.timestamp() * 1_000_000),
                "end": int(end_time.timestamp() * 1_000_000),
                "limit": limit,
            }
        )
    
    traces = response.json()["data"]
    results = []
    for trace in traces:
        spans = trace["spans"]
        root_span = next((s for s in spans if not s.get("references")), spans[0])
        duration_ms = root_span["duration"] / 1000
        
        results.append({
            "trace_id": trace["traceID"],
            "duration_ms": duration_ms,
            "span_count": len(spans),
        })
    
    return sorted(results, key=lambda x: x["duration_ms"], reverse=True)
```

---

## Step 328-329: Zipkin

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: zipkin
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: zipkin
  template:
    metadata:
      labels:
        app: zipkin
    spec:
      containers:
        - name: zipkin
          image: openzipkin/zipkin:3.0
          ports:
            - containerPort: 9411
          env:
            - name: STORAGE_TYPE
              value: elasticsearch
            - name: ES_HOSTS
              value: http://elasticsearch:9200
          resources:
            requests:
              cpu: 200m
              memory: 512Mi
```

---

## Step 330: Workshop

```yaml
apiVersion: opentelemetry.io/v1alpha1
kind: OpenTelemetryCollector
metadata:
  name: otel
  namespace: monitoring
spec:
  mode: DaemonSet
  config: |
    receivers:
      otlp:
        protocols:
          grpc:
            endpoint: 0.0.0.0:4317
          http:
            endpoint: 0.0.0.0:4318

    processors:
      batch:
        send_batch_size: 10000
        timeout: 10s
      tail_sampling:
        decision_wait: 10s
        policies:
          - name: errors
            type: status_code
            status_code:
              status_codes: [ERROR]
          - name: slow
            type: latency
            latency:
              threshold_ms: 500

    exporters:
      jaeger:
        endpoint: http://jaeger-collector:14250
        tls:
          insecure: true

    service:
      pipelines:
        traces:
          receivers: [otlp]
          processors: [batch, tail_sampling]
          exporters: [jaeger]

---
apiVersion: opentelemetry.io/v1alpha1
kind: Instrumentation
metadata:
  name: python-instrumentation
  namespace: production
spec:
  exporter:
    endpoint: http://otel-collector:4317
  propagators:
    - tracecontext
    - baggage
  sampler:
    type: parentbased_traceidratio
    argument: "0.05"
```

---

## 📊 สรุป Part 34

| Component | หน้าที่ |
|-----------|----------|
| Jaeger | Distributed tracing backend |
| Zipkin | Alternative tracing backend |
| OpenTelemetry | Instrumentation library + collector |
| Context Propagation | ส่ง trace context ระหว่าง services |
| Tail Sampling | ตัดสินใจ sample หลัง trace สมบูรณ์ |

---

## 🔗 ต่อไป
- [Part 35: KEDA Autoscaling](./part-35-keda.md)

---
*Part 34 | Steps 321-330 | ระดับสูง*
