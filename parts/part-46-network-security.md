# Part 46: Network Security
## Steps 441-450: Kubernetes Network Security Deep Dive

---

## 📖 บทนำ

Network security ใน Kubernetes ครอบคลุม NetworkPolicies, service mesh security, TLS everywhere, ingress security และการป้องกัน lateral movement ระหว่าง pods

> **หมายเหตุ**: เนื้อหาด้าน security ในหลักสูตรนี้มีไว้เพื่อการศึกษาและการทดสอบระบบที่ได้รับอนุญาตเท่านั้น การนำไปใช้โจมตีระบบที่ไม่ได้รับอนุญาตเป็นสิ่งผิดกฎหมาย

---

## Step 441: Network Security Overview

```
Kubernetes Network Security Model:
┌─────────────────────────────────────────────────────────────┐
│                   Internet                                  │
│                      │                                      │
│            ┌─────────▼─────────┐                           │
│            │   Load Balancer    │                           │
│            │   (WAF/DDoS)       │                           │
│            └─────────┬─────────┘                           │
│                      │ HTTPS                                │
│            ┌─────────▼─────────┐                           │
│            │     Ingress        │                           │
│            │  (TLS termination) │                           │
│            └─────────┬─────────┘                           │
│                      │ mTLS                                 │
│  ┌───────────────────▼──────────────────────┐             │
│  │              Service Mesh                  │             │
│  │  (Istio/Linkerd - mTLS, policy, observ.)  │             │
│  │                                            │             │
│  │  ┌────────┐  NetworkPolicy  ┌────────┐    │             │
│  │  │  Pod A │◄───────────────►│  Pod B │    │             │
│  │  └────────┘                 └────────┘    │             │
│  └────────────────────────────────────────────┘             │
└─────────────────────────────────────────────────────────────┘
```

---

## Step 442: Default Deny NetworkPolicy

```yaml
# Default deny all ingress
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Ingress

---
# Default deny all egress
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-egress
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Egress

---
# Allow DNS (จำเป็นสำหรับ name resolution)
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
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
      ports:
        - port: 53
          protocol: UDP
        - port: 53
          protocol: TCP
```

---

## Step 443: 3-Tier Architecture Policies

```yaml
# Frontend NetworkPolicy
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-policy
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
        - port: 8080
  egress:
    - to:
        - podSelector:
            matchLabels:
              tier: backend
      ports:
        - port: 8080
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
      ports:
        - port: 53
          protocol: UDP

---
# Backend NetworkPolicy
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-policy
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
        - port: 8080
  egress:
    - to:
        - podSelector:
            matchLabels:
              tier: database
      ports:
        - port: 5432
    - to: []
      ports:
        - port: 443

---
# Database NetworkPolicy
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: database-policy
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
        - port: 5432
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
      ports:
        - port: 53
          protocol: UDP
```

---

## Step 444: Cross-Namespace Policies

```yaml
# Allow monitoring to scrape metrics
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
          podSelector:
            matchLabels:
              app: prometheus
      ports:
        - port: 9090
        - port: 8080

---
# Allow api-gateway from other namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-api-gateway
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: user-service
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              environment: production
          podSelector:
            matchLabels:
              app: api-gateway
      ports:
        - port: 8080
```

---

## Step 445: Cilium L7 Policies

```yaml
# Cilium CiliumNetworkPolicy - L7 HTTP
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: l7-policy
  namespace: production
spec:
  endpointSelector:
    matchLabels:
      app: backend
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
                path: "/api/.*"
              - method: POST
                path: "/api/users"

---
# Cilium DNS policy
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: dns-fqdn-policy
  namespace: production
spec:
  endpointSelector:
    matchLabels:
      app: payment-service
  egress:
    - toEndpoints:
        - matchLabels:
            k8s:io.kubernetes.pod.namespace: kube-system
      toPorts:
        - ports:
            - port: "53"
              protocol: ANY
          rules:
            dns:
              - matchPattern: "*.svc.cluster.local"
    - toFQDNs:
        - matchName: "api.stripe.com"
      toPorts:
        - ports:
            - port: "443"
```

---

## Step 446: mTLS with Istio

```yaml
# PeerAuthentication - enforce mTLS
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: production
spec:
  mtls:
    mode: STRICT

---
# AuthorizationPolicy - L7 access control
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: backend-authz
  namespace: production
spec:
  selector:
    matchLabels:
      app: backend
  action: ALLOW
  rules:
    - from:
        - source:
            principals:
              - "cluster.local/ns/production/sa/frontend"
      to:
        - operation:
            methods: ["GET", "POST"]
            paths: ["/api/*"]

---
# RequestAuthentication - JWT
apiVersion: security.istio.io/v1beta1
kind: RequestAuthentication
metadata:
  name: jwt-auth
  namespace: production
spec:
  selector:
    matchLabels:
      app: backend
  jwtRules:
    - issuer: "https://accounts.google.com"
      jwksUri: "https://www.googleapis.com/oauth2/v3/certs"
      audiences:
        - "my-cluster"
```

---

## Step 447: Ingress Security

```yaml
# Secure Ingress พร้อม security headers + WAF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: secure-ingress
  namespace: production
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    nginx.ingress.kubernetes.io/configuration-snippet: |
      more_set_headers "X-Frame-Options: DENY";
      more_set_headers "X-Content-Type-Options: nosniff";
      more_set_headers "X-XSS-Protection: 1; mode=block";
      more_set_headers "Strict-Transport-Security: max-age=31536000; includeSubDomains";
    nginx.ingress.kubernetes.io/limit-connections: "10"
    nginx.ingress.kubernetes.io/limit-rps: "100"
    nginx.ingress.kubernetes.io/enable-modsecurity: "true"
    nginx.ingress.kubernetes.io/enable-owasp-core-rules: "true"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - app.example.com
      secretName: app-tls-cert
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend
                port:
                  number: 80

---
# cert-manager Certificate
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: app-tls
  namespace: production
spec:
  secretName: app-tls-cert
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
    - app.example.com
```

---

## Step 448: Block AWS Metadata

```yaml
# ป้องกัน SSRF ไปยัง cloud metadata endpoints
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: block-metadata-server
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
              - 169.254.170.2/32    # ECS metadata
              - 100.100.100.200/32  # Alibaba metadata

---
# Allow VPN/office ranges only for restricted services
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-vpn-access
  namespace: production
spec:
  podSelector:
    matchLabels:
      access: restricted
  policyTypes:
    - Ingress
  ingress:
    - from:
        - ipBlock:
            cidr: 10.100.0.0/16
        - ipBlock:
            cidr: 192.168.1.0/24
```

---

## Step 449-450: Workshop - Network Audit

```yaml
# ValidatingAdmissionPolicy - ห้าม hostNetwork/hostPID/hostIPC
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicy
metadata:
  name: no-host-namespace
spec:
  failurePolicy: Fail
  matchConstraints:
    resourceRules:
      - apiGroups: [""]
        apiVersions: ["v1"]
        operations: ["CREATE", "UPDATE"]
        resources: ["pods"]
  validations:
    - expression: "!object.spec.?hostNetwork.orValue(false)"
      message: "hostNetwork is not allowed"
    - expression: "!object.spec.?hostPID.orValue(false)"
      message: "hostPID is not allowed"
    - expression: "!object.spec.?hostIPC.orValue(false)"
      message: "hostIPC is not allowed"

---
# Network security checklist:
# ☑ Default deny ingress/egress per namespace
# ☑ DNS egress allowed
# ☑ NetworkPolicies สำหรับทุก tier
# ☑ Block AWS metadata endpoint (169.254.169.254)
# ☑ mTLS ด้วย service mesh
# ☑ TLS termination ที่ ingress
# ☑ Security headers
# ☑ Rate limiting + WAF
# ☑ ห้าม hostNetwork/hostPID/hostIPC
```

---

## 📊 สรุป Part 46

| Component | Security Value |
|-----------|---------------|
| Default Deny | ป้องกัน lateral movement |
| 3-Tier Policies | Zero trust per tier |
| Cilium L7 | HTTP-level access control |
| mTLS (Istio) | Encrypt + authenticate all traffic |
| Ingress + WAF | ป้องกัน external attacks |
| Block Metadata | ป้องกัน SSRF |

---

## 🔗 ต่อไป
- [Part 47: Supply Chain Security](./part-47-supply-chain.md)

---
*Part 46 | Steps 441-450 | ระดับสูง*
