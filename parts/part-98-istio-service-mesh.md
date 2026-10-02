# Part 98: Microservices Patterns with Istio
## Steps 941-950: Service Mesh, Traffic Management, mTLS, Observability

---

## Step 941: Service Mesh Overview

```
Service Mesh Architecture:

What is a Service Mesh?
  - Infrastructure layer for service-to-service communication
  - Sidecar proxy (Envoy) injected into every pod
  - Intercepts all traffic (transparent to application)

Capabilities:
  Traffic Management:
    - Load balancing (round-robin, least-conn, consistent hash)
    - Traffic splitting (A/B testing, canary)
    - Circuit breaking, retries, timeouts
    - Fault injection (for chaos testing)
  
  Security:
    - mTLS: automatic mutual TLS between services
    - Authorization policies (deny by default)
    - JWT validation, OIDC integration
  
  Observability:
    - Distributed traces (OpenTelemetry, Jaeger)
    - Golden metrics per service (RED: Rate, Errors, Duration)
    - Service dependency graph (Kiali)

Istio Components:
  istiod: control plane (Pilot + Citadel + Galley merged)
  Envoy sidecar: data plane (injected via MutatingWebhook)
  Ingress Gateway: replaces Nginx/Traefik for L7
  Egress Gateway: control outbound traffic

When to use Istio:
  YES: microservices > 5, need mTLS, need canary routing
  NO: monolith, simple apps, overhead concern
  
Alternatives:
  Linkerd: lighter weight, Rust proxy
  Consul Connect: HashiCorp ecosystem
  Cilium Service Mesh: eBPF sidecarless (no Envoy per pod)
```

---

## Step 942: Istio Installation

```yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  name: istio-config
  namespace: istio-system
spec:
  profile: production
  meshConfig:
    enableTracing: true
    accessLogFile: /dev/stdout
    defaultConfig:
      tracing:
        sampling: 1.0
        zipkin:
          address: jaeger-collector.monitoring:9411
    outboundTrafficPolicy:
      mode: REGISTRY_ONLY
  values:
    global:
      proxy:
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 256Mi
    pilot:
      autoscaleMin: 2
      autoscaleMax: 5
```

---

## Step 943: Traffic Management - VirtualService

```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: payment-api
  namespace: production
spec:
  host: payment-api
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        http2MaxRequests: 1000
        pendingHttpRequests: 100
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 10s
      baseEjectionTime: 30s
      maxEjectionPercent: 50
  subsets:
    - name: v1
      labels:
        version: v1
    - name: v2
      labels:
        version: v2

---
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: payment-api
  namespace: production
spec:
  hosts:
    - payment-api
  http:
    - match:
        - headers:
            x-user-group:
              exact: beta
      route:
        - destination:
            host: payment-api
            subset: v2
    
    - route:
        - destination:
            host: payment-api
            subset: v1
          weight: 90
        - destination:
            host: payment-api
            subset: v2
          weight: 10
      timeout: 10s
      retries:
        attempts: 3
        perTryTimeout: 3s
        retryOn: connect-failure,refused-stream,5xx
```

---

## Step 944: Circuit Breaking and Fault Injection

```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: payment-api-cb
  namespace: production
spec:
  host: payment-api
  trafficPolicy:
    outlierDetection:
      consecutive5xxErrors: 3
      interval: 5s
      baseEjectionTime: 1m
      maxEjectionPercent: 100
    connectionPool:
      http:
        http1MaxPendingRequests: 50
        http2MaxRequests: 100

---
# Fault injection for chaos testing
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: payment-api-chaos
  namespace: production
spec:
  hosts:
    - payment-api
  http:
    - match:
        - headers:
            x-chaos-test:
              exact: "true"
      fault:
        delay:
          percentage:
            value: 50.0
          fixedDelay: 2s
        abort:
          percentage:
            value: 10.0
          httpStatus: 500
      route:
        - destination:
            host: payment-api
    - route:
        - destination:
            host: payment-api
```

---

## Step 945: Mutual TLS

```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: istio-system
spec:
  mtls:
    mode: STRICT

---
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: deny-all
  namespace: production
spec: {}

---
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: allow-payment-api
  namespace: production
spec:
  selector:
    matchLabels:
      app: payment-api
  action: ALLOW
  rules:
    - from:
        - source:
            principals:
              - cluster.local/ns/production/sa/frontend
              - cluster.local/ns/production/sa/order-service
      to:
        - operation:
            methods: [GET, POST]
            paths: ["/api/payments*"]
    - from:
        - source:
            namespaces: [monitoring]
      to:
        - operation:
            ports: ["9090"]
```

---

## Step 946: JWT Authentication

```yaml
apiVersion: security.istio.io/v1beta1
kind: RequestAuthentication
metadata:
  name: jwt-auth
  namespace: production
spec:
  selector:
    matchLabels:
      app: payment-api
  jwtRules:
    - issuer: https://auth.mycompany.com
      jwksUri: https://auth.mycompany.com/.well-known/jwks.json
      audiences:
        - payment-api
      forwardOriginalToken: true

---
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: require-jwt
  namespace: production
spec:
  selector:
    matchLabels:
      app: payment-api
  action: ALLOW
  rules:
    - from:
        - source:
            requestPrincipals: ["https://auth.mycompany.com/*"]
      when:
        - key: request.auth.claims[scope]
          values: [payments:write, payments:read]
```

---

## Step 947: Istio Gateway (Ingress)

```yaml
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: production-gateway
  namespace: istio-system
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
        credentialName: api-mycompany-tls
      hosts:
        - api.mycompany.com
    - port:
        number: 80
        name: http
        protocol: HTTP
      tls:
        httpsRedirect: true
      hosts:
        - "*"

---
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: payment-api-external
  namespace: production
spec:
  hosts:
    - api.mycompany.com
  gateways:
    - istio-system/production-gateway
  http:
    - match:
        - uri:
            prefix: /api/payments
      route:
        - destination:
            host: payment-api.production.svc.cluster.local
            port:
              number: 8080
      corsPolicy:
        allowOrigins:
          - exact: https://app.mycompany.com
        allowMethods: [GET, POST, OPTIONS]
        allowHeaders: [Authorization, Content-Type]
        maxAge: 24h
```

---

## Step 948: Egress Control

```yaml
apiVersion: networking.istio.io/v1beta1
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
  location: MESH_EXTERNAL
  resolution: DNS

---
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: stripe-api
  namespace: production
spec:
  hosts:
    - api.stripe.com
  http:
    - timeout: 10s
      retries:
        attempts: 3
        perTryTimeout: 3s
        retryOn: connect-failure,5xx
      route:
        - destination:
            host: api.stripe.com
            port:
              number: 443
```

---

## Step 949: Kiali Service Graph

```yaml
apiVersion: kiali.io/v1alpha1
kind: Kiali
metadata:
  name: kiali
  namespace: istio-system
spec:
  istio_namespace: istio-system
  auth:
    strategy: openid
    openid:
      client_id: kiali
      issuer_uri: https://auth.mycompany.com
      username_claim: email
  deployment:
    ingress:
      class_name: nginx
      enabled: true
  external_services:
    prometheus:
      url: http://prometheus-operated.monitoring:9090
    grafana:
      url: https://grafana.mycompany.com
    tracing:
      enabled: true
      in_cluster_url: http://jaeger-query.monitoring:16686
```

---

## Step 950: Workshop - Istio Summary

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: istio-slo-alerts
  namespace: monitoring
spec:
  groups:
    - name: istio-slo
      rules:
        - record: job:istio_requests:rate5m
          expr: |
            sum(rate(istio_requests_total[5m])) by (destination_service_name, destination_service_namespace)
        
        - record: job:istio_errors:rate5m
          expr: |
            sum(rate(istio_requests_total{response_code=~"5.."}[5m])) by (destination_service_name, destination_service_namespace)
        
        - alert: IstioHighErrorRate
          expr: |
            job:istio_errors:rate5m / job:istio_requests:rate5m > 0.05
          for: 5m
          annotations:
            summary: "{{ $labels.destination_service_name }} error rate > 5%"
          labels:
            severity: critical

        - alert: IstioHighLatency
          expr: |
            histogram_quantile(0.99,
              rate(istio_request_duration_milliseconds_bucket[5m])
            ) > 1000
          for: 5m
          annotations:
            summary: "{{ $labels.destination_service_name }} P99 latency > 1s"
          labels:
            severity: warning

        - alert: IstioCertExpiringSoon
          expr: |
            (istio_agent_cert_expiry_seconds - time()) < 86400
          for: 0m
          annotations:
            summary: "Istio certificate expiring < 24h on {{ $labels.instance }}"
          labels:
            severity: warning
```

---

## 📊 สรุป Part 98

| Feature | Istio | Linkerd | Cilium Service Mesh |
|---------|-------|---------|---------------------|
| Proxy | Envoy (C++) | Linkerd2-proxy (Rust) | None (eBPF) |
| mTLS | Yes | Yes | Yes (WireGuard) |
| L7 Policy | Yes | Yes | Yes |
| Overhead/pod | ~100ms CPU, 128Mi | ~10ms CPU, 50Mi | Minimal |
| Maturity | High | High | Growing |
| Best for | Full-featured | Low overhead | eBPF-first |

---

## 🔗 ต่อไป
- [Part 99: Testing, Validation, and Quality Assurance](./part-99-testing-validation.md)

---
*Part 98 | Steps 941-950 | Microservices Patterns with Istio | Educational Content*
