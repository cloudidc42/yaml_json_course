# Part 45: Secrets Management
## Steps 431-440: Managing Sensitive Configuration

---

## 📖 บทนำ

การจัดการ secrets อย่างถูกต้องเป็นพื้นฐานของ security ใน Kubernetes ครอบคลุมตั้งแต่ Kubernetes Secrets, External Secrets Operator, HashiCorp Vault ไปจนถึง cloud-native secret stores

> **หมายเหตุ**: เนื้อหาด้าน security ในหลักสูตรนี้มีไว้เพื่อการศึกษาและการทดสอบระบบที่ได้รับอนุญาตเท่านั้น การนำไปใช้โจมตีระบบที่ไม่ได้รับอนุญาตเป็นสิ่งผิดกฎหมาย

---

## Step 431: Kubernetes Secrets Types

```yaml
# 1. Opaque (generic)
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
  namespace: production
type: Opaque
stringData:
  username: dbuser
  password: "s3cur3P@ssw0rd"
  connection-string: "postgresql://dbuser:s3cur3P@ssw0rd@postgres:5432/mydb"

---
# 2. TLS Secret
apiVersion: v1
kind: Secret
metadata:
  name: tls-cert
type: kubernetes.io/tls
data:
  tls.crt: <base64-cert>
  tls.key: <base64-key>

---
# 3. Docker Registry
apiVersion: v1
kind: Secret
metadata:
  name: registry-credentials
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: <base64-docker-config>

---
# 4. Service Account Token
apiVersion: v1
kind: Secret
metadata:
  name: sa-token
  annotations:
    kubernetes.io/service-account.name: my-sa
type: kubernetes.io/service-account-token

---
# 5. SSH Private Key
apiVersion: v1
kind: Secret
metadata:
  name: ssh-key
type: kubernetes.io/ssh-auth
data:
  ssh-privatekey: <base64-key>
```

---

## Step 432: Secret Usage Patterns

```yaml
# 1. Environment variable
apiVersion: v1
kind: Pod
spec:
  containers:
    - name: app
      env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: password
      envFrom:
        - secretRef:
            name: app-secrets

---
# 2. Volume mount (preferred)
apiVersion: v1
kind: Pod
spec:
  volumes:
    - name: secrets
      secret:
        secretName: db-credentials
        defaultMode: 0400
  containers:
    - name: app
      volumeMounts:
        - name: secrets
          mountPath: /run/secrets
          readOnly: true

---
# 3. Projected volume
apiVersion: v1
kind: Pod
spec:
  volumes:
    - name: combined
      projected:
        sources:
          - secret:
              name: db-credentials
              items:
                - key: password
                  path: db-password
                  mode: 0400
          - configMap:
              name: app-config
          - serviceAccountToken:
              path: token
              expirationSeconds: 3600
```

---

## Step 433: External Secrets Operator

```yaml
# SecretStore - AWS Secrets Manager
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: aws-secrets-manager
spec:
  provider:
    aws:
      service: SecretsManager
      region: ap-southeast-1
      auth:
        jwt:
          serviceAccountRef:
            name: external-secrets-sa
            namespace: external-secrets

---
# ExternalSecret
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
  namespace: production
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: db-credentials
    creationPolicy: Owner
    template:
      type: Opaque
      data:
        connection-string: "{{ .username }}:{{ .password }}@postgres:5432/mydb"
  data:
    - secretKey: username
      remoteRef:
        key: production/db
        property: username
    - secretKey: password
      remoteRef:
        key: production/db
        property: password
  dataFrom:
    - extract:
        key: production/app-secrets
```

---

## Step 434: HashiCorp Vault

```yaml
# Vault Agent Sidecar
apiVersion: v1
kind: Pod
metadata:
  name: vault-app
  annotations:
    vault.hashicorp.com/agent-inject: "true"
    vault.hashicorp.com/role: "my-app"
    vault.hashicorp.com/agent-inject-secret-db: "secret/data/production/db"
    vault.hashicorp.com/agent-inject-template-db: |
      {{- with secret "secret/data/production/db" -}}
      export DB_PASSWORD="{{ .Data.data.password }}"
      {{- end }}
spec:
  serviceAccountName: my-app-sa
  containers:
    - name: app
      command: ["/bin/sh", "-c", "source /vault/secrets/db && ./app"]

---
# Vault CSI Provider
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: vault-db-creds
  namespace: production
spec:
  provider: vault
  parameters:
    vaultAddress: "https://vault.example.com"
    roleName: "my-app"
    objects: |
      - objectName: "db-password"
        secretPath: "secret/data/production/db"
        secretKey: "password"
  secretObjects:
    - secretName: db-credentials
      type: Opaque
      data:
        - objectName: db-password
          key: password
```

---

## Step 435: IRSA - Cloud Credentials

```yaml
# IRSA (IAM Role for Service Accounts)
# Pod เข้าถึง AWS resources โดยไม่ใช้ static credentials

apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app-sa
  namespace: production
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789:role/my-app-role

---
# ExternalSecret ใช้ IRSA
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: app-secrets
  namespace: production
spec:
  refreshInterval: 30m
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: app-secrets
  dataFrom:
    - extract:
        key: production/my-app
```

---

## Step 436: Secret Rotation

```yaml
# Reloader - restart pods เมื่อ secret เปลี่ยน
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  annotations:
    secret.reloader.stakater.com/reload: "db-credentials,app-secrets"
spec:
  template:
    spec:
      containers:
        - name: app
          envFrom:
            - secretRef:
                name: db-credentials

---
# ExternalSecret ที่ refresh บ่อยๆ
# refreshInterval: 5m  <- ตรวจสอบทุก 5 นาที

# Vault Dynamic Secrets (สร้าง DB credentials ชั่วคราว)
# vault write database/roles/my-app-role \
#   db_name=postgresql \
#   creation_statements="CREATE ROLE '{{name}}' WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'" \
#   default_ttl="1h" \
#   max_ttl="24h"
```

---

## Step 437: Secret Anti-patterns

```yaml
# Anti-patterns ที่ต้องหลีกเลี่ยง:

# 1. Hardcoded ใน image
# ENV DB_PASSWORD=mysecret  <- ห้ามทำ!

# 2. Plaintext ใน ConfigMap
# db-password: mysecret  <- ห้ามทำ!

# 3. Secret ใน git
# ใช้ .gitignore:
# *.secret
# .env
# secrets/
# **/secret.yaml

# 4. Log secrets
# console.log(process.env.DB_PASSWORD)  <- ห้ามทำ!

---
# RBAC ปิดกั้น secret access
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: app-role
  namespace: production
rules:
  - apiGroups: [""]
    resources: ["pods", "services", "configmaps"]
    verbs: ["get", "list", "watch"]
  # ไม่มี secrets ใน rules = ไม่มีสิทธิ์ access
```

---

## Step 438: Secret Scanning

```yaml
# Gitleaks สำหรับ scan secrets ใน git
# gitleaks detect --source=.
# gitleaks protect --staged  (pre-commit hook)

# Trivy secret scanning
# trivy fs --security-checks secret .
# trivy image --security-checks secret myapp:1.0

# GitHub Actions CI scan
# name: Secret Scan
# on: [push, pull_request]
# jobs:
#   scan:
#     runs-on: ubuntu-latest
#     steps:
#       - uses: actions/checkout@v4
#       - name: TruffleHog OSS
#         uses: trufflesecurity/trufflehog@v3
#         with:
#           path: ./
#           base: main

---
# Falco rule สำหรับ detect secret access
# - rule: Read secret from volume
#   condition: >
#     open_read and container and
#     fd.name startswith /run/secrets and
#     not proc.name in (node, python, java, app)
#   output: "Unexpected process reading secret: proc=%proc.name file=%fd.name"
#   priority: WARNING
```

---

## Step 439: Sealed Secrets

```yaml
# Sealed Secrets - เข้ารหัส secrets ก่อน commit ลง git

# Install: kubectl apply -f sealed-secrets-controller.yaml
# Fetch cert: kubeseal --fetch-cert > pub-cert.pem
# Seal: kubectl create secret generic db-credentials \
#   --from-literal=password=mysecret \
#   --dry-run=client -o yaml | \
#   kubeseal --cert pub-cert.pem -o yaml > sealed-secret.yaml

apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: db-credentials
  namespace: production
spec:
  encryptedData:
    password: AgBy3i4OJSWK+PiTySYZZA9rO43cGDEq...  # encrypted
  template:
    metadata:
      name: db-credentials
      namespace: production
    type: Opaque

# Controller decrypt และสร้าง Secret จริงใน cluster
# Backup master key:
# kubectl get secret -n kube-system sealed-secrets-key -o yaml > sealed-secrets-key.yaml
```

---

## Step 440: Workshop - Complete Secrets Architecture

```yaml
# Production secrets architecture

# 1. Namespace
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted

---
# 2. ServiceAccount พร้อม IRSA
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app
  namespace: production
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789:role/my-app-role
automountServiceAccountToken: false

---
# 3. ExternalSecret
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: my-app-secrets
  namespace: production
spec:
  refreshInterval: 15m
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: my-app-secrets
    creationPolicy: Owner
  dataFrom:
    - extract:
        key: production/my-app

---
# 4. Deployment ที่ใช้ secrets อย่างปลอดภัย
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: production
  annotations:
    secret.reloader.stakater.com/reload: "my-app-secrets"
spec:
  template:
    spec:
      serviceAccountName: my-app
      automountServiceAccountToken: false
      securityContext:
        runAsNonRoot: true
        runAsUser: 10001
        seccompProfile:
          type: RuntimeDefault
      volumes:
        - name: secrets
          secret:
            secretName: my-app-secrets
            defaultMode: 0400
      containers:
        - name: app
          image: myregistry.io/my-app:1.0
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop: [ALL]
          volumeMounts:
            - name: secrets
              mountPath: /run/secrets
              readOnly: true
```

---

## 📊 สรุป Part 45

| Solution | Use Case |
|----------|----------|
| Kubernetes Secrets + Encryption | Simple secrets + encryption at rest |
| External Secrets Operator | Sync จาก AWS/GCP/Vault |
| HashiCorp Vault | Dynamic credentials, PKI |
| Sealed Secrets | GitOps (secrets ใน git) |
| IRSA/Workload Identity | Cloud credentials ไม่ต้องใช้ static keys |

---

## 🔗 ต่อไป
- [Part 46: Network Security](./part-46-network-security.md)

---
*Part 45 | Steps 431-440 | ระดับสูง*
