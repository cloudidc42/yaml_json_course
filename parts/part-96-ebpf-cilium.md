# Part 96: Advanced Networking with eBPF and Cilium
## Steps 921-930: eBPF, Cilium CNI, Network Policies, Hubble, Service Mesh

---

## Step 921: eBPF Overview

```
eBPF (Extended Berkeley Packet Filter):

What is eBPF?
  - Linux kernel technology (kernel 4.4+)
  - Run sandboxed programs in kernel space
  - No kernel module, no recompilation
  - Verified by kernel JIT compiler (safe)
  - Event-driven: triggered by kernel hooks

eBPF Use Cases:
  Networking:
    - XDP (eXpress Data Path): packet processing at NIC level
    - Traffic shaping, load balancing (L4)
    - Network policy enforcement (no iptables)
    - Service mesh without sidecar proxy
  
  Observability:
    - Syscall tracing (bpftrace)
    - CPU/memory profiling (Parca, Pyroscope)
    - Network flow visibility (Hubble)
    - Security monitoring (Falco, Tetragon)
  
  Security:
    - LSM (Linux Security Modules) hooks
    - Runtime security enforcement
    - Falco + eBPF for container security
    - Tetragon: Kubernetes-aware eBPF security

Why eBPF is better than iptables for Kubernetes:
  iptables:
    - O(n) rule evaluation (linear scan)
    - Kernel netfilter complexity
    - No service-level awareness
  
  eBPF (Cilium):
    - O(1) hash table lookups
    - Direct packet modification (XDP)
    - Service-aware at kernel level
    - 99th percentile latency reduction

Tools:
  Cilium: CNI + service mesh + network policy
  Hubble: eBPF-based network observability
  Tetragon: eBPF security observability
  Falco eBPF probe: alternative to kernel module
```

---

## Step 922: Cilium Installation

```yaml
# Cilium CiliumNetworkPolicy: L7 (application layer) policy
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: api-policy
  namespace: production
spec:
  endpointSelector:
    matchLabels:
      app: payment-api
  
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
              - method: POST
                path: "/api/payments"
              - method: GET
                path: "/api/payments/[0-9]+"
    
    - fromEndpoints:
        - matchLabels:
            app: prometheus
      toPorts:
        - ports:
            - port: "9090"
              protocol: TCP
  
  egress:
    - toEndpoints:
        - matchLabels:
            app: postgres
      toPorts:
        - ports:
            - port: "5432"
              protocol: TCP
    
    - toFQDNs:
        - matchName: api.stripe.com
      toPorts:
        - ports:
            - port: "443"
              protocol: TCP
```

---

## Step 923: Cilium L7 DNS Policy

```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: dns-egress-policy
  namespace: production
spec:
  endpointSelector:
    matchLabels:
      app: payment-api
  
  egress:
    - toEndpoints:
        - matchLabels:
            k8s:io.kubernetes.pod.namespace: kube-system
            k8s-app: kube-dns
      toPorts:
        - ports:
            - port: "53"
              protocol: UDP
          rules:
            dns:
              - matchPattern: "*.svc.cluster.local"
              - matchPattern: "*.stripe.com"
              - matchPattern: "*.amazonaws.com"
    
    - toFQDNs:
        - matchPattern: "*.stripe.com"
        - matchName: api.pagerduty.com
      toPorts:
        - ports:
            - port: "443"
              protocol: TCP
```

---

## Step 924: Hubble Network Observability

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hubble-ui
  namespace: kube-system
spec:
  replicas: 1
  template:
    spec:
      containers:
        - name: frontend
          image: quay.io/cilium/hubble-ui:v0.13.0
          ports:
            - containerPort: 8081

---
apiVersion: v1
kind: Service
metadata:
  name: hubble-relay
  namespace: kube-system
spec:
  type: ClusterIP
  selector:
    k8s-app: hubble-relay
  ports:
    - port: 80
      targetPort: 4245

---
apiVersion: v1
kind: ConfigMap
metadata:
  name: hubble-otel-exporter
  namespace: kube-system
data:
  config.yaml: |
    flows:
      - encoding: json
      - filters:
          - verdict: DROPPED
          - verdict: ERROR
    exporter:
      type: otel-grpc
      endpoint: otelcol.monitoring.svc:4317
```

---

## Step 925: Tetragon Security Observability

```yaml
# Tetragon: eBPF-based Kubernetes security
# NOTE: เนื้อหาด้าน security ในหลักสูตรนี้มีไว้เพื่อการศึกษาและการทดสอบระบบที่ได้รับอนุญาตเท่านั้น
apiVersion: cilium.io/v1alpha1
kind: TracingPolicy
metadata:
  name: detect-shell-execution
spec:
  kprobes:
    - call: sys_execve
      syscall: true
      args:
        - index: 0
          type: string
      selectors:
        - matchBinaries:
            - operator: In
              values:
                - /bin/sh
                - /bin/bash
                - /usr/bin/python3
          matchActions:
            - action: Sigkill
            - action: Post

---
apiVersion: cilium.io/v1alpha1
kind: TracingPolicy
metadata:
  name: detect-sensitive-file-read
spec:
  kprobes:
    - call: sys_openat
      syscall: true
      args:
        - index: 1
          type: string
      selectors:
        - matchArgs:
            - index: 1
              operator: Prefix
              values:
                - /etc/shadow
                - /etc/passwd
                - /var/run/secrets/kubernetes.io
          matchActions:
            - action: Post
```

---

## Step 926: Cilium Service Mesh (Sidecarless)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: payment-api-ingress
  namespace: production
  annotations:
    ingress.cilium.io/loadbalancer-mode: shared
spec:
  ingressClassName: cilium
  rules:
    - host: api.mycompany.com
      http:
        paths:
          - path: /api/payments
            pathType: Prefix
            backend:
              service:
                name: payment-api
                port:
                  number: 80

---
apiVersion: cilium.io/v2
kind: CiliumEnvoyConfig
metadata:
  name: payment-api-retries
  namespace: production
spec:
  services:
    - name: payment-api
      namespace: production
  backendServices:
    - name: payment-api
      namespace: production
  resources:
    - "@type": type.googleapis.com/envoy.config.route.v3.RouteConfiguration
      name: payment-api-routes
      virtual_hosts:
        - name: payment-api
          domains: ["*"]
          routes:
            - match:
                prefix: "/api/payments"
              route:
                cluster: payment-api
                retry_policy:
                  retry_on: connect-failure,reset,5xx
                  num_retries: 3
                  per_try_timeout: 5s
```

---

## Step 927: BGP and Load Balancer IP

```yaml
apiVersion: cilium.io/v2alpha1
kind: CiliumBGPPeeringPolicy
metadata:
  name: bgp-policy
spec:
  nodeSelector:
    matchLabels:
      bgp-speaker: "true"
  virtualRouters:
    - localASN: 65001
      exportPodCIDR: true
      serviceSelector:
        matchLabels:
          expose-via-bgp: "true"
      neighbors:
        - peerAddress: 10.0.0.1/32
          peerASN: 65000
          eBGPMultihop: true
          connectRetryTimeSeconds: 30

---
apiVersion: cilium.io/v2alpha1
kind: CiliumLoadBalancerIPPool
metadata:
  name: production-pool
spec:
  cidrs:
    - cidr: 10.100.0.0/24
  serviceSelector:
    matchLabels:
      io.cilium.lb-ipam.ips: production-pool

---
apiVersion: v1
kind: Service
metadata:
  name: payment-api-lb
  namespace: production
  labels:
    io.cilium.lb-ipam.ips: production-pool
    expose-via-bgp: "true"
spec:
  type: LoadBalancer
  selector:
    app: payment-api
  ports:
    - port: 443
      targetPort: 8443
```

---

## Step 928: Network Performance Tuning

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: payment-api
  annotations:
    kubernetes.io/egress-bandwidth: "100M"
    kubernetes.io/ingress-bandwidth: "100M"
spec:
  containers:
    - name: api
      image: myorg/payment-api:1.0

---
apiVersion: cilium.io/v2alpha1
kind: CiliumNodeConfig
metadata:
  name: high-performance-nodes
  namespace: kube-system
spec:
  nodeSelector:
    matchLabels:
      performance: high
  defaults:
    bpf-lb-algorithm: maglev
    enable-bandwidth-manager: "true"
```

---

## Step 929: Network Troubleshooting

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: network-debug-commands
  namespace: kube-system
data:
  commands.sh: |
    # Check Cilium status
    cilium status --verbose
    
    # List all network policies
    cilium policy get
    
    # Observe dropped packets
    cilium monitor --type drop
    
    # Hubble: DNS flows
    hubble observe --protocol DNS --follow
    
    # Export flows for specific pod
    hubble observe --pod production/payment-api \
      --since 1h --output json > flows.json
    
    # Check BGP peers
    cilium bgp peers
    cilium bgp routes
    
    # BPF maps inspection
    bpftool map list
    bpftool prog list | grep cilium
```

---

## Step 930: Workshop - eBPF + Cilium Summary

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: cilium-alerts
  namespace: monitoring
spec:
  groups:
    - name: cilium
      rules:
        - alert: CiliumAgentDown
          expr: up{job="cilium-agent"} == 0
          for: 2m
          annotations:
            summary: "Cilium agent down on {{ $labels.instance }}"
          labels:
            severity: critical

        - alert: CiliumEndpointNotReady
          expr: |
            cilium_endpoint_state{endpoint_state!="ready"} > 0
          for: 5m
          annotations:
            summary: "{{ $value }} Cilium endpoints not ready"
          labels:
            severity: warning

        - alert: HighDroppedPackets
          expr: |
            rate(cilium_drop_count_total[5m]) > 100
          for: 5m
          annotations:
            summary: "High packet drop rate: {{ $value }}/s on {{ $labels.reason }}"
          labels:
            severity: warning

        - alert: TetragonSecurityEvent
          expr: |
            rate(tetragon_events_total{type="PROCESS_EXEC"}[5m]) > 10
          for: 1m
          annotations:
            summary: "High rate of process executions detected"
          labels:
            severity: critical
```

---

## 📊 สรุป Part 96

| Feature | Cilium | Traditional (iptables) |
|---------|--------|------------------------|
| Rule lookup | O(1) hash | O(n) linear |
| L7 policy | HTTP/gRPC/DNS | No |
| Service mesh | Sidecarless eBPF | Sidecar required |
| Observability | Hubble flows | tcpdump only |
| BGP | Built-in | External MetalLB |
| Encryption | WireGuard/IPsec | External cert-manager |

---

## 🔗 ต่อไป
- [Part 97: Storage, CSI Drivers, and Rook-Ceph](./part-97-storage-csi.md)

---
*Part 96 | Steps 921-930 | Advanced Networking with eBPF and Cilium | Educational Content*
