# Part 40: Kubernetes Operators
## Steps 381-390: Building Custom Controllers and Operators

---

## 📖 บทนำ

Kubernetes Operators ขยาย Kubernetes API ด้วย Custom Resources และ Custom Controllers ช่วย automate การ manage complex stateful applications

> **หมายเหตุ**: เนื้อหานี้มีไว้เพื่อการศึกษาและพัฒนา Operators สำหรับระบบที่ได้รับอนุญาต

---

## Step 381: Operator Pattern

```
Operator Pattern:
┌─────────────────────────────────────────────────────────┐
│                                                         │
│  CRD defines "Database" resource                        │
│  ┌─────────────────────────────────────────────────┐   │
│  │  apiVersion: db.example.com/v1                   │   │
│  │  kind: Database                                  │   │
│  │  spec:                                           │   │
│  │    replicas: 3                                   │   │
│  │    version: "14.5"                               │   │
│  │    storage: 100Gi                                │   │
│  └─────────────────────────────────────────────────┘   │
│                         │                               │
│  Operator watches and reconciles:                       │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Actual State → Desired State                    │   │
│  │                                                  │   │
│  │  Create StatefulSet (3 replicas)                 │   │
│  │  Create Services (headless + regular)            │   │
│  │  Create ConfigMaps (pg_hba.conf, postgresql.conf)│   │
│  │  Configure replication                           │   │
│  │  Handle failover                                 │   │
│  └─────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

---

## Step 382: Custom Resource Definition

```yaml
# CRD - defines the "Database" resource
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: databases.db.example.com
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
              required: ["engine", "version", "storage"]
              properties:
                engine:
                  type: string
                  enum: ["postgresql", "mysql", "redis"]
                version:
                  type: string
                replicas:
                  type: integer
                  minimum: 1
                  maximum: 7
                  default: 1
                storage:
                  type: string
                  pattern: "^[0-9]+(Gi|Ti)$"
                resources:
                  type: object
                  properties:
                    requests:
                      type: object
                      properties:
                        cpu:
                          type: string
                        memory:
                          type: string
                    limits:
                      type: object
                      properties:
                        cpu:
                          type: string
                        memory:
                          type: string
                backup:
                  type: object
                  properties:
                    enabled:
                      type: boolean
                      default: false
                    schedule:
                      type: string
                    s3Bucket:
                      type: string
            status:
              type: object
              properties:
                phase:
                  type: string
                  enum: ["Pending", "Running", "Degraded", "Failed"]
                readyReplicas:
                  type: integer
                endpoint:
                  type: string
                conditions:
                  type: array
                  items:
                    type: object
                    properties:
                      type:
                        type: string
                      status:
                        type: string
                      lastTransitionTime:
                        type: string
                      reason:
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

## Step 383: Custom Resource Instance

```yaml
# Database CR instance
apiVersion: db.example.com/v1
kind: Database
metadata:
  name: my-postgres
  namespace: production
spec:
  engine: postgresql
  version: "16.1"
  replicas: 3
  storage: 100Gi
  resources:
    requests:
      cpu: 500m
      memory: 1Gi
    limits:
      cpu: 2000m
      memory: 4Gi
  backup:
    enabled: true
    schedule: "0 2 * * *"
    s3Bucket: my-backup-bucket
```

---

## Step 384: Operator Controller (Go)

```go
// controller/database_controller.go
package controller

import (
    "context"
    "fmt"
    
    appsv1 "k8s.io/api/apps/v1"
    corev1 "k8s.io/api/core/v1"
    "k8s.io/apimachinery/pkg/api/errors"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/apimachinery/pkg/runtime"
    ctrl "sigs.k8s.io/controller-runtime"
    "sigs.k8s.io/controller-runtime/pkg/client"
    "sigs.k8s.io/controller-runtime/pkg/log"
    
    dbv1 "github.com/example/db-operator/api/v1"
)

type DatabaseReconciler struct {
    client.Client
    Scheme *runtime.Scheme
}

func (r *DatabaseReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    logger := log.FromContext(ctx)
    
    db := &dbv1.Database{}
    if err := r.Get(ctx, req.NamespacedName, db); err != nil {
        if errors.IsNotFound(err) {
            return ctrl.Result{}, nil
        }
        return ctrl.Result{}, err
    }
    
    if db.Status.Phase == "" {
        db.Status.Phase = "Pending"
        if err := r.Status().Update(ctx, db); err != nil {
            return ctrl.Result{}, err
        }
    }
    
    sts, err := r.reconcileStatefulSet(ctx, db)
    if err != nil {
        logger.Error(err, "Failed to reconcile StatefulSet")
        return ctrl.Result{}, err
    }
    
    if err := r.reconcileService(ctx, db); err != nil {
        logger.Error(err, "Failed to reconcile Service")
        return ctrl.Result{}, err
    }
    
    db.Status.ReadyReplicas = int(sts.Status.ReadyReplicas)
    if sts.Status.ReadyReplicas == *sts.Spec.Replicas {
        db.Status.Phase = "Running"
        db.Status.Endpoint = fmt.Sprintf("%s.%s.svc.cluster.local", db.Name, db.Namespace)
    } else {
        db.Status.Phase = "Degraded"
    }
    
    if err := r.Status().Update(ctx, db); err != nil {
        return ctrl.Result{}, err
    }
    
    logger.Info("Reconciliation complete", "database", db.Name, "phase", db.Status.Phase)
    return ctrl.Result{}, nil
}

func (r *DatabaseReconciler) reconcileStatefulSet(ctx context.Context, db *dbv1.Database) (*appsv1.StatefulSet, error) {
    replicas := int32(db.Spec.Replicas)
    
    desired := &appsv1.StatefulSet{
        ObjectMeta: metav1.ObjectMeta{
            Name:      db.Name,
            Namespace: db.Namespace,
        },
        Spec: appsv1.StatefulSetSpec{
            Replicas:    &replicas,
            ServiceName: db.Name + "-headless",
            Selector: &metav1.LabelSelector{
                MatchLabels: map[string]string{"app": db.Name},
            },
            Template: corev1.PodTemplateSpec{
                ObjectMeta: metav1.ObjectMeta{
                    Labels: map[string]string{"app": db.Name},
                },
                Spec: corev1.PodSpec{
                    Containers: []corev1.Container{
                        {
                            Name:      "database",
                            Image:     fmt.Sprintf("%s:%s", db.Spec.Engine, db.Spec.Version),
                            Resources: db.Spec.Resources,
                        },
                    },
                },
            },
        },
    }
    
    if err := ctrl.SetControllerReference(db, desired, r.Scheme); err != nil {
        return nil, err
    }
    
    existing := &appsv1.StatefulSet{}
    err := r.Get(ctx, client.ObjectKeyFromObject(desired), existing)
    if errors.IsNotFound(err) {
        return desired, r.Create(ctx, desired)
    }
    if err != nil {
        return nil, err
    }
    
    existing.Spec = desired.Spec
    return existing, r.Update(ctx, existing)
}

func (r *DatabaseReconciler) SetupWithManager(mgr ctrl.Manager) error {
    return ctrl.NewControllerManagedBy(mgr).
        For(&dbv1.Database{}).
        Owns(&appsv1.StatefulSet{}).
        Owns(&corev1.Service{}).
        Complete(r)
}
```

---

## Step 385: Operator Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: db-operator
  namespace: db-operator-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: db-operator
  template:
    metadata:
      labels:
        app: db-operator
    spec:
      serviceAccountName: db-operator
      securityContext:
        runAsNonRoot: true
        seccompProfile:
          type: RuntimeDefault
      containers:
        - name: manager
          image: myregistry.io/db-operator:1.0
          args:
            - --leader-elect
            - --health-probe-bind-address=:8081
            - --metrics-bind-address=:8080
          securityContext:
            allowPrivilegeEscalation: false
            capabilities:
              drop: ["ALL"]
          livenessProbe:
            httpGet:
              path: /healthz
              port: 8081
            initialDelaySeconds: 15
          readinessProbe:
            httpGet:
              path: /readyz
              port: 8081
            initialDelaySeconds: 5
          resources:
            requests:
              cpu: 10m
              memory: 64Mi
            limits:
              cpu: 500m
              memory: 128Mi
```

---

## Step 386: Finalizers

```go
const databaseFinalizer = "db.example.com/finalizer"

func (r *DatabaseReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    db := &dbv1.Database{}
    if err := r.Get(ctx, req.NamespacedName, db); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }
    
    if !db.DeletionTimestamp.IsZero() {
        if containsString(db.Finalizers, databaseFinalizer) {
            if err := r.cleanupExternalResources(ctx, db); err != nil {
                return ctrl.Result{}, err
            }
            db.Finalizers = removeString(db.Finalizers, databaseFinalizer)
            if err := r.Update(ctx, db); err != nil {
                return ctrl.Result{}, err
            }
        }
        return ctrl.Result{}, nil
    }
    
    if !containsString(db.Finalizers, databaseFinalizer) {
        db.Finalizers = append(db.Finalizers, databaseFinalizer)
        if err := r.Update(ctx, db); err != nil {
            return ctrl.Result{}, err
        }
    }
    
    return r.reconcile(ctx, db)
}
```

---

## Step 387: Status Conditions

```go
func (r *DatabaseReconciler) setCondition(db *dbv1.Database, conditionType string,
    status metav1.ConditionStatus, reason, message string) {
    
    condition := metav1.Condition{
        Type:               conditionType,
        Status:             status,
        Reason:             reason,
        Message:            message,
        LastTransitionTime: metav1.Now(),
    }
    meta.SetStatusCondition(&db.Status.Conditions, condition)
}

// ใช้ใน reconcile:
// r.setCondition(db, "Ready", metav1.ConditionTrue, "DatabaseRunning",
//     "All replicas are running")
```

---

## Step 388-390: Popular Operators

```yaml
# Cert-Manager
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: my-cert
  namespace: production
spec:
  secretName: my-tls-secret
  duration: 2160h
  renewBefore: 360h
  dnsNames:
    - myapp.example.com
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer

---
# Strimzi Kafka Operator
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: my-cluster
  namespace: kafka
spec:
  kafka:
    version: 3.6.0
    replicas: 3
    listeners:
      - name: plain
        port: 9092
        type: internal
        tls: false
      - name: tls
        port: 9093
        type: internal
        tls: true
    config:
      offsets.topic.replication.factor: 3
    storage:
      type: persistent-claim
      size: 100Gi
  zookeeper:
    replicas: 3
    storage:
      type: persistent-claim
      size: 10Gi
  entityOperator:
    topicOperator: {}
    userOperator: {}

---
# ArgoCD Application
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/my-app
    targetRevision: main
    path: k8s/overlays/production
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

---

## 📊 สรุป Part 40

| Component | หน้าที่ |
|-----------|------|
| CRD | กำหนด custom resource types |
| Controller | Reconcile loop: desired vs actual state |
| Finalizer | Cleanup ก่อน delete |
| Status Conditions | รายงาน resource status |
| Owner Reference | Garbage collection |
| Operator SDK | Framework สร้าง operators |

---

## 🔗 ต่อไป
- [Part 41: CRDs และ API Extensions](./part-41-crds.md)

---
*Part 40 | Steps 381-390 | ระดับสูง*
