# Part 38: Pod Security Standards
## Steps 361-370: Securing Kubernetes Pods

---

## 📖 บทนำ

Pod Security Standards (PSS) เป็น built-in security policy framework สำหรับ Kubernetes 1.23+ แทนที่ PodSecurityPolicy ที่ deprecated

> **หมายเหตุ**: เนื้อหาด้าน security ในหลักสูตรนี้มีไว้เพื่อการศึกษาและการทดสอบระบบที่ได้รับอนุญาตเท่านั้น

---

## Step 361: PSS Levels

```yaml
# Pod Security Standards - 3 levels:
#
# restricted: ข้อกำหนดเข้มงวดที่สุด - production workloads
# baseline: บล็อก escalations ชัดเจน - general purpose
# privileged: ไม่มีข้อจำกัด - trusted workloads เช่น monitoring
#
# 3 modes:
# enforce: reject pods ที่ไม่ผ่าน
# audit: บันทึก log แต่ไม่ reject
# warn: แสดง warning แต่ไม่ reject

# Namespace labels
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: v1.28
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/audit-version: v1.28
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/warn-version: v1.28

---
# Development namespace - baseline
apiVersion: v1
kind: Namespace
metadata:
  name: development
  labels:
    pod-security.kubernetes.io/enforce: baseline
    pod-security.kubernetes.io/warn: restricted  # warn แต่ไม่ block

---
# System namespace - privileged
apiVersion: v1
kind: Namespace
metadata:
  name: monitoring
  labels:
    pod-security.kubernetes.io/enforce: privileged
```

---

## Step 362: Restricted Profile Requirements

```yaml
# Pod ที่ผ่าน restricted profile
apiVersion: v1
kind: Pod
metadata:
  name: compliant-pod
  namespace: production
spec:
  securityContext:
    runAsNonRoot: true           # ต้องไม่ root
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
    seccompProfile:
      type: RuntimeDefault       # ต้องมี seccomp
    supplementalGroups: [4000]
  
  containers:
    - name: app
      image: myregistry.io/app:1.0
      securityContext:
        allowPrivilegeEscalation: false  # required
        readOnlyRootFilesystem: true     # required
        runAsNonRoot: true               # required
        capabilities:
          drop:
            - ALL                        # drop all capabilities
          # add ไม่ได้เลย ใน restricted profile
        seccompProfile:
          type: RuntimeDefault
      
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
  
  hostPID: false       # ห้ามใช้ host PID
  hostIPC: false       # ห้ามใช้ host IPC
  hostNetwork: false   # ห้ามใช้ host network
```

---

## Step 363: Baseline Profile

```yaml
# Baseline profile - บล็อก escalations ที่ชัดเจน
# ห้าม:
# - privileged: true
# - hostPID: true, hostIPC: true, hostNetwork: true
# - hostPath volumes
# - hostPort
# - capabilities: NET_RAW, SYS_ADMIN, etc.
# - AppArmor override (baseline+)
# - Seccomp Unconfined

# ยังอนุญาต:
# - runAsRoot (ใน baseline)
# - non-readonly filesystem
# - allowPrivilegeEscalation
# - NET_BIND_SERVICE capability

apiVersion: v1
kind: Pod
metadata:
  name: baseline-pod
  namespace: development
spec:
  containers:
    - name: app
      image: myregistry.io/app:1.0
      securityContext:
        # Baseline อนุญาตสิ่งเหล่านี้
        runAsUser: 0             # root ยังได้ใน baseline
        allowPrivilegeEscalation: true  # ยังได้
        capabilities:
          add:
            - NET_BIND_SERVICE  # ยังได้
      ports:
        - containerPort: 80
  
  # ห้ามสิ่งเหล่านี้ใน baseline:
  # hostPID: true   # ❌
  # hostIPC: true   # ❌
  # hostNetwork: true  # ❌
  # containers[].securityContext.privileged: true  # ❌
```

---

## Step 364: Falco Runtime Security

```yaml
# Falco - runtime security monitoring
apiVersion: v1
kind: ConfigMap
metadata:
  name: falco-rules
  namespace: falco
data:
  custom_rules.yaml: |
    # Detect container escape attempts
    - rule: Container Namespace Escape Attempt
      desc: Detect attempts to escape container namespace
      condition: >
        spawned_process and container and
        (proc.name in (nsenter, unshare) or
         (proc.name = chroot and fd.name contains "/"))
      output: >
        Namespace escape attempt (user=%user.name %container.info
        command=%proc.cmdline)
      priority: CRITICAL
      tags: [container, escape]
    
    # Detect write to /etc
    - rule: Write below etc
      desc: Detect write to /etc in container
      condition: >
        write_etc_common and container
      output: >
        File below /etc opened for writing (user=%user.name 
        command=%proc.cmdline file=%fd.name %container.info)
      priority: ERROR
      tags: [filesystem, mitre_persistence]
    
    # Detect sensitive mount
    - rule: Sensitive Mount in Container
      desc: Detect sensitive path mounts in containers
      condition: >
        container and
        (proc.name = mount or proc.name = umount) and
        (evt.arg.target startswith /proc or
         evt.arg.target startswith /sys or
         evt.arg.target = /etc)
      output: >
        Sensitive path mount detected (user=%user.name 
        command=%proc.cmdline target=%evt.arg.target %container.info)
      priority: WARNING
      tags: [container, filesystem]
    
    # Detect shell in container
    - rule: Terminal Shell in Container
      desc: A shell was used as the entrypoint or is run interactively
      condition: >
        spawned_process and container and shell_procs and proc.tty != 0
        and container_entrypoint
      output: >
        A shell was spawned in a container with an attached terminal
        (user=%user.name %container.info shell=%proc.name parent=%proc.pname
        cmdline=%proc.cmdline terminal=%proc.tty)
      priority: NOTICE
      tags: [container, shell]
    
    # Network scan detection
    - rule: Port Scan Attempt
      desc: Detect port scan attempts from containers
      condition: >
        inbound_outbound and container and
        fd.sport < 1024 and fd.sport != 0 and
        (fd.l4proto = tcp) and
        not fd.connected
      output: >
        Port scan attempt from container (user=%user.name
        %container.info sport=%fd.sport dport=%fd.dport)
      priority: WARNING
      tags: [network, scan]
```

---

## Step 365: Falco Deployment

```yaml
# Falco DaemonSet values
tolerations:
  - effect: NoSchedule
    key: node-role.kubernetes.io/control-plane

driver:
  kind: ebpf   # eBPF driver (preferred), หรือ module, modern_ebpf

falco:
  rules_file:
    - /etc/falco/falco_rules.yaml
    - /etc/falco/falco_rules.local.yaml
    - /etc/falco/rules.d
  
  plugins:
    - name: k8saudit
      library_path: libk8saudit.so
      init_config:
        webserver:
          listen_port: 9765
      open_params: "http://localhost:9765/k8s-audit"
  
  outputs:
    rate: 1
    max_burst: 1000
  
  json_output: true
  json_include_output_property: true
  
  grpc:
    enabled: true
    bind_address: "unix:///run/falco/falco.sock"
    threadiness: 8
  
  grpc_output:
    enabled: true
  
  http_output:
    enabled: true
    url: "http://falcosidekick:2801/"

# Alerting via falcosidekick
falcosidekick:
  enabled: true
  config:
    slack:
      webhookurl: "https://hooks.slack.com/services/xxx"
      channel: "#security-alerts"
      minimumpriority: WARNING
    
    pagerduty:
      routingkey: "xxxxx"
      minimumpriority: CRITICAL
    
    loki:
      hostport: "http://loki.monitoring:3100"
      minimumpriority: WARNING
```

---

## Step 366: Seccomp Profiles

```yaml
# Custom Seccomp profile
apiVersion: security-profiles-operator.x-k8s.io/v1beta1
kind: SeccompProfile
metadata:
  name: nginx-profile
  namespace: production
spec:
  defaultAction: SCMP_ACT_ERRNO
  syscalls:
    - action: SCMP_ACT_ALLOW
      names:
        # Process
        - fork
        - execve
        - exit
        - exit_group
        - wait4
        - waitid
        # Memory
        - mmap
        - mprotect
        - munmap
        - brk
        - mlock
        # Files
        - open
        - openat
        - read
        - write
        - close
        - stat
        - fstat
        - lstat
        - access
        # Network
        - socket
        - bind
        - listen
        - accept
        - accept4
        - connect
        - sendto
        - recvfrom
        # Signals
        - rt_sigaction
        - rt_sigprocmask
        - rt_sigreturn
        - kill
        - tkill

---
# ใช้ custom seccomp profile ใน pod
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  namespace: production
spec:
  securityContext:
    seccompProfile:
      type: Localhost
      localhostProfile: operator/production/nginx-profile.json
  containers:
    - name: nginx
      image: nginx:1.25
```

---

## Step 367: AppArmor Profiles

```yaml
# AppArmor Profile (โหลดบน node)
# /etc/apparmor.d/nginx-container
# profile nginx-container flags=(attach_disconnected,mediate_deleted) {
#   include <abstractions/base>
#   network inet tcp,
#   network inet6 tcp,
#   /etc/nginx/** r,
#   /var/log/nginx/** w,
#   /tmp/** rw,
#   deny /proc/** rw,
#   deny /sys/** rw,
#   deny @{PROC}/sysrq-trigger rwklx,
#   deny @{PROC}/mem rwklx,
#   deny @{PROC}/kmem rwklx,
#   deny /sys/[^f]*/** wklx,
#   deny /sys/f[^s]*/** wklx,
# }

# ใช้ AppArmor ใน pod
apiVersion: v1
kind: Pod
metadata:
  name: nginx-apparmor
  namespace: production
  annotations:
    container.apparmor.security.beta.kubernetes.io/nginx: localhost/nginx-container
spec:
  containers:
    - name: nginx
      image: nginx:1.25
      securityContext:
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
```

---

## Step 368: Security Scanning

```yaml
# Trivy Operator - automatic image scanning
apiVersion: aquasecurity.github.io/v1alpha1
kind: VulnerabilityReport
# auto-generated by trivy-operator

---
# trivy-operator values
trivy:
  image:
    registry: ghcr.io
    repository: aquasecurity/trivy
    tag: 0.47.0
  
  severity: UNKNOWN,LOW,MEDIUM,HIGH,CRITICAL
  
  ignoreUnfixed: false
  
  dbRepository: ghcr.io/aquasecurity/trivy-db

operator:
  scanJobTimeout: 5m
  concurrentScanJobsLimit: 10
  
  vulnerabilityScannerEnabled: true
  configAuditScannerEnabled: true
  rbacAssessmentScannerEnabled: true
  
  metricsFindingsEnabled: true

# Scan results prometheus metrics
# trivy_vulnerability_id{...} gauge
```

---

## Step 369-370: Workshop - Security Hardening

```yaml
# Security hardening checklist:

# 1. Namespace PSS
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: latest

---
# 2. Pod security template
apiVersion: v1
kind: Pod
metadata:
  name: hardened-app
spec:
  automountServiceAccountToken: false
  securityContext:
    runAsNonRoot: true
    runAsUser: 65534  # nobody
    runAsGroup: 65534
    fsGroup: 65534
    seccompProfile:
      type: RuntimeDefault
  
  containers:
    - name: app
      image: myregistry.io/app:1.0
      securityContext:
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        runAsNonRoot: true
        capabilities:
          drop: [ALL]
        seccompProfile:
          type: RuntimeDefault
      
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
        - name: app-config
          mountPath: /etc/app
          readOnly: true
  
  volumes:
    - name: tmp
      emptyDir:
        sizeLimit: 100Mi
    - name: app-config
      configMap:
        name: app-config

---
# 3. Falco alert + auto-response
# ใช้ Falco + falcosidekick + webhook
# เมื่อ detect suspicious activity -> terminate pod หรือ alert
```

---

## 📊 สรุป Part 38

| Component | หน้าที่ |
|-----------|------|
| PSS restricted | เข้มงวดสูงสุด: no root, no privilege |
| PSS baseline | บล็อก escalations ชัดเจน |
| PSS privileged | ไม่มีข้อจำกัด (system components) |
| Falco | Runtime threat detection |
| Seccomp | System call filtering |
| AppArmor | Mandatory access control |
| Trivy Operator | Automated vulnerability scanning |

---

## 🔗 ต่อไป
- [Part 39: Kubernetes RBAC Advanced](./part-39-rbac-advanced.md)

---
*Part 38 | Steps 361-370 | ระดับสูง*
