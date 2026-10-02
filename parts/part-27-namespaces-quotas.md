# Part 27: Namespaces และ Resource Quotas
## Steps 261-270: การแบ่ง Cluster และจัดการ Resources

---

## 📖 บทนำ

Namespaces เป็นกลไกในการแบ่ง Kubernetes cluster ออกเป็น virtual clusters สำหรับ multi-team, multi-environment

---

## Step 261: Namespace Design

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    environment: production
    team: platform
    cost-center: cc-001
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: latest
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/audit: restricted
  annotations:
    owner: "platform-team@company.com"
    slack-channel: "#platform-alerts"

---
apiVersion: v1
kind: Namespace
metadata:
  name: monitoring
  labels:
    environment: shared
    team: platform
    pod-security.kubernetes.io/enforce: privileged
```

---

## Step 262: ResourceQuota

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: production-quota
  namespace: production
spec:
  hard:
    requests.cpu: "20"
    requests.memory: "40Gi"
    limits.cpu: "40"
    limits.memory: "80Gi"
    pods: "100"
    services: "30"
    secrets: "50"
    configmaps: "50"
    persistentvolumeclaims: "20"
    services.loadbalancers: "3"
    services.nodeports: "0"
    requests.storage: "500Gi"
    fast-ssd.storageclass.storage.k8s.io/requests.storage: "200Gi"

---
# ดู quota usage
# kubectl describe quota -n production
```

---

## Step 263: LimitRange

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: production-limits
  namespace: production
spec:
  limits:
    - type: Container
      default:
        cpu: "500m"
        memory: "512Mi"
      defaultRequest:
        cpu: "100m"
        memory: "128Mi"
      max:
        cpu: "4"
        memory: "8Gi"
      min:
        cpu: "50m"
        memory: "64Mi"
      maxLimitRequestRatio:
        cpu: "10"
        memory: "4"
    - type: Pod
      max:
        cpu: "8"
        memory: "16Gi"
    - type: PersistentVolumeClaim
      max:
        storage: 50Gi
      min:
        storage: 1Gi
```

---

## Step 264: Network Isolation

```yaml
# Default deny all
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
# Allow DNS
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

---
# Allow from monitoring
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-monitoring
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: monitoring
      ports:
        - port: 9090
        - port: 8080
```

---

## Step 265: RBAC per Namespace

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app
  namespace: production
automountServiceAccountToken: false

---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: app-developer
  namespace: production
rules:
  - apiGroups: [""]
    resources: ["pods", "pods/log"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: app-developer-binding
  namespace: production
subjects:
  - kind: User
    name: john.doe@company.com
    apiGroup: rbac.authorization.k8s.io
  - kind: Group
    name: dev-team
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: app-developer
  apiGroup: rbac.authorization.k8s.io
```

---

## Step 266: Namespace Labels for Policy

```yaml
# Istio injection
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    istio-injection: enabled

---
# Gatekeeper constraint
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredLabels
metadata:
  name: require-team-label
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Namespace"]
    excludedNamespaces:
      - kube-system
      - kube-public
  parameters:
    labels: ["team", "environment"]
```

---

## Step 267: Multi-tenancy

```yaml
# ตัวอย่าง Soft multi-tenancy

# Tenant namespace
apiVersion: v1
kind: Namespace
metadata:
  name: tenant-acme
  labels:
    tenant: acme
    environment: production

---
# Tenant ResourceQuota
apiVersion: v1
kind: ResourceQuota
metadata:
  name: acme-quota
  namespace: tenant-acme
spec:
  hard:
    requests.cpu: "10"
    requests.memory: "20Gi"
    pods: "50"

---
# Tenant network isolation
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: tenant-isolation
  namespace: tenant-acme
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              tenant: acme
  egress:
    - ports:
        - port: 53
          protocol: UDP
```

---

## Step 268-270: Workshop - Namespace Provisioner

```python
#!/usr/bin/env python3
"""Namespace Provisioner Script"""
import kubernetes
from kubernetes import client, config
from typing import Optional

config.load_kube_config()
v1 = client.CoreV1Api()


def create_namespace(name: str, labels: dict, annotations: Optional[dict] = None):
    namespace = client.V1Namespace(
        metadata=client.V1ObjectMeta(
            name=name,
            labels=labels,
            annotations=annotations or {}
        )
    )
    try:
        v1.create_namespace(namespace)
        print(f"✅ Created namespace: {name}")
    except client.exceptions.ApiException as e:
        if e.status == 409:
            print(f"ℹ️  Namespace already exists: {name}")
        else:
            raise


def create_resource_quota(namespace: str, cpu_limit: str, memory_limit: str, pod_limit: int):
    quota = client.V1ResourceQuota(
        metadata=client.V1ObjectMeta(
            name=f"{namespace}-quota",
            namespace=namespace
        ),
        spec=client.V1ResourceQuotaSpec(
            hard={
                "requests.cpu": cpu_limit,
                "limits.cpu": cpu_limit,
                "requests.memory": memory_limit,
                "limits.memory": memory_limit,
                "pods": str(pod_limit),
                "services.nodeports": "0"
            }
        )
    )
    v1.create_namespaced_resource_quota(namespace, quota)
    print(f"✅ Created ResourceQuota for: {namespace}")


def provision_namespace(name: str, environment: str, team: str,
                        cpu_limit: str = "10", memory_limit: str = "20Gi",
                        pod_limit: int = 50):
    print(f"\n🚀 Provisioning namespace: {name}")
    labels = {
        "environment": environment,
        "team": team,
        "pod-security.kubernetes.io/enforce": "restricted",
    }
    annotations = {"team-contact": f"{team}@company.com"}
    
    create_namespace(name, labels, annotations)
    create_resource_quota(name, cpu_limit, memory_limit, pod_limit)
    print(f"✅ Namespace {name} provisioned successfully!")


if __name__ == "__main__":
    provision_namespace("production", "production", "platform",
                        cpu_limit="40", memory_limit="80Gi", pod_limit=200)
    provision_namespace("staging", "staging", "platform",
                        cpu_limit="20", memory_limit="40Gi", pod_limit=100)
    provision_namespace("development", "development", "platform",
                        cpu_limit="10", memory_limit="20Gi", pod_limit=50)
```

---

## 📊 สรุป Part 27

| Feature | หน้าที่ |
|---------|--------|
| Namespace | Virtual cluster isolation |
| ResourceQuota | Limit resource consumption |
| LimitRange | Default limits and min/max |
| NetworkPolicy | Network isolation |
| RBAC per NS | Access control |
| Multi-tenancy | Share cluster |

---

## 🔗 ต่อไป
- [Part 28: Ingress และ Service Mesh](./part-28-ingress-service-mesh.md)

---
*Part 27 | Steps 261-270 | ระดับสูง*
