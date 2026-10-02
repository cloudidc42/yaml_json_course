# Part 50: Disaster Recovery
## Steps 481-490: Business Continuity & DR Planning

---

## 📖 บทนำ

Disaster Recovery (DR) ใน Kubernetes ครอบคลุม backup/restore ของ etcd, application state, persistent volumes, configuration และการทำ failover ข้าม clusters เพื่อ business continuity

> **หมายเหตุ**: เนื้อหาด้าน security ในหลักสูตรนี้มีไว้เพื่อการศึกษาและการทดสอบระบบที่ได้รับอนุญาตเท่านั้น การนำไปใช้โจมตีระบบที่ไม่ได้รับอนุญาตเป็นสิ่งผิดกฎหมาย

---

## Step 481: DR Concepts & RTO/RPO

```
DR Terminology:

RTO (Recovery Time Objective)   = เวลาสูงสุดที่ยอมรับได้ในการ recovery
RPO (Recovery Point Objective)  = ข้อมูลสูงสุดที่ยอมสูญเสียได้

DR Strategies:
  Backup-Restore:   RTO: hours   RPO: hours   Cost: $
  Pilot Light:      RTO: 10-30m  RPO: minutes Cost: $$
  Warm Standby:     RTO: minutes RPO: seconds  Cost: $$$
  Multi-site:       RTO: seconds RPO: near-0   Cost: $$$$

Kubernetes DR Layers:
  1. etcd backup (cluster state)
  2. Application backup (PVCs, databases)
  3. Configuration backup (Helm values, Secrets)
  4. Container image registry mirror
  5. DNS & Load Balancer failover
```

---

## Step 482: etcd Backup & Restore

```yaml
# etcd backup CronJob (ทุก 6 ชั่วโมง)
apiVersion: batch/v1
kind: CronJob
metadata:
  name: etcd-backup
  namespace: kube-system
spec:
  schedule: "0 */6 * * *"
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 5
  failedJobsHistoryLimit: 3
  jobTemplate:
    spec:
      template:
        spec:
          hostNetwork: true
          hostPID: true
          tolerations:
            - operator: Exists
              effect: NoSchedule
          nodeSelector:
            node-role.kubernetes.io/control-plane: ""
          restartPolicy: OnFailure
          volumes:
            - name: etcd-certs
              hostPath:
                path: /etc/kubernetes/pki/etcd
            - name: backup-storage
              persistentVolumeClaim:
                claimName: etcd-backup-pvc
          containers:
            - name: backup
              image: registry.k8s.io/etcd:3.5.9-0
              env:
                - name: ETCDCTL_API
                  value: "3"
              command:
                - /bin/sh
                - -c
                - |
                  TIMESTAMP=$(date +%Y%m%d-%H%M%S)
                  BACKUP_FILE="/backup/etcd-snapshot-${TIMESTAMP}.db"
                  
                  etcdctl snapshot save "${BACKUP_FILE}" \
                    --endpoints=https://127.0.0.1:2379 \
                    --cacert=/etc/kubernetes/pki/etcd/ca.crt \
                    --cert=/etc/kubernetes/pki/etcd/server.crt \
                    --key=/etc/kubernetes/pki/etcd/server.key
                  
                  etcdctl snapshot status "${BACKUP_FILE}" --write-out=table
                  
                  aws s3 cp "${BACKUP_FILE}" "s3://my-etcd-backups/${TIMESTAMP}/"
                  
                  find /backup -name "etcd-snapshot-*.db" -mtime +7 -delete
              volumeMounts:
                - name: etcd-certs
                  mountPath: /etc/kubernetes/pki/etcd
                  readOnly: true
                - name: backup-storage
                  mountPath: /backup
```

---

## Step 483: Velero Application Backup

```yaml
# Velero Schedule - full cluster backup
apiVersion: velero.io/v1
kind: Schedule
metadata:
  name: full-cluster-backup
  namespace: velero
spec:
  schedule: "0 1 * * *"
  template:
    ttl: 720h
    storageLocation: default
    snapshotVolumes: true
    volumeSnapshotLocations:
      - aws-default
    includedNamespaces: ["*"]
    excludedNamespaces:
      - kube-system
      - kube-public
      - kube-node-lease
    hooks:
      resources:
        - name: postgres-backup
          includedNamespaces: [production]
          labelSelector:
            matchLabels:
              app: postgres
          pre:
            - exec:
                container: postgres
                command:
                  - /bin/bash
                  - -c
                  - pg_dump -U postgres mydb > /tmp/backup.sql
                timeout: 5m
          post:
            - exec:
                container: postgres
                command:
                  - /bin/bash
                  - -c
                  - rm -f /tmp/backup.sql
                timeout: 1m
```

---

## Step 484: Persistent Volume Backup

```yaml
# VolumeSnapshotClass (CSI snapshots)
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: aws-ebs-vsc
  annotations:
    snapshot.storage.kubernetes.io/is-default-class: "true"
driver: ebs.csi.aws.com
deletionPolicy: Delete
parameters:
  tagSpecification_1: "Name={{ .VolumeSnapshotNamespace }}/{{ .VolumeSnapshotName }}"

---
# VolumeSnapshot (on-demand)
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: postgres-data-snapshot
  namespace: production
spec:
  volumeSnapshotClassName: aws-ebs-vsc
  source:
    persistentVolumeClaimName: postgres-data

---
# Restore from snapshot
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-data-restored
  namespace: production
spec:
  dataSource:
    name: postgres-data-snapshot
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Gi
```

---

## Step 485: Database DR with Patroni

```yaml
# PostgreSQL HA with Patroni
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres-ha
  namespace: production
spec:
  serviceName: postgres-ha
  replicas: 3
  selector:
    matchLabels:
      app: postgres-ha
  template:
    metadata:
      labels:
        app: postgres-ha
    spec:
      containers:
        - name: postgres
          image: registry.opensource.zalan.do/acid/spilo-15:3.0-p1
          ports:
            - containerPort: 5432
              name: postgresql
            - containerPort: 8008
              name: patroni
          env:
            - name: PATRONI_SCOPE
              value: postgres-cluster
            - name: PATRONI_KUBERNETES_POD_IP
              valueFrom:
                fieldRef:
                  fieldPath: status.podIP
            - name: PATRONI_SUPERUSER_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-secrets
                  key: superuser-password
          livenessProbe:
            httpGet:
              path: /liveness
              port: 8008
          readinessProbe:
            httpGet:
              path: /readiness
              port: 8008
  volumeClaimTemplates:
    - metadata:
        name: pgdata
      spec:
        accessModes: [ReadWriteOnce]
        storageClassName: gp3
        resources:
          requests:
            storage: 100Gi
```

---

## Step 486: Cross-region DR

```yaml
# BackupStorageLocation in DR cluster
apiVersion: velero.io/v1
kind: BackupStorageLocation
metadata:
  name: primary-region-backups
  namespace: velero
spec:
  provider: aws
  objectStorage:
    bucket: my-velero-backups-us-east-1
    prefix: production
  config:
    region: us-east-1
  accessMode: ReadOnly

# Restore on DR cluster:
# velero restore create dr-restore \
#   --from-backup daily-backup-20241201010000 \
#   --restore-volumes=true \
#   --namespace-mappings production:production \
#   --wait
```

---

## Step 487: Chaos Engineering (DR Testing)

```yaml
# Chaos Mesh - controlled failure injection
apiVersion: chaos-mesh.org/v1alpha1
kind: PodChaos
metadata:
  name: pod-failure-test
  namespace: production
spec:
  action: pod-failure
  mode: random-max-percent
  value: "30"
  selector:
    namespaces:
      - production
    labelSelectors:
      app: backend
  duration: "2m"
  scheduler:
    cron: "@hourly"

---
apiVersion: chaos-mesh.org/v1alpha1
kind: NetworkChaos
metadata:
  name: test-network-partition
  namespace: production
spec:
  action: partition
  mode: all
  selector:
    namespaces: [production]
    labelSelectors:
      app: database
  direction: both
  target:
    selector:
      namespaces: [production]
      labelSelectors:
        app: backend
  duration: "5m"
```

---

## Step 488: DR Runbook Automation

```yaml
# Argo Workflows DR Runbook
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  name: dr-failover
  namespace: argo
spec:
  entrypoint: dr-failover-steps
  templates:
    - name: dr-failover-steps
      steps:
        - - name: verify-backup
            template: verify-backup
        - - name: notify-team
            template: send-notification
        - - name: restore-apps
            template: velero-restore
        - - name: update-dns
            template: update-route53
        - - name: health-check
            template: verify-services
    
    - name: verify-backup
      container:
        image: amazon/aws-cli:latest
        command: ["aws", "s3", "ls", "s3://my-velero-backups/production/"]
    
    - name: velero-restore
      container:
        image: velero/velero:latest
        command: ["/bin/sh", "-c"]
        args:
          - |
            velero restore create dr-restore-$(date +%s) \
              --from-backup daily-backup-latest \
              --restore-volumes=true \
              --wait
    
    - name: verify-services
      container:
        image: bitnami/kubectl:latest
        command: ["/bin/sh", "-c"]
        args:
          - |
            kubectl wait --for=condition=available deployment \
              --all -n production --timeout=300s
```

---

## Step 489: DR Monitoring

```yaml
# PrometheusRules for DR monitoring
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: dr-monitoring
  namespace: monitoring
spec:
  groups:
    - name: backup-alerts
      rules:
        - alert: EtcdBackupMissing
          expr: |
            (time() - etcd_backup_last_success_timestamp_seconds) > 7 * 3600
          for: 30m
          labels:
            severity: critical
          annotations:
            summary: "etcd backup ไม่สำเร็จใน 7 ชั่วโมง"
        
        - alert: VeleroBackupFailed
          expr: |
            velero_backup_last_status{status="Failed"} == 1
          for: 1m
          labels:
            severity: critical
          annotations:
            summary: "Velero backup {{ $labels.schedule }} ล้มเหลว"
        
        - alert: RPOAtRisk
          expr: |
            (time() - velero_backup_last_success_timestamp_seconds{schedule="full-cluster-backup"}) > 2 * 3600
          for: 15m
          labels:
            severity: warning
          annotations:
            summary: "RPO at risk: backup เก่ากว่า 2 ชั่วโมง"
```

---

## Step 490: DR Test Plan

```yaml
# DR Test Plan Documentation
apiVersion: v1
kind: ConfigMap
metadata:
  name: dr-test-plan
  namespace: kube-system
data:
  runbook.md: |
    # DR Test Runbook
    
    ## Monthly DR Test Steps
    
    ### 1. Verify Backups (5 min)
    - velero backup get | grep SUCCESS
    - aws s3 ls s3://my-velero-backups/ | tail -5
    - etcdctl snapshot status /backup/etcd-latest.db
    
    ### 2. Restore to Test Namespace (30 min)
    - velero restore create test-$(date +%s) \
        --from-backup daily-backup-YYYYMMDD \
        --namespace-mappings production:dr-test \
        --wait
    
    ### 3. Verify Application Health (15 min)
    - kubectl get pods -n dr-test
    - Run smoke tests against dr-test endpoints
    
    ### 4. Test DNS Failover (10 min)
    - Update Route53 to DR cluster
    - Measure actual RTO
    
    ### 5. Cleanup
    - Delete dr-test namespace
    - Revert DNS
    - Document RTO/RPO achieved
    
    ## RTO/RPO Targets
    - RTO: < 30 minutes
    - RPO: < 1 hour
```

---

## 📊 สรุป Part 50

| Component | Strategy | Frequency |
|-----------|----------|----------|
| etcd | Snapshot to S3 | Every 6 hours |
| Application state | Velero backup | Daily |
| PVCs | CSI Volume Snapshots | Daily |
| Databases | App-aware + WAL | Continuous |
| Config/Secrets | GitOps | Real-time |
| DR Test | Full failover drill | Monthly |

---

## 🔗 ต่อไป
- [Part 52: Container Escape Techniques](./part-52-container-escape.md)

---
*Part 50 | Steps 481-490 | ระดับสูง*
