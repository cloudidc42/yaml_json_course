# Part 54: Secrets Exposure & Protection
## Steps 511-520: Kubernetes Secrets Security

---

## 📖 บทนำ

> **⚠️ หมายเหตุ**: เนื้อหาด้าน security ในหลักสูตรนี้มีไว้เพื่อการศึกษาและการทดสอบระบบที่ได้รับอนุญาตเท่านั้น การนำไปใช้โจมตีระบบที่ไม่ได้รับอนุญาตเป็นสิ่งผิดกฎหมาย

Kubernetes Secrets มีหลายจุดอ่อนโดยธรรมชาติ - ถูกเก็บใน etcd เป็น base64 (ไม่ใช่ encryption), ถูก expose ผ่าน env vars, และ mount เป็น volumes

---

## Step 511: Secrets Exposure Vectors

```
Kubernetes Secrets Exposure Points:

1. etcd (unencrypted by default)
   - etcd data directory
   - etcd backup files
   - etcd snapshots in S3

2. Pod environment variables
   - Process environment (/proc/<pid>/environ)
   - Application logs (if logged)
   - Error messages (stack traces)

3. Mounted volumes
   - Container filesystem (/run/secrets/)
   - Accessible to other containers in same pod

4. API Server
   - RBAC: any SA with get/list secrets
   - Audit logs may contain secret values

5. CI/CD pipelines
   - Hardcoded in Dockerfiles
   - Git history
   - CI/CD env variables

6. Application-level
   - Logging secret values
   - Error reporting (Sentry, etc.)
   - Heap dumps
```

---

## Step 512: Encryption at Rest

```yaml
# EncryptionConfiguration
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
      - configmaps
    providers:
      - aesgcm:
          keys:
            - name: key1
              secret: c2VjcmV0aXMzMmJ5dGVzISEhISEhISEhISEhISE=
      - aescbc:
          keys:
            - name: key1
              secret: c2VjcmV0aXMzMmJ5dGVzISEhISEhISEhISEhISE=

---
# KMS provider (production recommended)
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
    providers:
      - kms:
          apiVersion: v2
          name: aws-kms
          endpoint: unix:///var/run/kmsplugin/socket.sock
          timeout: 3s
      - identity: {}

# Verify: etcdctl get /registry/secrets/default/my-secret | hexdump
# Should show 'k8s:enc:aesgcm' prefix
```

---

## Step 513: Secret Scanning

```yaml
# Secret scanning tools:
# gitleaks detect --source . --report-path report.json
# trufflehog git file://. --only-verified
# trivy fs . --scanners secret
# trivy image myapp:1.0 --scanners secret

# GitHub Actions secret scan:
# - uses: trufflesecurity/trufflehog@main
#   with:
#     base: main
#     head: HEAD
#     extra_args: --only-verified

# Pre-commit (.pre-commit-config.yaml):
# repos:
#   - repo: https://github.com/gitleaks/gitleaks
#     rev: v8.18.0
#     hooks:
#       - id: gitleaks

# Falco rule: detect secret reads
# - rule: Read sensitive Kubernetes secret
#   condition: >
#     ka.verb=get and
#     ka.target.resource=secrets and
#     ka.target.name in (prod-db-password, aws-credentials)
#   output: Sensitive secret read (user=%ka.user.name secret=%ka.target.name)
#   priority: WARNING
```

---

## Step 514: External Secrets Best Practices

```yaml
# ExternalSecret with template
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: app-credentials
  namespace: production
spec:
  refreshInterval: 15m
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: app-credentials
    creationPolicy: Owner
    deletionPolicy: Retain
    template:
      type: Opaque
      engineVersion: v2
      data:
        DATABASE_URL: "postgresql://{{ .username }}:{{ .password }}@{{ .host }}/{{ .dbname }}"
  data:
    - secretKey: username
      remoteRef:
        key: production/my-app/db
        property: username
    - secretKey: password
      remoteRef:
        key: production/my-app/db
        property: password
    - secretKey: host
      remoteRef:
        key: production/my-app/db
        property: host
    - secretKey: dbname
      remoteRef:
        key: production/my-app/db
        property: dbname
```

---

## Step 515: Vault Integration

```yaml
# Vault Agent Sidecar
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: production
spec:
  template:
    metadata:
      annotations:
        vault.hashicorp.com/agent-inject: "true"
        vault.hashicorp.com/role: "my-app"
        vault.hashicorp.com/agent-inject-secret-database: "secret/data/production/database"
        vault.hashicorp.com/agent-inject-template-database: |
          {{- with secret "secret/data/production/database" -}}
          DATABASE_URL=postgresql://{{ .Data.data.username }}:{{ .Data.data.password }}@{{ .Data.data.host }}/{{ .Data.data.dbname }}
          {{- end -}}
        vault.hashicorp.com/agent-inject-perms-database: "0400"
    spec:
      serviceAccountName: my-app

---
# Vault CSI Provider
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: vault-secrets
  namespace: production
spec:
  provider: vault
  parameters:
    vaultAddress: "https://vault.vault.svc:8200"
    vaultKubernetesMountPath: "kubernetes"
    roleName: "my-app"
    objects: |
      - objectName: "db-password"
        secretPath: "secret/data/production/database"
        secretKey: "password"
  secretObjects:
    - secretName: vault-secret
      type: Opaque
      data:
        - objectName: db-password
          key: password
```

---

## Step 516: Secret Rotation

```yaml
# Reloader - auto restart pods on secret change
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: production
  annotations:
    secret.reloader.stakater.com/reload: "db-secret,api-key-secret"
    configmap.reloader.stakater.com/reload: "app-config"

# Vault dynamic secrets:
# vault write database/roles/my-app \
#   db_name=my-postgres \
#   creation_statements="CREATE ROLE '{{name}}' WITH ENCRYPTED PASSWORD '{{password}}'
#     VALID UNTIL '{{expiration}}' IN ROLE readonly" \
#   default_ttl=1h
# -> New credentials generated per pod, old expire automatically
```

---

## Step 517: Sealed Secrets

```yaml
# SealedSecret CRD (safe to store in Git)
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: my-sealed-secret
  namespace: production
spec:
  encryptedData:
    api-key: AgBy3i4OJSWK+PiTySYZZA9rO43cGDEq...
    db-password: AgAKo04dPBREuXJEGSHikKaOHPNXGRtG...
  template:
    metadata:
      name: my-secret
      namespace: production
    type: Opaque

# Create sealed secret:
# echo -n "my-value" | kubectl create secret generic my-secret \
#   --dry-run=client --from-file=key=/dev/stdin -o yaml | \
#   kubeseal -o yaml > my-sealed-secret.yaml

# Backup master key (CRITICAL):
# kubectl get secret -n kube-system \
#   -l sealedsecrets.bitnami.com/sealed-secrets-key \
#   -o yaml > sealed-secrets-master-key.yaml
```

---

## Step 518: Secret Anti-patterns Detection

```yaml
# Kyverno: prevent hardcoded secrets in env vars
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: no-hardcoded-secrets
spec:
  validationFailureAction: Enforce
  rules:
    - name: check-env-vars
      match:
        any:
          - resources:
              kinds: [Pod]
      validate:
        message: "Use secretKeyRef for sensitive env vars, not hardcoded values"
        deny:
          conditions:
            any:
              - key: "{{ request.object.spec.containers[].env[?value != null].name | to_array(@) }}"
                operator: AnyIn
                value:
                  - "PASSWORD"
                  - "SECRET"
                  - "API_KEY"
                  - "TOKEN"
                  - "AWS_SECRET_ACCESS_KEY"
```

---

## Step 519: Network Protection

```yaml
# NetworkPolicy: restrict secret store access
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-vault-access
  namespace: production
spec:
  podSelector:
    matchLabels:
      needs-secrets: "true"
  policyTypes: [Egress]
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: vault
      ports:
        - port: 8200
          protocol: TCP
    - ports:
        - port: 53
          protocol: UDP
```

---

## Step 520: Workshop - Production Secrets

```yaml
# Complete secrets architecture

# ServiceAccount with IRSA
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-secrets-sa
  namespace: production
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789:role/app-secrets-role
automountServiceAccountToken: false

---
# Deployment: volume mount (NOT env var)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: production
  annotations:
    secret.reloader.stakater.com/reload: "app-db-credentials"
spec:
  template:
    spec:
      serviceAccountName: my-app
      automountServiceAccountToken: false
      securityContext:
        runAsNonRoot: true
        runAsUser: 10001
      volumes:
        - name: secrets
          secret:
            secretName: app-db-credentials
            defaultMode: 0400  # read-only by owner
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
          # App reads /run/secrets/DATABASE_URL
          # Not from environment variable
```

---

## 📊 สรุป Part 54

| Secret Risk | Protection |
|-------------|------------|
| etcd plaintext | EncryptionConfiguration + KMS |
| Git exposure | Secret scanning + Sealed Secrets |
| Env var logging | Volume mounts instead |
| Stale credentials | Auto-rotation + Reloader |
| Broad RBAC access | resourceNames restriction |
| Network transit | TLS + NetworkPolicy |

---

## 🔗 ต่อไป
- [Part 55: Kubernetes CVE Analysis](./part-55-k8s-privilege-escalation.md)

---
*Part 54 | Steps 511-520 | ระดับผู้เชี่ยวชาญ | Educational Use Only*
