# Part 93: Multi-Cluster and Federation
## Steps 891-900: Cluster API, Fleet Management, Cross-Cluster Networking

---

## Step 891: Multi-Cluster Overview

```
Multi-Cluster Kubernetes Patterns:

Why Multiple Clusters?
  - Fault isolation: blast radius containment
  - Regulatory: data sovereignty (EU data in EU)
  - Scaling: single cluster limit ~5000 nodes
  - Team autonomy: separate dev/staging/prod
  - Multi-cloud: avoid vendor lock-in

Architecture Patterns:
  Hub-and-Spoke:
    - One management cluster (hub)
    - Many workload clusters (spokes)
    - Tools: Fleet, ArgoCD + ApplicationSet
  
  Federated:
    - Multiple equal clusters
    - Shared configuration
    - Tools: KubeFed, Admiralty
  
  Active-Active:
    - Traffic split across clusters
    - Global load balancing (AWS Global Accelerator)
    - Requires data replication
  
  Active-Passive:
    - Primary cluster handles all traffic
    - Secondary cluster is warm standby
    - Failover triggered manually or automatically

Tools:
  Cluster API: declarative cluster lifecycle
  Fleet (Rancher): Helm + GitOps across clusters
  Liqo: transparent cross-cluster pod scheduling
  Submariner: cross-cluster networking
  Cilium Cluster Mesh: eBPF-based multi-cluster
```

---

## Step 892: Cluster API

```yaml
apiVersion: cluster.x-k8s.io/v1beta1
kind: Cluster
metadata:
  name: production-us-east
  namespace: clusters
spec:
  clusterNetwork:
    pods:
      cidrBlocks: [10.0.0.0/16]
    services:
      cidrBlocks: [172.20.0.0/16]
  infrastructureRef:
    apiVersion: infrastructure.cluster.x-k8s.io/v1beta2
    kind: AWSCluster
    name: production-us-east
  controlPlaneRef:
    apiVersion: controlplane.cluster.x-k8s.io/v1beta2
    kind: KubeadmControlPlane
    name: production-us-east-control-plane

---
apiVersion: infrastructure.cluster.x-k8s.io/v1beta2
kind: AWSCluster
metadata:
  name: production-us-east
  namespace: clusters
spec:
  region: us-east-1
  sshKeyName: my-keypair
  network:
    vpc:
      cidrBlock: 10.0.0.0/16
      availabilityZoneUsageLimit: 3

---
apiVersion: cluster.x-k8s.io/v1beta1
kind: MachineDeployment
metadata:
  name: production-workers
  namespace: clusters
spec:
  clusterName: production-us-east
  replicas: 5
  selector:
    matchLabels:
      cluster.x-k8s.io/cluster-name: production-us-east
  template:
    metadata:
      labels:
        cluster.x-k8s.io/cluster-name: production-us-east
    spec:
      version: v1.29.0
      clusterName: production-us-east
      bootstrap:
        configRef:
          apiVersion: bootstrap.cluster.x-k8s.io/v1beta1
          kind: KubeadmConfigTemplate
          name: worker-template
      infrastructureRef:
        apiVersion: infrastructure.cluster.x-k8s.io/v1beta2
        kind: AWSMachineTemplate
        name: worker-template
```

---

## Step 893: ArgoCD Multi-Cluster

```yaml
# ArgoCD: add remote cluster secret
apiVersion: v1
kind: Secret
metadata:
  name: staging-cluster-secret
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: cluster
type: Opaque
stringData:
  name: staging-cluster
  server: https://staging-api.mycompany.com
  config: |
    {
      "bearerToken": "<SERVICE_ACCOUNT_TOKEN>",
      "tlsClientConfig": {
        "insecure": false,
        "caData": "<BASE64_CA>"
      }
    }

---
# ApplicationSet: deploy to all production clusters
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: myapp-all-clusters
  namespace: argocd
spec:
  generators:
    - clusters:
        selector:
          matchLabels:
            environment: production
  template:
    metadata:
      name: "myapp-{{name}}"
    spec:
      project: default
      source:
        repoURL: https://github.com/myorg/gitops-repo
        targetRevision: main
        path: kubernetes/overlays/production
      destination:
        server: "{{server}}"
        namespace: myapp
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
```

---

## Step 894: Fleet (Rancher Fleet)

```yaml
apiVersion: fleet.cattle.io/v1alpha1
kind: GitRepo
metadata:
  name: myapp
  namespace: fleet-default
spec:
  repo: https://github.com/myorg/gitops-repo
  branch: main
  paths:
    - kubernetes/
  targets:
    - name: production
      clusterSelector:
        matchLabels:
          environment: production
      helm:
        values:
          replicas: 5
          resources:
            cpu: 500m
            memory: 512Mi
    
    - name: staging
      clusterSelector:
        matchLabels:
          environment: staging
      helm:
        values:
          replicas: 2
    
    - name: dev
      clusterSelector:
        matchLabels:
          environment: dev
      helm:
        values:
          replicas: 1
```

---

## Step 895: Cross-Cluster Service Discovery

```yaml
# ServiceExport: make service available cross-cluster
apiVersion: multicluster.x-k8s.io/v1alpha1
kind: ServiceExport
metadata:
  name: user-service
  namespace: production

---
# Cilium Cluster Mesh: global service
apiVersion: v1
kind: Service
metadata:
  name: user-service
  namespace: production
  annotations:
    service.cilium.io/global: "true"
    service.cilium.io/shared: "true"
spec:
  selector:
    app: user-service
  ports:
    - port: 8080
```

---

## Step 896: Centralized Policy Management

```yaml
# ArgoCD: sync Kyverno policies to all clusters
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: kyverno-policies
  namespace: argocd
spec:
  generators:
    - clusters: {}
  template:
    metadata:
      name: "kyverno-policies-{{name}}"
    spec:
      project: platform
      source:
        repoURL: https://github.com/myorg/k8s-policies
        targetRevision: main
        path: kyverno/
      destination:
        server: "{{server}}"
        namespace: kyverno
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
```

---

## Step 897: Multi-Cluster Observability

```yaml
# Thanos Query: federate across clusters
apiVersion: apps/v1
kind: Deployment
metadata:
  name: thanos-query
  namespace: monitoring
spec:
  template:
    spec:
      containers:
        - name: thanos
          image: quay.io/thanos/thanos:v0.35.0
          args:
            - query
            - --http-address=0.0.0.0:9090
            - --query.replica-label=replica
            - --endpoint=us-east-1-prometheus.monitoring.svc:10901
            - --endpoint=eu-west-1-prometheus.monitoring.svc:10901
            - --endpoint=ap-southeast-1-prometheus.monitoring.svc:10901
```

---

## Step 898: Cluster Upgrades at Scale

```yaml
# ClusterClass: reusable cluster template
apiVersion: cluster.x-k8s.io/v1beta1
kind: ClusterClass
metadata:
  name: production-class
  namespace: clusters
spec:
  controlPlane:
    ref:
      apiVersion: controlplane.cluster.x-k8s.io/v1beta2
      kind: KubeadmControlPlaneTemplate
      name: production-control-plane-template
      namespace: clusters
    machineInfrastructure:
      ref:
        apiVersion: infrastructure.cluster.x-k8s.io/v1beta2
        kind: AWSMachineTemplate
        name: production-control-plane-machine
        namespace: clusters
  workers:
    machineDeployments:
      - class: default-worker
        template:
          bootstrap:
            ref:
              apiVersion: bootstrap.cluster.x-k8s.io/v1beta1
              kind: KubeadmConfigTemplate
              name: production-worker-template
              namespace: clusters
          infrastructure:
            ref:
              apiVersion: infrastructure.cluster.x-k8s.io/v1beta2
              kind: AWSMachineTemplate
              name: production-worker-machine
              namespace: clusters
  variables:
    - name: region
      required: true
      schema:
        openAPIV3Schema:
          type: string
    - name: nodeCount
      schema:
        openAPIV3Schema:
          type: integer
          default: 3
```

---

## Step 899: Disaster Recovery

```yaml
# Velero: daily full cluster backup
apiVersion: velero.io/v1
kind: Schedule
metadata:
  name: daily-cluster-backup
  namespace: velero
spec:
  schedule: "0 2 * * *"
  template:
    ttl: 720h
    storageLocation: aws-s3
    volumeSnapshotLocations:
      - aws-us-east-1
    includedNamespaces: ["*"]
    excludedNamespaces: [kube-system, velero, monitoring]
    includeClusterResources: true

---
# PodDisruptionBudget
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: myapp-pdb
  namespace: production
spec:
  maxUnavailable: 1
  selector:
    matchLabels:
      app: myapp
```

---

## Step 900: Workshop - Multi-Cluster Health

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: multi-cluster-alerts
  namespace: monitoring
spec:
  groups:
    - name: multi-cluster
      rules:
        - alert: ClusterAPIClusterNotReady
          expr: |
            capi_cluster_status_condition{type="Ready",status="False"} == 1
          for: 5m
          annotations:
            summary: "Cluster {{ $labels.name }} is not Ready"
          labels:
            severity: critical

        - alert: ClusterVersionSkew
          expr: |
            count(count by (version) (kubernetes_build_info)) > 2
          for: 1h
          annotations:
            summary: "More than 2 Kubernetes versions running across clusters"
          labels:
            severity: warning

        - alert: VeleroBackupFailed
          expr: |
            velero_backup_failure_total > 0
          for: 0m
          annotations:
            summary: "Velero backup failed in cluster {{ $labels.cluster }}"
          labels:
            severity: critical
```

---

## 📊 สรุป Part 93

| Tool | Use Case | Approach |
|------|----------|---------|
| Cluster API | Cluster lifecycle | Declarative K8s CRDs |
| ArgoCD ApplicationSet | GitOps multi-cluster | Hub-and-spoke |
| Fleet | Simple multi-cluster | GitRepo + BundleDeployment |
| Submariner | Cross-cluster networking | Gateway + ServiceExport |
| Cilium Cluster Mesh | eBPF cross-cluster | L3/L4 mesh |
| Thanos | Multi-cluster metrics | Remote write federation |
| Velero | Backup + DR | S3-based snapshots |

---

## 🔗 ต่อไป
- [Part 94: Platform Engineering and Developer Portals](./part-94-platform-engineering.md)

---
*Part 93 | Steps 891-900 | Multi-Cluster and Federation | Educational Content*
