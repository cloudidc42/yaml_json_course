# Part 21: Kubernetes Introduction
## Steps 201-210: เริ่มต้นกับ Kubernetes

---

## 📖 บทนำ

Kubernetes (K8s) คือ open-source container orchestration platform ที่สร้างโดย Google และบริจาคให้ CNCF (Cloud Native Computing Foundation) ในปี 2014

### ทำไมต้อง Kubernetes?
- **Automated deployment**: Deploy containers โดยอัตโนมัติ
- **Scaling**: Scale up/down ตาม load
- **Self-healing**: Restart containers ที่ fail อัตโนมัติ
- **Load balancing**: กระจาย traffic
- **Rolling updates**: Update โดยไม่มี downtime
- **Secret management**: จัดการ credentials อย่างปลอดภัย

---

## Step 201: ประวัติและ Ecosystem

### Timeline
- **2003**: Google ใช้ Borg (internal container orchestration)
- **2013**: Docker release
- **2014**: Kubernetes open-sourced โดย Google
- **2016**: CNCF (Cloud Native Computing Foundation) ก่อตั้ง
- **2018**: Kubernetes 1.0 stable
- **2022**: Kubernetes ลบ Docker shim - ใช้ containerd/CRI-O แทน
- **2024**: Kubernetes 1.29+

### CNCF Landscape
```
Container Runtime:
  - containerd (ค่าเริ่มต้น)
  - CRI-O
  - Docker (via cri-dockerd)

Networking:
  - Calico
  - Flannel
  - Cilium (eBPF-based, modern)

Ingress Controllers:
  - NGINX
  - Traefik
  - Istio (service mesh)

Monitoring:
  - Prometheus + Grafana
  - Datadog

Logging:
  - ELK Stack
  - Loki + Grafana
  - Fluent Bit

Service Mesh:
  - Istio
  - Linkerd

Security:
  - OPA/Gatekeeper
  - Falco
```

---

## Step 202: Kubernetes Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Kubernetes Cluster                    │
│                                                         │
│  ┌──────────────────────────────────┐                   │
│  │         Control Plane            │                   │
│  │                                  │                   │
│  │  ┌──────────────┐  ┌──────────┐  │                   │
│  │  │  API Server  │  │  etcd    │  │                   │
│  │  │  (kube-      │  │(key-value│  │                   │
│  │  │  apiserver)  │  │  store)  │  │                   │
│  │  └──────────────┘  └──────────┘  │                   │
│  │                                  │                   │
│  │  ┌──────────────┐  ┌──────────┐  │                   │
│  │  │  Controller  │  │Scheduler │  │                   │
│  │  │  Manager     │  │          │  │                   │
│  │  └──────────────┘  └──────────┘  │                   │
│  └──────────────────────────────────┘                   │
│                                                         │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐              │
│  │  Worker  │  │  Worker  │  │  Worker  │              │
│  │  Node 1  │  │  Node 2  │  │  Node 3  │              │
│  │ kubelet  │  │ kubelet  │  │ kubelet  │              │
│  │ kube-    │  │ kube-    │  │ kube-    │              │
│  │ proxy    │  │ proxy    │  │ proxy    │              │
│  │ [Pod]    │  │ [Pod]    │  │ [Pod]    │              │
│  └──────────┘  └──────────┘  └──────────┘              │
└─────────────────────────────────────────────────────────┘
```

### Control Plane Components

#### 1. kube-apiserver
- Frontend ของ Kubernetes API
- ทุกอย่างผ่าน API Server: kubectl, controllers, schedulers
- Authentication/Authorization และ Admission Control

#### 2. etcd
- Distributed key-value store
- เก็บ state ทุกอย่างของ cluster
- Critical: ถ้า etcd down, cluster down
- **ต้อง backup etcd สม่ำเสมอ!**

#### 3. kube-scheduler
- ตัดสินใจว่า Pod จะรันบน Node ไหน
- พิจารณา: Resource requirements, constraints, affinity

#### 4. kube-controller-manager
- รัน control loops (reconciliation loops)
- Controllers: ReplicaSet, Deployment, Job, ServiceAccount, Node, etc.

### Worker Node Components

#### 5. kubelet
- Agent บน Node แต่ละตัว
- จัดการ Container lifecycle ผ่าน CRI
- รายงาน Node status กลับ API Server

#### 6. kube-proxy
- Network proxy บน Node
- จัดการ iptables/ipvs rules สำหรับ Services

#### 7. Container Runtime
- containerd (ค่าเริ่มต้นใน modern Kubernetes)
- CRI-O (ตัวเลือก)

---

## Step 203: Kubernetes Objects

### Object Hierarchy
```
Cluster
├── Namespace
│   ├── Pod (smallest deployable unit)
│   ├── ReplicaSet (maintain pod replicas)
│   ├── Deployment (rolling updates)
│   ├── StatefulSet (stateful apps)
│   ├── DaemonSet (run on every node)
│   ├── Job (run to completion)
│   ├── CronJob (scheduled jobs)
│   ├── Service (network endpoint)
│   ├── Ingress (HTTP routing)
│   ├── ConfigMap (configuration data)
│   ├── Secret (sensitive data)
│   ├── PersistentVolumeClaim (storage request)
│   └── ServiceAccount (identity)
├── Node (worker machine)
├── PersistentVolume (storage)
├── StorageClass (storage type)
├── ClusterRole (cluster-wide permissions)
└── ClusterRoleBinding (bind role to user)
```

### Object Metadata
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-deployment
  namespace: default
  labels:
    app: myapp
    version: v1
    environment: production
  annotations:
    kubernetes.io/change-cause: "Initial deployment"
    prometheus.io/scrape: "true"
spec:
  # ... object-specific fields
status:
  # ... auto-populated by system
```

---

## Step 204: ติดตั้ง Local Kubernetes

### Option 1: minikube
```bash
# Linux
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube

# macOS
brew install minikube

minikube start
minikube start --cpus=4 --memory=8192
minikube status
minikube dashboard
```

### Option 2: kind (Kubernetes in Docker)
```bash
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.22.0/kind-linux-amd64
chmod +x ./kind && sudo mv ./kind /usr/local/bin/kind

# Multi-node cluster
cat > kind-config.yaml << EOF
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
- role: worker
- role: worker
EOF

kind create cluster --config kind-config.yaml
```

### Option 3: k3s (Lightweight)
```bash
curl -sfL https://get.k3s.io | sh -
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
```

### ติดตั้ง kubectl
```bash
# Linux
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install kubectl /usr/local/bin/kubectl

# macOS
brew install kubectl

# เพิ่ม autocomplete
echo 'source <(kubectl completion bash)' >> ~/.bashrc
echo 'alias k=kubectl' >> ~/.bashrc
source ~/.bashrc
```

---

## Step 205: kubectl พื้นฐาน

### Cluster Information
```bash
kubectl cluster-info
kubectl get nodes
kubectl describe node <node-name>
kubectl top nodes
kubectl api-resources
kubectl explain pod.spec
```

### Context Management
```bash
kubectl config get-contexts
kubectl config use-context minikube
kubectl config current-context
kubectl config set-context --current --namespace=my-namespace
```

### CRUD Operations
```bash
# Create
kubectl apply -f deployment.yaml
kubectl create deployment nginx --image=nginx

# Read
kubectl get pods
kubectl get pods -o wide
kubectl get pods -o yaml
kubectl get pods --all-namespaces
kubectl get all
kubectl get pods --watch

# Describe
kubectl describe pod <pod-name>
kubectl describe deployment <name>

# Delete
kubectl delete pod <pod-name>
kubectl delete -f deployment.yaml

# Update
kubectl apply -f new-deployment.yaml
kubectl edit deployment <name>
kubectl set image deployment/<name> container=image:tag
```

### Namespaces
```bash
kubectl get namespaces
kubectl create namespace my-app
kubectl get pods -n my-app
kubectl delete namespace my-app

# Default namespaces:
# default     - ถ้าไม่ระบุ namespace
# kube-system - system components
# kube-public - public resources
# kube-node-lease - node heartbeats
```

### Logs และ Exec
```bash
kubectl logs <pod-name>
kubectl logs <pod-name> -c <container-name>
kubectl logs <pod-name> -f
kubectl logs <pod-name> --tail=100
kubectl logs <pod-name> --since=1h
kubectl exec -it <pod-name> -- bash
kubectl port-forward service/<svc-name> 8080:80
```

---

## Step 206: First Kubernetes Deployment

```yaml
# nginx-deployment.yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:1.25-alpine
          ports:
            - containerPort: 80
          resources:
            limits:
              cpu: 200m
              memory: 128Mi
            requests:
              cpu: 50m
              memory: 64Mi
---
apiVersion: v1
kind: Service
metadata:
  name: nginx
spec:
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
  type: NodePort
```

```bash
kubectl apply -f nginx-deployment.yaml
kubectl get deployment nginx
kubectl get pods -l app=nginx
kubectl scale deployment nginx --replicas=5
kubectl set image deployment/nginx nginx=nginx:1.26-alpine
kubectl rollout status deployment/nginx
kubectl rollout undo deployment/nginx
```

---

## Step 207: Understanding Pods

### Single Container Pod
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: simple-pod
  labels:
    app: myapp
spec:
  containers:
    - name: app
      image: nginx:alpine
      ports:
        - containerPort: 80
      env:
        - name: MY_VAR
          value: "hello"
      resources:
        limits:
          cpu: 500m
          memory: 256Mi
        requests:
          cpu: 100m
          memory: 128Mi
```

### Multi-Container Pod
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: multi-container-pod
spec:
  initContainers:
    - name: init-db
      image: busybox
      command: ['sh', '-c', 'until nc -z db-service 5432; do sleep 2; done']
  
  containers:
    - name: app
      image: myapp:1.0
      ports:
        - containerPort: 8000
      volumeMounts:
        - name: shared-data
          mountPath: /data
    
    - name: log-shipper
      image: fluent/fluent-bit:latest
      volumeMounts:
        - name: shared-data
          mountPath: /data
          readOnly: true
    
    - name: metrics
      image: prom/node-exporter:latest
      ports:
        - containerPort: 9100
  
  volumes:
    - name: shared-data
      emptyDir: {}
```

### Pod Lifecycle
```
Pending → Running → Succeeded/Failed
              ↓
           CrashLoopBackOff (ถ้า restart หลายครั้ง)

Pending:    - Pod accepted, waiting to be scheduled
            - Downloading images
Running:    - At least 1 container running
Succeeded:  - All containers exited 0
Failed:     - At least 1 container exited non-0
Unknown:    - Cannot determine state (node lost)
```

---

## Step 208: Labels, Selectors, และ Annotations

### Labels
```yaml
metadata:
  labels:
    app.kubernetes.io/name: myapp
    app.kubernetes.io/instance: myapp-production
    app.kubernetes.io/version: "1.0.0"
    app.kubernetes.io/component: backend
    app.kubernetes.io/part-of: myplatform
    app.kubernetes.io/managed-by: helm
    
    environment: production
    team: backend
    tier: api
```

### Label Selectors
```bash
kubectl get pods -l app=myapp
kubectl get pods -l environment=production,team=backend
kubectl get pods -l "environment in (production, staging)"
kubectl get pods -l "!debug-mode"
```

```yaml
selector:
  matchLabels:
    app: myapp
  matchExpressions:
    - key: environment
      operator: In
      values: [production, staging]
    - key: debug-mode
      operator: DoesNotExist
```

### Annotations
```yaml
metadata:
  annotations:
    kubernetes.io/change-cause: "Update image to v1.2.0"
    prometheus.io/scrape: "true"
    prometheus.io/port: "9090"
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    git.commit: "abc123def"
    owner: "team-backend@company.com"
```

---

## Step 209: Resource Management

### Resource Requests และ Limits
```yaml
spec:
  containers:
    - name: app
      resources:
        requests:
          cpu: 100m
          memory: 128Mi
        limits:
          cpu: 500m
          memory: 512Mi
```

### Quality of Service (QoS)
```yaml
# 1. Guaranteed (requests == limits)
resources:
  requests:
    cpu: 500m
    memory: 256Mi
  limits:
    cpu: 500m
    memory: 256Mi

# 2. Burstable (requests < limits)
resources:
  requests:
    cpu: 100m
  limits:
    cpu: 500m

# 3. BestEffort (ไม่มี requests/limits)
# ไม่ระบุ resources section
```

### LimitRange
```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: my-app
spec:
  limits:
    - type: Container
      default:
        cpu: 500m
        memory: 512Mi
      defaultRequest:
        cpu: 100m
        memory: 128Mi
      max:
        cpu: 2
        memory: 2Gi
      min:
        cpu: 50m
        memory: 64Mi
```

### ResourceQuota
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: namespace-quota
  namespace: my-app
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 4Gi
    limits.cpu: "8"
    limits.memory: 8Gi
    pods: "20"
    services: "10"
    secrets: "20"
    configmaps: "20"
    requests.storage: 50Gi
```

---

## Step 210: First Real Application Deployment

```yaml
# application.yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: workshop
  labels:
    environment: learning

---
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: workshop
data:
  APP_ENV: development
  APP_PORT: "8080"
  LOG_LEVEL: debug

---
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
  namespace: workshop
type: Opaque
data:
  DATABASE_PASSWORD: bXlzZWNyZXRwYXNzd29yZA==
  API_KEY: c2VjcmV0LWFwaS1rZXktMTIz

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: workshop-app
  namespace: workshop
  labels:
    app: workshop-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: workshop-app
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: workshop-app
    spec:
      containers:
        - name: app
          image: nginx:1.25-alpine
          ports:
            - name: http
              containerPort: 80
          envFrom:
            - configMapRef:
                name: app-config
          env:
            - name: DATABASE_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: DATABASE_PASSWORD
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
            limits:
              cpu: 200m
              memory: 128Mi
          readinessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 5
            periodSeconds: 10

---
apiVersion: v1
kind: Service
metadata:
  name: workshop-app
  namespace: workshop
spec:
  selector:
    app: workshop-app
  ports:
    - name: http
      port: 80
      targetPort: 80
  type: ClusterIP
```

```bash
kubectl apply -f application.yaml
kubectl get all -n workshop
kubectl logs -n workshop -l app=workshop-app
kubectl port-forward -n workshop service/workshop-app 8080:80
curl http://localhost:8080
kubectl delete namespace workshop
```

---

## 📊 สรุป Part 21

| หัวข้อ | สิ่งที่เรียน |
|--------|------------|
| Architecture | Control Plane, Worker Nodes, Components |
| Objects | Pod, Deployment, Service hierarchy |
| kubectl | CRUD operations, logs, exec |
| Namespaces | Organization, isolation |
| Resources | CPU/Memory requests/limits, QoS |
| Labels | Selectors, organization |
| First Deploy | Complete app stack |

---

## 🔗 ต่อไป
- [Part 22: Kubernetes Architecture Deep Dive](./part-22-kubernetes-architecture.md)

---
*Part 21 | Steps 201-210 | ระดับกลาง*
