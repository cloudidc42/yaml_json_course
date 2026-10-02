# Part 16: Kubernetes Operators และ CRDs
## Steps 151-160: สร้าง Custom Resources และ Operators

---

## 📖 บทนำ

Kubernetes Operator pattern ช่วยให้เราสามารถ automate การจัดการ complex stateful applications โดยใช้ Kubernetes API และ custom controllers

### Operator Pattern คืออะไร?
```
Human Operator Knowledge
         ↓
   Encoded in Code
         ↓
   Kubernetes Operator
         ↓
  Automated Operations:
  - Install
  - Upgrade
  - Backup
  - Scale
  - Heal
  - Monitor
```

---

## Step 151: Custom Resource Definitions (CRDs)

### CRD พื้นฐาน
```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: databases.example.com
spec:
  group: example.com
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              required: ["engine", "version", "storage"]
              properties:
                engine:
                  type: string
                  enum: ["postgresql", "mysql", "redis"]
                version:
                  type: string
                  pattern: '^\d+\.\d+(\.\d+)?$'
                storage:
                  type: object
                  properties:
                    size:
                      type: string
                    storageClass:
                      type: string
                replicas:
                  type: integer
                  minimum: 1
                  maximum: 10
                  default: 1
                backup:
                  type: object
                  properties:
                    enabled:
                      type: boolean
                      default: false
                    schedule:
                      type: string
            status:
              type: object
              properties:
                phase:
                  type: string
                  enum: ["Pending", "Running", "Failed", "Upgrading"]
                readyReplicas:
                  type: integer
                conditions:
                  type: array
                  items:
                    type: object
                    properties:
                      type:
                        type: string
                      status:
                        type: string
                      message:
                        type: string
      additionalPrinterColumns:
        - name: Engine
          type: string
          jsonPath: .spec.engine
        - name: Version
          type: string
          jsonPath: .spec.version
        - name: Replicas
          type: integer
          jsonPath: .spec.replicas
        - name: Phase
          type: string
          jsonPath: .status.phase
        - name: Age
          type: date
          jsonPath: .metadata.creationTimestamp
      subresources:
        status: {}
        scale:
          specReplicasPath: .spec.replicas
          statusReplicasPath: .status.readyReplicas
  scope: Namespaced
  names:
    plural: databases
    singular: database
    kind: Database
    shortNames:
      - db
```

---

## Step 152: Custom Resource ตัวอย่าง

```yaml
# ใช้ CRD ที่สร้างไว้
apiVersion: example.com/v1
kind: Database
metadata:
  name: my-postgresql
  namespace: data
  labels:
    app: my-app
spec:
  engine: postgresql
  version: "15.3"
  storage:
    size: 100Gi
    storageClass: fast-ssd
  replicas: 3
  backup:
    enabled: true
    schedule: "0 2 * * *"
  resources:
    requests:
      cpu: 500m
      memory: 1Gi
    limits:
      cpu: 2
      memory: 4Gi
  parameters:
    max_connections: "200"
    shared_buffers: "256MB"
    work_mem: "4MB"
```

```bash
# จัดการ Custom Resources
kubectl get databases
kubectl get db          # shortname
kubectl describe database my-postgresql -n data
kubectl get db -o wide  # แสดง additional columns
kubectl delete database my-postgresql -n data
```

---

## Step 153: Controller-Runtime สำหรับ Golang

```go
// main.go - Kubernetes Operator ด้วย Go
package main

import (
    "context"
    "fmt"
    
    "k8s.io/apimachinery/pkg/runtime"
    ctrl "sigs.k8s.io/controller-runtime"
    "sigs.k8s.io/controller-runtime/pkg/client"
    "sigs.k8s.io/controller-runtime/pkg/log/zap"
    
    examplev1 "example.com/database-operator/api/v1"
)

// DatabaseReconciler reconciles Database objects
type DatabaseReconciler struct {
    client.Client
    Scheme *runtime.Scheme
}

// +kubebuilder:rbac:groups=example.com,resources=databases,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=example.com,resources=databases/status,verbs=get;update;patch
// +kubebuilder:rbac:groups=apps,resources=statefulsets,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups="",resources=services;configmaps;secrets,verbs=get;list;watch;create;update;patch;delete

func (r *DatabaseReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    log := ctrl.LoggerFrom(ctx)
    
    // 1. ดึง Database object
    db := &examplev1.Database{}
    if err := r.Get(ctx, req.NamespacedName, db); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }
    
    log.Info("Reconciling Database", "name", db.Name, "engine", db.Spec.Engine)
    
    // 2. สร้างหรืออัพเดต StatefulSet
    if err := r.reconcileStatefulSet(ctx, db); err != nil {
        return ctrl.Result{}, err
    }
    
    // 3. สร้างหรืออัพเดต Services
    if err := r.reconcileServices(ctx, db); err != nil {
        return ctrl.Result{}, err
    }
    
    // 4. Setup backup ถ้า enabled
    if db.Spec.Backup.Enabled {
        if err := r.reconcileBackup(ctx, db); err != nil {
            return ctrl.Result{}, err
        }
    }
    
    // 5. อัพเดต status
    db.Status.Phase = "Running"
    if err := r.Status().Update(ctx, db); err != nil {
        return ctrl.Result{}, err
    }
    
    return ctrl.Result{}, nil
}

func (r *DatabaseReconciler) SetupWithManager(mgr ctrl.Manager) error {
    return ctrl.NewControllerManagedBy(mgr).
        For(&examplev1.Database{}).
        Owns(&appsv1.StatefulSet{}).
        Owns(&corev1.Service{}).
        Complete(r)
}

func main() {
    ctrl.SetLogger(zap.New(zap.UseDevMode(true)))
    
    mgr, err := ctrl.NewManager(ctrl.GetConfigOrDie(), ctrl.Options{
        Scheme: scheme,
    })
    if err != nil {
        panic(err)
    }
    
    if err = (&DatabaseReconciler{
        Client: mgr.GetClient(),
        Scheme: mgr.GetScheme(),
    }).SetupWithManager(mgr); err != nil {
        panic(err)
    }
    
    if err := mgr.Start(ctrl.SetupSignalHandler()); err != nil {
        panic(err)
    }
}
```

---

## Step 154: Operator สำหรับ Python

```python
#!/usr/bin/env python3
"""
Python Kubernetes Operator ด้วย kopf library
pip install kopf kubernetes
"""
import kopf
import kubernetes
import yaml
from typing import Any, Dict

kubernetes.config.load_incluster_config()

apps_v1 = kubernetes.client.AppsV1Api()
core_v1 = kubernetes.client.CoreV1Api()


@kopf.on.create('databases.example.com')
async def create_database(spec: Dict[str, Any], name: str, namespace: str, **kwargs):
    """Handler สำหรับ Database creation"""
    
    engine = spec['engine']
    version = spec['version']
    replicas = spec.get('replicas', 1)
    storage = spec['storage']
    
    kopf.info(f"Creating {engine} database '{name}' v{version}")
    
    # สร้าง StatefulSet
    statefulset = build_statefulset(name, namespace, engine, version, replicas, storage)
    apps_v1.create_namespaced_stateful_set(namespace=namespace, body=statefulset)
    
    # สร้าง Service
    service = build_service(name, namespace, engine)
    core_v1.create_namespaced_service(namespace=namespace, body=service)
    
    # อัพเดต status
    return {'phase': 'Running', 'message': f'{engine} database created successfully'}


@kopf.on.update('databases.example.com')
async def update_database(spec: Dict[str, Any], name: str, namespace: str, **kwargs):
    """Handler สำหรับ Database update"""
    replicas = spec.get('replicas', 1)
    
    # อัพเดต replicas
    patch = {'spec': {'replicas': replicas}}
    apps_v1.patch_namespaced_stateful_set(
        name=name, namespace=namespace, body=patch
    )
    
    return {'phase': 'Running', 'message': f'Updated to {replicas} replicas'}


@kopf.on.delete('databases.example.com')
async def delete_database(name: str, namespace: str, **kwargs):
    """Handler สำหรับ Database deletion"""
    
    # ลบ StatefulSet
    try:
        apps_v1.delete_namespaced_stateful_set(name=name, namespace=namespace)
    except kubernetes.client.exceptions.ApiException:
        pass
    
    # ลบ Service
    try:
        core_v1.delete_namespaced_service(name=name, namespace=namespace)
    except kubernetes.client.exceptions.ApiException:
        pass
    
    kopf.info(f"Database '{name}' deleted")


def build_statefulset(name: str, namespace: str, engine: str, version: str, 
                       replicas: int, storage: dict) -> dict:
    """สร้าง StatefulSet manifest"""
    
    images = {
        'postgresql': f'postgres:{version}-alpine',
        'mysql': f'mysql:{version}',
        'redis': f'redis:{version}-alpine',
    }
    
    return {
        'apiVersion': 'apps/v1',
        'kind': 'StatefulSet',
        'metadata': {
            'name': name,
            'namespace': namespace,
            'labels': {'app': name, 'engine': engine}
        },
        'spec': {
            'serviceName': name,
            'replicas': replicas,
            'selector': {'matchLabels': {'app': name}},
            'template': {
                'metadata': {'labels': {'app': name}},
                'spec': {
                    'containers': [{
                        'name': engine,
                        'image': images[engine],
                        'resources': {
                            'requests': {'cpu': '500m', 'memory': '1Gi'},
                            'limits': {'cpu': '2', 'memory': '4Gi'}
                        }
                    }]
                }
            },
            'volumeClaimTemplates': [{
                'metadata': {'name': 'data'},
                'spec': {
                    'accessModes': ['ReadWriteOnce'],
                    'storageClassName': storage.get('storageClass', 'standard'),
                    'resources': {'requests': {'storage': storage['size']}}
                }
            }]
        }
    }


def build_service(name: str, namespace: str, engine: str) -> dict:
    ports = {
        'postgresql': 5432,
        'mysql': 3306,
        'redis': 6379,
    }
    
    return {
        'apiVersion': 'v1',
        'kind': 'Service',
        'metadata': {'name': name, 'namespace': namespace},
        'spec': {
            'selector': {'app': name},
            'ports': [{'port': ports[engine], 'targetPort': ports[engine]}],
            'clusterIP': 'None'  # Headless service สำหรับ StatefulSet
        }
    }


if __name__ == '__main__':
    kopf.run()
```

---

## Step 155: Popular Kubernetes Operators

### cert-manager
```yaml
# cert-manager: จัดการ TLS certificates อัตโนมัติ

# 2. สร้าง ClusterIssuer สำหรับ Let's Encrypt
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    email: admin@example.com
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-prod-key
    solvers:
      - http01:
          ingress:
            class: nginx

---
# 3. สร้าง Certificate
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: example-tls
  namespace: production
spec:
  secretName: example-tls-secret
  duration: 2160h    # 90 days
  renewBefore: 360h  # 15 days before expiry
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  commonName: example.com
  dnsNames:
    - example.com
    - www.example.com
    - api.example.com
```

### Prometheus Operator
```yaml
# Prometheus Operator: จัดการ Prometheus stack

# ServiceMonitor - บอก Prometheus ว่า scrape อะไร
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: my-app-monitor
  namespace: monitoring
  labels:
    app: my-app
spec:
  selector:
    matchLabels:
      app: my-app
  namespaceSelector:
    matchNames:
      - production
  endpoints:
    - port: metrics
      interval: 30s
      path: /metrics

---
# PrometheusRule - Alert rules
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: my-app-rules
  namespace: monitoring
spec:
  groups:
    - name: my-app
      rules:
        - alert: HighErrorRate
          expr: rate(http_requests_total{status=~"5.."}[5m]) > 0.1
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "High error rate in my-app"
```

---

## Step 156: Operator Lifecycle Manager (OLM)

```yaml
# CatalogSource
apiVersion: operators.coreos.com/v1alpha1
kind: CatalogSource
metadata:
  name: community-operators
  namespace: olm
spec:
  sourceType: grpc
  image: quay.io/operatorhubio/catalog:latest
  displayName: Community Operators
  publisher: OperatorHub.io

---
# Subscription - ติดตั้ง operator จาก catalog
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: prometheus-operator
  namespace: operators
spec:
  channel: stable
  name: prometheus
  source: community-operators
  sourceNamespace: olm
  installPlanApproval: Automatic
```

---

## Step 157: Operator SDK - สร้าง Operator ใหม่

```bash
# สร้าง project ใหม่
mkdir my-operator && cd my-operator
operator-sdk init \
  --domain example.com \
  --repo github.com/myorg/my-operator

# สร้าง API และ Controller
operator-sdk create api \
  --group apps \
  --version v1 \
  --kind MyApp \
  --resource \
  --controller

# Generate code
make generate
make manifests

# Build และ Push image
make docker-build docker-push IMG=registry.io/myorg/my-operator:v0.1.0

# Deploy ไป cluster
make deploy IMG=registry.io/myorg/my-operator:v0.1.0
```

---

## Step 158: Admission Webhooks

### Validating Webhook
```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: pod-validator
  annotations:
    cert-manager.io/inject-ca-from: my-namespace/webhook-tls
webhooks:
  - name: validate.pods.example.com
    admissionReviewVersions: ["v1"]
    clientConfig:
      service:
        name: webhook-service
        namespace: my-namespace
        path: "/validate-pods"
    rules:
      - operations: ["CREATE", "UPDATE"]
        apiGroups: [""]
        apiVersions: ["v1"]
        resources: ["pods"]
    failurePolicy: Fail
    sideEffects: None
    namespaceSelector:
      matchLabels:
        admission: enabled
```

### Webhook Server (Python)
```python
#!/usr/bin/env python3
"""
Kubernetes Admission Webhook Server
pip install flask kubernetes
"""
import json
import base64
from flask import Flask, request, jsonify

app = Flask(__name__)


@app.route('/validate-pods', methods=['POST'])
def validate_pod():
    """Validate pod specs"""
    
    admission_review = request.get_json()
    pod = admission_review['request']['object']
    
    allowed = True
    reason = ""
    
    spec = pod.get('spec', {})
    
    # ตรวจสอบ containers ไม่ใช้ latest tag
    for container in spec.get('containers', []):
        image = container.get('image', '')
        if ':' not in image or image.endswith(':latest'):
            allowed = False
            reason += f"Container '{container['name']}' must use specific image tag. "
    
    # ตรวจสอบ resource limits กำหนดไว้
    for container in spec.get('containers', []):
        resources = container.get('resources', {})
        if not resources.get('limits'):
            allowed = False
            reason += f"Container '{container['name']}' must set resource limits. "
    
    # ตรวจสอบ privileged containers
    for container in spec.get('containers', []):
        security = container.get('securityContext', {})
        if security.get('privileged'):
            allowed = False
            reason += f"Container '{container['name']}' cannot be privileged. "
    
    uid = admission_review['request']['uid']
    response = {
        'apiVersion': 'admission.k8s.io/v1',
        'kind': 'AdmissionReview',
        'response': {
            'uid': uid,
            'allowed': allowed,
        }
    }
    
    if not allowed:
        response['response']['status'] = {
            'code': 400,
            'message': reason.strip()
        }
    
    return jsonify(response)


@app.route('/mutate-pods', methods=['POST'])
def mutate_pod():
    """Mutate pod specs - inject sidecar"""
    
    admission_review = request.get_json()
    uid = admission_review['request']['uid']
    
    patches = []
    
    sidecar = {
        "name": "log-collector",
        "image": "fluent/fluent-bit:2.2",
        "resources": {
            "limits": {"cpu": "100m", "memory": "64Mi"}
        }
    }
    
    containers = admission_review['request']['object']['spec'].get('containers', [])
    if not any(c['name'] == 'log-collector' for c in containers):
        patches.append({
            "op": "add",
            "path": "/spec/containers/-",
            "value": sidecar
        })
    
    patch_bytes = json.dumps(patches).encode()
    patch_b64 = base64.b64encode(patch_bytes).decode()
    
    response = {
        'apiVersion': 'admission.k8s.io/v1',
        'kind': 'AdmissionReview',
        'response': {
            'uid': uid,
            'allowed': True,
            'patchType': 'JSONPatch',
            'patch': patch_b64
        }
    }
    
    return jsonify(response)


if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8443, 
            ssl_context=('/tls/tls.crt', '/tls/tls.key'))
```

---

## Step 159: OPA Gatekeeper - Policy as Code

```yaml
# ConstraintTemplate - กำหนด policy template
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequiredlabels
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredLabels
      validation:
        openAPIV3Schema:
          type: object
          properties:
            labels:
              type: array
              items:
                type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequiredlabels

        violation[{"msg": msg, "details": {"missing_labels": missing}}] {
          provided := {label | input.review.object.metadata.labels[label]}
          required := {label | label := input.parameters.labels[_]}
          missing := required - provided
          count(missing) > 0
          msg := sprintf("You must provide labels: %v", [missing])
        }

---
# Constraint - apply policy
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredLabels
metadata:
  name: pods-must-have-labels
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
    namespaces:
      - production
  parameters:
    labels:
      - app
      - version
      - team
```

---

## Step 160: Workshop - Database Operator

```yaml
# 1. สร้าง CRD
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: postgresqls.db.example.com
spec:
  group: db.example.com
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                version:
                  type: string
                database:
                  type: string
                storage:
                  type: string
                replicas:
                  type: integer
                  default: 1
  scope: Namespaced
  names:
    plural: postgresqls
    singular: postgresql
    kind: PostgreSQL
    shortNames: [pg]

---
# 2. สร้าง PostgreSQL instance
apiVersion: db.example.com/v1
kind: PostgreSQL
metadata:
  name: my-pg
  namespace: data
spec:
  version: "15.3"
  database: myapp
  storage: 50Gi
  replicas: 3
```

---

## 📊 สรุป Part 16

| หัวข้อ | สิ่งที่เรียน |
|--------|------------|
| CRDs | Custom Resource Definitions, validation schema |
| Custom Resources | Create/Use custom objects |
| Go Operator | controller-runtime, reconciliation loop |
| Python Operator | kopf library, event handlers |
| Popular Operators | cert-manager, Prometheus Operator |
| OLM | Operator Lifecycle Manager |
| Admission Webhooks | Validating, Mutating webhooks |
| OPA Gatekeeper | Policy as Code |

---

## 🔗 ต่อไป
- [Part 17: Kubernetes Multi-cluster Management](./part-17-kubernetes-multicluster.md)

---
*Part 16 | Steps 151-160 | ระดับสูง*
