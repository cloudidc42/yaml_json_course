# Part 86: FinOps and Cost Optimization
## Steps 821-830: Kubecost, Spot Instances, Right-Sizing

---

## Step 821: FinOps Overview

```
FinOps for Kubernetes:

Core Principles (FinOps Foundation):
  Inform: visibility into cloud spend
  Optimize: eliminate waste, right-size
  Operate: build cost culture

Cost Drivers in Kubernetes:
  Compute (EC2/GCE/AKS nodes):
    - Over-provisioned instances
    - Idle dev/test clusters
    - Wrong instance types
  
  Storage:
    - Unused PersistentVolumes
    - Unattached EBS volumes
    - Over-sized PVC requests
  
  Networking:
    - Cross-AZ data transfer
    - NAT Gateway charges
    - LoadBalancer per service

Cost Allocation:
  Labels: team, env, app, cost-center
  Namespace == team == cost center
  Chargeback vs Showback models

Tools:
  Kubecost: per-namespace/workload cost breakdown
  Infracost: IaC cost estimation
  AWS Cost Explorer + Container Insights
  OpenCost: CNCF open-source cost monitoring
  Goldilocks: VPA-based right-sizing recommendations
```

---

## Step 822: Kubecost Setup

```yaml
# Kubecost Helm values.yaml
kubecostToken: ""

prometheus:
  server:
    global:
      external_labels:
        cluster_id: production

kubecostProductConfigs:
  clusterName: production
  labelMappingConfigs:
    enabled: true
    owner_label: team
    team_label: team
    department_label: department
    product_label: app
    environment_label: environment

savings:
  enabled: true
  requestSizing:
    enabled: true
    targetCPUUtilization: 0.65
    targetMemUtilization: 0.65

---
# Namespace: add cost labels
apiVersion: v1
kind: Namespace
metadata:
  name: team-alpha
  labels:
    team: alpha
    department: engineering
    environment: production
    cost-center: cc-1234
```

---

## Step 823: Spot Instance Handling

```yaml
# Karpenter NodePool: mixed spot/on-demand
apiVersion: karpenter.sh/v1beta1
kind: NodePool
metadata:
  name: general-workloads
spec:
  template:
    spec:
      requirements:
        - key: karpenter.sh/capacity-type
          operator: In
          values: [spot, on-demand]
        
        - key: node.kubernetes.io/instance-type
          operator: In
          values:
            - m5.large
            - m5.xlarge
            - m5a.large
            - m5a.xlarge
            - m6i.large
            - m6i.xlarge
        
        - key: topology.kubernetes.io/zone
          operator: In
          values: [us-east-1a, us-east-1b, us-east-1c]
  
  disruption:
    consolidationPolicy: WhenUnderutilized
    consolidateAfter: 30s
    expireAfter: 720h

---
# Spot interruption handler DaemonSet
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: aws-node-termination-handler
  namespace: kube-system
spec:
  selector:
    matchLabels:
      app: aws-node-termination-handler
  template:
    spec:
      serviceAccountName: aws-node-termination-handler
      containers:
        - name: aws-node-termination-handler
          image: public.ecr.aws/aws-ec2/aws-node-termination-handler:v1.21
          env:
            - name: ENABLE_SPOT_INTERRUPTION_DRAINING
              value: "true"
            - name: ENABLE_SCHEDULED_EVENT_DRAINING
              value: "true"
            - name: DRAIN_GRACE_PERIOD
              value: "120"
            - name: NODE_NAME
              valueFrom:
                fieldRef:
                  fieldPath: spec.nodeName
      tolerations:
        - operator: Exists
```

---

## Step 824: Workload Scheduling for Cost

```yaml
# Prefer spot for batch workloads
apiVersion: apps/v1
kind: Deployment
metadata:
  name: batch-processor
  namespace: processing
spec:
  template:
    spec:
      affinity:
        nodeAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              preference:
                matchExpressions:
                  - key: karpenter.sh/capacity-type
                    operator: In
                    values: [spot]
      
      terminationGracePeriodSeconds: 120
      containers:
        - name: processor
          image: batch-processor:1.0
          lifecycle:
            preStop:
              exec:
                command: ["/bin/sh", "-c", "save-checkpoint.sh && sleep 5"]
          env:
            - name: CHECKPOINT_INTERVAL
              value: "30"

---
# Force on-demand for stateful workloads
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: databases
spec:
  template:
    spec:
      nodeSelector:
        karpenter.sh/capacity-type: on-demand
```

---

## Step 825: Right-Sizing Workflow

```yaml
# VPA right-sizing process
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
    updateMode: "Off"  # Observe only for 7-14 days

# Check recommendations:
# kubectl get vpa myapp-vpa -n production -o json | jq \
#   '.status.recommendation.containerRecommendations[0]'
#
# Output:
# {
#   "target": {"cpu": "87m", "memory": "242Mi"},
#   "upperBound": {"cpu": "150m", "memory": "400Mi"},
#   "lowerBound": {"cpu": "50m", "memory": "128Mi"}
# }
#
# Apply:
# kubectl set resources deployment myapp \
#   --requests=cpu=87m,memory=242Mi \
#   --limits=cpu=150m,memory=400Mi \
#   -n production

---
apiVersion: v1
kind: ConfigMap
metadata:
  name: rightsizing-schedule
data:
  schedule.md: |
    Right-sizing Cadence:
    - Week 1-2: Deploy VPA Off mode
    - Week 3: Review recommendations
    - Week 4: Apply to 10% of workloads
    - Month 2: Apply to all non-critical
    - Quarterly: Review and re-apply
    
    Expected savings: 30-50% compute cost
```

---

## Step 826: Node Consolidation

```yaml
# Karpenter: aggressive consolidation config
apiVersion: karpenter.sh/v1beta1
kind: NodePool
metadata:
  name: consolidating-pool
spec:
  disruption:
    consolidationPolicy: WhenUnderutilized
    consolidateAfter: 30s
    budgets:
      - nodes: "10%"
  limits:
    cpu: 1000
    memory: 2000Gi

---
# Pods that allow eviction for consolidation
apiVersion: apps/v1
kind: Deployment
metadata:
  name: evictable-worker
spec:
  template:
    metadata:
      annotations:
        cluster-autoscaler.kubernetes.io/safe-to-evict: "true"
    spec:
      containers:
        - name: worker
          image: worker:1.0
          resources:
            requests:
              cpu: 100m
              memory: 128Mi

---
# PodDisruptionBudget: allow consolidation
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: worker-pdb
spec:
  maxUnavailable: 50%   # Allow 50% down during consolidation
  selector:
    matchLabels:
      app: evictable-worker
```

---

## Step 827: Storage Cost Optimization

```yaml
# gp3 StorageClass: cheaper + faster than gp2
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"

---
# Loki retention policy
apiVersion: v1
kind: ConfigMap
metadata:
  name: loki-config
data:
  loki.yaml: |
    compactor:
      retention_enabled: true
      retention_delete_delay: 2h
    limits_config:
      retention_period: 30d

---
# Find orphaned PVs:
# kubectl get pv -o json | jq '.items[] |
#   select(.status.phase == "Released") |
#   {name: .metadata.name, size: .spec.capacity.storage}'
#
# Delete orphaned PVs:
# kubectl patch pv <pv-name> -p '{"spec":{"persistentVolumeReclaimPolicy":"Delete"}}'
```

---

## Step 828: Network Cost Optimization

```yaml
# Topology-aware routing: prefer same-zone
apiVersion: v1
kind: Service
metadata:
  name: myapp
  namespace: production
  annotations:
    service.kubernetes.io/topology-mode: "auto"
spec:
  selector:
    app: myapp
  ports:
    - port: 80

---
# Shared ALB: one LB for multiple services
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: shared-alb
  namespace: production
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/group.name: production
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS":443}]'
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:us-east-1:123:certificate/xxx
spec:
  rules:
    - host: myapp.mycompany.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: myapp
                port:
                  number: 80
    - host: api.mycompany.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 80
```

---

## Step 829: Cost Governance

```yaml
# Kyverno: require cost labels
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-cost-labels
spec:
  validationFailureAction: Enforce
  rules:
    - name: check-cost-labels
      match:
        any:
          - resources:
              kinds: [Namespace]
      validate:
        message: "Namespaces must have labels: team, environment"
        pattern:
          metadata:
            labels:
              team: "?*"
              environment: "?*"

---
# Kyverno: restrict LoadBalancer services
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: restrict-load-balancer
spec:
  validationFailureAction: Enforce
  rules:
    - name: no-loadbalancer-without-approval
      match:
        any:
          - resources:
              kinds: [Service]
      validate:
        message: "LoadBalancer Services require annotation cost.mycompany.com/loadbalancer-approved: 'true'"
        deny:
          conditions:
            all:
              - key: "{{ request.object.spec.type }}"
                operator: Equals
                value: LoadBalancer
              - key: "{{ request.object.metadata.annotations.\"cost.mycompany.com/loadbalancer-approved\" || '' }}"
                operator: NotEquals
                value: "true"
```

---

## Step 830: Workshop - Cost Alerts

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: cost-alerts
  namespace: monitoring
spec:
  groups:
    - name: finops
      rules:
        - alert: HighCPUWaste
          expr: |
            (
              kube_pod_container_resource_requests{resource="cpu"}
              - on(pod, namespace, container) group_left
              rate(container_cpu_usage_seconds_total[24h])
            ) / kube_pod_container_resource_requests{resource="cpu"}
            > 0.7
          for: 1d
          annotations:
            summary: "Container {{ $labels.container }} in {{ $labels.namespace }} wastes > 70% requested CPU"
          labels:
            severity: info

        - alert: UnusedPersistentVolume
          expr: |
            kube_persistentvolume_status_phase{phase="Released"} == 1
          for: 1h
          annotations:
            summary: "PV {{ $labels.persistentvolume }} is Released (orphaned)"
          labels:
            severity: warning

        - alert: HighSpotInterruptionRate
          expr: |
            rate(node_termination_handler_actions_total{action="cordon-and-drain"}[1h]) > 0.1
          for: 0m
          annotations:
            summary: "High spot interruption rate in cluster"
          labels:
            severity: warning
```

---

## 📊 สรุป Part 86

| Area | Technique | Typical Savings |
|------|-----------|----------------|
| Compute | Spot instances | 60-80% vs on-demand |
| Compute | Right-sizing (VPA) | 30-50% |
| Compute | Node consolidation | 20-40% |
| Storage | gp3 vs gp2 | 20% |
| Storage | Lifecycle policies | 40-70% |
| Network | Topology-aware routing | 5-20% |
| Network | Shared ALB/NLB | Per-LB cost |

---

## 🔗 ต่อไป
- [Part 87: API Gateway and Microservices](./part-87-api-gateway.md)

---
*Part 86 | Steps 821-830 | FinOps and Cost Optimization | Educational Content*
