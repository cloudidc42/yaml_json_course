# Part 83: Developer Experience and Tooling
## Steps 791-800: Local Dev, Skaffold, Telepresence, ArgoCD

---

## Step 791: Developer Experience Overview

```
Developer Experience (DX) for Kubernetes:

Inner Loop (local development):
  - Minikube / Kind / k3d: local cluster
  - Tilt / Skaffold: file-watch + hot reload
  - Telepresence: local code, cloud cluster services
  - DevSpace: cloud dev environments

Outer Loop (CI/CD pipeline):
  - GitHub Actions / GitLab CI: build + test
  - Kaniko / ko / Buildah: in-cluster image builds
  - Helm / Kustomize: packaging and templating
  - ArgoCD / Flux: GitOps delivery

Testing in Kubernetes:
  - Helm test hooks
  - Testkube: k8s-native test framework
  - kube-score / kube-linter: manifest linting
  - Polaris: best practice checks

Developer Self-Service:
  - Backstage: Internal Developer Portal
  - Crossplane: infrastructure as composition
  - Radius: cloud-agnostic application model

Key metrics (DORA):
  - Lead time: commit -> production
  - Deployment frequency
  - MTTR (Mean Time to Restore)
  - Change Failure Rate
```

---

## Step 792: Local Development with Kind

```yaml
# kind-config.yaml: multi-node local cluster
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: dev-cluster
nodes:
  - role: control-plane
    kubeadmConfigPatches:
      - |
        kind: InitConfiguration
        nodeRegistration:
          kubeletExtraArgs:
            node-labels: "ingress-ready=true"
    extraPortMappings:
      - containerPort: 80
        hostPort: 80
        protocol: TCP
      - containerPort: 443
        hostPort: 443
        protocol: TCP
  
  - role: worker
    extraMounts:
      - hostPath: /tmp/kind-pv
        containerPath: /data/pv
  
  - role: worker

# Create: kind create cluster --config kind-config.yaml
# Load image: kind load docker-image myapp:latest --name dev-cluster

---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: local-dev-pv
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: local-storage
  local:
    path: /data/pv
  nodeAffinity:
    required:
      nodeSelectorTerms:
        - matchExpressions:
            - key: kubernetes.io/hostname
              operator: In
              values: [dev-cluster-worker]
```

---

## Step 793: Skaffold for Inner Loop

```yaml
# skaffold.yaml
apiVersion: skaffold/v4beta6
kind: Config
metadata:
  name: myapp

build:
  artifacts:
    - image: myapp
      docker:
        dockerfile: Dockerfile
      sync:
        manual:
          - src: src/**/*.py
            dest: /app
  
  tagPolicy:
    gitCommit: {}
  
  local:
    push: false
    useBuildkit: true

deploy:
  helm:
    releases:
      - name: myapp
        chartPath: charts/myapp
        valuesFiles:
          - charts/myapp/values-dev.yaml
        setValues:
          image.repository: myapp
          image.tag: "{{.IMAGE_TAG}}"
        upgradeOnChange: true

portForward:
  - resourceType: service
    resourceName: myapp
    namespace: default
    port: 80
    localPort: 8080

profiles:
  - name: staging
    patches:
      - op: replace
        path: /deploy/helm/releases/0/valuesFiles/0
        value: charts/myapp/values-staging.yaml

# Run: skaffold dev --port-forward
```

---

## Step 794: Telepresence for Remote Debugging

```yaml
# Telepresence: intercept cluster traffic to local process
# Usage:
# 1. telepresence connect
# 2. telepresence intercept myapp --port 8080:http --env-file /tmp/env
# 3. source /tmp/env && python app.py
# Result: cluster traffic -> local process

---
# DevSpace: dev environment in cluster
version: v2beta1
name: myapp

vars:
  IMAGE: myapp

pipelines:
  dev:
    run: |-
      run_dependencies --all
      create_deployments --all
      start_dev app

images:
  app:
    image: ${IMAGE}
    dockerfile: ./Dockerfile

deployments:
  app:
    helm:
      chart:
        path: ./charts/myapp
      values:
        image:
          repository: ${IMAGE}
          tag: $(devspace.imageTag)

dev:
  app:
    imageSelector: ${IMAGE}
    devImage: golang:1.21-alpine
    command: ["go", "run", "main.go"]
    sync:
      - path: ./:/app
        excludePaths:
          - .git/
    ports:
      - port: "8080"
    terminal:
      enabled: true
```

---

## Step 795: Helm Best Practices

```yaml
# values.yaml: well-structured Helm values
image:
  repository: myapp
  tag: "1.0.0"
  pullPolicy: IfNotPresent

replicaCount: 2

resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 512Mi

autoscaling:
  enabled: false
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70

ingress:
  enabled: false
  className: nginx
  annotations: {}
  hosts:
    - host: myapp.example.com
      paths:
        - path: /
          pathType: Prefix

serviceAccount:
  create: true
  annotations: {}

podDisruptionBudget:
  enabled: true
  minAvailable: 1

---
# _helpers.tpl snippet:
# {{- define "myapp.fullname" -}}
# {{- printf "%s-%s" .Release.Name .Chart.Name | trunc 63 | trimSuffix "-" }}
# {{- end }}
#
# {{- define "myapp.labels" -}}
# helm.sh/chart: {{ .Chart.Name }}-{{ .Chart.Version }}
# app.kubernetes.io/name: {{ .Chart.Name }}
# app.kubernetes.io/instance: {{ .Release.Name }}
# app.kubernetes.io/managed-by: {{ .Release.Service }}
# {{- end }}
```

---

## Step 796: Kustomize for Environment Overlays

```yaml
# overlays/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: production
resources:
  - ../../base
  - hpa.yaml
  - pdb.yaml

namePrefix: prod-

images:
  - name: myapp
    newTag: "2.0.0"

patches:
  - target:
      kind: Deployment
      name: myapp
    patch: |-
      - op: replace
        path: /spec/replicas
        value: 5
      - op: replace
        path: /spec/template/spec/containers/0/resources/requests/cpu
        value: 500m

configMapGenerator:
  - name: app-config
    literals:
      - ENV=production
      - LOG_LEVEL=warn

# Apply: kubectl apply -k overlays/production/
```

---

## Step 797: ArgoCD GitOps

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
    repoURL: https://github.com/myorg/myapp
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
    syncOptions:
      - CreateNamespace=true
      - PrunePropagationPolicy=foreground
      - ApplyOutOfSyncOnly=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m

---
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: production
  namespace: argocd
spec:
  description: Production workloads
  sourceRepos:
    - https://github.com/myorg/*
    - https://charts.bitnami.com/bitnami
  destinations:
    - namespace: production
      server: https://kubernetes.default.svc
    - namespace: databases
      server: https://kubernetes.default.svc
  clusterResourceWhitelist:
    - group: ""
      kind: Namespace
  namespaceResourceBlacklist:
    - group: ""
      kind: ResourceQuota
  roles:
    - name: developer
      description: Read-only + sync access
      policies:
        - p, proj:production:developer, applications, get, production/*, allow
        - p, proj:production:developer, applications, sync, production/*, allow
      groups:
        - developers
```

---

## Step 798: GitHub Actions CI/CD

```yaml
# .github/workflows/deploy.yml
name: Build and Deploy

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      
      - name: Login to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=sha,prefix={{branch}}-
            type=ref,event=branch
            type=semver,pattern={{version}}
      
      - name: Build and push
        id: build-push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
      
      - name: Sign image with Cosign
        uses: sigstore/cosign-installer@v3
      - run: |
          cosign sign --yes \
            ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}@${{ steps.build-push.outputs.digest }}

  deploy:
    needs: build
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v4
      - name: Update image tag in GitOps repo
        run: |
          git clone https://x-access-token:${{ secrets.GITOPS_TOKEN }}@github.com/myorg/gitops-repo
          cd gitops-repo
          sed -i "s|newTag: .*|newTag: main-${{ github.sha }}|" \
            kubernetes/overlays/staging/kustomization.yaml
          git config user.email ci@myorg.com
          git config user.name "CI Bot"
          git commit -am "Update myapp to ${{ github.sha }}"
          git push
```

---

## Step 799: Manifest Validation

```yaml
# .github/workflows/validate.yml
name: Validate Kubernetes Manifests
on: [pull_request]

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Run kube-score
        run: |
          docker run --rm -v $(pwd):/repo zegl/kube-score:latest \
            score kubernetes/**/*.yaml \
            --output-format ci
      
      - name: Run Polaris
        run: |
          docker run --rm -v $(pwd):/repo \
            quay.io/fairwinds/polaris:latest \
            polaris audit \
            --audit-path /repo/kubernetes \
            --format json \
            --set-exit-code-on-danger \
            --set-exit-code-below-score 90
      
      - name: OPA Conftest
        run: |
          docker run --rm -v $(pwd):/project \
            openpolicyagent/conftest:latest \
            test kubernetes/ --policy policy/

# policy/deny_latest_tag.rego:
# package main
# deny[msg] {
#   input.kind == "Deployment"
#   container := input.spec.template.spec.containers[_]
#   endswith(container.image, ":latest")
#   msg := sprintf("Container %v uses :latest tag", [container.name])
# }
```

---

## Step 800: Workshop - Backstage Service Catalog

```yaml
# catalog-info.yaml: Backstage component registration
apiVersion: backstage.io/v1alpha1
kind: Component
metadata:
  name: myapp
  description: Main application service
  annotations:
    github.com/project-slug: myorg/myapp
    backstage.io/kubernetes-id: myapp
    backstage.io/kubernetes-namespace: production
    argocd/app-name: myapp-production
    grafana/dashboard-url: https://grafana.example.com/d/myapp
    pagerduty.com/service-id: PXXXXXX
  tags:
    - python
    - api
    - production
  links:
    - url: https://myapp.example.com
      title: Production URL
    - url: https://runbook.myorg.com/myapp
      title: Runbook
spec:
  type: service
  lifecycle: production
  owner: group:team-alpha
  system: order-platform
  providesApis:
    - myapp-api
  dependsOn:
    - resource:postgres-db
    - resource:redis-cache
    - component:payment-service

---
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: dx-metrics
  namespace: monitoring
spec:
  groups:
    - name: dx
      rules:
        - alert: ArgoCDSyncFailed
          expr: argocd_app_info{sync_status="OutOfSync"} == 1
          for: 30m
          annotations:
            summary: "ArgoCD app {{ $labels.name }} out of sync for 30m"
          labels:
            severity: warning

        - alert: DeploymentFrequencyLow
          expr: increase(argocd_app_sync_total[24h]) < 1
          for: 0m
          annotations:
            summary: "App {{ $labels.name }} not deployed in 24h"
          labels:
            severity: info
```

---

## 📊 สรุป Part 83

| Phase | Tool | Purpose |
|-------|------|---------|
| Inner Loop | Kind + Skaffold | Fast local iteration |
| Intercept | Telepresence | Debug against real cluster |
| Package | Helm + Kustomize | Environment-specific config |
| CI | GitHub Actions | Build, test, sign images |
| CD | ArgoCD | GitOps delivery |
| Validate | kube-score, Polaris | Manifest quality gates |
| Portal | Backstage | Service catalog |

---

## 🔗 ต่อไป
- [Part 84: Advanced Helm and Operators](./part-84-helm-operators.md)

---
*Part 83 | Steps 791-800 | Developer Experience | Educational Content*
