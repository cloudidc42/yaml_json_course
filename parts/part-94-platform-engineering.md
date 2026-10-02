# Part 94: Platform Engineering and Developer Portals
## Steps 901-910: Internal Developer Platform, Backstage, Self-Service, Golden Paths

---

## Step 901: Platform Engineering Overview

```
Platform Engineering:

Definition:
  Building Internal Developer Platforms (IDP) so
  application teams can self-serve infrastructure
  without deep Kubernetes knowledge.

Goals:
  - Reduce cognitive load for developers
  - Standardize golden paths (opinionated)
  - Self-service: provision env in minutes, not days
  - Guardrails: enforce policies via platform
  - Visibility: one portal for all services

IDP Components:
  Portal: Backstage (catalog, scaffolding, docs)
  Templates: Helm Charts, Kustomize, Crossplane
  Pipelines: GitHub Actions, Tekton
  Secrets: External Secrets, Vault
  Networking: Ingress, Service Mesh (Istio)
  Monitoring: auto-provisioned dashboards + alerts
  GitOps: ArgoCD auto-creates environment

Platform Team Responsibilities:
  - Kubernetes clusters (Cluster API)
  - Shared services (cert-manager, Ingress, Velero)
  - Developer portal (Backstage)
  - Golden path templates (Helm charts)
  - Policy enforcement (Kyverno, OPA)
  - Observability stack (Prometheus, Grafana, Loki)

Developer Self-Service:
  1. Open portal (Backstage)
  2. Choose service template
  3. Fill in: team, tier, repo, language, resources
  4. Portal creates: GitHub repo + GitOps config + Jira project
  5. ArgoCD auto-deploys to dev in 5 min
  6. Promote: 1-click staging, 1-click production
```

---

## Step 902: Backstage Setup

```yaml
# Backstage: developer portal
# helm install backstage backstage/backstage -n backstage

# Backstage app-config.yaml
app:
  title: MyCompany Developer Portal
  baseUrl: https://portal.mycompany.com

backend:
  baseUrl: https://portal.mycompany.com
  cors:
    origin: https://portal.mycompany.com
  database:
    client: pg
    connection:
      host: postgres.backstage.svc
      port: 5432
      user: backstage
      database: backstage

integrations:
  github:
    - host: github.com
      token: ${GITHUB_TOKEN}

catalog:
  rules:
    - allow: [Component, API, System, Domain, Resource, Location]
  locations:
    - type: url
      target: https://github.com/myorg/backstage-catalog/blob/main/catalog-info.yaml
    - type: github-discovery
      target: https://github.com/myorg

techdocs:
  builder: external
  generator:
    runIn: docker
  publisher:
    type: awsS3
    awsS3:
      bucketName: mycompany-techdocs
      region: us-east-1

auth:
  providers:
    github:
      development:
        clientId: ${GITHUB_CLIENT_ID}
        clientSecret: ${GITHUB_CLIENT_SECRET}

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backstage
  namespace: backstage
spec:
  replicas: 2
  template:
    spec:
      containers:
        - name: backstage
          image: ghcr.io/myorg/backstage:1.0
          env:
            - name: GITHUB_TOKEN
              valueFrom:
                secretKeyRef:
                  name: backstage-secrets
                  key: github-token
          ports:
            - containerPort: 7007
          resources:
            requests:
              cpu: 500m
              memory: 512Mi
            limits:
              cpu: 2000m
              memory: 2Gi
```

---

## Step 903: Backstage Software Catalog

```yaml
apiVersion: backstage.io/v1alpha1
kind: Component
metadata:
  name: payment-service
  title: Payment Service
  description: Processes payment transactions
  annotations:
    github.com/project-slug: myorg/payment-service
    argocd/app-name: payment-service-production
    pagerduty.com/service-id: P1234567
    grafana/dashboard-selector: "app=payment-service"
    backstage.io/techdocs-ref: dir:.
  links:
    - url: https://payment-service.mycompany.com
      title: Production
    - url: https://grafana.mycompany.com/d/payment-service
      title: Grafana Dashboard
  tags:
    - payments
    - critical
    - tier-1
spec:
  type: service
  lifecycle: production
  owner: group:payments-team
  system: payment-system
  dependsOn:
    - component:postgres-db
    - component:redis-cache
  providesApis:
    - payment-api
  consumesApis:
    - fraud-detection-api

---
apiVersion: backstage.io/v1alpha1
kind: API
metadata:
  name: payment-api
  description: Payment processing API
spec:
  type: openapi
  lifecycle: production
  owner: group:payments-team
  definition:
    $text: https://api.mycompany.com/openapi.json
```

---

## Step 904: Backstage Software Templates

```yaml
apiVersion: scaffolder.backstage.io/v1beta3
kind: Template
metadata:
  name: microservice-template
  title: New Microservice
  description: Create a new microservice with all best practices
  tags: [microservice, python, kubernetes]
spec:
  owner: platform-team
  type: service
  
  parameters:
    - title: Service Information
      required: [name, description, owner, tier]
      properties:
        name:
          title: Service Name
          type: string
          pattern: "^[a-z][a-z0-9-]{2,30}$"
        description:
          title: Description
          type: string
        owner:
          title: Owning Team
          type: string
          ui:field: OwnerPicker
        tier:
          title: Service Tier
          type: string
          enum: ["1", "2", "3"]
    
    - title: Infrastructure
      properties:
        cpu:
          title: CPU Request
          type: string
          default: "200m"
          enum: ["100m", "200m", "500m", "1000m"]
        memory:
          title: Memory Request
          type: string
          default: "256Mi"
          enum: ["128Mi", "256Mi", "512Mi", "1Gi"]
        autoscaling:
          title: Enable Autoscaling
          type: boolean
          default: true
  
  steps:
    - id: fetch
      name: Fetch Template
      action: fetch:template
      input:
        url: ./skeleton
        values:
          name: ${{ parameters.name }}
          owner: ${{ parameters.owner }}
    
    - id: publish
      name: Publish to GitHub
      action: publish:github
      input:
        allowedHosts: [github.com]
        description: ${{ parameters.description }}
        repoUrl: github.com?owner=myorg&repo=${{ parameters.name }}
    
    - id: register
      name: Register in Catalog
      action: catalog:register
      input:
        repoContentsUrl: ${{ steps.publish.output.repoContentsUrl }}
        catalogInfoPath: /catalog-info.yaml
    
    - id: create-argocd-app
      name: Create ArgoCD Application
      action: argocd:create-resources
      input:
        appName: ${{ parameters.name }}-dev
        projectName: ${{ parameters.owner }}
        namespace: development
        repoUrl: ${{ steps.publish.output.remoteUrl }}
  
  output:
    links:
      - title: Repository
        url: ${{ steps.publish.output.remoteUrl }}
      - title: Open in catalog
        icon: catalog
        entityRef: ${{ steps.register.output.entityRef }}
```

---

## Step 905: Crossplane for Infrastructure

```yaml
# Crossplane: Kubernetes-native infrastructure provisioning
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata:
  name: xpostgresinstances.database.mycompany.com
spec:
  group: database.mycompany.com
  names:
    kind: XPostgresInstance
    plural: xpostgresinstances
  claimNames:
    kind: PostgresInstance
    plural: postgresinstances
  connectionSecretKeys:
    - username
    - password
    - endpoint
    - port
  versions:
    - name: v1alpha1
      served: true
      referenceable: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                storageGB:
                  type: integer
                  minimum: 20
                  maximum: 1000
                instanceClass:
                  type: string
                  enum: [small, medium, large]
                  default: small

---
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: xpostgresinstances-aws
spec:
  compositeTypeRef:
    apiVersion: database.mycompany.com/v1alpha1
    kind: XPostgresInstance
  resources:
    - name: rds-instance
      base:
        apiVersion: rds.aws.upbound.io/v1beta1
        kind: Instance
        spec:
          forProvider:
            region: us-east-1
            engine: postgres
            engineVersion: "16"
            skipFinalSnapshot: true
            publiclyAccessible: false
      patches:
        - type: FromCompositeFieldPath
          fromFieldPath: spec.storageGB
          toFieldPath: spec.forProvider.allocatedStorage

---
apiVersion: database.mycompany.com/v1alpha1
kind: PostgresInstance
metadata:
  name: payment-db
  namespace: production
spec:
  storageGB: 100
  instanceClass: medium
  writeConnectionSecretToRef:
    name: payment-db-credentials
```

---

## Step 906: Self-Service Namespace Automation

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: namespace-setup
spec:
  rules:
    - name: add-default-resources
      match:
        any:
          - resources:
              kinds: [Namespace]
              selector:
                matchLabels:
                  managed-by: platform
      generate:
        - apiVersion: v1
          kind: ResourceQuota
          name: default-quota
          namespace: "{{request.object.metadata.name}}"
          data:
            spec:
              hard:
                requests.cpu: "10"
                requests.memory: 20Gi
                limits.cpu: "20"
                limits.memory: 40Gi
                pods: "100"
        
        - apiVersion: v1
          kind: LimitRange
          name: default-limits
          namespace: "{{request.object.metadata.name}}"
          data:
            spec:
              limits:
                - type: Container
                  default:
                    cpu: 200m
                    memory: 256Mi
                  defaultRequest:
                    cpu: 100m
                    memory: 128Mi

---
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: namespace-network-policy
spec:
  rules:
    - name: default-deny
      match:
        any:
          - resources:
              kinds: [Namespace]
              selector:
                matchLabels:
                  managed-by: platform
      generate:
        - apiVersion: networking.k8s.io/v1
          kind: NetworkPolicy
          name: default-deny-all
          namespace: "{{request.object.metadata.name}}"
          data:
            spec:
              podSelector: {}
              policyTypes: [Ingress, Egress]
```

---

## Step 907: Golden Path Helm Chart

```yaml
# values.yaml: opinionated defaults for all microservices
replicaCount: 2

image:
  repository: ""
  pullPolicy: IfNotPresent
  tag: ""

service:
  type: ClusterIP
  port: 80
  targetPort: 8080

ingress:
  enabled: true
  className: nginx
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"

resources:
  requests:
    cpu: 200m
    memory: 256Mi
  limits:
    cpu: 1000m
    memory: 512Mi

autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 20
  targetCPUUtilizationPercentage: 70

podDisruptionBudget:
  enabled: true
  maxUnavailable: 1

serviceAccount:
  create: true

serviceMonitor:
  enabled: true
  interval: 30s
  path: /metrics

livenessProbe:
  httpGet:
    path: /health
    port: http

readinessProbe:
  httpGet:
    path: /ready
    port: http

securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  readOnlyRootFilesystem: true
  allowPrivilegeEscalation: false
  capabilities:
    drop: [ALL]
```

---

## Step 908: Self-Service Promote to Staging

```yaml
apiVersion: scaffolder.backstage.io/v1beta3
kind: Template
metadata:
  name: promote-to-staging
  title: Promote to Staging
  tags: [operations]
spec:
  parameters:
    - title: Select Service
      properties:
        service:
          title: Service
          type: string
          ui:field: EntityPicker
          ui:options:
            catalogFilter:
              kind: Component
              spec.lifecycle: production
        imageTag:
          title: Image Tag
          type: string
  
  steps:
    - id: update-image
      name: Update Image Tag
      action: github:create-pull-request
      input:
        repoUrl: github.com?owner=myorg&repo=gitops-repo
        title: Promote ${{ parameters.service }} to staging
        branchName: promote/${{ parameters.service }}-${{ parameters.imageTag }}
        description: |
          Auto-promoted by Developer Portal
          Service: ${{ parameters.service }}
          Tag: ${{ parameters.imageTag }}
```

---

## Step 909: Platform Metrics (DORA)

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: dora-queries
  namespace: monitoring
data:
  deployment-frequency.promql: |
    sum(increase(argocd_app_sync_total{phase="Succeeded"}[1d])) by (name)
  
  lead-time.promql: |
    histogram_quantile(0.50, 
      rate(pipeline_lead_time_seconds_bucket[7d])
    )
  
  change-failure-rate.promql: |
    sum(increase(kubectl_rollout_undo_total[7d]))
    /
    sum(increase(argocd_app_sync_total{phase="Succeeded"}[7d]))

---
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: platform-health
  namespace: monitoring
spec:
  groups:
    - name: platform
      rules:
        - alert: PortalDown
          expr: up{job="backstage"} == 0
          for: 2m
          annotations:
            summary: "Developer portal is down"
          labels:
            severity: critical

        - alert: HighOnboardingTime
          expr: |
            histogram_quantile(0.90,
              rate(platform_service_onboarding_duration_seconds_bucket[7d])
            ) > 3600
          for: 1h
          annotations:
            summary: "P90 service onboarding time > 1h"
          labels:
            severity: warning
```

---

## Step 910: Workshop - Platform Stack Summary

```yaml
platform:
  backstage:
    enabled: true
    replicas: 2
    ingress:
      host: portal.mycompany.com
  
  crossplane:
    enabled: true
    providers:
      - aws
      - gcp
  
  argocd:
    enabled: true
    sso:
      enabled: true
      provider: github
  
  certManager:
    enabled: true
    clusterIssuer:
      email: platform@mycompany.com
  
  ingressNginx:
    enabled: true
    replicas: 3
  
  externalSecrets:
    enabled: true
    vault:
      address: https://vault.mycompany.com
  
  kyverno:
    enabled: true
    policies:
      namespaceSetup: true
      securityBaseline: true
      costGovernance: true
  
  monitoring:
    prometheus:
      enabled: true
      retention: 30d
    grafana:
      enabled: true
    loki:
      enabled: true
    tempo:
      enabled: true
```

---

## 📊 สรุป Part 94

| Component | Tool | Purpose |
|-----------|------|---------|
| Developer Portal | Backstage | Catalog, templates, docs |
| IaC Self-Service | Crossplane | RDS, S3, queues via CRD |
| GitOps | ArgoCD + ApplicationSet | Auto-deploy all services |
| Policy | Kyverno | Auto-setup namespaces |
| Golden Path | Helm Library Charts | Opinionated defaults |
| DORA Metrics | Prometheus | Measure platform impact |

---

## 🔗 ต่อไป
- [Part 95: WebAssembly and Edge Computing](./part-95-wasm-edge.md)

---
*Part 94 | Steps 901-910 | Platform Engineering and Developer Portals | Educational Content*
