# Part 80: Advanced Security Hardening
## Steps 761-770: Supply Chain, Runtime Security, Zero Trust

---

## Step 761: Kubernetes Security Layers

```
Defense in Depth - Security Layers:

Layer 1: Infrastructure
  - Node OS hardening (CIS benchmarks)
  - Container runtime (containerd, CRI-O)
  - Node encryption at rest
  - Private networking, no public API server

Layer 2: Cluster
  - RBAC (least privilege)
  - Pod Security Standards (restricted)
  - Network policies (default deny)
  - Audit logging
  - Secrets encryption at rest (KMS)

Layer 3: Workload
  - Non-root containers
  - Read-only root filesystem
  - No privileged containers
  - Resource limits
  - seccomp/AppArmor profiles

Layer 4: Application
  - mTLS (service mesh)
  - JWT validation
  - Input validation
  - OWASP Top 10 mitigations

Layer 5: Supply Chain
  - Image scanning (Trivy)
  - Image signing (Cosign/Notation)
  - SBOM generation
  - Policy enforcement (OPA/Kyverno)
  - Provenance attestation (SLSA)

> เนื้อหาด้าน security ในหลักสูตรนี้มีไว้เพื่อการศึกษาและการทดสอบระบบที่ได้รับอนุญาตเท่านั้น
```

---

## Step 762: Supply Chain Security (SLSA + Cosign)

```yaml
# Kyverno: require image signature
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-signed-images
spec:
  validationFailureAction: Enforce
  background: false
  rules:
    - name: check-image-signature
      match:
        any:
          - resources:
              kinds: [Pod]
              namespaces: [production]
      verifyImages:
        - imageReferences:
            - "ghcr.io/myorg/*"
          attestors:
            - count: 1
              entries:
                - keyless:
                    subject: "https://github.com/myorg/*"
                    issuer: "https://token.actions.githubusercontent.com"
                    rekor:
                      url: https://rekor.sigstore.dev

# CLI commands:
# Sign: cosign sign --yes ghcr.io/myorg/myapp:v1.0.0
# Verify: cosign verify \
#   --certificate-identity "https://github.com/myorg/myapp/.github/workflows/build.yml@refs/heads/main" \
#   --certificate-oidc-issuer "https://token.actions.githubusercontent.com" \
#   ghcr.io/myorg/myapp:v1.0.0
```

---

## Step 763: OPA Gatekeeper - Policy as Code

```yaml
# ConstraintTemplate: define reusable policy
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequiredsecuritycontext
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredSecurityContext
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequiredsecuritycontext

        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not container.securityContext.runAsNonRoot
          msg := sprintf("Container '%v' must set runAsNonRoot: true", [container.name])
        }

        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not container.securityContext.readOnlyRootFilesystem
          msg := sprintf("Container '%v' must use read-only root filesystem", [container.name])
        }

        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          drops := {cap | cap := container.securityContext.capabilities.drop[_]}
          not drops["ALL"]
          msg := sprintf("Container '%v' must drop ALL capabilities", [container.name])
        }

---
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredSecurityContext
metadata:
  name: require-security-context-production
spec:
  enforcementAction: deny
  match:
    kinds:
      - apiGroups: [""]
        kinds: [Pod]
    namespaces: [production, staging]
```

---

## Step 764: Secrets Management with Vault

```yaml
# SecretStore: connect to Vault
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: vault-backend
  namespace: production
spec:
  provider:
    vault:
      server: "https://vault.internal.example.com"
      path: "secret"
      version: "v2"
      auth:
        kubernetes:
          mountPath: "kubernetes"
          role: "production-reader"
          serviceAccountRef:
            name: vault-reader

---
# ExternalSecret: sync from Vault to K8s Secret
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: database-credentials
  namespace: production
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-backend
    kind: SecretStore
  target:
    name: database-credentials
    creationPolicy: Owner
  data:
    - secretKey: password
      remoteRef:
        key: production/database
        property: password
    - secretKey: username
      remoteRef:
        key: production/database
        property: username
```

---

## Step 765: Runtime Security with Falco

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: falco-custom-rules
  namespace: falco
data:
  custom-rules.yaml: |
    # Detect shell execution in container
    - rule: Shell in Container
      desc: Detect a shell being run inside a container
      condition: >
        spawned_process and container and
        proc.name in (bash, sh, zsh, dash, ash) and
        not proc.pname in (bash, sh, zsh)
      output: >
        Shell spawned in container
        (user=%user.name container=%container.name
         image=%container.image.repository:%container.image.tag
         cmd=%proc.cmdline)
      priority: WARNING
      tags: [shell, container, T1059]
    
    # Detect read of sensitive files
    - rule: Read Sensitive File Untrusted
      desc: Detect reading of sensitive system files
      condition: >
        open_read and container and
        fd.name in (/etc/shadow, /etc/sudoers, /root/.ssh/authorized_keys) and
        not proc.name in (sshd, sudo)
      output: >
        Sensitive file read (user=%user.name file=%fd.name
        container=%container.name image=%container.image.repository)
      priority: ERROR
      tags: [file, sensitive, T1003]
```

---

## Step 766: AppArmor and Seccomp Profiles

```yaml
# SeccompProfile: custom syscall filtering
apiVersion: security-profiles-operator.x-k8s.io/v1beta1
kind: SeccompProfile
metadata:
  name: nginx-restricted
  namespace: production
spec:
  defaultAction: SCMP_ACT_ERRNO
  syscalls:
    - action: SCMP_ACT_ALLOW
      names:
        - accept4
        - bind
        - brk
        - clock_gettime
        - close
        - connect
        - epoll_ctl
        - epoll_wait
        - exit_group
        - fstat
        - futex
        - listen
        - mmap
        - mprotect
        - munmap
        - open
        - openat
        - read
        - recvfrom
        - sendto
        - socket
        - stat
        - write

---
# Use seccomp profile in pod
apiVersion: v1
kind: Pod
metadata:
  name: nginx-secure
  namespace: production
  annotations:
    container.apparmor.security.beta.kubernetes.io/nginx: localhost/k8s-nginx
spec:
  securityContext:
    seccompProfile:
      type: Localhost
      localhostProfile: operator/production/nginx-restricted.json
  containers:
    - name: nginx
      image: nginx:alpine
      securityContext:
        runAsNonRoot: true
        runAsUser: 101
        readOnlyRootFilesystem: true
        allowPrivilegeEscalation: false
        capabilities:
          drop: [ALL]
```

---

## Step 767: Zero Trust with SPIFFE/SPIRE

```yaml
# ClusterSPIFFEID: assign SPIFFE ID to workloads
apiVersion: spire.spiffe.io/v1alpha1
kind: ClusterSPIFFEID
metadata:
  name: production-workloads
spec:
  spiffeIDTemplate: "spiffe://example.com/ns/{{.PodMeta.Namespace}}/sa/{{.PodSpec.ServiceAccountName}}"
  podSelector:
    matchLabels:
      spiffe-enabled: "true"

---
# JWT validation at gateway
apiVersion: security.istio.io/v1beta1
kind: RequestAuthentication
metadata:
  name: jwt-validation
  namespace: production
spec:
  selector:
    matchLabels:
      app: api-gateway
  jwtRules:
    - issuer: "https://accounts.google.com"
      jwksUri: "https://www.googleapis.com/oauth2/v3/certs"
    - issuer: "https://auth.example.com"
      jwksUri: "https://auth.example.com/.well-known/jwks.json"
      forwardOriginalToken: false

---
# AuthorizationPolicy: deny unless valid JWT
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: require-valid-jwt
  namespace: production
spec:
  selector:
    matchLabels:
      app: api-gateway
  action: DENY
  rules:
    - from:
        - source:
            notRequestPrincipals: ["*"]
```

---

## Step 768: Kubernetes Audit Policy

```yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  # Log exec/portforward (security relevant)
  - level: Request
    resources:
      - group: ""
        resources: [pods/exec, pods/portforward, pods/attach]
    namespaces: [production, staging]
  
  # Log secret access metadata only
  - level: Metadata
    resources:
      - group: ""
        resources: [secrets]
  
  # Log RBAC changes fully
  - level: RequestResponse
    resources:
      - group: rbac.authorization.k8s.io
        resources: [clusterroles, clusterrolebindings, roles, rolebindings]
  
  # Log anonymous requests
  - level: Request
    users: ["system:anonymous"]
  
  # Skip noisy read-only calls
  - level: None
    verbs: [get, list, watch]
    resources:
      - group: ""
        resources: [configmaps, pods, services]
    namespaces: [monitoring, logging]
  
  # Default: metadata
  - level: Metadata

# kube-apiserver flags:
# --audit-policy-file=/etc/kubernetes/audit-policy.yaml
# --audit-log-path=/var/log/kubernetes/audit/audit.log
# --audit-log-maxage=30
```

---

## Step 769: Image Scanning Pipeline

```yaml
# Trivy Operator: continuous scanning in cluster
# Auto-creates VulnerabilityReport per workload

# Kyverno: block images with critical CVEs
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: block-critical-vulnerabilities
spec:
  validationFailureAction: Enforce
  rules:
    - name: check-vulnerability-report
      match:
        any:
          - resources:
              kinds: [Pod]
              namespaces: [production]
      context:
        - name: criticalCount
          apiCall:
            urlPath: "/apis/aquasecurity.github.io/v1alpha1/namespaces/{{request.namespace}}/vulnerabilityreports"
            jmesPath: "items[?report.summary.criticalCount > `0`] | length(@)"
      validate:
        message: "Pod image has critical vulnerabilities. Fix CVEs before deploying to production."
        deny:
          conditions:
            all:
              - key: "{{criticalCount}}"
                operator: GreaterThan
                value: 0
```

---

## Step 770: Workshop - Security Hardening Checklist

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: security-hardening-checklist
  namespace: kube-system
data:
  checklist.yaml: |
    infrastructure:
      - CIS Kubernetes Benchmark (kube-bench score > 90%)
      - Encrypted etcd at rest (KMS provider)
      - Private API server endpoint
      - Node OS: CIS hardened image (Bottlerocket/Flatcar)
    
    cluster:
      - RBAC audit (no cluster-admin bindings for users)
      - Pod Security Standards: restricted on prod namespaces
      - NetworkPolicy: default deny all
      - Audit logging enabled
      - OPA Gatekeeper or Kyverno policies
    
    workloads:
      - All containers: runAsNonRoot: true
      - All containers: readOnlyRootFilesystem: true
      - All containers: drop ALL capabilities
      - All containers: allowPrivilegeEscalation: false
      - Resource requests/limits on all containers
    
    supply_chain:
      - Image scanning in CI (Trivy, Snyk)
      - Image signing (Cosign/Notation)
      - Policy: only signed images in production
      - SBOM generation per release
    
    runtime:
      - Falco DaemonSet with custom rules
      - Alerts to SIEM (Splunk, Elastic, Datadog)
      - Incident response playbook tested quarterly
    
    secrets:
      - No secrets in container images
      - No secrets in ConfigMaps
      - Vault or ESO for secret management
      - Secrets rotation policy (90 days)

---
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: security-alerts
  namespace: monitoring
spec:
  groups:
    - name: security
      rules:
        - alert: PrivilegedContainerRunning
          expr: |
            kube_pod_container_info * on (pod, namespace) group_left()
            kube_pod_spec_containers_security_context_privileged > 0
          for: 1m
          labels:
            severity: critical
          annotations:
            summary: "Privileged container running in {{ $labels.namespace }}"

        - alert: RBACClusterAdminBinding
          expr: |
            count(kube_clusterrolebinding_info{clusterrole="cluster-admin"}) > 3
          for: 1m
          labels:
            severity: warning
          annotations:
            summary: "More than 3 cluster-admin bindings detected"
```

---

## 📊 สรุป Part 80

| Layer | Tool | Purpose |
|-------|------|--------|
| Supply Chain | Cosign, Trivy | Sign + scan images |
| Policy | OPA Gatekeeper / Kyverno | Admission control |
| Secrets | Vault + ESO | Centralized secrets |
| Runtime | Falco | Threat detection |
| Network | Istio mTLS, NetworkPolicy | Zero trust |
| Profiling | Seccomp, AppArmor | Syscall/FS restriction |

---

## 🔗 ต่อไป
- [Part 81: Performance Tuning and Optimization](./part-81-performance.md)

---
*Part 80 | Steps 761-770 | Advanced Security | Educational Content*

> เนื้อหาด้าน security ในหลักสูตรนี้มีไว้เพื่อการศึกษาและการทดสอบระบบที่ได้รับอนุญาตเท่านั้น การนำไปใช้โจมตีระบบที่ไม่ได้รับอนุญาตเป็นสิ่งผิดกฎหมาย
