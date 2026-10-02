# Part 88: Database Patterns on Kubernetes
## Steps 841-850: StatefulSets, Operators, Migrations, Caching

---

## Step 841: Database on Kubernetes Overview

```
Databases on Kubernetes: Trade-offs

Pros:
  - Unified operations (same GitOps tooling)
  - Cost savings (colocate with app)
  - Kubernetes operators for automation
  - Namespace isolation
  - Disaster recovery with Velero

Cons:
  - Complex storage management
  - Performance overhead (network stack)
  - Stateful workloads = harder to reschedule
  - Snapshot/PITR complexity

When to run DB on Kubernetes:
  - Dev/test environments: always
  - Production: use operators (cloudnative-pg, redis-operator)
  - Large datasets: consider managed service (RDS, Cloud SQL)

Patterns:
  1. StatefulSet: ordered pod management, stable network IDs
  2. Operators: CloudNativePG, Strimzi, Redis Operator
  3. Sidecar: Backup agent, metrics exporter
  4. Init Container: Schema migration before app starts
  5. PodDisruptionBudget: prevent data loss during maintenance

Storage:
  - WaitForFirstConsumer: avoid cross-AZ PV binding
  - gp3 for most databases
  - io2 (30000 IOPS) for high-throughput DBs
  - local-path for latency-critical (Redis, ClickHouse)
```

---

## Step 842: PostgreSQL StatefulSet

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: databases
spec:
  serviceName: postgres
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      securityContext:
        runAsUser: 999
        fsGroup: 999
      
      initContainers:
        - name: init-permissions
          image: busybox:latest
          command: ["sh", "-c", "chown -R 999:999 /var/lib/postgresql/data"]
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
      
      containers:
        - name: postgres
          image: postgres:16-alpine
          env:
            - name: POSTGRES_DB
              value: myapp
            - name: POSTGRES_USER
              valueFrom:
                secretKeyRef:
                  name: postgres-credentials
                  key: username
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-credentials
                  key: password
            - name: PGDATA
              value: /var/lib/postgresql/data/pgdata
          ports:
            - containerPort: 5432
          resources:
            requests:
              cpu: 500m
              memory: 1Gi
            limits:
              cpu: 2000m
              memory: 4Gi
          readinessProbe:
            exec:
              command: [pg_isready, -U, postgres]
            initialDelaySeconds: 10
            periodSeconds: 5
          livenessProbe:
            exec:
              command: [pg_isready, -U, postgres]
            initialDelaySeconds: 30
            periodSeconds: 10
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
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: databases
spec:
  clusterIP: None
  selector:
    app: postgres
  ports:
    - port: 5432
```

---

## Step 843: Database Schema Migrations

```yaml
# Init container: run migrations before app starts
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: production
spec:
  template:
    spec:
      initContainers:
        - name: migrate
          image: myapp:1.0
          command: ["python", "manage.py", "migrate", "--noinput"]
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: db-credentials
                  key: url
          resources:
            requests:
              cpu: 100m
              memory: 256Mi
      containers:
        - name: app
          image: myapp:1.0

---
# Helm hook: migration Job
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migrate
  annotations:
    helm.sh/hook: pre-install,pre-upgrade
    helm.sh/hook-weight: "-10"
    helm.sh/hook-delete-policy: hook-succeeded
spec:
  backoffLimit: 3
  activeDeadlineSeconds: 300
  template:
    spec:
      restartPolicy: OnFailure
      containers:
        - name: migrate
          image: myapp:1.0
          command: ["python", "manage.py", "migrate", "--noinput"]
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: db-credentials
                  key: url
```

---

## Step 844: Redis Cluster

```yaml
apiVersion: databases.spotahome.com/v1
kind: RedisFailover
metadata:
  name: redis-cluster
  namespace: production
spec:
  sentinel:
    replicas: 3
    resources:
      requests:
        cpu: 100m
        memory: 64Mi
  
  redis:
    replicas: 3
    image: redis:7-alpine
    resources:
      requests:
        cpu: 200m
        memory: 256Mi
      limits:
        cpu: 500m
        memory: 1Gi
    
    storage:
      persistentVolumeClaim:
        metadata:
          name: redis-data
        spec:
          accessModes: [ReadWriteOnce]
          storageClassName: gp3
          resources:
            requests:
              storage: 20Gi
    
    customConfig:
      - "maxmemory 800mb"
      - "maxmemory-policy allkeys-lru"
      - "appendonly yes"

---
apiVersion: v1
kind: ConfigMap
metadata:
  name: redis-config
  namespace: production
data:
  REDIS_HOST: rfs-redis-cluster.production.svc.cluster.local
  REDIS_PORT: "26379"
  REDIS_MASTER_NAME: mymaster
```

---

## Step 845: Connection Pooling

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pgbouncer
  namespace: production
spec:
  replicas: 2
  template:
    spec:
      containers:
        - name: pgbouncer
          image: pgbouncer/pgbouncer:1.21
          env:
            - name: DATABASES_HOST
              value: postgres.databases.svc.cluster.local
            - name: DATABASES_PORT
              value: "5432"
            - name: DATABASES_DBNAME
              value: myapp
            - name: DATABASES_USER
              valueFrom:
                secretKeyRef:
                  name: db-credentials
                  key: username
            - name: DATABASES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: db-credentials
                  key: password
            - name: POOL_MODE
              value: transaction
            - name: MAX_CLIENT_CONN
              value: "1000"
            - name: DEFAULT_POOL_SIZE
              value: "20"
          ports:
            - containerPort: 5432
          readinessProbe:
            tcpSocket:
              port: 5432
            initialDelaySeconds: 5
          resources:
            requests:
              cpu: 100m
              memory: 64Mi
```

---

## Step 846: Database Backup

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: postgres-backup
  namespace: databases
spec:
  schedule: "0 2 * * *"
  concurrencyPolicy: Forbid
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: backup
              image: postgres:16-alpine
              command:
                - sh
                - -c
                - |
                  DATE=$(date +%Y-%m-%d-%H%M%S)
                  BACKUP_FILE="/tmp/backup-${DATE}.sql.gz"
                  pg_dump \
                    -h postgres.databases.svc.cluster.local \
                    -U $POSTGRES_USER \
                    $POSTGRES_DB \
                    | gzip > $BACKUP_FILE
                  aws s3 cp $BACKUP_FILE \
                    s3://mycompany-db-backups/postgres/$(date +%Y/%m)/
              env:
                - name: POSTGRES_USER
                  valueFrom:
                    secretKeyRef:
                      name: postgres-credentials
                      key: username
                - name: POSTGRES_DB
                  value: myapp
                - name: PGPASSWORD
                  valueFrom:
                    secretKeyRef:
                      name: postgres-credentials
                      key: password
              resources:
                requests:
                  cpu: 200m
                  memory: 256Mi
```

---

## Step 847: Read Replicas

```yaml
# CloudNativePG with replicas
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: production-postgres
  namespace: databases
spec:
  instances: 3  # 1 primary + 2 replicas
  storage:
    storageClass: gp3
    size: 100Gi

---
# Read-write service (primary only)
apiVersion: v1
kind: Service
metadata:
  name: postgres-rw
  namespace: databases
spec:
  selector:
    cnpg.io/cluster: production-postgres
    role: primary
  ports:
    - port: 5432

---
# Read-only service (replicas)
apiVersion: v1
kind: Service
metadata:
  name: postgres-ro
  namespace: databases
spec:
  selector:
    cnpg.io/cluster: production-postgres
    role: replica
  ports:
    - port: 5432

---
# App: use separate read/write connections
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: production
spec:
  template:
    spec:
      containers:
        - name: app
          env:
            - name: DATABASE_WRITE_URL
              value: postgresql://myapp@postgres-rw.databases.svc:5432/myapp
            - name: DATABASE_READ_URL
              value: postgresql://myapp@postgres-ro.databases.svc:5432/myapp
```

---

## Step 848: Database Security

```yaml
# NetworkPolicy: restrict DB access
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: postgres-access
  namespace: databases
spec:
  podSelector:
    matchLabels:
      app: postgres
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: production
          podSelector:
            matchLabels:
              db-client: "true"
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: monitoring
      ports:
        - port: 5432

---
# ExternalSecret: auto-rotate DB password
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: postgres-credentials
  namespace: production
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-backend
    kind: SecretStore
  target:
    name: postgres-credentials
    creationPolicy: Owner
  data:
    - secretKey: url
      remoteRef:
        key: secret/databases/postgres
        property: url
    - secretKey: username
      remoteRef:
        key: secret/databases/postgres
        property: username
    - secretKey: password
      remoteRef:
        key: secret/databases/postgres
        property: password
```

---

## Step 849: ClickHouse Analytics

```yaml
apiVersion: clickhouse.altinity.com/v1
kind: ClickHouseInstallation
metadata:
  name: analytics-cluster
  namespace: analytics
spec:
  configuration:
    clusters:
      - name: analytics
        layout:
          shardsCount: 2
          replicasCount: 2
    
    settings:
      max_memory_usage: 10000000000
      distributed_aggregation_memory_efficient: 1
  
  templates:
    podTemplates:
      - name: default
        spec:
          containers:
            - name: clickhouse
              image: clickhouse/clickhouse-server:23.12
              resources:
                requests:
                  cpu: "2"
                  memory: 8Gi
                limits:
                  cpu: "4"
                  memory: 16Gi
    
    volumeClaimTemplates:
      - name: data-volume-template
        spec:
          accessModes: [ReadWriteOnce]
          storageClassName: gp3
          resources:
            requests:
              storage: 500Gi
```

---

## Step 850: Workshop - Database Monitoring

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: database-alerts
  namespace: monitoring
spec:
  groups:
    - name: databases
      rules:
        - alert: PostgresDown
          expr: pg_up == 0
          for: 1m
          annotations:
            summary: "PostgreSQL {{ $labels.instance }} is down"
          labels:
            severity: critical

        - alert: PostgresHighConnections
          expr: |
            sum(pg_stat_activity_count) by (instance)
            /
            sum(pg_settings_max_connections) by (instance)
            > 0.8
          for: 5m
          annotations:
            summary: "PostgreSQL {{ $labels.instance }} connections > 80%"
          labels:
            severity: warning

        - alert: PostgresReplicationLag
          expr: pg_replication_lag > 30
          for: 5m
          annotations:
            summary: "PostgreSQL replication lag > 30s"
          labels:
            severity: warning

        - alert: RedisMemoryHigh
          expr: |
            redis_memory_used_bytes / redis_memory_max_bytes > 0.9
          for: 5m
          annotations:
            summary: "Redis {{ $labels.instance }} memory > 90%"
          labels:
            severity: warning

        - alert: BackupJobFailed
          expr: |
            kube_job_status_failed{job_name=~".*backup.*"} > 0
          for: 0m
          annotations:
            summary: "Backup job {{ $labels.job_name }} has failed"
          labels:
            severity: critical
```

---

## 📊 สรุป Part 88

| Database | Tool | Use Case |
|---------|------|----------|
| PostgreSQL | CloudNativePG | OLTP, transactional |
| Redis | Redis Operator | Cache, sessions, queues |
| ClickHouse | ClickHouse Operator | Analytics, logs |
| Connection pool | PgBouncer | Reduce DB connections |
| Migrations | Init containers | Schema versioning |
| Backup | CronJob + S3 | PITR, disaster recovery |

---

## 🔗 ต่อไป
- [Part 89: CI/CD Pipelines Advanced](./part-89-cicd-advanced.md)

---
*Part 88 | Steps 841-850 | Database Patterns | Educational Content*
