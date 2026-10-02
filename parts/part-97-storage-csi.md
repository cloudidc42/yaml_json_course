# Part 97: Storage, CSI Drivers, and Rook-Ceph
## Steps 931-940: Persistent Volumes, StorageClass, CSI, Rook-Ceph, Velero

---

## Step 931: Kubernetes Storage Overview

```
Kubernetes Storage Architecture:

Layers:
  1. StorageClass: defines provisioner + parameters
  2. PersistentVolumeClaim (PVC): developer request
  3. PersistentVolume (PV): actual storage resource
  4. CSI Driver: vendor plugin (AWS EBS, GCP PD, Ceph)

Access Modes:
  ReadWriteOnce (RWO): single node read-write (block)
  ReadOnlyMany (ROX): multiple nodes read-only
  ReadWriteMany (RWX): multiple nodes read-write (NFS/CephFS)
  ReadWriteOncePod (RWOP): single pod (K8s 1.22+)

Volume Modes:
  Filesystem: mounted as directory (default)
  Block: raw block device (database performance)

Reclaim Policies:
  Delete: PV deleted when PVC deleted (default for dynamic)
  Retain: PV kept, manual cleanup required

CSI (Container Storage Interface):
  - Standard interface between K8s and storage vendors
  - Dynamic provisioning, snapshots, resize
  - Drivers: AWS EBS, GCP PD, Azure Disk, Ceph, Portworx
  
Volume Snapshots:
  VolumeSnapshotClass: defines snapshot driver
  VolumeSnapshot: request a snapshot
  VolumeSnapshotContent: actual snapshot resource

Popular Storage Solutions:
  Cloud: AWS EBS gp3, GCP Persistent Disk, Azure Managed Disk
  Distributed: Rook-Ceph, Longhorn, OpenEBS, Portworx
  Network: NFS, SMB (Azure Files), GlusterFS
  Local: local-path (K3s), hostPath (dev only)
```

---

## Step 932: StorageClass

```yaml
# AWS EBS gp3 StorageClass
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
  encrypted: "true"
  kmsKeyId: arn:aws:kms:us-east-1:123456789:key/abc-123
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
reclaimPolicy: Delete

---
# GCP Persistent Disk StorageClass
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gcp-pd-ssd
provisioner: pd.csi.storage.gke.io
parameters:
  type: pd-ssd
  replication-type: regional-pd
  zones: us-central1-a,us-central1-b
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true

---
# Azure Managed Disk StorageClass
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: azure-ssd-premium
provisioner: disk.csi.azure.com
parameters:
  skuName: Premium_LRS
  kind: Managed
  cachingmode: ReadOnly
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```

---

## Step 933: PVC and StatefulSet

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-data
  namespace: production
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: gp3
  resources:
    requests:
      storage: 100Gi
  volumeMode: Filesystem

---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: production
spec:
  replicas: 3
  serviceName: postgres-headless
  selector:
    matchLabels:
      app: postgres
  template:
    spec:
      containers:
        - name: postgres
          image: postgres:16
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: [ReadWriteOnce]
        storageClassName: gp3
        resources:
          requests:
            storage: 100Gi

---
# Static PV: NFS pre-provisioned storage
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv-001
spec:
  capacity:
    storage: 500Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  storageClassName: nfs
  nfs:
    server: nfs.mycompany.com
    path: /exports/shared-data
  mountOptions:
    - hard
    - nfsvers=4.1
```

---

## Step 934: Volume Snapshots

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: aws-ebs-snapshots
  annotations:
    snapshot.storage.kubernetes.io/is-default-class: "true"
driver: ebs.csi.aws.com
deletionPolicy: Delete
parameters:
  tagSpecification_1: "key=backup,value=true"

---
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: postgres-snapshot-20240101
  namespace: production
spec:
  volumeSnapshotClassName: aws-ebs-snapshots
  source:
    persistentVolumeClaimName: postgres-data

---
# Restore PVC from snapshot
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-data-restored
  namespace: production
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: gp3
  resources:
    requests:
      storage: 100Gi
  dataSource:
    name: postgres-snapshot-20240101
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io

---
# CronJob: automatic daily snapshots
apiVersion: batch/v1
kind: CronJob
metadata:
  name: postgres-snapshot
  namespace: production
spec:
  schedule: "0 3 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: snapshot-creator
          containers:
            - name: snapshotter
              image: bitnami/kubectl:1.29
              command:
                - /bin/sh
                - -c
                - |
                  DATE=$(date +%Y%m%d)
                  kubectl apply -f - <<EOF
                  apiVersion: snapshot.storage.k8s.io/v1
                  kind: VolumeSnapshot
                  metadata:
                    name: postgres-snapshot-$DATE
                    namespace: production
                  spec:
                    volumeSnapshotClassName: aws-ebs-snapshots
                    source:
                      persistentVolumeClaimName: postgres-data
                  EOF
          restartPolicy: OnFailure
```

---

## Step 935: Rook-Ceph

```yaml
apiVersion: ceph.rook.io/v1
kind: CephCluster
metadata:
  name: rook-ceph
  namespace: rook-ceph
spec:
  cephVersion:
    image: quay.io/ceph/ceph:v18.2.0
  dataDirHostPath: /var/lib/rook
  mon:
    count: 3
    allowMultiplePerNode: false
  mgr:
    count: 2
  dashboard:
    enabled: true
    ssl: true
  storage:
    useAllNodes: false
    useAllDevices: false
    nodes:
      - name: storage-node-01
        devices:
          - name: nvme0n1
          - name: nvme1n1
      - name: storage-node-02
        devices:
          - name: nvme0n1
  resources:
    osd:
      requests:
        cpu: 1000m
        memory: 4Gi
      limits:
        cpu: 4000m
        memory: 8Gi

---
apiVersion: ceph.rook.io/v1
kind: CephBlockPool
metadata:
  name: replicapool
  namespace: rook-ceph
spec:
  replicated:
    size: 3
    requireSafeReplicaSize: true
  parameters:
    compression_mode: aggressive

---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: rook-ceph-block
provisioner: rook-ceph.rbd.csi.ceph.com
parameters:
  clusterID: rook-ceph
  pool: replicapool
  imageFormat: "2"
  imageFeatures: layering,fast-diff,object-map,deep-flatten,exclusive-lock
  csi.storage.k8s.io/provisioner-secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/provisioner-secret-namespace: rook-ceph
reclaimPolicy: Delete
allowVolumeExpansion: true
```

---

## Step 936: CephFS for ReadWriteMany

```yaml
apiVersion: ceph.rook.io/v1
kind: CephFilesystem
metadata:
  name: myfs
  namespace: rook-ceph
spec:
  metadataPool:
    replicated:
      size: 3
  dataPools:
    - name: data0
      replicated:
        size: 3
  metadataServer:
    activeCount: 1
    activeStandby: true
    resources:
      requests:
        cpu: 1000m
        memory: 4Gi

---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: rook-cephfs
provisioner: rook-ceph.cephfs.csi.ceph.com
parameters:
  clusterID: rook-ceph
  fsName: myfs
  pool: myfs-data0
  csi.storage.k8s.io/provisioner-secret-name: rook-csi-cephfs-provisioner
  csi.storage.k8s.io/provisioner-secret-namespace: rook-ceph
reclaimPolicy: Delete
allowVolumeExpansion: true

---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: shared-uploads
  namespace: production
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: rook-cephfs
  resources:
    requests:
      storage: 500Gi
```

---

## Step 937: Longhorn

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: longhorn
provisioner: driver.longhorn.io
parameters:
  numberOfReplicas: "3"
  staleReplicaTimeout: "2880"
  diskSelector: ssd
  nodeSelector: storage
  dataLocality: best-effort
allowVolumeExpansion: true
reclaimPolicy: Delete

---
apiVersion: longhorn.io/v1beta2
kind: RecurringJob
metadata:
  name: backup-to-s3
  namespace: longhorn-system
spec:
  cron: "0 4 * * *"
  task: backup
  groups: []
  retain: 7
  concurrency: 2
  labels:
    environment: production
```

---

## Step 938: Volume Resize and Migration

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: pvc-ops-guide
  namespace: production
data:
  resize.sh: |
    # Resize PVC (requires allowVolumeExpansion: true)
    kubectl patch pvc postgres-data -n production \
      --type=merge \
      -p '{"spec":{"resources":{"requests":{"storage":"200Gi"}}}}'
    
    # Watch resize status
    kubectl get pvc postgres-data -n production -w
  
  migrate.sh: |
    # Migrate PVC to different StorageClass
    # 1. Scale down workload
    kubectl scale deployment myapp -n production --replicas=0
    
    # 2. Snapshot source PVC
    kubectl apply -f snapshot.yaml
    
    # 3. Wait for snapshot ready
    kubectl wait volumesnapshot/migration-snapshot \
      -n production \
      --for=jsonpath='{.status.readyToUse}'=true
    
    # 4. Create new PVC from snapshot with new StorageClass
    # (change storageClassName in restore spec)
    
    # 5. Scale workload back up
    kubectl scale deployment myapp -n production --replicas=3
```

---

## Step 939: Storage Monitoring

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: storage-alerts
  namespace: monitoring
spec:
  groups:
    - name: storage
      rules:
        - alert: PVCUsageHigh
          expr: |
            kubelet_volume_stats_used_bytes / kubelet_volume_stats_capacity_bytes > 0.85
          for: 5m
          annotations:
            summary: "PVC {{ $labels.persistentvolumeclaim }} in {{ $labels.namespace }} is {{ $value | humanizePercentage }} full"
          labels:
            severity: warning

        - alert: PVCUsageCritical
          expr: |
            kubelet_volume_stats_used_bytes / kubelet_volume_stats_capacity_bytes > 0.95
          for: 2m
          annotations:
            summary: "PVC {{ $labels.persistentvolumeclaim }} is critically full"
          labels:
            severity: critical

        - alert: CephHealthError
          expr: ceph_health_status == 2
          for: 1m
          annotations:
            summary: "Ceph cluster health is HEALTH_ERR"
          labels:
            severity: critical

        - alert: LonghornVolumeRobustnessDegraded
          expr: |
            longhorn_volume_robustness == 2
          for: 5m
          annotations:
            summary: "Longhorn volume {{ $labels.volume }} is degraded"
          labels:
            severity: warning
```

---

## Step 940: Workshop - Storage Summary

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: storage-decision-guide
  namespace: kube-system
data:
  guide.md: |
    # Storage Selection Guide
    
    | Workload | Access Mode | Recommended | Notes |
    |----------|-------------|-------------|-------|
    | PostgreSQL | RWO | gp3/Ceph RBD | Block, high IOPS |
    | Redis | RWO | gp3 | Fast, single-node |
    | Shared uploads | RWX | CephFS/NFS | Multi-pod access |
    | ML model serving | ROX | CephFS | Read-only replicated |
    | Dev environment | RWO | local-path | No HA needed |
    
    Performance Tiers:
    
    High Performance:
      - AWS: io2 Block Express (64,000 IOPS)
      - GCP: Hyperdisk Extreme
      - Ceph: NVMe OSDs with BlueStore
    
    Standard:
      - AWS: gp3 (3,000 IOPS baseline)
      - GCP: pd-ssd
      - Azure: Premium_LRS
    
    Archival:
      - AWS: sc1 (cold HDD)
      - S3 via CSI
      - Ceph Object Gateway (S3-compatible)
```

---

## 📊 สรุป Part 97

| Solution | Type | Use Case | RWX |
|----------|------|----------|-----|
| AWS EBS gp3 | Block | General purpose | No |
| Rook-Ceph RBD | Block | High performance | No |
| CephFS | Filesystem | Shared files | Yes |
| Longhorn | Block | Edge/on-prem | No |
| NFS | Filesystem | Legacy shared | Yes |
| MinIO | Object | S3-compatible | N/A |

---

## 🔗 ต่อไป
- [Part 98: Microservices Patterns with Istio](./part-98-istio-service-mesh.md)

---
*Part 97 | Steps 931-940 | Storage, CSI Drivers, and Rook-Ceph | Educational Content*
