# Part 101: GitOps Advanced Patterns
## Steps 971-980: Flux v2, ArgoCD Patterns, Progressive Delivery, Secrets

---

## Step 971: GitOps Principles

```
GitOps Core Principles (OpenGitOps):

1. Declarative:
   - All desired system state stored as declarations
   - Kubernetes YAML, Helm values, Kustomize overlays
   - No imperative scripts in the main flow

2. Versioned and Immutable:
   - Git as single source of truth
   - All changes go through PRs
   - History: git log shows every change + who + why
   - Rollback = revert commit

3. Pulled Automatically:
   - Agent (ArgoCD/Flux) pulls from Git, not pushed to
   - No inbound ports to cluster required
   - Works behind firewalls

4. Continuously Reconciled:
   - Agent constantly compares desired vs actual state
   - Auto-heals drift: someone kubectl applies? Reverted.
   - No "snowflake" clusters

GitOps Workflow:
  Developer → PR → Code Review → Merge to main
  → ArgoCD/Flux detects change
  → Pulls new manifests
  → Applies to cluster
  → Reconciliation loop confirms success

Multi-Environment GitOps:
  Option 1: Branch per environment
    main → dev
    staging → staging
    production → prod
    Con: branch divergence, merge conflicts
  
  Option 2: Directory per environment (recommended)
    environments/
      dev/kustomization.yaml
      staging/kustomization.yaml
      production/kustomization.yaml
    Pros: single branch, clear history
  
  Option 3: Helm values per environment
    values-dev.yaml
    values-staging.yaml
    values-production.yaml
```

---

## Step 972: Flux v2 Bootstrap

```yaml
# Flux: bootstrap into cluster
# flux bootstrap github \
#   --owner=myorg \
#   --repository=fleet-infra \
#   --branch=main \
#   --path=clusters/production \
#   --personal

# Flux: reconcile all sources
# flux reconcile source git flux-system

# GitRepository: source of truth
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: fleet-infra
  namespace: flux-system
spec:
  interval: 1m
  url: https://github.com/myorg/fleet-infra
  ref:
    branch: main
  secretRef:
    name: flux-github-token

---
# Kustomization: apply manifests from repo
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: infrastructure
  namespace: flux-system
spec:
  interval: 5m
  path: ./infrastructure
  prune: true
  sourceRef:
    kind: GitRepository
    name: fleet-infra
  healthChecks:
    - apiVersion: apps/v1
      kind: Deployment
      name: ingress-nginx
      namespace: ingress-nginx
  postBuild:
    substitute:
      CLUSTER_NAME: production-us-east
      CLUSTER_REGION: us-east-1

---
# HelmRelease: manage Helm charts via Flux
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: kube-prometheus-stack
  namespace: monitoring
spec:
  interval: 1h
  chart:
    spec:
      chart: kube-prometheus-stack
      version: ">=55.0.0 <60.0.0"
      sourceRef:
        kind: HelmRepository
        name: prometheus-community
        namespace: flux-system
  values:
    prometheus:
      prometheusSpec:
        retention: 30d
  upgrade:
    remediation:
      retries: 3
  rollback:
    timeout: 5m
```

---

## Step 973: Flux Image Automation

```yaml
# ImageRepository: watch container registry
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImageRepository
metadata:
  name: payment-api
  namespace: flux-system
spec:
  image: ghcr.io/myorg/payment-api
  interval: 5m
  secretRef:
    name: ghcr-credentials

---
# ImagePolicy: select image version
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImagePolicy
metadata:
  name: payment-api
  namespace: flux-system
spec:
  imageRepositoryRef:
    name: payment-api
  filterTags:
    pattern: '^v(?P<version>[0-9]+\.[0-9]+\.[0-9]+)$'
    extract: '$version'
  policy:
    semver:
      range: ">=1.0.0 <2.0.0"

---
# ImageUpdateAutomation: auto-commit image updates
apiVersion: image.toolkit.fluxcd.io/v1beta1
kind: ImageUpdateAutomation
metadata:
  name: image-updater
  namespace: flux-system
spec:
  interval: 5m
  sourceRef:
    kind: GitRepository
    name: fleet-infra
  git:
    checkout:
      ref:
        branch: main
    commit:
      author:
        name: Flux Image Updater
        email: fluxbot@mycompany.com
      messageTemplate: |
        Auto-update images from Flux
        
        Updated: {{range .Updated.Images -}}
        - {{.}}
        {{end}}
    push:
      branch: main
  update:
    strategy: Setters

---
# Deployment: with imagepolicy setter comment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-api
  namespace: production
spec:
  template:
    spec:
      containers:
        - name: api
          image: ghcr.io/myorg/payment-api:v1.2.3 # {"$imagepolicy": "flux-system:payment-api"}
```

---

## Step 974: Argo Rollouts

```yaml
# Argo Rollouts: progressive delivery
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: payment-api
  namespace: production
spec:
  replicas: 10
  selector:
    matchLabels:
      app: payment-api
  template:
    metadata:
      labels:
        app: payment-api
    spec:
      containers:
        - name: api
          image: ghcr.io/myorg/payment-api:v2.0.0
  strategy:
    canary:
      analysis:
        templates:
          - templateName: payment-api-analysis
        startingStep: 2
      canaryService: payment-api-canary
      stableService: payment-api-stable
      trafficRouting:
        istio:
          virtualService:
            name: payment-api-vsvc
            routes:
              - primary
      steps:
        - setWeight: 5
        - pause: {duration: 10m}
        - setWeight: 20
        - pause: {duration: 10m}
        - setWeight: 50
        - pause: {duration: 5m}
        - setWeight: 80
        - pause: {duration: 5m}

---
# AnalysisTemplate: auto gate based on metrics
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: payment-api-analysis
  namespace: production
spec:
  metrics:
    - name: error-rate
      interval: 1m
      failureLimit: 3
      successCondition: result[0] < 0.05
      provider:
        prometheus:
          address: http://prometheus-operated.monitoring:9090
          query: |
            sum(rate(istio_requests_total{destination_service_name="payment-api",response_code=~"5.."}[5m]))
            /
            sum(rate(istio_requests_total{destination_service_name="payment-api"}[5m]))
    
    - name: p99-latency
      interval: 1m
      failureLimit: 3
      successCondition: result[0] < 1.0
      provider:
        prometheus:
          address: http://prometheus-operated.monitoring:9090
          query: |
            histogram_quantile(0.99,
              rate(istio_request_duration_milliseconds_bucket{destination_service_name="payment-api"}[5m])
            ) / 1000
```

---

## Step 975: ArgoCD ApplicationSet Matrix

```yaml
# ApplicationSet: matrix generator (env x cluster)
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: myapp-matrix
  namespace: argocd
spec:
  generators:
    - matrix:
        generators:
          - list:
              elements:
                - app: payment-api
                  chart: payment-api
                  version: 1.2.3
                - app: order-service
                  chart: order-service
                  version: 2.0.1
          - clusters:
              selector:
                matchLabels:
                  environment: production
  template:
    metadata:
      name: "{{app}}-{{name}}"
    spec:
      project: production
      source:
        repoURL: https://charts.mycompany.com
        chart: "{{chart}}"
        targetRevision: "{{version}}"
        helm:
          valueFiles:
            - "values-{{metadata.labels.region}}.yaml"
      destination:
        server: "{{server}}"
        namespace: "{{app}}"
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
          - PrunePropagationPolicy=foreground
```

---

## Step 976: GitOps with Sealed Secrets

```yaml
# Sealed Secrets: encrypt secrets for GitOps
# kubeseal --fetch-cert > pub-cert.pem
# kubeseal --cert pub-cert.pem \
#   --namespace production \
#   --name stripe-credentials \
#   < secret.yaml > sealed-secret.yaml

apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: stripe-credentials
  namespace: production
spec:
  encryptedData:
    STRIPE_API_KEY: AgBx2K...  # encrypted, safe to commit
    WEBHOOK_SECRET: AgCy3L...

---
# External Secrets Operator (ESO)
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: stripe-credentials
  namespace: production
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-backend
    kind: ClusterSecretStore
  target:
    name: stripe-credentials
    creationPolicy: Owner
  data:
    - secretKey: STRIPE_API_KEY
      remoteRef:
        key: production/payment-api
        property: stripe_api_key
    - secretKey: WEBHOOK_SECRET
      remoteRef:
        key: production/payment-api
        property: webhook_secret
```

---

## Step 977: Drift Detection

```yaml
# ArgoCD: drift detection and auto-heal
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: payment-api
  namespace: argocd
spec:
  project: production
  source:
    repoURL: https://github.com/myorg/gitops-repo
    targetRevision: main
    path: kubernetes/overlays/production
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
      allowEmpty: false
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
    syncOptions:
      - Validate=true
      - CreateNamespace=true
      - PrunePropagationPolicy=foreground
      - ApplyOutOfSyncOnly=true
  revisionHistoryLimit: 10
  ignoreDifferences:
    - group: apps
      kind: Deployment
      jsonPointers:
        - /spec/replicas
```

---

## Step 978: GitOps Promotion Workflow

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: gitops-promotion-workflow
  namespace: kube-system
data:
  workflow.yaml: |
    name: Promote to Staging
    
    on:
      workflow_dispatch:
        inputs:
          image_tag:
            description: 'Image tag to promote'
            required: true
    
    jobs:
      promote:
        runs-on: ubuntu-latest
        steps:
          - uses: actions/checkout@v4
          
          - name: Update staging image
            run: |
              cd environments/staging
              kustomize edit set image \
                payment-api=ghcr.io/myorg/payment-api:$IMAGE_TAG
          
          - name: Commit and push
            run: |
              git config user.name 'GitHub Actions'
              git config user.email 'actions@github.com'
              git add .
              git commit -m 'chore: promote payment-api to staging'
              git push
```

---

## Step 979: Flux Notification

```yaml
# Flux: send alerts to Slack on failures
apiVersion: notification.toolkit.fluxcd.io/v1beta3
kind: Provider
metadata:
  name: slack-platform
  namespace: flux-system
spec:
  type: slack
  channel: platform-alerts
  secretRef:
    name: slack-webhook-url

---
apiVersion: notification.toolkit.fluxcd.io/v1beta3
kind: Alert
metadata:
  name: on-error
  namespace: flux-system
spec:
  summary: "Flux reconciliation failed in production"
  providerRef:
    name: slack-platform
  eventSeverity: error
  eventSources:
    - kind: Kustomization
      namespace: flux-system
      name: "*"
    - kind: HelmRelease
      namespace: "*"
      name: "*"

---
apiVersion: notification.toolkit.fluxcd.io/v1beta3
kind: Alert
metadata:
  name: on-info
  namespace: flux-system
spec:
  summary: "Flux deployed new image"
  providerRef:
    name: slack-platform
  eventSeverity: info
  eventSources:
    - kind: ImageUpdateAutomation
      namespace: flux-system
      name: "*"
```

---

## Step 980: Workshop - GitOps Summary

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: gitops-alerts
  namespace: monitoring
spec:
  groups:
    - name: gitops
      rules:
        - alert: FluxKustomizationNotReady
          expr: |
            gotk_reconcile_condition{type="Ready",status="False",kind="Kustomization"} == 1
          for: 15m
          annotations:
            summary: "Flux Kustomization {{ $labels.name }} not ready for 15m"
          labels:
            severity: critical

        - alert: FluxHelmReleaseNotReady
          expr: |
            gotk_reconcile_condition{type="Ready",status="False",kind="HelmRelease"} == 1
          for: 15m
          annotations:
            summary: "Flux HelmRelease {{ $labels.name }} not ready for 15m"
          labels:
            severity: critical

        - alert: ArgoCDAppOutOfSync
          expr: |
            argocd_app_info{sync_status="OutOfSync"} == 1
          for: 30m
          annotations:
            summary: "ArgoCD app {{ $labels.name }} out of sync for 30m"
          labels:
            severity: warning

        - alert: RolloutPaused
          expr: |
            rollout_phase{phase="Paused"} == 1
          for: 1h
          annotations:
            summary: "Rollout {{ $labels.name }} paused for 1h"
          labels:
            severity: warning
```

---

## 📊 สรุป Part 101

| Tool | Purpose | Approach |
|------|---------|----------|
| Flux v2 | Pull-based GitOps | Git → K8s |
| ArgoCD | Push-based UI GitOps | Git → K8s |
| Argo Rollouts | Progressive delivery | Canary/Blue-Green |
| Sealed Secrets | Encrypt secrets in Git | Asymmetric crypto |
| ESO | Runtime secret sync | Vault/AWS SM |
| Image Automation | Auto-update images | Semver policy |

---

## 🔗 ต่อไป
- [Part 102: Cost Management and FinOps](./part-102-finops.md)

---
*Part 101 | Steps 971-980 | GitOps Advanced Patterns | Educational Content*
