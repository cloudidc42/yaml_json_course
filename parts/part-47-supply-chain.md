# Part 47: Supply Chain Security
## Steps 451-460: Securing Software Supply Chain

---

## 📖 บทนำ

Supply chain security ป้องกันการโจมตีผ่าน dependencies, build pipelines, container registries และ third-party components เป็นส่วนสำคัญของ SLSA framework

> **หมายเหตุ**: เนื้อหาด้าน security ในหลักสูตรนี้มีไว้เพื่อการศึกษาและการทดสอบระบบที่ได้รับอนุญาตเท่านั้น การนำไปใช้โจมตีระบบที่ไม่ได้รับอนุญาตเป็นสิ่งผิดกฎหมาย

---

## Step 451: SLSA Framework

```
SLSA (Supply-chain Levels for Software Artifacts) Levels:
┌───────────────────────────────────────────────┐
│  Level 4: Two-person review, hermetic builds          │
├───────────────────────────────────────────────┤
│  Level 3: Non-falsifiable provenance, protected repo  │
├───────────────────────────────────────────────┤
│  Level 2: Hosted source, build service, signed prov.  │
├───────────────────────────────────────────────┤
│  Level 1: Build scripts, provenance exists            │
└───────────────────────────────────────────────┘
```

---

## Step 452: SBOM Generation

```yaml
# SBOM (Software Bill of Materials)
# syft myapp:1.0 -o cyclonedx-json > sbom.json
# syft myapp:1.0 -o spdx-json > sbom.spdx.json
# trivy image --format cyclonedx myapp:1.0 > sbom.json
# grype sbom:sbom.json

# GitHub Actions: SBOM + scan
# - name: Generate SBOM
#   uses: anchore/sbom-action@v0
#   with:
#     image: myapp:${{ github.sha }}
#     artifact-name: sbom.spdx.json
#
# - name: Scan
#   uses: anchore/scan-action@v3
#   with:
#     image: myapp:${{ github.sha }}
#     fail-build: true
#     severity-cutoff: critical

# Attach SBOM to image
# cosign attach sbom --sbom sbom.spdx.json myregistry.io/myapp:1.0
```

---

## Step 453: Image Signing with Cosign

```yaml
# Sign image
# cosign sign --key cosign.key myregistry.io/myapp:1.0
# cosign verify --key cosign.pub myregistry.io/myapp:1.0

# Attest provenance
# cosign attest --key cosign.key \
#   --type slsaprovenance \
#   --predicate provenance.json \
#   myregistry.io/myapp:1.0

# Keyless signing (Sigstore)
# cosign sign myregistry.io/myapp:1.0
# cosign verify \
#   --certificate-identity=ci@myorg.github.com \
#   --certificate-oidc-issuer=https://token.actions.githubusercontent.com \
#   myregistry.io/myapp:1.0

# GitHub Actions keyless sign
# permissions:
#   id-token: write
# steps:
#   - uses: sigstore/cosign-installer@v3
#   - run: cosign sign --yes ghcr.io/${{ github.repository }}:${{ github.sha }}
```

---

## Step 454: Dependency Security

```yaml
# .github/dependabot.yml
# version: 2
# updates:
#   - package-ecosystem: npm
#     directory: /
#     schedule:
#       interval: weekly
#     groups:
#       dependencies:
#         patterns: ["*"]
#   
#   - package-ecosystem: docker
#     directory: /
#     schedule:
#       interval: weekly
#   
#   - package-ecosystem: github-actions
#     directory: /
#     schedule:
#       interval: weekly

# Renovate Bot (renovate.json)
# {
#   "extends": ["config:base"],
#   "vulnerabilityAlerts": {
#     "labels": ["security"],
#     "assignees": ["security-team"]
#   },
#   "automerge": true (minor/patch only)
# }
```

---

## Step 455: Secure CI/CD Pipeline

```yaml
# Tekton Pipeline
apiVersion: tekton.dev/v1
kind: Pipeline
metadata:
  name: secure-build
spec:
  tasks:
    - name: clone
      taskRef:
        name: git-clone
    - name: scan-secrets
      taskRef:
        name: trufflehog-scan
      runAfter: [clone]
    - name: test
      taskRef:
        name: golang-test
      runAfter: [clone]
    - name: build
      taskRef:
        name: kaniko
      runAfter: [test, scan-secrets]
    - name: scan-image
      taskRef:
        name: trivy-scan
      runAfter: [build]
    - name: sign-image
      taskRef:
        name: cosign-sign
      runAfter: [scan-image]
    - name: deploy
      taskRef:
        name: kubectl-deploy
      runAfter: [sign-image]
```

---

## Step 456: Registry Security

```yaml
# OPA Policy: ห้าม Docker Hub
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8sblockdockerhub
spec:
  crd:
    spec:
      names:
        kind: K8sBlockDockerHub
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8sblockdockerhub
        
        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          startswith(container.image, "docker.io/")
          msg := sprintf("Docker Hub not allowed: %v", [container.image])
        }
        
        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not contains(container.image, "/")
          msg := sprintf("Must specify registry: %v", [container.image])
        }

---
# Harbor Project Policy
# prevent_vul: true (block pull ถ้ามี HIGH/CRITICAL)
# enable_content_trust: true (ต้อง sign)
# auto_scan: true (สแกนอัตโนมัติใน push)
```

---

## Step 457: Admission for Supply Chain

```yaml
# Kyverno: require signed images
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-signed-images
spec:
  validationFailureAction: Enforce
  background: false
  rules:
    - name: verify-signature
      match:
        any:
          - resources:
              kinds: ["Pod"]
              namespaces: ["production"]
      verifyImages:
        - imageReferences:
            - "myregistry.io/*"
          attestors:
            - count: 1
              entries:
                - keys:
                    publicKeys: |-
                      -----BEGIN PUBLIC KEY-----
                      MFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAE...
                      -----END PUBLIC KEY-----
```

---

## Step 458: GitOps Security

```yaml
# ArgoCD AppProject
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: production
  namespace: argocd
spec:
  sourceRepos:
    - https://github.com/myorg/helm-charts
  destinations:
    - namespace: production
      server: https://kubernetes.default.svc
  namespaceResourceWhitelist:
    - group: apps
      kind: Deployment
    - group: ""
      kind: Service
  clusterResourceBlacklist:
    - group: rbac.authorization.k8s.io
      kind: ClusterRoleBinding

---
# ArgoCD Application
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
spec:
  project: production
  source:
    repoURL: https://github.com/myorg/helm-charts
    targetRevision: v1.2.3
    path: charts/my-app
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

---

## Step 459-460: Workshop - Complete Pipeline

```yaml
# Kyverno: complete supply chain policy
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: supply-chain-policy
spec:
  validationFailureAction: Enforce
  rules:
    - name: approved-registry
      match:
        any:
          - resources:
              kinds: ["Pod"]
      validate:
        message: "Images must come from approved registries"
        pattern:
          spec:
            containers:
              - image: "myregistry.io/* | gcr.io/distroless/*"
    
    - name: no-latest-tag
      match:
        any:
          - resources:
              kinds: ["Pod"]
      validate:
        message: "latest tag not allowed"
        deny:
          conditions:
            any:
              - key: "{{ request.object.spec.containers[].image }}"
                operator: AnyIn
                value: "*.latest"
    
    - name: require-limits
      match:
        any:
          - resources:
              kinds: ["Pod"]
      validate:
        message: "Resource limits required"
        pattern:
          spec:
            containers:
              - resources:
                  limits:
                    memory: "?*"
                    cpu: "?*"
```

---

## 📊 สรุป Part 47

| Layer | Security Control |
|-------|-----------------|
| Source | Secret scanning, SAST, code review |
| Dependencies | Dependabot, dependency audit |
| Build | SBOM, provenance, reproducible builds |
| Image | Scan, sign, approved registry |
| Deploy | Verify signature, admission policies |
| Runtime | Policy, monitoring |

---

## 🔗 ต่อไป
- [Part 48: GitOps Advanced with ArgoCD](./part-48-gitops-argocd.md)

---
*Part 47 | Steps 451-460 | ระดับสูง*
