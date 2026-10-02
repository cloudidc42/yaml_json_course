# Part 76: Kubernetes Autoscaling
## Steps 721-730: HPA, VPA, KEDA, Karpenter

---

## Step 721: Autoscaling Overview

```
Kubernetes Autoscaling Dimensions:

Horizontal Pod Autoscaler (HPA):
  - Scale pod replicas based on metrics
  - CPU/memory (built-in)
  - Custom metrics (Prometheus, KEDA)
  - External metrics (queue depth, etc.)

Vertical Pod Autoscaler (VPA):
  - Adjust CPU/memory requests/limits
  - Three modes: Off, Initial, Auto
  - Do NOT use HPA + VPA on same deployment (conflict)
  - VPA + HPA: OK if HPA uses custom metrics (not CPU/mem)

Cluster Autoscaler (CA):
  - Add/remove nodes based on pending pods
  - Works with cloud node groups
  - Slow (1-2 min to provision node)

Karpenter:
  - Next-gen node autoscaler (AWS, Azure)
  - Faster than CA (seconds)
  - Right-sizes nodes to pod requirements
  - Supports spot instances, diverse instance types

KEDA:
  - Event-driven autoscaling
  - Scale to zero
  - 60+ scalers (Kafka, Redis, RabbitMQ, SQS, etc.)
```

---

## Step 722: HPA - CPU and Memory

```yaml
# HPA: scale based on CPU
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 2
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: AverageValue
          averageValue: 512Mi
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
        - type: Pods
          value: 4
          periodSeconds: 60
        - type: Percent
          value: 100
          periodSeconds: 60
      selectPolicy: Max
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Pods
          value: 1
          periodSeconds: 60
      selectPolicy: Min

---
# Deployment: must have resource requests for HPA to work
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: production
spec:
  replicas: 2
  template:
    spec:
      containers:
        - name: app
          image: myapp:1.0
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 500m
              memory: 512Mi
```

---

## Step 723: HPA - Custom Metrics (Prometheus)

```yaml
# Prometheus Adapter: expose custom metrics to HPA
# values.yaml for prometheus-adapter Helm chart
rules:
  custom:
    - seriesQuery: 'http_requests_total{namespace!="",pod!=""}'
      resources:
        overrides:
          namespace: {resource: "namespace"}
          pod: {resource: "pod"}
      name:
        matches: "^(.*)_total$"
        as: "${1}_per_second"
      metricsQuery: 'sum(rate(<<.Series>>{<<.LabelMatchers>>}[2m])) by (<<.GroupBy>>)'

---
# HPA using custom metrics from Prometheus
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-custom-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api
  minReplicas: 2
  maxReplicas: 50
  metrics:
    - type: Pods
      pods:
        metric:
          name: http_requests_per_second
        target:
          type: AverageValue
          averageValue: 100
    - type: External
      external:
        metric:
          name: queue_depth
          selector:
            matchLabels:
              queue: api-requests
        target:
          type: Value
          value: 1000
```

---

## Step 724: VPA - Vertical Pod Autoscaler

```yaml
# VPA in Recommendation mode
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: myapp-vpa
  namespace: production
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  updatePolicy:
    updateMode: "Off"  # Only recommend, don't apply

---
# VPA in Auto mode
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: batch-job-vpa
  namespace: batch
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: batch-processor
  updatePolicy:
    updateMode: "Auto"
    minReplicas: 1
  resourcePolicy:
    containerPolicies:
      - containerName: processor
        minAllowed:
          cpu: 100m
          memory: 128Mi
        maxAllowed:
          cpu: 4000m
          memory: 8Gi
        controlledResources: ["cpu", "memory"]
        controlledValues: RequestsAndLimits

---
# VPA in Initial mode (set on pod creation only)
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: statefulset-vpa
  namespace: databases
spec:
  targetRef:
    apiVersion: apps/v1
    kind: StatefulSet
    name: postgres
  updatePolicy:
    updateMode: "Initial"
```

---

## Step 725: KEDA - Event-Driven Autoscaling

```yaml
# KEDA: scale based on Prometheus metric
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: api-keda
  namespace: production
spec:
  scaleTargetRef:
    name: api
  pollingInterval: 30
  cooldownPeriod: 300
  minReplicaCount: 0
  maxReplicaCount: 100
  triggers:
    - type: prometheus
      metadata:
        serverAddress: http://prometheus.monitoring.svc:9090
        metricName: http_requests_per_second
        threshold: "100"
        query: |
          sum(rate(http_requests_total{app="api"}[2m]))

---
# KEDA: scale based on AWS SQS queue
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: worker-sqs-keda
  namespace: production
spec:
  scaleTargetRef:
    name: sqs-worker
  minReplicaCount: 0
  maxReplicaCount: 50
  triggers:
    - type: aws-sqs-queue
      authenticationRef:
        name: aws-credentials
      metadata:
        queueURL: https://sqs.us-east-1.amazonaws.com/123456789/my-queue
        queueLength: "5"
        awsRegion: us-east-1

---
# KEDA: scale based on Kafka topic lag
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: kafka-consumer-keda
  namespace: production
spec:
  scaleTargetRef:
    name: kafka-consumer
  minReplicaCount: 1
  maxReplicaCount: 30
  triggers:
    - type: kafka
      metadata:
        bootstrapServers: kafka.kafka.svc:9092
        consumerGroup: my-consumer-group
        topic: my-topic
        lagThreshold: "100"

---
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: aws-credentials
  namespace: production
spec:
  podIdentity:
    provider: aws
```

---

## Step 726: Karpenter - Node Autoscaling

```yaml
# NodePool: define node requirements
apiVersion: karpenter.sh/v1beta1
kind: NodePool
metadata:
  name: default
spec:
  template:
    spec:
      requirements:
        - key: karpenter.k8s.aws/instance-category
          operator: In
          values: [c, m, r]
        - key: karpenter.k8s.aws/instance-cpu
          operator: In
          values: ["4", "8", "16", "32"]
        - key: karpenter.io/arch
          operator: In
          values: [amd64, arm64]
        - key: karpenter.sh/capacity-type
          operator: In
          values: [spot, on-demand]
      nodeClassRef:
        name: default
      expireAfter: 720h
  disruption:
    consolidationPolicy: WhenUnderutilized
    consolidateAfter: 30s
  limits:
    cpu: 1000
    memory: 4000Gi

---
# EC2NodeClass: AWS-specific node configuration
apiVersion: karpenter.k8s.aws/v1beta1
kind: EC2NodeClass
metadata:
  name: default
spec:
  amiFamily: Bottlerocket
  subnetSelectorTerms:
    - tags:
        karpenter.sh/discovery: my-cluster
  securityGroupSelectorTerms:
    - tags:
        karpenter.sh/discovery: my-cluster
  instanceProfile: KarpenterNodeInstanceProfile
  blockDeviceMappings:
    - deviceName: /dev/xvda
      ebs:
        volumeSize: 100Gi
        volumeType: gp3
        iops: 3000
        encrypted: true
  tags:
    Environment: production
    ManagedBy: karpenter

---
# NodePool for GPU workloads
apiVersion: karpenter.sh/v1beta1
kind: NodePool
metadata:
  name: gpu-pool
spec:
  template:
    metadata:
      labels:
        node-type: gpu
    spec:
      requirements:
        - key: karpenter.k8s.aws/instance-gpu-name
          operator: In
          values: [t4, a10g, v100]
        - key: karpenter.sh/capacity-type
          operator: In
          values: [on-demand]
      taints:
        - key: nvidia.com/gpu
          effect: NoSchedule
      nodeClassRef:
        name: gpu
  limits:
    cpu: 200
```

---

## Step 727: Cluster Autoscaler

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cluster-autoscaler
  namespace: kube-system
spec:
  template:
    spec:
      containers:
        - name: cluster-autoscaler
          image: registry.k8s.io/autoscaling/cluster-autoscaler:v1.28.2
          command:
            - ./cluster-autoscaler
            - --cloud-provider=aws
            - --node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled,k8s.io/cluster-autoscaler/my-cluster
            - --balance-similar-node-groups
            - --skip-nodes-with-system-pods=false
            - --scale-down-delay-after-add=10m
            - --scale-down-unneeded-time=10m
            - --scale-down-unready-time=20m
            - --max-node-provision-time=15m
            - --ok-total-unready-count=3
          env:
            - name: AWS_REGION
              value: us-east-1

---
# Protect pods from CA scale-down
apiVersion: v1
kind: Pod
metadata:
  name: protected-pod
  annotations:
    cluster-autoscaler.kubernetes.io/safe-to-evict: "false"
spec:
  containers:
    - name: app
      image: myapp:1.0
      volumeMounts:
        - name: local-cache
          mountPath: /cache
  volumes:
    - name: local-cache
      emptyDir:
        medium: Memory
```

---

## Step 728: Autoscaling - Cron and Scheduled

```yaml
# KEDA: scale based on cron schedule
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: batch-cron-scaler
  namespace: production
spec:
  scaleTargetRef:
    name: batch-worker
  minReplicaCount: 0
  maxReplicaCount: 20
  triggers:
    - type: cron
      metadata:
        timezone: Asia/Bangkok
        start: "0 8 * * 1-5"
        end: "0 18 * * 1-5"
        desiredReplicas: "10"
    - type: cron
      metadata:
        timezone: Asia/Bangkok
        start: "0 10 * * 1-5"
        end: "0 12 * * 1-5"
        desiredReplicas: "20"

---
# CronJob-based scaling (no KEDA required)
apiVersion: batch/v1
kind: CronJob
metadata:
  name: scale-up-morning
  namespace: production
spec:
  schedule: "0 7 * * 1-5"
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: scaler-sa
          containers:
            - name: scaler
              image: bitnami/kubectl:latest
              command:
                - kubectl
                - scale
                - deployment/api
                - --replicas=20
                - -n
                - production
          restartPolicy: OnFailure
```

---

## Step 729: PodDisruptionBudget with Autoscaling

```yaml
# PDB: protect app during scale-down
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: myapp-pdb
  namespace: production
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: myapp

---
# maxUnavailable approach
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: critical-service-pdb
  namespace: production
spec:
  maxUnavailable: 1
  selector:
    matchLabels:
      app: critical-service

---
# PDB for StatefulSet
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: postgres-pdb
  namespace: databases
spec:
  minAvailable: "50%"
  selector:
    matchLabels:
      app: postgres
```

---

## Step 730: Workshop - Autoscaling Best Practices

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: autoscaling-best-practices
  namespace: production
data:
  practices.yaml: |
    hpa:
      - Always set resource requests (required for metrics)
      - Use stabilizationWindowSeconds to prevent flapping
      - Set minReplicas >= 2 for HA
      - Use custom metrics for better scaling signals
      - Test behavior under load before production
    
    vpa:
      - Use Off mode first (get recommendations)
      - Never combine VPA Auto + HPA CPU/memory
      - VPA + HPA OK with custom metrics (KEDA)
      - Set min/maxAllowed to prevent runaway resources
    
    keda:
      - Best for event-driven workloads
      - Scale to zero for batch/dev workloads
      - Use TriggerAuthentication for credentials
      - Test scale-to-zero cold start latency
    
    karpenter:
      - Use diverse instance types (avoid stockouts)
      - Mix spot/on-demand per workload criticality
      - Set expireAfter for node rotation
      - Use Bottlerocket for security
    
    pdb:
      - Always set PDB for production deployments
      - PDB required for cluster autoscaler to respect
      - Use minAvailable: 1 minimum
      - Test drain behavior regularly

---
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: autoscaling-alerts
  namespace: monitoring
spec:
  groups:
    - name: autoscaling
      rules:
        - alert: HPAMaxedOut
          expr: |
            kube_horizontalpodautoscaler_status_current_replicas
            ==
            kube_horizontalpodautoscaler_spec_max_replicas
          for: 15m
          annotations:
            summary: "HPA {{ $labels.horizontalpodautoscaler }} is at max replicas for 15m"
          labels:
            severity: warning

        - alert: PodsWaitingForNodes
          expr: |
            count(kube_pod_status_phase{phase="Pending"}) > 5
          for: 10m
          annotations:
            summary: "More than 5 pods pending for 10m - check node autoscaling"
          labels:
            severity: warning
```

---

## 📊 สรุป Part 76

| Tool | Scale Target | Trigger | Scale to Zero |
|------|-------------|---------|---------------|
| HPA | Pods | CPU/mem/custom | No |
| VPA | Pod resources | Historical usage | N/A |
| KEDA | Pods | Events/queues | Yes |
| CA | Nodes | Pending pods | No |
| Karpenter | Nodes | Pending pods | No |

---

## 🔗 ต่อไป
- [Part 77: Kubernetes Observability](./part-77-observability.md)

---
*Part 76 | Steps 721-730 | Autoscaling | Educational Content*
