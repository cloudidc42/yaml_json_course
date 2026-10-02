# Part 35: KEDA Event-Driven Autoscaling
## Steps 331-340: Scale ด้วย External Events

---

## 📖 บทนำ

KEDA (Kubernetes Event-Driven Autoscaling) ขยายความสามารถ HPA ให้ scale based on external metrics เช่น queue length, Kafka lag, database connections

---

## Step 331: KEDA Architecture

```bash
# ติดตั้ง KEDA
# helm repo add kedacore https://kedacore.github.io/charts
# helm install keda kedacore/keda --namespace keda --create-namespace
```

```
KEDA Components:
- Operator: จัดการ ScaledObject/ScaledJob
- Metrics Adapter: expose metrics เป็น custom metrics API
- Scalers: connector กับ external systems
- HPA: Kubernetes native autoscaler (บริหารโดย KEDA)

Supported Scalers: Kafka, RabbitMQ, AWS SQS,
Azure Service Bus, GCP Pub/Sub, Redis, PostgreSQL,
Prometheus, Datadog, Cron, HTTP, and 50+ more
```

---

## Step 332: ScaledObject - Kafka

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: kafka-consumer-scaler
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: kafka-consumer
  
  pollingInterval: 15
  cooldownPeriod: 300
  idleReplicaCount: 0   # scale to zero
  minReplicaCount: 1
  maxReplicaCount: 50
  
  triggers:
    - type: kafka
      metadata:
        bootstrapServers: kafka.kafka.svc.cluster.local:9092
        consumerGroup: my-consumer-group
        topic: orders
        lagThreshold: "50"
        offsetResetPolicy: latest
      authenticationRef:
        name: kafka-auth

---
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: kafka-auth
  namespace: production
spec:
  secretTargetRef:
    - parameter: username
      name: kafka-credentials
      key: username
    - parameter: password
      name: kafka-credentials
      key: password
```

---

## Step 333: ScaledObject - AWS SQS

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: sqs-scaler
  namespace: production
spec:
  scaleTargetRef:
    name: sqs-worker
  minReplicaCount: 0
  maxReplicaCount: 100
  pollingInterval: 30
  cooldownPeriod: 60
  
  triggers:
    - type: aws-sqs-queue
      metadata:
        queueURL: https://sqs.ap-southeast-1.amazonaws.com/123456789/my-queue
        queueLength: "5"
        awsRegion: ap-southeast-1
        identityOwner: pod  # ใช้ IRSA
        scaleOnInFlight: "true"

---
apiVersion: keda.sh/v1alpha1
kind: ClusterTriggerAuthentication
metadata:
  name: aws-iam-auth
spec:
  podIdentity:
    provider: aws
```

---

## Step 334: ScaledObject - Redis

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: redis-scaler
  namespace: production
spec:
  scaleTargetRef:
    name: redis-worker
  minReplicaCount: 0
  maxReplicaCount: 20
  
  triggers:
    - type: redis
      metadata:
        address: redis-master.production.svc.cluster.local:6379
        listName: job-queue
        listLength: "10"
        dbIndex: "0"
      authenticationRef:
        name: redis-auth

    - type: redis-streams
      metadata:
        address: redis-master.production.svc.cluster.local:6379
        stream: mystream
        consumerGroup: my-group
        pendingEntriesCount: "20"

---
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: redis-auth
  namespace: production
spec:
  secretTargetRef:
    - parameter: password
      name: redis-secret
      key: password
```

---

## Step 335: ScaledObject - Prometheus

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: prometheus-scaler
  namespace: production
spec:
  scaleTargetRef:
    name: api-worker
  minReplicaCount: 2
  maxReplicaCount: 20
  
  triggers:
    - type: prometheus
      metadata:
        serverAddress: http://prometheus-operated.monitoring:9090
        metricName: http_requests_total
        threshold: "100"
        query: |
          sum(rate(http_requests_total{job="api-worker"}[2m]))
    
    - type: prometheus
      metadata:
        serverAddress: http://prometheus-operated.monitoring:9090
        metricName: pending_jobs
        threshold: "50"
        query: |
          sum(pending_jobs{service="api-worker"})
        activationThreshold: "5"
```

---

## Step 336: ScaledJob

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledJob
metadata:
  name: image-processor
  namespace: production
spec:
  jobTargetRef:
    parallelism: 1
    completions: 1
    activeDeadlineSeconds: 600
    backoffLimit: 3
    template:
      spec:
        restartPolicy: Never
        containers:
          - name: processor
            image: myregistry.io/image-processor:1.0
            resources:
              requests:
                cpu: 500m
                memory: 512Mi
              limits:
                cpu: 2000m
                memory: 2Gi
  
  pollingInterval: 30
  maxReplicaCount: 50
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 5
  
  scalingStrategy:
    strategy: "accurate"
  
  triggers:
    - type: aws-sqs-queue
      metadata:
        queueURL: https://sqs.ap-southeast-1.amazonaws.com/123456789/image-jobs
        queueLength: "1"
        awsRegion: ap-southeast-1
        identityOwner: pod
```

---

## Step 337: HTTP Add-On

```yaml
apiVersion: http.keda.sh/v1alpha1
kind: HTTPScaledObject
metadata:
  name: my-app
  namespace: production
spec:
  hosts:
    - myapp.example.com
  pathPrefixes:
    - /api
  scaledownPeriod: 300
  scaleTargetRef:
    apiVersion: apps/v1
    deployment: my-app
    service: my-app
    port: 80
  replicas:
    min: 0
    max: 50
  targetPendingRequests: 100
```

---

## Step 338-339: Cron + Fallback

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: cron-scaler
  namespace: production
spec:
  scaleTargetRef:
    name: my-app
  minReplicaCount: 2
  maxReplicaCount: 20
  
  fallback:
    failureThreshold: 3
    replicas: 5  # ถ้า scaler fail ให้ scale to 5
  
  triggers:
    - type: cron
      metadata:
        timezone: Asia/Bangkok
        start: "0 7 * * 1-5"
        end: "0 22 * * 1-5"
        desiredReplicas: "10"
    
    - type: cron
      metadata:
        timezone: Asia/Bangkok
        start: "0 11 * * 1-5"
        end: "0 14 * * 1-5"
        desiredReplicas: "20"
    
    - type: prometheus
      metadata:
        serverAddress: http://prometheus-operated.monitoring:9090
        query: sum(rate(http_requests_total[2m]))
        threshold: "50"
```

---

## Step 340: Workshop

```yaml
# E-commerce complete KEDA setup

# 1. Order processor
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: order-processor
  namespace: ecommerce
spec:
  scaleTargetRef:
    name: order-processor
  minReplicaCount: 0
  maxReplicaCount: 100
  cooldownPeriod: 300
  triggers:
    - type: kafka
      metadata:
        bootstrapServers: kafka.kafka:9092
        consumerGroup: order-processors
        topic: new-orders
        lagThreshold: "10"

---
# 2. Image resizer (batch jobs)
apiVersion: keda.sh/v1alpha1
kind: ScaledJob
metadata:
  name: image-resizer
  namespace: ecommerce
spec:
  jobTargetRef:
    template:
      spec:
        restartPolicy: Never
        containers:
          - name: resizer
            image: myregistry.io/image-resizer:1.0
  maxReplicaCount: 20
  pollingInterval: 15
  triggers:
    - type: aws-sqs-queue
      metadata:
        queueURL: https://sqs.ap-southeast-1.amazonaws.com/123/image-jobs
        queueLength: "1"
        awsRegion: ap-southeast-1
        identityOwner: pod

---
# 3. Business hours
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: api-server-cron
  namespace: ecommerce
spec:
  scaleTargetRef:
    name: api-server
  minReplicaCount: 2
  maxReplicaCount: 30
  triggers:
    - type: cron
      metadata:
        timezone: Asia/Bangkok
        start: "0 8 * * 1-5"
        end: "0 20 * * 1-5"
        desiredReplicas: "10"
    - type: prometheus
      metadata:
        serverAddress: http://prometheus-operated.monitoring:9090
        query: sum(rate(http_requests_total{app="api-server"}[2m]))
        threshold: "100"
```

---

## 📊 สรุป Part 35

| Scaler | Use Case |
|--------|----------|
| Kafka | Consumer lag-based |
| AWS SQS | Queue depth |
| Redis | List/Stream length |
| Prometheus | Custom metrics |
| Cron | Schedule-based |
| HTTP | Pending requests |
| ScaledJob | Batch jobs |

---

## 🔗 ต่อไป
- [Part 36: Network Policies Deep Dive](./part-36-network-policies.md)

---
*Part 35 | Steps 331-340 | ระดับสูง*
