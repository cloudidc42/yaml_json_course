# Part 10: Kubernetes YAML Advanced
## Steps 91-100: Kubernetes Resources เชิงลึก

---

## Step 91: Kubernetes API Versions

```yaml
# Core API Group (v1)
apiVersion: v1
kind: Pod | Service | ConfigMap | Secret | Namespace |
       PersistentVolume | PersistentVolumeClaim | ServiceAccount

# apps group
apiVersion: apps/v1
kind: Deployment | ReplicaSet | DaemonSet | StatefulSet

# batch group
apiVersion: batch/v1
kind: Job | CronJob

# networking group
apiVersion: networking.k8s.io/v1
kind: Ingress | NetworkPolicy | IngressClass

# rbac group
apiVersion: rbac.authorization.k8s.io/v1
kind: Role | ClusterRole | RoleBinding | ClusterRoleBinding

# autoscaling
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler

# Custom Resources
apiVersion: cert-manager.io/v1
kind: Certificate | ClusterIssuer
```

---

## Step 92: Pod Spec เชิงลึก

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: advanced-pod
  namespace: production
  labels:
    app: myapp
    version: "2.0"
spec:
  nodeSelector:
    kubernetes.io/os: linux
    node-type: high-memory
  
  tolerations:
    - key: "special"
      operator: "Equal"
      value: "true"
      effect: "NoSchedule"
  
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: topology.kubernetes.io/zone
                operator: In
                values: ["ap-southeast-1a", "ap-southeast-1b"]
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 100
          podAffinityTerm:
            labelSelector:
              matchExpressions:
                - key: app
                  operator: In
                  values: ["myapp"]
            topologyKey: kubernetes.io/hostname
  
  serviceAccountName: myapp-sa
  automountServiceAccountToken: false
  
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
    seccompProfile:
      type: RuntimeDefault
  
  initContainers:
    - name: init-db
      image: postgres:15-alpine
      command: ['sh', '-c',
        'until pg_isready -h postgres -p 5432; do echo waiting; sleep 2; done']
  
  containers:
    - name: app
      image: myapp:2.0
      ports:
        - name: http
          containerPort: 8080
        - name: metrics
          containerPort: 9090
      
      env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: password
        - name: APP_CONFIG
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: config.json
      
      envFrom:
        - configMapRef:
            name: app-env-config
        - secretRef:
            name: app-secrets
            optional: true
      
      resources:
        requests:
          cpu: "100m"
          memory: "128Mi"
        limits:
          cpu: "500m"
          memory: "512Mi"
      
      volumeMounts:
        - name: config
          mountPath: /etc/config
          readOnly: true
        - name: data
          mountPath: /data
        - name: tmp
          mountPath: /tmp
      
      startupProbe:
        httpGet:
          path: /health
          port: http
        failureThreshold: 30
        periodSeconds: 10
      
      livenessProbe:
        httpGet:
          path: /health
          port: http
        initialDelaySeconds: 30
        periodSeconds: 30
        timeoutSeconds: 5
        failureThreshold: 3
      
      readinessProbe:
        httpGet:
          path: /ready
          port: http
        initialDelaySeconds: 10
        periodSeconds: 10
      
      lifecycle:
        postStart:
          exec:
            command: ["/bin/sh", "-c", "echo started > /tmp/started"]
        preStop:
          exec:
            command: ["/bin/sh", "-c", "sleep 5"]
      
      securityContext:
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        capabilities:
          drop: ["ALL"]
          add: ["NET_BIND_SERVICE"]
  
  volumes:
    - name: config
      configMap:
        name: app-config
    - name: data
      persistentVolumeClaim:
        claimName: app-data-pvc
    - name: tmp
      emptyDir:
        medium: Memory
        sizeLimit: 100Mi
  
  terminationGracePeriodSeconds: 60
  restartPolicy: Always
```

---

## Step 93: Deployment Advanced

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: production
  annotations:
    kubernetes.io/change-cause: "Deploy v2.0"
spec:
  replicas: 5
  selector:
    matchLabels:
      app: myapp
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 2
      maxUnavailable: 1
  revisionHistoryLimit: 10
  minReadySeconds: 10
  progressDeadlineSeconds: 600
  template:
    metadata:
      labels:
        app: myapp
        version: "2.0"
    spec:
      terminationGracePeriodSeconds: 30
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: myapp
      containers:
        - name: app
          image: myapp:2.0
          resources:
            requests: {cpu: 100m, memory: 128Mi}
            limits: {cpu: 500m, memory: 512Mi}
```

```bash
# Rollout commands
kubectl rollout status deployment/myapp
kubectl rollout history deployment/myapp
kubectl rollout undo deployment/myapp
kubectl rollout undo deployment/myapp --to-revision=2
kubectl rollout restart deployment/myapp
```

---

## Step 94: StatefulSet

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: data
spec:
  serviceName: "postgres"
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
                  name: postgres-credentials
                  key: password
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
          livenessProbe:
            exec:
              command: [pg_isready, -U, postgres]
            initialDelaySeconds: 30
  
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: [ReadWriteOnce]
        storageClassName: fast-ssd
        resources:
          requests:
            storage: 50Gi

---
# Headless Service
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: data
spec:
  clusterIP: None
  selector:
    app: postgres
  ports:
    - port: 5432
# DNS: postgres-0.postgres.data.svc.cluster.local
```

---

## Step 95: DaemonSet

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
      tolerations:
        - key: node-role.kubernetes.io/control-plane
          operator: Exists
          effect: NoSchedule
      hostNetwork: true
      hostPID: true
      containers:
        - name: node-exporter
          image: prom/node-exporter:v1.7.0
          args:
            - --path.procfs=/host/proc
            - --path.sysfs=/host/sys
          ports:
            - containerPort: 9100
              hostPort: 9100
              name: metrics
          volumeMounts:
            - name: proc
              mountPath: /host/proc
              readOnly: true
            - name: sys
              mountPath: /host/sys
              readOnly: true
          securityContext:
            runAsNonRoot: true
            runAsUser: 65534
      volumes:
        - name: proc
          hostPath: {path: /proc}
        - name: sys
          hostPath: {path: /sys}
      nodeSelector:
        kubernetes.io/os: linux
```

---

## Step 96: HorizontalPodAutoscaler

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
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
          averageValue: 400Mi
    - type: Pods
      pods:
        metric:
          name: http_requests_per_second
        target:
          type: AverageValue
          averageValue: "1000"
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
        - type: Pods
          value: 2
          periodSeconds: 60
```

---

## Step 97-100: PDB, ConfigMap, Secret, NetworkPolicy

```yaml
# PodDisruptionBudget
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: myapp-pdb
  namespace: production
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: myapp

---
# ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: production
data:
  DATABASE_HOST: "postgres.data.svc.cluster.local"
  DATABASE_PORT: "5432"
  LOG_LEVEL: "info"
  config.yaml: |
    server:
      port: 8080
      timeout: 30
    features:
      dark_mode: true

---
# Secret
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
  namespace: production
type: Opaque
stringData:
  JWT_SECRET: "my-jwt-secret-key-change-in-production"
  config.json: |
    {"secret_key": "my-secret", "encryption_key": "my-encryption-key"}

---
# Ingress
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
  namespace: production
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/limit-rpm: "100"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  ingressClassName: nginx
  tls:
    - hosts: [myapp.example.com]
      secretName: myapp-tls
  rules:
    - host: myapp.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend-service
                port: {number: 80}
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: api-service
                port: {number: 8080}

---
# NetworkPolicy - Default deny all
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]

---
# Allow frontend → API
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-api
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes: [Ingress]
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend
      ports:
        - protocol: TCP
          port: 8080
```

---

## 📊 สรุป Part 10

| Resource | ใช้สำหรับ |
|---------|-----------|
| Pod | Unit ที่เล็กที่สุด |
| Deployment | Stateless apps |
| StatefulSet | Stateful apps (DB, cache) |
| DaemonSet | ทุก node (monitoring, logging) |
| HPA | Auto-scaling |
| PDB | High availability |
| ConfigMap | Non-sensitive config |
| Secret | Sensitive data |
| Ingress | External HTTP/HTTPS routing |
| NetworkPolicy | Pod-to-pod firewall |

---
*Part 10 | Steps 91-100 | ระดับกลาง*
