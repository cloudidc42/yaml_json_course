# Part 43: etcd Security
## Steps 411-420: Securing the Cluster State Store

---

## 📖 บทนำ

etcd เป็น distributed key-value store ที่เก็บ state ทั้งหมดของ Kubernetes cluster รวมถึง secrets, configs, certificates การ secure etcd จึงสำคัญมาก

> **หมายเหตุ**: เนื้อหาด้าน security ในหลักสูตรนี้มีไว้เพื่อการศึกษาและการทดสอบระบบที่ได้รับอนุญาตเท่านั้น การนำไปใช้โจมตีระบบที่ไม่ได้รับอนุญาตเป็นสิ่งผิดกฎหมาย

---

## Step 411: etcd Architecture

```
etcd Cluster Architecture:
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │   etcd-1     │  │   etcd-2     │  │   etcd-3     │     │
│  │  (leader)    │  │  (follower)  │  │  (follower)  │     │
│  │              │  │              │  │              │     │
│  │  Port 2379   │  │  Port 2379   │  │  Port 2379   │     │
│  │  (client)    │  │  (client)    │  │  (client)    │     │
│  │  Port 2380   │◄─┤  Port 2380   ├─►│  Port 2380   │     │
│  │  (peer)      │  │  (peer)      │  │  (peer)      │     │
│  └──────────────┘  └──────────────┘  └──────────────┘     │
│         │                                                   │
│         ▼                                                   │
│  kube-apiserver (only client that should access)           │
│                                                             │
│  Raft Consensus:                                           │
│  - Leader election                                          │
│  - Log replication                                          │
│  - Quorum: (N/2)+1 nodes must agree                        │
│  - 3 nodes: tolerate 1 failure                             │
│  - 5 nodes: tolerate 2 failures                            │
└─────────────────────────────────────────────────────────────┘
```

---

## Step 412: etcd TLS Configuration

```yaml
# etcd flags (peer TLS + client TLS)

# Peer TLS (inter-node communication)
# --peer-cert-file=/etc/etcd/pki/peer.crt
# --peer-key-file=/etc/etcd/pki/peer.key
# --peer-ca-file=/etc/etcd/pki/ca.crt
# --peer-client-cert-auth=true

# Client TLS (kube-apiserver connection)
# --cert-file=/etc/etcd/pki/server.crt
# --key-file=/etc/etcd/pki/server.key
# --client-cert-auth=true
# --trusted-ca-file=/etc/etcd/pki/ca.crt

# Listen on internal network only
# --listen-client-urls=https://10.0.0.1:2379
# --advertise-client-urls=https://etcd-1.example.com:2379
# --listen-peer-urls=https://10.0.0.1:2380

---
# kube-apiserver etcd connection flags
# --etcd-servers=https://etcd-1.example.com:2379,https://etcd-2.example.com:2379
# --etcd-cafile=/etc/kubernetes/pki/etcd/ca.crt
# --etcd-certfile=/etc/kubernetes/pki/apiserver-etcd-client.crt
# --etcd-keyfile=/etc/kubernetes/pki/apiserver-etcd-client.key
```

---

## Step 413: Encryption at Rest

```yaml
# /etc/kubernetes/encryption/config.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
    providers:
      # AES-GCM (recommended)
      - aesgcm:
          keys:
            - name: key1
              secret: <base64-encoded-32-byte-key>
      - identity: {}
  
  - resources:
      - configmaps
    providers:
      - aesgcm:
          keys:
            - name: key1
              secret: <base64-encoded-32-byte-key>
      - identity: {}

# Enable: --encryption-provider-config=/etc/kubernetes/encryption/config.yaml
# Generate key: head -c 32 /dev/urandom | base64
# Re-encrypt existing: kubectl get secrets --all-namespaces -o json | kubectl replace -f -
```

---

## Step 414: KMS Encryption Provider

```yaml
# KMS provider (AWS/GCP/Vault)
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
    providers:
      - kms:
          apiVersion: v2
          name: aws-kms
          endpoint: unix:///var/run/kmsplugin/socket.sock
          timeout: 3s
          cachesize: 1000
      - identity: {}

---
# KMS plugin DaemonSet
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: aws-kms-provider
  namespace: kube-system
spec:
  selector:
    matchLabels:
      app: aws-kms-provider
  template:
    metadata:
      labels:
        app: aws-kms-provider
    spec:
      tolerations:
        - effect: NoSchedule
          key: node-role.kubernetes.io/control-plane
      nodeSelector:
        node-role.kubernetes.io/control-plane: ""
      containers:
        - name: aws-kms-provider
          image: amazon/aws-encryption-provider:latest
          args:
            - --key=arn:aws:kms:ap-southeast-1:123456789:key/abc-def
            - --region=ap-southeast-1
            - --listen=unix:///var/run/kmsplugin/socket.sock
          volumeMounts:
            - name: socket
              mountPath: /var/run/kmsplugin
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            runAsNonRoot: true
            runAsUser: 65534
      volumes:
        - name: socket
          hostPath:
            path: /var/run/kmsplugin
            type: DirectoryOrCreate
```

---

## Step 415: etcd Backup

```yaml
# CronJob สำหรับ etcd backup
apiVersion: batch/v1
kind: CronJob
metadata:
  name: etcd-backup
  namespace: kube-system
spec:
  schedule: "0 */6 * * *"
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  jobTemplate:
    spec:
      template:
        spec:
          tolerations:
            - key: node-role.kubernetes.io/control-plane
              effect: NoSchedule
          nodeSelector:
            node-role.kubernetes.io/control-plane: ""
          restartPolicy: OnFailure
          containers:
            - name: etcd-backup
              image: registry.k8s.io/etcd:3.5.9-0
              command:
                - /bin/sh
                - -c
                - |
                  BACKUP_FILE="/backup/etcd-$(date +%Y%m%d-%H%M%S).db"
                  etcdctl snapshot save "$BACKUP_FILE" \
                    --endpoints=https://127.0.0.1:2379 \
                    --cacert=/etc/etcd/pki/ca.crt \
                    --cert=/etc/etcd/pki/server.crt \
                    --key=/etc/etcd/pki/server.key
                  
                  etcdctl snapshot status "$BACKUP_FILE"
                  
                  # Delete old local backups (keep 5)
                  ls -t /backup/*.db | tail -n +6 | xargs rm -f
              env:
                - name: ETCDCTL_API
                  value: "3"
              volumeMounts:
                - name: etcd-pki
                  mountPath: /etc/etcd/pki
                  readOnly: true
                - name: backup
                  mountPath: /backup
          volumes:
            - name: etcd-pki
              hostPath:
                path: /etc/kubernetes/pki/etcd
            - name: backup
              hostPath:
                path: /var/lib/etcd-backup
```

---

## Step 416: etcd Restore

```yaml
# etcd restore procedure

# 1. Stop kube-apiserver
# mv /etc/kubernetes/manifests/kube-apiserver.yaml /tmp/

# 2. Stop etcd
# systemctl stop etcd
# (หรือ mv /etc/kubernetes/manifests/etcd.yaml /tmp/ ถ้า static pod)

# 3. Remove old etcd data
# rm -rf /var/lib/etcd

# 4. Restore from snapshot
# ETCDCTL_API=3 etcdctl snapshot restore /backup/etcd-20240101.db \
#   --name=etcd-1 \
#   --data-dir=/var/lib/etcd \
#   --initial-cluster=etcd-1=https://etcd-1.example.com:2380 \
#   --initial-cluster-token=etcd-cluster-1 \
#   --initial-advertise-peer-urls=https://etcd-1.example.com:2380

# 5. Fix permissions
# chown -R etcd:etcd /var/lib/etcd

# 6. Start etcd
# mv /tmp/etcd.yaml /etc/kubernetes/manifests/

# 7. Restore kube-apiserver
# mv /tmp/kube-apiserver.yaml /etc/kubernetes/manifests/

# 8. Verify cluster
# kubectl get nodes
# kubectl get pods --all-namespaces
```

---

## Step 417: etcd Access Control

```yaml
# etcd NetworkPolicy (ถ้า etcd อยู่ใน Kubernetes)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: etcd-access-policy
  namespace: kube-system
spec:
  podSelector:
    matchLabels:
      component: etcd
  policyTypes:
    - Ingress
  ingress:
    # kube-apiserver access
    - from:
        - podSelector:
            matchLabels:
              component: kube-apiserver
      ports:
        - port: 2379
          protocol: TCP
    # etcd peer communication
    - from:
        - podSelector:
            matchLabels:
              component: etcd
      ports:
        - port: 2380
          protocol: TCP

---
# Prometheus scraping (read-only metrics endpoint)
# Port 2381: metrics (ไม่มี auth, ต้อง secure ด้วย network)
    - from:
        - podSelector:
            matchLabels:
              app: prometheus
      ports:
        - port: 2381
          protocol: TCP
```

---

## Step 418: etcd Monitoring

```yaml
# PrometheusRule สำหรับ etcd alerts
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: etcd-alerts
  namespace: monitoring
spec:
  groups:
    - name: etcd
      rules:
        - alert: EtcdNoLeader
          expr: etcd_server_has_leader == 0
          for: 1m
          annotations:
            summary: "etcd cluster has no leader"
          labels:
            severity: critical
        
        - alert: EtcdHighNumberOfLeaderChanges
          expr: increase(etcd_server_leader_changes_seen_total[15m]) > 3
          annotations:
            summary: "etcd leader changes frequently (>3 in 15m)"
          labels:
            severity: warning
        
        - alert: EtcdDatabaseSizeLimitClose
          expr: >
            etcd_mvcc_db_total_size_in_bytes / etcd_server_quota_backend_bytes * 100 > 80
          annotations:
            summary: "etcd database size > 80% of quota"
          labels:
            severity: warning
        
        - alert: EtcdHighFsyncDurations
          expr: >
            histogram_quantile(0.99, rate(etcd_disk_wal_fsync_duration_seconds_bucket[5m]))
            > 0.5
          annotations:
            summary: "etcd WAL fsync p99 > 500ms"
          labels:
            severity: warning
        
        - alert: EtcdMemberCommunicationSlow
          expr: >
            histogram_quantile(0.99, rate(etcd_network_peer_round_trip_time_seconds_bucket[5m]))
            > 0.15
          annotations:
            summary: "etcd peer communication p99 > 150ms"
          labels:
            severity: warning
```

---

## Step 419: etcd Defragmentation

```yaml
# CronJob สำหรับ defragmentation
apiVersion: batch/v1
kind: CronJob
metadata:
  name: etcd-defrag
  namespace: kube-system
spec:
  schedule: "0 3 * * 0"  # ทุกอาทิตย์ 3am
  jobTemplate:
    spec:
      template:
        spec:
          tolerations:
            - key: node-role.kubernetes.io/control-plane
              effect: NoSchedule
          nodeSelector:
            node-role.kubernetes.io/control-plane: ""
          restartPolicy: OnFailure
          containers:
            - name: etcd-defrag
              image: registry.k8s.io/etcd:3.5.9-0
              command:
                - /bin/sh
                - -c
                - |
                  ENDPOINTS="https://etcd-1:2379,https://etcd-2:2379,https://etcd-3:2379"
                  FLAGS="--cacert=/etc/etcd/pki/ca.crt \
                    --cert=/etc/etcd/pki/server.crt \
                    --key=/etc/etcd/pki/server.key"
                  
                  # Get current revision
                  REVISION=$(etcdctl endpoint status \
                    --endpoints=$ENDPOINTS $FLAGS \
                    --write-out=json | jq 'max_by(.Status.header.revision) | .Status.header.revision')
                  
                  echo "Compacting revision: $REVISION"
                  etcdctl compact $REVISION --endpoints=$ENDPOINTS $FLAGS
                  
                  echo "Defragmenting..."
                  etcdctl defrag --endpoints=$ENDPOINTS $FLAGS
                  
                  echo "Status after defrag:"
                  etcdctl endpoint status --endpoints=$ENDPOINTS $FLAGS --write-out=table
              env:
                - name: ETCDCTL_API
                  value: "3"
              volumeMounts:
                - name: etcd-pki
                  mountPath: /etc/etcd/pki
                  readOnly: true
          volumes:
            - name: etcd-pki
              hostPath:
                path: /etc/kubernetes/pki/etcd
```

---

## Step 420: Workshop - etcd Security Hardening

```yaml
# etcd security hardening checklist

# 1. TLS Verification
# ☑ peer-client-cert-auth=true
# ☑ client-cert-auth=true
# ☑ Rotate certificates before expiry
# ☑ Use separate CA for etcd

# 2. Network Isolation
# ☑ etcd listens on internal network only (not 0.0.0.0)
# ☑ Firewall: port 2379 only from kube-apiserver
# ☑ Firewall: port 2380 only from etcd peers
# ☑ etcd nodes on dedicated control plane nodes

# 3. Encryption
# ☑ Encryption at rest (AES-GCM or KMS)
# ☑ etcd data directory on encrypted volume
# ☑ TLS for all communications

# 4. Access Control
# ☑ No direct etcd access from worker nodes
# ☑ etcd accessible only via kube-apiserver
# ☑ Audit etcdctl usage

# 5. Backup
# ☑ Automated backup every 6 hours
# ☑ Off-site backup (S3/GCS)
# ☑ Tested restore procedure
# ☑ Backup encryption

# 6. Monitoring
# ☑ Leader changes alert
# ☑ High disk usage alert (>80%)
# ☑ Slow WAL fsync alert
# ☑ Member connectivity alert

# 7. Separate etcd for events (performance)
# --etcd-servers-overrides=/events#https://etcd-events:2379

---
# etcd health check commands
# etcdctl endpoint health --cluster \
#   --endpoints=https://etcd-1:2379,https://etcd-2:2379,https://etcd-3:2379
#
# etcdctl endpoint status --cluster --write-out=table
#
# Output:
# | ENDPOINT              | ID               | VERSION | DB SIZE | IS LEADER |
# |-----------------------|------------------|---------|---------|----------|
# | https://etcd-1:2379   | 8e9e05c52164694d | 3.5.9   | 25 MB   | true      |
# | https://etcd-2:2379   | 91bc3c398fb3c146 | 3.5.9   | 25 MB   | false     |
# | https://etcd-3:2379   | fd422379fda50e48 | 3.5.9   | 25 MB   | false     |
```

---

## 📊 สรุป Part 43

| ด้าน | Best Practice |
|------|-------------- |
| Transport Security | TLS peer + client auth |
| Encryption at Rest | AES-GCM หรือ KMS |
| Access Control | Firewall + เฉพาะ kube-apiserver |
| Backup | ทุก 6 ชั่วโมง + off-site |
| Monitoring | Leader, disk, latency alerts |
| Defrag | รายสัปดาห์ |
| Certificate Rotation | ก่อน expire |

---

## 🔗 ต่อไป
- [Part 44: Container Security Fundamentals](./part-44-container-security.md)

---
*Part 43 | Steps 411-420 | ระดับสูง*
