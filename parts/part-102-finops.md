# Part 102: Cost Optimization and FinOps
## Steps 981-990: Kubernetes Cost Management, Resource Optimization, FinOps

---

## Step 981: FinOps Overview

```
FinOps (Financial Operations) for Kubernetes:

Why Kubernetes Cost is Hard:
  - Resources shared across teams and namespaces
  - Pods can request more than they use (over-provisioning)
  - Persistent volumes charged even when unused
  - Data transfer costs hidden until billing shock
  - Multiple clouds/regions hard to reconcile

FinOps Maturity Model:
  Crawl (Visibility):
    - Know what you're spending
    - Resource requests vs actual usage reports
    - Cost per namespace/team
  
  Walk (Optimization):
    - Right-size pods (VPA recommendations)
    - Use Spot/Preemptible nodes for batch
    - Delete unused PVCs and LoadBalancers
    - Reserved instances for stable workloads
  
  Run (Accountability):
    - Chargeback/showback by team
    - Budget alerts and cost policies
    - Engineering teams own their cloud spend
    - Cost targets in OKRs

Key Kubernetes Cost Levers:
  1. Node sizing: right-size node pools
  2. Pod requests: match to actual usage
  3. Spot instances: 60-90% savings for tolerant workloads
  4. Reserved/Committed use: 30-50% savings for baseline
  5. PVC rightsizing: delete unused volumes
  6. Data transfer: keep traffic in same zone/region
  7. Autoscaling: scale to zero when idle
  8. Multi-tenancy: consolidate small clusters

Tools:
  OpenCost: CNCF cost allocation (open source)
  Kubecost: commercial with more features
  Goldilocks: VPA-based right-sizing recommendations
  Karpenter: intelligent node consolidation (AWS)
  KEDA: scale-to-zero event-driven
```

---

## Step 982: Resource Quotas and LimitRanges

```yaml
# ResourceQuota: limit total resources per namespace
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-alpha-quota
  namespace: team-alpha
spec:
  hard:
    requests.cpu: "20"
    requests.memory: 40Gi
    limits.cpu: "40"
    limits.memory: 80Gi
    pods: "50"
    services: "10"
    services.loadbalancers: "2"
    persistentvolumeclaims: "20"
    requests.storage: 500Gi
    gold.storageclass.storage.k8s.io/requests.storage: 100Gi

---
# LimitRange: defaults + max per container
apiVersion: v1
kind: LimitRange
metadata:
  name: team-alpha-limits
  namespace: team-alpha
spec:
  limits:
    - type: Container
      default:
        cpu: 500m
        memory: 512Mi
      defaultRequest:
        cpu: 100m
        memory: 128Mi
      max:
        cpu: "4"
        memory: 8Gi
      min:
        cpu: 10m
        memory: 16Mi
    - type: PersistentVolumeClaim
      max:
        storage: 100Gi
      min:
        storage: 1Gi
```

---

## Step 983: Vertical Pod Autoscaler (VPA)

```yaml
# VPA: auto right-size pods
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: payment-api-vpa
  namespace: production
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: payment-api
  updatePolicy:
    updateMode: "Off"  # Recommendation only
  resourcePolicy:
    containerPolicies:
      - containerName: api
        minAllowed:
          cpu: 50m
          memory: 64Mi
        maxAllowed:
          cpu: "2"
          memory: 2Gi
        controlledResources: [cpu, memory]

---
# Goldilocks: VPA recommendations dashboard
apiVersion: v1
kind: ConfigMap
metadata:
  name: goldilocks-usage
  namespace: production
data:
  setup: |
    # Label namespace to enable Goldilocks VPA recommendations
    # kubectl label namespace production goldilocks.fairwinds.com/enabled=true
    #
    # Access dashboard:
    # kubectl port-forward svc/goldilocks-dashboard -n goldilocks 8080:80
    #
    # Goldilocks shows:
    # - Current requests vs recommended
    # - Estimated monthly savings
    # - QoS class change impact
```

---

## Step 984: OpenCost Cost Allocation

```yaml
# OpenCost: CNCF cost allocation tool
apiVersion: apps/v1
kind: Deployment
metadata:
  name: opencost
  namespace: opencost
spec:
  selector:
    matchLabels:
      app: opencost
  template:
    metadata:
      labels:
        app: opencost
    spec:
      serviceAccountName: opencost
      containers:
        - name: opencost
          image: ghcr.io/opencost/opencost:1.108.0
          env:
            - name: PROMETHEUS_SERVER_ENDPOINT
              value: http://prometheus-operated.monitoring:9090
            - name: CLUSTER_ID
              value: production-us-east
          ports:
            - containerPort: 9003
              name: http
          resources:
            requests:
              cpu: 100m
              memory: 256Mi
            limits:
              cpu: 500m
              memory: 1Gi

---
apiVersion: v1
kind: ConfigMap
metadata:
  name: opencost-api-examples
  namespace: opencost
data:
  queries: |
    # Cost by namespace last 30 days:
    # GET /model/allocation?window=30d&aggregate=namespace
    #
    # Cost by label (team):
    # GET /model/allocation?window=30d&aggregate=label:team
    #
    # Top 10 expensive pods:
    # GET /model/top?window=7d&aggregate=pod&limit=10
```

---

## Step 985: Karpenter Cost Optimization

```yaml
# Karpenter: consolidate underutilized nodes
apiVersion: karpenter.sh/v1beta1
kind: NodePool
metadata:
  name: cost-optimized
spec:
  template:
    spec:
      nodeClassRef:
        apiVersion: karpenter.k8s.aws/v1beta1
        kind: EC2NodeClass
        name: default
      requirements:
        - key: karpenter.sh/capacity-type
          operator: In
          values: [spot, on-demand]
        - key: node.kubernetes.io/instance-type
          operator: In
          values:
            - m5.large
            - m5.xlarge
            - m5.2xlarge
            - m5a.large
            - m5a.xlarge
            - m6i.large
            - m6i.xlarge
  disruption:
    consolidationPolicy: WhenUnderutilized
    consolidateAfter: 30s
    expireAfter: 720h
    budgets:
      - nodes: 10%
        schedule: "0 9 * * mon-fri"
        duration: 8h

---
apiVersion: karpenter.k8s.aws/v1beta1
kind: EC2NodeClass
metadata:
  name: default
spec:
  amiFamily: Bottlerocket
  role: KarpenterNodeRole
  subnetSelectorTerms:
    - tags:
        karpenter.sh/discovery: production-cluster
  securityGroupSelectorTerms:
    - tags:
        karpenter.sh/discovery: production-cluster
  blockDeviceMappings:
    - deviceName: /dev/xvda
      ebs:
        volumeSize: 50Gi
        volumeType: gp3
        iops: 3000
        encrypted: true
```

---

## Step 986: Scale-to-Zero with KEDA

```yaml
# KEDA: scale batch workloads to zero when idle
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: batch-processor-scaledobject
  namespace: production
spec:
  scaleTargetRef:
    name: batch-processor
  minReplicaCount: 0
  maxReplicaCount: 50
  cooldownPeriod: 300
  triggers:
    - type: aws-sqs-queue
      authenticationRef:
        name: keda-aws-credentials
      metadata:
        queueURL: https://sqs.us-east-1.amazonaws.com/123456789/batch-jobs
        queueLength: "5"
        awsRegion: us-east-1

---
# CronJob: delete completed jobs to save resources
apiVersion: batch/v1
kind: CronJob
metadata:
  name: cleanup-completed-jobs
  namespace: production
spec:
  schedule: "0 2 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: cleanup
              image: bitnami/kubectl:latest
              command:
                - /bin/sh
                - -c
                - |
                  kubectl delete jobs \
                    --field-selector=status.conditions[0].type=Complete \
                    --all-namespaces
                  kubectl delete pods \
                    --field-selector=status.phase=Succeeded \
                    --all-namespaces
```

---

## Step 987: Namespace Cost Labels

```yaml
# Label namespaces for cost allocation
apiVersion: v1
kind: Namespace
metadata:
  name: payment-service
  labels:
    team: payments
    environment: production
    cost-center: "CC-1001"
    product: payment-api

---
# Kyverno: enforce cost labels on namespace
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-cost-labels
spec:
  validationFailureAction: Enforce
  rules:
    - name: check-namespace-labels
      match:
        any:
          - resources:
              kinds: [Namespace]
      validate:
        message: "Namespace must have team, cost-center labels"
        pattern:
          metadata:
            labels:
              team: "?*"
              cost-center: "?*"
```

---

## Step 988: Unused Resource Cleanup

```yaml
# CronJob: report unused PVCs weekly
apiVersion: batch/v1
kind: CronJob
metadata:
  name: unused-pvc-reporter
  namespace: kube-system
spec:
  schedule: "0 6 * * 1"
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          serviceAccountName: pvc-auditor
          containers:
            - name: auditor
              image: bitnami/kubectl:latest
              command:
                - /bin/sh
                - -c
                - |
                  echo "=== Unused PVCs (Released/Available) ==="
                  kubectl get pvc --all-namespaces \
                    -o custom-columns='NS:.metadata.namespace,NAME:.metadata.name,STATUS:.status.phase,SIZE:.spec.resources.requests.storage' \
                    | grep -E "(Released|Available)"
                  
                  echo "=== PVCs not mounted to any pod ==="
                  kubectl get pvc --all-namespaces -o json \
                    | jq -r '.items[] | select(.status.phase == "Bound") | "\(.metadata.namespace)/\(.metadata.name)"'
```

---

## Step 989: Cost Optimization Policies

```yaml
# Kyverno: require resource requests/limits
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-resource-limits
spec:
  validationFailureAction: Enforce
  rules:
    - name: check-container-resources
      match:
        any:
          - resources:
              kinds: [Pod]
      validate:
        message: "All containers must have CPU and memory requests/limits"
        foreach:
          - list: "request.object.spec.containers"
            deny:
              conditions:
                any:
                  - key: "{{ element.resources.requests.cpu || '' }}"
                    operator: Equals
                    value: ""
                  - key: "{{ element.resources.requests.memory || '' }}"
                    operator: Equals
                    value: ""
                  - key: "{{ element.resources.limits.memory || '' }}"
                    operator: Equals
                    value: ""
```

---

## Step 990: Workshop - FinOps Dashboard

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: finops-alerts
  namespace: monitoring
spec:
  groups:
    - name: finops
      rules:
        - alert: HighCPUUnderutilization
          expr: |
            (
              sum(rate(container_cpu_usage_seconds_total{container!="",namespace=~"production|staging"}[1h])) by (namespace, pod, container)
              /
              sum(kube_pod_container_resource_requests{resource="cpu",namespace=~"production|staging"}) by (namespace, pod, container)
            ) < 0.2
          for: 24h
          annotations:
            summary: "Pod {{ $labels.pod }} using < 20% of CPU request for 24h"
          labels:
            severity: info

        - alert: HighMemoryUnderutilization
          expr: |
            (
              sum(container_memory_working_set_bytes{container!="",namespace=~"production|staging"}) by (namespace, pod, container)
              /
              sum(kube_pod_container_resource_requests{resource="memory",namespace=~"production|staging"}) by (namespace, pod, container)
            ) < 0.3
          for: 24h
          annotations:
            summary: "Pod {{ $labels.pod }} using < 30% of memory request for 24h"
          labels:
            severity: info

        - alert: UnusedPVCDetected
          expr: |
            kube_persistentvolumeclaim_status_phase{phase="Lost"} == 1
          for: 1h
          annotations:
            summary: "PVC {{ $labels.persistentvolumeclaim }} in Lost state"
          labels:
            severity: warning

        - alert: NamespaceMissingCostLabels
          expr: |
            kube_namespace_labels{label_cost_center=""} == 1
          for: 0m
          annotations:
            summary: "Namespace {{ $labels.namespace }} missing cost-center label"
          labels:
            severity: info
```

---

## 📊 สรุป Part 102

| Strategy | Savings | Effort | Risk |
|----------|---------|--------|------|
| Spot instances | 60-90% | Low | Medium |
| Right-size pods (VPA) | 30-50% | Medium | Low |
| Reserved instances | 30-50% | Low | Low |
| Scale-to-zero (KEDA) | 100% idle | Medium | Medium |
| Delete unused PVCs | Variable | Low | Low |
| Node consolidation (Karpenter) | 20-40% | Low | Low |
| Multi-tenancy | 40-60% | High | Medium |

---

## 🔗 ต่อไป
- [Part 103: Capstone - Production-Ready Architecture](./part-103-capstone.md)

---
*Part 102 | Steps 981-990 | Cost Optimization and FinOps | Educational Content*
