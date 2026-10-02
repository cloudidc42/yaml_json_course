# Part 87: API Gateway and Microservices
## Steps 831-840: Kong, Rate Limiting, gRPC, GraphQL

---

## Step 831: API Gateway Overview

```
API Gateway Patterns for Kubernetes:

Why API Gateway (not just Ingress):
  - Rate limiting per client/API key
  - Authentication (JWT, OAuth2, API keys)
  - Request/response transformation
  - Analytics and logging per endpoint
  - Developer portal + API documentation
  - Monetization (quota, billing)

Popular API Gateways:
  Kong Gateway:
    - Most popular open-source
    - Plugin ecosystem (100+ plugins)
    - Helm chart for K8s
  
  Traefik:
    - Cloud-native, K8s-first
    - Automatic service discovery
    - IngressRoute CRD
  
  Envoy Gateway:
    - Gateway API implementation
    - Backed by CNCF
  
  AWS API Gateway:
    - Fully managed
    - Integrates with Lambda, ECS

Gateway API (K8s standard):
  - Gateway, HTTPRoute, GRPCRoute, TCPRoute
  - Standard across all gateway implementations
  - Replacing Ingress API
```

---

## Step 832: Kong Gateway Setup

```yaml
# Kong Helm values.yaml
proxy:
  type: LoadBalancer
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: nlb

admin:
  enabled: true
  type: ClusterIP

ingressController:
  enabled: true
  ingressClass: kong

database:
  type: postgres

postgresql:
  enabled: true
  auth:
    postgresPassword: changeme
    database: kong

env:
  plugins: bundled,rate-limiting,jwt,cors,prometheus,request-transformer

---
# KongIngress: advanced routing config
apiVersion: configuration.konghq.com/v1
kind: KongIngress
metadata:
  name: myapp-kong-ingress
  namespace: production
proxy:
  connect_timeout: 5000
  read_timeout: 30000
  write_timeout: 30000
  retries: 3

upstream:
  algorithm: round-robin
  healthchecks:
    active:
      healthy:
        interval: 5
        successes: 2
      unhealthy:
        interval: 5
        http_failures: 3
```

---

## Step 833: Rate Limiting

```yaml
apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  name: rate-limit-per-consumer
  namespace: production
plugin: rate-limiting
config:
  minute: 1000
  hour: 50000
  policy: redis
  redis_host: redis.cache.svc
  redis_port: 6379
  redis_timeout: 2000
  fault_tolerant: true
  hide_client_headers: false

---
apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  name: jwt-validation
  namespace: production
plugin: jwt
config:
  uri_param_names: [jwt]
  cookie_names: [token]
  key_claim_name: kid
  claims_to_verify: [exp, nbf]
  maximum_expiration: 3600

---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-api
  namespace: production
  annotations:
    kubernetes.io/ingress.class: kong
    konghq.com/plugins: rate-limit-per-consumer,jwt-validation
    konghq.com/strip-path: "true"
spec:
  rules:
    - host: api.mycompany.com
      http:
        paths:
          - path: /api/v1
            pathType: Prefix
            backend:
              service:
                name: myapp
                port:
                  number: 8080

---
apiVersion: configuration.konghq.com/v1
kind: KongConsumer
metadata:
  name: mobile-app
  namespace: production
  annotations:
    kubernetes.io/ingress.class: kong
username: mobile-app
custom_id: app-001
```

---

## Step 834: gRPC on Kubernetes

```yaml
apiVersion: v1
kind: Service
metadata:
  name: grpc-service
  namespace: production
  annotations:
    alb.ingress.kubernetes.io/backend-protocol-version: GRPC
spec:
  selector:
    app: grpc-service
  ports:
    - name: grpc
      port: 50051
      targetPort: 50051

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grpc-service
  namespace: production
spec:
  template:
    spec:
      containers:
        - name: grpc-app
          image: grpc-app:1.0
          ports:
            - containerPort: 50051
          readinessProbe:
            grpc:
              port: 50051
            initialDelaySeconds: 5
            periodSeconds: 10
          livenessProbe:
            grpc:
              port: 50051

---
# Ingress: gRPC via nginx
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: grpc-ingress
  namespace: production
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/backend-protocol: GRPC
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  tls:
    - hosts:
        - grpc.mycompany.com
      secretName: grpc-tls-secret
  rules:
    - host: grpc.mycompany.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: grpc-service
                port:
                  number: 50051

---
# Istio: force HTTP/2 for gRPC
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: grpc-service
  namespace: production
spec:
  host: grpc-service
  trafficPolicy:
    connectionPool:
      http:
        h2UpgradePolicy: UPGRADE
```

---

## Step 835: GraphQL API

```yaml
# Hasura: GraphQL over PostgreSQL
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hasura
  namespace: production
spec:
  replicas: 2
  template:
    spec:
      containers:
        - name: hasura
          image: hasura/graphql-engine:v2.36
          env:
            - name: HASURA_GRAPHQL_DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: db-credentials
                  key: url
            - name: HASURA_GRAPHQL_ADMIN_SECRET
              valueFrom:
                secretKeyRef:
                  name: hasura-secret
                  key: admin-secret
            - name: HASURA_GRAPHQL_ENABLE_CONSOLE
              value: "false"
            - name: HASURA_GRAPHQL_JWT_SECRET
              valueFrom:
                secretKeyRef:
                  name: hasura-secret
                  key: jwt-secret
            - name: HASURA_GRAPHQL_CORS_DOMAIN
              value: "https://app.mycompany.com"
          ports:
            - containerPort: 8080
          readinessProbe:
            httpGet:
              path: /healthz
              port: 8080
          resources:
            requests:
              cpu: 200m
              memory: 256Mi
            limits:
              cpu: 1000m
              memory: 1Gi
```

---

## Step 836: API Versioning

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: api-versioning
  namespace: production
spec:
  hosts:
    - api.mycompany.com
  gateways:
    - production/api-gateway
  http:
    - match:
        - uri:
            prefix: /api/v2/
      rewrite:
        uri: /
      route:
        - destination:
            host: myapp-v2
            port:
              number: 8080
    
    - match:
        - uri:
            prefix: /api/v1/
      rewrite:
        uri: /
      route:
        - destination:
            host: myapp-v1
            port:
              number: 8080
    
    - match:
        - uri:
            exact: /api
      redirect:
        uri: /api/v1/
        redirectCode: 301
```

---

## Step 837: Distributed Rate Limiting

```yaml
# Redis for rate limiting state
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis-ratelimit
  namespace: production
spec:
  replicas: 1
  template:
    spec:
      containers:
        - name: redis
          image: redis:7-alpine
          command: [redis-server, --maxmemory, 256mb, --maxmemory-policy, allkeys-lru]
          ports:
            - containerPort: 6379
          resources:
            requests:
              cpu: 100m
              memory: 256Mi
            limits:
              cpu: 500m
              memory: 512Mi
```

---

## Step 838: Service Discovery

```yaml
# Headless service: direct pod DNS
apiVersion: v1
kind: Service
metadata:
  name: myapp-headless
  namespace: production
spec:
  clusterIP: None
  selector:
    app: myapp
  ports:
    - port: 8080

# DNS: myapp-headless.production.svc.cluster.local
# Returns A records for each pod directly

---
# ExternalName: proxy to external
apiVersion: v1
kind: Service
metadata:
  name: legacy-api
  namespace: production
spec:
  type: ExternalName
  externalName: legacy-api.mycompany.com
```

---

## Step 839: OpenAPI Documentation

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: swagger-ui
  namespace: production
spec:
  replicas: 1
  template:
    spec:
      containers:
        - name: swagger-ui
          image: swaggerapi/swagger-ui:v5
          env:
            - name: SWAGGER_JSON_URL
              value: https://api.mycompany.com/openapi.json
            - name: BASE_URL
              value: /docs
          ports:
            - containerPort: 8080
          resources:
            requests:
              cpu: 50m
              memory: 64Mi

---
apiVersion: v1
kind: ConfigMap
metadata:
  name: openapi-spec
  namespace: production
data:
  openapi.yaml: |
    openapi: "3.0.3"
    info:
      title: MyApp API
      version: "1.0.0"
    servers:
      - url: https://api.mycompany.com/api/v1
    security:
      - BearerAuth: []
    paths:
      /users:
        get:
          summary: List users
          responses:
            "200":
              description: User list
    components:
      securitySchemes:
        BearerAuth:
          type: http
          scheme: bearer
          bearerFormat: JWT
```

---

## Step 840: Workshop - API Gateway Monitoring

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: api-gateway-alerts
  namespace: monitoring
spec:
  groups:
    - name: api-gateway
      rules:
        - alert: KongHighErrorRate
          expr: |
            sum(rate(kong_http_requests_total{code=~"5.."}[5m]))
            /
            sum(rate(kong_http_requests_total[5m]))
            > 0.01
          for: 5m
          annotations:
            summary: "Kong API Gateway error rate > 1%"
          labels:
            severity: warning

        - alert: APIRateLimitExceeded
          expr: |
            sum(rate(kong_http_requests_total{code="429"}[5m]))
            by (service) > 10
          for: 2m
          annotations:
            summary: "Rate limit exceeded for service {{ $labels.service }}"
          labels:
            severity: info

        - alert: APILatencyHigh
          expr: |
            histogram_quantile(0.99,
              sum(rate(kong_latency_bucket[5m])) by (le, service)
            ) > 2000
          for: 5m
          annotations:
            summary: "API {{ $labels.service }} P99 latency > 2s"
          labels:
            severity: warning
```

---

## 📊 สรุป Part 87

| Feature | Tool | Config |
|---------|------|--------|
| Rate limiting | Kong + Redis | KongPlugin rate-limiting |
| JWT auth | Kong JWT plugin | KongConsumer + credentials |
| gRPC routing | nginx/Istio | HTTP/2 backend protocol |
| GraphQL | Hasura + Kong | GraphQL rate-limiting |
| API versioning | Istio VirtualService | URI/header matching |
| Docs | Swagger UI | ConfigMap + Deployment |

---

## 🔗 ต่อไป
- [Part 88: Database Patterns on Kubernetes](./part-88-database-patterns.md)

---
*Part 87 | Steps 831-840 | API Gateway and Microservices | Educational Content*
