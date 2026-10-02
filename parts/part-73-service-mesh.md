# Part 73: Service Mesh Deep Dive
## Steps 691-700: Istio, Linkerd, and Cilium

---

## Step 691: Service Mesh Overview

```
Service Mesh: infrastructure layer for service-to-service communication

Core capabilities:
  Traffic management:
    - Load balancing (round-robin, least-conn, consistent hash)
    - Traffic splitting (canary, A/B, blue-green)
    - Circuit breaking, retries, timeouts
    - Fault injection
    - Request routing (header/weight-based)
  
  Security:
    - mTLS between all services
    - Certificate rotation (SPIFFE/SPIRE)
    - Authorization policies (L7 RBAC)
  
  Observability:
    - Distributed tracing (Jaeger, Zipkin)
    - Metrics (Prometheus-compatible)
    - Service topology (Kiali)

Popular service meshes:
  Istio: feature-rich, Envoy sidecar
  Linkerd: lightweight, Rust proxy, CNCF graduated
  Cilium: eBPF-based, no sidecar, high performance
  Consul Connect: HashiCorp, multi-platform
```

---

## Step 692: Istio - Mutual TLS

```yaml
# Enforce mTLS in namespace
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: production
spec:
  mtls:
    mode: STRICT

---
# Cluster-wide mTLS
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: istio-system
spec:
  mtls:
    mode: STRICT

---
# Exception for legacy service
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: legacy-exception
  namespace: production
spec:
  selector:
    matchLabels:
      app: legacy-service
  mtls:
    mode: PERMISSIVE

# Verify mTLS:
# kubectl exec deploy/sleep -n production -- \
#   curl http://httpbin.production:8000/headers
# Look for x-forwarded-client-cert header
```

---

## Step 693: Istio - Authorization Policies

```yaml
# Deny all by default
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: deny-all
  namespace: production
spec: {}

---
# Allow frontend to call backend
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: allow-frontend-to-backend
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
# Allow Prometheus to scrape metrics
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: allow-prometheus-scrape
  namespace: production
spec:
  action: ALLOW
  rules:
    - from:
        - source:
            namespaces: ["monitoring"]
      to:
        - operation:
            ports: ["15090", "9090"]

---
# Require JWT
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: require-jwt
  namespace: production
spec:
  selector:
    matchLabels:
      app: api-gateway
  action: ALLOW
  rules:
    - from:
        - source:
            requestPrincipals: ["*"]
      when:
        - key: request.auth.claims[iss]
          values: ["https://accounts.google.com"]
```

---

## Step 694: Istio - Traffic Management

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: myapp
  namespace: production
spec:
  hosts:
    - myapp
  http:
    # Canary header routing
    - match:
        - headers:
            x-canary:
              exact: "true"
      route:
        - destination:
            host: myapp
            subset: v2
    # Weight-based canary: 10% to v2
    - route:
        - destination:
            host: myapp
            subset: v1
          weight: 90
        - destination:
            host: myapp
            subset: v2
          weight: 10
      timeout: 10s
      retries:
        attempts: 3
        perTryTimeout: 5s
        retryOn: "5xx,gateway-error,connect-failure"

---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: myapp
  namespace: production
spec:
  host: myapp
  trafficPolicy:
    loadBalancer:
      simple: LEAST_CONN
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        http1MaxPendingRequests: 100
        http2MaxRequests: 1000
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 30s
      baseEjectionTime: 30s
      maxEjectionPercent: 50
  subsets:
    - name: v1
      labels:
        version: v1
    - name: v2
      labels:
        version: v2
```

---

## Step 695: Istio - Fault Injection

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ratings-fault
  namespace: production
spec:
  hosts:
    - ratings
  http:
    # 10% of requests get 7s delay
    - fault:
        delay:
          percentage:
            value: 10
          fixedDelay: 7s
      route:
        - destination:
            host: ratings
            subset: v1
    
    # 10% of requests get 500 error
    - fault:
        abort:
          percentage:
            value: 10
          httpStatus: 500
      route:
        - destination:
            host: ratings
            subset: v1
    
    - route:
        - destination:
            host: ratings
            subset: v1

---
# Circuit breaker via outlier detection
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: circuit-breaker
  namespace: production
spec:
  host: external-api
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 10
      http:
        http1MaxPendingRequests: 10
        maxRequestsPerConnection: 1
    outlierDetection:
      consecutive5xxErrors: 3
      interval: 10s
      baseEjectionTime: 1m
      maxEjectionPercent: 100
```

---

## Step 696: Linkerd

```yaml
# Enable Linkerd injection
apiVersion: v1
kind: Namespace
metadata:
  name: production
  annotations:
    linkerd.io/inject: enabled

---
# ServiceProfile: retries and timeouts
apiVersion: linkerd.io/v1alpha2
kind: ServiceProfile
metadata:
  name: myapp.production.svc.cluster.local
  namespace: production
spec:
  routes:
    - name: GET /api/users
      condition:
        method: GET
        pathRegex: /api/users.*
      responseClasses:
        - condition:
            status:
              min: 500
              max: 599
          isFailure: true
      timeout: 5s
      isRetryable: true
    
    - name: POST /api/orders
      condition:
        method: POST
        pathRegex: /api/orders
      timeout: 30s
      isRetryable: false

---
# Linkerd mTLS authorization
apiVersion: policy.linkerd.io/v1beta1
kind: Server
metadata:
  name: backend-server
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend
  port: 8080
  proxyProtocol: HTTP/2

---
apiVersion: policy.linkerd.io/v1beta1
kind: ServerAuthorization
metadata:
  name: allow-frontend
  namespace: production
spec:
  server:
    name: backend-server
  client:
    meshTLS:
      serviceAccounts:
        - name: frontend
          namespace: production
```

---

## Step 697: Cilium Service Mesh (eBPF)

```yaml
# Cilium L7 HTTP policy
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: api-l7-policy
  namespace: production
spec:
  endpointSelector:
    matchLabels:
      app: api
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
                path: "/api/orders"
                headers:
                  - "Content-Type: application/json"

---
# Block external egress with allowlist
apiVersion: "cilium.io/v2"
kind: CiliumClusterwideNetworkPolicy
metadata:
  name: deny-external-egress
spec:
  endpointSelector:
    matchLabels:
      environment: production
  egressDeny:
    - toEntities:
        - world
  egress:
    - toEntities:
        - cluster
    - toFQDNs:
        - matchPattern: "*.mycompany.com"

---
# Cilium mutual auth
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: mutual-auth
  namespace: production
spec:
  endpointSelector:
    matchLabels:
      app: backend
  ingress:
    - fromEndpoints:
        - matchLabels:
            app: frontend
      authentication:
        mode: required
```

---

## Step 698: Service Mesh Observability

```yaml
apiVersion: telemetry.istio.io/v1alpha1
kind: Telemetry
metadata:
  name: custom-metrics
  namespace: production
spec:
  metrics:
    - providers:
        - name: prometheus
  tracing:
    - providers:
        - name: jaeger
      randomSamplingPercentage: 5.0

---
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: service-mesh-slos
  namespace: monitoring
spec:
  groups:
    - name: service-mesh
      rules:
        - alert: HighErrorRate
          expr: |
            sum(rate(istio_requests_total{response_code=~"5.*",destination_service_namespace="production"}[5m]))
            /
            sum(rate(istio_requests_total{destination_service_namespace="production"}[5m]))
            > 0.01
          for: 5m
          annotations:
            summary: "Production error rate > 1%"
          labels:
            severity: critical

        - alert: HighP99Latency
          expr: |
            histogram_quantile(0.99,
              sum(rate(istio_request_duration_milliseconds_bucket{destination_service_namespace="production"}[5m]))
              by (le, destination_service_name)
            ) > 500
          for: 5m
          annotations:
            summary: "P99 latency > 500ms for {{ $labels.destination_service_name }}"
          labels:
            severity: warning

        - alert: mTLSNotEnforced
          expr: istio_requests_total{connection_security_policy="none",destination_service_namespace="production"} > 0
          for: 1m
          annotations:
            summary: "Non-mTLS traffic detected in production"
          labels:
            severity: critical
```

---

## Step 699: Service Mesh Migration

```yaml
# Phase 1: PERMISSIVE (accept both plaintext and mTLS)
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: legacy-app
spec:
  mtls:
    mode: PERMISSIVE

---
# Phase 2: Inject sidecars into new namespace
apiVersion: v1
kind: Namespace
metadata:
  name: new-services
  labels:
    istio-injection: enabled

---
# Phase 3: STRICT for new services
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: new-services
spec:
  mtls:
    mode: STRICT

---
# Opt-out specific pods from sidecar injection
apiVersion: v1
kind: Pod
metadata:
  name: excluded-pod
  annotations:
    sidecar.istio.io/inject: "false"
spec:
  containers:
    - name: app
      image: myapp:1.0
```

---

## Step 700: Workshop - Service Mesh Checklist

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: service-mesh-checklist
  namespace: istio-system
data:
  checklist.yaml: |
    mesh_selection:
      - Istio: full features, Envoy, complex
      - Linkerd: lightweight, low overhead, simpler
      - Cilium: eBPF, no sidecar, kernel-level
    
    security_hardening:
      - Enable STRICT mTLS cluster-wide
      - Deploy deny-all AuthorizationPolicies
      - Allowlist service-to-service calls
      - Enable JWT validation at ingress
    
    traffic_management:
      - Set retries and timeouts per service
      - Configure circuit breakers
      - Implement canary via weights
      - Add fault injection tests
    
    observability:
      - Kiali for topology visualization
      - Jaeger for distributed tracing
      - Grafana Istio dashboards
      - SLO alerts (error rate, P99 latency)
    
    performance:
      - Monitor sidecar CPU/memory overhead
      - Tune connection pool settings
      - Use exclude annotations for non-mesh workloads

---
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: mesh-coverage
  namespace: monitoring
spec:
  groups:
    - name: mesh
      rules:
        - record: mesh:coverage:ratio
          expr: |
            count(kube_pod_labels{label_security_istio_io_tlsMode="istio"})
            /
            count(kube_pod_info{namespace=~"production|staging"})
        
        - alert: LowMeshCoverage
          expr: mesh:coverage:ratio < 0.95
          for: 5m
          annotations:
            summary: "Less than 95% of pods are in the service mesh"
          labels:
            severity: warning
```

---

## 📊 สรุป Part 73

| Feature | Istio | Linkerd | Cilium |
|---------|-------|---------|--------|
| Proxy | Envoy (C++) | Linkerd2 (Rust) | eBPF (kernel) |
| Sidecar | Yes | Yes | No |
| mTLS | SPIFFE/X.509 | SPIFFE | SPIFFE/SPIRE |
| L7 policy | Full | HTTP/gRPC | HTTP |
| Overhead | ~50-100ms | ~1ms | ~minimal |
| Complexity | High | Low | Medium |

---

## 🔗 ต่อไป
- [Part 74: Kubernetes Storage and StatefulSets](./part-74-storage.md)

---
*Part 73 | Steps 691-700 | Service Mesh | Educational Content*
