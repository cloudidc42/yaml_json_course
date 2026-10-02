# Part 31: Kustomize
## Steps 291-300: Kubernetes Configuration Management

---

## 📖 บทนำ

Kustomize เป็น tool สำหรับจัดการ Kubernetes configurations แบบ declarative รองรับการ overlay configurations สำหรับแต่ละ environment โดยไม่ต้อง fork YAML

---

## Step 291: Kustomize Basics

```bash
# Kustomize built-in ใน kubectl (v1.14+)
kubectl apply -k ./overlays/production

# Preview output
kubectl kustomize ./overlays/production

# Diff
kubectl diff -k ./overlays/production
```

```
# โครงสร้างมาตรฐาน
├── base/
│   ├── kustomization.yaml
│   ├── deployment.yaml
│   ├── service.yaml
│   └── configmap.yaml
└── overlays/
    ├── development/
    │   ├── kustomization.yaml
    │   └── deployment-patch.yaml
    ├── staging/
    │   ├── kustomization.yaml
    │   └── deployment-patch.yaml
    └── production/
        ├── kustomization.yaml
        ├── deployment-patch.yaml
        └── hpa.yaml
```

---

## Step 292: Base Resources

```yaml
# base/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: my-app
          image: myapp:latest
          ports:
            - containerPort: 8080
          env:
            - name: LOG_LEVEL
              value: debug
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 500m
              memory: 512Mi
```

```yaml
# base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml
  - service.yaml
  - configmap.yaml

commonLabels:
  app.kubernetes.io/name: my-app
  app.kubernetes.io/part-of: my-platform

images:
  - name: myapp
    newName: myregistry.io/myapp
    newTag: "1.0.0"
```

---

## Step 293: Production Overlay

```yaml
# overlays/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: production

bases:
  - ../../base

resources:
  - hpa.yaml
  - ingress.yaml

patchesStrategicMerge:
  - deployment-patch.yaml

patches:
  - target:
      kind: Deployment
      name: my-app
    patch: |-
      - op: replace
        path: /spec/replicas
        value: 5

images:
  - name: myapp
    newName: myregistry.io/myapp
    newTag: "2.1.0"

configMapGenerator:
  - name: env-config
    envs:
      - production.env
    options:
      disableNameSuffixHash: true

secretGenerator:
  - name: db-secret
    literals:
      - DB_PASSWORD=supersecretprod
    type: Opaque
    options:
      disableNameSuffixHash: true
```

```yaml
# overlays/production/deployment-patch.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 5
  template:
    spec:
      containers:
        - name: my-app
          env:
            - name: LOG_LEVEL
              value: warn
          resources:
            requests:
              cpu: 500m
              memory: 512Mi
            limits:
              cpu: 2000m
              memory: 2Gi
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                topologyKey: kubernetes.io/hostname
                labelSelector:
                  matchLabels:
                    app: my-app
```

---

## Step 294: Development Overlay

```yaml
# overlays/development/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: development

bases:
  - ../../base

patchesStrategicMerge:
  - deployment-patch.yaml

images:
  - name: myapp
    newName: myregistry.io/myapp
    newTag: "dev-latest"

configMapGenerator:
  - name: env-config
    literals:
      - LOG_LEVEL=debug
      - DEBUG=true
    options:
      disableNameSuffixHash: true
```

---

## Step 295: Components (สำหรับ Optional Features)

```yaml
# components/monitoring/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1alpha1
kind: Component

resources:
  - servicemonitor.yaml

patchesStrategicMerge:
  - deployment-monitoring-patch.yaml

---
# components/monitoring/servicemonitor.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: my-app
spec:
  selector:
    matchLabels:
      app: my-app
  endpoints:
    - port: metrics
      interval: 30s
```

```yaml
# overlays/production/kustomization.yaml (ใช้ components)
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: production

bases:
  - ../../base

components:
  - ../../components/monitoring
  - ../../components/autoscaling
  - ../../components/pod-disruption-budget

patchesStrategicMerge:
  - deployment-patch.yaml
```

---

## Step 296: Transformers

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base

namePrefix: prod-
nameSuffix: -v2

replicas:
  - name: my-app
    count: 5

labels:
  - pairs:
      environment: production
      team: platform
    includeSelectors: true
    includeTemplates: true

annotations:
  - pairs:
      monitored: "true"
      backup: "daily"
```

---

## Step 297: ConfigMap และ Secret Generators

```yaml
configMapGenerator:
  # จาก env file
  - name: app-env
    envs:
      - config.env

  # จาก files
  - name: nginx-config
    files:
      - nginx.conf
      - conf.d/default.conf

  # จาก literals
  - name: feature-flags
    literals:
      - FEATURE_A=enabled
      - FEATURE_B=disabled
    options:
      disableNameSuffixHash: true

secretGenerator:
  - name: tls-certs
    files:
      - tls.crt=certs/tls.crt
      - tls.key=certs/tls.key
    type: kubernetes.io/tls

  - name: db-creds
    literals:
      - username=admin
      - password=secret123
    type: Opaque
    options:
      disableNameSuffixHash: true
```

---

## Step 298-299: Multi-Cluster + Flux GitOps

```
# Multi-cluster structure
├── clusters/
│   ├── us-east-1/
│   │   └── kustomization.yaml
│   └── ap-southeast-1/
│       └── kustomization.yaml
└── apps/
    └── my-app/
        └── base/
```

```yaml
# Flux Kustomization
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: my-app-production
  namespace: flux-system
spec:
  interval: 10m
  timeout: 5m
  sourceRef:
    kind: GitRepository
    name: my-repo
  path: ./overlays/production
  prune: true
  wait: true
  healthChecks:
    - apiVersion: apps/v1
      kind: Deployment
      name: my-app
      namespace: production
  postBuild:
    substituteFrom:
      - kind: ConfigMap
        name: cluster-vars
      - kind: Secret
        name: cluster-secrets
```

---

## Step 300: Workshop

```bash
# Create structure
mkdir -p my-platform/{base,components/{monitoring},overlays/{dev,staging,prod}}

# Apply dev
kubectl apply -k my-platform/overlays/dev

# Diff production
kubectl diff -k my-platform/overlays/production

# Apply production
kubectl apply -k my-platform/overlays/production

# Build and validate
kubectl kustomize my-platform/overlays/production | \
  kubectl apply --dry-run=client -f -

# Flux reconcile
flux reconcile kustomization my-app-production --with-source
flux get kustomizations -n flux-system
```

---

## 📊 สรุป Part 31

| Feature | ประโยชน์ |
|---------|----------|
| Base/Overlay | แยก environment config |
| Strategic Merge Patch | อัปเดต resource บางส่วน |
| JSON Patch | เปลี่ยนค่าเฉพาะจุด |
| ConfigMap Generator | จัดการ config อัตโนมัติ |
| Components | Reusable optional features |
| Flux Integration | GitOps with Kustomize |

---

## 🔗 ต่อไป
- [Part 32: Prometheus Monitoring](./part-32-prometheus.md)

---
*Part 31 | Steps 291-300 | ระดับสูง*
