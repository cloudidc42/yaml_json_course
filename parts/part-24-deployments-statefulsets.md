# Part 24: Deployments และ StatefulSets
## Steps 231-240: การจัดการ Workloads อย่างมืออาชีพ

---

## 📖 บทนำ

Deployments และ StatefulSets เป็น workload resources หลักใน Kubernetes ที่มีการทำงานแตกต่างกันตามลักษณะของ application

---

## Step 231: Deployment Strategies

```yaml
# Strategy 1: RollingUpdate (default)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rolling-update-demo
spec:
  replicas: 6
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 2
      maxUnavailable: 1
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
          image: myapp:2.0.0
          readinessProbe:
            httpGet:
              path: /readyz
              port: 8080
            periodSeconds: 5

---
# Strategy 2: Recreate
apiVersion: apps/v1
kind: Deployment
metadata:
  name: recreate-demo
spec:
  strategy:
    type: Recreate
  replicas: 3
  selector:
    matchLabels:
      app: legacy-app
  template:
    metadata:
      labels:
        app: legacy-app
    spec:
      containers:
        - name: app
          image: legacy-app:2.0.0
```

---

## Step 232: Deployment Rollback

```bash
# ดู rollout history
kubectl rollout history deployment/my-app
kubectl rollout history deployment/my-app --revision=2

# Rollback
kubectl rollout undo deployment/my-app
kubectl rollout undo deployment/my-app --to-revision=2

# Pause / Resume
kubectl rollout pause deployment/my-app
kubectl rollout resume deployment/my-app

# Restart (zero-downtime)
kubectl rollout restart deployment/my-app
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  revisionHistoryLimit: 10
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
```

---

## Step 233: Blue-Green Deployment

```yaml
# Blue Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-blue
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
      version: blue
  template:
    metadata:
      labels:
        app: my-app
        version: blue
    spec:
      containers:
        - name: app
          image: myapp:1.0.0

---
# Green Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-green
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
      version: green
  template:
    metadata:
      labels:
        app: my-app
        version: green
    spec:
      containers:
        - name: app
          image: myapp:2.0.0

---
# Service - ชี้ไป Blue ก่อน
apiVersion: v1
kind: Service
metadata:
  name: my-app
  namespace: production
spec:
  selector:
    app: my-app
    version: blue  # เปลี่ยนเป็น green เมื่อพร้อม
  ports:
    - port: 80
      targetPort: 8080
```

---

## Step 234: Canary Deployment

```yaml
# Stable (90% traffic)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-stable
  namespace: production
spec:
  replicas: 9
  selector:
    matchLabels:
      app: my-app
      track: stable
  template:
    metadata:
      labels:
        app: my-app
        track: stable
    spec:
      containers:
        - name: app
          image: myapp:1.0.0

---
# Canary (10% traffic)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-canary
  namespace: production
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-app
      track: canary
  template:
    metadata:
      labels:
        app: my-app
        track: canary
    spec:
      containers:
        - name: app
          image: myapp:2.0.0

---
apiVersion: v1
kind: Service
metadata:
  name: my-app
  namespace: production
spec:
  selector:
    app: my-app  # match ทั้ง stable และ canary
  ports:
    - port: 80
      targetPort: 8080
```

---

## Step 235: StatefulSet

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: data
spec:
  serviceName: postgres-headless
  replicas: 3
  podManagementPolicy: OrderedReady
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      partition: 0
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:15
          ports:
            - containerPort: 5432
          env:
            - name: POSTGRES_DB
              value: mydb
            - name: POSTGRES_USER
              value: user
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: password
            - name: PGDATA
              value: /var/lib/postgresql/data/pgdata
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
          livenessProbe:
            exec:
              command: ["pg_isready", "-U", "user", "-d", "mydb"]
            initialDelaySeconds: 30
          readinessProbe:
            exec:
              command: ["pg_isready", "-U", "user", "-d", "mydb"]
            initialDelaySeconds: 5
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: fast-ssd
        resources:
          requests:
            storage: 50Gi

---
apiVersion: v1
kind: Service
metadata:
  name: postgres-headless
  namespace: data
spec:
  clusterIP: None
  selector:
    app: postgres
  ports:
    - port: 5432
      name: postgres
# DNS: postgres-0.postgres-headless.data.svc.cluster.local
```

---

## Step 236: StatefulSet Update Strategies

```yaml
# Partition strategy - Canary for StatefulSet
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      partition: 2  # อัพเดตเฉพาะ postgres-2+ ก่อน
  # หลังทดสอบแล้ว ลด partition = 0

---
# OnDelete Strategy
apiVersion: apps/v1
kind: StatefulSet
spec:
  updateStrategy:
    type: OnDelete
  # kubectl delete pod postgres-0
  # pod-0 จะถูกสร้างใหม่ด้วย image ใหม่
```

---

## Step 237: DaemonSet

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: node-exporter
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
  template:
    metadata:
      labels:
        app: node-exporter
    spec:
      hostPID: true
      hostNetwork: true
      tolerations:
        - key: node-role.kubernetes.io/control-plane
          operator: Exists
          effect: NoSchedule
      containers:
        - name: node-exporter
          image: prom/node-exporter:v1.7.0
          args:
            - --path.sysfs=/host/sys
            - --path.procfs=/host/proc
          ports:
            - containerPort: 9100
              hostPort: 9100
          securityContext:
            runAsNonRoot: true
            runAsUser: 65534
          volumeMounts:
            - name: proc
              mountPath: /host/proc
              readOnly: true
            - name: sys
              mountPath: /host/sys
              readOnly: true
      volumes:
        - name: proc
          hostPath:
            path: /proc
        - name: sys
          hostPath:
            path: /sys
```

---

## Step 238: Jobs และ CronJobs

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: data-migration
  namespace: production
spec:
  completions: 1
  parallelism: 1
  backoffLimit: 3
  activeDeadlineSeconds: 300
  ttlSecondsAfterFinished: 3600
  template:
    spec:
      restartPolicy: OnFailure
      containers:
        - name: migration
          image: myapp:1.0.0
          command: ["python", "migrate.py"]

---
apiVersion: batch/v1
kind: CronJob
metadata:
  name: backup-job
  namespace: production
spec:
  schedule: "0 2 * * *"
  timeZone: "Asia/Bangkok"
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  startingDeadlineSeconds: 120
  jobTemplate:
    spec:
      backoffLimit: 2
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: backup
              image: backup-tool:1.0
              command:
                - /bin/sh
                - -c
                - backup-tool --destination s3://my-bucket/backups/
```

---

## Step 239: Horizontal Pod Autoscaler (HPA)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: app-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicas: 2
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: AverageValue
          averageValue: "512Mi"
    - type: Pods
      pods:
        metric:
          name: requests_per_second
        target:
          type: AverageValue
          averageValue: "100"
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
        - type: Percent
          value: 100
          periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Percent
          value: 25
          periodSeconds: 60

---
# VPA
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: app-vpa
  namespace: production
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  updatePolicy:
    updateMode: "Auto"
  resourcePolicy:
    containerPolicies:
      - containerName: app
        minAllowed:
          cpu: 100m
          memory: 128Mi
        maxAllowed:
          cpu: 4000m
          memory: 4Gi
```

---

## Step 240: Workshop - Redis StatefulSet

```yaml
apiVersion: v1
kind: Service
metadata:
  name: redis-headless
  namespace: data
spec:
  clusterIP: None
  selector:
    app: redis
  ports:
    - port: 6379

---
apiVersion: v1
kind: ConfigMap
metadata:
  name: redis-config
  namespace: data
data:
  redis.conf: |
    maxmemory 512mb
    maxmemory-policy allkeys-lru
    appendonly yes
    appendfsync everysec

---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis
  namespace: data
spec:
  serviceName: redis-headless
  replicas: 3
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 1000
      containers:
        - name: redis
          image: redis:7-alpine
          command: ["redis-server", "/config/redis.conf"]
          ports:
            - containerPort: 6379
          resources:
            requests:
              cpu: 100m
              memory: 256Mi
            limits:
              cpu: 500m
              memory: 768Mi
          livenessProbe:
            exec:
              command: ["redis-cli", "ping"]
            initialDelaySeconds: 30
          readinessProbe:
            exec:
              command: ["redis-cli", "ping"]
            initialDelaySeconds: 5
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop: [ALL]
          volumeMounts:
            - name: data
              mountPath: /data
            - name: config
              mountPath: /config
              readOnly: true
            - name: tmp
              mountPath: /tmp
      volumes:
        - name: config
          configMap:
            name: redis-config
        - name: tmp
          emptyDir: {}
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: fast-ssd
        resources:
          requests:
            storage: 10Gi
```

---

## 📊 สรุป Part 24

| Workload | ใช้กับ |
|---------|--------|
| Deployment | Stateless apps, web servers, APIs |
| StatefulSet | Databases, message queues |
| DaemonSet | Node-level daemons, log shippers |
| Job | Batch processing, migrations |
| CronJob | Scheduled tasks, backups |
| HPA | Auto-scale stateless workloads |
| VPA | Right-size resource requests |

---

## 🔗 ต่อไป
- [Part 25: ConfigMaps และ Secrets](./part-25-configmaps-secrets.md)

---
*Part 24 | Steps 231-240 | ระดับสูง*
