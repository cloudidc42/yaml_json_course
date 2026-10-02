# Part 71: Kubernetes Operators
## Steps 671-680: Building and Using Operators

---

## Step 671: Operator Pattern Overview

```
Kubernetes Operator Pattern:
  = CRD (Custom Resource Definition)
  + Custom Controller (watches + reconciles)
  + Domain knowledge encoded in code

Use cases:
  - Database operators (PostgreSQL, MongoDB, Redis)
  - Certificate management (cert-manager)
  - Service mesh (Istio, Linkerd)
  - Backup/restore automation
  - Auto-scaling with custom metrics
  - GitOps (ArgoCD, Flux)

Operator maturity levels (OperatorHub):
  Level 1: Basic Install
  Level 2: Seamless Upgrades
  Level 3: Full Lifecycle
  Level 4: Deep Insights
  Level 5: Auto Pilot

Tools to build operators:
  - Kubebuilder (Go, official)
  - Operator SDK (Go/Ansible/Helm)
  - kopf (Python)
  - Metacontroller (webhook-based)
```

---

## Step 672: Custom Resource Definition (CRD)

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: databaseclusters.db.example.com
spec:
  group: db.example.com
  versions:
    - name: v1alpha1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              required: ["engine", "replicas"]
              properties:
                engine:
                  type: string
                  enum: ["postgresql", "mysql", "redis"]
                version:
                  type: string
                  default: "15"
                replicas:
                  type: integer
                  minimum: 1
                  maximum: 10
                  default: 1
                storage:
                  type: object
                  properties:
                    size:
                      type: string
                      default: "10Gi"
                    storageClass:
                      type: string
                backup:
                  type: object
                  properties:
                    enabled:
                      type: boolean
                      default: false
                    schedule:
                      type: string
                      default: "0 2 * * *"
            status:
              type: object
              properties:
                phase:
                  type: string
                  enum: ["Pending", "Running", "Degraded", "Failed"]
                readyReplicas:
                  type: integer
                connectionString:
                  type: string
                conditions:
                  type: array
                  items:
                    type: object
                    properties:
                      type: {type: string}
                      status: {type: string}
                      reason: {type: string}
                      message: {type: string}
                      lastTransitionTime: {type: string, format: date-time}
      subresources:
        status: {}
        scale:
          specReplicasPath: .spec.replicas
          statusReplicasPath: .status.readyReplicas
      additionalPrinterColumns:
        - name: Engine
          type: string
          jsonPath: .spec.engine
        - name: Replicas
          type: integer
          jsonPath: .spec.replicas
        - name: Status
          type: string
          jsonPath: .status.phase
        - name: Age
          type: date
          jsonPath: .metadata.creationTimestamp
  scope: Namespaced
  names:
    plural: databaseclusters
    singular: databasecluster
    kind: DatabaseCluster
    shortNames: [db, dbc]
```

---

## Step 673: Custom Resource Instances

```yaml
apiVersion: db.example.com/v1alpha1
kind: DatabaseCluster
metadata:
  name: production-postgres
  namespace: databases
  labels:
    app: production
    tier: data
spec:
  engine: postgresql
  version: "15"
  replicas: 3
  storage:
    size: 100Gi
    storageClass: fast-ssd
  backup:
    enabled: true
    schedule: "0 1 * * *"

---
apiVersion: db.example.com/v1alpha1
kind: DatabaseCluster
metadata:
  name: session-cache
  namespace: databases
spec:
  engine: redis
  version: "7"
  replicas: 3
  storage:
    size: 10Gi
```

```bash
# Interact with custom resources
kubectl get databaseclusters -n databases
kubectl get db -n databases
kubectl describe databasecluster production-postgres -n databases
kubectl get db production-postgres -o jsonpath='{.status.connectionString}'
```

---

## Step 674: Operator RBAC

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: database-operator
rules:
  - apiGroups: ["db.example.com"]
    resources: ["databaseclusters", "databaseclusters/status", "databaseclusters/finalizers"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  - apiGroups: ["apps"]
    resources: ["statefulsets"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  - apiGroups: [""]
    resources: ["services", "endpoints", "configmaps"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "create", "delete"]
  - apiGroups: ["batch"]
    resources: ["cronjobs", "jobs"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["create", "patch"]

---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: database-operator
  namespace: database-operator-system

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: database-operator
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: database-operator
subjects:
  - kind: ServiceAccount
    name: database-operator
    namespace: database-operator-system
```

---

## Step 675: Operator Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: database-operator
  namespace: database-operator-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: database-operator
  template:
    metadata:
      labels:
        app: database-operator
    spec:
      serviceAccountName: database-operator
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        seccompProfile:
          type: RuntimeDefault
      containers:
        - name: operator
          image: mycompany/database-operator:v1.2.0
          command: ["/manager"]
          args:
            - "--leader-elect"
            - "--metrics-bind-address=:8080"
            - "--health-probe-bind-address=:8081"
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop: [ALL]
          ports:
            - containerPort: 8080
              name: metrics
            - containerPort: 8081
              name: health
          livenessProbe:
            httpGet:
              path: /healthz
              port: 8081
            initialDelaySeconds: 15
            periodSeconds: 20
          readinessProbe:
            httpGet:
              path: /readyz
              port: 8081
            initialDelaySeconds: 5
            periodSeconds: 10
          resources:
            limits:
              cpu: 500m
              memory: 256Mi
            requests:
              cpu: 100m
              memory: 128Mi
          env:
            - name: WATCH_NAMESPACE
              value: ""
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
      terminationGracePeriodSeconds: 10
```

---

## Step 676: Operator Webhooks

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: database-operator-validating
  annotations:
    cert-manager.io/inject-ca-from: database-operator-system/database-operator-serving-cert
webhooks:
  - name: vdatabasecluster.kb.io
    admissionReviewVersions: ["v1"]
    clientConfig:
      service:
        name: database-operator-webhook-service
        namespace: database-operator-system
        path: /validate-db-example-com-v1alpha1-databasecluster
    rules:
      - apiGroups: ["db.example.com"]
        apiVersions: ["v1alpha1"]
        operations: ["CREATE", "UPDATE"]
        resources: ["databaseclusters"]
    failurePolicy: Fail
    sideEffects: None

---
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingWebhookConfiguration
metadata:
  name: database-operator-mutating
  annotations:
    cert-manager.io/inject-ca-from: database-operator-system/database-operator-serving-cert
webhooks:
  - name: mdatabasecluster.kb.io
    admissionReviewVersions: ["v1"]
    clientConfig:
      service:
        name: database-operator-webhook-service
        namespace: database-operator-system
        path: /mutate-db-example-com-v1alpha1-databasecluster
    rules:
      - apiGroups: ["db.example.com"]
        apiVersions: ["v1alpha1"]
        operations: ["CREATE"]
        resources: ["databaseclusters"]
    failurePolicy: Fail
    sideEffects: None
```

---

## Step 677: Finalizers and OwnerReferences

```yaml
# CR with finalizer (set by operator)
apiVersion: db.example.com/v1alpha1
kind: DatabaseCluster
metadata:
  name: production-postgres
  namespace: databases
  finalizers:
    - db.example.com/cleanup
spec:
  engine: postgresql
  replicas: 3

# Operator logic on delete:
# 1. Detect deletionTimestamp is set
# 2. Run cleanup (snapshot, revoke access)
# 3. Remove finalizer -> K8s completes deletion

---
# StatefulSet owned by DatabaseCluster (cascade delete)
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: production-postgres
  namespace: databases
  ownerReferences:
    - apiVersion: db.example.com/v1alpha1
      kind: DatabaseCluster
      name: production-postgres
      uid: abc-123-def
      controller: true
      blockOwnerDeletion: true
spec:
  serviceName: production-postgres
  replicas: 3
  selector:
    matchLabels:
      app: production-postgres
  template:
    metadata:
      labels:
        app: production-postgres
    spec:
      containers:
        - name: postgres
          image: postgres:15
          env:
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: production-postgres-secret
                  key: password
```

---

## Step 678: Popular Operators

```yaml
# cert-manager: TLS certificate automation
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: my-cert
  namespace: default
spec:
  secretName: my-cert-tls
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
    - example.com
    - www.example.com
  duration: 2160h
  renewBefore: 360h

---
# External Secrets Operator
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: my-external-secret
  namespace: default
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-backend
    kind: SecretStore
  target:
    name: my-secret
    creationPolicy: Owner
  data:
    - secretKey: password
      remoteRef:
        key: secret/myapp
        property: password

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
    repoURL: https://github.com/myorg/myapp
    targetRevision: HEAD
    path: k8s/
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

---

## Step 679: Operator Metrics and Alerting

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: database-operator-metrics
  namespace: database-operator-system
  labels:
    release: prometheus
spec:
  selector:
    matchLabels:
      app: database-operator
  endpoints:
    - port: metrics
      interval: 30s

---
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: database-operator-alerts
  namespace: database-operator-system
spec:
  groups:
    - name: database-operator
      rules:
        - alert: OperatorDown
          expr: absent(up{job="database-operator"}) == 1
          for: 1m
          annotations:
            summary: "Database operator is down"
          labels:
            severity: critical

        - alert: DatabaseClusterDegraded
          expr: db_cluster_status{phase="Degraded"} > 0
          for: 5m
          annotations:
            summary: "DatabaseCluster {{ $labels.name }} is degraded"
          labels:
            severity: warning

        - alert: ReconciliationFailures
          expr: rate(controller_runtime_reconcile_errors_total[5m]) > 0.1
          for: 10m
          annotations:
            summary: "Operator has high reconciliation error rate"
          labels:
            severity: warning

        - alert: DatabaseBackupFailing
          expr: db_backup_last_success_timestamp_seconds < (time() - 86400)
          for: 1h
          annotations:
            summary: "Database backup has not succeeded in 24h"
          labels:
            severity: critical
```

---

## Step 680: Workshop - Operator Best Practices

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: operator-best-practices
data:
  design.yaml: |
    crd_design:
      - Use semantic versioning (v1alpha1 -> v1beta1 -> v1)
      - Add validation with openAPIV3Schema
      - Include status subresource
      - Add printer columns for kubectl output
      - Use finalizers for cleanup
      - Set ownerReferences on child resources
    
    controller_design:
      - Idempotent reconciliation
      - Exponential backoff on errors
      - Record K8s Events for debugging
      - Update status conditions after each phase
      - Leader election for HA
      - Use informers/caches (not direct API calls)
    
    security:
      - Minimal RBAC
      - runAsNonRoot: true
      - readOnlyRootFilesystem: true
      - Validate webhooks with cert-manager
      - Scan operator image with Trivy
    
    observability:
      - Export Prometheus metrics
      - Create ServiceMonitor
      - Log with structured JSON
      - Record K8s Events on state changes
      - Define PrometheusRules
    
    testing:
      - Unit tests for reconciliation logic
      - Integration tests with envtest
      - e2e tests against real cluster
      - Test upgrade paths
      - Test failure/finalizer scenarios

---
# CR with full status conditions
apiVersion: db.example.com/v1alpha1
kind: DatabaseCluster
metadata:
  name: production-postgres
status:
  phase: Running
  readyReplicas: 3
  connectionString: "postgresql://postgres:5432/myapp?sslmode=require"
  conditions:
    - type: Available
      status: "True"
      reason: StatefulSetReady
      message: "All 3 replicas are ready"
      lastTransitionTime: "2024-01-15T10:00:00Z"
    - type: BackupReady
      status: "True"
      reason: BackupCompleted
      message: "Last backup: 2024-01-15T01:00:00Z"
      lastTransitionTime: "2024-01-15T01:05:00Z"
```

---

## 📊 สรุป Part 71

| Component | Role | Example |
|-----------|------|---------|
| CRD | Define custom API | `DatabaseCluster` with schema |
| Controller | Watch & reconcile | Ensure StatefulSet matches spec |
| Webhook | Validate/mutate | Block invalid configs |
| Finalizer | Cleanup on delete | Snapshot before PVC delete |
| OwnerRef | Cascade delete | StatefulSet owned by CR |
| Status | Report state | `phase: Running`, conditions |

---

## 🔗 ต่อไป
- [Part 72: GitOps with ArgoCD and Flux](./part-72-gitops.md)

---
*Part 71 | Steps 671-680 | Kubernetes Operators | Educational Content*
