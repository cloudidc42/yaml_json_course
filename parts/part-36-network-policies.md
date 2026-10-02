# Part 36: Network Policies Deep Dive
## Steps 341-350: Zero-Trust Networking ใน Kubernetes

---

## 📖 บทนำ

Network Policies เป็นกลไก control plane level สำหรับกำหนด traffic rules ระหว่าง pods โดยใช้ label selectors และ namespace selectors

> **หมายเหตุด้าน Security**: เนื้อหาด้าน security ในหลักสูตรนี้มีไว้เพื่อการศึกษาและการทดสอบระบบที่ได้รับอนุญาตเท่านั้น การนำไปใช้โจมตีระบบที่ไม่ได้รับอนุญาตเป็นสิ่งผิดกฎหมาย

---

## Step 341: Default Deny All

```yaml
# Zero-trust starting point
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
  name: allow-dns-egress
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53

---
# Allow same-namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-same-namespace
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector: {}
```

---

## Step 342: 3-Tier Architecture

```yaml
# Frontend: รับ traffic จาก ingress
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-netpol
  namespace: production
spec:
  podSelector:
    matchLabels:
      tier: frontend
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: ingress-nginx
      ports:
        - protocol: TCP
          port: 80
  egress:
    - to:
        - podSelector:
            matchLabels:
              tier: backend
      ports:
        - protocol: TCP
          port: 8080
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
      ports:
        - protocol: UDP
          port: 53

---
# Backend
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-netpol
  namespace: production
spec:
  podSelector:
    matchLabels:
      tier: backend
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              tier: frontend
      ports:
        - protocol: TCP
          port: 8080
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: monitoring
        - podSelector:
            matchLabels:
              app: prometheus
      ports:
        - protocol: TCP
          port: 9090
  egress:
    - to:
        - podSelector:
            matchLabels:
              tier: database
      ports:
        - protocol: TCP
          port: 5432

---
# Database: รับ traffic จาก backend เท่านั้น
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: database-netpol
  namespace: production
spec:
  podSelector:
    matchLabels:
      tier: database
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              tier: backend
      ports:
        - protocol: TCP
          port: 5432
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
      ports:
        - protocol: UDP
          port: 53
```

---

## Step 343: Cross-Namespace Policies

```yaml
# Allow monitoring scrape
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-prometheus-scrape
  namespace: production
spec:
  podSelector:
    matchLabels:
      prometheus: scrape
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: monitoring
        - podSelector:
            matchLabels:
              app.kubernetes.io/name: prometheus
      ports:
        - protocol: TCP
          port: 9090

---
# Cross-namespace AND condition
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-auth-service
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: my-app
  policyTypes:
    - Egress
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              team: auth-team
          podSelector:
            matchLabels:
              app: auth-service
      ports:
        - protocol: TCP
          port: 8080
```

---

## Step 344: External Traffic Control

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-external-apis
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: payment-service
  policyTypes:
    - Egress
  egress:
    - to:
        - ipBlock:
            cidr: 54.187.174.169/32
      ports:
        - protocol: TCP
          port: 443
    - to:
        - ipBlock:
            cidr: 52.94.0.0/16
            except:
              - 52.94.5.0/24
      ports:
        - protocol: TCP
          port: 443

---
# Block metadata service
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: block-metadata
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - to:
        - ipBlock:
            cidr: 0.0.0.0/0
            except:
              - 169.254.169.254/32  # AWS metadata
              - 169.254.170.2/32    # AWS ECS metadata
```

---

## Step 345: Calico GlobalNetworkPolicy

```yaml
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: deny-all-external-egress
spec:
  order: 1000
  selector: all()
  types:
    - Egress
  egress:
    - action: Allow
      protocol: UDP
      destination:
        ports: [53]
    - action: Allow
      destination:
        nets:
          - 10.0.0.0/8
          - 172.16.0.0/12
          - 192.168.0.0/16
    - action: Deny

---
apiVersion: projectcalico.org/v3
kind: NetworkPolicy
metadata:
  name: allow-backend-to-db
  namespace: production
spec:
  tier: application
  order: 100
  selector: tier == 'database'
  types:
    - Ingress
  ingress:
    - action: Allow
      source:
        selector: tier == 'backend'
      protocol: TCP
      destination:
        ports: [5432]
    - action: Deny
```

---

## Step 346: Cilium L7 Policy

```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: api-http-policy
  namespace: production
spec:
  endpointSelector:
    matchLabels:
      app: api-service
  
  ingress:
    - fromEndpoints:
        - matchLabels:
            app: frontend
      toPorts:
        - ports:
            - port: "8080"
              protocol: TCP
          rules:
            http:
              - method: GET
                path: "^/api/v1/products.*"
              - method: GET
                path: "^/api/v1/products/[0-9]+"
    
    - fromEndpoints:
        - matchLabels:
            role: admin
      toPorts:
        - ports:
            - port: "8080"
              protocol: TCP
          rules:
            http:
              - method: POST
                path: "^/api/v1/.*"
              - method: PUT
                path: "^/api/v1/.*"

---
# Cilium DNS-based
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: allow-external-dns
  namespace: production
spec:
  endpointSelector:
    matchLabels:
      app: backend
  egress:
    - toFQDNs:
        - matchName: "api.stripe.com"
        - matchPattern: "*.amazonaws.com"
      toPorts:
        - ports:
            - port: "443"
              protocol: TCP
    - toEndpoints:
        - matchLabels:
            "k8s:io.kubernetes.pod.namespace": kube-system
      toPorts:
        - ports:
            - port: "53"
              protocol: ANY
          rules:
            dns:
              - matchPattern: "*"
```

---

## Step 347: Testing

```python
import subprocess

def test_connectivity(
    source_pod: str,
    source_namespace: str,
    target_host: str,
    target_port: int,
    expected_success: bool
) -> bool:
    cmd = [
        "kubectl", "exec", source_pod,
        "-n", source_namespace, "--",
        "nc", "-zv", "-w", "3", target_host, str(target_port)
    ]
    result = subprocess.run(cmd, capture_output=True, text=True, timeout=10)
    success = result.returncode == 0
    passed = success == expected_success
    status = "✅ PASS" if passed else "❌ FAIL"
    print(f"{status}: {source_pod} -> {target_host}:{target_port}")
    return passed

def run_tests(namespace: str):
    tests = [
        ("frontend-pod", namespace, "backend-service", 8080, True),
        ("frontend-pod", namespace, "postgres-service", 5432, False),
        ("backend-pod", namespace, "postgres-service", 5432, True),
        ("database-pod", namespace, "8.8.8.8", 53, False),
    ]
    results = [test_connectivity(*t) for t in tests]
    print(f"\nResults: {sum(results)}/{len(results)} passed")
```

---

## Step 348-349: Namespace Isolation

```yaml
# Template สำหรับ namespace isolation
# ใช้ configmap + init container สร้างอัตโนมัติ policies

apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-shared-services
  namespace: team-alpha
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: shared-data
          podSelector:
            matchLabels:
              app: postgres
      ports:
        - protocol: TCP
          port: 5432
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: shared-data
          podSelector:
            matchLabels:
              app: redis
      ports:
        - protocol: TCP
          port: 6379
```

---

## Step 350: Workshop

```yaml
# Complete zero-trust
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-ingress-traffic
  namespace: production
spec:
  podSelector:
    matchLabels:
      expose: external
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: ingress-nginx
      ports:
        - protocol: TCP
          port: 8080

---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: block-cloud-metadata
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - to:
        - ipBlock:
            cidr: 0.0.0.0/0
            except:
              - 169.254.169.254/32
```

---

## 📊 สรุป Part 36

| Concept | รายละเอียด |
|---------|----------|
| Default Deny | Zero-trust baseline |
| Ingress Policy | Control incoming traffic |
| Egress Policy | Control outgoing traffic |
| ipBlock | External IP ranges |
| Calico | Global policies, tiers |
| Cilium L7 | HTTP-level policies |

---

## 🔗 ต่อไป
- [Part 37: Admission Controllers](./part-37-admission-controllers.md)

---
*Part 36 | Steps 341-350 | ระดับสูง*
