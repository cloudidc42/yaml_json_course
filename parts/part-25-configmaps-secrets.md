# Part 25: ConfigMaps และ Secrets
## Steps 241-250: การจัดการ Configuration และ Sensitive Data

---

## 📖 บทนำ

ConfigMaps และ Secrets เป็นเครื่องมือหลักในการแยก configuration ออกจาก container images ตาม 12-factor app principles

---

## Step 241: ConfigMap Basics

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: production
data:
  app.name: "my-application"
  app.env: "production"
  app.port: "8080"
  database.host: "postgres.data.svc.cluster.local"
  
  app.yaml: |
    server:
      port: 8080
      timeout: 30s
    database:
      host: postgres.data.svc.cluster.local
      pool:
        min: 5
        max: 20
    logging:
      level: info
      format: json
  
  nginx.conf: |
    worker_processes auto;
    events { worker_connections 1024; }
    http {
      server {
        listen 80;
        location / { proxy_pass http://backend:8080; }
      }
    }
```

---

## Step 242: ConfigMap Usage Patterns

```yaml
# Pattern 1: Environment variables
apiVersion: v1
kind: Pod
spec:
  containers:
    - name: app
      image: myapp:1.0.0
      env:
        - name: DB_HOST
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: database.host
      envFrom:
        - configMapRef:
            name: app-config

---
# Pattern 2: Volume mount
apiVersion: v1
kind: Pod
spec:
  containers:
    - name: app
      image: myapp:1.0.0
      volumeMounts:
        - name: config
          mountPath: /app/config
        - name: nginx-config
          mountPath: /etc/nginx/nginx.conf
          subPath: nginx.conf
  volumes:
    - name: config
      configMap:
        name: app-config
        defaultMode: 0644
        items:
          - key: app.yaml
            path: app.yaml
    - name: nginx-config
      configMap:
        name: app-config
        items:
          - key: nginx.conf
            path: nginx.conf
```

---

## Step 243: Immutable ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: immutable-config
  namespace: production
data:
  version: "1.0.0"
  feature.flags: "flag1=true,flag2=false"
immutable: true  # ป้องกันการแก้ไข

---
# Reloader annotation
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  annotations:
    reloader.stakater.com/auto: "true"
spec:
  template:
    spec:
      containers:
        - name: app
          image: myapp:1.0.0
          envFrom:
            - configMapRef:
                name: app-config
```

---

## Step 244: Secrets Types

```yaml
# Opaque Secret
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
  namespace: production
type: Opaque
stringData:  # Kubernetes encode ให้เอง
  db-url: "postgresql://user:password@host:5432/db"
  api-key: "my-api-key-value"

---
# TLS Secret
apiVersion: v1
kind: Secret
metadata:
  name: tls-secret
  namespace: production
type: kubernetes.io/tls
data:
  tls.crt: <base64-encoded-cert>
  tls.key: <base64-encoded-key>

---
# Docker registry Secret
apiVersion: v1
kind: Secret
metadata:
  name: registry-credentials
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: <base64-encoded-docker-config>
```

---

## Step 245: Secrets Usage

```yaml
apiVersion: v1
kind: Pod
spec:
  imagePullSecrets:
    - name: registry-credentials
  containers:
    - name: app
      image: private-registry.example.com/myapp:1.0.0
      env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: db-password
              optional: false
      envFrom:
        - secretRef:
            name: app-secrets
      volumeMounts:
        - name: secrets
          mountPath: /app/secrets
          readOnly: true
  volumes:
    - name: secrets
      secret:
        secretName: app-secrets
        defaultMode: 0400
```

---

## Step 246: External Secrets Operator

```yaml
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
        serviceAccount:
          name: external-secrets
          namespace: external-secrets

---
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: app-secrets
  namespace: production
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: app-secrets
    creationPolicy: Owner
  data:
    - secretKey: db-password
      remoteRef:
        key: production/app/database
        property: password
  dataFrom:
    - extract:
        key: production/app/all-secrets
```

---

## Step 247: SOPS

```yaml
# .sops.yaml
creation_rules:
  - path_regex: .*/production/.*\.yaml$
    kms: 'arn:aws:kms:ap-southeast-1:123456789:key/mrk-xxx'
  - path_regex: .*/staging/.*\.yaml$
    age: 'age1xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx'

---
# Flux SOPS decryption
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: production-secrets
spec:
  decryption:
    provider: sops
    secretRef:
      name: sops-age
```

---

## Step 248: Sealed Secrets

```yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: app-secrets
  namespace: production
spec:
  encryptedData:
    db-password: AgBy...
    api-key: AgCx...
  template:
    metadata:
      name: app-secrets
      namespace: production
    type: Opaque

# สร้าง SealedSecret:
# kubectl create secret generic app-secrets \
#   --from-literal=db-password=my-password \
#   --dry-run=client -o yaml | \
#   kubeseal --cert pub-cert.pem --format yaml > sealed-secret.yaml
```

---

## Step 249: Encryption at Rest

```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
      - configmaps
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: <32-byte-base64-encoded-key>
      - kms:
          apiVersion: v2
          name: aws-kms
          endpoint: unix:///var/run/kmsplugin/socket.sock
          timeout: 3s
      - identity: {}

# kube-apiserver flag:
# --encryption-provider-config=/etc/kubernetes/enc/encryption.yaml

# Re-encrypt existing secrets:
# kubectl get secrets --all-namespaces -o json | kubectl replace -f -
```

---

## Step 250: Workshop - Secrets Audit

```python
#!/usr/bin/env python3
"""Secrets Management Audit Script"""
import kubernetes
from datetime import datetime

kubernetes.config.load_kube_config()
v1 = kubernetes.client.CoreV1Api()


def audit_secrets(namespaces: list[str]) -> dict:
    findings = {
        "old_secrets": [],
        "default_sa_tokens": [],
    }
    
    for ns in namespaces:
        try:
            secrets = v1.list_namespaced_secret(ns)
            for s in secrets.items:
                # ตรวจ old secrets (>90 วัน)
                if s.metadata.creation_timestamp:
                    age = datetime.now(s.metadata.creation_timestamp.tzinfo) \
                          - s.metadata.creation_timestamp
                    if age.days > 90:
                        findings["old_secrets"].append(
                            f"{ns}/{s.metadata.name} (type: {s.type})"
                        )
                
                # ตรวจ default SA tokens
                if s.metadata.name.startswith("default-token-"):
                    findings["default_sa_tokens"].append(f"{ns}/{s.metadata.name}")
        except kubernetes.client.exceptions.ApiException as e:
            print(f"Error accessing {ns}: {e}")
    
    return findings


if __name__ == "__main__":
    findings = audit_secrets(["default", "production", "staging"])
    
    print("=== Secrets Audit Report ===")
    if findings["old_secrets"]:
        print(f"\n⚠️  Old Secrets: {len(findings['old_secrets'])}")
        for s in findings["old_secrets"][:5]:
            print(f"   - {s}")
    
    if findings["default_sa_tokens"]:
        print(f"\n🔑 Default SA Tokens: {len(findings['default_sa_tokens'])}")
        for s in findings["default_sa_tokens"][:3]:
            print(f"   - {s}")
```

```yaml
# RBAC สำหรับ Secret access
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: secret-reader
  namespace: production
rules:
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get"]
    resourceNames: ["app-secrets"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: secret-reader-binding
  namespace: production
subjects:
  - kind: ServiceAccount
    name: my-app
    namespace: production
roleRef:
  kind: Role
  name: secret-reader
  apiGroup: rbac.authorization.k8s.io
```

---

## 📊 สรุป Part 25

| เครื่องมือ | ใช้เมื่อ |
|----------|---------|
| ConfigMap | Non-sensitive configuration |
| Secret (Opaque) | Passwords, API keys |
| Secret (TLS) | TLS certificates |
| External Secrets | Production, centralized management |
| SOPS | GitOps, encrypted secrets in git |
| Sealed Secrets | Simple GitOps pattern |
| Encryption at rest | Encrypt secrets in etcd |

---

## 🔗 ต่อไป
- [Part 26: Volumes และ Storage](./part-26-volumes-storage.md)

---
*Part 25 | Steps 241-250 | ระดับสูง*
