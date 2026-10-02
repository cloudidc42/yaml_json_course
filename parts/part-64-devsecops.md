# Part 64: DevSecOps Pipeline
## Steps 601-610: Security in CI/CD for Kubernetes

---

> **⚠️**: เนื้อหาด้าน security ในหลักสูตรนี้มีไว้เพื่อการศึกษาและการทดสอบระบบที่ได้รับอนุญาตเท่านั้น

## Step 601: DevSecOps Pipeline Overview

```
DevSecOps: Shift security left into CI/CD

Security gates in pipeline:
  PRE-COMMIT:
    - Secret scanning (git-secrets, gitleaks)
    - IaC linting (checkov)
    - Code quality (pre-commit hooks)
  
  BUILD:
    - SAST (semgrep, CodeQL)
    - Dependency scan (trivy)
    - Image build (multi-stage, minimal base)
  
  IMAGE SCAN:
    - CVE scan (trivy, grype)
    - SBOM generation (syft)
    - Image signing (cosign)
    - Attestation (cosign attest)
  
  DEPLOY:
    - Kyverno policy gates
    - OPA Conftest (YAML lint)
    - Helm chart security scan
  
  RUNTIME:
    - Falco (anomaly detection)
    - Cilium (network policy)
    - Prometheus (metric alerts)
```

---

## Step 602: GitHub Actions - Security Pipeline

```yaml
# .github/workflows/security.yml
name: Security Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

permissions:
  contents: read
  security-events: write
  actions: read

jobs:
  secret-scan:
    name: Secret Scanning
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      
      - name: Gitleaks scan
        uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
      
      - name: TruffleHog scan
        uses: trufflesecurity/trufflehog@main
        with:
          path: ./
          base: ${{ github.event.repository.default_branch }}
          head: HEAD
  
  sast:
    name: SAST
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Semgrep SAST
        uses: returntocorp/semgrep-action@v1
        with:
          config: >-
            p/owasp-top-ten
            p/kubernetes
            p/docker
          generateSarif: "1"
      
      - name: Upload SARIF
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: semgrep.sarif
  
  dependency-scan:
    name: Dependency Scan
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Trivy dependency scan
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: fs
          scan-ref: .
          format: sarif
          output: trivy-results.sarif
          severity: CRITICAL,HIGH
          exit-code: 1
      
      - name: Upload SARIF
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: trivy-results.sarif
  
  image-build-scan:
    name: Image Build and Scan
    runs-on: ubuntu-latest
    needs: [secret-scan, sast, dependency-scan]
    outputs:
      image-digest: ${{ steps.build.outputs.digest }}
    steps:
      - uses: actions/checkout@v4
      
      - name: Build image
        id: build
        uses: docker/build-push-action@v5
        with:
          push: false
          tags: myapp:${{ github.sha }}
      
      - name: Scan image with Trivy
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: myapp:${{ github.sha }}
          format: table
          exit-code: 1
          severity: CRITICAL
      
      - name: Push to registry
        if: github.event_name != 'pull_request'
        uses: docker/build-push-action@v5
        with:
          push: true
          tags: myregistry.io/myapp:${{ github.sha }}
```

---

## Step 603: Image Signing with Cosign

```yaml
  sign-image:
    name: Sign Image
    runs-on: ubuntu-latest
    needs: [image-build-scan]
    if: github.event_name != 'pull_request'
    permissions:
      contents: read
      id-token: write
    steps:
      - name: Install cosign
        uses: sigstore/cosign-installer@v3
      
      - name: Sign image (keyless via OIDC)
        env:
          DIGEST: ${{ needs.image-build-scan.outputs.image-digest }}
        run: |
          cosign sign --yes \
            myregistry.io/myapp@${{ env.DIGEST }}
      
      - name: Generate SBOM
        uses: anchore/sbom-action@v0
        with:
          image: myregistry.io/myapp@${{ needs.image-build-scan.outputs.image-digest }}
          format: spdx-json
          output-file: sbom.spdx.json
      
      - name: Attest SBOM
        run: |
          cosign attest --yes \
            --predicate sbom.spdx.json \
            --type spdxjson \
            myregistry.io/myapp@${{ needs.image-build-scan.outputs.image-digest }}
```

---

## Step 604: OPA Conftest - Policy Gates

```rego
# policy/kubernetes.rego
package kubernetes

# Deny: no resource limits
deny[msg] {
  input.kind == "Deployment"
  container := input.spec.template.spec.containers[_]
  not container.resources.limits.memory
  msg := sprintf("Container %v must have memory limits", [container.name])
}

# Deny: privileged containers
deny[msg] {
  input.kind == "Pod"
  container := input.spec.containers[_]
  container.securityContext.privileged == true
  msg := sprintf("Container %v must not be privileged", [container.name])
}

# Deny: writable root filesystem
deny[msg] {
  input.kind == "Deployment"
  container := input.spec.template.spec.containers[_]
  not container.securityContext.readOnlyRootFilesystem == true
  msg := sprintf("Container %v must have readOnlyRootFilesystem: true", [container.name])
}

# Warn: no liveness probe
warn[msg] {
  input.kind == "Deployment"
  container := input.spec.template.spec.containers[_]
  not container.livenessProbe
  msg := sprintf("Container %v has no livenessProbe", [container.name])
}
```

```yaml
# GitHub Actions: Conftest gate
  conftest:
    name: Policy Gate
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Install conftest
        run: |
          wget https://github.com/open-policy-agent/conftest/releases/latest/download/conftest_Linux_x86_64.tar.gz
          tar -xzf conftest_Linux_x86_64.tar.gz
          sudo mv conftest /usr/local/bin/
      
      - name: Run Conftest
        run: |
          conftest test k8s/ \
            --policy policy/ \
            --output table
```

---

## Step 605: Checkov - IaC Security

```yaml
  iac-scan:
    name: IaC Security Scan
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Checkov scan
        uses: bridgecrewio/checkov-action@master
        with:
          directory: .
          framework: kubernetes,helm,terraform
          output_format: sarif
          output_file_path: checkov.sarif
          soft_fail: false
          check: |
            CKV_K8S_1,CKV_K8S_6,CKV_K8S_8,
            CKV_K8S_9,CKV_K8S_10,CKV_K8S_11,
            CKV_K8S_14,CKV_K8S_15,CKV_K8S_16,
            CKV_K8S_17,CKV_K8S_20,CKV_K8S_21,
            CKV_K8S_22,CKV_K8S_25,CKV_K8S_28
      
      - name: Upload SARIF
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: checkov.sarif
```

---

## Step 606: ArgoCD - GitOps Secure Deployment

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: production-app
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: production
  source:
    repoURL: https://github.com/myorg/k8s-configs
    targetRevision: HEAD
    path: apps/production
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
      limit: 3
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
  description: "Production project - restricted"
  sourceRepos:
    - https://github.com/myorg/k8s-configs
  destinations:
    - namespace: production
      server: https://kubernetes.default.svc
    - namespace: monitoring
      server: https://kubernetes.default.svc
  namespaceResourceBlacklist:
    - group: ""
      kind: ResourceQuota
  clusterResourceWhitelist:
    - group: ""
      kind: Namespace
  roles:
    - name: deployers
      description: "Deploy to production"
      policies:
        - p, proj:production:deployers, applications, sync, production/*, allow
        - p, proj:production:deployers, applications, get, production/*, allow
      groups:
        - cicd-team
```

---

## Step 607: Helm Security Values

```yaml
# values-production.yaml
replicaCount: 3

image:
  repository: myregistry.io/myapp
  tag: ""
  pullPolicy: Always

securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  readOnlyRootFilesystem: true
  allowPrivilegeEscalation: false
  capabilities:
    drop: [ALL]

podSecurityContext:
  seccompProfile:
    type: RuntimeDefault

resources:
  limits:
    cpu: 500m
    memory: 256Mi
  requests:
    cpu: 100m
    memory: 128Mi

networkPolicy:
  enabled: true
  ingress:
    - from:
        - podSelector:
            matchLabels:
              role: frontend
      ports:
        - port: 8080

serviceAccount:
  create: true
  automountServiceAccountToken: false
```

---

## Step 608: Supply Chain Security (SLSA)

```yaml
# SLSA Level 3 provenance
  slsa-provenance:
    name: SLSA Provenance
    needs: [image-build-scan]
    uses: slsa-framework/slsa-github-generator/.github/workflows/generator_container_slsa3.yml@v1.9.0
    with:
      image: myregistry.io/myapp
      digest: ${{ needs.image-build-scan.outputs.image-digest }}
    secrets:
      registry-username: ${{ secrets.REGISTRY_USER }}
      registry-password: ${{ secrets.REGISTRY_TOKEN }}

---
# Kyverno: verify SLSA provenance
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: verify-slsa-provenance
spec:
  validationFailureAction: Enforce
  rules:
    - name: check-provenance
      match:
        any:
          - resources:
              kinds: [Pod]
              namespaces: [production]
      verifyImages:
        - imageReferences:
            - "myregistry.io/myapp:*"
          attestors:
            - entries:
                - keyless:
                    subject: "https://github.com/myorg/myapp/.github/workflows/build.yml@refs/heads/main"
                    issuer: "https://token.actions.githubusercontent.com"
          attestations:
            - predicateType: https://slsa.dev/provenance/v1
```

---

## Step 609: Security Metrics Pipeline

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: devsecops-metrics
  namespace: monitoring
spec:
  groups:
    - name: devsecops
      rules:
        - record: devsecops:critical_cves_in_production
          expr: |
            sum(trivy_image_vulnerabilities{severity="CRITICAL", namespace="production"})
        
        - alert: UnpatchedCriticalCVE
          expr: devsecops:critical_cves_in_production > 0
          for: 168h
          annotations:
            summary: "Critical CVE unpatched for 7+ days"
          labels:
            severity: critical
        
        - record: devsecops:signed_image_ratio
          expr: |
            sum(kyverno_policy_results_total{result="pass", rule="check-image-signature"}) /
            sum(kyverno_policy_results_total{rule="check-image-signature"})
        
        - alert: UnsignedImagesInProduction
          expr: devsecops:signed_image_ratio < 1.0
          annotations:
            summary: "Unsigned images running in production"
          labels:
            severity: critical
```

---

## Step 610: Workshop - Complete DevSecOps Checklist

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: devsecops-checklist
  namespace: security
data:
  checklist.yaml: |
    pre_commit:
      - tool: gitleaks
        status: enabled
      - tool: checkov
        status: enabled
    
    ci_pipeline:
      - stage: secret_scan
        tools: [gitleaks, trufflehog]
        gate: blocking
      - stage: sast
        tools: [semgrep, codeql]
        gate: blocking_on_critical
      - stage: dependency_scan
        tools: [trivy, snyk]
        gate: blocking_on_critical
      - stage: image_build
        base_image: distroless
        multistage: true
      - stage: image_scan
        tools: [trivy, grype]
        gate: blocking_on_critical
      - stage: sign_image
        tool: cosign
        mode: keyless
      - stage: sbom_generation
        tool: syft
        format: spdx-json
      - stage: policy_gate
        tools: [conftest, checkov]
        gate: blocking
      - stage: slsa_provenance
        level: 3
    
    deployment:
      - tool: argocd
        gitops: true
        auto_sync: true
      - policy: kyverno_verify_signature
        enforced: true
    
    runtime:
      - tool: falco
        custom_rules: true
      - tool: cilium
        networkpolicy: default_deny
      - monitoring: prometheus_grafana
```

---

## 📊 สรุป Part 64

| Stage | Tool | Security Gate |
|-------|------|---------------|
| Pre-commit | gitleaks, checkov | Block secrets/misconfig |
| Build | semgrep, trivy | Block critical vulns |
| Image | cosign, syft | Sign + SBOM |
| Deploy | conftest, kyverno | Policy enforcement |
| Runtime | falco, cilium | Anomaly detection |

---

## 🔗 ต่อไป
- [Part 65: Threat Modeling](./part-65-threat-modeling.md)

---
*Part 64 | Steps 601-610 | ระดับผู้เชี่ยวชาญ | Educational Use Only*
