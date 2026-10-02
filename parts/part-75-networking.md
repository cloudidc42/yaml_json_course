# Part 75: Kubernetes Networking Deep Dive
## Steps 711-720: CNI, Services, Ingress, DNS

---

## Step 711: Kubernetes Networking Model

```
Kubernetes Networking Requirements:
  1. Every pod gets a unique cluster-routable IP
  2. Pods can communicate without NAT
  3. No NAT between pods and nodes

Networking layers:
  L3: IP routing (pod CIDR, node CIDR)
  L4: TCP/UDP (Services, NodePort)
  L7: HTTP/HTTPS (Ingress, Service Mesh)

Key components:
  CNI: Assigns IP to pods, implements NetworkPolicy
    Options: Calico, Cilium, Flannel, Weave
  kube-proxy: Implements Services (iptables/IPVS)
  CoreDNS: Cluster DNS, resolves service names
```

---

## Step 712: Pod Networking

```yaml
# Multi-container pod sharing localhost
apiVersion: v1
kind: Pod
metadata:
  name: multi-container
spec:
  containers:
    - name: app
      image: myapp:1.0
      ports:
        - containerPort: 8080
    
    # Sidecar: nginx reverse proxy on port 80
    - name: nginx-proxy
      image: nginx:alpine
      ports:
        - containerPort: 80
      volumeMounts:
        - name: nginx-config
          mountPath: /etc/nginx/nginx.conf
          subPath: nginx.conf
  
  volumes:
    - name: nginx-config
      configMap:
        name: nginx-config

---
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
data:
  nginx.conf: |
    events {}
    http {
      server {
        listen 80;
        location / {
          proxy_pass http://localhost:8080;
          proxy_set_header Host $host;
          proxy_set_header X-Real-IP $remote_addr;
        }
      }
    }
```

---

## Step 713: Services Deep Dive

```yaml
# ClusterIP: internal only
apiVersion: v1
kind: Service
metadata:
  name: backend
  namespace: production
spec:
  type: ClusterIP
  selector:
    app: backend
  ports:
    - port: 80
      targetPort: 8080
      name: http

---
# NodePort: expose on every node
apiVersion: v1
kind: Service
metadata:
  name: backend-nodeport
spec:
  type: NodePort
  selector:
    app: backend
  ports:
    - port: 80
      targetPort: 8080
      nodePort: 30080

---
# LoadBalancer with source IP restriction
apiVersion: v1
kind: Service
metadata:
  name: frontend-lb
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: nlb
    service.beta.kubernetes.io/aws-load-balancer-internal: "true"
spec:
  type: LoadBalancer
  loadBalancerSourceRanges:
    - "10.0.0.0/8"
  selector:
    app: frontend
  ports:
    - port: 443
      targetPort: 8443

---
# ExternalName: alias to external DNS
apiVersion: v1
kind: Service
metadata:
  name: external-db
  namespace: production
spec:
  type: ExternalName
  externalName: database.external.example.com

---
# Headless: DNS round-robin to pod IPs
apiVersion: v1
kind: Service
metadata:
  name: backend-headless
spec:
  clusterIP: None
  selector:
    app: backend
  ports:
    - port: 8080
```

---

## Step 714: Endpoints and EndpointSlices

```yaml
# Manual Endpoints for external service
apiVersion: v1
kind: Service
metadata:
  name: external-service
  namespace: production
spec:
  ports:
    - port: 5432
      protocol: TCP

---
apiVersion: v1
kind: Endpoints
metadata:
  name: external-service
  namespace: production
subsets:
  - addresses:
      - ip: 10.0.1.100
      - ip: 10.0.1.101
    ports:
      - port: 5432

---
# EndpointSlice (newer, more scalable)
apiVersion: discovery.k8s.io/v1
kind: EndpointSlice
metadata:
  name: external-service-xyz
  namespace: production
  labels:
    kubernetes.io/service-name: external-service
addressType: IPv4
ports:
  - port: 5432
    protocol: TCP
endpoints:
  - addresses: ["10.0.1.100"]
    conditions:
      ready: true
  - addresses: ["10.0.1.101"]
    conditions:
      ready: true

---
# Traffic distribution: prefer local zone
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: my-app
  ports:
    - port: 80
  trafficDistribution: PreferClose
```

---

## Step 715: Ingress and IngressClass

```yaml
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: nginx
  annotations:
    ingressclass.kubernetes.io/is-default-class: "true"
spec:
  controller: k8s.io/ingress-nginx

---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: main-ingress
  namespace: production
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/rate-limit: "100"
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - api.example.com
      secretName: example-com-tls
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /v1(/|$)(.*)
            pathType: Prefix
            backend:
              service:
                name: api-v1
                port:
                  number: 80
          - path: /v2(/|$)(.*)
            pathType: Prefix
            backend:
              service:
                name: api-v2
                port:
                  number: 80

---
# Gateway API (next-gen Ingress)
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: my-route
  namespace: production
spec:
  parentRefs:
    - name: main-gateway
      namespace: istio-system
  hostnames:
    - api.example.com
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /api
      backendRefs:
        - name: api-service
          port: 80
          weight: 100
```

---

## Step 716: CoreDNS Configuration

```yaml
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
    
    # Route corporate domains to internal DNS
    mycompany.internal:53 {
      errors
      cache 30
      forward . 10.0.0.2 10.0.0.3
    }

# DNS FQDN patterns:
# <service>.<namespace>.svc.cluster.local
# <pod-ip-dashes>.<namespace>.pod.cluster.local
# <pod>.<svc>.<ns>.svc.cluster.local (StatefulSets)
```

---

## Step 717: NetworkPolicy Advanced

```yaml
# Default deny all in namespace
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
# Allow DNS egress (required for all pods)
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
        - podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - port: 53
          protocol: UDP
        - port: 53
          protocol: TCP

---
# Database: only from backend, monitoring
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
        - namespaceSelector:
            matchLabels:
              name: monitoring
      ports:
        - port: 5432
        - port: 9187
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

## Step 718: CNI Comparison

```yaml
# Calico GlobalNetworkPolicy: block NodePort from internet
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: deny-nodeport-from-internet
spec:
  selector: has(app)
  types:
    - Ingress
  ingress:
    - action: Deny
      protocol: TCP
      source:
        nets:
          - 0.0.0.0/0
        notNets:
          - 10.0.0.0/8
          - 172.16.0.0/12
          - 192.168.0.0/16
      destination:
        ports:
          - 30000:32767

# CNI choice guide:
# Production multi-cluster: Cilium (eBPF, ClusterMesh)
# On-prem with BGP: Calico
# Dev/test: Flannel (simple)
# EKS: AWS VPC CNI (native VPC IPs)
# GKE: Dataplane V2 (Cilium-based)
# AKS: Azure CNI Overlay or Cilium
```

---

## Step 719: Service Discovery and Load Balancing

```yaml
# Session affinity (sticky sessions)
apiVersion: v1
kind: Service
metadata:
  name: sticky-service
spec:
  selector:
    app: my-app
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 10800
  ports:
    - port: 80

---
# External DNS: auto-create DNS records
apiVersion: v1
kind: Service
metadata:
  name: my-service
  annotations:
    external-dns.alpha.kubernetes.io/hostname: my-service.example.com
    external-dns.alpha.kubernetes.io/ttl: "300"
spec:
  type: LoadBalancer
  selector:
    app: my-app
  ports:
    - port: 80

---
# AWS NLB via AWS Load Balancer Controller
apiVersion: v1
kind: Service
metadata:
  name: nlb-service
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: external
    service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: ip
    service.beta.kubernetes.io/aws-load-balancer-scheme: internet-facing
spec:
  type: LoadBalancer
  selector:
    app: my-app
  ports:
    - port: 443
      targetPort: 8443
```

---

## Step 720: Workshop - Networking Troubleshooting

```bash
# ===== NETWORK TROUBLESHOOTING =====

# 1. Test DNS resolution
kubectl run dns-test --image=busybox --rm -it --restart=Never -- \
  nslookup kubernetes.default.svc.cluster.local

# 2. Test service connectivity
kubectl run curl-test --image=curlimages/curl --rm -it --restart=Never -- \
  curl -v http://my-service.my-namespace.svc.cluster.local

# 3. Check endpoints
kubectl get endpoints my-service -n my-namespace
kubectl describe endpoints my-service -n my-namespace

# 4. Check NetworkPolicy (Cilium)
kubectl exec -n kube-system ds/cilium -- cilium policy get

# 5. Packet capture in pod
kubectl debug -it pod/my-pod --image=nicolaka/netshoot -- \
  tcpdump -i eth0 port 80

# 6. Check CoreDNS logs
kubectl logs -n kube-system -l k8s-app=kube-dns --tail=100
```

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: network-alerts
  namespace: monitoring
spec:
  groups:
    - name: networking
      rules:
        - alert: CoreDNSErrorsHigh
          expr: rate(coredns_dns_responses_total{rcode="SERVFAIL"}[5m]) > 0.01
          for: 5m
          annotations:
            summary: "CoreDNS SERVFAIL rate is high"
          labels:
            severity: warning

        - alert: ServiceEndpointsDown
          expr: kube_endpoint_address_not_ready > 0
          for: 5m
          annotations:
            summary: "Service {{ $labels.endpoint }} has not-ready endpoints"
          labels:
            severity: warning
```

---

## 📊 สรุป Part 75

| Component | Role | Key Config |
|-----------|------|------------|
| CNI | Pod networking | Cilium, Calico, Flannel |
| ClusterIP | Internal LB | selector, ports |
| NodePort | Node-level expose | nodePort: 30000-32767 |
| LoadBalancer | Cloud LB | annotations |
| Ingress | HTTP routing | rules, tls |
| CoreDNS | Cluster DNS | Corefile, custom zones |
| NetworkPolicy | L3/L4 firewall | podSelector, namespaceSelector |

---

## 🔗 ต่อไป
- [Part 76: Kubernetes Autoscaling](./part-76-autoscaling.md)

---
*Part 75 | Steps 711-720 | Networking Deep Dive | Educational Content*
