# Part 30: Helm Package Manager
## Steps 281-290: จัดการ Kubernetes Applications ด้วย Helm

---

## 📖 บทนำ

Helm เป็น package manager สำหรับ Kubernetes ช่วยให้การ deploy และ manage applications เป็นเรื่องง่าย ลด boilerplate YAML และเพิ่ม reusability

---

## Step 281: Helm Basics

```bash
# เพิ่ม repository
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo add jetstack https://charts.jetstack.io
helm repo update

# ค้นหา charts
helm search repo nginx
helm search hub wordpress

# ดู chart info
helm show chart bitnami/postgresql
helm show values bitnami/postgresql > postgres-default-values.yaml

# ติดตั้ง chart
helm install my-postgres bitnami/postgresql \
  --namespace data \
  --create-namespace \
  --values postgres-values.yaml \
  --version 13.2.0

# ดู releases
helm list -A
helm status my-postgres -n data

# Upgrade
helm upgrade my-postgres bitnami/postgresql \
  --namespace data \
  --values postgres-values.yaml \
  --version 13.3.0

# Rollback
helm rollback my-postgres 1 -n data
helm history my-postgres -n data

# Uninstall
helm uninstall my-postgres -n data
```

---

## Step 282: Chart Structure

```
my-app/
├── Chart.yaml          # Chart metadata
├── values.yaml         # Default configuration
├── charts/             # Chart dependencies
│   └── redis-17.0.0.tgz
├── templates/
│   ├── NOTES.txt
│   ├── _helpers.tpl    # Template helpers
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── configmap.yaml
│   ├── secret.yaml
│   ├── hpa.yaml
│   ├── serviceaccount.yaml
│   └── tests/
│       └── test-connection.yaml
└── .helmignore
```

```yaml
# Chart.yaml
apiVersion: v2
name: my-app
description: My application Helm chart
type: application
version: 1.2.3       # Chart version (semver)
appVersion: "2.0.0"  # App version (ไว้แสดง)
keywords:
  - web
  - api
home: https://github.com/myorg/my-app
maintainers:
  - name: Platform Team
    email: platform@company.com
dependencies:
  - name: redis
    version: "17.x.x"
    repository: https://charts.bitnami.com/bitnami
    condition: redis.enabled
  - name: postgresql
    version: "13.x.x"
    repository: https://charts.bitnami.com/bitnami
    condition: postgresql.enabled
```

---

## Step 283: values.yaml Design

```yaml
# values.yaml
image:
  repository: myregistry.io/myapp
  tag: "1.0.0"
  pullPolicy: IfNotPresent
  pullSecrets: []

replicaCount: 2

nameOverride: ""
fullnameOverride: ""

serviceAccount:
  create: true
  annotations: {}
  name: ""

podAnnotations:
  prometheus.io/scrape: "true"
  prometheus.io/port: "9090"

podSecurityContext:
  runAsNonRoot: true
  runAsUser: 1000
  fsGroup: 2000
  seccompProfile:
    type: RuntimeDefault

securityContext:
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
  capabilities:
    drop:
      - ALL

service:
  type: ClusterIP
  port: 80
  targetPort: 8080
  annotations: {}

ingress:
  enabled: false
  className: nginx
  annotations: {}
  hosts:
    - host: myapp.example.com
      paths:
        - path: /
          pathType: Prefix
  tls: []

resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 100m
    memory: 128Mi

autoscaling:
  enabled: false
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70

config:
  logLevel: info
  database:
    host: ""
    port: 5432
    name: mydb

redis:
  enabled: false

postgresql:
  enabled: false
  auth:
    database: mydb

nodeSelector: {}
tolerations: []
affinity: {}
```

---

## Step 284: Template Helpers

```yaml
# templates/_helpers.tpl
{{/*
Expand the name of the chart.
*/}}
{{- define "my-app.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Create a default fully qualified app name.
*/}}
{{- define "my-app.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- $name := default .Chart.Name .Values.nameOverride }}
{{- if contains $name .Release.Name }}
{{- .Release.Name | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}
{{- end }}

{{/*
Common labels
*/}}
{{- define "my-app.labels" -}}
helm.sh/chart: {{ printf "%s-%s" .Chart.Name .Chart.Version | replace "+" "_" | trunc 63 | trimSuffix "-" }}
{{ include "my-app.selectorLabels" . }}
{{- if .Chart.AppVersion }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
{{- end }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

{{/*
Selector labels
*/}}
{{- define "my-app.selectorLabels" -}}
app.kubernetes.io/name: {{ include "my-app.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}

{{/*
ServiceAccount name
*/}}
{{- define "my-app.serviceAccountName" -}}
{{- if .Values.serviceAccount.create }}
{{- default (include "my-app.fullname" .) .Values.serviceAccount.name }}
{{- else }}
{{- default "default" .Values.serviceAccount.name }}
{{- end }}
{{- end }}
```

---

## Step 285: Deployment Template

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "my-app.fullname" . }}
  namespace: {{ .Release.Namespace }}
  labels:
    {{- include "my-app.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "my-app.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      {{- with .Values.podAnnotations }}
      annotations:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      labels:
        {{- include "my-app.labels" . | nindent 8 }}
    spec:
      {{- with .Values.image.pullSecrets }}
      imagePullSecrets:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      serviceAccountName: {{ include "my-app.serviceAccountName" . }}
      securityContext:
        {{- toYaml .Values.podSecurityContext | nindent 8 }}
      containers:
        - name: {{ .Chart.Name }}
          securityContext:
            {{- toYaml .Values.securityContext | nindent 12 }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - name: http
              containerPort: {{ .Values.service.targetPort }}
              protocol: TCP
          env:
            - name: LOG_LEVEL
              value: {{ .Values.config.logLevel | quote }}
            {{- if .Values.config.database.host }}
            - name: DB_HOST
              value: {{ .Values.config.database.host | quote }}
            {{- else if .Values.postgresql.enabled }}
            - name: DB_HOST
              value: {{ printf "%s-postgresql" .Release.Name | quote }}
            {{- end }}
          livenessProbe:
            httpGet:
              path: /healthz
              port: http
            initialDelaySeconds: 0
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /readyz
              port: http
            initialDelaySeconds: 5
            periodSeconds: 5
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
          volumeMounts:
            - name: tmp
              mountPath: /tmp
      volumes:
        - name: tmp
          emptyDir: {}
      {{- with .Values.nodeSelector }}
      nodeSelector:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.affinity }}
      affinity:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.tolerations }}
      tolerations:
        {{- toYaml . | nindent 8 }}
      {{- end }}
```

---

## Step 286: HPA Template

```yaml
# templates/hpa.yaml
{{- if .Values.autoscaling.enabled }}
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: {{ include "my-app.fullname" . }}
  namespace: {{ .Release.Namespace }}
  labels:
    {{- include "my-app.labels" . | nindent 4 }}
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: {{ include "my-app.fullname" . }}
  minReplicas: {{ .Values.autoscaling.minReplicas }}
  maxReplicas: {{ .Values.autoscaling.maxReplicas }}
  metrics:
    {{- if .Values.autoscaling.targetCPUUtilizationPercentage }}
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: {{ .Values.autoscaling.targetCPUUtilizationPercentage }}
    {{- end }}
    {{- if .Values.autoscaling.targetMemoryUtilizationPercentage }}
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: {{ .Values.autoscaling.targetMemoryUtilizationPercentage }}
    {{- end }}
{{- end }}
```

---

## Step 287: Helm Hooks

```yaml
# templates/pre-upgrade-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ include "my-app.fullname" . }}-migrate
  namespace: {{ .Release.Namespace }}
  labels:
    {{- include "my-app.labels" . | nindent 4 }}
  annotations:
    "helm.sh/hook": pre-install,pre-upgrade
    "helm.sh/hook-weight": "-5"
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
spec:
  template:
    spec:
      restartPolicy: OnFailure
      serviceAccountName: {{ include "my-app.serviceAccountName" . }}
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
      containers:
        - name: migrate
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          command: ["python", "manage.py", "migrate"]
          env:
            - name: DB_URL
              valueFrom:
                secretKeyRef:
                  name: {{ include "my-app.fullname" . }}-secrets
                  key: dbUrl
```

---

## Step 288: Testing

```yaml
# templates/tests/test-connection.yaml
apiVersion: v1
kind: Pod
metadata:
  name: {{ include "my-app.fullname" . }}-test
  namespace: {{ .Release.Namespace }}
  annotations:
    "helm.sh/hook": test
    "helm.sh/hook-delete-policy": hook-succeeded
spec:
  restartPolicy: Never
  containers:
    - name: test
      image: curlimages/curl:8.4.0
      command:
        - sh
        - -c
        - |
          curl -sf http://{{ include "my-app.fullname" . }}:{{ .Values.service.port }}/healthz
          echo "Health check passed!"
          curl -sf http://{{ include "my-app.fullname" . }}:{{ .Values.service.port }}/readyz
          echo "Readiness check passed!"
```

```bash
# รัน tests
helm test my-app -n production

# Lint
helm lint ./my-app

# Dry run + debug
helm install my-app ./my-app \
  -f values-production.yaml \
  --dry-run --debug

# Template rendering
helm template my-app ./my-app \
  -f values-production.yaml
```

---

## Step 289: Production Values

```yaml
# values-production.yaml
replicaCount: 5

image:
  tag: "2.1.0"

ingress:
  enabled: true
  className: nginx
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
  hosts:
    - host: myapp.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: myapp-tls
      hosts:
        - myapp.example.com

resources:
  limits:
    cpu: 2000m
    memory: 2Gi
  requests:
    cpu: 500m
    memory: 512Mi

autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 20
  targetCPUUtilizationPercentage: 70
  targetMemoryUtilizationPercentage: 80

config:
  logLevel: warn
  database:
    host: postgres-primary.data.svc.cluster.local
    port: 5432
    name: mydb_prod

affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          topologyKey: kubernetes.io/hostname
          labelSelector:
            matchLabels:
              app.kubernetes.io/name: my-app
```

---

## Step 290: Workshop + OCI Registry

```bash
# สร้าง Helm chart ใหม่
helm create my-app

# Build dependencies
helm dependency update ./my-app

# Package
helm package ./my-app
# Output: my-app-1.2.3.tgz

# OCI Registry (Helm 3.8+)
helm push my-app-1.2.3.tgz oci://ghcr.io/myorg/helm-charts
helm install my-app oci://ghcr.io/myorg/helm-charts/my-app --version 1.2.3

# Install with atomic (rollback ถ้า fail)
helm install my-app ./my-app \
  --namespace production \
  --create-namespace \
  --values values-production.yaml \
  --atomic \
  --wait \
  --timeout 5m

# Upgrade
helm upgrade my-app ./my-app \
  --namespace production \
  --values values-production.yaml \
  --atomic \
  --wait \
  --timeout 5m

# ดู rendered resources
kubectl get all -n production -l app.kubernetes.io/instance=my-app
```

```yaml
# helmfile.yaml - declarative multi-chart management
repositories:
  - name: bitnami
    url: https://charts.bitnami.com/bitnami
  - name: ingress-nginx
    url: https://kubernetes.github.io/ingress-nginx

releases:
  - name: nginx-ingress
    chart: ingress-nginx/ingress-nginx
    namespace: ingress-nginx
    version: 4.8.3
    values:
      - values/ingress-nginx.yaml

  - name: cert-manager
    chart: jetstack/cert-manager
    namespace: cert-manager
    version: 1.13.2
    set:
      - name: installCRDs
        value: true

  - name: my-app
    chart: ./charts/my-app
    namespace: production
    version: 1.2.3
    values:
      - values/production.yaml
    needs:
      - ingress-nginx/nginx-ingress
```

---

## 📊 สรุป Part 30

| Feature | รายละเอียด |
|---------|----------|
| `helm install` | Deploy chart |
| `helm upgrade` | Update release |
| `helm rollback` | Revert to previous |
| `helm test` | Run test pods |
| `helm lint` | Validate chart |
| Hooks | Pre/post-install/upgrade/delete |
| OCI Registry | Modern chart distribution |
| Helmfile | Declarative multi-chart |

---

## 🔗 ต่อไป
- [Part 31: Kustomize](./part-31-kustomize.md)

---
*Part 30 | Steps 281-290 | ระดับสูง*
