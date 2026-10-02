# Part 52: Container Escape Techniques & Defenses
## Steps 491-500: Understanding Container Breakout

---

## 📖 บทนำ

> **⚠️ คำเตือนสำคัญ**: เนื้อหาด้าน security ในหลักสูตรนี้มีไว้เพื่อการศึกษาและการทดสอบระบบที่ได้รับอนุญาตเท่านั้น การนำไปใช้โจมตีระบบที่ไม่ได้รับอนุญาตเป็นสิ่งผิดกฎหมาย

Container escape เป็นเทคนิคที่ attacker ใช้ break out จาก container ไปยัง host system การทำความเข้าใจ attack vectors ช่วยให้ defender สามารถ hardening ได้ถูกจุด

---

## Step 491: Container Security Model

```
Container Isolation Layers:

┌─────────────────────────────────────────────────────────────┐
│                    Container Process                        │
│  - PID namespace isolation                                  │
│  - Network namespace isolation                              │
│  - Mount namespace isolation                                │
│  - User namespace isolation                                 │
│  - Cgroups (resource limits)                               │
├─────────────────────────────────────────────────────────────┤
│                    Linux Kernel                             │
│  - seccomp (syscall filtering)                             │
│  - AppArmor/SELinux (MAC)                                  │
│  - Capabilities (privilege control)                        │
├─────────────────────────────────────────────────────────────┤
│                    Container Runtime                        │
│  - runc (default)                                          │
│  - gVisor (sandbox)                                        │
│  - Kata Containers (VM-based)                              │
└─────────────────────────────────────────────────────────────┘

Attack Surfaces:
  1. Privileged container
  2. Dangerous capabilities (SYS_ADMIN, SYS_PTRACE)
  3. Host path mounts
  4. Host network/PID namespace
  5. Kernel exploits
  6. Container runtime vulnerabilities
```

---

## Step 492: Privileged Container Defense

```yaml
# ห้าม privileged container ด้วย ValidatingAdmissionPolicy
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicy
metadata:
  name: no-privileged-containers
spec:
  failurePolicy: Fail
  matchConstraints:
    resourceRules:
      - apiGroups: [""]
        apiVersions: ["v1"]
        operations: ["CREATE", "UPDATE"]
        resources: ["pods"]
  validations:
    - expression: |
        !has(object.spec.containers) ||
        object.spec.containers.all(c,
          !has(c.securityContext) ||
          !has(c.securityContext.privileged) ||
          c.securityContext.privileged == false
        )
      message: "Privileged containers are not allowed"

---
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicyBinding
metadata:
  name: no-privileged-containers-binding
spec:
  policyName: no-privileged-containers
  validationActions: [Deny]
  matchResources:
    namespaceSelector:
      matchExpressions:
        - key: kubernetes.io/metadata.name
          operator: NotIn
          values: [kube-system]

# Falco detection:
# - rule: Privileged Container Launch
#   condition: container.privileged=true and evt.type=container
#   output: Privileged container started (%container.name)
#   priority: WARNING
```

---

## Step 493: Host Path Mount Restrictions

```yaml
# Kyverno: ห้าม sensitive host paths
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: restrict-host-paths
spec:
  validationFailureAction: Enforce
  rules:
    - name: no-sensitive-host-paths
      match:
        any:
          - resources:
              kinds: [Pod]
      validate:
        message: "Sensitive host paths are not allowed"
        deny:
          conditions:
            any:
              - key: "{{ request.object.spec.volumes[].hostPath.path | to_array(@) | not_null(@) }}"
                operator: AnyIn
                value:
                  - "/"
                  - "/proc"
                  - "/sys"
                  - "/dev"
                  - "/var/run/docker.sock"
                  - "/var/run/containerd/containerd.sock"
                  - "/var/lib/kubelet"
                  - "/etc/kubernetes"

# Dangerous paths and why:
# /                     -> read/write any file on host
# /var/run/docker.sock  -> create privileged containers
# /var/lib/kubelet      -> steal node credentials
# /etc/kubernetes       -> steal cluster admin certs
```

---

## Step 494: Capability Restrictions

```yaml
# Secure Pod: drop ALL capabilities
apiVersion: v1
kind: Pod
metadata:
  name: secure-app
  namespace: production
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 10001
    seccompProfile:
      type: RuntimeDefault
  containers:
    - name: app
      image: myregistry.io/my-app:1.0
      securityContext:
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        capabilities:
          drop: [ALL]
          add: []  # Add only specific ones if required
      volumeMounts:
        - name: tmp
          mountPath: /tmp
  volumes:
    - name: tmp
      emptyDir:
        medium: Memory
        sizeLimit: 100Mi

---
# ValidatingAdmissionPolicy: deny dangerous capabilities
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicy
metadata:
  name: deny-dangerous-capabilities
spec:
  failurePolicy: Fail
  matchConstraints:
    resourceRules:
      - apiGroups: [""]
        apiVersions: ["v1"]
        operations: ["CREATE", "UPDATE"]
        resources: ["pods"]
  validations:
    - expression: |
        object.spec.containers.all(c,
          !has(c.securityContext) ||
          !has(c.securityContext.capabilities) ||
          !has(c.securityContext.capabilities.add) ||
          !c.securityContext.capabilities.add.exists(cap,
            cap in ["SYS_ADMIN", "SYS_PTRACE", "SYS_MODULE",
                    "NET_ADMIN", "DAC_READ_SEARCH", "DAC_OVERRIDE"]
          )
        )
      message: "Dangerous capabilities are not allowed"
```

---

## Step 495: Seccomp Profile Defense

```yaml
# Security Profiles Operator SeccompProfile
apiVersion: security-profiles-operator.x-k8s.io/v1beta1
kind: SeccompProfile
metadata:
  name: app-seccomp
  namespace: production
spec:
  defaultAction: SCMP_ACT_ERRNO
  syscalls:
    - action: SCMP_ACT_ALLOW
      names:
        - accept4
        - bind
        - connect
        - epoll_pwait
        - execve
        - exit_group
        - futex
        - getcwd
        - getdents64
        - mmap
        - mprotect
        - munmap
        - open
        - openat
        - read
        - rt_sigaction
        - socket
        - stat
        - write

# AppArmor (K8s 1.30+ native field):
# securityContext:
#   appArmorProfile:
#     type: Localhost
#     localhostProfile: k8s-my-app
```

---

## Step 496: Falco Runtime Detection

```yaml
# Falco custom rules ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: falco-custom-rules
  namespace: falco-system
data:
  custom_rules.yaml: |
    - rule: Terminal Shell in Container
      desc: A shell was spawned in a container with TTY
      condition: >
        spawned_process and container and shell_procs and proc.tty != 0
      output: >
        Shell in container (user=%user.name container=%container.name
        image=%container.image.repository shell=%proc.name)
      priority: WARNING
    
    - rule: Write to sensitive host path
      desc: Write to sensitive system files
      condition: >
        open_write and
        (fd.name startswith /etc/cron.d or
         fd.name startswith /etc/sudoers or
         fd.name = /etc/passwd or
         fd.name = /etc/shadow)
      output: >
        Write to sensitive path (file=%fd.name container=%container.name)
      priority: CRITICAL
    
    - rule: Container namespace escape attempt
      desc: Detect namespace escape via unshare
      condition: syscall.type = unshare and container
      output: >
        Namespace escape attempt (container=%container.name user=%user.name)
      priority: CRITICAL
    
    - rule: Unexpected outbound connection
      desc: Container making unexpected outbound connection
      condition: >
        outbound and container and
        not fd.sip.name in (allowed_outbound_hosts)
      output: >
        Unexpected outbound connection (container=%container.name dest=%fd.rip)
      priority: WARNING
```

---

## Step 497: gVisor Sandbox Runtime

```yaml
# RuntimeClass for gVisor
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: gvisor
handler: runsc

---
# Pod using gVisor sandbox
apiVersion: v1
kind: Pod
metadata:
  name: sandboxed-app
  namespace: untrusted-workloads
spec:
  runtimeClassName: gvisor
  containers:
    - name: app
      image: myregistry.io/my-app:1.0
      securityContext:
        runAsNonRoot: true
        runAsUser: 10001
        allowPrivilegeEscalation: false
        capabilities:
          drop: [ALL]

---
# Kata Containers (VM-based isolation)
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: kata-containers
handler: kata

---
# Require sandbox runtime in untrusted namespace
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-sandbox-runtime
spec:
  validationFailureAction: Enforce
  rules:
    - name: require-runtime-class
      match:
        any:
          - resources:
              kinds: [Pod]
              namespaces: [untrusted-workloads]
      validate:
        message: "Must use gvisor or kata runtime in untrusted-workloads"
        pattern:
          spec:
            runtimeClassName: "gvisor | kata-containers"
```

---

## Step 498: Cloud Metadata Block

```yaml
# NetworkPolicy: block cloud metadata server
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: block-cloud-metadata
  namespace: production
spec:
  podSelector: {}
  policyTypes: [Egress]
  egress:
    - ports:
        - port: 53
          protocol: UDP
        - port: 53
          protocol: TCP
    - to:
        - ipBlock:
            cidr: 0.0.0.0/0
            except:
              - 169.254.169.254/32  # AWS/GCP/Azure metadata
              - 169.254.170.2/32    # ECS metadata
              - 100.100.100.200/32  # Alibaba metadata
```

---

## Step 499: Image Hardening

```yaml
# Verify distroless + no latest tag with Kyverno
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-hardened-images
spec:
  validationFailureAction: Enforce
  rules:
    - name: no-latest-tag
      match:
        any:
          - resources:
              kinds: [Pod]
              namespaces: [production]
      validate:
        message: "Image tag must not be latest"
        deny:
          conditions:
            any:
              - key: "{{ request.object.spec.containers[].image }}"
                operator: AnyIn
                value: ["*:latest", "*:*latest*"]
    
    - name: approved-registry-only
      match:
        any:
          - resources:
              kinds: [Pod]
              namespaces: [production]
      validate:
        message: "Images must be from approved registries"
        pattern:
          spec:
            containers:
              - image: "myregistry.io/* | gcr.io/distroless/*"
```

---

## Step 500: Complete Hardening Checklist

```yaml
# PSA: enforce restricted profile
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: latest
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/audit: restricted

# Container escape prevention checklist:
# CRITICAL:
#   No privileged containers
#   No SYS_ADMIN capability
#   No hostPID/hostNetwork/hostIPC
#   No dangerous hostPath mounts
#   runAsNonRoot: true
#   allowPrivilegeEscalation: false
#
# HIGH:
#   capabilities.drop: [ALL]
#   readOnlyRootFilesystem: true
#   seccompProfile: RuntimeDefault
#   AppArmor profile
#
# DEFENSE IN DEPTH:
#   NetworkPolicy default deny
#   Block cloud metadata
#   Falco runtime monitoring
#   Image vulnerability scanning
#   Sandbox runtime for untrusted
#   Regular kube-bench audits
```

---

## 📊 สรุป Part 52

| Attack Vector | Mitigation |
|--------------|------------|
| Privileged container | ValidatingAdmissionPolicy + PSA |
| SYS_ADMIN capability | Drop ALL + add-only-needed |
| Host path mount | OPA/Kyverno policies |
| Cloud metadata | NetworkPolicy egress deny |
| Kernel exploits | Seccomp + AppArmor + gVisor |
| Runtime detection | Falco rules + SIEM |

---

## 🔗 ต่อไป
- [Part 53: RBAC Abuse & Privilege Escalation](./part-53-rbac-abuse.md)

---
*Part 52 | Steps 491-500 | ระดับผู้เชี่ยวชาญ | Educational Use Only*
