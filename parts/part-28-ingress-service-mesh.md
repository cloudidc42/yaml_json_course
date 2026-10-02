# Part 28: Ingress Advanced และ Service Mesh
## Steps 271-280: Traffic Management ระดับ Production

---

## 📖 บทนำ

Ingress และ Service Mesh เป็น traffic management layers สำคัญที่ช่วยจัดการ routing, security, observability

---

## Step 271: NGINX Ingress Advanced

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: advanced-ingress
  namespace: production
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/limit-rps: "10"
    nginx.ingress.kubernetes.io/limit-connections: "5"
    nginx.ingress.kubernetes.io/enable-cors: "true"
    nginx.ingress.kubernetes.io/cors-allow-origin: "https://app.example.com"
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "60"
    nginx.ingress.kubernetes.io/configuration-snippet: |
      more_set_headers "X-Frame-Options: DENY";
      more_set_headers "X-Content-Type-Options: nosniff";
    nginx.ingress.kubernetes.io/affinity: "cookie"
    nginx.ingress.kubernetes.io/session-cookie-name: "INGRESSCOOKIE"
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - api.example.com
        - app.example.com
      secretName: example-tls
  rules:
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
```

---

## Step 272: cert-manager

```yaml
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
      - dns01:
          route53:
            region: ap-southeast-1
            hostedZoneID: ZXXXXX
          selector:
            dnsZones:
              - "*.example.com"

---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: example-cert
  namespace: production
spec:
  secretName: example-tls
  duration: 2160h
  renewBefore: 360h
  commonName: example.com
  dnsNames:
    - example.com
    - "*.example.com"
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
```

---

## Step 273: Traefik

```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: my-app
  namespace: production
spec:
  entryPoints:
    - websecure
  routes:
    - match: Host(`app.example.com`) && PathPrefix(`/api`)
      kind: Rule
      services:
        - name: api
          port: 8080
      middlewares:
        - name: rate-limit
    - match: Host(`app.example.com`)
      kind: Rule
      services:
        - name: frontend
          port: 80
  tls:
    certResolver: letsencrypt

---
apiVersion: traefik.containo.us/v1alpha1
kind: Middleware
metadata:
  name: rate-limit
  namespace: production
spec:
  rateLimit:
    average: 100
    period: 1m
    burst: 50
```

---

## Step 274: Istio Advanced Routing

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: my-app-progressive
  namespace: production
spec:
  hosts:
    - my-app
  http:
    # Header-based routing
    - match:
        - headers:
            x-user-type:
              exact: developer
      route:
        - destination:
            host: my-app
            subset: v2
    
    # Weight-based
    - route:
        - destination:
            host: my-app
            subset: v1
          weight: 80
        - destination:
            host: my-app
            subset: v2
          weight: 20
      fault:
        delay:
          percentage:
            value: 0.1
          fixedDelay: 5s
      timeout: 30s
      retries:
        attempts: 3
        perTryTimeout: 10s
        retryOn: "gateway-error,connect-failure,retriable-4xx"

---
# ServiceEntry
apiVersion: networking.istio.io/v1beta1
kind: ServiceEntry
metadata:
  name: external-api
  namespace: production
spec:
  hosts:
    - api.external.com
  ports:
    - number: 443
      name: https
      protocol: HTTPS
  location: MESH_EXTERNAL
  resolution: DNS
```

---

## Step 275: Istio Security

```yaml
apiVersion: security.istio.io/v1beta1
kind: RequestAuthentication
metadata:
  name: jwt-authn
  namespace: production
spec:
  selector:
    matchLabels:
      app: my-app
  jwtRules:
    - issuer: "https://accounts.google.com"
      jwksUri: "https://www.googleapis.com/oauth2/v3/certs"
      audiences:
        - "my-app-client-id"
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
      app: my-app
  rules:
    - from:
        - source:
            requestPrincipals: ["*"]
      to:
        - operation:
            methods: ["GET", "POST"]
            paths: ["/api/*"]
    - to:
        - operation:
            methods: ["GET"]
            paths: ["/healthz", "/readyz"]
```

---

## Step 276-277: DestinationRule และ Circuit Breaker

```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: api-service
  namespace: production
spec:
  host: api-service
  trafficPolicy:
    loadBalancer:
      simple: LEAST_CONN
    connectionPool:
      tcp:
        maxConnections: 1000
        connectTimeout: 30ms
      http:
        http2MaxRequests: 10000
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
```

---

## Step 278-279: Traffic Mirroring

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: api-mirroring
  namespace: production
spec:
  hosts:
    - api-service
  http:
    - route:
        - destination:
            host: api-service
            subset: v1
          weight: 100
      mirror:
        host: api-service
        subset: v2
      mirrorPercentage:
        value: 100
```

---

## Step 280: Workshop - Complete Setup

```yaml
# mTLS
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: production
spec:
  mtls:
    mode: STRICT

---
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: api-access
  namespace: production
spec:
  selector:
    matchLabels:
      app: api-service
  rules:
    - from:
        - source:
            principals:
              - "cluster.local/ns/production/sa/frontend"
      to:
        - operation:
            methods: ["GET", "POST"]
```

---

## 📊 สรุป Part 28

| Component | ใช้สำหรับ |
|-----------|----------|
| NGINX Ingress | HTTP/HTTPS ingress, rate limiting |
| cert-manager | Automatic TLS certificates |
| Traefik | Dynamic routing, middleware |
| Istio VirtualService | Advanced traffic routing |
| Istio DestinationRule | Load balancing, circuit breaker |
| Traffic Mirroring | Shadow testing |

---

## 🔗 ต่อไป
- [Part 30: Helm Package Manager](./part-30-helm.md)

---
*Part 28 | Steps 271-280 | ระดับสูง*
