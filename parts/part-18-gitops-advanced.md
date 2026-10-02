# Part 18: GitOps Advanced
## Steps 171-180: GitOps ขั้นสูงด้วย Flux และ ArgoCD

---

## 📖 บทนำ

GitOps เป็น paradigm ที่ใช้ Git เป็น single source of truth สำหรับ infrastructure และ application configurations ทำให้ deployments มีความ reproducible, auditable, และ recoverable

### GitOps Principles
```
1. Declarative   - describe desired state ใน Git
2. Versioned     - ทุก change มี history ใน Git  
3. Automatic     - pull changes ไป apply อัตโนมัติ
4. Continuous    - reconcile lifelessly ตลอดเวลา

GitOps Flow:
Developer → Git Commit → CI Pipeline → Git Repo
                                           ↓
                                    GitOps Agent
                                    (Flux/ArgoCD)
                                           ↓
                                   K8s Cluster
                                   (Reconcile)
```

---

## Step 171: Flux v2 - ติดตั้งและ Bootstrap

```bash
# ติดตั้ง Flux CLI
curl -s https://fluxcd.io/install.sh | sudo bash

# Bootstrap Flux กับ GitHub
flux bootstrap github \
  --owner=myorg \
  --repository=fleet-infra \
  --branch=main \
  --path=./clusters/my-cluster \
  --personal
```

```yaml
# Flux HelmRepository - source สำหรับ Helm charts
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: HelmRepository
metadata:
  name: bitnami
  namespace: flux-system
spec:
  interval: 30m
  url: https://charts.bitnami.com/bitnami

---
# HelmRelease - deploy Helm chart ผ่าน Flux
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: redis
  namespace: data
spec:
  interval: 5m
  chart:
    spec:
      chart: redis
      version: "18.x"
      sourceRef:
        kind: HelmRepository
        name: bitnami
        namespace: flux-system
  values:
    auth:
      enabled: false
    master:
      resources:
        requests:
          cpu: 100m
          memory: 128Mi
        limits:
          cpu: 500m
          memory: 512Mi
  upgrade:
    remediation:
      retries: 3
  rollback:
    timeout: 5m
    cleanupOnFail: true
```

---

## Step 172: Flux Kustomization

```yaml
# GitRepository - source จาก Git repo
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: my-app
  namespace: flux-system
spec:
  interval: 1m
  url: https://github.com/myorg/k8s-manifests
  ref:
    branch: main
  secretRef:
    name: github-credentials

---
# Kustomization - apply Kustomize configs
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: my-app
  namespace: flux-system
spec:
  interval: 5m
  path: ./apps/my-app
  prune: true  # ลบ resources ที่ถูกเอาออกจาก Git
  sourceRef:
    kind: GitRepository
    name: my-app
  healthChecks:
    - apiVersion: apps/v1
      kind: Deployment
      name: my-app
      namespace: production
  timeout: 2m
  retryInterval: 30s
  decryption:
    provider: sops
    secretRef:
      name: sops-gpg

---
# Dependencies - รอ resources อื่น ready ก่อน
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: my-app-db
  namespace: flux-system
spec:
  interval: 5m
  path: ./apps/my-app/database
  prune: true
  sourceRef:
    kind: GitRepository
    name: my-app
  dependsOn:
    - name: infrastructure  # รอ infrastructure ready ก่อน
  healthChecks:
    - apiVersion: apps/v1
      kind: StatefulSet
      name: postgresql
      namespace: data
```

---

## Step 173: Flux Image Automation

```yaml
# ImageRepository - scan container registry
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImageRepository
metadata:
  name: my-app
  namespace: flux-system
spec:
  image: registry.io/myorg/my-app
  interval: 5m
  secretRef:
    name: registry-credentials

---
# ImagePolicy - กำหนด policy สำหรับ image update
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImagePolicy
metadata:
  name: my-app
  namespace: flux-system
spec:
  imageRepositoryRef:
    name: my-app
  policy:
    semver:
      range: ">=1.0.0 <2.0.0"

---
# ImageUpdateAutomation - auto-commit image updates
apiVersion: image.toolkit.fluxcd.io/v1beta1
kind: ImageUpdateAutomation
metadata:
  name: my-app
  namespace: flux-system
spec:
  interval: 30m
  sourceRef:
    kind: GitRepository
    name: my-app
  git:
    checkout:
      ref:
        branch: main
    commit:
      author:
        email: fluxcdbot@users.noreply.github.com
        name: fluxcdbot
      messageTemplate: |
        Auto-update images

        Updated:
        {{ range .Updated.Images -}}
        - {{.}}
        {{ end -}}
    push:
      branch: main
  update:
    path: ./apps/my-app
    strategy: Setters
```

```yaml
# ใน deployment.yaml - ใส่ marker สำหรับ image update
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  template:
    spec:
      containers:
        - name: app
          image: registry.io/myorg/my-app:1.2.3 # {"$imagepolicy": "flux-system:my-app"}
```

---

## Step 174: ArgoCD - Advanced Configuration

```yaml
# ArgoCD AppProject - จัดการ permissions
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: production
  namespace: argocd
spec:
  description: Production applications
  
  sourceRepos:
    - "https://github.com/myorg/*"
    - "https://charts.bitnami.com/bitnami"
  
  destinations:
    - namespace: production
      server: https://kubernetes.default.svc
    - namespace: data
      server: https://kubernetes.default.svc
  
  clusterResourceWhitelist:
    - group: ""
      kind: Namespace
  
  namespaceResourceBlacklist:
    - group: ""
      kind: ResourceQuota  # ห้ามแก้ ResourceQuota
  
  roles:
    - name: developer
      description: Developer access
      policies:
        - p, proj:production:developer, applications, get, production/*, allow
        - p, proj:production:developer, applications, sync, production/*, allow
      groups:
        - org:developers
    - name: ops
      description: Ops full access
      policies:
        - p, proj:production:ops, applications, *, production/*, allow
      groups:
        - org:ops

---
# ArgoCD Application แบบ advanced
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app-production
  namespace: argocd
  annotations:
    notifications.argoproj.io/subscribe.on-sync-succeeded.slack: "#deployments"
    notifications.argoproj.io/subscribe.on-health-degraded.slack: "#alerts"
spec:
  project: production
  source:
    repoURL: https://github.com/myorg/k8s-manifests
    targetRevision: main
    path: apps/my-app/overlays/production
    kustomize:
      images:
        - "myapp=registry.io/myorg/my-app:1.2.3"
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
      allowEmpty: false
    syncOptions:
      - Validate=true
      - CreateNamespace=true
      - PrunePropagationPolicy=foreground
      - PruneLast=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
  ignoreDifferences:
    - group: apps
      kind: Deployment
      jsonPointers:
        - /spec/replicas  # ละเว้น HPA-managed replicas
  revisionHistoryLimit: 10
```

---

## Step 175: Secrets Management ใน GitOps

```yaml
# SOPS - .sops.yaml configuration
creation_rules:
  - path_regex: .*/production/.*\.yaml
    age: age1...publickey...  # prod key
  - path_regex: .*/staging/.*\.yaml
    age: age1...stagingkey...

---
# External Secrets Operator
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: database-credentials
  namespace: production
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secretsmanager
    kind: ClusterSecretStore
  target:
    name: database-credentials
    creationPolicy: Owner
  data:
    - secretKey: username
      remoteRef:
        key: prod/database
        property: username
    - secretKey: password
      remoteRef:
        key: prod/database
        property: password

---
# ClusterSecretStore - AWS Secrets Manager
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: aws-secretsmanager
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-1
      auth:
        jwt:
          serviceAccountRef:
            name: external-secrets-sa
            namespace: external-secrets
```

---

## Step 176: Progressive Delivery

```yaml
# Argo Rollouts - Canary deployment
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: my-app
  namespace: production
spec:
  replicas: 10
  revisionHistoryLimit: 2
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: app
          image: myapp:1.2.3
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 500m
              memory: 512Mi
  strategy:
    canary:
      steps:
        - setWeight: 5   # 5% traffic ไป canary
        - pause:
            duration: 5m
        - setWeight: 25   # 25% traffic
        - pause:
            duration: 5m
        - setWeight: 50   # 50% traffic
        - pause:
            duration: 5m
        - setWeight: 100  # 100% traffic
      analysis:
        successCondition: result[0] >= 0.95
        templates:
          - templateName: success-rate
        startingStep: 2

---
# AnalysisTemplate
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: success-rate
  namespace: production
spec:
  args:
    - name: service-name
  metrics:
    - name: success-rate
      interval: 2m
      successCondition: result[0] >= 0.95
      failureLimit: 3
      provider:
        prometheus:
          address: http://prometheus:9090
          query: |
            sum(rate(http_requests_total{
              service="{{ args.service-name }}",
              status!~"5.."
            }[2m])) /
            sum(rate(http_requests_total{
              service="{{ args.service-name }}"
            }[2m]))
```

---

## Step 177: Flux Notifications

```yaml
# Provider - Slack notification
apiVersion: notification.toolkit.fluxcd.io/v1beta3
kind: Provider
metadata:
  name: slack
  namespace: flux-system
spec:
  type: slack
  channel: "#deployments"
  secretRef:
    name: slack-token

---
# Alert - กำหนดว่าจะ notify เมื่อไหน
apiVersion: notification.toolkit.fluxcd.io/v1beta3
kind: Alert
metadata:
  name: deployment-alert
  namespace: flux-system
spec:
  providerRef:
    name: slack
  eventSeverity: info
  eventSources:
    - kind: Kustomization
      name: "*"
    - kind: HelmRelease
      name: "*"
  inclusionList:
    - ".*succeeded.*"
    - ".*failed.*"
```

---

## Step 178: GitOps Repository Structure

```
fleet-infra/               # GitOps repo
├── clusters/
│   ├── production/
│   │   ├── flux-system/   # Flux components
│   │   ├── infrastructure.yaml
│   │   └── apps.yaml
│   └── staging/
│       ├── flux-system/
│       ├── infrastructure.yaml
│       └── apps.yaml
├── infrastructure/
│   ├── base/
│   │   ├── cert-manager/
│   │   ├── ingress-nginx/
│   │   └── monitoring/
│   └── overlays/
│       ├── production/
│       └── staging/
└── apps/
    ├── base/
    │   ├── my-app/
    │   │   ├── kustomization.yaml
    │   │   ├── deployment.yaml
    │   │   └── service.yaml
    │   └── another-app/
    └── overlays/
        ├── production/
        │   └── my-app/
        │       ├── kustomization.yaml
        │       └── hpa.yaml
        └── staging/
            └── my-app/
```

```yaml
# clusters/production/apps.yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: apps
  namespace: flux-system
spec:
  interval: 5m
  dependsOn:
    - name: infrastructure
  sourceRef:
    kind: GitRepository
    name: fleet-infra
  path: ./apps/overlays/production
  prune: true
  wait: true
  timeout: 10m
```

---

## Step 179: GitOps Security Best Practices

```yaml
# 1. ใช้ specific git commit SHA แทน branch
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: my-app
spec:
  ref:
    commit: abc123def456...  # pinned SHA

---
# 2. Network Policy สำหรับ Flux
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: flux-system-default
  namespace: flux-system
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
  egress:
    - ports:
        - port: 443
        - port: 53
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: flux-system

---
# 3. Restrict ArgoCD permissions
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-rbac-cm
  namespace: argocd
data:
  policy.csv: |
    p, role:readonly, applications, get, */*, allow
    p, role:developer, applications, get, */*, allow
    p, role:developer, applications, sync, myteam/*, allow
    g, org:developers, role:developer
    g, org:ops, role:admin
  policy.default: role:readonly
  scopes: '[groups, email]'
```

---

## Step 180: Workshop - Full GitOps Pipeline

```yaml
# GitHub Actions CI → update image tag
# .github/workflows/update-image.yaml
name: Update Image Tag
on:
  push:
    branches: [main]

jobs:
  update-tag:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          repository: myorg/fleet-infra
          token: ${{ secrets.GIT_TOKEN }}
      
      - name: Update image tag
        run: |
          cd apps/overlays/production/my-app
          kustomize edit set image myapp=registry.io/myorg/my-app:${{ github.sha }}
      
      - name: Commit and push
        run: |
          git config user.email "ci@myorg.com"
          git config user.name "CI Bot"
          git add .
          git commit -m "Update my-app to ${{ github.sha }}"
          git push
```

---

## 📊 สรุป Part 18

| หัวข้อ | เครื่องมือ |
|--------|----------|
| GitOps Bootstrap | Flux bootstrap, ArgoCD |
| Flux Sources | GitRepository, HelmRepository |
| Flux Sync | Kustomization, HelmRelease |
| Image Automation | ImageRepository, ImagePolicy |
| Secrets | SOPS, Sealed Secrets, External Secrets |
| Progressive Delivery | Argo Rollouts, canary |
| Notifications | Slack, webhook |
| Security | commit SHA pinning, RBAC |

---

## 🔗 ต่อไป
- [Part 19: Kubernetes Security Policies](./part-19-k8s-security-policies.md)

---
*Part 18 | Steps 171-180 | ระดับสูง*
