# Part 69: Advanced YAML Features
## Steps 651-660: YAML 1.2 Deep Dive

---

## Step 651: YAML Document Structure

```yaml
# YAML 1.2 specification features

# Multi-document YAML (separated by ---)
---
document: one
---
document: two
...  # Optional end marker

# YAML header
%YAML 1.2
---
name: example

# Tags (explicit type declarations)
integer: !!int 42
float: !!float 3.14
string: !!str "hello"
binary: !!binary |
  SGVsbG8gV29ybGQ=
null_value: !!null ~
boolean_true: !!bool true
sequence: !!seq [1, 2, 3]
mapping: !!map {key: value}
```

---

## Step 652: YAML Scalars

```yaml
# String styles
plain: just a string
single: 'can''t use double here'
double: "escape: \n\tA"
literal: |
  This preserves
  newlines exactly
folded: >
  This folds
  newlines into spaces

# Block chomping
strip: |-
  No trailing newline
clip: |
  Default: one trailing newline
keep: |+
  Keeps all trailing newlines


# Scalars: numbers
decimal: 42
octal: 0o52      # YAML 1.2 (was 052 in 1.1)
hex: 0x2A
float: 3.14
scientific: 1.5e3
infinity: .inf
not-a-number: .nan
negative-inf: -.inf

# Booleans (YAML 1.2: only true/false)
yes: true    # "yes" is a string in YAML 1.2!
no: false    # "no" is a string in YAML 1.2!
on: "on"     # Must quote to be a string
off: "off"   # Must quote to be a string

# Null
null1: ~
null2: null
null3:   # Empty value = null
```

---

## Step 653: YAML Collections

```yaml
# Sequences (arrays)
seq1: [1, 2, 3]  # Flow style
seq2:             # Block style
  - 1
  - 2
  - 3

# Nested sequences
matrix:
  - [1, 2, 3]
  - [4, 5, 6]
  - [7, 8, 9]

# Mappings (objects)
map1: {a: 1, b: 2}  # Flow style
map2:                # Block style
  a: 1
  b: 2

# Ordered keys
ordered:
  first: 1
  second: 2
  third: 3

# Nested complex structures
people:
  - name: Alice
    age: 30
    skills:
      - Python
      - Kubernetes
    address:
      city: Bangkok
      country: Thailand
  
  - name: Bob
    age: 25
    skills: [Go, Docker]
    address: {city: Chiang Mai, country: Thailand}
```

---

## Step 654: YAML Anchors and Aliases

```yaml
# Anchors (&) and Aliases (*) - reuse content
defaults: &defaults
  timeout: 30
  retries: 3
  log_level: info

production:
  <<: *defaults
  log_level: warn
  timeout: 60

staging:
  <<: *defaults
  log_level: debug

development:
  <<: *defaults
  log_level: debug
  retries: 1

---
# Real Kubernetes use case: environment variables
x-common-env: &common-env
  - name: TZ
    value: Asia/Bangkok
  - name: LOG_FORMAT
    value: json

x-security-context: &security-context
  runAsNonRoot: true
  runAsUser: 1000
  readOnlyRootFilesystem: true
  allowPrivilegeEscalation: false
  capabilities:
    drop: [ALL]

x-resources: &resources
  limits:
    cpu: 500m
    memory: 256Mi
  requests:
    cpu: 100m
    memory: 128Mi

---
# values.yaml (Helm) - anchors work here
frontend:
  image: frontend:1.0
  env: *common-env
  securityContext: *security-context
  resources: *resources

backend:
  image: backend:1.0
  env: *common-env
  securityContext: *security-context
  resources: *resources
```

---

## Step 655: YAML Merge Keys

```yaml
# Merge key (<<): merge multiple mappings
base: &base
  kind: Deployment
  apiVersion: apps/v1

production-labels: &prod-labels
  environment: production
  managed-by: helm

# Merge multiple mappings
deployment:
  <<: [*base, *prod-labels]
  metadata:
    name: my-app

# Result:
# kind: Deployment
# apiVersion: apps/v1
# environment: production
# managed-by: helm
# metadata:
#   name: my-app

---
# Helm values: DRY config
x-env-common: &env-common
  LOG_LEVEL: info
  TZ: Asia/Bangkok

x-env-db: &env-db
  DB_HOST: postgres
  DB_PORT: "5432"
  DB_NAME: myapp

app:
  env:
    <<: *env-common
    APP_PORT: "8080"
    DB_HOST: postgres
```

---

## Step 656: YAML Schema Validation with JSON Schema

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "type": "object",
  "required": ["apiVersion", "kind", "metadata", "spec"],
  "properties": {
    "apiVersion": {
      "type": "string",
      "pattern": "^[a-zA-Z]+(/[a-zA-Z0-9]+)?/v[0-9]+(alpha|beta)?[0-9]*$"
    },
    "kind": {
      "type": "string",
      "enum": ["Deployment", "Service", "ConfigMap", "Secret"]
    },
    "metadata": {
      "type": "object",
      "required": ["name"],
      "properties": {
        "name": {"type": "string", "maxLength": 253},
        "namespace": {"type": "string"},
        "labels": {
          "type": "object",
          "additionalProperties": {"type": "string"}
        }
      }
    }
  }
}
```

```yaml
# Validate tools:
# ajv validate -s schema.json -d manifest.yaml
# kubeval deployment.yaml
# kubeconform -strict deployment.yaml

# IDE hint via comment:
# yaml-language-server: $schema=https://json.schemastore.org/kustomization
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
  - service.yaml
```

---

## Step 657: Helm Templates - Advanced YAML

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "myapp.fullname" . }}
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
    {{- with .Values.extraLabels }}
    {{- toYaml . | nindent 4 }}
    {{- end }}
spec:
  replicas: {{ .Values.replicaCount | default 1 }}
  template:
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          
          {{- if .Values.env }}
          env:
            {{- range $key, $value := .Values.env }}
            - name: {{ $key }}
              value: {{ $value | quote }}
            {{- end }}
          {{- end }}
          
          {{- with .Values.resources }}
          resources:
            {{- toYaml . | nindent 12 }}
          {{- end }}
          
          {{- if .Values.persistence.enabled }}
          volumeMounts:
            - name: data
              mountPath: {{ .Values.persistence.mountPath }}
          {{- end }}
      
      {{- if .Values.persistence.enabled }}
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: {{ include "myapp.fullname" . }}-data
      {{- end }}
      
      {{- with .Values.nodeSelector }}
      nodeSelector:
        {{- toYaml . | nindent 8 }}
      {{- end }}
```

---

## Step 658: Kustomize - YAML Transformation

```yaml
# kustomize/base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
  - service.yaml
  - configmap.yaml
commonLabels:
  app.kubernetes.io/managed-by: kustomize

---
# kustomize/overlays/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
bases:
  - ../../base
namespace: production
namePrefix: prod-

patchesJson6902:
  - target:
      group: apps
      version: v1
      kind: Deployment
      name: myapp
    patch: |-
      - op: replace
        path: /spec/replicas
        value: 3
      - op: add
        path: /spec/template/spec/containers/0/resources
        value:
          limits:
            cpu: 1000m
            memory: 512Mi
          requests:
            cpu: 500m
            memory: 256Mi

images:
  - name: myapp
    newTag: v2.0.0

configMapGenerator:
  - name: app-config
    literals:
      - ENV=production
      - LOG_LEVEL=warn

secretGenerator:
  - name: app-secrets
    envs:
      - secrets.env
```

---

## Step 659: CUE Language for YAML/JSON

```cue
// CUE: type-safe configuration language
package k8s

#Deployment: {
  apiVersion: "apps/v1"
  kind:       "Deployment"
  metadata:   #Metadata
  spec:       #DeploymentSpec
}

#Metadata: {
  name:       string & =~"^[a-z0-9-]{1,63}$"
  namespace?: string
  labels?:    {[string]: string}
}

#Container: {
  name:  string
  image: string & =~"^.+:.+$"  // Must have tag
  resources:       #Resources
  securityContext: #SecurityContext
}

#SecurityContext: {
  runAsNonRoot:             true
  readOnlyRootFilesystem:   true
  allowPrivilegeEscalation: false
  capabilities: drop: ["ALL"]
}

#Resources: {
  limits: {
    cpu:    string & =~"^[0-9]+(m|[0-9])$"
    memory: string & =~"^[0-9]+(Mi|Gi)$"
  }
  requests: {
    cpu:    string
    memory: string
  }
}

// Concrete value (CUE validates against schema):
myDeployment: #Deployment & {
  metadata: name: "my-app"
  spec: replicas: 3
}
```

---

## Step 660: Workshop - YAML Best Practices

```yaml
# YAML Best Practices for Kubernetes

# 1. Use explicit types when ambiguous
port: "8080"      # String for env vars
version: "1.0"    # String (not float)
enabled: true     # Boolean (not "yes"/"no")

# 2. Quote strings with special characters
label: "value with: colon"
annotation: "text with # hash"
regex: "^[a-z]+$"

# 3. Use block style for long content
script: |
  #!/bin/bash
  echo "Hello"
  echo "World"

# 4. Document structure
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  labels:
    app.kubernetes.io/name: frontend
    app.kubernetes.io/version: "1.2.3"
    app.kubernetes.io/component: frontend
    app.kubernetes.io/part-of: myapp

# 5. Consistent indentation (2 spaces for K8s)

# 6. Validate before apply
# kubeval manifest.yaml
# kubeconform manifest.yaml
# kubectl apply --dry-run=server -f manifest.yaml

# 7. Use Helm or Kustomize for templating
# Never use sed/envsubst for variable substitution

# 8. Keep secrets out of YAML
# Use ExternalSecrets, Vault, or sealed-secrets
# Never commit secrets to git

---
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: yaml-quality
  namespace: monitoring
spec:
  groups:
    - name: yaml-quality
      rules:
        - alert: InvalidManifestDeployed
          expr: |
            kube_deployment_labels{label_validated="false"} > 0
          annotations:
            summary: "Deployment deployed without validation"
          labels:
            severity: warning
```

---

## 📊 สรุป Part 69

| Feature | Use Case | Example |
|---------|----------|---------|
| Anchors | DRY config | `&defaults`, `*defaults` |
| Merge keys | Combine mappings | `<<: *base` |
| Block scalars | Multi-line text | `\|` and `>` |
| JSON Schema | Validation | kubeval, ajv |
| Kustomize | Environment overlays | overlays/production |
| Helm templates | Parameterized YAML | `{{ .Values.x }}` |
| CUE | Type-safe config | `#Container` |

---

## 🔗 ต่อไป
- [Part 70: JSON Advanced Features](./part-70-advanced-json.md)

---
*Part 69 | Steps 651-660 | ระดับกลาง-ผู้เชี่ยวชาญ | Educational Content*
