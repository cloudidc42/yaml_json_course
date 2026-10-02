# Part 20: Kubernetes Networking Advanced
## Steps 191-200: Networking ขั้นสูง, Service Mesh, และ Ingress

---

## 📖 บทนำ

Kubernetes Networking มีหลายชั้นและหลาย concepts ที่ซับซ้อน บทนี้จะครอบคลุม CNI plugins, Service Mesh, Ingress controllers, และ DNS

### Kubernetes Network Model
```
Kubernetes Network Rules:
1. Every Pod gets unique IP
2. All Pods can communicate without NAT
3. Nodes can communicate with Pods without NAT
4. Pod's IP is same from both inside and outside

Network Flow:
Pod A → Pod B (same node): Linux bridge/veth
Pod A → Pod B (different node): CNI overlay (VXLAN/BGP)
Pod A → Service: kube-proxy iptables/IPVS
External → Service: LoadBalancer/NodePort/Ingress
```

---

## Step 191: CNI Plugins

```yaml
# Calico IPPool - กำหนด Pod CIDR
apiVersion: projectcalico.org/v3
kind: IPPool
metadata:
  name: default-ipv4-ippool
spec:
  cidr: 192.168.0.0/16
  ipipMode: Always
  natOutgoing: true
  nodeSelector: all()
  vxlanMode: Never

---
# Calico GlobalNetworkPolicy
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: default-deny
spec:
  selector: all()
  types:
    - Ingress
    - Egress
  egress:
    - action: Allow
      protocol: UDP
      destination:
        ports: [53]

---
# CiliumNetworkPolicy - L7 policy
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: allow-api-server
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
                path: "/api/data"
  egress:
    - toEndpoints:
        - matchLabels:
            app: database
      toPorts:
        - ports:
            - port: "5432"
              protocol: TCP
```

---

## Step 192: Ingress Controllers

```yaml
# NGINX Ingress พร้อม TLS + Rate Limiting
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app
  namespace: production
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
    nginx.ingress.kubernetes.io/limit-rps: "10"
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - myapp.example.com
        - api.example.com
      secretName: myapp-tls
  rules:
    - host: myapp.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend
                port:
                  number: 80
    - host: api.example.com
      http:
        paths:
          - path: /v1
            pathType: Prefix
            backend:
              service:
                name: api-v1
                port:
                  number: 8080
          - path: /v2
            pathType: Prefix
            backend:
              service:
                name: api-v2
                port:
                  number: 8080
```

---

## Step 193: Service Types

```yaml
# ClusterIP - internal only
apiVersion: v1
kind: Service
metadata:
  name: backend-internal
  namespace: production
spec:
  type: ClusterIP
  selector:
    app: backend
  ports:
    - port: 8080
      targetPort: 8080

---
# LoadBalancer - cloud LB (AWS NLB)
apiVersion: v1
kind: Service
metadata:
  name: app-loadbalancer
  namespace: production
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "external"
    service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: "ip"
    service.beta.kubernetes.io/aws-load-balancer-scheme: "internet-facing"
spec:
  type: LoadBalancer
  selector:
    app: my-app
  ports:
    - port: 443
      targetPort: 8443
  externalTrafficPolicy: Local  # preserve client IP

---
# Headless Service - สำหรับ StatefulSet
apiVersion: v1
kind: Service
metadata:
  name: postgres-headless
  namespace: data
spec:
  clusterIP: None
  selector:
    app: postgres
  ports:
    - port: 5432
# DNS: postgres-0.postgres-headless.data.svc.cluster.local
```

---

## Step 194: CoreDNS Configuration

```yaml
# CoreDNS ConfigMap - customize DNS
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
        prometheus :9153
        forward . /etc/resolv.conf {
           max_concurrent 1000
        }
        cache 30
        loop
        reload
        loadbalance
    }
    # Forward specific domain ไป internal DNS
    example.internal:53 {
        errors
        cache 30
        forward . 10.0.0.53
    }
```

---

## Step 195: Istio Service Mesh

```yaml
# VirtualService - canary traffic splitting
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: my-app
  namespace: production
spec:
  hosts:
    - my-app
    - myapp.example.com
  http:
    - match:
        - headers:
            x-canary:
              exact: "true"
      route:
        - destination:
            host: my-app
            subset: canary
    - route:
        - destination:
            host: my-app
            subset: stable
          weight: 90
        - destination:
            host: my-app
            subset: canary
          weight: 10
      timeout: 30s
      retries:
        attempts: 3
        perTryTimeout: 10s

---
# DestinationRule - circuit breaker
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: my-app
  namespace: production
spec:
  host: my-app
  trafficPolicy:
    loadBalancer:
      simple: LEAST_CONN
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        http2MaxRequests: 10000
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 30s
      baseEjectionTime: 30s
      maxEjectionPercent: 50
  subsets:
    - name: stable
      labels:
        version: stable
    - name: canary
      labels:
        version: canary

---
# PeerAuthentication - mTLS STRICT
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: production
spec:
  mtls:
    mode: STRICT

---
# AuthorizationPolicy - L7 authorization
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: allow-frontend
  namespace: production
spec:
  selector:
    matchLabels:
      app: backend
  rules:
    - from:
        - source:
            principals:
              - "cluster.local/ns/production/sa/frontend"
      to:
        - operation:
            methods: ["GET", "POST"]
            paths: ["/api/*"]
```

---

## Step 196: Kubernetes Gateway API

```yaml
# GatewayClass
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: nginx
spec:
  controllerName: k8s.nginx.org/nginx-gateway-controller

---
# Gateway - entry point
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: main-gateway
  namespace: production
spec:
  gatewayClassName: nginx
  listeners:
    - name: https
      port: 443
      protocol: HTTPS
      tls:
        mode: Terminate
        certificateRefs:
          - kind: Secret
            name: wildcard-tls
      allowedRoutes:
        namespaces:
          from: Selector
          selector:
            matchLabels:
              gateway: allowed

---
# HTTPRoute - routing rules
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: my-app
  namespace: production
spec:
  parentRefs:
    - name: main-gateway
      namespace: production
  hostnames:
    - "myapp.example.com"
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /api
      filters:
        - type: RequestHeaderModifier
          requestHeaderModifier:
            add:
              - name: X-Source
                value: gateway
      backendRefs:
        - name: api
          port: 8080
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: frontend
          port: 80
```

---

## Step 197: eBPF Networking ด้วย Cilium

```yaml
# Cilium Helm values
kubeProxyReplacement: "strict"
bpf:
  masquerade: true
  tproxy: true
loadBalancer:
  algorithm: maglev
  acceleration: native  # XDP acceleration
hubble:
  enabled: true
  relay:
    enabled: true
  ui:
    enabled: true

---
# CiliumClusterwideNetworkPolicy
apiVersion: cilium.io/v2
kind: CiliumClusterwideNetworkPolicy
metadata:
  name: allow-kube-system
spec:
  nodeSelector:
    matchLabels: {}
  egress:
    - toCIDR:
        - 10.0.0.0/8
    - toEntities:
        - world
      toPorts:
        - ports:
            - port: "443"
              protocol: TCP
```

---

## Step 198: Service Mesh Observability

```yaml
# Jaeger - Distributed tracing
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jaeger
  namespace: istio-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: jaeger
  template:
    metadata:
      labels:
        app: jaeger
    spec:
      containers:
        - name: jaeger
          image: jaegertracing/all-in-one:1.52
          env:
            - name: COLLECTOR_OTLP_ENABLED
              value: "true"
            - name: SPAN_STORAGE_TYPE
              value: elasticsearch
            - name: ES_SERVER_URLS
              value: http://elasticsearch:9200
          ports:
            - containerPort: 16686  # Jaeger UI
            - containerPort: 14250  # gRPC
            - containerPort: 4317   # OTLP gRPC

---
# Istio Telemetry
apiVersion: telemetry.istio.io/v1alpha1
kind: Telemetry
metadata:
  name: custom-metrics
  namespace: production
spec:
  metrics:
    - providers:
        - name: prometheus
      overrides:
        - match:
            metric: ALL_METRICS
          tagOverrides:
            response_code:
              value: "response.code"
            source_app:
              value: "source.workload.name"
```

---

## Step 199: DNS Policies

```yaml
# Pod DNS Config
apiVersion: v1
kind: Pod
metadata:
  name: custom-dns-pod
spec:
  dnsPolicy: None
  dnsConfig:
    nameservers:
      - 8.8.8.8
      - 1.1.1.1
    searches:
      - production.svc.cluster.local
      - svc.cluster.local
      - cluster.local
    options:
      - name: ndots
        value: "2"  # ลด DNS lookups
      - name: timeout
        value: "2"
  containers:
    - name: app
      image: myapp:1.0.0
```

---

## Step 200: Workshop - Complete Networking Stack

```yaml
# Complete example: App ที่ใช้ Ingress + mTLS + Network Policy

# Namespace เปิด Istio injection
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    istio-injection: enabled
    pod-security.kubernetes.io/enforce: baseline

---
# Frontend Service + Ingress
apiVersion: v1
kind: Service
metadata:
  name: frontend
  namespace: production
spec:
  selector:
    app: frontend
  ports:
    - port: 80
      targetPort: 80

---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: frontend
  namespace: production
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - myapp.example.com
      secretName: myapp-tls
  rules:
    - host: myapp.example.com
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
# Network Policy
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-netpol
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: frontend
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
              app: backend
      ports:
        - port: 8080
    - ports:
        - port: 53
          protocol: UDP

---
# Istio VirtualService
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: frontend
  namespace: production
spec:
  hosts:
    - frontend
  http:
    - route:
        - destination:
            host: frontend
            port:
              number: 80
      timeout: 10s
      retries:
        attempts: 3
        perTryTimeout: 5s
```

---

## 📊 สรุป Part 20

| หัวข้อ | เครื่องมือ/แนวทาง |
|--------|---------------|
| CNI | Calico, Cilium, Flannel |
| Ingress | NGINX, Traefik, Gateway API |
| Service Types | ClusterIP, NodePort, LB, ExternalName, Headless |
| DNS | CoreDNS, NodeLocal DNSCache |
| Service Mesh | Istio, VirtualService, DestinationRule |
| Gateway API | HTTPRoute, Gateway, GatewayClass |
| eBPF | Cilium, XDP acceleration |
| Observability | Kiali, Jaeger, Hubble |

---

## 🔗 ต่อไป
- [Part 22: Kubernetes Architecture Deep Dive](./part-22-k8s-architecture.md)

---
*Part 20 | Steps 191-200 | ระดับสูง*
