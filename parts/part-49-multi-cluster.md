# Part 49: Multi-cluster Management
## Steps 471-480: Managing Multiple Kubernetes Clusters

---

## 📖 บทนำ

Multi-cluster architecture เป็นสิ่งจำเป็นสำหรับ high availability, geographic distribution, compliance และ isolation ระหว่าง environments

> **หมายเหตุ**: เนื้อหาด้าน security ในหลักสูตรนี้มีไว้เพื่อการศึกษาและการทดสอบระบบที่ได้รับอนุญาตเท่านั้น การนำไปใช้โจมตีระบบที่ไม่ได้รับอนุญาตเป็นสิ่งผิดกฎหมาย

---

## Step 471: Multi-cluster Patterns

```
Multi-cluster Topologies:

1. Hub-and-Spoke (Management + Workload clusters)
   ┌─────────────┐
   │  Hub        │
   │  (ArgoCD,   │──────► Workload Cluster 1 (prod-us)
   │  Policy,    │──────► Workload Cluster 2 (prod-eu)
   │  Monitoring)│──────► Workload Cluster 3 (staging)
   └─────────────┘

2. Federation (Independent + shared config)
   Cluster A ◄──── Shared Config ────► Cluster B
      │                                    │
   Region US                           Region EU

3. Active-Active (HA across clusters)
   Load Balancer / DNS
        │         │
   Cluster A   Cluster B
   (us-east)  (us-west)
```

---

## Step 472: Cluster API (CAPI)

```yaml
# Cluster API - declarative cluster lifecycle management

# AWS Cluster
apiVersion: cluster.x-k8s.io/v1beta1
kind: Cluster
metadata:
  name: production-us
  namespace: clusters
spec:
  clusterNetwork:
    pods:
      cidrBlocks: ["192.168.0.0/16"]
  infrastructureRef:
    apiVersion: infrastructure.cluster.x-k8s.io/v1beta2
    kind: AWSCluster
    name: production-us
  controlPlaneRef:
    apiVersion: controlplane.cluster.x-k8s.io/v1beta2
    kind: KubeadmControlPlane
    name: production-us-cp

---
apiVersion: infrastructure.cluster.x-k8s.io/v1beta2
kind: AWSCluster
metadata:
  name: production-us
  namespace: clusters
spec:
  region: us-east-1
  sshKeyName: my-key
  network:
    vpc:
      availabilityZoneUsageLimit: 3
      availabilityZoneSelection: Ordered

---
# KubeadmControlPlane
apiVersion: controlplane.cluster.x-k8s.io/v1beta2
kind: KubeadmControlPlane
metadata:
  name: production-us-cp
  namespace: clusters
spec:
  replicas: 3
  version: v1.29.0
  machineTemplate:
    infrastructureRef:
      apiVersion: infrastructure.cluster.x-k8s.io/v1beta2
      kind: AWSMachineTemplate
      name: production-us-cp
  kubeadmConfigSpec:
    clusterConfiguration:
      apiServer:
        extraArgs:
          audit-log-path: /var/log/audit.log
          encryption-provider-config: /etc/kubernetes/encryption.yaml
      etcd:
        local:
          extraArgs:
            auto-compaction-retention: "1"

---
# MachineDeployment (worker nodes)
apiVersion: cluster.x-k8s.io/v1beta1
kind: MachineDeployment
metadata:
  name: production-us-workers
  namespace: clusters
spec:
  clusterName: production-us
  replicas: 5
  selector:
    matchLabels:
      cluster.x-k8s.io/cluster-name: production-us
  template:
    spec:
      clusterName: production-us
      version: v1.29.0
      bootstrap:
        configRef:
          apiVersion: bootstrap.cluster.x-k8s.io/v1beta1
          kind: KubeadmConfigTemplate
          name: production-us-workers
      infrastructureRef:
        apiVersion: infrastructure.cluster.x-k8s.io/v1beta2
        kind: AWSMachineTemplate
        name: production-us-workers
```

---

## Step 473: Cluster Fleet Management

```yaml
# Open Cluster Management (OCM) - Hub cluster
# ManagedCluster registration
apiVersion: cluster.open-cluster-management.io/v1
kind: ManagedCluster
metadata:
  name: production-us
  labels:
    cloud: aws
    region: us-east-1
    environment: production
spec:
  hubAcceptsClient: true
  leaseDurationSeconds: 60

---
# ManifestWork - deploy to specific cluster
apiVersion: work.open-cluster-management.io/v1
kind: ManifestWork
metadata:
  name: deploy-app
  namespace: production-us
spec:
  workload:
    manifests:
      - apiVersion: apps/v1
        kind: Deployment
        metadata:
          name: my-app
          namespace: default
        spec:
          replicas: 3
          selector:
            matchLabels:
              app: my-app
          template:
            metadata:
              labels:
                app: my-app
            spec:
              containers:
                - name: app
                  image: myregistry.io/my-app:1.0

---
# Placement - select clusters intelligently
apiVersion: cluster.open-cluster-management.io/v1beta1
kind: Placement
metadata:
  name: production-placement
  namespace: default
spec:
  numberOfClusters: 3
  clusterSets:
    - production
  predicates:
    - requiredClusterSelector:
        labelSelector:
          matchLabels:
            environment: production
  prioritizerPolicy:
    mode: Additive
    configurations:
      - scoreCoordinate:
          builtIn: ResourceAllocatableCPU
        weight: 2
      - scoreCoordinate:
          builtIn: ResourceAllocatableMemory
        weight: 1
```

---

## Step 474: Submariner (Cross-cluster Networking)

```yaml
# Submariner - connect pod networks across clusters
# subctl deploy-broker --kubeconfig hub.kubeconfig
# subctl join broker-info.subm \
#   --kubeconfig cluster1.kubeconfig \
#   --clusterid cluster1

# ServiceExport - export service from cluster1
apiVersion: multicluster.x-k8s.io/v1alpha1
kind: ServiceExport
metadata:
  name: my-app
  namespace: production

---
# ServiceImport
apiVersion: multicluster.x-k8s.io/v1alpha1
kind: ServiceImport
metadata:
  name: my-app
  namespace: production
spec:
  type: ClusterSetIP
  ports:
    - port: 8080
      protocol: TCP

# Cross-cluster DNS
# my-app.production.svc.clusterset.local
```

---

## Step 475: Multi-cluster Observability

```yaml
# Thanos (multi-cluster Prometheus)
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  name: prometheus
  namespace: monitoring
spec:
  replicas: 2
  thanos:
    image: quay.io/thanos/thanos:v0.34.0
    objectStorageConfig:
      secret:
        name: thanos-objstore-config
  externalLabels:
    cluster: production-us
    region: us-east-1

---
apiVersion: v1
kind: Secret
metadata:
  name: thanos-objstore-config
  namespace: monitoring
stringData:
  objstore.yml: |
    type: S3
    config:
      bucket: my-thanos-bucket
      endpoint: s3.us-east-1.amazonaws.com
      region: us-east-1

---
# Thanos Query (on hub cluster)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: thanos-query
  namespace: monitoring
spec:
  replicas: 2
  selector:
    matchLabels:
      app: thanos-query
  template:
    metadata:
      labels:
        app: thanos-query
    spec:
      containers:
        - name: thanos
          image: quay.io/thanos/thanos:v0.34.0
          args:
            - query
            - --http-address=0.0.0.0:9090
            - --store=cluster1-prometheus.monitoring.svc:10901
            - --store=cluster2-prometheus.monitoring.svc:10901
            - --query.replica-label=prometheus_replica
          ports:
            - name: http
              containerPort: 9090
            - name: grpc
              containerPort: 10901
```

---

## Step 476: Velero Multi-cluster Backup

```yaml
# Velero backup across clusters
# velero install \
#   --provider aws \
#   --plugins velero/velero-plugin-for-aws:v1.9.0 \
#   --bucket my-velero-bucket \
#   --backup-location-config region=us-east-1

# BackupStorageLocation
apiVersion: velero.io/v1
kind: BackupStorageLocation
metadata:
  name: production-backup
  namespace: velero
spec:
  provider: aws
  objectStorage:
    bucket: my-velero-bucket
    prefix: production-us
  config:
    region: us-east-1
  accessMode: ReadWrite

---
# Scheduled backup
apiVersion: velero.io/v1
kind: Schedule
metadata:
  name: daily-backup
  namespace: velero
spec:
  schedule: "0 2 * * *"
  template:
    ttl: 720h
    includedNamespaces:
      - production
      - staging
    excludedResources:
      - events
      - nodes
    storageLocation: production-backup
    snapshotVolumes: true
    hooks:
      resources:
        - name: db-backup-hook
          includedNamespaces: [production]
          labelSelector:
            matchLabels:
              app: postgres
          pre:
            - exec:
                container: postgres
                command: ["/bin/bash", "-c", "pg_dump > /backup/pre-backup.sql"]
```

---

## Step 477: KubeVela Multi-cluster

```yaml
# KubeVela - Application delivery across multi-cluster
apiVersion: core.oam.dev/v1beta1
kind: Application
metadata:
  name: my-app
  namespace: production
spec:
  components:
    - name: backend
      type: webservice
      properties:
        image: myregistry.io/backend:1.0
        port: 8080
        cpu: "0.5"
        memory: "512Mi"
  
  policies:
    - name: target-default
      type: topology
      properties:
        clusters: ["production-us", "production-eu"]
        namespace: production
    
    - name: deploy-ha
      type: override
      properties:
        components:
          - name: backend
            properties:
              replicas: 5

  workflow:
    steps:
      - name: deploy-to-staging
        type: deploy
        properties:
          policies: ["target-staging"]
      
      - name: approval
        type: suspend
      
      - name: deploy-to-production
        type: deploy
        properties:
          policies: ["target-default", "deploy-ha"]
```

---

## Step 478: Cross-cluster Service Mesh

```yaml
# Istio multi-cluster Primary-Remote

# Enable endpoint discovery
apiVersion: v1
kind: Secret
metadata:
  name: istio-remote-secret-cluster2
  namespace: istio-system
  labels:
    istio/multiCluster: "true"
type: Opaque
data:
  cluster2: <base64-kubeconfig>

---
# EastWest Gateway
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: cross-network-gateway
  namespace: istio-system
spec:
  selector:
    istio: eastwestgateway
  servers:
    - port:
        number: 15443
        name: tls
        protocol: TLS
      tls:
        mode: AUTO_PASSTHROUGH
      hosts:
        - "*.local"

---
# VirtualService - traffic split
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: my-app
  namespace: production
spec:
  hosts:
    - my-app
  http:
    - route:
        - destination:
            host: my-app.production.svc.cluster.local
            subset: us
          weight: 50
        - destination:
            host: my-app.production.svc.cluster2.local
            subset: us
          weight: 50
```

---

## Step 479: Karpenter Node Autoscaling

```yaml
# Karpenter - Node autoprovisioning
apiVersion: karpenter.sh/v1beta1
kind: NodePool
metadata:
  name: default
spec:
  template:
    metadata:
      labels:
        cluster: production-us
    spec:
      requirements:
        - key: kubernetes.io/arch
          operator: In
          values: ["amd64", "arm64"]
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["spot", "on-demand"]
        - key: node.kubernetes.io/instance-type
          operator: In
          values:
            - m5.large
            - m5.xlarge
            - m6i.large
      nodeClassRef:
        apiVersion: karpenter.k8s.aws/v1beta1
        kind: EC2NodeClass
        name: default
  
  limits:
    cpu: 1000
    memory: 4000Gi
  
  disruption:
    consolidationPolicy: WhenUnderutilized
    consolidateAfter: 30s

---
apiVersion: karpenter.k8s.aws/v1beta1
kind: EC2NodeClass
metadata:
  name: default
spec:
  amiFamily: Bottlerocket
  role: KarpenterNodeRole-production
  subnetSelectorTerms:
    - tags:
        karpenter.sh/discovery: production
  securityGroupSelectorTerms:
    - tags:
        karpenter.sh/discovery: production
  blockDeviceMappings:
    - deviceName: /dev/xvda
      ebs:
        volumeSize: 100Gi
        volumeType: gp3
        encrypted: true
```

---

## Step 480: Workshop - Multi-cluster Setup

```yaml
# Workshop: Complete multi-cluster deployment

# 1. Cluster labels for ArgoCD targeting
apiVersion: v1
kind: Secret
metadata:
  name: prod-us-cluster
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: cluster
    environment: production
    region: us-east-1
stringData:
  name: production-us
  server: https://prod-us.example.com

---
# 2. ApplicationSet - deploy to all production clusters
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: production-apps
  namespace: argocd
spec:
  generators:
    - clusters:
        selector:
          matchLabels:
            environment: production
  template:
    metadata:
      name: "my-app-{{name}}"
    spec:
      project: production
      source:
        repoURL: https://github.com/myorg/helm-charts
        targetRevision: HEAD
        path: charts/my-app
        helm:
          valueFiles:
            - "values-{{metadata.labels.region}}.yaml"
      destination:
        server: "{{server}}"
        namespace: production
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
```

---

## 📊 สรุป Part 49

| Technology | Use Case |
|------------|----------|
| Cluster API | Declarative cluster lifecycle |
| OCM | Fleet management |
| Submariner | Cross-cluster networking |
| Thanos | Multi-cluster metrics |
| Velero | Backup & disaster recovery |
| KubeVela | Multi-cluster app delivery |
| Istio MC | Service mesh federation |
| Karpenter | Smart node autoscaling |

---

## 🔗 ต่อไป
- [Part 50: Disaster Recovery](./part-50-disaster-recovery.md)

---
*Part 49 | Steps 471-480 | ระดับสูง*
