# Part 57: Lateral Movement & Detection
## Steps 531-540: Kubernetes Lateral Movement Techniques & Defenses

---

## 📖 บทนำ

> **⚠️ หมายเหตุ**: เนื้อหาด้าน security ในหลักสูตรนี้มีไว้เพื่อการศึกษาและการทดสอบระบบที่ได้รับอนุญาตเท่านั้น การนำไปใช้โจมตีระบบที่ไม่ได้รับอนุญาตเป็นสิ่งผิดกฎหมาย

Lateral movement ใน Kubernetes คือการที่ผู้โจมตีที่เข้าถึง pod หนึ่งแล้วพยายามขยายการเข้าถึงไปยัง pods หรือ nodes อื่นๆ

---

## Step 531: Lateral Movement Techniques Overview

```
Kubernetes Lateral Movement Paths:

1. Service Account Token Abuse
   Pod (compromised) → read /var/run/secrets/kubernetes.io/serviceaccount/token
   → kubectl --token=<SA token> get secrets -A
   → escalate via RBAC

2. Kubernetes DNS Discovery
   Pod → nslookup kubernetes.default.svc.cluster.local
   → enumerate services: <service>.<namespace>.svc.cluster.local
   → probe each service for vulnerabilities

3. Environment Variable Mining
   Pod → env | grep -E "(PASSWORD|SECRET|TOKEN|KEY)"
   → credentials to other services

4. Pod-to-Pod Direct Attack
   No NetworkPolicy → curl http://<other-pod-IP>:<port>
   → exploit vulnerable services

5. Node Pivot via hostPath
   Pod with /etc/kubernetes hostPath → read API server kubeconfig
   → admin-level access

6. Container Registry Attack
   Compromise CI/CD → inject malicious image
   → gets deployed everywhere
```

---

## Step 532: Service Account Token Detection

```yaml
# Falco rules: detect SA token abuse
- rule: Service Account Token Read
  desc: Detect read of service account token from inside container
  condition: >
    open_read and
    container and
    fd.name startswith /var/run/secrets/kubernetes.io/serviceaccount/token and
    not proc.name in (python, python3, node, java, ruby, go)
  output: >
    Service account token read (user=%user.name container=%container.id
    image=%container.image.repository:%container.image.tag
    file=%fd.name proc=%proc.name)
  priority: WARNING
  tags: [lateral_movement, credential_access]

---
# Detect: kubectl exec into running container
- rule: Kubernetes Client Tool Launched in Container
  desc: Detect kubectl or helm executed inside container
  condition: >
    spawned_process and
    container and
    proc.name in (kubectl, helm, kubeadm, crictl)
  output: >
    Kubernetes client tool executed in container
    (proc=%proc.name container=%container.id
    image=%container.image.repository:%container.image.tag
    cmdline=%proc.cmdline)
  priority: CRITICAL
  tags: [lateral_movement, execution]

---
# Audit policy: log SA token access
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  - level: Metadata
    verbs: ["get", "list", "watch"]
    resources:
      - group: ""
        resources: ["secrets"]
  
  - level: RequestResponse
    verbs: ["create"]
    resources:
      - group: ""
        resources: ["pods/exec", "pods/attach", "pods/portforward"]
  
  - level: RequestResponse
    verbs: ["create", "update", "patch", "delete"]
    resources:
      - group: "rbac.authorization.k8s.io"
        resources: ["clusterroles", "clusterrolebindings", "roles", "rolebindings"]
```

---

## Step 533: Network Reconnaissance Detection

```yaml
# Prometheus: track lateral movement metrics
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: lateral-movement-alerts
  namespace: monitoring
spec:
  groups:
    - name: lateral-movement
      interval: 30s
      rules:
        - alert: PodExecSessionStarted
          expr: |
            increase(apiserver_audit_event_total{
              verb="create",
              resource="pods/exec"
            }[5m]) > 0
          annotations:
            summary: "kubectl exec session started"
            description: "Pod exec session - verify this is authorized"
          labels:
            severity: warning
        
        - alert: SecretAccessSpike
          expr: |
            increase(apiserver_audit_event_total{
              verb=~"get|list",
              resource="secrets"
            }[5m]) > 20
          annotations:
            summary: "Unusual spike in secret access"
          labels:
            severity: critical
```

---

## Step 534: Preventing Lateral Movement

```yaml
# ValidatingAdmissionPolicy: deny exec in production
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicy
metadata:
  name: deny-pod-exec-production
spec:
  failurePolicy: Fail
  matchConstraints:
    resourceRules:
      - apiGroups: [""]
        apiVersions: ["v1"]
        operations: ["CREATE"]
        resources: ["pods/exec", "pods/attach"]
  validations:
    - expression: |
        object.metadata.namespace != "production"
      message: "exec/attach not allowed in production namespace"

---
# OPA Gatekeeper: restrict exec to specific users
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: restrictpodexec
spec:
  crd:
    spec:
      names:
        kind: RestrictPodExec
      validation:
        openAPIV3Schema:
          properties:
            allowedUsers:
              type: array
              items:
                type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package restrictpodexec
        
        violation[{"msg": msg}] {
          input.review.kind.kind == "PodExecOptions"
          username := input.review.userInfo.username
          not username in input.parameters.allowedUsers
          msg := sprintf("User %v is not allowed to exec into pods", [username])
        }
```

---

## Step 535: Container Image Security

```yaml
# Kyverno: image provenance verification
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: verify-image-provenance
spec:
  validationFailureAction: Enforce
  rules:
    - name: verify-image-signature
      match:
        any:
          - resources:
              kinds: [Pod]
              namespaces: [production]
      verifyImages:
        - imageReferences:
            - "myregistry.io/myapp/*"
          attestors:
            - count: 1
              entries:
                - keyless:
                    subject: "https://github.com/myorg/myapp/.github/workflows/build.yml@refs/heads/main"
                    issuer: "https://token.actions.githubusercontent.com"
                    rekor:
                      url: https://rekor.sigstore.dev

---
# Kyverno: enforce imagePullPolicy Always in production
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-image-pull-always
spec:
  validationFailureAction: Enforce
  rules:
    - name: require-always
      match:
        any:
          - resources:
              kinds: [Pod]
              namespaces: [production]
      validate:
        message: "imagePullPolicy must be Always in production"
        pattern:
          spec:
            containers:
              - imagePullPolicy: Always
```

---

## Step 536: Service Mesh for Lateral Movement Control

```yaml
# Istio: deny all by default, allow only needed paths
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: deny-all
  namespace: production
spec: {}

---
# Allow only frontend → backend
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: production
spec:
  selector:
    matchLabels:
      app: backend
  action: ALLOW
  rules:
    - from:
        - source:
            principals:
              - "cluster.local/ns/production/sa/frontend-sa"
      to:
        - operation:
            methods: ["GET", "POST"]
            paths: ["/api/v1/*"]

---
# DestinationRule: enforce mTLS
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: production-mtls
  namespace: production
spec:
  host: "*.production.svc.cluster.local"
  trafficPolicy:
    tls:
      mode: ISTIO_MUTUAL
```

---

## Step 537: Node Isolation

```yaml
# Kyverno: block dangerous hostPath mounts
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: block-lateral-movement-paths
spec:
  validationFailureAction: Enforce
  rules:
    - name: no-sensitive-hostpaths
      match:
        any:
          - resources:
              kinds: [Pod]
      validate:
        message: "Mounting sensitive host paths enables lateral movement"
        deny:
          conditions:
            any:
              - key: "{{ request.object.spec.volumes[].hostPath.path | to_array(@) }}"
                operator: AnyIn
                value:
                  - "/etc/kubernetes"
                  - "/var/lib/kubelet"
                  - "/var/lib/etcd"
                  - "/root/.kube"
                  - "/proc"
                  - "/sys"
```

---

## Step 538: Runtime Detection with eBPF

```yaml
# Tetragon: eBPF-based runtime security
apiVersion: cilium.io/v1alpha1
kind: TracingPolicy
metadata:
  name: detect-lateral-movement
spec:
  kprobes:
    - call: "security_file_open"
      syscall: false
      args:
        - index: 0
          type: "file"
      selectors:
        - matchArgs:
            - index: 0
              operator: "Prefix"
              values:
                - "/etc/kubernetes/"
                - "/var/lib/kubelet/pods/"
          matchNamespaces:
            - namespace: Mnt
              operator: NotIn
              values:
                - "host"
          matchActions:
            - action: Sigkill
```

---

## Step 539: Honeypots in Kubernetes

```yaml
# Honeypot: fake secret to detect token theft
apiVersion: v1
kind: Secret
metadata:
  name: backup-credentials
  namespace: production
  annotations:
    description: "Canary - access triggers alert"
type: Opaque
stringData:
  access-key-id: "AKIAIOSFODNN7EXAMPLE"

---
# Falco rule: alert when honeypot secret is accessed
- rule: Honeypot Secret Accessed
  desc: Alert when canary secret is read - indicates compromise
  condition: >
    ka.verb = get and
    ka.target.resource = secrets and
    ka.target.name = backup-credentials and
    ka.target.namespace = production
  output: >
    CRITICAL: Honeypot secret accessed! Possible compromise!
    (user=%ka.user.name ip=%ka.source.ip)
  priority: CRITICAL
  tags: [honeypot, lateral_movement]
```

---

## Step 540: Workshop - Lateral Movement Simulation Lab

```yaml
# Lab scenario: trace lateral movement path (authorized testing only)
# SETUP:
# 1. Deploy vulnerable app in "lab" namespace
# 2. NetworkPolicy: allow all in lab namespace
# 3. Audit logging: enabled
# 4. Falco: running with k8s audit rules
#
# ATTACK PATH (for detection testing, NOT production):
# 1. Gain pod shell (simulated)
# 2. cat /var/run/secrets/kubernetes.io/serviceaccount/token
# 3. env | grep KUBERNETES
# 4. kubectl --token=<token> get ns
#
# DETECTION EXPECTED:
# - Falco: "Kubernetes Client Tool Launched in Container"
# - Audit log: SA token used from unexpected IP
# - Prometheus: SecretAccessSpike alert

# NetworkPolicy for lab (isolated from production)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: lab-isolation
  namespace: lab
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: lab
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: lab
    - ports:
        - port: 53
          protocol: UDP
```

---

## 📊 สรุป Part 57

| Lateral Movement Vector | Detection | Prevention |
|------------------------|-----------|------------|
| SA token abuse | Falco + Audit logs | RBAC least privilege |
| Pod-to-pod attack | Falco network rules | NetworkPolicy |
| kubectl in container | Falco process rules | ValidatingAdmissionPolicy |
| hostPath pivot | Tetragon eBPF | Kyverno policy |
| Image replacement | Image signing | Kyverno verify |
| DNS enumeration | Falco DNS rules | CoreDNS filter |

---

## 🔗 ต่อไป
- [Part 58: Persistence Mechanisms & Detection](./part-58-persistence.md)

---
*Part 57 | Steps 531-540 | ระดับผู้เชี่ยวชาญ | Educational Use Only*
