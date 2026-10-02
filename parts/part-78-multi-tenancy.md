# Part 78: Multi-Tenancy and Resource Management
## Steps 741-750: Namespaces, Quotas, LimitRanges, Isolation

---

## Step 741: Multi-Tenancy Models

```
Kubernetes Multi-Tenancy Approaches:

1. Namespace-per-team (Soft isolation):
   - Cheapest, fastest
   - Shared control plane, shared nodes
   - RBAC for access control
   - ResourceQuota for resource limits
   - NetworkPolicy for network isolation
   - Risk: noisy neighbor, escape via privileged pods

2. Virtual Clusters (vcluster):
   - Each tenant gets full API server
   - Runs inside host namespace (pods/synced)
   - Better isolation, same node sharing
   - Tools: vcluster, Kamaji

3. Dedicated Node Pools:
   - node selectors/taints per tenant
   - Still shared control plane
   - Physical isolation for compliance

4. Dedicated Clusters:
   - Maximum isolation
   - Highest cost
   - Fleet management: ArgoCD ApplicationSet
   - Use for regulatory/data residency requirements

Hierarchy tools:
  - HNC (Hierarchical Namespace Controller)
  - Capsule: operator for multi-tenancy
  - Loft: enterprise multi-tenancy platform
```

---

## Step 742: Namespace Setup and RBAC

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: team-alpha
  labels:
    team: alpha
    environment: production
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted

---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: team-developer
  namespace: team-alpha
rules:
  - apiGroups: [""]
    resources: [pods, services, configmaps, secrets, serviceaccounts]
    verbs: [get, list, watch, create, update, patch, delete]
  - apiGroups: [apps]
    resources: [deployments, replicasets, statefulsets, daemonsets]
    verbs: [get, list, watch, create, update, patch, delete]
  - apiGroups: [autoscaling]
    resources: [horizontalpodautoscalers]
    verbs: [get, list, watch, create, update, patch, delete]
  - apiGroups: [""]
    resources: [pods/log, pods/exec, pods/portforward]
    verbs: [get, create]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: team-alpha-developers
  namespace: team-alpha
subjects:
  - kind: Group
    name: team-alpha
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: team-developer
  apiGroup: rbac.authorization.k8s.io
```

---

## Step 743: ResourceQuota

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-alpha-quota
  namespace: team-alpha
spec:
  hard:
    requests.cpu: "20"
    limits.cpu: "40"
    requests.memory: 40Gi
    limits.memory: 80Gi
    pods: "50"
    services: "20"
    secrets: "50"
    configmaps: "50"
    persistentvolumeclaims: "20"
    requests.storage: 500Gi
    requests.ephemeral-storage: 100Gi
    services.loadbalancers: "2"
    services.nodeports: "0"

---
# Quota by PriorityClass
apiVersion: v1
kind: ResourceQuota
metadata:
  name: burstable-quota
  namespace: team-alpha
spec:
  hard:
    pods: "10"
  scopeSelector:
    matchExpressions:
      - operator: In
        scopeName: PriorityClass
        values: ["low-priority"]
```

---

## Step 744: LimitRange

```yaml
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
        cpu: 50m
        memory: 64Mi
      maxLimitRequestRatio:
        cpu: "10"
        memory: "4"
    
    - type: Pod
      max:
        cpu: "8"
        memory: 16Gi
    
    - type: PersistentVolumeClaim
      max:
        storage: 50Gi
      min:
        storage: 1Gi
```

---

## Step 745: Network Isolation Between Tenants

```yaml
# Deny all cross-namespace traffic
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-cross-namespace
  namespace: team-alpha
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - podSelector: {}
  egress:
    - to:
        - podSelector: {}
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
      ports:
        - port: 53
          protocol: UDP
    - to:
        - namespaceSelector:
            matchLabels:
              shared-services: "true"

---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-ingress-controller
  namespace: team-alpha
spec:
  podSelector: {}
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: ingress-nginx

---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-prometheus-scrape
  namespace: team-alpha
spec:
  podSelector: {}
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: monitoring
      ports:
        - port: 9090
        - port: 8080
```

---

## Step 746: Capsule - Multi-Tenancy Operator

```yaml
apiVersion: capsule.clastix.io/v1beta2
kind: Tenant
metadata:
  name: team-alpha
spec:
  owners:
    - name: alice
      kind: User
    - name: team-alpha-admins
      kind: Group
  
  namespaceOptions:
    quota: 10
    additionalMetadata:
      labels:
        team: alpha
  
  resourceQuotas:
    scope: Tenant
    items:
      - hard:
          requests.cpu: "20"
          limits.memory: 40Gi
  
  limitRanges:
    items:
      - limits:
          - type: Container
            default:
              cpu: 200m
              memory: 256Mi
            defaultRequest:
              cpu: 100m
              memory: 128Mi
  
  storageClasses:
    allowed:
      - gp3
      - standard
  
  ingressOptions:
    hostnameCollisionScope: Tenant
    allowedClasses:
      allowed:
        - nginx
  
  networkPolicies:
    items:
      - podSelector: {}
        policyTypes: [Ingress, Egress]
        ingress:
          - from:
              - podSelector: {}
  
  nodeSelector:
    team: alpha
  
  priorityClasses:
    allowed:
      - standard
      - low-priority
```

---

## Step 747: Priority Classes

```yaml
# Platform infrastructure
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: platform-critical
value: 1000000
globalDefault: false
description: "Platform infrastructure that must not be evicted"
preemptionPolicy: PreemptLowerPriority

---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: production-high
value: 100000
globalDefault: false
description: "Production workloads - high priority"

---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: default-priority
value: 0
globalDefault: true
description: "Default priority for all workloads"

---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: low-priority
value: -100
globalDefault: false
preemptionPolicy: Never
description: "Batch and dev workloads"

---
# Use priority class in deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: critical-service
spec:
  template:
    spec:
      priorityClassName: production-high
      containers:
        - name: app
          image: myapp:1.0
```

---

## Step 748: Kyverno Policies for Multi-Tenancy

```yaml
# Enforce resource requests/limits
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-resource-requests
spec:
  validationFailureAction: Enforce
  rules:
    - name: check-resource-requests
      match:
        any:
          - resources:
              kinds: [Pod]
              namespaces: ["team-*"]
      validate:
        message: "All containers must have resource requests and limits"
        pattern:
          spec:
            containers:
              - resources:
                  requests:
                    cpu: "?*"
                    memory: "?*"
                  limits:
                    memory: "?*"

---
# Enforce priority class in production
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-priority-class
spec:
  validationFailureAction: Enforce
  rules:
    - name: check-priority-class
      match:
        any:
          - resources:
              kinds: [Pod]
              namespaces: ["production"]
      validate:
        message: "Pods in production must have a priorityClassName"
        pattern:
          spec:
            priorityClassName: "?*"

---
# Disallow host namespaces
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: disallow-host-namespaces
spec:
  validationFailureAction: Enforce
  rules:
    - name: host-namespaces
      match:
        any:
          - resources:
              kinds: [Pod]
      validate:
        message: "Host namespaces are not allowed"
        pattern:
          spec:
            hostPID: "false"
            hostIPC: "false"
            hostNetwork: "false"
```

---

## Step 749: HNC - Hierarchical Namespaces

```yaml
# Create parent namespace
apiVersion: v1
kind: Namespace
metadata:
  name: platform

---
# Create child namespace under platform
apiVersion: hnc.x-k8s.io/v1alpha2
kind: SubnamespaceAnchor
metadata:
  name: team-alpha
  namespace: platform

---
# HierarchyConfiguration: propagate resources down
apiVersion: hnc.x-k8s.io/v1alpha2
kind: HierarchyConfiguration
metadata:
  name: hierarchy
  namespace: platform
spec:
  children:
    - team-alpha
    - team-beta
  propagate:
    - group: rbac.authorization.k8s.io
      resource: roles
    - group: rbac.authorization.k8s.io
      resource: rolebindings

---
# Role in parent propagates to all children
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: platform-reader
  namespace: platform
  annotations:
    propagate.hnc.x-k8s.io/select: ""
rules:
  - apiGroups: [""]
    resources: [pods, services]
    verbs: [get, list, watch]
```

---

## Step 750: Workshop - Multi-Tenancy Alerts

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: multi-tenancy-alerts
  namespace: monitoring
spec:
  groups:
    - name: multi-tenancy
      rules:
        - alert: NamespaceQuotaHigh
          expr: |
            kube_resourcequota{type="used"}
            /
            kube_resourcequota{type="hard"}
            > 0.85
          for: 5m
          annotations:
            summary: "Namespace {{ $labels.namespace }} quota {{ $labels.resource }} > 85%"
          labels:
            severity: warning

        - alert: NamespaceQuotaCritical
          expr: |
            kube_resourcequota{type="used"}
            /
            kube_resourcequota{type="hard"}
            > 0.95
          for: 2m
          annotations:
            summary: "Namespace {{ $labels.namespace }} quota {{ $labels.resource }} > 95%"
          labels:
            severity: critical

        - alert: UnlimitedContainers
          expr: |
            count(kube_pod_container_info) by (namespace)
            -
            count(kube_pod_container_resource_limits{resource="memory"}) by (namespace)
            > 0
          for: 10m
          annotations:
            summary: "Namespace {{ $labels.namespace }} has containers without memory limits"
          labels:
            severity: warning
```

---

## 📊 สรุป Part 78

| Approach | Isolation | Cost | Complexity |
|----------|-----------|------|------------|
| Namespace | Soft | Low | Low |
| vcluster | Medium | Medium | Medium |
| Node pools | Hard (nodes) | Medium-High | Medium |
| Dedicated clusters | Full | High | High |

**เครื่องมือหลัก:**
- `ResourceQuota`: จำกัด total resources ต่อ namespace
- `LimitRange`: default + max ต่อ container
- `NetworkPolicy`: network isolation
- `Capsule`: multi-tenancy operator
- `PriorityClass`: scheduling priority

---

## 🔗 ต่อไป
- [Part 79: Disaster Recovery and Backup](./part-79-disaster-recovery.md)

---
*Part 78 | Steps 741-750 | Multi-Tenancy | Educational Content*
