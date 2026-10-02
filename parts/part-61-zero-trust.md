# Part 61: Zero Trust Architecture
## Steps 571-580: Zero Trust in Kubernetes

---

> **⚠️**: เนื้อหาด้าน security ในหลักสูตรนี้มีไว้เพื่อการศึกษาและการทดสอบระบบที่ได้รับอนุญาตเท่านั้น

## Step 571: Zero Trust Principles for Kubernetes

```
Zero Trust: "Never Trust, Always Verify"

Kubernetes Zero Trust pillars:
  1. Identity: mTLS, SPIFFE/SPIRE, JWT
  2. Network: NetworkPolicy, service mesh, microsegmentation
  3. Workload: signed images, runtime security
  4. Data: encryption at rest/transit, secret management
  5. Visibility: audit logs, tracing, anomaly detection
```

## Step 572: SPIFFE/SPIRE Identity

```yaml
apiVersion: spire.spiffe.io/v1alpha1
kind: ClusterSPIFFEID
metadata:
  name: production-workloads
spec:
  spiffeIDTemplate: "spiffe://example.org/ns/{{ .PodMeta.Namespace }}/sa/{{ .PodSpec.ServiceAccountName }}"
  podSelector:
    matchLabels:
      spiffe-enabled: "true"
  workloadSelectorTemplates:
    - "k8s:ns:{{ .PodMeta.Namespace }}"
    - "k8s:sa:{{ .PodSpec.ServiceAccountName }}"
```

## Step 573: Istio Zero Trust

```yaml
# Strict mTLS mesh-wide
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: istio-system
spec:
  mtls:
    mode: STRICT

---
# OIDC-based user authentication
apiVersion: security.istio.io/v1beta1
kind: RequestAuthentication
metadata:
  name: user-jwt
  namespace: production
spec:
  selector:
    matchLabels:
      app: api-gateway
  jwtRules:
    - issuer: "https://auth.example.com"
      jwksUri: "https://auth.example.com/.well-known/jwks.json"
      audiences:
        - "api.example.com"
      outputClaimToHeaders:
        - header: x-user-id
          claim: sub

---
# Zero trust authorization: explicit allow only
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: frontend-zero-trust
  namespace: production
spec:
  selector:
    matchLabels:
      app: frontend
  action: ALLOW
  rules:
    - from:
        - source:
            principals:
              - "cluster.local/ns/ingress-nginx/sa/ingress-nginx"
      to:
        - operation:
            methods: ["GET", "POST"]
            ports: ["8080"]
```

## Step 574: Microsegmentation

```yaml
# Namespace-level isolation
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: namespace-isolation
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - podSelector: {}
  egress:
    - to:
        - podSelector: {}
    - ports:
        - port: 53
          protocol: UDP

---
# Cilium: microsegmentation at L7
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: micro-segment-api
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
                path: "^/api/v1/users/[0-9]+$"
              - method: POST
                path: "^/api/v1/orders$"
```

## Step 575: Workload Identity Federation

```yaml
# Projected SA token - short-lived, audience-scoped
apiVersion: v1
kind: Pod
metadata:
  name: payment-app
  namespace: production
spec:
  serviceAccountName: payment-service
  automountServiceAccountToken: false
  volumes:
    - name: payment-token
      projected:
        sources:
          - serviceAccountToken:
              audience: payment-gateway
              expirationSeconds: 3600
              path: token
  containers:
    - name: app
      image: myregistry.io/payment:1.0
      volumeMounts:
        - name: payment-token
          mountPath: /var/run/payment-token
          readOnly: true
      securityContext:
        readOnlyRootFilesystem: true
        allowPrivilegeEscalation: false
        runAsNonRoot: true
        capabilities:
          drop: [ALL]
```

## Step 576: Zero Trust Data Protection

```yaml
# Ingress: TLS 1.3 only
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: secure-ingress
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/ssl-protocols: "TLSv1.3"
spec:
  tls:
    - hosts:
        - api.example.com
      secretName: api-tls

---
# Database: require TLS
apiVersion: v1
kind: ConfigMap
metadata:
  name: postgres-config
  namespace: production
data:
  postgresql.conf: |
    ssl = on
    ssl_min_protocol_version = 'TLSv1.3'
  pg_hba.conf: |
    hostssl all all 0.0.0.0/0 scram-sha-256
    host    all all 0.0.0.0/0 reject
```

## Step 577: Continuous Verification

```yaml
# OPA: continuous policy evaluation
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: zerotrustrequirements
spec:
  crd:
    spec:
      names:
        kind: ZeroTrustRequirements
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package zerotrustrequirements
        
        # Deny: no resource limits
        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not container.resources.limits.cpu
          msg := sprintf("Container %v must have CPU limits", [container.name])
        }
        
        # Deny: writable rootfs in production
        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not container.securityContext.readOnlyRootFilesystem == true
          input.review.object.metadata.namespace == "production"
          msg := sprintf("Container %v must have readOnlyRootFilesystem: true", [container.name])
        }

---
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: ZeroTrustRequirements
metadata:
  name: production-zero-trust
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
    namespaces: ["production"]
```

## Step 578: Zero Trust Networking

```yaml
# Istio: enforce strict source verification
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: zero-trust-database
  namespace: production
spec:
  selector:
    matchLabels:
      app: postgres
  action: ALLOW
  rules:
    - from:
        - source:
            principals:
              - "cluster.local/ns/production/sa/backend-sa"
              - "cluster.local/ns/production/sa/migration-sa"
      to:
        - operation:
            ports: ["5432"]
```

## Step 579: Zero Trust Observability

```yaml
# OpenTelemetry: trace security events
apiVersion: v1
kind: ConfigMap
metadata:
  name: otel-collector-config
  namespace: monitoring
data:
  config.yaml: |
    receivers:
      otlp:
        protocols:
          grpc:
            endpoint: 0.0.0.0:4317
    processors:
      attributes:
        actions:
          - key: kubernetes.cluster.name
            value: production-cluster
            action: upsert
    exporters:
      jaeger:
        endpoint: jaeger-collector:14250
    service:
      pipelines:
        traces:
          receivers: [otlp]
          processors: [attributes]
          exporters: [jaeger]
```

## Step 580: Workshop - Zero Trust Checklist

```yaml
# Zero Trust maturity levels:

# Level 1 - Basic:
# [x] TLS everywhere (ingress + service mesh)
# [x] NetworkPolicy: default deny
# [x] RBAC: least privilege

# Level 2 - Intermediate:
# [x] mTLS: Istio PeerAuthentication STRICT
# [x] JWT: user authentication at gateway
# [x] IRSA/Workload Identity: no long-lived keys

# Level 3 - Advanced:
# [x] SPIFFE/SPIRE: workload identity federation
# [x] OPA: continuous policy evaluation
# [x] Falco: runtime anomaly detection

# Level 4 - Mature:
# [x] Cilium L7: HTTP-level microsegmentation
# [x] Tetragon: eBPF kernel-level control
# [x] Distributed tracing: full request lineage

# PrometheusRule: mTLS coverage
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: zero-trust-compliance
  namespace: monitoring
spec:
  groups:
    - name: zero-trust
      rules:
        - record: zero_trust:mtls_coverage:ratio
          expr: |
            sum(istio_requests_total{connection_security_policy="mutual_tls"}) /
            sum(istio_requests_total)
        
        - alert: ZeroTrustMTLSCoverageBelow100
          expr: zero_trust:mtls_coverage:ratio < 1.0
          for: 5m
          annotations:
            summary: "Not all traffic using mTLS"
          labels:
            severity: warning
```

---

## 📊 สรุป Part 61

| Zero Trust Pillar | Implementation | Tool |
|------------------|----------------|------|
| Identity | mTLS + SPIFFE | Istio/SPIRE |
| Network | Default deny + L7 | NetworkPolicy + Cilium |
| Workload | Signed images | Cosign + Kyverno |
| Data | Encrypt everywhere | cert-manager + KMS |
| Visibility | Full observability | Falco + Jaeger + Loki |

---

## 🔗 ต่อไป
- [Part 62: Incident Response Playbook](./part-62-incident-response.md)

---
*Part 61 | Steps 571-580 | ระดับผู้เชี่ยวชาญ | Educational Use Only*
