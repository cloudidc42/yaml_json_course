# Part 56: Network Attacks & Defense
## Steps 521-530: Kubernetes Network Security Deep Dive

---

## 📖 บทนำ

> **⚠️ หมายเหตุ**: เนื้อหาด้าน security ในหลักสูตรนี้มีไว้เพื่อการศึกษาและการทดสอบระบบที่ได้รับอนุญาตเท่านั้น การนำไปใช้โจมตีระบบที่ไม่ได้รับอนุญาตเป็นสิ่งผิดกฎหมาย

การทำความเข้าใจ network attack vectors ใน Kubernetes ช่วยในการออกแบบ defense in depth ที่มีประสิทธิภาพ

---

## Step 521: Network Attack Landscape

```
Kubernetes Network Attack Surface:

External Attacks:
  Internet → LB/WAF → Ingress → Services → Pods
  Attack: DDoS, injection, credential stuffing

East-West (Internal):
  Pod A → Pod B (no NetworkPolicy = open)
  Attack: lateral movement, service-to-service exploitation

DNS:
  CoreDNS → Service discovery
  Attack: DNS spoofing, DNS rebinding

API Server:
  kubectl → API Server
  Attack: credential exposure, RBAC misconfiguration

Node Level:
  Container → Host network (with hostNetwork:true)
  Attack: network sniffing, connection hijacking

Cloud Metadata:
  Pod → 169.254.169.254
  Attack: credential theft via IMDS
```

---

## Step 522: Default Deny Network Policies

```yaml
# Default deny: all ingress and egress
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
# Allow DNS (required for pod communication)
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

---
# Tiered application policy

# Frontend: accept from ingress only, send to backend
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
  egress:
    - to:
        - podSelector:
            matchLabels:
              tier: backend
      ports:
        - port: 8080
    - ports:
        - port: 53
          protocol: UDP

---
# Backend: accept from frontend only, send to database
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
    - ports:
        - port: 443
    - ports:
        - port: 53
          protocol: UDP

---
# Database: accept from backend only, no egress
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
    - ports:
        - port: 53
          protocol: UDP
```

---

## Step 523: Cilium L7 Policies

```yaml
# Cilium L7 HTTP policy
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: api-http-policy
  namespace: production
spec:
  endpointSelector:
    matchLabels:
      app: api-server
  
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
              - method: POST
                path: "^/api/v1/orders$"
              - method: GET
                path: "^/health$"
  
  egress:
    - toEndpoints:
        - matchLabels:
            app: database
      toPorts:
        - ports:
            - port: "5432"
              protocol: TCP
    
    - toEndpoints:
        - matchLabels:
            io.kubernetes.pod.namespace: kube-system
            k8s-app: kube-dns
      toPorts:
        - ports:
            - port: "53"
              protocol: ANY
          rules:
            dns:
              - matchPattern: "*.production.svc.cluster.local."
              - matchPattern: "*.cluster.local."

---
# Cilium: FQDN-based policy
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: fqdn-policy
  namespace: production
spec:
  endpointSelector:
    matchLabels:
      app: backend
  egress:
    - toFQDNs:
        - matchName: "api.approved-service.com"
        - matchPattern: "*.approved-domain.com"
      toPorts:
        - ports:
            - port: "443"
              protocol: TCP
```

---

## Step 524: mTLS with Istio

```yaml
# Istio: enforce strict mTLS everywhere
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: production
spec:
  mtls:
    mode: STRICT

---
# AuthorizationPolicy: fine-grained access
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
# JWT validation
apiVersion: security.istio.io/v1beta1
kind: RequestAuthentication
metadata:
  name: jwt-auth
  namespace: production
spec:
  selector:
    matchLabels:
      app: api-gateway
  jwtRules:
    - issuer: "https://accounts.google.com"
      jwksUri: "https://www.googleapis.com/oauth2/v3/certs"
      audiences:
        - "my-api"
      forwardOriginalToken: true

---
# Deny unauthenticated requests
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: require-jwt
  namespace: production
spec:
  selector:
    matchLabels:
      app: api-gateway
  action: DENY
  rules:
    - from:
        - source:
            notRequestPrincipals: ["*"]
```

---

## Step 525: WAF Configuration

```yaml
# Ingress with ModSecurity WAF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: protected-ingress
  namespace: production
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/enable-modsecurity: "true"
    nginx.ingress.kubernetes.io/enable-owasp-core-rules: "true"
    nginx.ingress.kubernetes.io/modsecurity-snippet: |
      SecRuleEngine On
      SecRequestBodyAccess On
      SecResponseBodyAccess Off
      SecRequestBodyLimit 13107200
      SecRule ARGS "@detectSQLi" \
        "id:9002,phase:2,block,msg:'SQL Injection Attack'"
      SecRule ARGS "@detectXSS" \
        "id:9003,phase:2,block,msg:'XSS Attack'"
    
    nginx.ingress.kubernetes.io/configuration-snippet: |
      more_set_headers "X-Frame-Options: DENY";
      more_set_headers "X-Content-Type-Options: nosniff";
      more_set_headers "X-XSS-Protection: 1; mode=block";
      more_set_headers "Strict-Transport-Security: max-age=31536000; includeSubDomains; preload";
    
    nginx.ingress.kubernetes.io/limit-rps: "100"
    nginx.ingress.kubernetes.io/limit-connections: "20"
    nginx.ingress.kubernetes.io/limit-req-status-code: "429"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    cert-manager.io/cluster-issuer: letsencrypt-prod

spec:
  tls:
    - hosts:
        - api.example.com
      secretName: api-tls
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 80
```

---

## Step 526: SSRF Protection

```yaml
# NetworkPolicy: block internal network from application pods
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: ssrf-protection
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: web-app
  policyTypes:
    - Egress
  egress:
    - ports:
        - port: 443
          protocol: TCP
      to:
        - ipBlock:
            cidr: 0.0.0.0/0
            except:
              - 10.0.0.0/8
              - 172.16.0.0/12
              - 192.168.0.0/16
              - 169.254.0.0/16    # Link-local (metadata)
              - 127.0.0.0/8       # Loopback
              - 100.64.0.0/10     # Shared address space
    - ports:
        - port: 53
          protocol: UDP
      to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system

---
# Istio ServiceEntry: allowlist external services
apiVersion: networking.istio.io/v1beta1
kind: ServiceEntry
metadata:
  name: allowed-external-apis
  namespace: production
spec:
  hosts:
    - api.github.com
    - api.stripe.com
    - api.sendgrid.com
  ports:
    - number: 443
      name: https
      protocol: HTTPS
  location: MESH_EXTERNAL
  resolution: DNS
```

---

## Step 527: DNS Security

```yaml
# CoreDNS: filter malicious DNS
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors
        health { lameduck 5s }
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
           pods insecure
           fallthrough in-addr.arpa ip6.arpa
           ttl 30
        }
        hosts /etc/coredns/block-list.txt {
           fallthrough
        }
        prometheus :9153
        forward . 8.8.8.8 8.8.4.4 {
           max_concurrent 1000
        }
        cache 30
        loop
        reload
        loadbalance
    }
  
  block-list.txt: |
    0.0.0.0 malware-c2.example.com
    0.0.0.0 phishing.bad-domain.net
```

---

## Step 528: DDoS Protection

```yaml
# HPA: scale out during attack
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-server
  minReplicas: 3
  maxReplicas: 100
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
        - type: Pods
          value: 10
          periodSeconds: 60
```

---

## Step 529: Egress Security

```yaml
# Egress gateway (Istio)
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: egress-gateway
  namespace: istio-system
spec:
  selector:
    istio: egressgateway
  servers:
    - port:
        number: 443
        name: https
        protocol: HTTPS
      hosts:
        - "*.external.example.com"
      tls:
        mode: PASSTHROUGH

---
# NetworkPolicy: only egress gateway can reach internet
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: restrict-internet-egress
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - to:
        - ipBlock:
            cidr: 10.0.0.0/8
    - ports:
        - port: 53
          protocol: UDP
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: istio-system
          podSelector:
            matchLabels:
              istio: egressgateway
```

---

## Step 530: Workshop - Network Security Audit

```yaml
# Network security audit checklist
# 1. Check for pods with hostNetwork:
# kubectl get pods --all-namespaces -o json | \
#   jq '.items[] | select(.spec.hostNetwork==true) | .metadata.name'

# 2. Test NetworkPolicy enforcement:
# kubectl run nettest --image=nicolaka/netshoot --rm -it -- /bin/bash
# curl -s http://kubernetes.default.svc   # should fail
# curl -s http://169.254.169.254           # should fail

# ValidatingAdmissionPolicy: block host namespaces
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
    - expression: |
        !has(object.spec.hostNetwork) ||
        object.spec.hostNetwork == false
      message: "hostNetwork is not allowed"
    - expression: |
        !has(object.spec.hostPID) ||
        object.spec.hostPID == false
      message: "hostPID is not allowed"
    - expression: |
        !has(object.spec.hostIPC) ||
        object.spec.hostIPC == false
      message: "hostIPC is not allowed"

---
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicyBinding
metadata:
  name: no-host-namespace-binding
spec:
  policyName: no-host-namespace
  validationActions: [Deny, Audit]
  matchResources:
    namespaceSelector:
      matchExpressions:
        - key: kubernetes.io/metadata.name
          operator: NotIn
          values: [kube-system, monitoring]
```

---

## 📊 สรุป Part 56

| Network Layer | Defense |
|--------------|---------|
| Ingress | WAF + Rate limiting + TLS |
| East-West | NetworkPolicy default deny |
| Service Mesh | mTLS + AuthorizationPolicy |
| Egress | Egress gateway + FQDN policy |
| DNS | CoreDNS filtering |
| Cloud Metadata | NetworkPolicy block 169.254.x.x |

---

## 🔗 ต่อไป
- [Part 57: Lateral Movement & Detection](./part-57-lateral-movement.md)

---
*Part 56 | Steps 521-530 | ระดับผู้เชี่ยวชาญ | Educational Use Only*
