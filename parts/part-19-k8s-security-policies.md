# Part 19: Kubernetes Security Policies
## Steps 181-190: Pod Security, Network Policies, และ Security Hardening

---

## 📖 บทนำ

⚠️ **คำเตือน**: เนื้อหาด้าน security ในส่วนนี้มีไว้เพื่อการศึกษาและการ hardening ระบบที่ได้รับอนุญาตเท่านั้น

Kubernetes Security มีหลายชั้น ตั้งแต่ Node level, Pod level, Network level, ไปจนถึง Application level

### Defense in Depth
```
┌─────────────────────────────────────────────┐
│              Cloud Provider Security         │
│  ┌───────────────────────────────────────┐  │
│  │            Cluster Security           │  │
│  │  ┌───────────────────────────────┐  │  │
│  │  │        Namespace Security       │  │  │
│  │  │  ┌────────────────────────┐ │  │  │
│  │  │  │      Pod Security         │ │  │  │
│  │  │  │  ┌────────────────────┐ │ │  │  │
│  │  │  │  │  Container Security  │ │ │  │  │
│  │  │  │  └────────────────────┘ │ │  │  │
│  │  │  └────────────────────────┘ │  │  │
│  │  └───────────────────────────────┘  │  │
│  └───────────────────────────────────────┘  │
└─────────────────────────────────────────────┘
```

---

## Step 181: Pod Security Standards (PSS)

```yaml
# Apply PSS ด้วย namespace labels
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
# Pod ที่ผ่าน restricted policy
apiVersion: v1
kind: Pod
metadata:
  name: secure-app
  namespace: production
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
    seccompProfile:
      type: RuntimeDefault
  containers:
    - name: app
      image: myapp:1.0.0
      securityContext:
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        capabilities:
          drop:
            - ALL
      resources:
        requests:
          cpu: 100m
          memory: 128Mi
        limits:
          cpu: 500m
          memory: 512Mi
      volumeMounts:
        - name: tmp
          mountPath: /tmp
  volumes:
    - name: tmp
      emptyDir: {}
  automountServiceAccountToken: false
```

---

## Step 182: Network Policies

```yaml
# Default deny all - สำคัญมาก!
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
# อนุญาต DNS resolution
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - ports:
        - port: 53
          protocol: UDP
        - port: 53
          protocol: TCP
      to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system

---
# อนุญาต frontend → backend
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend
      ports:
        - protocol: TCP
          port: 8080

---
# อนุญาต monitoring scrape
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-prometheus-scrape
  namespace: production
spec:
  podSelector:
    matchLabels:
      prometheus.io/scrape: "true"
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: monitoring
          podSelector:
            matchLabels:
              app: prometheus
      ports:
        - protocol: TCP
          port: 9090
```

---

## Step 183: OPA Gatekeeper Policies

```yaml
# ConstraintTemplate: ไม่อนุญาต latest tag
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8sdisallowedtags
spec:
  crd:
    spec:
      names:
        kind: K8sDisallowedTags
      validation:
        openAPIV3Schema:
          type: object
          properties:
            tags:
              type: array
              items:
                type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8sdisallowedtags

        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          tag := [t | t = split(container.image, ":")[1]]
          count(tag) > 0
          tag[0] == input.parameters.tags[_]
          msg := sprintf("Container '%v' uses disallowed tag '%v'", [container.name, tag[0]])
        }

        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not contains(container.image, ":")
          msg := sprintf("Container '%v' must specify image tag", [container.name])
        }

---
# Constraint: enforce no-latest policy
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sDisallowedTags
metadata:
  name: no-latest-tag
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
    namespaces:
      - production
  parameters:
    tags:
      - latest
      - ""

---
# ConstraintTemplate: resource limits required
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequiredresources
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredResources
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequiredresources

        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not container.resources.limits.cpu
          msg := sprintf("Container '%v' must set CPU limit", [container.name])
        }

        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not container.resources.limits.memory
          msg := sprintf("Container '%v' must set memory limit", [container.name])
        }
```

---

## Step 184: Security Context Deep Dive

```yaml
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
        runAsUser: 10000
        runAsGroup: 10000
        fsGroup: 10000
        fsGroupChangePolicy: OnRootMismatch
        seccompProfile:
          type: RuntimeDefault
      serviceAccountName: secure-app
      automountServiceAccountToken: false
      containers:
        - name: app
          image: myapp:1.2.3
          securityContext:
            runAsNonRoot: true
            runAsUser: 10000
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            privileged: false
            capabilities:
              drop:
                - ALL
          volumeMounts:
            - name: tmp
              mountPath: /tmp
            - name: run
              mountPath: /run
      volumes:
        - name: tmp
          emptyDir:
            medium: Memory
            sizeLimit: 64Mi
        - name: run
          emptyDir:
            medium: Memory
            sizeLimit: 10Mi
```

---

## Step 185: Seccomp Profiles

```json
// custom.json - Custom Seccomp Profile
// บันทึกไว้ที่ /var/lib/kubelet/seccomp/profiles/custom.json
{
  "defaultAction": "SCMP_ACT_ERRNO",
  "architectures": ["SCMP_ARCH_X86_64"],
  "syscalls": [
    {
      "names": [
        "accept", "bind", "brk", "close", "connect",
        "execve", "exit", "exit_group", "futex",
        "getpid", "mmap", "mprotect", "munmap",
        "nanosleep", "open", "openat", "poll",
        "read", "socket", "stat", "write"
      ],
      "action": "SCMP_ACT_ALLOW"
    }
  ]
}
```

```yaml
# ใช้ custom seccomp profile
apiVersion: v1
kind: Pod
metadata:
  name: custom-seccomp-pod
spec:
  securityContext:
    seccompProfile:
      type: Localhost
      localhostProfile: profiles/custom.json
  containers:
    - name: app
      image: myapp:1.0.0
```

---

## Step 186: LimitRange และ ResourceQuota

```yaml
# LimitRange - กำหนด default + max limits ต่อ namespace
apiVersion: v1
kind: LimitRange
metadata:
  name: production-limits
  namespace: production
spec:
  limits:
    - type: Container
      default:
        cpu: 500m
        memory: 512Mi
      defaultRequest:
        cpu: 100m
        memory: 128Mi
      max:
        cpu: "4"
        memory: 4Gi
      min:
        cpu: 50m
        memory: 64Mi
    - type: Pod
      max:
        cpu: "8"
        memory: 8Gi
    - type: PersistentVolumeClaim
      max:
        storage: 100Gi
      min:
        storage: 1Gi

---
# ResourceQuota - กำหนด quota ต่อ namespace
apiVersion: v1
kind: ResourceQuota
metadata:
  name: production-quota
  namespace: production
spec:
  hard:
    requests.cpu: "20"
    requests.memory: 40Gi
    limits.cpu: "40"
    limits.memory: 80Gi
    pods: "100"
    services: "20"
    secrets: "50"
    configmaps: "50"
    persistentvolumeclaims: "20"
    requests.storage: 500Gi
    services.loadbalancers: "2"
    services.nodeports: "0"
```

---

## Step 187: Audit Policies

```yaml
# /etc/kubernetes/audit-policy.yaml
apiVersion: audit.k8s.io/v1
kind: Policy
omitStages:
  - "RequestReceived"

rules:
  # Log ทุก request บน secrets, configmaps
  - level: Metadata
    resources:
      - group: ""
        resources: ["secrets", "configmaps"]
  
  # Log changes บน RBAC objects
  - level: RequestResponse
    verbs: ["create", "update", "patch", "delete"]
    resources:
      - group: "rbac.authorization.k8s.io"
        resources: ["roles", "clusterroles", "rolebindings", "clusterrolebindings"]
  
  # Log pod exec/attach/portforward
  - level: Metadata
    resources:
      - group: ""
        resources: ["pods/exec", "pods/attach", "pods/portforward"]
  
  # Log pod creation
  - level: Request
    verbs: ["create", "delete"]
    resources:
      - group: ""
        resources: ["pods"]
  
  # Default: log metadata only
  - level: Metadata
    omitStages:
      - RequestReceived
```

---

## Step 188: Falco - Runtime Security

```yaml
# Falco rules
- rule: Shell in Container
  desc: Detect shell spawned in container
  condition: >
    spawned_process and
    container and
    proc.name in (shell_binaries)
  output: >
    Shell spawned in container
    (user=%user.name container=%container.name
     shell=%proc.name cmdline=%proc.cmdline)
  priority: WARNING
  tags: [container, shell, T1059]

- rule: Sensitive Mount
  desc: Container mounting sensitive host directory
  condition: >
    mount and
    container and
    (evt.arg.target startswith /etc or
     evt.arg.target startswith /usr or
     evt.arg.target startswith /sys)
  output: >
    Sensitive directory mounted in container
    (dir=%evt.arg.target container=%container.name)
  priority: CRITICAL
  tags: [container, filesystem]

- rule: Crypto Mining
  desc: Detect potential crypto mining process
  condition: >
    spawned_process and
    container and
    (proc.name in (crypto_miners) or
     proc.cmdline contains "--mine")
  output: >
    Potential crypto mining (cmdline=%proc.cmdline)
  priority: CRITICAL
```

---

## Step 189: CIS Benchmark Compliance

```yaml
# kube-bench Job
apiVersion: batch/v1
kind: Job
metadata:
  name: kube-bench
  namespace: security
spec:
  template:
    spec:
      hostPID: true
      containers:
        - name: kube-bench
          image: aquasec/kube-bench:v0.7.2
          command: ["kube-bench", "run", "--targets", "master,node,etcd,policies"]
          volumeMounts:
            - name: var-lib-kubelet
              mountPath: /var/lib/kubelet
              readOnly: true
            - name: etc-kubernetes
              mountPath: /etc/kubernetes
              readOnly: true
      restartPolicy: Never
      volumes:
        - name: var-lib-kubelet
          hostPath:
            path: "/var/lib/kubelet"
        - name: etc-kubernetes
          hostPath:
            path: "/etc/kubernetes"
```

---

## Step 190: Workshop - Security Hardening Checklist

```yaml
# Security Hardening Checklist สำหรับ Production

# 1. Enable Pod Security Standards
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted

---
# 2. Default Deny Network Policies
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
  namespace: production
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]

---
# 3. Resource Quotas
apiVersion: v1
kind: ResourceQuota
metadata:
  name: namespace-quota
  namespace: production
spec:
  hard:
    requests.cpu: "10"
    requests.memory: 20Gi
    limits.cpu: "20"
    limits.memory: 40Gi
    pods: "50"

---
# 4. Disable default ServiceAccount token mounting
apiVersion: v1
kind: ServiceAccount
metadata:
  name: default
  namespace: production
automountServiceAccountToken: false

---
# 5. LimitRange
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: production
spec:
  limits:
    - type: Container
      default:
        cpu: 200m
        memory: 256Mi
      defaultRequest:
        cpu: 50m
        memory: 64Mi
      max:
        cpu: "2"
        memory: 2Gi
```

```python
#!/usr/bin/env python3
"""
Security Audit Script
ตรวจสอบ security configurations ใน Kubernetes cluster
"""
import kubernetes
import sys
from dataclasses import dataclass
from typing import List

kubernetes.config.load_kube_config()
v1 = kubernetes.client.CoreV1Api()
networking_v1 = kubernetes.client.NetworkingV1Api()


@dataclass
class Finding:
    severity: str
    resource: str
    issue: str
    remediation: str


def audit_pods(namespace: str = "production") -> List[Finding]:
    findings = []
    pods = v1.list_namespaced_pod(namespace)
    
    for pod in pods.items:
        name = pod.metadata.name
        for container in pod.spec.containers:
            if container.security_context and container.security_context.privileged:
                findings.append(Finding(
                    severity="CRITICAL",
                    resource=f"Pod/{name}/{container.name}",
                    issue="Container is privileged",
                    remediation="Set securityContext.privileged: false"
                ))
            if not container.resources or not container.resources.limits:
                findings.append(Finding(
                    severity="HIGH",
                    resource=f"Pod/{name}/{container.name}",
                    issue="No resource limits set",
                    remediation="Set resources.limits.cpu and memory"
                ))
    return findings


if __name__ == '__main__':
    namespace = sys.argv[1] if len(sys.argv) > 1 else "production"
    findings = audit_pods(namespace)
    severity_order = {"CRITICAL": 0, "HIGH": 1, "MEDIUM": 2, "LOW": 3}
    findings.sort(key=lambda f: severity_order.get(f.severity, 4))
    
    for f in findings:
        print(f"[{f.severity}] {f.resource}: {f.issue}")
    print(f"\nTotal: {len(findings)} findings")
```

---

## 📊 สรุป Part 19

| หัวข้อ | เครื่องมือ/แนวทาง |
|--------|---------------|
| Pod Security Standards | namespace labels, enforce/warn/audit |
| Network Policies | default-deny, allow specific traffic |
| OPA Gatekeeper | ConstraintTemplate, Constraint |
| Security Context | runAsNonRoot, readOnlyRootFilesystem |
| Seccomp Profiles | RuntimeDefault, Localhost custom |
| LimitRange & Quota | default limits, namespace quotas |
| Audit Policies | Log secrets, RBAC changes |
| Falco | Runtime threat detection |
| CIS Benchmark | kube-bench |

---

## 🔗 ต่อไป
- [Part 20: Kubernetes Networking Advanced](./part-20-kubernetes-networking-advanced.md)

---
*Part 19 | Steps 181-190 | ระดับสูง*
