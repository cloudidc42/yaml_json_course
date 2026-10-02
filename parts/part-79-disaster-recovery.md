# Part 79: Disaster Recovery and Backup
## Steps 751-760: Velero, etcd Backup, RTO/RPO

---

## Step 751: DR Concepts for Kubernetes

```
Disaster Recovery Terminology:

RTO (Recovery Time Objective):
  - Maximum acceptable downtime
  - How fast must we restore?
  - Example: 4 hours RTO = restore within 4h of disaster

RPO (Recovery Point Objective):
  - Maximum acceptable data loss
  - How much data can we lose?
  - Example: 1 hour RPO = backups every hour (max 1h loss)

DR Tiers (for Kubernetes):
  Tier 1 (RPO~0, RTO<15min): Active-Active multi-region
  Tier 2 (RPO<1h, RTO<1h):  Active-Passive + Velero
  Tier 3 (RPO<24h, RTO<4h): Daily backups + manual restore

What to back up:
  1. etcd: cluster state (all K8s objects)
  2. Persistent Volumes: application data
  3. Secrets: credentials (use ESO or Vault)
  4. Custom Resources: CRDs and their instances
  5. Helm releases: chart + values

What GitOps handles automatically:
  - All Kubernetes manifests (in Git)
  - Re-deploy from Git = instant cluster rebuild
  - Only need to back up stateful data
```

---

## Step 752: etcd Backup

```bash
# Save etcd snapshot
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-snapshot-$(date +%Y%m%d-%H%M%S).db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Verify snapshot
ETCDCTL_API=3 etcdctl snapshot status /backup/etcd-snapshot.db --write-out=table

# Restore from snapshot
ETCDCTL_API=3 etcdctl snapshot restore /backup/etcd-snapshot.db \
  --name=master \
  --initial-cluster=master=https://127.0.0.1:2380 \
  --initial-cluster-token=etcd-cluster-1 \
  --initial-advertise-peer-urls=https://127.0.0.1:2380 \
  --data-dir=/var/lib/etcd-restore
```

```yaml
# CronJob: automated etcd backup to S3
apiVersion: batch/v1
kind: CronJob
metadata:
  name: etcd-backup
  namespace: kube-system
spec:
  schedule: "0 */6 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          hostNetwork: true
          nodeName: control-plane-1
          tolerations:
            - key: node-role.kubernetes.io/control-plane
              effect: NoSchedule
          containers:
            - name: etcd-backup
              image: bitnami/etcd:latest
              command:
                - /bin/sh
                - -c
                - |
                  SNAPSHOT=/tmp/etcd-$(date +%Y%m%d-%H%M%S).db
                  ETCDCTL_API=3 etcdctl snapshot save $SNAPSHOT \
                    --endpoints=https://127.0.0.1:2379 \
                    --cacert=/etc/kubernetes/pki/etcd/ca.crt \
                    --cert=/etc/kubernetes/pki/etcd/server.crt \
                    --key=/etc/kubernetes/pki/etcd/server.key
                  aws s3 cp $SNAPSHOT s3://my-etcd-backups/
              volumeMounts:
                - name: etcd-certs
                  mountPath: /etc/kubernetes/pki/etcd
                  readOnly: true
          volumes:
            - name: etcd-certs
              hostPath:
                path: /etc/kubernetes/pki/etcd
          restartPolicy: OnFailure
```

---

## Step 753: Velero Installation

```yaml
# BackupStorageLocation
apiVersion: velero.io/v1
kind: BackupStorageLocation
metadata:
  name: default
  namespace: velero
spec:
  provider: aws
  objectStorage:
    bucket: my-velero-backup-bucket
    prefix: velero
  config:
    region: us-east-1
    kmsKeyId: arn:aws:kms:us-east-1:123456789012:key/mrk-xxx

---
# VolumeSnapshotLocation
apiVersion: velero.io/v1
kind: VolumeSnapshotLocation
metadata:
  name: default
  namespace: velero
spec:
  provider: aws
  config:
    region: us-east-1
```

---

## Step 754: Velero Backups

```yaml
# On-demand backup
apiVersion: velero.io/v1
kind: Backup
metadata:
  name: full-cluster-backup
  namespace: velero
spec:
  includedNamespaces:
    - "*"
  excludedNamespaces:
    - kube-system
    - kube-public
    - velero
  excludedResources:
    - events
    - events.events.k8s.io
  storageLocation: default
  volumeSnapshotLocations:
    - default
  snapshotVolumes: true
  ttl: 720h

---
# Scheduled daily backup
apiVersion: velero.io/v1
kind: Schedule
metadata:
  name: daily-backup
  namespace: velero
spec:
  schedule: "0 2 * * *"
  template:
    includedNamespaces:
      - production
      - staging
      - databases
    storageLocation: default
    snapshotVolumes: true
    ttl: 720h
    hooks:
      resources:
        - name: postgres-freeze
          includedNamespaces: [databases]
          labelSelector:
            matchLabels:
              app: postgres
          pre:
            - exec:
                container: postgres
                command: ["/bin/bash", "-c", "psql -c 'SELECT pg_start_backup(\"velero\");'"]
          post:
            - exec:
                container: postgres
                command: ["/bin/bash", "-c", "psql -c 'SELECT pg_stop_backup();'"]
```

---

## Step 755: Velero Restore

```yaml
# Restore entire backup
apiVersion: velero.io/v1
kind: Restore
metadata:
  name: full-restore
  namespace: velero
spec:
  backupName: full-cluster-backup
  includedNamespaces:
    - "*"
  excludedNamespaces:
    - kube-system
  restorePVs: true
  namespaceMapping:
    production: production-restored

---
# Restore specific resources
apiVersion: velero.io/v1
kind: Restore
metadata:
  name: restore-deployments
  namespace: velero
spec:
  backupName: production-backup
  includedNamespaces:
    - production
  includedResources:
    - deployments
    - services
    - configmaps
  labelSelector:
    matchLabels:
      app: myapp
  restorePVs: false
```

---

## Step 756: Multi-Region HA Architecture

```yaml
# ArgoCD ApplicationSet: deploy to both regions
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: myapp-multi-region
  namespace: argocd
spec:
  generators:
    - list:
        elements:
          - cluster: us-east-1
            url: https://k8s-us-east-1.example.com
            region: us-east-1
          - cluster: eu-west-1
            url: https://k8s-eu-west-1.example.com
            region: eu-west-1
  template:
    metadata:
      name: "myapp-{{cluster}}"
    spec:
      project: production
      source:
        repoURL: https://github.com/myorg/k8s-manifests
        path: apps/myapp/base
        targetRevision: HEAD
      destination:
        server: "{{url}}"
        namespace: production
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
```

---

## Step 757: Chaos Engineering for DR Testing

```yaml
# Chaos Mesh: inject failures
apiVersion: chaos-mesh.org/v1alpha1
kind: PodChaos
metadata:
  name: pod-kill-test
  namespace: chaos-testing
spec:
  action: pod-kill
  mode: random-max-percent
  value: "20"
  selector:
    namespaces:
      - production
    labelSelectors:
      app: myapp
  scheduler:
    cron: "@every 10m"

---
apiVersion: chaos-mesh.org/v1alpha1
kind: NetworkChaos
metadata:
  name: network-delay-test
  namespace: chaos-testing
spec:
  action: delay
  mode: all
  selector:
    namespaces:
      - production
    labelSelectors:
      tier: database
  delay:
    latency: 100ms
    correlation: "25"
    jitter: 20ms
  direction: to
  target:
    mode: all
    selector:
      namespaces:
        - production
      labelSelectors:
        tier: backend
  duration: 5m
```

---

## Step 758: Cluster API for DR

```yaml
# Cluster API: declarative cluster lifecycle
apiVersion: cluster.x-k8s.io/v1beta1
kind: Cluster
metadata:
  name: production-cluster
  namespace: clusters
spec:
  clusterNetwork:
    pods:
      cidrBlocks: ["10.244.0.0/16"]
    services:
      cidrBlocks: ["10.96.0.0/12"]
  infrastructureRef:
    apiVersion: infrastructure.cluster.x-k8s.io/v1beta2
    kind: AWSCluster
    name: production-cluster
  controlPlaneRef:
    apiVersion: controlplane.cluster.x-k8s.io/v1beta2
    kind: KubeadmControlPlane
    name: production-control-plane

---
# DR scenario with Cluster API + GitOps:
# 1. New cluster from CAPI definition (15-20 min)
# 2. ArgoCD syncs all K8s manifests from Git
# 3. Velero restores PersistentVolumes
# 4. DNS failover completes
```

---

## Step 759: DR Alerts

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: dr-readiness-alerts
  namespace: monitoring
spec:
  groups:
    - name: disaster-recovery
      rules:
        - alert: VeleroBackupFailed
          expr: velero_backup_failure_total > 0
          for: 1m
          annotations:
            summary: "Velero backup failed"
          labels:
            severity: critical

        - alert: VeleroBackupMissing
          expr: time() - velero_backup_last_successful_timestamp > 86400
          for: 1m
          annotations:
            summary: "No successful Velero backup in 24 hours"
          labels:
            severity: critical

        - alert: EtcdBackupStale
          expr: time() - etcd_backup_last_success_timestamp > 21600
          for: 1m
          annotations:
            summary: "etcd backup older than 6 hours"
          labels:
            severity: warning
```

---

## Step 760: Workshop - DR Runbook ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: dr-runbook
  namespace: operations
data:
  checklist.yaml: |
    monthly_dr_test:
      - Run Velero backup
      - Restore to test cluster
      - Verify all services functional
      - Measure actual RTO/RPO vs targets
      - Test failback procedure
      - Update runbook with lessons learned
    
    scenario_namespace_corruption:
      rto: 30 minutes
      rpo: 1 hour
      steps:
        - Identify affected namespace
        - velero restore create --from-backup daily-backup
        - Verify restore and smoke test
    
    scenario_cluster_failure:
      rto: 2 hours
      rpo: 6 hours
      steps:
        - Provision new control plane nodes
        - Restore etcd from snapshot
        - Update kubeconfig and verify health
    
    scenario_region_failure:
      rto: 4 hours
      rpo: 1 hour
      steps:
        - Activate DR region
        - DNS failover via Route53 health check
        - Scale up DR cluster
        - Restore latest Velero backup
        - Verify applications and update DNS TTLs
```

---

## 📊 สรุป Part 79

| Component | Tool | Schedule | Storage |
|-----------|------|----------|---------|
| etcd | etcdctl | Every 6h | S3 |
| PersistentVolumes | Velero + CSI snapshots | Daily | S3 |
| K8s objects | Velero | Daily | S3 |
| Cluster infra | Cluster API + Git | On change | Git |
| Secrets | ESO + Vault | Continuous | Vault |

---

## 🔗 ต่อไป
- [Part 80: Advanced Security Hardening](./part-80-advanced-security.md)

---
*Part 79 | Steps 751-760 | Disaster Recovery | Educational Content*

> เนื้อหาด้าน security ในหลักสูตรนี้มีไว้เพื่อการศึกษาและการทดสอบระบบที่ได้รับอนุญาตเท่านั้น การนำไปใช้โจมตีระบบที่ไม่ได้รับอนุญาตเป็นสิ่งผิดกฎหมาย
