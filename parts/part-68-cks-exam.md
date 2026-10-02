# Part 68: CKS Exam Preparation
## Steps 641-650: Certified Kubernetes Security Specialist

---

> **⚠️**: เนื้อหาด้าน security ในหลักสูตรนี้มีไว้เพื่อการศึกษาและการทดสอบระบบที่ได้รับอนุญาตเท่านั้น

## Step 641: CKS Exam Overview

```
CKS (Certified Kubernetes Security Specialist):
  Prerequisite: CKA certification
  Duration: 2 hours
  Format: Performance-based (hands-on)
  Pass score: 67%

Exam Domains (2024):
  15% - Cluster Setup
  15% - Cluster Hardening
  20% - System Hardening
  20% - Minimize Microservice Vulnerabilities
  20% - Supply Chain Security
  10% - Monitoring, Logging, Runtime Security

Key tools to master:
  - kube-bench (CIS benchmark)
  - Falco (runtime security)
  - Trivy (image scanning)
  - OPA/Gatekeeper
  - AppArmor/Seccomp
  - NetworkPolicy
  - RBAC
  - ImagePolicyWebhook

Exam tips:
  - Use kubectl imperative commands (faster)
  - Know shortcuts: po=pods, svc=services, cm=configmaps
  - Keep bookmarks: official K8s docs
  - Time management: skip hard questions, come back
  - Read the question carefully (namespace!)
```

---

## Step 642: CKS Domain 1 - Cluster Setup

```yaml
# NetworkPolicy: restrict to only allow ingress from pods labeled role=frontend
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-restrict
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              role: frontend
      ports:
        - port: 3000

---
# Ingress with TLS
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: secure-ingress
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  tls:
    - hosts:
        - example.com
      secretName: example-tls
  rules:
    - host: example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: my-service
                port:
                  number: 80

---
# Audit Policy
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  - level: RequestResponse
    resources:
      - group: ""
        resources: ["secrets"]
  - level: RequestResponse
    resources:
      - group: ""
        resources: ["pods/exec", "pods/portforward"]
  - level: None
    verbs: ["get", "watch", "list"]
    resources:
      - group: ""
        resources: ["configmaps", "services"]
  - level: Metadata
```

---

## Step 643: CKS Domain 2 - Cluster Hardening

```yaml
# Least-privilege RBAC
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: dev
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pod-reader-binding
  namespace: dev
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: pod-reader
subjects:
  - kind: User
    name: jane
    apiGroup: rbac.authorization.k8s.io

---
# Imperative commands (faster in exam):
# kubectl create role pod-reader --verb=get,list,watch --resource=pods -n dev
# kubectl create rolebinding pod-reader-binding --role=pod-reader --user=jane -n dev

# Disable SA auto-mount
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-sa
  namespace: default
automountServiceAccountToken: false
```

---

## Step 644: CKS Domain 3 - System Hardening

```yaml
# AppArmor: apply profile to pod
apiVersion: v1
kind: Pod
metadata:
  name: apparmor-pod
  annotations:
    container.apparmor.security.beta.kubernetes.io/app: localhost/k8s-deny-write
spec:
  containers:
    - name: app
      image: nginx:alpine

---
# Seccomp: RuntimeDefault profile
apiVersion: v1
kind: Pod
metadata:
  name: seccomp-pod
spec:
  securityContext:
    seccompProfile:
      type: RuntimeDefault
  containers:
    - name: app
      image: nginx:alpine

---
# Drop all capabilities
apiVersion: v1
kind: Pod
metadata:
  name: hardened-pod
spec:
  containers:
    - name: app
      image: nginx:alpine
      securityContext:
        allowPrivilegeEscalation: false
        runAsNonRoot: true
        runAsUser: 1000
        readOnlyRootFilesystem: true
        capabilities:
          drop: [ALL]
```

---

## Step 645: CKS Domain 4 - Minimize Microservice Vulnerabilities

```yaml
# OPA Gatekeeper: deny privileged pods
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8spspprivilegedcontainer
spec:
  crd:
    spec:
      names:
        kind: K8sPSPPrivilegedContainer
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8spspprivilegedcontainer
        
        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          container.securityContext.privileged == true
          msg := sprintf("Container %v is privileged", [container.name])
        }

---
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sPSPPrivilegedContainer
metadata:
  name: psp-privileged-container
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]

---
# PSS: apply to namespace
# kubectl label namespace production \
#   pod-security.kubernetes.io/enforce=restricted

---
# Use secret in pod
apiVersion: v1
kind: Pod
metadata:
  name: secret-pod
spec:
  containers:
    - name: app
      image: nginx
      env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: my-secret
              key: password
      volumeMounts:
        - name: secret-vol
          mountPath: /etc/secret
          readOnly: true
  volumes:
    - name: secret-vol
      secret:
        secretName: my-secret
        defaultMode: 0400
```

---

## Step 646: CKS Domain 5 - Supply Chain Security

```yaml
# ImagePolicyWebhook configuration
apiVersion: apiserver.config.k8s.io/v1
kind: AdmissionConfiguration
plugins:
  - name: ImagePolicyWebhook
    configuration:
      imagePolicy:
        kubeConfigFile: /etc/kubernetes/image-review-kubeconfig.yaml
        allowTTL: 50
        denyTTL: 50
        retryBackoff: 500
        defaultAllow: false

---
# Signed image verification with Kyverno
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: verify-image-signature
spec:
  validationFailureAction: Enforce
  rules:
    - name: check-image-signature
      match:
        any:
          - resources:
              kinds: [Pod]
              namespaces: [production]
      verifyImages:
        - imageReferences:
            - "myregistry.io/*"
          attestors:
            - entries:
                - keyless:
                    subject: "https://github.com/myorg/*"
                    issuer: "https://token.actions.githubusercontent.com"

---
# High security score pod (kubesec friendly)
apiVersion: v1
kind: Pod
metadata:
  name: high-score-pod
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    seccompProfile:
      type: RuntimeDefault
  containers:
    - name: app
      image: nginx:alpine
      securityContext:
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        capabilities:
          drop: [ALL]
      resources:
        limits:
          cpu: 200m
          memory: 128Mi
        requests:
          cpu: 100m
          memory: 64Mi
```

---

## Step 647: CKS Domain 6 - Monitoring and Runtime Security

```yaml
# Falco custom rules
apiVersion: v1
kind: ConfigMap
metadata:
  name: falco-cks-rules
  namespace: falco
data:
  cks-rules.yaml: |
    - rule: Shell spawned in container
      desc: Detect shell spawned in container
      condition: >
        spawned_process and container and
        proc.name in (bash, sh, zsh)
      output: >
        Shell spawned in container
        (user=%user.name proc=%proc.name pod=%k8s.pod.name)
      priority: WARNING
      tags: [cks, shell]
    
    - rule: Sensitive file read
      desc: Detect reading sensitive files
      condition: >
        open_read and container and
        fd.name in (/etc/passwd, /etc/shadow, /etc/hosts)
      output: >
        Sensitive file read in container
        (file=%fd.name proc=%proc.name pod=%k8s.pod.name)
      priority: HIGH
      tags: [cks, file-access]

---
# Container immutability with Kyverno
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: enforce-immutable-containers
spec:
  validationFailureAction: Enforce
  rules:
    - name: readonly-rootfs
      match:
        any:
          - resources:
              kinds: [Pod]
      validate:
        message: "readOnlyRootFilesystem must be true"
        pattern:
          spec:
            containers:
              - securityContext:
                  readOnlyRootFilesystem: true
```

---

## Step 648: CKS Quick Reference

```bash
# ===== CKS EXAM QUICK COMMANDS =====

# RBAC imperative
kubectl create role NAME --verb=get,list --resource=pods -n NAMESPACE
kubectl create rolebinding NAME --role=ROLE --user=USER -n NAMESPACE
kubectl create clusterrole NAME --verb=get,list --resource=pods
kubectl auth can-i get pods --as=USER -n NAMESPACE

# Pod Security Standards
kubectl label ns NAMESPACE \
  pod-security.kubernetes.io/enforce=restricted --overwrite

# Secrets
kubectl create secret generic NAME --from-literal=KEY=VALUE
kubectl get secret NAME -o jsonpath='{.data.KEY}' | base64 -d

# AppArmor
apparmor_parser -q /etc/apparmor.d/PROFILE
aa-status | grep PROFILE

# Trivy
trivy image IMAGE:TAG
trivy image --severity CRITICAL IMAGE:TAG
trivy fs /path/to/dir

# kube-bench
kube-bench run --targets master
kube-bench run --targets node

# Falco
# Config: /etc/falco/falco.yaml
# Custom rules: /etc/falco/falco_rules.local.yaml
# Restart: systemctl restart falco

# Security Context key settings:
# runAsNonRoot: true
# readOnlyRootFilesystem: true
# allowPrivilegeEscalation: false
# capabilities.drop: [ALL]
```

---

## Step 649: CKS Practice Scenarios

```yaml
# SCENARIO: Harden existing deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hardened-app
spec:
  template:
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        seccompProfile:
          type: RuntimeDefault
      containers:
        - name: app
          image: nginx:alpine
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop: [ALL]
          resources:
            limits:
              cpu: 200m
              memory: 128Mi
            requests:
              cpu: 100m
              memory: 64Mi
          volumeMounts:
            - name: tmp
              mountPath: /tmp
            - name: cache
              mountPath: /var/cache/nginx
      volumes:
        - name: tmp
          emptyDir: {}
        - name: cache
          emptyDir: {}
```

---

## Step 650: Workshop - CKS Mock Exam Checklist

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: cks-mock-exam
  namespace: security
data:
  exam-topics.yaml: |
    cluster_setup:
      - Create NetworkPolicy to isolate pods
      - Configure Ingress with TLS
      - Set up CIS benchmark scan
      - Fix API server audit logging
    
    cluster_hardening:
      - Create least-privilege RBAC
      - Disable SA auto-mount
      - Upgrade cluster with kubeadm
      - Apply audit policy
    
    system_hardening:
      - Apply AppArmor profile to pod
      - Use RuntimeDefault seccomp
      - Remove unnecessary capabilities
      - Set read-only filesystem
    
    microservice_vulnerabilities:
      - Apply OPA/Gatekeeper constraint
      - Enforce PSS restricted on namespace
      - Enable Istio mTLS STRICT
      - Create minimal secret + use in pod
    
    supply_chain:
      - Scan image with Trivy
      - Sign image with Cosign
      - Configure ImagePolicyWebhook
      - Run kubesec against manifest
    
    monitoring_runtime:
      - Write Falco custom rule
      - Configure audit policy
      - Detect anomaly in Falco logs
      - Enforce readOnlyRootFilesystem
    
    time_management:
      - Easy questions: 2-3 min each
      - Medium questions: 5-7 min each
      - Hard questions: 10-15 min (do last)
      - Always verify with kubectl get/describe
```

---

## 📊 สรุป Part 68

| CKS Domain | Weight | Key Topics |
|------------|--------|------------|
| Cluster Setup | 15% | NetworkPolicy, CIS, Ingress TLS |
| Cluster Hardening | 15% | RBAC, SA, Audit, Upgrade |
| System Hardening | 20% | AppArmor, Seccomp, Capabilities |
| Microservice Vulns | 20% | OPA, PSS, mTLS, Secrets |
| Supply Chain | 20% | Trivy, Cosign, ImagePolicy |
| Monitoring/Runtime | 10% | Falco, Immutability |

---

## 🔗 ต่อไป
- [Part 69: Advanced YAML Features](./part-69-advanced-yaml.md)

---
*Part 68 | Steps 641-650 | CKS Exam Preparation | Educational Use Only*
