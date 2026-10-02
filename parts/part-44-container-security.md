# Part 44: Container Security Fundamentals
## Steps 421-430: Securing Containers and Images

---

## 📖 บทนำ

Container security ครอบคลุมตั้งแต่การ build image ที่ปลอดภัย การ scan vulnerabilities ไปจนถึงการ configure runtime security เป็นส่วนสำคัญของ defense-in-depth strategy

> **หมายเหตุ**: เนื้อหาด้าน security ในหลักสูตรนี้มีไว้เพื่อการศึกษาและการทดสอบระบบที่ได้รับอนุญาตเท่านั้น การนำไปใช้โจมตีระบบที่ไม่ได้รับอนุญาตเป็นสิ่งผิดกฎหมาย

---

## Step 421: Container Threat Model

```
Container Security Layers:
┌─────────────────────────────────────────────────────────────┐
│                    Host OS / Kernel                         │
│  ┌───────────────────────────────────────────────────┐   │
│  │                Container Runtime                     │   │
│  │  ┌────────────────────────────────────────────┐ │   │
│  │  │              Container                         │ │   │
│  │  │  ┌────────────────────────────────────────┐  │ │   │
│  │  │  │  Application (process)                   │  │ │   │
│  │  │  └────────────────────────────────────────┘  │ │   │
│  │  │  Filesystem | Network ns | PID ns | UTS ns     │ │   │
│  │  └────────────────────────────────────────────┘ │   │
│  └───────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘

Attack surfaces:
1. Image vulnerabilities (CVEs in base image/packages)
2. Misconfigured containers (root, privileged, writable FS)
3. Secrets in images or environment variables
4. Container escape (kernel exploits, privileged containers)
5. Supply chain attacks (compromised images)
6. Network attacks (lateral movement)
```

---

## Step 422: Secure Dockerfile

```dockerfile
# แย่ - ห้ามทำ
FROM ubuntu:latest
RUN apt-get install -y curl wget
COPY . /app
RUN chmod 777 /app
CMD ["./app"]
```

```dockerfile
# ดี - best practices

# 1. Multi-stage build
FROM golang:1.21 AS builder
WORKDIR /build
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o app ./cmd/app

# 2. Distroless runtime
FROM gcr.io/distroless/static-debian12:nonroot AS runtime
COPY --from=builder /build/app /app
EXPOSE 8080
USER nonroot:nonroot
ENTRYPOINT ["/app"]
```

```dockerfile
# Full secure Dockerfile (Node.js example)
FROM node:20-slim AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production && npm cache clean --force
COPY --chown=node:node src/ ./src/

FROM node:20-slim AS runtime
WORKDIR /app
COPY --from=builder --chown=node:node /app/node_modules ./node_modules
COPY --from=builder --chown=node:node /app/src ./src
COPY --chown=node:node package.json .
USER node
EXPOSE 3000
HEALTHCHECK --interval=30s --timeout=3s \
  CMD node healthcheck.js || exit 1
CMD ["node", "src/index.js"]
```

---

## Step 423: Image Scanning

```yaml
# Trivy Operator (automated in-cluster scanning)
apiVersion: v1
kind: ConfigMap
metadata:
  name: trivy-operator
  namespace: trivy-system
data:
  scanJobTTL: "24h"
  vulnerabilityScanner.enabled: "true"
  configAuditScanner.enabled: "true"
  rbacAssessmentScanner.enabled: "true"
  severity: "UNKNOWN,LOW,MEDIUM,HIGH,CRITICAL"
  trivyImageRef: "ghcr.io/aquasecurity/trivy:0.45.0"

---
# ดู vulnerability report
apiVersion: aquasecurity.github.io/v1alpha1
kind: VulnerabilityReport
metadata:
  name: replicaset-my-app-xxx-app
  namespace: default
report:
  artifact:
    digest: sha256:abc123
    repository: myregistry.io/my-app
    tag: "1.0"
  summary:
    criticalCount: 0
    highCount: 2
    mediumCount: 5
    lowCount: 10
  vulnerabilities:
    - fixedVersion: "1.2.3"
      installedVersion: "1.2.0"
      primaryLink: "https://avd.aquasec.com/nvd/cve-2024-12345"
      resource: "libssl"
      severity: HIGH
      title: "OpenSSL buffer overflow"
      vulnerabilityID: CVE-2024-12345

# CLI commands:
# trivy image nginx:latest
# trivy image --severity HIGH,CRITICAL myapp:1.0
# trivy image --exit-code 1 --severity CRITICAL myapp:1.0
# trivy fs --security-checks vuln,secret,config .
# trivy k8s --report summary cluster
```

---

## Step 424: Image Signing

```yaml
# Cosign - sign container images
# cosign generate-key-pair
# cosign sign --key cosign.key myregistry.io/myapp:1.0
# cosign verify --key cosign.pub myregistry.io/myapp:1.0

# Policy Controller - enforce signed images
apiVersion: policy.sigstore.dev/v1beta1
kind: ClusterImagePolicy
metadata:
  name: require-signed-images
spec:
  images:
    - glob: "myregistry.io/**"
  authorities:
    - key:
        data: |
          -----BEGIN PUBLIC KEY-----
          MFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAE...
          -----END PUBLIC KEY-----
      ctlog:
        url: https://rekor.sigstore.dev

---
# OPA Gatekeeper: require images from approved registry
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8sallowedinageregistries
spec:
  crd:
    spec:
      names:
        kind: K8sAllowedImageRegistries
      validation:
        openAPIV3Schema:
          properties:
            registries:
              type: array
              items:
                type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8sallowedinageregistries
        
        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not starts_with_allowed(container.image)
          msg := sprintf("image %v not from allowed registry", [container.image])
        }
        
        starts_with_allowed(image) {
          registry := input.parameters.registries[_]
          startswith(image, registry)
        }
```

---

## Step 425: Runtime Security Context

```yaml
# Comprehensive security context
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 10001
    runAsGroup: 10001
    fsGroup: 10001
    fsGroupChangePolicy: OnRootMismatch
    seccompProfile:
      type: RuntimeDefault
  
  automountServiceAccountToken: false
  
  containers:
    - name: app
      image: myregistry.io/app:1.0
      securityContext:
        allowPrivilegeEscalation: false
        privileged: false
        readOnlyRootFilesystem: true
        runAsNonRoot: true
        runAsUser: 10001
        capabilities:
          drop:
            - ALL
          add:
            - NET_BIND_SERVICE
      
      volumeMounts:
        - name: tmp
          mountPath: /tmp
        - name: cache
          mountPath: /app/cache
      
      resources:
        requests:
          cpu: "100m"
          memory: "128Mi"
        limits:
          cpu: "500m"
          memory: "256Mi"
  
  volumes:
    - name: tmp
      emptyDir: {}
    - name: cache
      emptyDir:
        sizeLimit: 100Mi
```

---

## Step 426: Seccomp Profiles

```yaml
# Custom Seccomp Profile (ลดเฉพาะ syscalls ที่จำเป็น)
apiVersion: security-profiles-operator.x-k8s.io/v1beta1
kind: SeccompProfile
metadata:
  name: app-seccomp
  namespace: default
spec:
  defaultAction: SCMP_ACT_ERRNO
  architectures:
    - SCMP_ARCH_X86_64
  syscalls:
    - action: SCMP_ACT_ALLOW
      names:
        # File I/O
        - read
        - write
        - open
        - openat
        - close
        - stat
        - fstat
        - lstat
        - mmap
        - mprotect
        - munmap
        - brk
        # Process
        - exit
        - exit_group
        - getpid
        - getuid
        - getgid
        # Network
        - socket
        - connect
        - accept
        - bind
        - listen
        - sendto
        - recvfrom
        - getsockname
        - setsockopt
        - getsockopt
        # Threading
        - clone
        - futex
        - nanosleep
        # Signals
        - rt_sigaction
        - rt_sigprocmask
        # Misc
        - arch_prctl
        - ioctl
        - fcntl
        - epoll_create1
        - epoll_ctl
        - epoll_wait
        - poll
        - clock_gettime

---
# ใช้ profile กับ Pod
apiVersion: v1
kind: Pod
spec:
  securityContext:
    seccompProfile:
      type: Localhost
      localhostProfile: operator/default/app-seccomp.json
```

---

## Step 427: AppArmor Profiles

```yaml
# AppArmor profile content (/etc/apparmor.d/k8s-nginx)
# 
# #include <tunables/global>
# profile k8s-nginx flags=(attach_disconnected,mediate_deleted) {
#   #include <abstractions/base>
#   network inet tcp,
#   network inet udp,
#   /usr/sbin/nginx mr,
#   /var/log/nginx/** wl,
#   /var/run/nginx.pid lk,
#   /etc/nginx/** r,
#   /usr/share/nginx/html/** r,
#   deny /etc/shadow r,
#   deny /etc/passwd w,
#   deny @{PROC}/** rw,
# }

# Load profile: apparmor_parser -r -W /etc/apparmor.d/k8s-nginx

---
# ใช้ AppArmor กับ Pod
apiVersion: v1
kind: Pod
metadata:
  name: nginx-secured
spec:
  containers:
    - name: nginx
      image: nginx:1.25
      securityContext:
        appArmorProfile:
          type: Localhost
          localhostProfile: k8s-nginx
```

---

## Step 428: Linux Capabilities

```yaml
# Linux Capabilities ที่อันตราย (ห้ามให้ถ้าไม่จำเป็น):
# CAP_SYS_ADMIN    - ทำได้เกือบทุกอย่าง (เหมือน root)
# CAP_NET_ADMIN    - configure network interfaces
# CAP_SYS_PTRACE   - trace any process
# CAP_DAC_OVERRIDE - bypass file permission
# CAP_NET_RAW      - use raw/packet sockets
# CAP_SETUID       - change UID
# CAP_SYS_MODULE   - load kernel modules

# Safe capabilities:
# CAP_NET_BIND_SERVICE - bind port < 1024

---
apiVersion: v1
kind: Pod
spec:
  containers:
    - name: minimal-caps
      image: myapp:1.0
      securityContext:
        capabilities:
          drop:
            - ALL
          add:
            - NET_BIND_SERVICE

# ตรวจสอบ capabilities:
# kubectl exec -it pod-name -- cat /proc/1/status | grep Cap
# capsh --decode=00000000a80425fb
```

---

## Step 429: Resource Limits

```yaml
# LimitRange - ป้องกัน containers ไม่ระบุ limits
apiVersion: v1
kind: LimitRange
metadata:
  name: container-limits
  namespace: production
spec:
  limits:
    - type: Container
      default:
        cpu: "500m"
        memory: "256Mi"
      defaultRequest:
        cpu: "100m"
        memory: "128Mi"
      max:
        cpu: "4"
        memory: "4Gi"
      min:
        cpu: "50m"
        memory: "64Mi"
    - type: Pod
      max:
        cpu: "8"
        memory: "8Gi"
    - type: PersistentVolumeClaim
      max:
        storage: "10Gi"
      min:
        storage: "1Gi"

---
# ResourceQuota
apiVersion: v1
kind: ResourceQuota
metadata:
  name: namespace-quota
  namespace: production
spec:
  hard:
    requests.cpu: "20"
    requests.memory: "40Gi"
    limits.cpu: "40"
    limits.memory: "80Gi"
    pods: "50"
    services: "20"
    services.loadbalancers: "2"
    requests.storage: "100Gi"
    persistentvolumeclaims: "20"
    secrets: "100"
    configmaps: "50"
```

---

## Step 430: Workshop - Container Security Policy

```yaml
# ValidatingAdmissionPolicy - enforce security context
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicy
metadata:
  name: container-security-policy
spec:
  failurePolicy: Fail
  matchConstraints:
    resourceRules:
      - apiGroups: [""]
        apiVersions: ["v1"]
        operations: ["CREATE", "UPDATE"]
        resources: ["pods"]
  validations:
    # ต้องไม่รัน root
    - expression: >
        object.spec.containers.all(c,
          c.?securityContext.?runAsNonRoot.orValue(false) == true ||
          (c.?securityContext.?runAsUser.orValue(0) > 0)
        )
      message: "All containers must run as non-root"
    
    # ต้องไม่ privileged
    - expression: >
        object.spec.containers.all(c,
          c.?securityContext.?privileged.orValue(false) == false
        )
      message: "Privileged containers are not allowed"
    
    # ต้องไม่ allowPrivilegeEscalation
    - expression: >
        object.spec.containers.all(c,
          c.?securityContext.?allowPrivilegeEscalation.orValue(true) == false
        )
      message: "allowPrivilegeEscalation must be false"
    
    # ต้องมี resource limits
    - expression: >
        object.spec.containers.all(c,
          has(c.resources) &&
          has(c.resources.limits) &&
          has(c.resources.limits.memory) &&
          has(c.resources.limits.cpu)
        )
      message: "All containers must have resource limits"
    
    # Image ต้องมาจาก approved registry
    - expression: >
        object.spec.containers.all(c,
          c.image.startsWith("myregistry.io/") ||
          c.image.startsWith("gcr.io/distroless/")
        )
      message: "Images must come from approved registries"

---
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicyBinding
metadata:
  name: container-security-policy-binding
spec:
  policyName: container-security-policy
  validationActions: [Deny]
  matchResources:
    namespaceSelector:
      matchExpressions:
        - key: security-policy
          operator: NotIn
          values: ["exempt"]
```

---

## 📊 สรุป Part 44

| ด้าน | Best Practice |
|------|-------------- |
| Image Building | Multi-stage, distroless, non-root |
| Image Scanning | Trivy, scan ทุก build |
| Image Signing | Cosign/Notation |
| Security Context | runAsNonRoot, readOnly FS, drop ALL caps |
| Seccomp | RuntimeDefault หรือ custom profile |
| AppArmor | Profile ที่ restrict filesystem/network |
| Resources | Limits + LimitRange + ResourceQuota |

---

## 🔗 ต่อไป
- [Part 45: Secrets Management](./part-45-secrets-management.md)

---
*Part 44 | Steps 421-430 | ระดับสูง*
