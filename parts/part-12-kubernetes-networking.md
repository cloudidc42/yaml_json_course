# Part 12: Kubernetes Networking
## Steps 111-120: Networking ใน Kubernetes

---

## Step 111: Kubernetes Networking Model

```
Kubernetes Networking Model มี 4 requirements:
1. Pods สามารถ communicate กับ pods อื่นได้โดยไม่ต้อง NAT
2. Nodes สามารถ communicate กับ pods ได้โดยไม่ต้อง NAT
3. Pod ที่ pod เห็น IP ตรงกับ IP ที่ node อื่นเห็น
4. Containers ใน pod เดียวกัน share network namespace

Network Plugin (CNI):
- Calico    → ใช้ BGP, รองรับ NetworkPolicy
- Flannel   → Simple overlay network
- Weave     → Multi-cloud
- Cilium    → eBPF-based, advanced features
- AWS VPC CNI → ใช้ native AWS networking
- GKE Dataplane V2 → eBPF-based สำหรับ GKE
```

---

## Step 112: Services

```yaml
# ClusterIP (default) - internal only
apiVersion: v1
kind: Service
metadata:
  name: api-service
  namespace: production
spec:
  type: ClusterIP
  selector:
    app: api
  ports:
    - name: http
      port: 80
      targetPort: 8080
    - name: metrics
      port: 9090
      targetPort: 9090
  
  # Session Affinity
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 10800

---
# NodePort - expose บน Node IP
apiVersion: v1
kind: Service
metadata:
  name: api-nodeport
spec:
  type: NodePort
  selector:
    app: api
  ports:
    - port: 80
      targetPort: 8080
      nodePort: 30080  # 30000-32767

---
# LoadBalancer - Cloud LB (AWS ELB, GCP LB)
apiVersion: v1
kind: Service
metadata:
  name: api-lb
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
    service.beta.kubernetes.io/aws-load-balancer-scheme: "internet-facing"
    service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: "true"
spec:
  type: LoadBalancer
  selector:
    app: api
  ports:
    - port: 80
      targetPort: 8080
  loadBalancerSourceRanges:
    - "1.2.3.4/32"  # Whitelist IP

---
# ExternalName - DNS alias
apiVersion: v1
kind: Service
metadata:
  name: external-db
  namespace: production
spec:
  type: ExternalName
  externalName: mydb.example.com  # resolves to external DNS

---
# Headless Service (สำหรับ StatefulSet)
apiVersion: v1
kind: Service
metadata:
  name: postgres-headless
spec:
  clusterIP: None  # Headless
  selector:
    app: postgres
  ports:
    - port: 5432
```

---

## Step 113: Ingress Controller Setup

```bash
# ติดตั้ง nginx ingress controller
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/cloud/deploy.yaml

# ตรวจสอบ
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx
```

```yaml
# Advanced Ingress
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: advanced-ingress
  annotations:
    # Rewrite
    nginx.ingress.kubernetes.io/rewrite-target: /$2
    
    # Rate Limiting
    nginx.ingress.kubernetes.io/limit-connections: "10"
    nginx.ingress.kubernetes.io/limit-rpm: "100"
    nginx.ingress.kubernetes.io/limit-burst-multiplier: "5"
    
    # Auth
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/auth-secret: basic-auth
    nginx.ingress.kubernetes.io/auth-realm: "Authentication Required"
    
    # Custom headers
    nginx.ingress.kubernetes.io/configuration-snippet: |
      more_set_headers "X-Frame-Options: SAMEORIGIN";
      more_set_headers "X-Content-Type-Options: nosniff";
      more_set_headers "X-XSS-Protection: 1; mode=block";
    
    # Backend protocol
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
    
    # Timeout
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "10"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "60"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "60"
    
    # Buffer
    nginx.ingress.kubernetes.io/proxy-buffer-size: "8k"
    nginx.ingress.kubernetes.io/proxy-buffers-number: "4"

spec:
  ingressClassName: nginx
  tls:
    - secretName: myapp-tls
      hosts: ["myapp.example.com"]
  rules:
    - host: myapp.example.com
      http:
        paths:
          - path: /api(/|$)(.*)
            pathType: Prefix
            backend:
              service:
                name: api
                port:
                  number: 80
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend
                port:
                  number: 80
```

---

## Step 114: cert-manager สำหรับ TLS

```bash
# ติดตั้ง cert-manager
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.0/cert-manager.yaml
```

```yaml
# ClusterIssuer (Let's Encrypt)
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@example.com
    privateKeySecretRef:
      name: letsencrypt-prod-key
    solvers:
      - http01:
          ingress:
            class: nginx
      # DNS challenge (สำหรับ wildcard certs)
      - dns01:
          route53:
            region: ap-southeast-1
            accessKeyIDSecretRef:
              name: route53-credentials
              key: access-key-id
            secretAccessKeySecretRef:
              name: route53-credentials
              key: secret-access-key

---
# Certificate manual request
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: myapp-cert
  namespace: production
spec:
  secretName: myapp-tls
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
    - myapp.example.com
    - "*.myapp.example.com"
  privateKey:
    algorithm: RSA
    size: 2048
  renewBefore: 720h  # Renew 30 days before expiry
```

---

## Step 115: Service Mesh (Istio)

```bash
# ติดตั้ง Istio
curl -L https://istio.io/downloadIstio | sh -
istioctl install --set profile=default -y

# Enable sidecar injection สำหรับ namespace
kubectl label namespace production istio-injection=enabled
```

```yaml
# VirtualService - Traffic routing
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: myapp-vs
  namespace: production
spec:
  hosts:
    - myapp.example.com
    - myapp.production.svc.cluster.local
  gateways:
    - myapp-gateway
  http:
    # Canary deployment (10% traffic ไป v2)
    - match:
        - uri:
            prefix: /api
      route:
        - destination:
            host: api
            subset: v1
          weight: 90
        - destination:
            host: api
            subset: v2
          weight: 10
    
    # Timeout และ Retry
    - route:
        - destination:
            host: api
            subset: v1
      timeout: 10s
      retries:
        attempts: 3
        perTryTimeout: 3s
        retryOn: "5xx,reset,connect-failure"

---
# DestinationRule - กำหนด subsets
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: api-dr
spec:
  host: api
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        http1MaxPendingRequests: 100
        http2MaxRequests: 1000
    outlierDetection:
      consecutiveGatewayErrors: 5
      interval: 30s
      baseEjectionTime: 30s
  subsets:
    - name: v1
      labels:
        version: v1
    - name: v2
      labels:
        version: v2

---
# PeerAuthentication - mTLS
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: production
spec:
  mtls:
    mode: STRICT  # บังคับ mTLS ทุก connection
```

---

## Step 116: DNS ใน Kubernetes

```
CoreDNS เป็น default DNS ใน Kubernetes

Service DNS Format:
<service-name>.<namespace>.svc.cluster.local
  ↓
api.production.svc.cluster.local → 10.100.200.1

Pod DNS Format:
<pod-ip-dashes>.<namespace>.pod.cluster.local
  ↓
10-244-0-5.production.pod.cluster.local

Headless Service Pods:
<pod-name>.<service-name>.<namespace>.svc.cluster.local
  ↓
postgres-0.postgres.data.svc.cluster.local
```

```yaml
# CoreDNS ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors
        health {
           lameduck 5s
        }
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
           pods insecure
           fallthrough in-addr.arpa ip6.arpa
           ttl 30
        }
        
        # Custom DNS records
        hosts {
          192.168.1.100 internal.example.com
          fallthrough
        }
        
        # Forward external queries
        forward . 8.8.8.8 8.8.4.4 {
          max_concurrent 1000
        }
        
        prometheus :9153
        cache 30
        loop
        reload
        loadbalance
    }

---
# Pod DNS Config
apiVersion: v1
kind: Pod
spec:
  dnsPolicy: ClusterFirst
  dnsConfig:
    nameservers:
      - 1.1.1.1  # Fallback
    searches:
      - production.svc.cluster.local
      - svc.cluster.local
      - cluster.local
    options:
      - name: ndots
        value: "5"
      - name: timeout
        value: "5"
```

---

## Step 117: Network Policies Advanced

```yaml
# ระบบ 3 tier: frontend → backend → database
# namespace: production

---
# Frontend: รับ traffic จาก ingress controller เท่านั้น
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
        - port: 80
  egress:
    - to:
        - podSelector:
            matchLabels:
              tier: backend
      ports:
        - port: 8080
    # Allow DNS
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - port: 53
          protocol: UDP

---
# Backend: รับจาก frontend, ส่งออกไปยัง database
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
    # External APIs
    - to: []
      ports:
        - port: 443

---
# Database: รับจาก backend เท่านั้น
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
  egress: []  # ไม่อนุญาต outbound จาก database
```

---

## Step 118: Load Balancer ขั้นสูง

### AWS Load Balancer Controller
```yaml
# Application Load Balancer (ALB) via Ingress
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-alb
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/load-balancer-name: myapp-alb
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:ap-southeast-1:xxx:certificate/yyy
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS":443}]'
    alb.ingress.kubernetes.io/ssl-redirect: "443"
    
    # WAF
    alb.ingress.kubernetes.io/wafv2-acl-arn: arn:aws:wafv2:...
    
    # Target group attributes
    alb.ingress.kubernetes.io/target-group-attributes: >-
      deregistration_delay.timeout_seconds=30,
      slow_start.duration_seconds=30
    
    # Health check
    alb.ingress.kubernetes.io/healthcheck-path: /health
    alb.ingress.kubernetes.io/healthcheck-interval-seconds: "15"
    alb.ingress.kubernetes.io/healthcheck-timeout-seconds: "5"
    alb.ingress.kubernetes.io/healthy-threshold-count: "2"
    alb.ingress.kubernetes.io/unhealthy-threshold-count: "3"

spec:
  rules:
    - host: myapp.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: myapp
                port:
                  number: 80
```

---

## Step 119: Monitoring Network

```yaml
# Network metrics ด้วย Prometheus
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: network-alerts
spec:
  groups:
    - name: network
      rules:
        - alert: HighNetworkErrors
          expr: |
            rate(container_network_transmit_errors_total[5m]) > 0.1
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "High network errors for pod {{ $labels.pod }}"
        
        - alert: NetworkPolicyDrops
          expr: |
            increase(cilium_drop_count_total[5m]) > 100
          for: 2m
          labels:
            severity: warning
          annotations:
            summary: "Possible NetworkPolicy misconfiguration"
```

---

## Step 120: Debugging Networking

```bash
# ตรวจสอบ network ใน pod
kubectl exec -it myapp -- /bin/sh

# ภายใน pod
ping 10.244.0.5              # Pod IP
nslookup api.production      # DNS lookup
curl http://api-service:8080 # Service discovery
wget -O- http://api-service/health

# ตรวจสอบ services
kubectl get endpoints api-service
kubectl describe service api-service

# ตรวจสอบ network policies
kubectl get networkpolicies -n production
kubectl describe networkpolicy frontend-policy -n production

# Test connectivity ด้วย netshoot
kubectl run netshoot --image=nicolaka/netshoot --rm -it -- /bin/bash
# ใน netshoot
tcpdump -i eth0 port 5432
ss -tlnp
nmap -sT api-service

# ตรวจสอบ DNS
kubectl exec -it myapp -- nslookup postgres.data.svc.cluster.local
kubectl exec -it myapp -- dig postgres.data.svc.cluster.local

# Port-forward สำหรับ debugging
kubectl port-forward svc/api-service 8080:80
kubectl port-forward pod/myapp-xxx 8080:8080

# Traffic flow debug ด้วย Cilium
cilium connectivity test
hubble observe --follow --namespace production
```

---

## 📊 สรุป Part 12

| Resource | ฟังก์ชัน |
|---------|----------|
| ClusterIP | Internal service discovery |
| NodePort | External access via node |
| LoadBalancer | Cloud LB integration |
| ExternalName | External DNS alias |
| Headless | StatefulSet DNS |
| Ingress | HTTP/HTTPS routing |
| NetworkPolicy | Pod firewall |
| Service Mesh | mTLS, traffic control |

---
*Part 12 | Steps 111-120 | ระดับกลาง*
