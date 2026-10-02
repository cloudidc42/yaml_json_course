# Part 48: GitOps Advanced with ArgoCD
## Steps 461-470: Production GitOps Patterns

---

## 📖 บทนำ

GitOps ใช้ Git เป็น single source of truth สำหรับ infrastructure และ application configuration ArgoCD เป็น tool หลักที่ sync cluster state จาก Git repository โดยอัตโนมัติ

> **หมายเหตุ**: เนื้อหาด้าน security ในหลักสูตรนี้มีไว้เพื่อการศึกษาและการทดสอบระบบที่ได้รับอนุญาตเท่านั้น การนำไปใช้โจมตีระบบที่ไม่ได้รับอนุญาตเป็นสิ่งผิดกฎหมาย

---

## Step 461: ArgoCD Architecture

```
ArgoCD Architecture:
┌─────────────────────────────────────────────────────────────┐
│                    Git Repository                           │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  /apps/production/  /apps/staging/  /charts/          │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────┬───────────────────────────────────────────┘
                  │ Poll/webhook
┌─────────────────▼───────────────────────────────────────────┐
│                    ArgoCD                                   │
│                                                             │
│  ┌────────────────┐  ┌────────────────┐  ┌──────────────┐  │
│  │  API Server    │  │  Repo Server   │  │  App         │  │
│  │  (UI/CLI/API)  │  │  (Template)    │  │  Controller  │  │
│  └────────────────┘  └────────────────┘  └──────────────┘  │
│                                                   │          │
│                                         Compare & Sync      │
└───────────────────────────────────────────────────┼─────────┘
                                                    │
┌───────────────────────────────────────────────────▼─────────┐
│                 Kubernetes Cluster                           │
│  Desired State (Git) ←→ Actual State (Cluster)              │
└─────────────────────────────────────────────────────────────┘
```

---

## Step 462: ArgoCD Installation

```yaml
# Install ArgoCD
# kubectl create namespace argocd
# kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# HA installation สำหรับ production
# kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/ha/install.yaml

---
# ArgoCD values.yaml (Helm installation)
global:
  domain: argocd.example.com

configs:
  params:
    server.insecure: false
  cm:
    # OIDC integration
    oidc.config: |
      name: GitHub
      issuer: https://dex.example.com
      clientID: argocd
      clientSecret: $oidc.dex.clientSecret
      requestedScopes: ["openid", "profile", "email", "groups"]
    # Policy
    policy.default: role:readonly
    # Resource customizations
    resource.customizations.health.certmanager.io_Certificate: |
      hs = {}
      if obj.status ~= nil then
        if obj.status.conditions ~= nil then
          for i, condition in ipairs(obj.status.conditions) do
            if condition.type == "Ready" and condition.status == "False" then
              hs.status = "Degraded"
              hs.message = condition.message
              return hs
            end
            if condition.type == "Ready" and condition.status == "True" then
              hs.status = "Healthy"
              hs.message = condition.message
              return hs
            end
          end
        end
      end
      hs.status = "Progressing"
      hs.message = "Waiting for certificate"
      return hs

server:
  ingress:
    enabled: true
    ingressClassName: nginx
    hostname: argocd.example.com
    tls: true
    annotations:
      cert-manager.io/cluster-issuer: letsencrypt-prod
      nginx.ingress.kubernetes.io/ssl-redirect: "true"
      nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"

repoServer:
  replicas: 2
  resources:
    requests:
      cpu: 100m
      memory: 256Mi
    limits:
      cpu: 1
      memory: 1Gi

applicationSet:
  enabled: true
  replicas: 2
```

---

## Step 463: ApplicationSet

```yaml
# ApplicationSet - สร้าง Applications หลายอันพร้อมกัน

# 1. List generator - hardcoded list
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: cluster-apps
  namespace: argocd
spec:
  generators:
    - list:
        elements:
          - cluster: production
            url: https://prod-cluster.example.com
          - cluster: staging
            url: https://staging-cluster.example.com
  template:
    metadata:
      name: "{{cluster}}-apps"
    spec:
      project: default
      source:
        repoURL: https://github.com/myorg/helm-charts
        targetRevision: HEAD
        path: "apps/{{cluster}}"
      destination:
        server: "{{url}}"
        namespace: production

---
# 2. Git Directory generator
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: git-dir-apps
  namespace: argocd
spec:
  generators:
    - git:
        repoURL: https://github.com/myorg/apps
        revision: HEAD
        directories:
          - path: "apps/production/*"
  template:
    metadata:
      name: "{{path.basename}}"
    spec:
      project: production
      source:
        repoURL: https://github.com/myorg/apps
        targetRevision: HEAD
        path: "{{path}}"
      destination:
        server: https://kubernetes.default.svc
        namespace: "{{path.basename}}"
      syncPolicy:
        automated:
          prune: true
          selfHeal: true

---
# 3. Matrix generator (clusters x apps)
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: matrix-apps
  namespace: argocd
spec:
  generators:
    - matrix:
        generators:
          - clusters:
              selector:
                matchLabels:
                  environment: production
          - git:
              repoURL: https://github.com/myorg/apps
              revision: HEAD
              directories:
                - path: "apps/*"
  template:
    metadata:
      name: "{{name}}-{{path.basename}}"
    spec:
      project: production
      source:
        repoURL: https://github.com/myorg/apps
        targetRevision: HEAD
        path: "{{path}}"
      destination:
        server: "{{server}}"
        namespace: "{{path.basename}}"
```

---

## Step 464: Sync Strategies

```yaml
# Application ที่มี sync options ครบ
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io  # cascade delete
spec:
  project: production
  source:
    repoURL: https://github.com/myorg/helm-charts
    targetRevision: v1.2.3
    path: charts/my-app
    helm:
      releaseName: my-app
      valueFiles:
        - values-production.yaml
      values: |
        image:
          tag: "1.0.5"
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  
  syncPolicy:
    automated:
      prune: true       # ลบ resources ที่ไม่มีใน Git
      selfHeal: true    # auto-fix ถ้า cluster state drift
      allowEmpty: false # ห้าม sync ถ้า source ว่าง
    
    syncOptions:
      - Validate=true           # validate manifests ก่อน apply
      - CreateNamespace=false   # ไม่สร้าง namespace อัตโนมัติ
      - PrunePropagationPolicy=foreground
      - PruneLast=true          # delete ทีหลังสุด
      - ApplyOutOfSyncOnly=true # apply เฉพาะ out-of-sync resources
      - ServerSideApply=true    # ใช้ SSA
    
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
        - /spec/replicas  # ignore HPA-managed replicas
    - group: ""
      kind: ServiceAccount
      jsonPointers:
        - /secrets  # ignore auto-added secrets
```

---

## Step 465: ArgoCD Notifications

```yaml
# ArgoCD Notifications - แจ้งเตือนเมื่อ sync events
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-notifications-cm
  namespace: argocd
data:
  trigger.on-sync-failed: |
    - when: app.status.operationState.phase in ['Error', 'Failed']
      send: [app-sync-failed]
  
  trigger.on-sync-succeeded: |
    - when: app.status.operationState.phase in ['Succeeded']
      send: [app-sync-succeeded]
  
  trigger.on-health-degraded: |
    - when: app.status.health.status == 'Degraded'
      send: [app-health-degraded]
  
  template.app-sync-failed: |
    message: |
      Application {{.app.metadata.name}} sync FAILED
      Sync Phase: {{.app.status.operationState.phase}}
      Message: {{.app.status.operationState.message}}
      Time: {{.app.status.operationState.finishedAt}}
  
  template.app-sync-succeeded: |
    message: |
      Application {{.app.metadata.name}} synced successfully
      Revision: {{.app.status.operationState.syncResult.revision}}
  
  template.app-health-degraded: |
    message: |
      Application {{.app.metadata.name}} is Degraded
      Health: {{.app.status.health.status}}
  
  service.slack: |
    token: $slack-token
    channel: "#deployments"

---
# Subscribe application to notifications
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  annotations:
    notifications.argoproj.io/subscribe.on-sync-failed.slack: ""
    notifications.argoproj.io/subscribe.on-sync-succeeded.slack: ""
    notifications.argoproj.io/subscribe.on-health-degraded.slack: "#alerts"
```

---

## Step 466: Multi-cluster GitOps

```yaml
# Register external cluster
# argocd cluster add production-eks \
#   --name production \
#   --kubeconfig /path/to/kubeconfig

# ClusterSecret สำหรับ external cluster
apiVersion: v1
kind: Secret
metadata:
  name: production-cluster
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: cluster
type: Opaque
stringData:
  name: production
  server: https://prod-cluster.example.com
  config: |
    {
      "bearerToken": "<token>",
      "tlsClientConfig": {
        "caData": "<base64-ca>",
        "certData": "<base64-cert>",
        "keyData": "<base64-key>"
      }
    }

---
# ApplicationSet deploy ไปทุก cluster
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: global-infra
  namespace: argocd
spec:
  generators:
    - clusters:
        selector:
          matchLabels:
            environment: production
  template:
    metadata:
      name: "infra-{{name}}"
    spec:
      project: platform
      source:
        repoURL: https://github.com/myorg/platform
        targetRevision: HEAD
        path: infrastructure
      destination:
        server: "{{server}}"
        namespace: kube-system
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
```

---

## Step 467: ArgoCD RBAC

```yaml
# ArgoCD RBAC policy
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-rbac-cm
  namespace: argocd
data:
  policy.default: role:readonly
  policy.csv: |
    # Admin role
    p, role:admin, applications, *, */*, allow
    p, role:admin, clusters, *, *, allow
    p, role:admin, repositories, *, *, allow
    p, role:admin, projects, *, *, allow
    
    # Developer role
    p, role:developer, applications, get, */*, allow
    p, role:developer, applications, sync, production/*, allow
    p, role:developer, applications, create, staging/*, allow
    p, role:developer, applications, delete, staging/*, allow
    p, role:developer, logs, get, */*, allow
    
    # Readonly role (default)
    p, role:readonly, applications, get, */*, allow
    p, role:readonly, logs, get, */*, allow
    
    # Group bindings
    g, myorg:platform-team, role:admin
    g, myorg:developers, role:developer
  
  scopes: '[groups]'

---
# AppProject with fine-grained access
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: team-backend
  namespace: argocd
spec:
  sourceRepos:
    - https://github.com/myorg/backend-*
  destinations:
    - namespace: backend-*
      server: https://kubernetes.default.svc
  roles:
    - name: developer
      description: Backend developers
      policies:
        - p, proj:team-backend:developer, applications, get, team-backend/*, allow
        - p, proj:team-backend:developer, applications, sync, team-backend/*, allow
        - p, proj:team-backend:developer, applications, create, team-backend/*, allow
      groups:
        - myorg:backend-team
```

---

## Step 468: Argo Rollouts

```yaml
# Argo Rollouts - progressive delivery
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: my-app
  namespace: production
spec:
  replicas: 10
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
          image: myregistry.io/my-app:1.0
          ports:
            - containerPort: 8080
  
  strategy:
    canary:
      steps:
        - setWeight: 10
        - pause: {duration: 5m}
        - setWeight: 25
        - pause: {duration: 10m}
        - setWeight: 50
        - pause: {duration: 10m}
        - setWeight: 75
        - pause: {duration: 5m}
      
      analysis:
        templates:
          - templateName: error-rate
        startingStep: 2
        args:
          - name: service-name
            value: my-app
      
      trafficRouting:
        istio:
          virtualService:
            name: my-app-vsvc
          destinationRule:
            name: my-app-destrule
            canarySubsetName: canary
            stableSubsetName: stable

---
# AnalysisTemplate
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: error-rate
  namespace: production
spec:
  args:
    - name: service-name
  metrics:
    - name: error-rate
      interval: 1m
      successCondition: result[0] < 0.01
      failureLimit: 3
      provider:
        prometheus:
          address: http://prometheus:9090
          query: |
            sum(rate(http_requests_total{app="{{args.service-name}}",status=~"5.."}[5m]))
            / sum(rate(http_requests_total{app="{{args.service-name}}"}[5m]))
```

---

## Step 469: Image Updater

```yaml
# ArgoCD Image Updater - auto update image tags
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
  annotations:
    argocd-image-updater.argoproj.io/image-list: app=myregistry.io/my-app
    argocd-image-updater.argoproj.io/app.update-strategy: semver
    argocd-image-updater.argoproj.io/app.allow-tags: "~1.0"
    argocd-image-updater.argoproj.io/write-back-method: git
    argocd-image-updater.argoproj.io/write-back-target: "helmvalues:values-production.yaml"
    argocd-image-updater.argoproj.io/app.helm.image-name: image.repository
    argocd-image-updater.argoproj.io/app.helm.image-tag: image.tag
spec:
  project: production
  source:
    repoURL: https://github.com/myorg/helm-charts
    targetRevision: HEAD
    path: charts/my-app
    helm:
      valueFiles:
        - values-production.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: production
```

---

## Step 470: Workshop - Production GitOps Setup

```yaml
# Workshop: production GitOps architecture

# Repository structure:
# myorg/platform-config (GitOps repo)
# ├── apps/
# │   ├── production/
# │   │   ├── app-of-apps.yaml
# │   │   ├── my-app/
# │   │   │   ├── kustomization.yaml
# │   │   │   └── values.yaml
# │   └── staging/
# ├── infrastructure/
# │   ├── monitoring/
# │   ├── cert-manager/
# │   └── ingress/
# └── clusters/
#     ├── production/
#     └── staging/

# App-of-Apps pattern
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: production-apps
  namespace: argocd
spec:
  project: platform
  source:
    repoURL: https://github.com/myorg/platform-config
    targetRevision: HEAD
    path: apps/production
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true

---
# Promotion workflow:
# 1. PR ไปยัง staging branch
# 2. Auto-sync ไป staging cluster
# 3. Run integration tests
# 4. PR ไปยัง main branch (requires approval)
# 5. Auto-sync ไป production cluster

# Kargo Stage
apiVersion: kargo.akuity.io/v1alpha1
kind: Stage
metadata:
  name: production
  namespace: my-project
spec:
  subscriptions:
    stages:
      - name: staging
  promotionMechanisms:
    argoCDAppUpdates:
      - appName: my-app-production
        sourceUpdates:
          - repoURL: https://github.com/myorg/helm-charts
            helm:
              images:
                - image: myregistry.io/my-app
```

---

## 📊 สรุป Part 48

| Feature | ประโยชน์ |
|---------|----------|
| ApplicationSet | Deploy หลาย apps/clusters พร้อมกัน |
| Sync Policies | Auto-sync, prune, self-heal |
| Notifications | แจ้งเตือน Slack/PagerDuty |
| RBAC | จำกัดสิทธิ์ตาม team |
| Argo Rollouts | Canary/Blue-Green deployments |
| Image Updater | Auto update image versions |

---

## 🔗 ต่อไป
- [Part 49: Multi-cluster Management](./part-49-multi-cluster.md)

---
*Part 48 | Steps 461-470 | ระดับสูง*
