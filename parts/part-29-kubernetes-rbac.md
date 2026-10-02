# Part 29: Kubernetes RBAC - Role-Based Access Control
## Steps 281-290: การจัดการสิทธิ์ใน Kubernetes

---

## 📖 บทนำ

RBAC (Role-Based Access Control) คือระบบควบคุมการเข้าถึงใน Kubernetes ที่กำหนดว่า "ใคร" สามารถทำ "อะไร" กับ "resource อะไร"

### RBAC = Subject + Verb + Resource
```
Alice (User)     → get/list  → pods         ใน namespace my-app
CI System (SA)   → create    → deployments  ใน namespace staging
Monitor (SA)     → get       → nodes        ทั้ง cluster
```

---

## Step 281: RBAC Components

### 4 Object ของ RBAC
```
Role              → สิทธิ์ใน namespace เดียว
ClusterRole       → สิทธิ์ทั้ง cluster
RoleBinding       → ผูก Role กับ Subject ใน namespace
ClusterRoleBinding → ผูก ClusterRole กับ Subject ทั้ง cluster
```

### Subjects
```yaml
# 1. User
subjects:
  - kind: User
    name: alice@example.com
    apiGroup: rbac.authorization.k8s.io

# 2. Group
subjects:
  - kind: Group
    name: system:masters
    apiGroup: rbac.authorization.k8s.io

# 3. ServiceAccount
subjects:
  - kind: ServiceAccount
    name: my-app-sa
    namespace: my-app
```

---

## Step 282: Verbs ที่ใช้ได้

```yaml
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs:
      - get
      - list
      - watch
      - create
      - update
      - patch
      - delete
      - deletecollection
      - "*"   # ทุก verbs (ระวัง!)

# Sub-resources
rules:
  - apiGroups: [""]
    resources: 
      - pods
      - pods/log
      - pods/exec
      - pods/portforward
      - pods/attach
      - pods/status
    verbs: ["get", "list"]
```

---

## Step 283: Resources และ API Groups

```yaml
rules:
  - apiGroups: [""]    # Core group
    resources:
      - pods
      - services
      - endpoints
      - namespaces
      - nodes
      - configmaps
      - secrets
      - persistentvolumes
      - persistentvolumeclaims
      - serviceaccounts

  - apiGroups: ["apps"]
    resources:
      - deployments
      - replicasets
      - statefulsets
      - daemonsets

  - apiGroups: ["batch"]
    resources: ["jobs", "cronjobs"]
  
  - apiGroups: ["networking.k8s.io"]
    resources: ["ingresses", "networkpolicies"]
  
  - apiGroups: ["rbac.authorization.k8s.io"]
    resources: ["roles", "rolebindings", "clusterroles", "clusterrolebindings"]
  
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses", "persistentvolumes"]
```

---

## Step 284: ตัวอย่าง Roles ที่ใช้บ่อย

### Read-Only Role
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: read-only
rules:
  - apiGroups: [""]
    resources: ["pods", "services", "configmaps", "endpoints", "events"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["apps"]
    resources: ["deployments", "replicasets", "statefulsets", "daemonsets"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["batch"]
    resources: ["jobs", "cronjobs"]
    verbs: ["get", "list", "watch"]
```

### Developer Role
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: developer
rules:
  - apiGroups: ["", "apps", "batch"]
    resources: ["pods", "services", "configmaps", "deployments", "replicasets", "jobs"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  
  # Read-only secrets
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get", "list", "watch"]
  
  # Logs และ exec
  - apiGroups: [""]
    resources: ["pods/log", "pods/exec", "pods/portforward"]
    verbs: ["get", "create"]
```

### CI/CD Pipeline Role
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: ci-deployer
  namespace: my-app
rules:
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
  
  - apiGroups: [""]
    resources: ["services", "configmaps"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
  
  - apiGroups: ["apps"]
    resources: ["deployments/scale"]
    verbs: ["patch"]
```

### Monitoring Role
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: prometheus-monitoring
rules:
  - apiGroups: [""]
    resources:
      - nodes
      - nodes/proxy
      - nodes/metrics
      - services
      - endpoints
      - pods
    verbs: ["get", "list", "watch"]
  
  - nonResourceURLs:
      - /metrics
      - /metrics/cadvisor
    verbs: ["get"]
```

---

## Step 285: ServiceAccount สำหรับ Pods

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app-sa
  namespace: my-app
  annotations:
    # AWS IRSA
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789:role/my-app-role
    
    # GKE Workload Identity
    iam.gke.io/gcp-service-account: my-app@project.iam.gserviceaccount.com
    
  labels:
    app: my-app
automountServiceAccountToken: false
```

```yaml
# ใช้ SA ใน Pod
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  template:
    spec:
      serviceAccountName: my-app-sa
      automountServiceAccountToken: false
      containers:
        - name: app
          image: my-app:latest
```

```yaml
# Role Binding
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: my-app-binding
  namespace: my-app
subjects:
  - kind: ServiceAccount
    name: my-app-sa
    namespace: my-app
roleRef:
  kind: Role
  name: ci-deployer
  apiGroup: rbac.authorization.k8s.io
```

---

## Step 286: Built-in ClusterRoles

```bash
kubectl get clusterroles | grep -v system:

# สำคัญที่สุด
# cluster-admin     → ทำได้ทุกอย่าง - ระวัง!
# admin             → admin สำหรับ namespace
# edit              → read/write ยกเว้น RBAC
# view              → read-only
```

```yaml
# ให้ user alice มีสิทธิ์ view ใน namespace my-app
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: alice-view
  namespace: my-app
subjects:
  - kind: User
    name: alice@example.com
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: view
  apiGroup: rbac.authorization.k8s.io
```

---

## Step 287: Advanced RBAC Patterns

### Resource Names (Fine-grained)
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: specific-secret-reader
  namespace: my-app
rules:
  - apiGroups: [""]
    resources: ["secrets"]
    resourceNames:
      - "app-secret"
      - "db-credentials"
    verbs: ["get"]
```

### Non-Resource URLs
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: metrics-reader
rules:
  - nonResourceURLs:
      - "/metrics"
      - "/healthz"
      - "/readyz"
    verbs: ["get"]
```

---

## Step 288: RBAC Best Practices

### Principle of Least Privilege
```yaml
# ❌ ไม่ดี - ให้สิทธิ์มากเกินจำเป็น
rules:
  - apiGroups: ["*"]
    resources: ["*"]
    verbs: ["*"]

# ✅ ดี - เจาะจงเฉพาะที่จำเป็น
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list"]
```

### Avoid Granting These Dangerous Permissions
```yaml
# ⚠️ สิทธิ์ที่อันตราย:
# 1. create pods     → อาจสร้าง privileged pods
# 2. pods/exec       → command execution
# 3. pods/portforward → access internal services
# 4. create/update rolebindings → RBAC escalation
# 5. get/list secrets → credential access
# 6. impersonate     → act as other users
# 7. escalate        → increase own permissions
# 8. bind            → create new role bindings
```

### Review Existing Permissions
```bash
#!/bin/bash

echo "=== Cluster Admin Bindings ==="
kubectl get clusterrolebindings -o json | jq -r '
  .items[] |
  select(.roleRef.name == "cluster-admin") |
  "\(.metadata.name): " + (.subjects[]? | "\(.kind)/\(.namespace)/\(.name)")'

echo ""
echo "=== Wildcard Permissions ==="
kubectl get clusterroles -o json | jq -r '
  .items[] |
  . as $role |
  .rules[]? |
  select(.apiGroups[]? == "*" or .resources[]? == "*" or .verbs[]? == "*") |
  $role.metadata.name'
```

---

## Step 289: RBAC ใน Real-world Scenario

### Multi-team Setup
```yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: frontend-dev
  labels:
    team: frontend
    environment: dev

---
# frontend team - read/write in dev
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: frontend-team-dev
  namespace: frontend-dev
subjects:
  - kind: Group
    name: frontend-team
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: edit
  apiGroup: rbac.authorization.k8s.io

---
# frontend team - read-only in staging
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: frontend-team-staging
  namespace: frontend-staging
subjects:
  - kind: Group
    name: frontend-team
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: view
  apiGroup: rbac.authorization.k8s.io
```

---

## Step 290: RBAC Debugging

```bash
# ตรวจสิทธิ์ของตัวเอง
kubectl auth can-i get pods
kubectl auth can-i create deployments --namespace=production
kubectl auth can-i --list
kubectl auth can-i --list --namespace=my-app

# ตรวจสิทธิ์ของ user/sa อื่น
kubectl auth can-i get pods --as=alice@example.com
kubectl auth can-i get pods --as=system:serviceaccount:default:my-sa
kubectl auth can-i --list --as=system:serviceaccount:kube-system:default

# Common Issues:
# Issue 1: "forbidden: User cannot get resource"
# → ตรวจ RoleBinding/ClusterRoleBinding
kubectl get rolebindings,clusterrolebindings -A | grep <username>

# Issue 2: SA ไม่มีสิทธิ์แม้จะ bind แล้ว
# → ตรวจ namespace ใน binding
kubectl describe rolebinding <name> -n <namespace>
```

### RBAC Audit Script
```python
#!/usr/bin/env python3
import subprocess
import json

def get_clusterrolebindings():
    result = subprocess.run(
        ["kubectl", "get", "clusterrolebindings", "-o", "json"],
        capture_output=True, text=True
    )
    return json.loads(result.stdout)

def find_cluster_admins(bindings):
    admins = []
    for binding in bindings['items']:
        if binding['roleRef']['name'] == 'cluster-admin':
            subjects = binding.get('subjects', [])
            admins.append({
                'binding': binding['metadata']['name'],
                'subjects': subjects
            })
    return admins

def find_wildcard_roles():
    result = subprocess.run(
        ["kubectl", "get", "clusterroles", "-o", "json"],
        capture_output=True, text=True
    )
    roles_data = json.loads(result.stdout)
    
    wildcards = []
    for role in roles_data['items']:
        if role['metadata']['name'].startswith('system:'):
            continue
        
        for rule in role.get('rules', []):
            if '*' in rule.get('apiGroups', []) or \
               '*' in rule.get('resources', []) or \
               '*' in rule.get('verbs', []):
                wildcards.append(role['metadata']['name'])
                break
    
    return wildcards

if __name__ == "__main__":
    print("=== RBAC Audit Report ===\n")
    
    bindings = get_clusterrolebindings()
    admins = find_cluster_admins(bindings)
    
    print(f"Cluster Admin Bindings: {len(admins)}")
    for admin in admins:
        print(f"  - {admin['binding']}")
        for subject in admin['subjects']:
            print(f"    → {subject['kind']}/{subject.get('namespace', 'cluster')}/{subject['name']}")
    
    print(f"\nWildcard ClusterRoles:")
    wildcards = find_wildcard_roles()
    for role in wildcards:
        print(f"  - {role}")
```

---

## 📊 สรุป Part 29

| หัวข้อ | สิ่งที่เรียน |
|--------|------------|
| RBAC Components | Role, ClusterRole, RoleBinding, ClusterRoleBinding |
| Subjects | User, Group, ServiceAccount |
| Verbs | get, list, create, update, delete, etc. |
| API Groups | "", apps, batch, networking.k8s.io |
| Built-in Roles | cluster-admin, admin, edit, view |
| Best Practices | Least privilege, namespace isolation |
| Real-world | Multi-team, GitOps scenarios |
| Debugging | auth can-i, audit logs |

---

## 🔗 ต่อไป
- [Part 30: Namespaces และ Resource Quotas](./part-30-kubernetes-namespaces.md)

---
*Part 29 | Steps 281-290 | ระดับกลาง*
