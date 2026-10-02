# Part 23: Pods Advanced
## Steps 221-230: Pod เชิงลึก, Init Containers, Sidecars, และ Probes

---

## 📖 บทนำ

Pod เป็น unit เล็กที่สุดใน Kubernetes แต่มี features มากมายที่ช่วยให้ application ทำงานได้อย่างเสถียรและปลอดภัย

---

## Step 221: Pod Lifecycle

```yaml
# Pod Phases: Pending → Running → Succeeded/Failed
# ContainerCreating → Running → Terminating

apiVersion: v1
kind: Pod
metadata:
  name: lifecycle-demo
  labels:
    app: demo
spec:
  restartPolicy: Always  # Always (default), OnFailure, Never
  terminationGracePeriodSeconds: 30
  
  containers:
    - name: app
      image: myapp:1.0.0
      lifecycle:
        postStart:
          exec:
            command: ["/bin/sh", "-c", "echo App started >> /tmp/events.log"]
        preStop:
          exec:
            command: ["/bin/sh", "-c", "sleep 5 && echo App stopping >> /tmp/events.log"]
```

---

## Step 222: Init Containers

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-init
  namespace: production
spec:
  initContainers:
    # Init 1: รอ database พร้อม
    - name: wait-for-db
      image: busybox:1.35
      command:
        - sh
        - -c
        - |
          until nc -z postgres 5432; do
            echo "Waiting for postgres..."
            sleep 2
          done
          echo "Postgres is ready!"
    
    # Init 2: รัน migrations
    - name: run-migrations
      image: myapp:1.0.0
      command: ["python", "manage.py", "migrate", "--noinput"]
      env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: database-url
    
    # Init 3: clone config
    - name: clone-config
      image: alpine/git:2.40.1
      command:
        - sh
        - -c
        - |
          git clone https://github.com/myorg/configs.git /config
          cp /config/app-config.yaml /shared/
      volumeMounts:
        - name: shared-config
          mountPath: /shared
  
  containers:
    - name: app
      image: myapp:1.0.0
      volumeMounts:
        - name: shared-config
          mountPath: /app/config
  
  volumes:
    - name: shared-config
      emptyDir: {}
```

---

## Step 223: Sidecar Containers

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-sidecars
spec:
  containers:
    # Main app container
    - name: app
      image: myapp:1.0.0
      ports:
        - containerPort: 8080
      volumeMounts:
        - name: logs
          mountPath: /var/log/app
    
    # Sidecar 1: Log shipping
    - name: log-shipper
      image: fluent/fluentd:v1.16
      volumeMounts:
        - name: logs
          mountPath: /var/log/app
          readOnly: true
      resources:
        requests:
          cpu: 50m
          memory: 64Mi
        limits:
          cpu: 100m
          memory: 128Mi
    
    # Sidecar 2: Metrics exporter
    - name: nginx-exporter
      image: nginx/nginx-prometheus-exporter:0.11
      args:
        - -nginx.scrape-uri=http://localhost:8080/nginx_status
      ports:
        - containerPort: 9113
          name: metrics
  
  volumes:
    - name: logs
      emptyDir: {}

---
# Native Sidecar (K8s 1.29+)
apiVersion: v1
kind: Pod
metadata:
  name: native-sidecar
spec:
  initContainers:
    - name: log-shipper
      image: fluent/fluentd:v1.16
      restartPolicy: Always  # นี่คือ native sidecar
      volumeMounts:
        - name: logs
          mountPath: /var/log/app
  containers:
    - name: app
      image: myapp:1.0.0
      volumeMounts:
        - name: logs
          mountPath: /var/log/app
  volumes:
    - name: logs
      emptyDir: {}
```

---

## Step 224: Health Probes

```yaml
# Health Probes 3 ประเภท:
# 1. startupProbe: ให้ app เวลา startup
# 2. livenessProbe: restart container ถ้า fail
# 3. readinessProbe: ถอด pod จาก Service ถ้า fail

apiVersion: apps/v1
kind: Deployment
metadata:
  name: healthy-app
spec:
  replicas: 3
  template:
    spec:
      containers:
        - name: app
          image: myapp:1.0.0
          
          startupProbe:
            httpGet:
              path: /healthz
              port: 8080
            failureThreshold: 30    # 30 * 10s = 5 นาที สำหรับ startup
            periodSeconds: 10
          
          livenessProbe:
            httpGet:
              path: /healthz
              port: 8080
            periodSeconds: 10
            failureThreshold: 3
            timeoutSeconds: 5
          
          readinessProbe:
            httpGet:
              path: /readyz
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 5
            failureThreshold: 3
            successThreshold: 2    # ต้องผ่าน 2 ครั้งติดกัน

---
# TCP probe
livenessProbe:
  tcpSocket:
    port: 5432
  initialDelaySeconds: 30

---
# gRPC probe (K8s 1.24+)
livenessProbe:
  grpc:
    port: 50051
    service: ""
  initialDelaySeconds: 10
```

---

## Step 225: Resource Management

```yaml
# QoS Classes:
# Guaranteed: requests == limits
# Burstable: limits > requests
# BestEffort: ไม่มี requests หรือ limits

apiVersion: v1
kind: Pod
metadata:
  name: resource-managed
spec:
  containers:
    - name: app
      image: myapp:1.0.0
      resources:
        requests:
          cpu: "250m"
          memory: "256Mi"
          ephemeral-storage: "1Gi"
        limits:
          cpu: "1000m"
          memory: "512Mi"
          ephemeral-storage: "2Gi"

---
# Memory-intensive app
apiVersion: v1
kind: Pod
metadata:
  name: memory-app
spec:
  containers:
    - name: app
      image: memory-hungry:1.0
      resources:
        requests:
          memory: "1Gi"
        limits:
          memory: "2Gi"  # ถ้าเกิน → OOMKilled → restart
      env:
        - name: JAVA_OPTS
          value: "-Xmx1536m -Xms512m"
```

---

## Step 226: Pod Security Context

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  securityContext:
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
    runAsNonRoot: true
    seccompProfile:
      type: RuntimeDefault
    supplementalGroups: [4000]
  
  containers:
    - name: app
      image: myapp:1.0.0
      securityContext:
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        runAsNonRoot: true
        runAsUser: 1000
        capabilities:
          drop:
            - ALL
          add:
            - NET_BIND_SERVICE
      
      volumeMounts:
        - name: tmp
          mountPath: /tmp
        - name: cache
          mountPath: /app/cache
  
  volumes:
    - name: tmp
      emptyDir: {}
    - name: cache
      emptyDir:
        sizeLimit: 500Mi
```

---

## Step 227: Environment Variables (Downward API)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: env-demo
spec:
  containers:
    - name: app
      image: myapp:1.0.0
      env:
        - name: APP_NAME
          value: "my-application"
        
        - name: DB_HOST
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: database.host
        
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: db-password
        
        # Downward API - Pod metadata
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        
        - name: CPU_LIMIT
          valueFrom:
            resourceFieldRef:
              containerName: app
              resource: limits.cpu
      
      envFrom:
        - configMapRef:
            name: app-config
          prefix: APP_
        - secretRef:
            name: app-secrets
```

---

## Step 228: Volume Mounts

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: volumes-demo
spec:
  containers:
    - name: app
      image: myapp:1.0.0
      volumeMounts:
        - name: config
          mountPath: /app/config
          readOnly: true
        - name: secrets
          mountPath: /app/secrets
          readOnly: true
        - name: data
          mountPath: /data
        # Mount single file
        - name: nginx-conf
          mountPath: /etc/nginx/nginx.conf
          subPath: nginx.conf
          readOnly: true
  
  volumes:
    - name: config
      configMap:
        name: app-config
        defaultMode: 0644
    
    - name: secrets
      secret:
        secretName: app-secrets
        defaultMode: 0400
    
    - name: data
      persistentVolumeClaim:
        claimName: app-data-pvc
    
    # Projected volume
    - name: projected
      projected:
        sources:
          - configMap:
              name: app-config
          - secret:
              name: app-secrets
          - serviceAccountToken:
              path: token
              expirationSeconds: 3600
              audience: my-service
```

---

## Step 229: Pod Disruption Budget

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: app-pdb
  namespace: production
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: my-app

---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: zookeeper-pdb
spec:
  maxUnavailable: 1
  selector:
    matchLabels:
      app: zookeeper
```

---

## Step 230: Workshop - Production-Ready Pod

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: production-app
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: production-app
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: production-app
        version: "1.0.0"
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9090"
    spec:
      serviceAccountName: production-app
      automountServiceAccountToken: false
      
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 2000
        seccompProfile:
          type: RuntimeDefault
      
      terminationGracePeriodSeconds: 60
      
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                topologyKey: kubernetes.io/hostname
                labelSelector:
                  matchLabels:
                    app: production-app
      
      initContainers:
        - name: wait-for-db
          image: busybox:1.35
          command: ["sh", "-c", "until nc -z postgres 5432; do sleep 1; done"]
          securityContext:
            runAsNonRoot: true
            runAsUser: 1000
            allowPrivilegeEscalation: false
      
      containers:
        - name: app
          image: myapp:1.0.0
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 8080
              name: http
          
          resources:
            requests:
              cpu: "250m"
              memory: "256Mi"
            limits:
              cpu: "1000m"
              memory: "512Mi"
          
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop:
                - ALL
          
          startupProbe:
            httpGet:
              path: /healthz
              port: 8080
            failureThreshold: 30
            periodSeconds: 10
          
          livenessProbe:
            httpGet:
              path: /healthz
              port: 8080
            periodSeconds: 10
            failureThreshold: 3
          
          readinessProbe:
            httpGet:
              path: /readyz
              port: 8080
            periodSeconds: 5
            failureThreshold: 3
          
          lifecycle:
            preStop:
              exec:
                command: ["/bin/sh", "-c", "sleep 5"]
          
          volumeMounts:
            - name: tmp
              mountPath: /tmp
            - name: config
              mountPath: /app/config
              readOnly: true
      
      volumes:
        - name: tmp
          emptyDir: {}
        - name: config
          configMap:
            name: production-app-config

---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: production-app-pdb
  namespace: production
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: production-app
```

---

## 📊 สรุป Part 23

| Feature | ใช้เมื่อ |
|---------|--------|
| Init Containers | รอ dependencies, run migrations |
| Sidecar Containers | logging, metrics, service mesh proxy |
| startupProbe | app ที่ start ช้า |
| livenessProbe | detect hung applications |
| readinessProbe | traffic routing control |
| Resources | ป้องกัน resource starvation |
| PodDisruptionBudget | safe maintenance |

---

## 🔗 ต่อไป
- [Part 24: Deployments และ StatefulSets](./part-24-deployments-statefulsets.md)

---
*Part 23 | Steps 221-230 | ระดับสูง*
