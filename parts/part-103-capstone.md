# Part 103: Capstone - Production-Ready Architecture
## Steps 991-1000: สรุปหลักสูตรและ Architecture สำหรับ Production

---

## Step 991: Production Architecture Overview

```
Production-Ready Kubernetes Architecture:

                    Internet
                       │
              ┌────────▼────────┐
              │   CloudFlare    │  DDoS, WAF, CDN
              └────────┬────────┘
                       │
              ┌────────▼────────┐
              │  Load Balancer  │  AWS ALB / GKE Gateway
              │  (cert-manager) │  TLS termination
              └────────┬────────┘
                       │
           ┌───────────▼───────────┐
           │   Istio Ingress GW    │  mTLS, Auth, CORS
           └───────────┬───────────┘
                       │
     ┌─────────────────┼─────────────────┐
     │                 │                 │
┌────▼────┐      ┌─────▼─────┐    ┌─────▼──────┐
│ Payment │      │   Order   │    │  Inventory │
│   API   │      │  Service  │    │  Service   │
│  (v1/v2)│      │           │    │            │
└────┬────┘      └─────┬─────┘    └─────┬──────┘
     │                 │                │
     └─────────────────┼────────────────┘
                       │
              ┌────────▼────────┐
              │   Data Layer    │
              │  Postgres RDS   │
              │  Redis Cache    │
              │  Kafka Events   │
              └─────────────────┘

Cross-Cutting Concerns:
  Observability: Prometheus + Grafana + Jaeger + Loki
  Security: Kyverno + OPA + Trivy + Falco
  GitOps: ArgoCD + Flux + Argo Rollouts
  Networking: Cilium + Istio service mesh
  Secrets: External Secrets + Vault
  Cost: OpenCost + Karpenter
```

---

## Step 992: Complete Application Stack

```yaml
# Namespace with all required labels
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    istio-injection: enabled
    team: platform
    environment: production
    cost-center: CC-1001

---
apiVersion: v1
kind: ConfigMap
metadata:
  name: payment-api-config
  namespace: production
data:
  APP_ENV: production
  LOG_LEVEL: info
  METRICS_PORT: "9090"
  TRACING_ENDPOINT: http://jaeger-collector.monitoring:14268/api/traces

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-api
  namespace: production
  labels:
    app: payment-api
    version: v1
spec:
  replicas: 3
  revisionHistoryLimit: 5
  selector:
    matchLabels:
      app: payment-api
      version: v1
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: payment-api
        version: v1
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9090"
    spec:
      serviceAccountName: payment-api
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 2000
        seccompProfile:
          type: RuntimeDefault
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: payment-api
      containers:
        - name: api
          image: ghcr.io/myorg/payment-api:v1.5.0
          ports:
            - containerPort: 8080
              name: http
            - containerPort: 9090
              name: metrics
          envFrom:
            - configMapRef:
                name: payment-api-config
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: payment-api-db
                  key: url
          resources:
            requests:
              cpu: 200m
              memory: 256Mi
            limits:
              cpu: "1"
              memory: 512Mi
          readinessProbe:
            httpGet:
              path: /ready
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 5
            failureThreshold: 3
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 15
            periodSeconds: 10
          securityContext:
            allowPrivilegeEscalation: false
            capabilities:
              drop: [ALL]
            readOnlyRootFilesystem: true
          volumeMounts:
            - name: tmp
              mountPath: /tmp
      volumes:
        - name: tmp
          emptyDir: {}
```

---

## Step 993: High Availability Setup

```yaml
# PodDisruptionBudget: ensure availability
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: payment-api-pdb
  namespace: production
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: payment-api

---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: payment-api-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: payment-api
  minReplicas: 3
  maxReplicas: 50
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
    - type: Pods
      pods:
        metric:
          name: http_requests_per_second
        target:
          type: AverageValue
          averageValue: 1000
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 30
      policies:
        - type: Pods
          value: 5
          periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300
```

---

## Step 994: Network Security Policies

```yaml
# NetworkPolicy: zero-trust ingress/egress
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: payment-api-netpol
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: payment-api
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: production
          podSelector:
            matchLabels:
              app: order-service
      ports:
        - protocol: TCP
          port: 8080
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: monitoring
      ports:
        - protocol: TCP
          port: 9090
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: production
      ports:
        - protocol: TCP
          port: 5432
        - protocol: TCP
          port: 6379
    - to: []
      ports:
        - protocol: TCP
          port: 443

---
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: payment-api-l7
  namespace: production
spec:
  endpointSelector:
    matchLabels:
      app: payment-api
  ingress:
    - fromEndpoints:
        - matchLabels:
            app: order-service
      toPorts:
        - ports:
            - port: "8080"
              protocol: TCP
          rules:
            http:
              - method: POST
                path: /api/payments
              - method: GET
                path: /api/payments
```

---

## Step 995: Observability Stack

```yaml
# ServiceMonitor: scrape payment-api metrics
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: payment-api
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: payment-api
  namespaceSelector:
    matchNames:
      - production
  endpoints:
    - port: metrics
      interval: 15s
      path: /metrics

---
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: payment-api-slo
  namespace: monitoring
spec:
  groups:
    - name: payment-api.slo
      rules:
        - record: job:payment_requests:rate5m
          expr: sum(rate(http_requests_total{job="payment-api"}[5m])) by (status_code)
        
        - alert: PaymentAPIErrorRateHigh
          expr: |
            sum(rate(http_requests_total{job="payment-api",status_code=~"5.."}[5m]))
            /
            sum(rate(http_requests_total{job="payment-api"}[5m])) > 0.01
          for: 2m
          annotations:
            summary: "Payment API error rate > 1%: {{ $value | humanizePercentage }}"
            runbook: https://runbooks.mycompany.com/payment-api-errors
          labels:
            severity: critical
            team: payments
        
        - alert: PaymentAPILatencyHigh
          expr: |
            histogram_quantile(0.99,
              rate(http_request_duration_seconds_bucket{job="payment-api"}[5m])
            ) > 0.5
          for: 5m
          annotations:
            summary: "Payment API P99 latency > 500ms"
          labels:
            severity: warning
```

---

## Step 996: ArgoCD GitOps Setup

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: payment-api-prod
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: production
  source:
    repoURL: https://github.com/myorg/gitops-fleet
    targetRevision: main
    path: environments/production/payment-api
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
      allowEmpty: false
    syncOptions:
      - CreateNamespace=false
      - Validate=true
      - PrunePropagationPolicy=foreground
      - ApplyOutOfSyncOnly=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
  revisionHistoryLimit: 10
  ignoreDifferences:
    - group: apps
      kind: Deployment
      jsonPointers:
        - /spec/replicas

---
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: production
  namespace: argocd
spec:
  description: Production workloads
  sourceRepos:
    - https://github.com/myorg/*
    - https://charts.mycompany.com
  destinations:
    - namespace: production
      server: https://kubernetes.default.svc
    - namespace: monitoring
      server: https://kubernetes.default.svc
  clusterResourceWhitelist:
    - group: ""
      kind: Namespace
  roles:
    - name: developers
      policies:
        - p, proj:production:developers, applications, get, production/*, allow
        - p, proj:production:developers, applications, sync, production/*, allow
```

---

## Step 997: Kyverno Security Policies

```yaml
# Kyverno: production security baseline
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: production-baseline
spec:
  validationFailureAction: Enforce
  background: true
  rules:
    - name: no-latest-image
      match:
        any:
          - resources:
              kinds: [Pod]
              namespaces: [production]
      validate:
        message: "Image tag ':latest' not allowed in production"
        foreach:
          - list: "request.object.spec.containers"
            deny:
              conditions:
                any:
                  - key: "{{ element.image }}"
                    operator: Contains
                    value: ":latest"

    - name: require-non-root
      match:
        any:
          - resources:
              kinds: [Pod]
              namespaces: [production]
      validate:
        message: "Containers must not run as root"
        pattern:
          spec:
            securityContext:
              runAsNonRoot: true

    - name: drop-capabilities
      match:
        any:
          - resources:
              kinds: [Pod]
              namespaces: [production]
      validate:
        message: "All capabilities must be dropped"
        foreach:
          - list: "request.object.spec.containers"
            pattern:
              securityContext:
                capabilities:
                  drop:
                    - ALL

    - name: readonly-rootfs
      match:
        any:
          - resources:
              kinds: [Pod]
              namespaces: [production]
      validate:
        message: "Containers must use read-only root filesystem"
        foreach:
          - list: "request.object.spec.containers"
            pattern:
              securityContext:
                readOnlyRootFilesystem: true
```

---

## Step 998: Disaster Recovery Setup

```yaml
# Velero: backup schedule
apiVersion: velero.io/v1
kind: Schedule
metadata:
  name: production-daily-backup
  namespace: velero
spec:
  schedule: "0 2 * * *"
  template:
    includedNamespaces:
      - production
      - monitoring
      - argocd
    excludedResources:
      - events
    includeClusterResources: true
    storageLocation: aws-s3-primary
    volumeSnapshotLocations:
      - aws-us-east-1
    ttl: 720h
  useOwnerReferencesInBackup: false

---
apiVersion: batch/v1
kind: CronJob
metadata:
  name: backup-restore-test
  namespace: velero
spec:
  schedule: "0 4 * * 0"
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: restore-test
              image: velero/velero:v1.12.0
              command:
                - /bin/sh
                - -c
                - |
                  BACKUP=$(velero backup get --output json | jq -r '.items[0].metadata.name')
                  echo "Testing restore of backup: $BACKUP"
                  velero restore create test-restore-$(date +%Y%m%d) \
                    --from-backup $BACKUP \
                    --namespace-mappings production:restore-test \
                    --wait
                  echo "Restore test completed successfully"
```

---

## Step 999: Production Checklist

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: production-readiness-checklist
  namespace: production
data:
  checklist.md: |
    # Production Readiness Checklist
    
    ## Security
    - [ ] All container images scanned (Trivy)
    - [ ] No latest image tags in production
    - [ ] Containers run as non-root
    - [ ] Read-only root filesystem
    - [ ] All capabilities dropped
    - [ ] Network policies enforce zero-trust
    - [ ] Secrets from Vault/ESO (not hardcoded)
    - [ ] mTLS between all services (Istio)
    - [ ] RBAC: least-privilege service accounts
    - [ ] Pod Security Standards: restricted
    
    ## Reliability
    - [ ] replicas >= 3 for critical services
    - [ ] PodDisruptionBudget defined
    - [ ] TopologySpreadConstraints set
    - [ ] HPA with min/max bounds
    - [ ] Resource requests AND limits set
    - [ ] Readiness + liveness probes configured
    - [ ] RollingUpdate with maxUnavailable: 0
    - [ ] Circuit breakers (Istio DestinationRule)
    - [ ] Retries with backoff configured
    - [ ] Timeout budgets set
    
    ## Observability
    - [ ] Prometheus metrics exposed
    - [ ] ServiceMonitor created
    - [ ] PrometheusRule: SLO alerts
    - [ ] Distributed tracing enabled (Jaeger)
    - [ ] Centralized logging (Loki/EFK)
    - [ ] Grafana dashboard published
    - [ ] Runbooks linked in alerts
    - [ ] PagerDuty/OpsGenie integration
    
    ## GitOps
    - [ ] All resources in Git (no manual kubectl)
    - [ ] ArgoCD selfHeal: true
    - [ ] Sealed Secrets or ESO for secret management
    - [ ] Separate manifests per environment
    - [ ] PR review required for production changes
    - [ ] Automated tests in CI pipeline
    - [ ] Image tag pinned (not latest)
    
    ## Cost
    - [ ] Namespace ResourceQuota set
    - [ ] LimitRange with sensible defaults
    - [ ] Cost labels: team, cost-center
    - [ ] Spot instances for non-critical
    - [ ] VPA recommendations reviewed
    - [ ] Unused PVCs cleaned up
    - [ ] OpenCost/Kubecost dashboard live
    
    ## Backup
    - [ ] Velero schedule configured
    - [ ] S3 cross-region replication enabled
    - [ ] Weekly restore test passing
    - [ ] RTO/RPO documented
    - [ ] Database point-in-time recovery enabled
```

---

## Step 1000: หลักสูตรสำเร็จ - Final Summary

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: course-completion-summary
  namespace: kube-system
  annotations:
    course: สอนเขียนและพัฒนาโปรแกรมและเว็บแอพพลิเคชันด้วย YAML/JSON
    total-steps: "1000"
data:
  summary.md: |
    # หลักสูตร YAML/JSON สำหรับ Kubernetes - สรุปทั้งหมด 1000 Steps
    
    ## Part 1-10: พื้นฐาน YAML และ JSON (Steps 1-100)
    - YAML syntax, indentation, data types
    - JSON format, validation, jq queries
    - Schema validation (JSON Schema, OpenAPI)
    - Helm templates, values, functions
    - Kustomize overlays, patches, generators
    
    ## Part 11-30: Kubernetes Core (Steps 101-300)
    - Pod, Deployment, ReplicaSet
    - Service (ClusterIP, NodePort, LoadBalancer)
    - ConfigMap, Secret management
    - Namespace, RBAC, ServiceAccount
    - StatefulSet, DaemonSet, Job, CronJob
    - PersistentVolume, StorageClass
    
    ## Part 31-60: Advanced Kubernetes (Steps 301-600)
    - Ingress, Gateway API
    - HPA, VPA, KEDA scaling
    - Node affinity, taints/tolerations
    - ResourceQuota, LimitRange
    - Admission Controllers, Webhooks
    - Operators and Custom Resources
    
    ## Part 61-80: CI/CD and DevOps (Steps 601-800)
    - ArgoCD, Flux GitOps
    - Tekton pipelines
    - GitHub Actions integration
    - Helm chart development
    
    ## Part 81-103: Production Mastery (Steps 801-1000)
    - Multi-cluster management
    - Platform Engineering (Backstage, Crossplane)
    - WASM and Edge computing
    - eBPF and Cilium networking
    - Storage and CSI drivers
    - Istio service mesh
    - Testing and validation
    - Cloud providers (EKS, GKE, AKS)
    - GitOps advanced patterns
    - FinOps and cost optimization
    - Production-ready architecture
    
    ## เนื้อหาด้าน Security Disclaimer:
    เนื้อหาด้าน security ในหลักสูตรนี้มีไว้เพื่อการศึกษาและการทดสอบระบบ
    ที่ได้รับอนุญาตเท่านั้น การนำไปใช้โจมตีระบบที่ไม่ได้รับอนุญาตเป็นสิ่งผิดกฎหมาย
    
    ## ขอบคุณ
    ขอบคุณทุกคนที่ร่วมเรียนหลักสูตรนี้ครบ 1000 Steps!
    คุณพร้อมสำหรับการทำงานกับ Kubernetes ในระดับ Production แล้ว

  production-grade-yaml-principles: |
    1. Declarative: ระบุ desired state ไม่ใช่ imperative steps
    2. Versioned: ทุกอย่างใน Git ไม่มี manual changes
    3. Validated: schema validation ก่อน apply เสมอ
    4. Least Privilege: RBAC minimal, drop ALL capabilities
    5. Immutable: image tags pinned, no :latest
    6. Observable: metrics + traces + logs ทุก service
    7. Resilient: PDB + HPA + readiness probes
    8. Cost-aware: requests/limits + VPA recommendations
    9. Tested: CI validation + chaos testing
    10. Documented: runbooks linked in alerts
```

---

## 🎓 ยินดีด้วย! จบหลักสูตร 1000 Steps

| Part | หัวข้อ | Steps |
|------|--------|-------|
| 1-10 | พื้นฐาน YAML/JSON | 1-100 |
| 11-30 | Kubernetes Core | 101-300 |
| 31-60 | Advanced K8s | 301-600 |
| 61-80 | CI/CD & DevOps | 601-800 |
| 81-92 | Production Ops | 801-920 |
| 93 | Multi-Cluster | 891-900 |
| 94 | Platform Engineering | 901-910 |
| 95 | WASM & Edge | 911-920 |
| 96 | eBPF & Cilium | 921-930 |
| 97 | Storage & CSI | 931-940 |
| 98 | Istio Service Mesh | 941-950 |
| 99 | Testing & Validation | 951-960 |
| 100 | Cloud Providers | 961-970 |
| 101 | GitOps Advanced | 971-980 |
| 102 | FinOps | 981-990 |
| **103** | **Capstone** | **991-1000** |

**🏆 Total: 1000 Steps | 103 Parts | Production-Ready YAML/JSON Mastery**

---
*Part 103 | Steps 991-1000 | Capstone - Production-Ready Architecture*
*หลักสูตร: สอนเขียนและพัฒนาโปรแกรมและเว็บแอพพลิเคชันด้วย YAML/JSON - ครบ 1000 Steps*
