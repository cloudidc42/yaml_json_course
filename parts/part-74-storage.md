# Part 74: Kubernetes Storage and StatefulSets
## Steps 701-710: PV, PVC, StorageClass, StatefulSets

---

## Step 701: Storage Concepts Overview

```
Kubernetes Storage Architecture:

PersistentVolume (PV):
  - Cluster-wide storage resource
  - Provisioned by admin or dynamically
  - Has capacity, access modes, reclaim policy
  - Lifecycle independent of pods

PersistentVolumeClaim (PVC):
  - Request for storage by user/pod
  - Binds to a matching PV
  - Specifies size, access mode, storageClass

StorageClass:
  - Defines storage "profiles" (fast-ssd, standard)
  - Enables dynamic provisioning
  - Different provisioners: AWS EBS, GCP PD, NFS, Ceph

Access Modes:
  ReadWriteOnce (RWO): single node read-write
  ReadOnlyMany (ROX): many nodes read-only
  ReadWriteMany (RWX): many nodes read-write
  ReadWriteOncePod (RWOP): single pod read-write

Reclaim Policies:
  Retain: PV stays after PVC deletion
  Delete: PV deleted with PVC (cloud volumes deleted)
```

---

## Step 702: StorageClass

```yaml
# AWS EBS gp3 (encrypted)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
  encrypted: "true"
  kmsKeyId: arn:aws:kms:us-east-1:123456789012:key/mrk-xxx
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Delete
allowVolumeExpansion: true

---
# GCP Regional Persistent Disk
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ssd-regional
provisioner: pd.csi.storage.gke.io
parameters:
  type: pd-ssd
  replication-type: regional-pd
  zones: us-central1-a,us-central1-b
volumeBind ingMode: WaitForFirstConsumer
reclaimPolicy: Retain

---
# NFS (ReadWriteMany)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-client
provisioner: nfs.csi.k8s.io
parameters:
  server: nfs.example.com
  share: /exports/k8s
  mountPermissions: "0755"
reclaimPolicy: Delete
volumeBind ingMode: Immediate
mountOptions:
  - nfsvers=4.1
  - hard
  - timeo=600
```

---

## Step 703: PersistentVolume and PVC

```yaml
# Static PV
apiVersion: v1
kind: PersistentVolume
metadata:
  name: postgres-data-pv
spec:
  capacity:
    storage: 100Gi
  accessModes:
    - ReadWriteOnce
  reclaimPolicy: Retain
  storageClassName: fast-ssd
  csi:
    driver: ebs.csi.aws.com
    volumeHandle: vol-0abc123def456789
    fsType: ext4

---
# PVC: request storage
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-data
  namespace: databases
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Gi
  storageClassName: fast-ssd

---
# Use PVC in Pod
apiVersion: v1
kind: Pod
metadata:
  name: postgres
  namespace: databases
spec:
  containers:
    - name: postgres
      image: postgres:15
      env:
        - name: PGDATA
          value: /var/lib/postgresql/data/pgdata
      volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: postgres-data

# Expand PVC:
# kubectl patch pvc postgres-data -p '{"spec":{"resources":{"requests":{"storage":"200Gi"}}}}'
```

---

## Step 704: StatefulSet Fundamentals

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: databases
spec:
  serviceName: postgres-headless
  replicas: 3
  selector:
    matchLabels:
      app: postgres
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      partition: 0
  podManagementPolicy: OrderedReady
  
  template:
    metadata:
      labels:
        app: postgres
    spec:
      securityContext:
        fsGroup: 999
        runAsUser: 999
        runAsNonRoot: true
      
      containers:
        - name: postgres
          image: postgres:15
          ports:
            - containerPort: 5432
              name: postgres
          env:
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: password
            - name: PGDATA
              value: /var/lib/postgresql/data/pgdata
          resources:
            limits:
              cpu: 2000m
              memory: 4Gi
            requests:
              cpu: 500m
              memory: 1Gi
          livenessProbe:
            exec:
              command: ["pg_isready", "-U", "postgres"]
            initialDelaySeconds: 30
            periodSeconds: 10
          readinessProbe:
            exec:
              command: ["pg_isready", "-U", "postgres"]
            initialDelaySeconds: 5
            periodSeconds: 5
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
  
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: fast-ssd
        resources:
          requests:
            storage: 100Gi

---
# Headless Service (required for StatefulSet DNS)
apiVersion: v1
kind: Service
metadata:
  name: postgres-headless
  namespace: databases
spec:
  clusterIP: None
  selector:
    app: postgres
  ports:
    - port: 5432
      name: postgres

# Pod DNS:
# postgres-0.postgres-headless.databases.svc.cluster.local
# postgres-1.postgres-headless.databases.svc.cluster.local
# postgres-2.postgres-headless.databases.svc.cluster.local
```

---

## Step 705: StatefulSet - PostgreSQL HA

```yaml
# Primary service (Patroni sets role=primary label)
apiVersion: v1
kind: Service
metadata:
  name: postgres-primary
  namespace: databases
spec:
  selector:
    app: postgres
    role: primary
  ports:
    - port: 5432
      targetPort: 5432

---
# Read replica service
apiVersion: v1
kind: Service
metadata:
  name: postgres-replica
  namespace: databases
spec:
  selector:
    app: postgres
    role: replica
  ports:
    - port: 5432
      targetPort: 5432

---
# Patroni config
apiVersion: v1
kind: ConfigMap
metadata:
  name: postgres-config
  namespace: databases
data:
  patroni.yaml: |
    scope: postgres-cluster
    name: ${POD_NAME}
    
    bootstrap:
      dcs:
        ttl: 30
        loop_wait: 10
        retry_timeout: 10
        maximum_lag_on_failover: 1048576
        postgresql:
          use_pg_rewind: true
          parameters:
            wal_level: replica
            hot_standby: "on"
            max_wal_senders: 5
    
    postgresql:
      listen: 0.0.0.0:5432
      connect_address: ${POD_IP}:5432
      data_dir: /var/lib/postgresql/data/pgdata
```

---

## Step 706: Volume Snapshots

```yaml
# VolumeSnapshotClass
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: ebs-snapshot-class
  annotations:
    snapshot.storage.kubernetes.io/is-default-class: "true"
driver: ebs.csi.aws.com
deletionPolicy: Delete

---
# Take snapshot
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: postgres-snapshot-20240115
  namespace: databases
spec:
  volumeSnapshotClassName: ebs-snapshot-class
  source:
    persistentVolumeClaimName: data-postgres-0

---
# Restore from snapshot
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-restored
  namespace: databases
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Gi
  storageClassName: fast-ssd
  dataSource:
    name: postgres-snapshot-20240115
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io

---
# CronJob: automated daily snapshots
apiVersion: batch/v1
kind: CronJob
metadata:
  name: postgres-daily-snapshot
  namespace: databases
spec:
  schedule: "0 2 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: snapshot-sa
          containers:
            - name: snapshot
              image: bitnami/kubectl:latest
              command:
                - /bin/sh
                - -c
                - |
                  DATE=$(date +%Y%m%d)
                  kubectl apply -f - <<EOF
                  apiVersion: snapshot.storage.k8s.io/v1
                  kind: VolumeSnapshot
                  metadata:
                    name: postgres-snapshot-${DATE}
                    namespace: databases
                  spec:
                    volumeSnapshotClassName: ebs-snapshot-class
                    source:
                      persistentVolumeClaimName: data-postgres-0
                  EOF
          restartPolicy: OnFailure
```

---

## Step 707: CSI Drivers

```yaml
# AWS EBS CSI
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-gp3
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  encrypted: "true"
volumeBind ingMode: WaitForFirstConsumer
allowVolumeExpansion: true

---
# Azure Disk CSI
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: azure-premium-ssd
provisioner: disk.csi.azure.com
parameters:
  skuName: Premium_LRS
  kind: Managed
volumeBind ingMode: WaitForFirstConsumer
reclaimPolicy: Delete
allowVolumeExpansion: true

---
# Rook-Ceph (on-prem)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: rook-ceph-block
provisioner: rook-ceph.rbd.csi.ceph.com
parameters:
  clusterID: rook-ceph
  pool: replicapool
  imageFormat: "2"
  imageFeatures: layering
  csi.storage.k8s.io/provisioner-secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/provisioner-secret-namespace: rook-ceph
reclaimPolicy: Delete
allowVolumeExpansion: true
mountOptions:
  - discard
```

---

## Step 708: Storage Security

```yaml
# Kyverno: require encrypted StorageClass
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-encrypted-storage
spec:
  validationFailureAction: Enforce
  rules:
    - name: check-encryption
      match:
        any:
          - resources:
              kinds: [PersistentVolumeClaim]
              namespaces: [production, databases]
      validate:
        message: "PVCs in production must use encrypted StorageClass"
        pattern:
          spec:
            storageClassName: "?(fast-ssd|ebs-gp3|azure-premium-ssd)"

---
# Kyverno: block hostPath volumes
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: block-host-path
spec:
  validationFailureAction: Enforce
  rules:
    - name: no-hostpath
      match:
        any:
          - resources:
              kinds: [Pod]
      validate:
        message: "hostPath volumes are not allowed"
        deny:
          conditions:
            any:
              - key: "{{ request.object.spec.volumes[].hostPath | length(@) }}"
                operator: GreaterThan
                value: 0
```

---

## Step 709: Storage Monitoring

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
        - alert: PersistentVolumeFillingUp
          expr: |
            kubelet_volume_stats_available_bytes
            /
            kubelet_volume_stats_capacity_bytes
            < 0.15
          for: 1m
          annotations:
            summary: "PVC {{ $labels.persistentvolumeclaim }} is filling up"
          labels:
            severity: warning

        - alert: PersistentVolumeCritical
          expr: |
            kubelet_volume_stats_available_bytes
            /
            kubelet_volume_stats_capacity_bytes
            < 0.05
          for: 1m
          annotations:
            summary: "PVC {{ $labels.persistentvolumeclaim }} critically full"
          labels:
            severity: critical

        - alert: PersistentVolumeClaimPending
          expr: kube_persistentvolumeclaim_status_phase{phase="Pending"} == 1
          for: 5m
          annotations:
            summary: "PVC {{ $labels.persistentvolumeclaim }} stuck in Pending"
          labels:
            severity: warning

        - alert: StatefulSetReplicasNotReady
          expr: |
            kube_statefulset_status_replicas_ready
            <
            kube_statefulset_status_replicas
          for: 5m
          annotations:
            summary: "StatefulSet {{ $labels.statefulset }} replicas not ready"
          labels:
            severity: warning
```

---

## Step 710: Workshop - Storage Best Practices

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: storage-best-practices
  namespace: databases
data:
  practices.yaml: |
    storageclass:
      - Use WaitForFirstConsumer (avoid cross-AZ issues)
      - Enable allowVolumeExpansion
      - Use encrypted StorageClass in production
      - Set Retain reclaimPolicy for databases
    
    statefulsets:
      - Always use headless service (clusterIP: None)
      - Use volumeClaimTemplates (not emptyDir)
      - Configure liveness/readiness probes
      - Set podDisruptionBudget for HA
    
    backups:
      - VolumeSnapshots for crash-consistent backups
      - Automate with CronJob
      - Test restore regularly
      - Keep N snapshots, delete old ones
    
    security:
      - Block hostPath volumes (Kyverno)
      - Require encrypted StorageClass
      - RBAC: no delete on production PVCs
      - Monitor PVC usage (alert at 85%)
    
    performance:
      - gp3 over gp2 (cheaper, configurable IOPS)
      - noatime mount option for read-heavy
      - Match StorageClass to workload IOPS needs

---
# PodDisruptionBudget for StatefulSet HA
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: postgres-pdb
  namespace: databases
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: postgres
```

---

## 📊 สรุป Part 74

| Concept | Description | Key Setting |
|---------|-------------|-------------|
| PV | Cluster storage resource | capacity, accessModes, reclaimPolicy |
| PVC | Storage request | storageClassName, resources |
| StorageClass | Storage profile | provisioner, volumeBindingMode |
| StatefulSet | Ordered pods with stable IDs | serviceName, volumeClaimTemplates |
| Snapshot | Point-in-time backup | VolumeSnapshotClass, source PVC |

---

## 🔗 ต่อไป
- [Part 75: Kubernetes Networking Deep Dive](./part-75-networking.md)

---
*Part 74 | Steps 701-710 | Storage and StatefulSets | Educational Content*
