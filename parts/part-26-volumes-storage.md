# Part 26: Volumes และ Storage
## Steps 251-260: Persistent Storage ใน Kubernetes

---

## 📖 บทนำ

Kubernetes Storage ประกอบด้วย StorageClass, PersistentVolume (PV), PersistentVolumeClaim (PVC) และ CSI drivers

---

## Step 251: StorageClass

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
  annotations:
    storageclass.kubernetes.io/is-default-class: "false"
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
  encrypted: "true"
  kmsKeyId: "arn:aws:kms:ap-southeast-1:123456789:key/xxx"
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBoundingMode: WaitForFirstConsumer
mountOptions:
  - discard

---
# NFS StorageClass (ReadWriteMany)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-client
provisioner: nfs.csi.k8s.io
parameters:
  server: nfs-server.example.com
  share: /shared
reclaimPolicy: Delete
volumeBoundingMode: Immediate
mountOptions:
  - nfsvers=4.1
  - hard
```

---

## Step 252: PersistentVolume

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: manual-pv-1
spec:
  capacity:
    storage: 100Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual
  nfs:
    server: 10.0.0.100
    path: /exports/data

---
# AWS EBS static PV
apiVersion: v1
kind: PersistentVolume
metadata:
  name: aws-ebs-pv
spec:
  capacity:
    storage: 50Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: fast-ssd
  csi:
    driver: ebs.csi.aws.com
    volumeHandle: vol-0123456789abcdef0
    fsType: ext4
```

---

## Step 253: PersistentVolumeClaim

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-data-pvc
  namespace: production
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: fast-ssd
  resources:
    requests:
      storage: 20Gi
  selector:
    matchLabels:
      environment: production

---
apiVersion: v1
kind: Pod
metadata:
  name: app-with-storage
spec:
  containers:
    - name: app
      image: myapp:1.0.0
      volumeMounts:
        - name: data
          mountPath: /data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: app-data-pvc
```

---

## Step 254: Volume Types

```yaml
volumes:
  # emptyDir
  - name: cache
    emptyDir:
      sizeLimit: 1Gi
  
  # emptyDir in memory
  - name: shared-memory
    emptyDir:
      medium: Memory
      sizeLimit: 512Mi
  
  # hostPath
  - name: node-logs
    hostPath:
      path: /var/log
      type: DirectoryOrCreate
  
  # projected volume
  - name: all-in-one
    projected:
      defaultMode: 0644
      sources:
        - configMap:
            name: app-config
        - secret:
            name: app-secrets
            items:
              - key: db-password
                path: db/password
                mode: 0400
        - serviceAccountToken:
            path: token
            expirationSeconds: 3600
            audience: my-service
```

---

## Step 255: CSI Drivers

```yaml
# VolumeSnapshot
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: app-data-snapshot
  namespace: production
spec:
  volumeSnapshotClassName: ebs-vsc
  source:
    persistentVolumeClaimName: app-data-pvc

---
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: ebs-vsc
driver: ebs.csi.aws.com
deletionPolicy: Delete
parameters:
  tagSpecification_1: "key=Environment,value=Production"
```

---

## Step 256: StatefulSet Storage

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: kafka
  namespace: messaging
spec:
  serviceName: kafka-headless
  replicas: 3
  selector:
    matchLabels:
      app: kafka
  template:
    metadata:
      labels:
        app: kafka
    spec:
      containers:
        - name: kafka
          image: confluentinc/cp-kafka:7.5.0
          resources:
            requests:
              cpu: 500m
              memory: 1Gi
            limits:
              cpu: 2000m
              memory: 4Gi
          volumeMounts:
            - name: kafka-data
              mountPath: /var/lib/kafka/data
  volumeClaimTemplates:
    - metadata:
        name: kafka-data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: fast-ssd
        resources:
          requests:
            storage: 100Gi
```

---

## Step 257: ReadWriteMany Storage

```yaml
# NFS PVC (RWX)
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: shared-storage
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: nfs-client
  resources:
    requests:
      storage: 100Gi

---
# AWS EFS StorageClass
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: efs-sc
provisioner: efs.csi.aws.com
parameters:
  provisioningMode: efs-ap
  fileSystemId: fs-xxxx
  directoryPerms: "700"

---
# Deployment ที่ share storage
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 5
  template:
    spec:
      containers:
        - name: app
          image: myapp:1.0.0
          volumeMounts:
            - name: shared-files
              mountPath: /app/uploads
      volumes:
        - name: shared-files
          persistentVolumeClaim:
            claimName: shared-storage
```

---

## Step 258: Storage Quotas

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: storage-quota
  namespace: production
spec:
  hard:
    persistentvolumeclaims: "20"
    requests.storage: "500Gi"
    fast-ssd.storageclass.storage.k8s.io/requests.storage: "200Gi"
    fast-ssd.storageclass.storage.k8s.io/persistentvolumeclaims: "10"

---
apiVersion: v1
kind: LimitRange
metadata:
  name: storage-limits
  namespace: production
spec:
  limits:
    - type: PersistentVolumeClaim
      max:
        storage: 50Gi
      min:
        storage: 1Gi
```

---

## Step 259: Velero Backup

```yaml
apiVersion: velero.io/v1
kind: Backup
metadata:
  name: production-backup
  namespace: velero
spec:
  includedNamespaces:
    - production
    - data
  snapshotVolumes: true
  storageLocation: default
  volumeSnapshotLocations:
    - default
  ttl: 720h

---
apiVersion: velero.io/v1
kind: Schedule
metadata:
  name: daily-backup
  namespace: velero
spec:
  schedule: "0 1 * * *"
  template:
    includedNamespaces:
      - production
    snapshotVolumes: true
    ttl: 168h

---
apiVersion: velero.io/v1
kind: Restore
metadata:
  name: production-restore
  namespace: velero
spec:
  backupName: production-backup
  includedNamespaces:
    - production
  restorePVs: true
  namespaceMapping:
    production: production-restored
```

---

## Step 260: Workshop - MySQL StatefulSet

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secret
  namespace: data
type: Opaque
stringData:
  root-password: "s3cur3P@ssw0rd"

---
apiVersion: v1
kind: ConfigMap
metadata:
  name: mysql-config
  namespace: data
data:
  my.cnf: |
    [mysqld]
    max_connections = 200
    innodb_buffer_pool_size = 1G
    slow_query_log = ON
    long_query_time = 2
    character-set-server = utf8mb4

---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
  namespace: data
spec:
  serviceName: mysql-headless
  replicas: 3
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      securityContext:
        runAsUser: 999
        fsGroup: 999
      containers:
        - name: mysql
          image: mysql:8.0
          env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: root-password
          ports:
            - containerPort: 3306
          resources:
            requests:
              cpu: 500m
              memory: 1Gi
            limits:
              cpu: 2000m
              memory: 4Gi
          livenessProbe:
            exec:
              command: ["mysqladmin", "ping"]
            initialDelaySeconds: 30
          readinessProbe:
            exec:
              command:
                - bash
                - -c
                - "mysql -u root -p${MYSQL_ROOT_PASSWORD} -e 'SELECT 1'"
            initialDelaySeconds: 10
          volumeMounts:
            - name: data
              mountPath: /var/lib/mysql
            - name: config
              mountPath: /etc/mysql/conf.d
              readOnly: true
      volumes:
        - name: config
          configMap:
            name: mysql-config
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: fast-ssd
        resources:
          requests:
            storage: 100Gi
```

---

## 📊 สรุป Part 26

| Component | หน้าที่ |
|-----------|--------|
| StorageClass | Define storage types |
| PV (static) | Pre-provisioned storage |
| PVC | Request storage |
| CSI Driver | Standard storage interface |
| VolumeSnapshot | Snapshots |
| Velero | Backup and restore |

---

## 🔗 ต่อไป
- [Part 27: Namespace และ Resource Quotas](./part-27-namespaces-quotas.md)

---
*Part 26 | Steps 251-260 | ระดับสูง*
