# Part 91: Kubernetes Security Advanced
## Steps 871-880: RBAC Deep Dive, Pod Security, Supply Chain, Secrets Management

---

## Step 871: Security Layers Overview

```
Kubernetes Security: Defense in Depth

Layer 1: Infrastructure
  - Node OS hardening (CIS Benchmark)
  - Encrypted etcd (AES-256)
  - Private API server endpoint
  - VPC network isolation

Layer 2: Cluster Access
  - RBAC: least-privilege roles
  - OIDC authentication (Dex, Okta, Google)
  - Audit logging (all API calls)
  - Network policies (zero-trust)

Layer 3: Workload
  - Pod Security Admission (PSA)
  - SecurityContext (non-root, read-only fs)
  - AppArmor / Seccomp profiles
  - Resource quotas + LimitRanges

Layer 4: Supply Chain
  - Image scanning (Trivy, Snyk)
  - Image signing (Cosign + Sigstore)
  - Attestations (SLSA provenance)
  - Admission: only signed images allowed

Layer 5: Runtime
  - Falco: runtime threat detection
  - eBPF-based monitoring
  - Audit log alerts
  - Network traffic anomaly detection

Security Posture Tools:
  kube-bench: CIS benchmark checks
  Polaris: workload best practices
  Trivy: vulnerability scanning
  Falco: runtime security
  OPA/Kyverno: policy enforcement
```

---

## Step 872: RBAC Deep Dive

```yaml
# Minimal RBAC: read-only for developers
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer-readonly
  namespace: production
rules:
  - apiGroups: [""]
    resources: [pods, services, configmaps, endpoints]
    verbs: [get, list, watch]
  - apiGroups: [""]
    resources: [pods/log]
    verbs: [get]
  - apiGroups: [apps]
    resources: [deployments, replicasets, statefulsets]
    verbs: [get, list, watch]
  - apiGroups: [autoscaling]
    resources: [horizontalpodautoscalers]
    verbs: [get, list, watch]

---
# ClusterRole: cross-namespace read
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: namespace-reader
rules:
  - apiGroups: [""]
    resources: [namespaces, nodes]
    verbs: [get, list, watch]
  - apiGroups: [metrics.k8s.io]
    resources: [nodes, pods]
    verbs: [get, list]

---
# Aggregate ClusterRoles
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: monitoring-view
  labels:
    rbac.authorization.k8s.io/aggregate-to-view: "true"
rules:
  - apiGroups: [monitoring.coreos.com]
    resources: [prometheusrules, servicemonitors]
    verbs: [get, list, watch]

---
# OIDC: map group to ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: platform-admins
subjects:
  - kind: Group
    name: platform-engineers@mycompany.com
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
```

---

## Step 873: Pod Security Admission

```yaml
# Namespace labels: enforce Pod Security Standards
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: v1.28
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/warn-version: v1.28
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/audit-version: v1.28

---
# Restricted security context (PSA compliant)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: secure-app
  namespace: production
spec:
  template:
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        runAsGroup: 1000
        fsGroup: 1000
        seccompProfile:
          type: RuntimeDefault
      containers:
        - name: app
          image: myapp:1.0
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop: [ALL]
              add: []
          volumeMounts:
            - name: tmp
              mountPath: /tmp
            - name: app-data
              mountPath: /app/data
      volumes:
        - name: tmp
          emptyDir: {}
        - name: app-data
          emptyDir: {}
```

---

## Step 874: Network Policies Zero-Trust

```yaml
# Default deny all ingress and egress
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress

---
# Allow specific ingress from frontend
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-api-ingress
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: api-service
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: production
          podSelector:
            matchLabels:
              app: frontend
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: ingress-nginx
      ports:
        - port: 8080

---
# Allow egress with DNS + database + external HTTPS
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-api-egress
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: api-service
  policyTypes:
    - Egress
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
      ports:
        - port: 53
          protocol: UDP
        - port: 53
          protocol: TCP
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: databases
      ports:
        - port: 5432
    - to:
        - ipBlock:
            cidr: 0.0.0.0/0
            except:
              - 10.0.0.0/8
              - 172.16.0.0/12
              - 192.168.0.0/16
      ports:
        - port: 443
```

---

## Step 875: Supply Chain Security

```yaml
# Kyverno: verify Cosign signature
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: verify-image-signatures
spec:
  validationFailureAction: Enforce
  background: false
  rules:
    - name: verify-signature
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
                    subject: "https://github.com/myorg/*/.github/workflows/*.yml@refs/heads/main"
                    issuer: "https://token.actions.githubusercontent.com"
                    rekor:
                      url: https://rekor.sigstore.dev
          attestations:
            - predicateType: https://slsa.dev/provenance/v0.2
              conditions:
                - all:
                    - key: "{{ buildType }}"
                      operator: Equals
                      value: "https://github.com/slsa-framework/slsa-github-generator/generic@v1"
```

---

## Step 876: Secrets Management

```yaml
# External Secrets: sync from Vault
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: vault-backend
  namespace: production
spec:
  provider:
    vault:
      server: https://vault.mycompany.com
      path: secret
      version: v2
      auth:
        kubernetes:
          mountPath: kubernetes
          role: production-reader
          serviceAccountRef:
            name: external-secrets-sa

---
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: app-secrets
  namespace: production
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-backend
    kind: SecretStore
  target:
    name: app-secrets
    creationPolicy: Owner
    template:
      type: Opaque
      data:
        DATABASE_URL: "{{ .db_url }}"
        API_KEY: "{{ .api_key }}"
  dataFrom:
    - extract:
        key: secret/production/myapp

---
# AWS Secrets Manager
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: aws-secretsmanager
  namespace: production
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-1
      auth:
        jwt:
          serviceAccountRef:
            name: external-secrets-sa
```

---

## Step 877: Falco Runtime Security

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: falco-custom-rules
  namespace: falco
data:
  custom-rules.yaml: |
    - rule: Shell in container
      desc: Detect shell execution in a container
      condition: >
        spawned_process and container and
        (proc.name = bash or proc.name = sh or proc.name = zsh) and
        not proc.pname in (entrypoint, docker-init)
      output: >
        Shell spawned in container
        (user=%user.name container=%container.name
         image=%container.image.repository:%container.image.tag
         command=%proc.cmdline)
      priority: WARNING
      tags: [container, shell]
    
    - rule: Read sensitive file
      desc: Attempt to read sensitive files
      condition: >
        open_read and
        (fd.name startswith /etc/shadow or
         fd.name startswith /root/.ssh) and
        not proc.name in (sshd, login, sudo)
      output: >
        Sensitive file read
        (user=%user.name file=%fd.name process=%proc.name)
      priority: ERROR
      tags: [filesystem, sensitive]
```

---

## Step 878: Audit Logging

```yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  - level: RequestResponse
    resources:
      - group: ""
        resources: [secrets]
  
  - level: Request
    verbs: [create, update, patch]
    resources:
      - group: apps
        resources: [deployments, daemonsets, statefulsets]
    omitStages: [RequestReceived]
  
  - level: Metadata
    users: [system:masters]
  
  - level: None
    verbs: [get, list, watch]
    resources:
      - group: ""
        resources: [configmaps, endpoints, events]
  
  - level: Metadata
    omitStages: [RequestReceived]
```

---

## Step 879: CIS Benchmark Compliance

```yaml
# Polaris: workload best practices
apiVersion: v1
kind: ConfigMap
metadata:
  name: polaris-config
  namespace: polaris
data:
  config.yaml: |
    checks:
      hostIPCSet: error
      hostPIDSet: error
      hostNetworkSet: warning
      privilegeEscalationAllowed: error
      runAsRootAllowed: warning
      runAsPrivileged: error
      cpuRequestsMissing: warning
      cpuLimitsMissing: warning
      memoryRequestsMissing: warning
      memoryLimitsMissing: error
      tagNotSpecified: error
      pullPolicyNotAlways: warning
      readinessProbeMissing: warning
      livenessProbeMissing: warning
```

---

## Step 880: Workshop - Security Monitoring

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: security-alerts
  namespace: monitoring
spec:
  groups:
    - name: security
      rules:
        - alert: FalcoSecurityEvent
          expr: |
            rate(falco_events{priority=~"Error|Critical"}[5m]) > 0
          for: 0m
          annotations:
            summary: "Falco detected high-severity event in {{ $labels.namespace }}"
          labels:
            severity: critical

        - alert: RBACPermissionEscalation
          expr: |
            rate(apiserver_audit_event_total{
              verb=~"create|update|patch",
              objectRef_resource=~"clusterrolebindings|rolebindings"
            }[5m]) > 1
          for: 2m
          annotations:
            summary: "High rate of RBAC binding changes detected"
          labels:
            severity: warning

        - alert: SecretMassAccess
          expr: |
            rate(apiserver_audit_event_total{
              verb="list",
              objectRef_resource="secrets"
            }[5m]) > 5
          for: 1m
          annotations:
            summary: "Mass secret list operations detected"
          labels:
            severity: critical

---
# เนื้อหาด้าน security ในหลักสูตรนี้มีไว้เพื่อการศึกษาและการทดสอบระบบที่ได้รับอนุญาตเท่านั้น
# การนำไปใช้โจมตีระบบที่ไม่ได้รับอนุญาตเป็นสิ่งผิดกฎหมาย
```

---

## 📊 สรุป Part 91

| Security Layer | Tool | Config |
|----------------|------|--------|
| Access control | RBAC + OIDC | Role/ClusterRole + RoleBinding |
| Pod security | Pod Security Admission | Namespace labels |
| Network | NetworkPolicy | Zero-trust default-deny |
| Supply chain | Cosign + Kyverno | keyless signing + verify |
| Secrets | External Secrets | Vault/AWS SM sync |
| Runtime | Falco | Rules + Sidekick alerts |
| Compliance | kube-bench + Polaris | CIS + workload checks |

---

## 🔗 ต่อไป
- [Part 92: Observability and SRE Practices](./part-92-observability-sre.md)

---
*Part 91 | Steps 871-880 | Kubernetes Security Advanced | Educational Content*
