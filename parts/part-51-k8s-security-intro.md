# Part 51: Kubernetes Security Introduction
## Steps 501-510: ความปลอดภัยใน Kubernetes

---

> ⚠️ **คำเตือนสำคัญ (Important Disclaimer)**
> เนื้อหาในส่วน Security นี้มีไว้เพื่อ:
> - **Educational purposes** - เรียนรู้วิธีป้องกันระบบ
> - **Authorized penetration testing** - ทดสอบระบบที่ได้รับอนุญาตเท่านั้น
> - **CTF challenges** - การแข่งขัน Capture The Flag
> - **Security research** - งานวิจัยด้านความปลอดภัย
>
> **ห้ามนำไปใช้โจมตีระบบที่ไม่ได้รับอนุญาต** - เป็นความผิดทางกฎหมายและจริยธรรม

---

## 📖 บทนำ

Kubernetes Security เป็นหัวข้อที่ซับซ้อนมาก เพราะ Kubernetes มี attack surface กว้างมากจาก:
- Container runtimes
- API Server
- etcd
- Network plugins
- Storage
- RBAC
- Service accounts
- Secrets management

### 4Cs of Cloud Native Security
```
Cloud     → ความปลอดภัยของ Infrastructure (AWS, GCP, Azure)
    ↓
Cluster   → ความปลอดภัยของ Kubernetes cluster
    ↓
Container → ความปลอดภัยของ container images
    ↓
Code      → ความปลอดภัยของ application code
```

---

## Step 501: Security Principles

### Principle of Least Privilege
```
ให้สิทธิ์น้อยที่สุดที่จำเป็นสำหรับการทำงาน
→ ลด blast radius ถ้า account ถูก compromise
```

### Defense in Depth
```
ไม่พึ่งพา security layer เดียว
→ ถ้า layer นึ่ง fail, ยังมี layer อื่นป้องกัน
```

### Zero Trust
```
"Never trust, always verify"
→ ไม่เชื่อถือ traffic ใดๆ แม้แต่ภายใน cluster
```

### Shift Left Security
```
ตรวจสอบ security ตั้งแต่ขั้นตอน development
→ ไม่รอแก้ปัญหาตอน production
```

---

## Step 502: Common Kubernetes Security Misconfigurations

### Top 10 Misconfigurations

```
1. Privileged containers
   └─ ให้ root access ของ host → container escape

2. HostPath volumes ที่ mount sensitive paths
   └─ อ่าน/เขียน /etc, /var/run/docker.sock, etc.

3. RBAC ที่กว้างเกินไป  
   └─ ServiceAccount มี cluster-admin
   └─ wildcard (*) permissions

4. Secrets stored in environment variables
   └─ ถูก expose ผ่าน /proc, logs, kubectl describe

5. Container images ที่ไม่ได้ scan
   └─ Known CVEs, malware, backdoors

6. Network Policies ไม่ได้กำหนด
   └─ Pods สามารถ communicate กันได้ทุก namespace

7. API Server exposed ไปอินเทอร์เน็ต
   └─ 6443/tcp exposed โดยไม่มี authentication

8. etcd ไม่ได้ encrypt
   └─ Secrets อยู่ใน plaintext ใน etcd

9. Dashboard exposed โดยไม่มี authentication
   └─ Kubernetes Dashboard ไม่มี password

10. ServiceAccount tokens auto-mounted
    └─ ทุก Pod มี SA token → อาจถูก exfiltrate
```

---

## Step 503: Attack Surface ของ Kubernetes

### External Attack Surface
```
Internet
    │
    ▼
┌─────────────────────────────────┐
│  Load Balancer / Ingress        │
│  ├─ HTTP/HTTPS (80/443)         │
│  ├─ NodePort services           │
│  └─ LoadBalancer services       │
└─────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────┐
│  Worker Nodes                   │
│  ├─ NodePort range 30000-32767  │
│  └─ SSH (22) - ถ้าเปิดไว้        │
└─────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────┐
│  Control Plane                  │
│  ├─ API Server (6443)           │
│  ├─ etcd (2379-2380)            │  ← อันตราย! ห้าม expose
│  └─ Scheduler/Controller        │
└─────────────────────────────────┘
```

### Internal Attack Surface
```
Application Code
    │ RCE, SSRF, Path Traversal
    ▼
Container
    │ Container Escape
    ▼
Pod
    │ ServiceAccount Token
    ▼
Kubernetes API
    │ RBAC Abuse
    ▼
Cluster Admin
    │ Node Compromise
    ▼
Host System
```

---

## Step 504: Security Assessment Methodology

### STRIDE Threat Model สำหรับ Kubernetes
```
S - Spoofing Identity
    └─ Impersonation ของ ServiceAccount
    └─ Forged JWT tokens

T - Tampering with Data
    └─ แก้ไข etcd data โดยตรง
    └─ Modify container images

R - Repudiation
    └─ ลบ audit logs
    └─ ปิด logging

I - Information Disclosure
    └─ Secrets leak ผ่าน environment variables
    └─ etcd data unencrypted

D - Denial of Service
    └─ Resource exhaustion
    └─ Fork bomb ใน container

E - Elevation of Privilege
    └─ Container escape → root on node
    └─ RBAC escalation → cluster-admin
```

### Penetration Testing Phases
```
1. Reconnaissance
   ├─ Scan for exposed API Server
   ├─ Identify Kubernetes version (CVEs)
   ├─ Enumerate namespaces, pods, services
   └─ Check for anonymous auth

2. Initial Access
   ├─ Exploit vulnerable application
   ├─ Abuse exposed API Server
   ├─ Supply chain attack (malicious image)
   └─ Social engineering (get kubeconfig)

3. Execution
   ├─ Run commands in pod (kubectl exec)
   ├─ Create malicious pods
   └─ Abuse Job/CronJob

4. Persistence
   ├─ Create backdoor ServiceAccount
   ├─ DaemonSet on all nodes
   ├─ Modify existing deployments
   └─ Create CronJob for reverse shell

5. Privilege Escalation
   ├─ RBAC misconfiguration
   ├─ Container escape
   ├─ Token theft
   └─ Node compromise

6. Lateral Movement
   ├─ Access other pods via network
   ├─ Access cloud metadata APIs
   ├─ Move between namespaces
   └─ Cross-cluster attacks

7. Exfiltration
   ├─ Steal Secrets
   ├─ Access application databases
   └─ Cloud credentials
```

---

## Step 505: Tools สำหรับ Kubernetes Security

### Scanning Tools (Defensive)

#### kube-bench
```bash
# ตรวจสอบ CIS Kubernetes Benchmark
docker run --rm --pid=host \
  -v /etc:/etc:ro \
  -v /var:/var:ro \
  aquasec/kube-bench:latest

# หรือ run ใน Kubernetes
kubectl apply -f https://raw.githubusercontent.com/aquasecurity/kube-bench/main/job.yaml
kubectl logs job/kube-bench
```

#### kube-hunter
```bash
# Penetration testing tool สำหรับ Kubernetes
# ติดตั้ง
pip install kube-hunter

# Scan จาก outside cluster
kube-hunter --remote <cluster-ip>

# Scan จาก inside cluster (pod)
kube-hunter --pod
```

#### Trivy (Container Scanning)
```bash
# Scan container image
trivy image nginx:latest
trivy image --severity HIGH,CRITICAL nginx:latest

# Scan Kubernetes YAML files
trivy config ./k8s-configs/

# Scan running cluster
trivy kubernetes --report summary cluster
```

#### Falco (Runtime Security)
```bash
# ติดตั้ง Falco
helm repo add falcosecurity https://falcosecurity.github.io/charts
helm install falco falcosecurity/falco \
  --namespace falco \
  --create-namespace

# ดู events
kubectl logs -n falco -l app.kubernetes.io/name=falco -f
```

---

## Step 506: พื้นฐาน Authentication ใน Kubernetes

### วิธี Authentication ที่ Kubernetes รองรับ
```
1. X.509 Client Certificates
   └─ kubeconfig ปกติใช้วิธีนี้

2. Bearer Tokens (Static Token File)
   └─ ไม่แนะนำ - ยาก rotate

3. OpenID Connect (OIDC)
   └─ ทำงานกับ Google, Azure AD, Okta
   └─ แนะนำสำหรับ production

4. Webhook Token Authentication
   └─ ส่ง token ไปยัง external service

5. ServiceAccount Tokens (JWT)
   └─ สำหรับ pods ที่ต้องการ call K8s API

6. Bootstrap Tokens
   └─ ใช้ join node ใหม่เข้า cluster
```

### ServiceAccount Token (JWT) Analysis
```bash
# ดู token ที่ mount ใน pod
cat /var/run/secrets/kubernetes.io/serviceaccount/token

# Decode JWT (educational)
# Base64 decode ส่วน payload
echo "eyJhbGciOiJSUzI1NiIsImtpZCI6Ii..." | cut -d. -f2 | base64 -d | jq

# ตัวอย่าง JWT payload:
# {
#   "iss": "kubernetes/serviceaccount",
#   "kubernetes.io/serviceaccount/namespace": "default",
#   "sub": "system:serviceaccount:default:default"
# }
```

---

## Step 507: RBAC Fundamentals

### RBAC Components
```
Subject (ใคร)          Verb (ทำอะไร)       Resource (กับอะไร)
───────────────────      ────────────      ──────────────────
User                   get                 pods
Group                  list                deployments
ServiceAccount         create              secrets
                       update              configmaps
                       patch               services
                       delete              namespaces
                       watch               nodes
                       * (all)             * (all)
```

### Role vs ClusterRole
```yaml
# Role - ใช้ได้แค่ใน namespace เดียว
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: my-app
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "watch", "list"]

---
# ClusterRole - ใช้ได้ทั้ง cluster
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: pod-reader
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "watch", "list"]
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list"]
```

### ตรวจสอบ RBAC
```bash
# ตรวจสอบสิทธิ์ของตัวเอง
kubectl auth can-i get pods
kubectl auth can-i create deployments
kubectl auth can-i --list

# ตรวจสอบสิทธิ์ของ user/sa อื่น
kubectl auth can-i get pods --as=alice
kubectl auth can-i get pods --as=system:serviceaccount:default:my-sa
kubectl auth can-i --list --as=system:serviceaccount:default:my-sa
```

---

## Step 508: Secrets Management

### Kubernetes Secrets (built-in)
```yaml
# สร้าง Secret
apiVersion: v1
kind: Secret
metadata:
  name: my-secret
  namespace: my-app
type: Opaque
data:
  # ต้อง base64 encode
  username: YWRtaW4=          # echo -n 'admin' | base64
  password: c2VjcmV0MTIz      # echo -n 'secret123' | base64
stringData:
  # หรือใช้ stringData (K8s จะ encode ให้)
  config.json: |
    {
      "api_key": "my-api-key",
      "endpoint": "https://api.example.com"
    }
```

### Encryption at Rest สำหรับ etcd
```yaml
# encryption-config.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
    providers:
      # ใช้ AES-GCM encryption
      - aescbc:
          keys:
            - name: key1
              secret: <base64-32-byte-key>
      
      # KMS provider (recommended สำหรับ production)
      - kms:
          name: aws-kms
          endpoint: unix:///tmp/socketfile.sock
          cachesize: 100
          timeout: 3s
      
      # identity = unencrypted (fallback)
      - identity: {}
```

---

## Step 509: Network Security Basics

### Network Policy คืออะไร?
```
โดย default: ทุก Pod สามารถ communicate กันได้ทุกตัว!
Network Policy: ใช้กำหนดว่า Pod ไหนคุยกับ Pod ไหนได้

เหมือน firewall ระดับ Pod/Namespace
```

### Default Deny All
```yaml
# Best practice: เริ่มต้น deny all แล้วค่อย allow เฉพาะที่จำเป็น

# Deny all ingress traffic สำหรับ namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: my-app
spec:
  podSelector: {}    # เลือกทุก pod ใน namespace
  policyTypes:
    - Ingress

---
# Deny all egress traffic
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-egress
  namespace: my-app
spec:
  podSelector: {}
  policyTypes:
    - Egress
```

### Allow Specific Traffic
```yaml
# Allow ingress จาก frontend ไป backend
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: my-app
spec:
  podSelector:
    matchLabels:
      role: backend
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              role: frontend
      ports:
        - protocol: TCP
          port: 8080
```

---

## Step 510: Security Context

### Pod Security Context
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  # Pod-level security context
  securityContext:
    runAsUser: 1000              # รัน container ด้วย user ID 1000
    runAsGroup: 3000             # primary group ID
    fsGroup: 2000                # group สำหรับ volumes
    runAsNonRoot: true           # ห้ามรัน as root
    seccompProfile:
      type: RuntimeDefault       # ใช้ default seccomp profile
  
  containers:
    - name: app
      image: myapp:latest
      
      # Container-level security context
      securityContext:
        allowPrivilegeEscalation: false    # ห้าม setuid/setgid
        readOnlyRootFilesystem: true        # root filesystem read-only
        capabilities:
          drop:
            - ALL                           # drop ทุก capabilities
          add:
            - NET_BIND_SERVICE             # เพิ่มกลับเฉพาะที่จำเป็น
        runAsNonRoot: true
        runAsUser: 1000
      
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

### Pod Security Standards (PSS)
```yaml
# Kubernetes 1.25+ ใช้ Pod Security Standards แทน PSP

# Label บน namespace เพื่อ enforce policy
apiVersion: v1
kind: Namespace
metadata:
  name: my-app
  labels:
    # policy levels: privileged, baseline, restricted
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

---

## 📊 สรุป Part 51

| หัวข้อ | สิ่งที่เรียน |
|--------|------------|
| 4Cs Security | Cloud → Cluster → Container → Code |
| Misconfigurations | Top 10 security issues |
| Attack Surface | External/Internal attack vectors |
| Methodology | Recon → Initial Access → Execution → etc. |
| Tools | kube-bench, kube-hunter, Trivy, Falco |
| Authentication | X.509, JWT, OIDC, Anonymous |
| RBAC | Role, ClusterRole, Binding |
| Secrets | Types, Encryption at rest |
| Network Policy | Default deny, selective allow |
| Security Context | runAsNonRoot, readOnly, capabilities |

---

## 🔗 ต่อไป
- [Part 52: Threat Modeling](./part-52-k8s-threat-model.md)
- [Part 55: Privilege Escalation](./part-55-k8s-privilege-escalation.md)

---
*Part 51 | Steps 501-510 | ระดับสูง - Security*
