# Part 84: Advanced Helm and Kubernetes Operators
## Steps 801-810: Helm Library Charts, Hooks, Operator SDK, CRDs

---

## Step 801: Advanced Helm Concepts Overview

```
Helm Advanced Topics:

Chart Types:
  Application Chart: deploys workloads (most common)
  Library Chart: shared helpers (type: library)
    - Cannot be installed standalone
    - Provides named templates to other charts

Helm Hooks:
  pre-install / post-install
  pre-upgrade / post-upgrade
  pre-delete / post-delete
  test: run on helm test

Chart Dependencies:
  Chart.yaml dependencies block
  helm dependency update
  Condition flags (redis.enabled: true)

Kubernetes Operators:
  Operator = Controller + CRD
  Extends K8s API for stateful workloads
  
  Operator Frameworks:
    - Operator SDK (Helm, Ansible, Go)
    - Kubebuilder
    - KUDO
  
  Popular Operators:
    - cert-manager: TLS certificate management
    - prometheus-operator: monitoring stack
    - strimzi: Apache Kafka
    - cloudnative-pg: PostgreSQL
    - redis-operator: Redis cluster

Operator Maturity Levels:
  Level 1: Basic Install
  Level 2: Seamless Upgrades
  Level 3: Full Lifecycle
  Level 4: Deep Insights
  Level 5: Auto Pilot
```

---

## Step 802: Helm Library Charts

```yaml
# Library chart: Chart.yaml
apiVersion: v2
name: company-library
description: Shared Helm templates for company
type: library
version: 1.0.0

---
# Application chart using library: Chart.yaml
apiVersion: v2
name: myapp
version: 1.0.0
dependencies:
  - name: company-library
    version: ">=1.0.0"
    repository: "https://charts.mycompany.com"

# templates/deployment.yaml:
# {{- include "company-library.deployment" . }}

# _helpers.tpl in library:
# {{- define "company-library.fullname" -}}
# {{- printf "%s-%s" .Release.Name .Chart.Name | trunc 63 | trimSuffix "-" }}
# {{- end }}
#
# {{- define "company-library.labels" -}}
# helm.sh/chart: {{ .Chart.Name }}-{{ .Chart.Version }}
# app.kubernetes.io/name: {{ .Chart.Name }}
# app.kubernetes.io/instance: {{ .Release.Name }}
# app.kubernetes.io/managed-by: {{ .Release.Service }}
# {{- end }}
```

---

## Step 803: Helm Hooks

```yaml
# pre-install hook: database migration
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ .Release.Name }}-db-migrate
  annotations:
    "helm.sh/hook": pre-install,pre-upgrade
    "helm.sh/hook-weight": "-5"
    "helm.sh/hook-delete-policy": hook-succeeded
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: migrate
          image: {{ .Values.image.repository }}:{{ .Values.image.tag }}
          command: ["python", "manage.py", "migrate"]
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: {{ .Release.Name }}-db-secret
                  key: url

---
# post-install hook: Slack notification
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ .Release.Name }}-notify
  annotations:
    "helm.sh/hook": post-install,post-upgrade
    "helm.sh/hook-weight": "10"
    "helm.sh/hook-delete-policy": hook-succeeded,hook-failed
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: notify
          image: curlimages/curl:latest
          command:
            - sh
            - -c
            - |
              curl -X POST $SLACK_WEBHOOK \
                -H 'Content-Type: application/json' \
                -d "{\"text\": \"Deployment {{ .Release.Name }} complete\"}"
          env:
            - name: SLACK_WEBHOOK
              valueFrom:
                secretKeyRef:
                  name: slack-webhooks
                  key: deploy-channel

---
# helm test pod
apiVersion: v1
kind: Pod
metadata:
  name: {{ .Release.Name }}-test
  annotations:
    "helm.sh/hook": test
spec:
  restartPolicy: Never
  containers:
    - name: test
      image: curlimages/curl:latest
      command:
        - curl
        - -f
        - http://{{ .Release.Name }}:{{ .Values.service.port }}/health
```

---

## Step 804: Custom Resource Definitions (CRDs)

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: databases.mycompany.com
spec:
  group: mycompany.com
  names:
    kind: Database
    listKind: DatabaseList
    plural: databases
    singular: database
    shortNames: [db]
  scope: Namespaced
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
              required: [engine, version, storage]
              properties:
                engine:
                  type: string
                  enum: [postgres, mysql, redis]
                version:
                  type: string
                storage:
                  type: string
                  pattern: "^[0-9]+(Gi|Ti)$"
                replicas:
                  type: integer
                  minimum: 1
                  maximum: 5
                  default: 1
                highAvailability:
                  type: boolean
                  default: false
            status:
              type: object
              properties:
                phase:
                  type: string
                  enum: [Pending, Running, Failed]
                endpoint:
                  type: string
      subresources:
        status: {}
      additionalPrinterColumns:
        - name: Engine
          type: string
          jsonPath: .spec.engine
        - name: Phase
          type: string
          jsonPath: .status.phase
        - name: Age
          type: date
          jsonPath: .metadata.creationTimestamp

---
# Use the custom resource
apiVersion: mycompany.com/v1
kind: Database
metadata:
  name: production-postgres
  namespace: databases
spec:
  engine: postgres
  version: "15.4"
  storage: 100Gi
  replicas: 3
  highAvailability: true
```

---

## Step 805: cert-manager Operator

```yaml
# ClusterIssuer: Let's Encrypt production
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@mycompany.com
    privateKeySecretRef:
      name: letsencrypt-prod-account-key
    solvers:
      - dns01:
          route53:
            region: us-east-1
            hostedZoneID: ZXXXXXXXXXXXXX
            accessKeyIDSecretRef:
              name: route53-credentials
              key: access-key-id
            secretAccessKeySecretRef:
              name: route53-credentials
              key: secret-access-key
        selector:
          dnsZones:
            - mycompany.com
      - http01:
          ingress:
            class: nginx
        selector:
          dnsNames:
            - api.mycompany.com

---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: myapp-tls
  namespace: production
spec:
  secretName: myapp-tls-secret
  duration: 2160h
  renewBefore: 360h
  dnsNames:
    - myapp.mycompany.com
    - api.myapp.mycompany.com
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer

---
# Ingress: auto-request cert via annotation
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp
  namespace: production
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  tls:
    - hosts:
        - myapp.mycompany.com
      secretName: myapp-tls-secret
  rules:
    - host: myapp.mycompany.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: myapp
                port:
                  number: 80
```

---

## Step 806: CloudNativePG Operator

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: postgres-cluster
  namespace: databases
spec:
  instances: 3
  postgresql:
    parameters:
      max_connections: "200"
      shared_buffers: 256MB
      effective_cache_size: 768MB
      wal_buffers: 16MB
      work_mem: 4MB
  primaryUpdateStrategy: unsupervised
  primaryUpdateMethod: switchover
  storage:
    storageClass: gp3
    size: 100Gi
  backup:
    retentionPolicy: 30d
    barmanObjectStore:
      destinationPath: s3://mycompany-db-backups/postgres
      s3Credentials:
        inheritFromIAMRole: true
      wal:
        compression: gzip
        maxParallel: 8
  monitoring:
    enablePodMonitor: true
  affinity:
    topologyKey: topology.kubernetes.io/zone
    podAntiAffinityType: required

---
apiVersion: postgresql.cnpg.io/v1
kind: ScheduledBackup
metadata:
  name: postgres-backup
  namespace: databases
spec:
  schedule: "0 2 * * *"
  cluster:
    name: postgres-cluster
  backupOwnerReference: self
  immediate: true
```

---

## Step 807: External DNS Operator

```yaml
# Service: auto-create DNS record
apiVersion: v1
kind: Service
metadata:
  name: myapp
  namespace: production
  annotations:
    external-dns.alpha.kubernetes.io/hostname: myapp.mycompany.com
    external-dns.alpha.kubernetes.io/ttl: "300"
spec:
  type: LoadBalancer
  selector:
    app: myapp
  ports:
    - port: 80

---
# external-dns Helm values.yaml
provider: aws
aws:
  region: us-east-1
  zoneType: public

domainFilters:
  - mycompany.com

annotationFilter: "external-dns.alpha.kubernetes.io/managed-by=external-dns"

txtOwnerId: production-cluster
txtPrefix: edns-

serviceAccount:
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789:role/external-dns

rbac:
  create: true
```

---

## Step 808: Operator Reconcile Loop

```yaml
# Operator controller reconcile (pseudocode Go):
# func (r *DatabaseReconciler) Reconcile(ctx, req) (ctrl.Result, error) {
#   db := &Database{}
#   r.Get(ctx, req.NamespacedName, db)
#
#   // Ensure StatefulSet exists
#   sts := buildStatefulSet(db)
#   ctrl.SetControllerReference(db, sts, r.Scheme)
#   if err := r.Create(ctx, sts); errors.IsAlreadyExists(err) {
#     r.Update(ctx, sts)
#   }
#
#   // Update status
#   if sts.Status.ReadyReplicas == *sts.Spec.Replicas {
#     db.Status.Phase = "Running"
#     db.Status.Endpoint = fmt.Sprintf("%s.%s.svc:5432", db.Name, db.Namespace)
#   }
#   r.Status().Update(ctx, db)
#   return ctrl.Result{RequeueAfter: 30 * time.Second}, nil
# }

---
# Validating webhook
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: database-validator
  annotations:
    cert-manager.io/inject-ca-from: operators/database-operator-serving-cert
webhooks:
  - name: vdatabase.kb.io
    rules:
      - apiGroups: [mycompany.com]
        apiVersions: [v1]
        operations: [CREATE, UPDATE]
        resources: [databases]
    clientConfig:
      service:
        name: database-operator-webhook
        namespace: operators
        path: /validate-mycompany-com-v1-database
    admissionReviewVersions: [v1]
    sideEffects: None
    failurePolicy: Fail
```

---

## Step 809: Operator Lifecycle Manager

```yaml
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: cert-manager
  namespace: operators
spec:
  channel: stable
  name: cert-manager
  source: operatorhubio-catalog
  sourceNamespace: olm
  installPlanApproval: Automatic

---
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: operators-group
  namespace: operators
spec:
  targetNamespaces:
    - operators
    - databases
    - monitoring

---
apiVersion: operators.coreos.com/v1alpha1
kind: CatalogSource
metadata:
  name: company-operators
  namespace: olm
spec:
  sourceType: grpc
  image: registry.mycompany.com/operator-catalog:latest
  displayName: Company Operators
  publisher: MyCompany
  updateStrategy:
    registryPoll:
      interval: 60m
```

---

## Step 810: Workshop - Operator Alerts

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: operator-health
  namespace: monitoring
spec:
  groups:
    - name: operators
      rules:
        - alert: OperatorReconcileErrors
          expr: |
            rate(controller_runtime_reconcile_errors_total[5m]) > 0.1
          for: 5m
          annotations:
            summary: "Operator {{ $labels.controller }} has high reconcile error rate"
          labels:
            severity: warning

        - alert: CertificateExpiringSoon
          expr: |
            certmanager_certificate_expiration_timestamp_seconds - time() < 7 * 24 * 3600
          for: 1h
          annotations:
            summary: "Certificate {{ $labels.name }} expires in < 7 days"
          labels:
            severity: warning

        - alert: CertificateExpired
          expr: |
            certmanager_certificate_expiration_timestamp_seconds - time() < 0
          for: 0m
          annotations:
            summary: "Certificate {{ $labels.name }} has expired!"
          labels:
            severity: critical

        - alert: PostgresClusterNotReady
          expr: |
            cnpg_pg_postmaster_start_time < 0
          for: 5m
          annotations:
            summary: "CloudNativePG cluster {{ $labels.cluster_name }} is not ready"
          labels:
            severity: critical
```

---

## 📊 สรุป Part 84

| Topic | Tool | Use Case |
|-------|------|----------|
| Library Charts | type: library | Shared Helm templates |
| Helm Hooks | pre/post-install | Migrations, notifications |
| CRDs | apiextensions.k8s.io/v1 | Extend K8s API |
| cert-manager | ClusterIssuer, Certificate | Auto TLS management |
| CloudNativePG | Cluster CRD | Production PostgreSQL |
| External DNS | Service annotations | Auto DNS records |
| OLM | Subscription | Manage operator installs |

---

## 🔗 ต่อไป
- [Part 85: Service Mesh with Istio](./part-85-service-mesh-istio.md)

---
*Part 84 | Steps 801-810 | Advanced Helm and Operators | Educational Content*
