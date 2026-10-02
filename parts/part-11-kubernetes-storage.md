# Part 11: Kubernetes Storage
## Steps 101-110: Persistent Storage ใน Kubernetes

---

## Step 101: Storage Concepts

```
Storage Hierarchy:
  StorageClass        → ประเภทของ storage (SSD, HDD, NFS)
       ↓
  PersistentVolume    → actual storage resource
       ↓
  PersistentVolumeClaim → request สำหรับ storage
       ↓
  Pod Volume Mount    → ใช้งานใน container

Access Modes:
  ReadWriteOnce (RWO)    → อ่าน/เขียนได้จาก 1 node
  ReadOnlyMany (ROX)     → อ่านได้จากหลาย nodes
  ReadWriteMany (RWX)    → อ่าน/เขียนได้จากหลาย nodes
  ReadWriteOncePod (RWOP)→ อ่าน/เขียนได้จาก 1 pod (K8s 1.22+)

Reclaim Policies:
  Retain  → ข้อมูลยังอยู่หลัง PVC ถูกลบ
  Delete  → ลบ storage พร้อมกับ PVC
```

---

## Step 102: StorageClass

```yaml
# StorageClass สำหรับ AWS EBS
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
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer

---
# StorageClass สำหรับ NFS
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-shared
provisioner: nfs.csi.k8s.io
parameters:
  server: nfs.example.com
  share: /shared
  mountPermissions: "0777"
reclaimPolicy: Retain
allowVolumeExpansion: true
mountOptions: [hard, nfsvers=4.1]
```

---

## Step 103: PersistentVolume

```yaml
# Static PV
apiVersion: v1
kind: PersistentVolume
metadata:
  name: postgres-pv-1
  labels:
    type: local
    app: postgres
spec:
  storageClassName: fast-ssd
  capacity:
    storage: 100Gi
  accessModes:
    - ReadWriteOnce
  reclaimPolicy: Retain
  csi:
    driver: ebs.csi.aws.com
    volumeHandle: vol-0123456789abcdef0
    fsType: ext4

---
# Local PV
apiVersion: v1
kind: PersistentVolume
metadata:
  name: local-pv-node1
spec:
  storageClassName: local-ssd
  capacity:
    storage: 500Gi
  accessModes:
    - ReadWriteOnce
  reclaimPolicy: Retain
  local:
    path: /mnt/disks/ssd0
  nodeAffinity:
    required:
      nodeSelectorTerms:
        - matchExpressions:
            - key: kubernetes.io/hostname
              operator: In
              values: ["node-1"]
```

---

## Step 104: PersistentVolumeClaim

```yaml
# PVC
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-data
  namespace: data
spec:
  storageClassName: fast-ssd
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 50Gi
  selector:
    matchLabels:
      app: postgres

---
# ใช้ PVC ใน Pod
apiVersion: v1
kind: Pod
metadata:
  name: postgres
spec:
  containers:
    - name: postgres
      image: postgres:15
      volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: postgres-data

# Volume Expansion
# kubectl patch pvc postgres-data -p '{"spec":{"resources":{"requests":{"storage":"100Gi"}}}}'
```

---

## Step 105: Volume Types

```yaml
spec:
  volumes:
    # 1. emptyDir - Temporary shared storage
    - name: cache
      emptyDir:
        medium: Memory
        sizeLimit: 500Mi
    
    # 2. configMap
    - name: app-config
      configMap:
        name: app-settings
        defaultMode: 0644
    
    # 3. secret
    - name: tls-certs
      secret:
        secretName: app-tls
        defaultMode: 0400
    
    # 4. projected - รวมหลาย sources
    - name: combined
      projected:
        sources:
          - configMap: {name: app-config}
          - secret: {name: app-secrets}
          - serviceAccountToken:
              path: token
              expirationSeconds: 3600
    
    # 5. downwardAPI
    - name: pod-info
      downwardAPI:
        items:
          - path: name
            fieldRef:
              fieldPath: metadata.name
          - path: labels
            fieldRef:
              fieldPath: metadata.labels
    
    # 6. hostPath (ระวัง security!)
    - name: host-logs
      hostPath:
        path: /var/log/myapp
        type: DirectoryOrCreate
    
    # 7. nfs
    - name: shared-data
      nfs:
        server: nfs.example.com
        path: /shared
    
    # 8. CSI
    - name: csi-volume
      csi:
        driver: secrets-store.csi.k8s.io
        readOnly: true
        volumeAttributes:
          secretProviderClass: "aws-secrets"
```

---

## Step 106: Secrets Store CSI Driver

```yaml
# SecretProviderClass (AWS)
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: aws-secrets-provider
  namespace: production
spec:
  provider: aws
  parameters:
    objects: |
      - objectName: "myapp/production/db"
        objectType: "secretsmanager"
        jmesPath:
          - path: "password"
            objectAlias: "db-password"
  secretObjects:
    - secretName: myapp-db-credentials
      type: Opaque
      data:
        - objectName: db-password
          key: password

---
# ใช้ใน Pod
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  volumes:
    - name: secrets-store
      csi:
        driver: secrets-store.csi.k8s.io
        readOnly: true
        volumeAttributes:
          secretProviderClass: aws-secrets-provider
  containers:
    - name: app
      image: myapp:latest
      volumeMounts:
        - name: secrets-store
          mountPath: /mnt/secrets
          readOnly: true
      env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: myapp-db-credentials
              key: password
```

---

## Step 107: Backup และ Restore

```bash
# Velero Backup
velero install \
    --provider aws \
    --plugins velero/velero-plugin-for-aws:v1.7.0 \
    --bucket my-backup-bucket \
    --backup-location-config region=ap-southeast-1

velero backup create daily-backup \
    --include-namespaces production,data \
    --exclude-resources pods,events

velero schedule create daily \
    --schedule="0 2 * * *" \
    --include-namespaces production \
    --ttl 720h

velero restore create --from-backup daily-backup

# etcd backup
ETCDCTL_API=3 etcdctl snapshot save \
    /backup/etcd-$(date +%Y%m%d-%H%M).db \
    --endpoints=https://127.0.0.1:2379 \
    --cacert=/etc/kubernetes/pki/etcd/ca.crt \
    --cert=/etc/kubernetes/pki/etcd/server.crt \
    --key=/etc/kubernetes/pki/etcd/server.key
```

---

## Step 108: Storage Monitoring

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
        - alert: PVCAlmostFull
          expr: |
            kubelet_volume_stats_used_bytes / kubelet_volume_stats_capacity_bytes > 0.85
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "PVC {{ $labels.persistentvolumeclaim }} is 85% full"
        
        - alert: PVCFull
          expr: |
            kubelet_volume_stats_used_bytes / kubelet_volume_stats_capacity_bytes > 0.95
          for: 1m
          labels:
            severity: critical
          annotations:
            summary: "PVC {{ $labels.persistentvolumeclaim }} is CRITICAL"
        
        - alert: PVCNotBound
          expr: kube_persistentvolumeclaim_status_phase{phase!="Bound"} == 1
          for: 5m
          labels:
            severity: warning
```

---

## Step 109: Cloud Storage Classes

```yaml
# GKE
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gke-ssd
provisioner: pd.csi.storage.gke.io
parameters:
  type: pd-ssd
  replication-type: regional-pd
volumeBindingMode: WaitForFirstConsumer

---
# AKS
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: azure-premium
provisioner: disk.csi.azure.com
parameters:
  storageaccounttype: Premium_LRS
  kind: Managed

---
# EKS
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: eks-gp3
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  encrypted: "true"
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```

---

## Step 110: Redis StatefulSet

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis
  namespace: cache
spec:
  serviceName: redis
  replicas: 3
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      initContainers:
        - name: config-init
          image: redis:7-alpine
          command: ['sh', '-c', 'cp /config/redis.conf /data/redis.conf']
          volumeMounts:
            - name: config
              mountPath: /config
            - name: data
              mountPath: /data
      containers:
        - name: redis
          image: redis:7-alpine
          command: ['redis-server', '/data/redis.conf']
          ports:
            - containerPort: 6379
              name: redis
          resources:
            limits: {cpu: 500m, memory: 1Gi}
          volumeMounts:
            - name: data
              mountPath: /data
          livenessProbe:
            exec:
              command: ['redis-cli', 'ping']
            initialDelaySeconds: 30
            periodSeconds: 10
      volumes:
        - name: config
          configMap: {name: redis-config}
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: [ReadWriteOnce]
        storageClassName: fast-ssd
        resources:
          requests:
            storage: 20Gi

---
apiVersion: v1
kind: ConfigMap
metadata:
  name: redis-config
  namespace: cache
data:
  redis.conf: |
    maxmemory 900mb
    maxmemory-policy allkeys-lru
    save 900 1
    appendonly yes
    appendfsync everysec
```

---

## 📊 สรุป Part 11

| ประเภท Volume | Use Case | Persistence |
|--------------|----------|-------------|
| emptyDir | Temporary cache | ไม่ (ลบเมื่อ pod ตาย) |
| hostPath | Node-local access | Node-local |
| PVC | Database, persistent data | ใช่ |
| ConfigMap | Config files | ไม่ persist |
| Secret | Sensitive config | ไม่ persist |
| NFS | Shared storage | ใช่ (external) |

---
*Part 11 | Steps 101-110 | ระดับกลาง*
