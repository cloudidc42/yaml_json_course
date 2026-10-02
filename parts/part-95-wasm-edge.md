# Part 95: WebAssembly and Edge Computing
## Steps 911-920: WASM Workloads, Spin, WasmEdge, Krustlet, Edge Kubernetes

---

## Step 911: WebAssembly Overview

```
WebAssembly (WASM) in Kubernetes:

Why WASM?
  - Near-native performance (compiled bytecode)
  - Sandboxed: memory-safe by design
  - Polyglot: Rust, C, C++, Go, Python → WASM
  - Instant startup: microseconds vs seconds (container)
  - Small: WASM binary 1-10MB vs container 100MB+
  - Portable: run anywhere (server, edge, browser)

WASM vs Containers:
  Containers:
    - Full OS userspace
    - 100ms+ cold start
    - MBs of image
    - Linux-specific
  
  WASM:
    - No OS overhead
    - Microsecond cold start
    - KB to MB binary
    - Truly portable
  
  WASM is NOT a replacement for containers:
  - No syscall access (needs WASI interface)
  - Limited language support
  - Young ecosystem
  - Best for: functions, plugins, edge, serverless

WASM Runtimes:
  Wasmtime: Bytecode Alliance, used in Spin
  WasmEdge: CNCF project, optimized for cloud-native
  Wasmer: standalone + embedded
  
Kubernetes Integration:
  Containerd shim: wasm (runwasi)
  RuntimeClass: specify WASM runtime
  Spin: Fermyon's WASM microservices framework
```

---

## Step 912: WASM RuntimeClass

```yaml
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: wasmtime
handler: wasmtime
scheduling:
  nodeSelector:
    runtime/wasmtime: "true"

---
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: wasmedge
handler: wasmedge
scheduling:
  nodeSelector:
    runtime/wasmedge: "true"

---
apiVersion: v1
kind: Pod
metadata:
  name: wasm-hello
  namespace: default
spec:
  runtimeClassName: wasmtime
  containers:
    - name: hello-wasm
      image: ghcr.io/myorg/hello-wasm:latest
      resources:
        requests:
          cpu: 10m
          memory: 16Mi
        limits:
          cpu: 100m
          memory: 64Mi
  restartPolicy: Never
```

---

## Step 913: Spin (Fermyon) WASM Framework

```yaml
# spin.toml: Spin application manifest
spin_manifest_version = 2

[application]
name = "payment-service"
version = "1.0.0"
description = "Payment processing WASM service"

[[trigger.http]]
route = "/api/payments/..."
component = "payment-handler"

[component.payment-handler]
source = "target/wasm32-wasi/release/payment_handler.wasm"
allowed_outbound_hosts = [
  "https://api.stripe.com",
  "postgres://payment-db.production.svc:5432",
]

[component.payment-handler.variables]
stripe_api_key = "{{ stripe_key }}"
db_url = "{{ database_url }}"

---
apiVersion: core.spinoperator.dev/v1alpha1
kind: SpinApp
metadata:
  name: payment-service
  namespace: production
spec:
  image: ghcr.io/myorg/payment-service:1.0
  replicas: 3
  executor: containerd-shim-spin
  resources:
    requests:
      cpu: 50m
      memory: 32Mi
    limits:
      cpu: 200m
      memory: 128Mi
  variables:
    - name: stripe_key
      valueFrom:
        secretKeyRef:
          name: payment-secrets
          key: stripe-api-key
    - name: database_url
      valueFrom:
        secretKeyRef:
          name: payment-secrets
          key: db-url
```

---

## Step 914: WasmEdge for AI Inference

```yaml
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: wasmedge
handler: wasmedge
scheduling:
  nodeSelector:
    runtime/wasmedge: "true"

---
apiVersion: v1
kind: Pod
metadata:
  name: llm-inference-edge
  namespace: edge-ai
spec:
  runtimeClassName: wasmedge
  containers:
    - name: llm-infer
      image: ghcr.io/second-state/llama-inference-wasm:latest
      env:
        - name: MODEL_PATH
          value: /models/llama-2-7b.gguf
        - name: PORT
          value: "8080"
      volumeMounts:
        - name: models
          mountPath: /models
      resources:
        requests:
          cpu: 500m
          memory: 8Gi
        limits:
          cpu: 4000m
          memory: 16Gi
  volumes:
    - name: models
      persistentVolumeClaim:
        claimName: ai-models-pvc
```

---

## Step 915: Edge Kubernetes (K3s)

```yaml
# K3s server config: /etc/rancher/k3s/config.yaml
cluster-init: true
disable:
  - traefik
  - servicelb
tls-san:
  - edge-cluster.mycompany.com
  - 192.168.1.100
node-label:
  - "location=factory-floor"
  - "tier=edge"

---
apiVersion: fleet.cattle.io/v1alpha1
kind: GitRepo
metadata:
  name: edge-workloads
  namespace: fleet-default
spec:
  repo: https://github.com/myorg/edge-configs
  branch: main
  targets:
    - name: factory-edge
      clusterSelector:
        matchLabels:
          location: factory-floor
      helm:
        values:
          replicaCount: 1
          storage: local-path
```

---

## Step 916: KubeEdge

```yaml
apiVersion: devices.kubeedge.io/v1beta1
kind: Device
metadata:
  name: temperature-sensor-01
  namespace: edge
spec:
  deviceModelRef:
    name: temperature-sensor-model
  nodeName: edge-node-factory-01
  properties:
    - name: temperature
      desired:
        value: "25"
      visitors:
        protocolName: modbus
        configData:
          register: "40001"
          offset: 0

---
apiVersion: devices.kubeedge.io/v1beta1
kind: DeviceModel
metadata:
  name: temperature-sensor-model
  namespace: edge
spec:
  properties:
    - name: temperature
      description: Current temperature reading
      type:
        float:
          accessMode: ReadOnly
          minimum: -50
          maximum: 100
          unit: Celsius

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: data-collector
  namespace: edge
spec:
  selector:
    matchLabels:
      app: data-collector
  template:
    spec:
      nodeSelector:
        kubernetes.io/hostname: edge-node-factory-01
      tolerations:
        - key: node-role.kubernetes.io/edge
          operator: Exists
          effect: NoSchedule
      containers:
        - name: collector
          image: myorg/edge-collector:1.0
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
```

---

## Step 917: Knative Serving

```yaml
apiVersion: operator.knative.dev/v1beta1
kind: KnativeServing
metadata:
  name: knative-serving
  namespace: knative-serving
spec:
  version: "1.13"
  ingress:
    kourier:
      enabled: true

---
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: payment-processor
  namespace: production
spec:
  template:
    metadata:
      annotations:
        autoscaling.knative.dev/minScale: "1"
        autoscaling.knative.dev/maxScale: "100"
        autoscaling.knative.dev/target: "50"
    spec:
      containers:
        - image: ghcr.io/myorg/payment-processor:1.0
          ports:
            - containerPort: 8080
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 1000m
              memory: 512Mi

---
apiVersion: sources.knative.dev/v1
kind: ApiServerSource
metadata:
  name: pod-event-source
  namespace: production
spec:
  serviceAccountName: event-watcher
  mode: Resource
  resources:
    - apiVersion: v1
      kind: Event
  sink:
    ref:
      apiVersion: serving.knative.dev/v1
      kind: Service
      name: event-processor
```

---

## Step 918: KEDA Event-Driven Autoscaling

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: payment-processor-scaler
  namespace: production
spec:
  scaleTargetRef:
    name: payment-processor
  minReplicaCount: 1
  maxReplicaCount: 50
  cooldownPeriod: 60
  triggers:
    - type: kafka
      metadata:
        bootstrapServers: kafka.kafka.svc:9092
        topic: payment-events
        consumerGroup: payment-processors
        lagThreshold: "10"

---
apiVersion: keda.sh/v1alpha1
kind: ScaledJob
metadata:
  name: image-resize-job
  namespace: media
spec:
  jobTargetRef:
    template:
      spec:
        containers:
          - name: resizer
            image: myorg/image-resizer:1.0
        restartPolicy: Never
  minReplicaCount: 0
  maxReplicaCount: 20
  triggers:
    - type: aws-sqs-queue
      authenticationRef:
        name: keda-aws-creds
      metadata:
        queueURL: https://sqs.us-east-1.amazonaws.com/123456789/image-resize-queue
        queueLength: "5"
        awsRegion: us-east-1

---
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: keda-aws-creds
  namespace: media
spec:
  podIdentity:
    provider: aws
```

---

## Step 919: OpenFaaS Functions

```yaml
apiVersion: openfaas.com/v1
kind: Function
metadata:
  name: payment-webhook
  namespace: openfaas-fn
spec:
  name: payment-webhook
  image: ghcr.io/myorg/payment-webhook:1.0
  labels:
    com.openfaas.scale.min: "2"
    com.openfaas.scale.max: "20"
    com.openfaas.scale.target: "50"
  environment:
    read_timeout: "10s"
    write_timeout: "10s"
    STRIPE_ENDPOINT: https://api.stripe.com
  secrets:
    - stripe-credentials
  requests:
    memory: "128Mi"
    cpu: "100m"
  limits:
    memory: "256Mi"
    cpu: "500m"
```

---

## Step 920: Workshop - WASM + Edge Summary

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: wasm-edge-alerts
  namespace: monitoring
spec:
  groups:
    - name: wasm-edge
      rules:
        - alert: WASMPodHighMemory
          expr: |
            container_memory_usage_bytes{runtime_class="wasmtime"} 
            > 128 * 1024 * 1024
          for: 5m
          annotations:
            summary: "WASM pod using > 128MB (unexpected)"
          labels:
            severity: warning

        - alert: EdgeNodeOffline
          expr: |
            kube_node_status_condition{
              condition="Ready",
              status="false",
              node=~"edge-.*"
            } == 1
          for: 2m
          annotations:
            summary: "Edge node {{ $labels.node }} is offline"
          labels:
            severity: critical

        - alert: KnativeServiceNotReady
          expr: |
            knative_serving_service_ready_total == 0
          for: 5m
          annotations:
            summary: "Knative service {{ $labels.name }} not ready"
          labels:
            severity: warning

        - alert: KEDAScalerError
          expr: keda_scaler_errors_total > 0
          for: 5m
          annotations:
            summary: "KEDA scaler {{ $labels.scaler }} has errors"
          labels:
            severity: warning
```

---

## 📊 สรุป Part 95

| Technology | Use Case | Maturity |
|------------|----------|---------|
| WASM + RuntimeClass | Microsecond startup functions | Early adopter |
| Spin (Fermyon) | WASM microservices | Beta |
| K3s | Edge/IoT Kubernetes | Production |
| KubeEdge | IoT device management | Production |
| Knative Serving | Scale-to-zero serverless | Production |
| KEDA | Event-driven autoscaling | Production |
| OpenFaaS | Simple FaaS on K8s | Production |

---

## 🔗 ต่อไป
- [Part 96: Advanced Networking with eBPF and Cilium](./part-96-ebpf-cilium.md)

---
*Part 95 | Steps 911-920 | WebAssembly and Edge Computing | Educational Content*
