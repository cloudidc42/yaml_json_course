# Part 22: Kubernetes Architecture Deep Dive
## Steps 211-220: ภายในของ Kubernetes

---

## 📖 บทนำ

การเข้าใจ Kubernetes Architecture อย่างลึกซึ้งช่วยให้เราสามารถ debug, optimize, และ secure cluster ได้อย่างมีประสิทธิภาพ

### Architecture Overview
```
┌────────────────────── Control Plane ──────────────────────┐
│                                                            │
│  ┌─────────────────┐  ┌──────────────────┐               │
│  │  kube-apiserver  │  │ cloud-controller  │               │
│  │  (RESTful API)   │  │ manager (CCM)     │               │
│  └────────┬─────────┘  └──────────────────┘               │
│           │                                                │
│  ┌────────▼──────┐  ┌──────────────────────┐              │
│  │     etcd       │  │ kube-controller-mgr  │              │
│  │  (key-value)   │  │  (reconcile loops)   │              │
│  └───────────────┘  └──────────────────────┘              │
│                                                            │
│  ┌──────────────────────────────────────────┐             │
│  │          kube-scheduler                  │             │
│  │  (assign pods to nodes)                  │             │
│  └──────────────────────────────────────────┘             │
└────────────────────────────────────────────────────────────┘

┌────────────── Worker Node ─────────────────┐
│  ┌─────────────┐  ┌──────────────────────┐ │
│  │   kubelet   │  │     kube-proxy       │ │
│  │  (pod mgmt) │  │  (iptables/IPVS)     │ │
│  └─────────────┘  └──────────────────────┘ │
│  ┌─────────────────────────────────────┐   │
│  │      Container Runtime (containerd) │   │
│  └─────────────────────────────────────┘   │
└────────────────────────────────────────────┘
```

---

## Step 211: kube-apiserver

```yaml
# kube-apiserver เป็น central hub ของ Kubernetes
# ทุก operations ผ่าน apiserver

# Request Flow:
# Client → Authentication → Authorization → Admission Control → etcd

# Authentication Methods
apiVersion: v1
kind: Config
clusters:
  - name: my-cluster
    cluster:
      server: https://apiserver:6443
      certificate-authority: /path/to/ca.crt
users:
  - name: admin
    user:
      # Method 1: X.509 Certificate
      client-certificate: /path/to/admin.crt
      client-key: /path/to/admin.key
      
      # Method 2: Bearer Token
      # token: eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...
      
      # Method 3: OIDC (external IdP)
      # auth-provider:
      #   name: oidc
      #   config:
      #     idp-issuer-url: https://accounts.google.com
      #     client-id: kubernetes

---
# kube-apiserver flags ที่สำคัญ (Static Pod manifest)
apiVersion: v1
kind: Pod
metadata:
  name: kube-apiserver
  namespace: kube-system
spec:
  containers:
    - name: kube-apiserver
      image: registry.k8s.io/kube-apiserver:v1.28.0
      command:
        - kube-apiserver
        - --advertise-address=10.0.0.1
        - --allow-privileged=true
        - --audit-policy-file=/etc/kubernetes/audit-policy.yaml
        - --audit-log-path=/var/log/kubernetes/audit.log
        - --authorization-mode=Node,RBAC
        - --client-ca-file=/etc/kubernetes/pki/ca.crt
        - --enable-admission-plugins=NodeRestriction,PodSecurity
        - --encryption-provider-config=/etc/kubernetes/enc/encryption.yaml
        - --etcd-servers=https://127.0.0.1:2379
        - --etcd-cafile=/etc/kubernetes/pki/etcd/ca.crt
        - --etcd-certfile=/etc/kubernetes/pki/apiserver-etcd-client.crt
        - --etcd-keyfile=/etc/kubernetes/pki/apiserver-etcd-client.key
        - --feature-gates=TTLAfterFinished=true
        - --oidc-issuer-url=https://accounts.google.com
        - --oidc-client-id=kubernetes
        - --service-cluster-ip-range=10.96.0.0/12
        - --tls-cert-file=/etc/kubernetes/pki/apiserver.crt
        - --tls-private-key-file=/etc/kubernetes/pki/apiserver.key
```

---

## Step 212: etcd - Distributed Key-Value Store

```bash
# etcd เก็บ state ทั้งหมดของ cluster
# Format: /registry/<resource-type>/<namespace>/<name>

# ดู keys ใน etcd (ต้องมี cert)
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
  --key=/etc/kubernetes/pki/etcd/healthcheck-client.key \
  get / --prefix --keys-only | head -20

# Backup etcd
ETCDCTL_API=3 etcdctl snapshot save \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
  --key=/etc/kubernetes/pki/etcd/healthcheck-client.key \
  /backup/etcd-snapshot-$(date +%Y%m%d).db

# Restore etcd
ETCDCTL_API=3 etcdctl snapshot restore \
  /backup/etcd-snapshot-20240101.db \
  --data-dir=/var/lib/etcd-restored
```

```yaml
# etcd static pod manifest
apiVersion: v1
kind: Pod
metadata:
  name: etcd
  namespace: kube-system
spec:
  containers:
    - name: etcd
      image: registry.k8s.io/etcd:3.5.9-0
      command:
        - etcd
        - --advertise-client-urls=https://10.0.0.1:2379
        - --cert-file=/etc/kubernetes/pki/etcd/server.crt
        - --client-cert-auth=true
        - --data-dir=/var/lib/etcd
        - --initial-cluster=master1=https://10.0.0.1:2380,master2=https://10.0.0.2:2380,master3=https://10.0.0.3:2380
        - --initial-cluster-state=new
        - --initial-cluster-token=etcd-cluster-1
        - --key-file=/etc/kubernetes/pki/etcd/server.key
        - --listen-client-urls=https://127.0.0.1:2379,https://10.0.0.1:2379
        - --name=master1
        - --peer-cert-file=/etc/kubernetes/pki/etcd/peer.crt
        - --peer-client-cert-auth=true
        - --peer-key-file=/etc/kubernetes/pki/etcd/peer.key
        - --peer-trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
        - --snapshot-count=10000
        - --trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
```

---

## Step 213: kube-scheduler

```yaml
# Node Affinity
apiVersion: v1
kind: Pod
metadata:
  name: affinity-pod
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: topology.kubernetes.io/zone
                operator: In
                values:
                  - us-east-1a
                  - us-east-1b
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 80
          preference:
            matchExpressions:
              - key: node.kubernetes.io/instance-type
                operator: In
                values:
                  - m5.large
                  - m5.xlarge
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 100
          podAffinityTerm:
            labelSelector:
              matchExpressions:
                - key: app
                  operator: In
                  values:
                    - my-app
            topologyKey: kubernetes.io/hostname
  containers:
    - name: app
      image: myapp:1.0.0

---
# Topology Spread Constraints
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 6
  template:
    spec:
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: my-app
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: ScheduleAnyway
          labelSelector:
            matchLabels:
              app: my-app
      containers:
        - name: app
          image: myapp:1.0.0
```

---

## Step 214: kube-controller-manager

```yaml
# Controller Manager รัน controllers ต่างๆ:
# - Node Controller, Replication, Endpoints, ServiceAccount
# - Namespace, Job, Deployment, StatefulSet, DaemonSet

apiVersion: v1
kind: Pod
metadata:
  name: kube-controller-manager
  namespace: kube-system
spec:
  containers:
    - name: kube-controller-manager
      command:
        - kube-controller-manager
        - --allocate-node-cidrs=true
        - --bind-address=127.0.0.1
        - --cluster-cidr=192.168.0.0/16
        - --cluster-signing-cert-file=/etc/kubernetes/pki/ca.crt
        - --cluster-signing-key-file=/etc/kubernetes/pki/ca.key
        - --leader-elect=true
        - --node-monitor-period=5s
        - --node-monitor-grace-period=40s
        - --pod-eviction-timeout=5m
        - --use-service-account-credentials=true
```

---

## Step 215: kubelet

```yaml
# kubelet KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
address: 0.0.0.0
port: 10250
readOnlyPort: 0
authentication:
  anonymous:
    enabled: false
  webhook:
    enabled: true
  x509:
    clientCAFile: /etc/kubernetes/pki/ca.crt
authorization:
  mode: Webhook
cgroupDriver: systemd
clusterDNS:
  - 10.96.0.10
clusterDomain: cluster.local
evictionHard:
  memory.available: "100Mi"
  nodefs.available: "10%"
  imagefs.available: "15%"
maxPods: 110
protectKernelDefaults: true
rotateCertificates: true
staticPodPath: /etc/kubernetes/manifests
```

---

## Step 216: Container Runtime Interface (CRI)

```yaml
# RuntimeClass - เลือก container runtime
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: gvisor
handler: runsc  # gVisor runtime (sandboxed)

---
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: kata
handler: kata-fc  # Kata with Firecracker

---
# Pod ใช้ RuntimeClass
apiVersion: v1
kind: Pod
metadata:
  name: sandboxed-app
spec:
  runtimeClassName: gvisor
  containers:
    - name: app
      image: myapp:1.0.0
```

---

## Step 217: API Server Extensions

```yaml
# API Aggregation Layer
apiVersion: apiregistration.k8s.io/v1
kind: APIService
metadata:
  name: v1beta1.metrics.k8s.io
spec:
  service:
    name: metrics-server
    namespace: kube-system
    port: 443
  group: metrics.k8s.io
  version: v1beta1
  insecureSkipTLSVerify: true
  groupPriorityMinimum: 100
  versionPriority: 100
```

---

## Step 218: Admission Controllers

```yaml
# ValidatingAdmissionPolicy (CEL-based, K8s 1.28+)
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicy
metadata:
  name: "no-privileged-containers"
spec:
  failurePolicy: Fail
  matchConstraints:
    resourceRules:
      - apiGroups: [""]
        apiVersions: ["v1"]
        operations: ["CREATE", "UPDATE"]
        resources: ["pods"]
  validations:
    - expression: "!object.spec.containers.exists(c, c.securityContext != null && c.securityContext.privileged == true)"
      message: "Privileged containers are not allowed"
    - expression: "object.spec.containers.all(c, c.resources.limits != null && c.resources.limits.cpu != null)"
      message: "All containers must have CPU limits"

---
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicyBinding
metadata:
  name: "no-privileged-containers-binding"
spec:
  policyName: "no-privileged-containers"
  validationActions: [Deny]
  matchResources:
    namespaceSelector:
      matchLabels:
        environment: production
```

---

## Step 219: Kubernetes API Versioning

```yaml
# API Versioning
# Alpha: v1alpha1 - disabled by default
# Beta: v1beta1 - enabled by default
# GA: v1 - stable

# API Groups:
# Core: v1 (pods, services, configmaps)
# apps/v1 (deployments, statefulsets)
# batch/v1 (jobs, cronjobs)
# networking.k8s.io/v1 (ingress, networkpolicies)
# rbac.authorization.k8s.io/v1 (roles, clusterroles)

# ดู API resources
# kubectl api-resources
# kubectl api-versions
# kubectl explain pod.spec.containers.securityContext
```

---

## Step 220: Workshop - Architecture Audit

```python
#!/usr/bin/env python3
"""Kubernetes Architecture Audit Script"""
import kubernetes

kubernetes.config.load_kube_config()
v1 = kubernetes.client.CoreV1Api()
version_api = kubernetes.client.VersionApi()


def check_cluster_version():
    version = version_api.get_code()
    print(f"Server Version: {version.git_version}")
    print(f"Platform: {version.platform}")


def check_nodes():
    nodes = v1.list_node()
    print(f"\n=== Nodes ({len(nodes.items)}) ===")
    for node in nodes.items:
        name = node.metadata.name
        roles = [
            k.split("/")[1] for k in node.metadata.labels.keys()
            if k.startswith("node-role.kubernetes.io/")
        ]
        role = ",".join(roles) if roles else "worker"
        
        ready = False
        for cond in node.status.conditions:
            if cond.type == "Ready":
                ready = cond.status == "True"
        
        status = "Ready" if ready else "NotReady"
        kubelet_version = node.status.node_info.kubelet_version
        print(f"  {name} [{role}]: {status} - kubelet {kubelet_version}")
        
        alloc = node.status.allocatable
        print(f"    Allocatable: CPU={alloc.get('cpu')}, Memory={alloc.get('memory')}")


def check_etcd_pods():
    pods = v1.list_namespaced_pod("kube-system", label_selector="component=etcd")
    print(f"\n=== etcd Pods ({len(pods.items)}) ===")
    for pod in pods.items:
        print(f"  {pod.metadata.name}: {pod.status.phase}")


if __name__ == "__main__":
    print("=== Kubernetes Architecture Audit ===")
    check_cluster_version()
    check_nodes()
    check_etcd_pods()
    print("Audit complete!")
```

---

## 📊 สรุป Part 22

| Component | หน้าที่ |
|-----------|--------|
| kube-apiserver | RESTful API gateway, authn/authz |
| etcd | Distributed key-value store, cluster state |
| kube-scheduler | Pod-to-node assignment |
| kube-controller-manager | Control loop, reconciliation |
| kubelet | Pod management on nodes |
| CRI (containerd) | Container lifecycle management |
| kube-proxy | Network rules (iptables/IPVS) |
| Admission Controllers | Policy enforcement |

---

## 🔗 ต่อไป
- [Part 23: Pods Advanced](./part-23-pods-advanced.md)

---
*Part 22 | Steps 211-220 | ระดับสูง*
