# Part 17: Kubernetes Multi-cluster Management
## Steps 161-170: จัดการหลาย Cluster

---

## 📖 บทนำ

Multi-cluster Kubernetes คือการบริหารจัดการหลาย Kubernetes clusters พร้อมกัน เพื่อ high availability, geographic distribution, และ workload isolation

### Multi-cluster Patterns
```
Single Region Multi-cluster:
  ┌─────────────────────────────────┐
  │         Management Cluster       │
  │  (Fleet Manager / ArgoCD Hub)   │
  └──────────┬────────────────────┘
             │
    ┌────────┴────────┐
    ▼                 ▼
┌───────┐         ┌───────┐
│Prod   │         │Stage  │
│Cluster│         │Cluster│
└───────┘         └───────┘

Multi-Region Multi-cluster:
  US-East    EU-West    AP-Southeast
  ┌───────┐  ┌───────┐  ┌───────┐
  │Cluster│  │Cluster│  │Cluster│
  └───────┘  └───────┘  └───────┘
       ↑           ↑          ↑
       └───────────╌──────────┘
              Global LB
```

---

## Step 161: kubeconfig สำหรับหลาย Clusters

```yaml
# ~/.kube/config - รวม kubeconfig หลาย clusters
apiVersion: v1
kind: Config
preferences: {}

clusters:
  - name: prod-us-east
    cluster:
      server: https://prod-us-east.example.com:6443
      certificate-authority-data: BASE64_CA_DATA
  - name: staging
    cluster:
      server: https://staging.example.com:6443
      certificate-authority-data: BASE64_CA_DATA
  - name: dev
    cluster:
      server: https://dev.example.com:6443
      insecure-skip-tls-verify: true  # dev only!

users:
  - name: admin-prod
    user:
      client-certificate-data: BASE64_CERT
      client-key-data: BASE64_KEY
  - name: admin-staging
    user:
      token: eyJhbGciOiJSUzI1...
  - name: dev-user
    user:
      exec:
        apiVersion: client.authentication.k8s.io/v1beta1
        command: aws
        args:
          - eks
          - get-token
          - --cluster-name
          - my-dev-cluster

contexts:
  - name: prod-us-east
    context:
      cluster: prod-us-east
      user: admin-prod
      namespace: production
  - name: staging
    context:
      cluster: staging
      user: admin-staging
      namespace: staging
  - name: dev
    context:
      cluster: dev
      user: dev-user
      namespace: default

current-context: dev
```

```bash
# จัดการ contexts
kubectl config get-contexts
kubectl config use-context prod-us-east
kubectl config current-context

# รัน command บน specific cluster
kubectl --context=staging get pods

# Merge kubeconfig files
KUBECONFIG=~/.kube/config:~/.kube/new-cluster.yaml \
  kubectl config view --flatten > /tmp/merged.yaml
mv /tmp/merged.yaml ~/.kube/config
```

---

## Step 162: kubectx และ kubens

```bash
# kubectx - switch contexts
kubectx                    # list contexts
kubectx prod-us-east       # switch to prod
kubectx -                  # switch to previous
kubectx prod=prod-us-east  # rename context

# kubens - switch namespaces
kubens                   # list namespaces
kubens production        # switch namespace
kubens -                 # switch to previous
```

---

## Step 163: Cluster Federation และ Kubefed

```yaml
# FederatedDeployment - deploy ไปหลาย clusters พร้อมกัน
apiVersion: types.kubefed.io/v1beta1
kind: FederatedDeployment
metadata:
  name: my-app
  namespace: production
spec:
  template:
    metadata:
      labels:
        app: my-app
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
              image: myapp:1.0.0
  placement:
    clusters:
      - name: us-east
      - name: eu-west
      - name: ap-southeast
  overrides:
    - clusterName: us-east
      clusterOverrides:
        - path: "/spec/replicas"
          value: 5  # us-east รับ traffic มากกว่า
    - clusterName: eu-west
      clusterOverrides:
        - path: "/spec/replicas"
          value: 3
    - clusterName: ap-southeast
      clusterOverrides:
        - path: "/spec/replicas"
          value: 2
```

---

## Step 164: Cluster API (CAPI)

```yaml
# Cluster object
apiVersion: cluster.x-k8s.io/v1beta1
kind: Cluster
metadata:
  name: my-cluster
  namespace: default
spec:
  clusterNetwork:
    pods:
      cidrBlocks: ["192.168.0.0/16"]
    services:
      cidrBlocks: ["10.96.0.0/12"]
  infrastructureRef:
    apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
    kind: AWSCluster
    name: my-cluster
  controlPlaneRef:
    apiVersion: controlplane.cluster.x-k8s.io/v1beta1
    kind: KubeadmControlPlane
    name: my-cluster-control-plane

---
# AWSCluster - infrastructure ที่ใช้ AWS
apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
kind: AWSCluster
metadata:
  name: my-cluster
  namespace: default
spec:
  region: us-east-1
  sshKeyName: my-ssh-key
  network:
    vpc:
      cidrBlock: "10.0.0.0/16"
    subnets:
      - availabilityZone: us-east-1a
        cidrBlock: "10.0.0.0/24"
        isPublic: false
      - availabilityZone: us-east-1b
        cidrBlock: "10.0.1.0/24"
        isPublic: false
      - availabilityZone: us-east-1c
        cidrBlock: "10.0.2.0/24"
        isPublic: false

---
# KubeadmControlPlane
apiVersion: controlplane.cluster.x-k8s.io/v1beta1
kind: KubeadmControlPlane
metadata:
  name: my-cluster-control-plane
spec:
  replicas: 3
  version: v1.28.0
  machineTemplate:
    infrastructureRef:
      apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
      kind: AWSMachineTemplate
      name: my-cluster-control-plane
  kubeadmConfigSpec:
    initConfiguration:
      nodeRegistration:
        name: '{{ ds.meta_data.local_hostname }}'
    joinConfiguration:
      nodeRegistration:
        name: '{{ ds.meta_data.local_hostname }}'

---
# MachineDeployment - worker nodes
apiVersion: cluster.x-k8s.io/v1beta1
kind: MachineDeployment
metadata:
  name: my-cluster-workers
spec:
  clusterName: my-cluster
  replicas: 5
  selector:
    matchLabels:
      cluster.x-k8s.io/cluster-name: my-cluster
  template:
    spec:
      clusterName: my-cluster
      version: v1.28.0
      bootstrap:
        configRef:
          apiVersion: bootstrap.cluster.x-k8s.io/v1beta1
          kind: KubeadmConfigTemplate
          name: my-cluster-workers
      infrastructureRef:
        apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
        kind: AWSMachineTemplate
        name: my-cluster-workers
```

---

## Step 165: Rancher - Multi-cluster Management UI

```yaml
# Fleet - GitOps สำหรับ Rancher multi-cluster
apiVersion: fleet.cattle.io/v1alpha1
kind: GitRepo
metadata:
  name: my-app
  namespace: fleet-default
spec:
  repo: https://github.com/myorg/k8s-manifests
  branch: main
  paths:
    - apps/my-app
  targets:
    - name: production
      clusterSelector:
        matchLabels:
          env: production
    - name: staging
      clusterSelector:
        matchLabels:
          env: staging
```

---

## Step 166: Cross-cluster Service Discovery

```yaml
# ServiceExport - export service ออกไปให้ clusters อื่นเห็น
apiVersion: multicluster.x-k8s.io/v1alpha1
kind: ServiceExport
metadata:
  name: my-database
  namespace: data

---
# ServiceImport - import service จาก cluster อื่น
apiVersion: multicluster.x-k8s.io/v1alpha1
kind: ServiceImport
metadata:
  name: my-database
  namespace: data
spec:
  type: ClusterSetIP
  ports:
    - port: 5432
      protocol: TCP
# ใช้งาน: my-database.data.svc.clusterset.local
```

---

## Step 167: Global Load Balancing

```yaml
# ExternalDNS สำหรับ multi-cluster
apiVersion: v1
kind: Service
metadata:
  name: my-app
  namespace: production
  annotations:
    external-dns.alpha.kubernetes.io/hostname: myapp.example.com
    external-dns.alpha.kubernetes.io/aws-routing-policy: latency
    external-dns.alpha.kubernetes.io/aws-region: us-east-1
spec:
  type: LoadBalancer
  selector:
    app: my-app
  ports:
    - port: 80
      targetPort: 8080

---
# Global Service Mesh ด้วย Istio ServiceEntry
apiVersion: networking.istio.io/v1beta1
kind: ServiceEntry
metadata:
  name: remote-service
  namespace: production
spec:
  hosts:
    - remote-api.example.com
  location: MESH_EXTERNAL
  ports:
    - number: 443
      name: https
      protocol: HTTPS
  resolution: DNS
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 30s
      baseEjectionTime: 30s
```

---

## Step 168: Cluster Disaster Recovery

```yaml
# Velero - Schedule regular backups
apiVersion: velero.io/v1
kind: Schedule
metadata:
  name: daily-backup
  namespace: velero
spec:
  schedule: "0 2 * * *"  # ทุกวัน 02:00
  template:
    includedNamespaces:
      - production
      - data
    excludedResources:
      - events
      - events.events.k8s.io
    storageLocation: default
    volumeSnapshotLocations:
      - default
    ttl: 720h  # 30 days

---
# Backup on-demand
apiVersion: velero.io/v1
kind: Backup
metadata:
  name: pre-upgrade-backup
  namespace: velero
spec:
  includedNamespaces:
    - production
  labelSelector:
    matchLabels:
      backup: "true"
  storageLocation: default
  volumeSnapshotLocations:
    - default

---
# Restore from backup
apiVersion: velero.io/v1
kind: Restore
metadata:
  name: restore-production
  namespace: velero
spec:
  backupName: daily-backup-20240101020000
  includedNamespaces:
    - production
  restorePVs: true
  namespaceMapping:
    production: production-restored
```

---

## Step 169: Multi-cluster Observability

```yaml
# Thanos - Prometheus multi-cluster aggregation
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  name: prometheus
  namespace: monitoring
spec:
  replicas: 2
  thanos:
    image: quay.io/thanos/thanos:v0.34.0
    version: v0.34.0
    objectStorageConfig:
      name: thanos-objstore-secret
      key: objstore.yml

---
# Thanos Query - รวม metrics จากทุก clusters
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
        - name: thanos-query
          image: quay.io/thanos/thanos:v0.34.0
          args:
            - query
            - --http-address=0.0.0.0:10902
            - --store=thanos-store-us-east:10901
            - --store=thanos-store-eu-west:10901
            - --store=thanos-store-ap-southeast:10901
            - --query.replica-label=replica
          ports:
            - containerPort: 10902
              name: http

---
# Grafana datasource เชื่อมต่อ Thanos
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-datasources
  namespace: monitoring
data:
  datasources.yaml: |
    apiVersion: 1
    datasources:
      - name: Thanos
        type: prometheus
        url: http://thanos-query:10902
        isDefault: true
        jsonData:
          httpMethod: GET
          timeInterval: "30s"
```

---

## Step 170: Workshop - Multi-cluster GitOps

```yaml
# ArgoCD ApplicationSet - deploy ไปหลาย clusters
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: my-app
  namespace: argocd
spec:
  generators:
    - list:
        elements:
          - cluster: prod-us-east
            url: https://prod-us-east.example.com
            env: production
            replicas: "5"
          - cluster: prod-eu-west
            url: https://prod-eu-west.example.com
            env: production
            replicas: "3"
          - cluster: staging
            url: https://staging.example.com
            env: staging
            replicas: "2"
  template:
    metadata:
      name: "my-app-{{cluster}}"
    spec:
      project: default
      source:
        repoURL: https://github.com/myorg/k8s-manifests
        targetRevision: HEAD
        path: "apps/my-app/overlays/{{env}}"
        kustomize:
          images:
            - "myapp=myapp:1.2.3"
          patches:
            - patch: |-
                - op: replace
                  path: /spec/replicas
                  value: {{replicas}}
              target:
                kind: Deployment
                name: my-app
      destination:
        server: "{{url}}"
        namespace: production
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true

---
# ClusterGenerator - auto-discover clusters
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: monitoring-stack
  namespace: argocd
spec:
  generators:
    - clusters:
        selector:
          matchLabels:
            monitoring: "enabled"
  template:
    metadata:
      name: "monitoring-{{name}}"
    spec:
      project: default
      source:
        repoURL: https://github.com/myorg/k8s-manifests
        path: monitoring
        targetRevision: HEAD
      destination:
        server: "{{server}}"
        namespace: monitoring
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
```

---

## 📊 สรุป Part 17

| หัวข้อ | เครื่องมือ |
|--------|----------|
| kubeconfig | contexts, kubectx, kubens |
| Federation | Kubefed, FederatedDeployment |
| Cluster Lifecycle | Cluster API (CAPI) |
| Management UI | Rancher, Fleet |
| Cross-cluster Network | Submariner, Istio multi-cluster |
| Global LB | ExternalDNS, GSLB |
| Disaster Recovery | Velero backup/restore |
| Observability | Thanos, Grafana |
| GitOps | ArgoCD ApplicationSet |

---

## 🔗 ต่อไป
- [Part 18: GitOps Advanced](./part-18-gitops-advanced.md)

---
*Part 17 | Steps 161-170 | ระดับสูง*
