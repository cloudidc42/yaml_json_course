# Part 89: CI/CD Pipelines Advanced
## Steps 851-860: Tekton, Flux, Image Promotion, Policy Gates

---

## Step 851: CI/CD Pipeline Overview

```
Advanced CI/CD on Kubernetes:

Pipeline Stages:
  1. Source: Git trigger (push, PR, tag)
  2. Build: Docker/Kaniko/ko image build
  3. Test: unit, integration, contract tests
  4. Security: SAST, dependency scan, image scan
  5. Publish: push to registry, sign with Cosign
  6. Deploy to staging: GitOps update
  7. Integration tests: k6, Newman, Cypress
  8. Promote to production: gate on metrics
  9. Monitor: deployment health check

Tools Comparison:
  Tekton: Kubernetes-native, CNCF, highly flexible
    - Task, Pipeline, PipelineRun, TaskRun
    - Tekton Triggers: Git webhooks
  
  GitHub Actions: Most popular, easy to start
    - .github/workflows/
    - Large marketplace (20k+ actions)
  
  Jenkins X: Cloud-native Jenkins
    - jx pipeline
    - Lighthouse (GitHub webhook handler)
  
  Argo Workflows: DAG-based, parallel steps
    - Good for ML pipelines
  
  Flux: GitOps-only CD
    - Source Controller: monitor Git/Helm/OCI
    - Kustomize/Helm Controller: reconcile
    - Notification Controller: alerts

GitOps Principles:
  - Declarative: desired state in Git
  - Versioned: every change is a commit
  - Automated: system reconciles continuously
  - Auditable: Git history = audit log
```

---

## Step 852: Tekton Pipeline

```yaml
# Tekton: Task = single step
apiVersion: tekton.dev/v1
kind: Task
metadata:
  name: build-and-push
  namespace: tekton-pipelines
spec:
  params:
    - name: image
      type: string
    - name: context
      type: string
      default: .
  
  workspaces:
    - name: source
    - name: dockerconfig
      mountPath: /kaniko/.docker
  
  steps:
    - name: build-push
      image: gcr.io/kaniko-project/executor:latest
      command:
        - /kaniko/executor
      args:
        - --context=$(workspaces.source.path)/$(params.context)
        - --dockerfile=$(workspaces.source.path)/Dockerfile
        - --destination=$(params.image)
        - --cache=true
        - --cache-ttl=24h

---
apiVersion: tekton.dev/v1
kind: Pipeline
metadata:
  name: ci-pipeline
  namespace: tekton-pipelines
spec:
  params:
    - name: git-url
    - name: git-revision
      default: main
    - name: image
  
  workspaces:
    - name: source
    - name: dockerconfig
  
  tasks:
    - name: clone
      taskRef:
        name: git-clone
        kind: ClusterTask
      workspaces:
        - name: output
          workspace: source
      params:
        - name: url
          value: $(params.git-url)
        - name: revision
          value: $(params.git-revision)
    
    - name: test
      runAfter: [clone]
      taskRef:
        name: run-tests
      workspaces:
        - name: source
          workspace: source
      params:
        - name: image
          value: python:3.11
    
    - name: build
      runAfter: [test]
      taskRef:
        name: build-and-push
      workspaces:
        - name: source
          workspace: source
        - name: dockerconfig
          workspace: dockerconfig
      params:
        - name: image
          value: $(params.image)
```

---

## Step 853: Tekton Triggers

```yaml
# EventListener: receive webhooks
apiVersion: triggers.tekton.dev/v1beta1
kind: EventListener
metadata:
  name: github-listener
  namespace: tekton-pipelines
spec:
  serviceAccountName: tekton-triggers-sa
  triggers:
    - name: github-push
      interceptors:
        - ref:
            name: github
          params:
            - name: secretRef
              value:
                secretName: github-webhook-secret
                secretKey: secret
            - name: eventTypes
              value: [push]
        - ref:
            name: cel
          params:
            - name: filter
              value: "body.ref == 'refs/heads/main'"
      bindings:
        - ref: github-push-binding
      template:
        ref: pipeline-run-template

---
apiVersion: triggers.tekton.dev/v1beta1
kind: TriggerBinding
metadata:
  name: github-push-binding
  namespace: tekton-pipelines
spec:
  params:
    - name: git-url
      value: $(body.repository.clone_url)
    - name: git-revision
      value: $(body.head_commit.id)
    - name: image
      value: ghcr.io/myorg/myapp:$(body.head_commit.id)

---
apiVersion: triggers.tekton.dev/v1beta1
kind: TriggerTemplate
metadata:
  name: pipeline-run-template
  namespace: tekton-pipelines
spec:
  params:
    - name: git-url
    - name: git-revision
    - name: image
  resourcetemplates:
    - apiVersion: tekton.dev/v1
      kind: PipelineRun
      metadata:
        generateName: ci-run-
      spec:
        pipelineRef:
          name: ci-pipeline
        params:
          - name: git-url
            value: $(tt.params.git-url)
          - name: git-revision
            value: $(tt.params.git-revision)
          - name: image
            value: $(tt.params.image)
        workspaces:
          - name: source
            volumeClaimTemplate:
              spec:
                accessModes: [ReadWriteOnce]
                resources:
                  requests:
                    storage: 1Gi
          - name: dockerconfig
            secret:
              secretName: registry-credentials
```

---

## Step 854: Flux GitOps

```yaml
# GitRepository: watch source repo
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: myapp
  namespace: flux-system
spec:
  interval: 1m
  url: https://github.com/myorg/myapp
  secretRef:
    name: github-credentials
  ref:
    branch: main

---
# Kustomization: reconcile Kustomize overlays
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: myapp-production
  namespace: flux-system
spec:
  interval: 5m
  path: ./kubernetes/overlays/production
  sourceRef:
    kind: GitRepository
    name: myapp
  prune: true
  wait: true
  timeout: 10m
  healthChecks:
    - apiVersion: apps/v1
      kind: Deployment
      name: myapp
      namespace: production
  postBuild:
    substitute:
      CLUSTER_NAME: production
    substituteFrom:
      - kind: ConfigMap
        name: cluster-vars

---
# HelmRelease: deploy Helm chart from OCI
apiVersion: helm.toolkit.fluxcd.io/v2beta2
kind: HelmRelease
metadata:
  name: podinfo
  namespace: production
spec:
  interval: 10m
  chart:
    spec:
      chart: podinfo
      version: ">=6.0.0"
      sourceRef:
        kind: HelmRepository
        name: podinfo
        namespace: flux-system
  values:
    replicaCount: 2
    resources:
      requests:
        cpu: 100m
        memory: 128Mi
  upgrade:
    remediation:
      remediateLastFailure: true
  rollback:
    timeout: 5m
    cleanupOnFail: true
```

---

## Step 855: Image Automation with Flux

```yaml
# ImageRepository: watch for new image tags
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImageRepository
metadata:
  name: myapp
  namespace: flux-system
spec:
  image: ghcr.io/myorg/myapp
  interval: 5m
  secretRef:
    name: registry-credentials

---
# ImagePolicy: define which tags to accept
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
# ImageUpdateAutomation: auto-commit new image tags
apiVersion: image.toolkit.fluxcd.io/v1beta1
kind: ImageUpdateAutomation
metadata:
  name: myapp
  namespace: flux-system
spec:
  interval: 1m
  sourceRef:
    kind: GitRepository
    name: myapp
  git:
    checkout:
      ref:
        branch: main
    commit:
      author:
        name: fluxcd-bot
        email: fluxcd@mycompany.com
      messageTemplate: "chore: update myapp to {{range .Updated.Images}}{{.}}{{end}}"
    push:
      branch: main
  update:
    strategy: Setters
    path: ./kubernetes

---
# kustomization.yaml: mark image field for auto-update
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../base
images:
  - name: ghcr.io/myorg/myapp
    newTag: 1.0.0  # {"$imagepolicy": "flux-system:myapp:tag"}
```

---

## Step 856: Policy Gates in CI/CD

```yaml
# Kyverno: block deploy if image not signed
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: verify-image-signature
spec:
  validationFailureAction: Enforce
  background: false
  rules:
    - name: verify-cosign-signature
      match:
        any:
          - resources:
              kinds: [Pod]
              namespaces: [production, staging]
      verifyImages:
        - imageReferences:
            - "ghcr.io/myorg/*"
          attestors:
            - count: 1
              entries:
                - keyless:
                    subject: "https://github.com/myorg/myapp/.github/workflows/*.yml@refs/heads/main"
                    issuer: "https://token.actions.githubusercontent.com"

---
# OPA Gatekeeper: require version labels
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredLabels
metadata:
  name: require-deploy-labels
spec:
  match:
    kinds:
      - apiGroups: [apps]
        kinds: [Deployment]
    namespaces: [production, staging]
  parameters:
    labels:
      - key: app.kubernetes.io/version
        allowedRegex: "^v?[0-9]+\\.[0-9]+\\.[0-9]+"
      - key: app.kubernetes.io/managed-by
      - key: team
```

---

## Step 857: Promotion Workflow

```yaml
# ArgoCD ApplicationSet: multi-environment promotion
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: myapp-promotion
  namespace: argocd
spec:
  generators:
    - list:
        elements:
          - env: dev
            cluster: dev-cluster
            namespace: development
            autoSync: "true"
          - env: staging
            cluster: staging-cluster
            namespace: staging
            autoSync: "true"
          - env: production
            cluster: prod-cluster
            namespace: production
            autoSync: "false"
  template:
    metadata:
      name: "myapp-{{env}}"
    spec:
      project: "{{env}}"
      source:
        repoURL: https://github.com/myorg/gitops-repo
        targetRevision: main
        path: "environments/{{env}}"
      destination:
        server: "{{cluster}}"
        namespace: "{{namespace}}"
      syncPolicy:
        automated:
          prune: true
          selfHeal: "{{autoSync}}"
```

---

## Step 858: Deployment Health Gates

```yaml
# Argo Rollouts: analysis after deployment
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: deployment-health
  namespace: production
spec:
  args:
    - name: deployment-name
    - name: namespace
  metrics:
    - name: error-rate
      interval: 2m
      successCondition: result[0] < 0.01
      failureLimit: 3
      provider:
        prometheus:
          address: http://prometheus.monitoring.svc:9090
          query: |
            sum(rate(http_requests_total{
              deployment="{{args.deployment-name}}",
              status_code=~"5.."
            }[2m]))
            /
            sum(rate(http_requests_total{
              deployment="{{args.deployment-name}}"
            }[2m]))
    
    - name: p99-latency
      interval: 2m
      successCondition: result[0] < 0.5
      failureLimit: 3
      provider:
        prometheus:
          address: http://prometheus.monitoring.svc:9090
          query: |
            histogram_quantile(0.99, sum(rate(
              http_request_duration_seconds_bucket{
                deployment="{{args.deployment-name}}"
              }[2m])) by (le))

---
# Flux notification on deploy
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
```

---

## Step 859: Multi-Environment Config Management

```yaml
# Sealed Secrets: encrypt secrets for GitOps
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: db-credentials
  namespace: production
spec:
  encryptedData:
    password: AgBvxyz123encrypted...
    username: AgBabc456encrypted...
  template:
    metadata:
      name: db-credentials
      namespace: production
    type: Opaque

---
# Kustomize: per-environment config
# environments/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: production
resources:
  - ../../base
  - sealed-db-creds.yaml
  - hpa.yaml

patches:
  - target:
      kind: Deployment
      name: myapp
    patch: |-
      - op: replace
        path: /spec/replicas
        value: 10
      - op: replace
        path: /spec/template/spec/containers/0/resources/requests/cpu
        value: 500m

configMapGenerator:
  - name: app-config
    behavior: replace
    literals:
      - ENV=production
      - LOG_LEVEL=warn
      - FEATURE_FLAG_ENABLED=true
```

---

## Step 860: Workshop - Pipeline Monitoring

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: cicd-alerts
  namespace: monitoring
spec:
  groups:
    - name: cicd
      rules:
        - alert: TektonPipelineRunFailed
          expr: |
            tekton_pipelinerun_count{status="failed"} > 0
          for: 0m
          annotations:
            summary: "Tekton PipelineRun {{ $labels.pipeline }} failed"
          labels:
            severity: warning

        - alert: FluxKustomizationNotReady
          expr: |
            gotk_reconcile_condition{type="Ready",status="False",kind="Kustomization"} == 1
          for: 10m
          annotations:
            summary: "Flux Kustomization {{ $labels.name }} not ready for 10m"
          labels:
            severity: warning

        - alert: ArgoCDApplicationOutOfSync
          expr: |
            argocd_app_info{sync_status="OutOfSync"} == 1
          for: 30m
          annotations:
            summary: "ArgoCD app {{ $labels.name }} out of sync for 30m"
          labels:
            severity: warning

        - alert: ImageNotUpdated
          expr: |
            time() - gotk_source_info{kind="ImageRepository"} > 3600
          for: 0m
          annotations:
            summary: "Image repository {{ $labels.name }} not polled in > 1h"
          labels:
            severity: info
```

---

## 📊 สรุป Part 89

| Tool | Role | Strength |
|------|------|----------|
| Tekton | CI pipeline | Kubernetes-native, flexible |
| GitHub Actions | CI/CD | Easy to start, large ecosystem |
| Flux | GitOps CD | Lightweight, multi-tenancy |
| ArgoCD | GitOps CD | UI, multi-cluster |
| Argo Rollouts | Progressive delivery | Canary + analysis |
| Sealed Secrets | Secret management | GitOps-safe secrets |
| Kyverno | Policy gates | Admission control |

---

## 🔗 ต่อไป
- [Part 90: Kubernetes API Extensions](./part-90-api-extensions.md)

---
*Part 89 | Steps 851-860 | CI/CD Pipelines Advanced | Educational Content*
