# Part 85: Service Mesh with Istio
## Steps 811-820: Traffic Management, mTLS, Observability

---

## Step 811: Istio Architecture Overview

```
Istio Service Mesh Architecture:

Control Plane (istiod):
  - Pilot: service discovery, traffic rules -> Envoy config
  - Citadel: certificate management (SPIFFE/SPIRE compatible)
  - Galley: config validation and distribution

Data Plane:
  - Envoy sidecar: injected into every pod
  - Intercepts all inbound/outbound traffic
  - Reports telemetry to control plane

Key Features:
  Traffic Management:
    - Load balancing (round-robin, least-conn, random, consistent hash)
    - Traffic splitting (canary, A/B testing)
    - Circuit breaker (outlier detection)
    - Fault injection (delays, aborts)
    - Retry and timeout policies
    - Mirror (traffic shadowing)

  Security:
    - mTLS: mutual TLS between all services (automatic)
    - PeerAuthentication: enforce mTLS strictness
    - AuthorizationPolicy: RBAC for service-to-service
    - JWT validation (RequestAuthentication)

  Observability:
    - Distributed tracing (Jaeger/Zipkin)
    - Metrics (Prometheus via Envoy stats)
    - Access logs (structured)

Install:
  istioctl install --set profile=production
  # Profiles: minimal, default, demo, production
```

---

## Step 812: Sidecar Injection and PeerAuthentication

```yaml
# Enable automatic sidecar injection per namespace
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    istio-injection: enabled

---
# Enforce mTLS across namespace
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: production
spec:
  mtls:
    mode: STRICT

---
# Port-specific mTLS for legacy service
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: legacy-service
  namespace: production
spec:
  selector:
    matchLabels:
      app: legacy-app
  mtls:
    mode: PERMISSIVE
  portLevelMtls:
    8080:
      mode: DISABLE
```

---

## Step 813: VirtualService Traffic Routing

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: myapp
  namespace: production
spec:
  hosts:
    - myapp
    - myapp.mycompany.com
  gateways:
    - production/myapp-gateway
    - mesh
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
    
    # URI rewrite
    - match:
        - uri:
            prefix: /api/v2/
      rewrite:
        uri: /api/
      route:
        - destination:
            host: myapp
            subset: v2
    
    # 90/10 weight split
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
        perTryTimeout: 3s
        retryOn: "5xx,reset,connect-failure"

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
        http2MaxRequests: 1000
        maxRetries: 3
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 30s
      baseEjectionTime: 30s
  subsets:
    - name: v1
      labels:
        version: v1
    - name: v2
      labels:
        version: v2
```

---

## Step 814: Istio Gateway

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: myapp-gateway
  namespace: production
spec:
  selector:
    istio: ingressgateway
  servers:
    - port:
        number: 443
        name: https
        protocol: HTTPS
      tls:
        mode: SIMPLE
        credentialName: myapp-tls-secret
      hosts:
        - myapp.mycompany.com
        - api.myapp.mycompany.com
    
    - port:
        number: 80
        name: http
        protocol: HTTP
      hosts:
        - myapp.mycompany.com
      tls:
        httpsRedirect: true

---
# Multi-service routing through one gateway
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: api-routing
  namespace: production
spec:
  hosts:
    - api.myapp.mycompany.com
  gateways:
    - production/myapp-gateway
  http:
    - match:
        - uri:
            prefix: /users
      route:
        - destination:
            host: user-service
            port:
              number: 8080
    
    - match:
        - uri:
            prefix: /orders
      route:
        - destination:
            host: order-service
            port:
              number: 8080
    
    - match:
        - uri:
            prefix: /products
      route:
        - destination:
            host: product-service
            port:
              number: 8080
```

---

## Step 815: Fault Injection

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: payment-fault-test
  namespace: production
spec:
  hosts:
    - payment-service
  http:
    - fault:
        delay:
          percentage:
            value: 10
          fixedDelay: 500ms
        abort:
          percentage:
            value: 5
          httpStatus: 503
      route:
        - destination:
            host: payment-service

---
# Traffic mirroring
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: myapp-mirror
  namespace: production
spec:
  hosts:
    - myapp
  http:
    - route:
        - destination:
            host: myapp
            subset: v1
          weight: 100
      mirror:
        host: myapp
        subset: v2
      mirrorPercentage:
        value: 100.0
```

---

## Step 816: AuthorizationPolicy

```yaml
# Default deny all
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: deny-all
  namespace: production
spec: {}

---
# Allow only order-service to call payment-service
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: allow-order-to-payment
  namespace: production
spec:
  selector:
    matchLabels:
      app: payment-service
  action: ALLOW
  rules:
    - from:
        - source:
            principals:
              - cluster.local/ns/production/sa/order-service
      to:
        - operation:
            methods: [POST]
            paths: [/api/charge]

---
# Allow monitoring to scrape
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: allow-prometheus
  namespace: production
spec:
  selector:
    matchLabels: {}
  action: ALLOW
  rules:
    - from:
        - source:
            namespaces: [monitoring]
      to:
        - operation:
            ports: ["15090", "15020"]
```

---

## Step 817: Telemetry and Service Entries

```yaml
apiVersion: telemetry.istio.io/v1alpha1
kind: Telemetry
metadata:
  name: production-telemetry
  namespace: production
spec:
  tracing:
    - providers:
        - name: tempo
      randomSamplingPercentage: 1.0
      customTags:
        team:
          literal:
            value: team-alpha
        version:
          header:
            name: x-app-version
  accessLogging:
    - providers:
        - name: otel

---
# ServiceEntry: register external service in mesh
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: stripe-api
  namespace: production
spec:
  hosts:
    - api.stripe.com
  ports:
    - number: 443
      name: https
      protocol: HTTPS
  resolution: DNS
  location: MESH_EXTERNAL
```

---

## Step 818: JWT Authentication

```yaml
apiVersion: security.istio.io/v1beta1
kind: RequestAuthentication
metadata:
  name: jwt-auth
  namespace: production
spec:
  selector:
    matchLabels:
      app: api-service
  jwtRules:
    - issuer: https://accounts.google.com
      jwksUri: https://www.googleapis.com/oauth2/v3/certs
      audiences:
        - myapp-api
      forwardOriginalToken: true
    
    - issuer: https://auth.mycompany.com
      jwksUri: https://auth.mycompany.com/.well-known/jwks.json
      audiences:
        - myapp-production

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
      app: api-service
  action: DENY
  rules:
    - from:
        - source:
            notRequestPrincipals: ["*"]

---
# Allow public endpoints
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: allow-public-endpoints
  namespace: production
spec:
  selector:
    matchLabels:
      app: api-service
  action: ALLOW
  rules:
    - to:
        - operation:
            methods: [GET]
            paths: [/health, /metrics, /api/v1/public/*]
```

---

## Step 819: Multi-Cluster Istio

```yaml
# Primary cluster IstioOperator
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  name: istio-primary
spec:
  profile: default
  values:
    global:
      meshID: mesh1
      multiCluster:
        clusterName: cluster-us-east-1
      network: network1

---
# Remote cluster IstioOperator
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  name: istio-remote
spec:
  profile: remote
  values:
    global:
      meshID: mesh1
      multiCluster:
        clusterName: cluster-eu-west-1
      network: network2
      remotePilotAddress: 1.2.3.4

---
# Cross-cluster service endpoint
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: remote-user-service
  namespace: production
spec:
  hosts:
    - user-service.production.svc.cluster.local
  location: MESH_INTERNAL
  ports:
    - number: 8080
      name: http
      protocol: HTTP
  resolution: STATIC
  endpoints:
    - address: 10.20.30.40
      locality: eu-west-1/eu-west-1a
      labels:
        version: v1
```

---

## Step 820: Workshop - Istio PrometheusRules

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: istio-alerts
  namespace: monitoring
spec:
  groups:
    - name: istio
      rules:
        - alert: IstioHighError5xxRate
          expr: |
            sum(rate(istio_requests_total{
              reporter="destination",
              response_code=~"5.*"
            }[5m])) by (destination_service_name)
            /
            sum(rate(istio_requests_total{
              reporter="destination"
            }[5m])) by (destination_service_name)
            > 0.01
          for: 5m
          annotations:
            summary: "Service {{ $labels.destination_service_name }} error rate > 1%"
          labels:
            severity: warning

        - alert: IstioHighLatency
          expr: |
            histogram_quantile(0.99,
              sum(rate(istio_request_duration_milliseconds_bucket{
                reporter="destination"
              }[5m])) by (le, destination_service_name)
            ) > 1000
          for: 10m
          annotations:
            summary: "Service {{ $labels.destination_service_name }} P99 latency > 1s"
          labels:
            severity: warning

        - alert: IstiomTLSNotEnforced
          expr: |
            sum(istio_requests_total{connection_security_policy="none",
              reporter="destination"}) by (destination_service_name) > 0
          for: 5m
          annotations:
            summary: "Service {{ $labels.destination_service_name }} receiving non-mTLS traffic"
          labels:
            severity: critical
```

---

## 📊 สรุป Part 85

| Feature | Resource | Purpose |
|---------|----------|---------|
| mTLS | PeerAuthentication | Encrypt + authenticate all traffic |
| Traffic routing | VirtualService + DestinationRule | A/B testing, canary |
| Ingress | Gateway | TLS termination, routing |
| Service RBAC | AuthorizationPolicy | Zero-trust network |
| JWT auth | RequestAuthentication | API authentication |
| Fault injection | VirtualService fault | Chaos/resilience testing |
| Tracing | Telemetry | Distributed tracing |

---

## 🔗 ต่อไป
- [Part 86: FinOps and Cost Optimization](./part-86-finops-cost.md)

---
*Part 85 | Steps 811-820 | Service Mesh with Istio | Educational Content*
