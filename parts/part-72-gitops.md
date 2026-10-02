# Part 72: GitOps with ArgoCD and Flux
## Steps 681-690: Declarative Continuous Delivery

---

## Step 681: GitOps Principles

```
GitOps Core Principles (OpenGitOps):
  1. Declarative: desired state described in files
  2. Versioned and Immutable: Git as source of truth
  3. Pulled Automatically: agents pull from Git
  4. Continuously Reconciled: agents detect + fix drift

Benefits:
  - Audit trail via Git history
  - Rollback = git revert
  - Disaster recovery from Git
  - Review via Pull Requests
  - No direct kubectl access to production

Tools:
  - ArgoCD: declarative, UI-focused, multi-cluster
  - Flux: GitOps toolkit, Helm/Kustomize native
  - Fleet (Rancher): large-scale multi-cluster
```

---

## Step 682: ArgoCD Configuration

```yaml
# ArgoCD ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
  namespace: argocd
data:
  repositories: |
    - url: https://github.com/myorg/k8s-manifests
      type: git
    - url: https://charts.bitnami.com/bitnami
      type: helm
      name: bitnami
  
  admin.enabled: "false"
  application.resourceTrackingMethod: annotation

---
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-rbac-cm
  namespace: argocd
data:
  policy.default: role:readonly
  policy.csv: |
    p, role:platform-admin, applications, *, */*, allow
    p, role:platform-admin, clusters, get, *, allow
    p, role:platform-admin, repositories, *, *, allow
    
    p, role:dev-team, applications, get, dev/*, allow
    p, role:dev-team, applications, sync, dev/*, allow
    
    g, platform-admin@example.com, role:platform-admin
    g, dev-team, role:dev-team
```

---

## Step 683: ArgoCD AppProject

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: production
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  description: "Production environment applications"
  
  sourceRepos:
    - "https://github.com/myorg/k8s-manifests"
    - "https://charts.bitnami.com/bitnami"
  
  destinations:
    - server: https://prod-cluster.example.com
      namespace: "production"
    - server: https://prod-cluster.example.com
      namespace: "databases"
  
  clusterResourceWhitelist:
    - group: ""
      kind: Namespace
  
  namespaceResourceBlacklist:
    - group: ""
      kind: ResourceQuota
  
  roles:
    - name: developer
      policies:
        - p, proj:production:developer, applications, get, production/*, allow
        - p, proj:production:developer, applications, sync, production/*, allow
      groups:
        - dev-team
  
  syncWindows:
    - kind: deny
      schedule: "0 22 * * *"
      duration: 8h
      applications: ["*"]
      namespaces: [production]
      manualSync: false
    
    - kind: allow
      schedule: "0 9 * * 1-5"
      duration: 8h
      applications: ["*"]
```

---

## Step 684: ArgoCD Application

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-production
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: production
  
  source:
    repoURL: https://github.com/myorg/k8s-manifests
    targetRevision: HEAD
    path: apps/myapp/overlays/production
  
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
      allowEmpty: false
    syncOptions:
      - CreateNamespace=true
      - PrunePropagationPolicy=foreground
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
        - /spec/replicas  # Managed by HPA
    - group: ""
      kind: Secret
      jsonPointers:
        - /data

---
# Helm chart application
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: prometheus-stack
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://prometheus-community.github.io/helm-charts
    chart: kube-prometheus-stack
    targetRevision: "55.5.0"
    helm:
      releaseName: prometheus
      values: |
        grafana:
          adminPassword: changeme
        prometheus:
          prometheusSpec:
            retention: 30d
  destination:
    server: https://kubernetes.default.svc
    namespace: monitoring
  syncPolicy:
    syncOptions:
      - CreateNamespace=true
```

---

## Step 685: Flux GitRepository and Kustomization

```yaml
# Flux bootstrap:
# flux bootstrap github \
#   --owner=myorg \
#   --repository=fleet-infra \
#   --branch=main \
#   --path=clusters/production

apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: k8s-manifests
  namespace: flux-system
spec:
  interval: 1m
  url: https://github.com/myorg/k8s-manifests
  ref:
    branch: main
  secretRef:
    name: github-credentials

---
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: myapp-production
  namespace: flux-system
spec:
  interval: 10m
  path: "./apps/myapp/overlays/production"
  prune: true
  sourceRef:
    kind: GitRepository
    name: k8s-manifests
  targetNamespace: production
  healthChecks:
    - apiVersion: apps/v1
      kind: Deployment
      name: myapp
      namespace: production
  postBuild:
    substitute:
      ENVIRONMENT: production
      IMAGE_TAG: v1.2.3
  timeout: 5m
```

---

## Step 686: Flux Helm Integration

```yaml
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: HelmRepository
metadata:
  name: bitnami
  namespace: flux-system
spec:
  interval: 1h
  url: https://charts.bitnami.com/bitnami

---
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: redis
  namespace: production
spec:
  interval: 30m
  chart:
    spec:
      chart: redis
      version: ">=18.0.0 <19.0.0"
      sourceRef:
        kind: HelmRepository
        name: bitnami
        namespace: flux-system
      interval: 1h
  values:
    auth:
      enabled: true
      existingSecret: redis-secret
    replica:
      replicaCount: 3
    master:
      persistence:
        size: 10Gi
  valuesFrom:
    - kind: ConfigMap
      name: redis-values
      valuesKey: values.yaml
  upgrade:
    remediation:
      retries: 3
      remediateLastFailure: true
```

---

## Step 687: Flux Image Automation

```yaml
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImageRepository
metadata:
  name: myapp
  namespace: flux-system
spec:
  image: ghcr.io/myorg/myapp
  interval: 1m
  secretRef:
    name: ghcr-credentials

---
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImagePolicy
metadata:
  name: myapp
  namespace: flux-system
spec:
  imageRepositoryRef:
    name: myapp
  policy:
    semver:
      range: ">=1.0.0 <2.0.0"

---
apiVersion: image.toolkit.fluxcd.io/v1beta1
kind: ImageUpdateAutomation
metadata:
  name: flux-system
  namespace: flux-system
spec:
  interval: 30m
  sourceRef:
    kind: GitRepository
    name: k8s-manifests
  git:
    checkout:
      ref:
        branch: main
    commit:
      author:
        email: fluxbot@example.com
        name: Flux Bot
      messageTemplate: |
        [ci skip] Update image to {{ range .Updated.Images -}}
        {{.}} {{ end -}}
    push:
      branch: main
  update:
    path: ./apps/myapp/overlays/production
    strategy: Setters

# Mark image in deployment.yaml:
# image: ghcr.io/myorg/myapp:v1.0.0 # {"$imagepolicy": "flux-system:myapp"}
```

---

## Step 688: GitOps Security

```yaml
# Kyverno: require GitOps managed label in production
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-gitops-labels
spec:
  validationFailureAction: Audit
  rules:
    - name: check-gitops-managed
      match:
        any:
          - resources:
              kinds: [Deployment, StatefulSet, DaemonSet]
              namespaces: [production]
      validate:
        message: "Workloads in production must be managed by GitOps"
        pattern:
          metadata:
            labels:
              app.kubernetes.io/managed-by: "?(argocd|Helm|kustomize)"

---
# RBAC: developers read-only in production
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: developer-readonly
rules:
  - apiGroups: [""]
    resources: ["pods", "services", "configmaps"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["apps"]
    resources: ["deployments", "replicasets"]
    verbs: ["get", "list", "watch"]
  # No create/update/delete in production
```

---

## Step 689: Multi-Cluster GitOps

```yaml
# ArgoCD ApplicationSet: deploy to all prod clusters
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: myapp-all-clusters
  namespace: argocd
spec:
  generators:
    - clusters:
        selector:
          matchLabels:
            environment: production
  
  template:
    metadata:
      name: "{{name}}-myapp"
    spec:
      project: production
      source:
        repoURL: https://github.com/myorg/k8s-manifests
        targetRevision: HEAD
        path: apps/myapp/overlays/{{metadata.labels.region}}
      destination:
        server: "{{server}}"
        namespace: production
      syncPolicy:
        automated:
          prune: true
          selfHeal: true

---
# Flux multi-cluster structure:
# clusters/
#   prod-eu/flux-system/ apps/
#   prod-us/flux-system/ apps/

apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: infrastructure
  namespace: flux-system
spec:
  interval: 10m
  path: "./infrastructure/production"
  prune: true
  sourceRef:
    kind: GitRepository
    name: fleet-infra
  decryption:
    provider: sops
    secretRef:
      name: sops-age
```

---

## Step 690: Workshop - GitOps Best Practices

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: gitops-best-practices
  namespace: argocd
data:
  practices.yaml: |
    repository_structure:
      - Separate app code from config repo
      - Kustomize overlays for environments
      - Pin chart versions (not latest)
      - Secrets externally (ESO/Vault/SOPS)
      - Tag releases with semver
    
    security:
      - No secrets in Git (use SOPS/age)
      - Developers read-only in production
      - PR reviews for production changes
      - Signed commits (GPG/SSH)
      - Branch protection on main
      - Scan manifests in CI (kubeconform, Trivy)
    
    argocd_tips:
      - AppProjects for access scoping
      - Sync windows for change freezes
      - ignoreDifferences for HPA replicas
      - Notifications for sync failures
    
    flux_tips:
      - dependsOn for ordered reconciliation
      - Flux alerts to Slack
      - ImageUpdateAutomation for CD
      - SOPS for encrypted secrets in Git

---
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: gitops-health
  namespace: monitoring
spec:
  groups:
    - name: gitops
      rules:
        - alert: ArgoAppOutOfSync
          expr: argocd_app_info{sync_status="OutOfSync"} > 0
          for: 15m
          annotations:
            summary: "ArgoCD app {{ $labels.name }} is OutOfSync"
          labels:
            severity: warning

        - alert: FluxKustomizationNotReady
          expr: gotk_reconcile_condition{type="Ready",status="False",kind="Kustomization"} > 0
          for: 10m
          annotations:
            summary: "Flux Kustomization {{ $labels.name }} not ready"
          labels:
            severity: warning
```

---

## 📊 สรุป Part 72

| Feature | ArgoCD | Flux |
|---------|--------|------|
| UI | Web UI included | Requires Weave GitOps |
| Multi-cluster | ApplicationSet | Cluster API + path |
| Image update | Plugin | Native ImageUpdateAutomation |
| Secrets | External only | SOPS native |
| Helm | Native | HelmRelease CRD |
| Kustomize | Native | Kustomization CRD |

---

## 🔗 ต่อไป
- [Part 73: Service Mesh Deep Dive](./part-73-service-mesh.md)

---
*Part 72 | Steps 681-690 | GitOps | Educational Content*
