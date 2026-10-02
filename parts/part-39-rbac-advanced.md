# Part 39: Kubernetes RBAC Advanced
## Steps 371-380: Role-Based Access Control Deep Dive

---

## 📖 บทนำ

RBAC (Role-Based Access Control) ควบคุม permission ใน Kubernetes โดยกำหนดว่า user/service account ใดสามารถทำอะไรกับ resource ใดได้บ้าง

> **หมายเหตุ**: เนื้อหาด้าน security ในหลักสูตรนี้มีไว้เพื่อการศึกษาและการทดสอบระบบที่ได้รับอนุญาตเท่านั้น

---

## Step 371: RBAC Architecture

```
RBAC Components:
┌─────────────────────────────────────────────────────────┐
│                                                         │
│  Subject          Binding           Role                │
│  ┌──────────┐    ┌──────────┐    ┌──────────────────┐  │
│  │ User     │    │ RoleB-   │    │ Role             │  │
│  │ Group    │───►│ inding   │───►│ (namespace)      │  │
│  │ Service  │    │ Cluster- │    │ ClusterRole      │  │
│  │ Account  │    │ RoleB-   │    │ (cluster-wide)   │  │
│  └──────────┘    │ inding   │    └──────────────────┘  │
│                  └──────────┘                           │
│                                                         │
│  Role:         verbs + resources                        │
│  ClusterRole:  cluster-scoped resources                 │
│  RoleBinding:  Subject + Role (namespace)               │
│  ClusterRB:    Subject + ClusterRole (cluster)          │
└─────────────────────────────────────────────────────────┘

API Verbs: get, list, watch, create, update, patch, delete, deletecollection
Resources: pods, deployments, services, secrets, configmaps, nodes, namespaces
```

---

## Step 372: Role และ ClusterRole

```yaml
# Role - namespace scoped
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: production
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["pods/log"]
    verbs: ["get"]
  - apiGroups: [""]
    resources: ["pods/exec"]
    verbs: []    # ไม่อนุญาต exec

---
# ClusterRole - cluster scoped
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-reader
rules:
  - apiGroups: [""]
    resources: ["nodes"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["namespaces"]
    verbs: ["get", "list"]
  - apiGroups: ["metrics.k8s.io"]
    resources: ["nodes", "pods"]
    verbs: ["get", "list"]

---
# ClusterRole สำหรับ developer
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: developer
rules:
  # Read pods/deployments/services
  - apiGroups: ["", "apps"]
    resources: ["pods", "deployments", "replicasets", "services", "endpoints"]
    verbs: ["get", "list", "watch"]
  # View logs
  - apiGroups: [""]
    resources: ["pods/log"]
    verbs: ["get"]
  # Forward ports
  - apiGroups: [""]
    resources: ["pods/portforward"]
    verbs: ["create"]
  # Read configmaps (ไม่ใช่ secrets)
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get", "list", "watch"]
  # Read ingress
  - apiGroups: ["networking.k8s.io"]
    resources: ["ingresses"]
    verbs: ["get", "list", "watch"]
  # View events
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["get", "list", "watch"]
```

---

## Step 373: RoleBinding และ ClusterRoleBinding

```yaml
# RoleBinding - bind user to Role in namespace
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: production
subjects:
  - kind: User
    name: john@example.com
    apiGroup: rbac.authorization.k8s.io
  - kind: Group
    name: developers
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io

---
# ClusterRoleBinding - bind to ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cluster-admin-binding
subjects:
  - kind: User
    name: admin@example.com
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io

---
# RoleBinding ใช้ ClusterRole (namespace-scoped)
# ClusterRole developer ถูก bind เฉพาะใน namespace production
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: developer-production
  namespace: production
subjects:
  - kind: Group
    name: developers
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: developer
  apiGroup: rbac.authorization.k8s.io
```

---

## Step 374: ServiceAccount RBAC

```yaml
# ServiceAccount สำหรับ application
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app
  namespace: production
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789:role/my-app-role  # IRSA

---
# Role สำหรับ app ที่ต้องการ read configmaps
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: configmap-reader
  namespace: production
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get", "list", "watch"]
    resourceNames: ["app-config", "feature-flags"]  # specific resources

---
# Bind ServiceAccount to Role
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: my-app-configmap
  namespace: production
subjects:
  - kind: ServiceAccount
    name: my-app
    namespace: production
roleRef:
  kind: Role
  name: configmap-reader
  apiGroup: rbac.authorization.k8s.io

---
# Pod ใช้ ServiceAccount
apiVersion: v1
kind: Pod
metadata:
  name: my-app
  namespace: production
spec:
  serviceAccountName: my-app
  automountServiceAccountToken: false  # disable ถ้าไม่ต้องการ K8s API
  containers:
    - name: app
      image: myregistry.io/app:1.0
```

---

## Step 375: RBAC Audit

```yaml
# API server audit policy - log RBAC events
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  # Log ทุก request ที่เกี่ยวกับ RBAC
  - level: RequestResponse
    verbs: ["create", "update", "patch", "delete"]
    resources:
      - group: "rbac.authorization.k8s.io"
        resources:
          - roles
          - rolebindings
          - clusterroles
          - clusterrolebindings
  
  # Log secret access
  - level: Metadata
    resources:
      - group: ""
        resources: ["secrets"]
  
  # Log pods/exec
  - level: RequestResponse
    resources:
      - group: ""
        resources: ["pods/exec", "pods/attach"]
  
  # Log ทุก authentication failure
  - level: Metadata
    omitStages: ["RequestReceived"]
    users: ["system:anonymous"]
  
  # Default: log metadata only
  - level: Metadata
    omitStages: ["RequestReceived"]
```

---

## Step 376: Least Privilege Patterns

```yaml
# Pattern: Application-specific roles
# Principle: กำหนด permission น้อยที่สุดที่จำเป็น

# Operator role - สามารถ create/update deployments
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: deployment-operator
rules:
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
  - apiGroups: ["apps"]
    resources: ["replicasets"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]

---
# CI/CD role - deploy applications
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cicd-deployer
rules:
  - apiGroups: ["apps"]
    resources: ["deployments", "statefulsets", "daemonsets"]
    verbs: ["get", "list", "create", "update", "patch"]
  - apiGroups: [""]
    resources: ["services", "configmaps"]
    verbs: ["get", "list", "create", "update", "patch"]
  - apiGroups: ["networking.k8s.io"]
    resources: ["ingresses"]
    verbs: ["get", "list", "create", "update", "patch"]
  # ห้าม delete หรือ access secrets โดยตรง

---
# Monitoring role - read-only metrics
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: monitoring-reader
rules:
  - apiGroups: [""]
    resources: ["pods", "nodes", "endpoints", "services"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["metrics.k8s.io"]
    resources: ["pods", "nodes"]
    verbs: ["get", "list"]
  - nonResourceURLs: ["/metrics", "/healthz", "/readyz"]
    verbs: ["get"]
```

---

## Step 377: RBAC for Custom Resources

```yaml
# Role สำหรับ CRD resources
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: application-operator
rules:
  # Custom resources
  - apiGroups: ["app.example.com"]
    resources: ["applications", "databases", "caches"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  - apiGroups: ["app.example.com"]
    resources: ["applications/status", "applications/scale"]
    verbs: ["get", "update", "patch"]
  
  # Manage standard resources
  - apiGroups: ["apps"]
    resources: ["deployments", "statefulsets"]
    verbs: ["get", "list", "create", "update", "patch", "delete"]
  - apiGroups: [""]
    resources: ["services", "configmaps", "persistentvolumeclaims"]
    verbs: ["get", "list", "create", "update", "patch", "delete"]
  
  # Watch events
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["create", "patch"]
```

---

## Step 378: RBAC Security Patterns

```yaml
# Anti-pattern: อย่าให้ wildcards
# ❌ ไม่ดี
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: bad-role
rules:
  - apiGroups: ["*"]
    resources: ["*"]
    verbs: ["*"]

---
# ✅ ดี - specific permissions
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: good-role
rules:
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "watch"]

---
# Impersonation - อนุญาตให้ act as another user
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: limited-impersonator
rules:
  - apiGroups: [""]
    resources: ["users"]
    verbs: ["impersonate"]
    resourceNames: ["test-user"]  # เฉพาะ test-user เท่านั้น
```

---

## Step 379: Multi-tenant RBAC

```yaml
# Namespace สำหรับแต่ละ team
# ClusterRole ที่ใช้ร่วมกัน
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: namespace-admin
rules:
  - apiGroups: ["", "apps", "batch", "extensions"]
    resources:
      - pods
      - deployments
      - services
      - configmaps
      - jobs
      - cronjobs
      - ingresses
      - horizontalpodautoscalers
    verbs: ["*"]
  - apiGroups: ["networking.k8s.io"]
    resources: ["ingresses", "networkpolicies"]
    verbs: ["*"]
  # ไม่มี access to nodes, namespaces, clusterroles

---
# Bind team-a ใน namespace team-a เท่านั้น
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: team-a-admin
  namespace: team-a
subjects:
  - kind: Group
    name: team-a
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: namespace-admin
  apiGroup: rbac.authorization.k8s.io

---
# LimitRange ต่อ namespace
apiVersion: v1
kind: LimitRange
metadata:
  name: team-a-limits
  namespace: team-a
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

---
# ResourceQuota ต่อ namespace
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-a-quota
  namespace: team-a
spec:
  hard:
    requests.cpu: "10"
    requests.memory: 20Gi
    limits.cpu: "20"
    limits.memory: 40Gi
    pods: "50"
    services: "20"
    persistentvolumeclaims: "10"
```

---

## Step 380: Workshop - RBAC Hardening

```yaml
# Workshop: ออกแบบ RBAC สำหรับ microservices platform

# 1. Platform admin
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: platform-admins
subjects:
  - kind: Group
    name: platform-admins@company.com
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io

---
# 2. Team developers (namespace-scoped)
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: team-developer
rules:
  - apiGroups: ["", "apps", "autoscaling"]
    resources: ["pods", "deployments", "services", "configmaps", "horizontalpodautoscalers"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
  - apiGroups: [""]
    resources: ["pods/log", "pods/portforward"]
    verbs: ["get", "create"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["get", "list", "watch"]

---
# 3. Read-only viewer
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: team-viewer
rules:
  - apiGroups: ["", "apps", "batch"]
    resources: ["*"]
    verbs: ["get", "list", "watch"]

---
# 4. Service accounts สำหรับ CI/CD
apiVersion: v1
kind: ServiceAccount
metadata:
  name: github-actions
  namespace: ci-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: ci-deployer
rules:
  - apiGroups: ["apps"]
    resources: ["deployments", "statefulsets"]
    verbs: ["get", "list", "create", "update", "patch"]
  - apiGroups: [""]
    resources: ["services", "configmaps"]
    verbs: ["get", "list", "create", "update", "patch"]
  - apiGroups: ["batch"]
    resources: ["jobs"]
    verbs: ["get", "list", "create", "delete"]
```

---

## 📊 สรุป Part 39

| Concept | รายละเอียด |
|---------|----------|
| Role | Permission ใน namespace |
| ClusterRole | Permission cluster-wide |
| RoleBinding | Subject + Role (namespace) |
| ClusterRoleBinding | Subject + ClusterRole (cluster) |
| ServiceAccount | Identity สำหรับ pods |
| Least Privilege | ให้ permission น้อยที่สุด |
| ResourceNames | จำกัด access เฉพาะ resources |

---

## 🔗 ต่อไป
- [Part 40: Kubernetes Operators](./part-40-operators.md)

---
*Part 39 | Steps 371-380 | ระดับสูง*
