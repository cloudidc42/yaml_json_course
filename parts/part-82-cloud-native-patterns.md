# Part 82: Cloud-Native Patterns
## Steps 781-790: Sidecar, Ambassador, Circuit Breaker, Saga

---

## Step 781: Cloud-Native Design Patterns Overview

```
Cloud-Native Patterns for Kubernetes:

Structural Patterns (Pod level):
  Sidecar: helper container alongside main
    - Logging agent, metrics exporter, proxy
  Ambassador: proxy for external services
    - twemproxy for Redis, pgbouncer for Postgres
  Adapter: normalize output format
    - Legacy app with non-standard logs -> JSON

Behavioral Patterns (Service level):
  Circuit Breaker: fail fast to prevent cascade
    - Istio outlier detection, Resilience4j
  Retry: automatic retry with backoff
    - Istio retries, k8s restartPolicy
  Bulkhead: isolate failures
    - Connection pools, thread pools, namespaces
  Timeout: limit wait time
    - Istio VirtualService timeout, HTTP client timeouts

Data Patterns:
  Saga: distributed transactions
    - Orchestration (Conductor) vs Choreography (events)
  CQRS: command vs query separation
    - Write to Postgres, read from Elasticsearch
  Event Sourcing: store events, not state
    - Kafka as event log, CQRS projections

Deployment Patterns:
  Blue-Green: switch traffic atomically
    - Two identical deployments, flip Service selector
  Canary: gradual traffic shift
    - Istio weights, Flagger, Argo Rollouts
  Feature Flags: runtime feature control
    - OpenFeature, LaunchDarkly, Unleash
```

---

## Step 782: Sidecar Pattern

```yaml
# Sidecar: fluentbit log collector
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-with-logging
  namespace: production
spec:
  template:
    spec:
      containers:
        # Main application
        - name: app
          image: myapp:1.0
          ports:
            - containerPort: 8080
          volumeMounts:
            - name: log-volume
              mountPath: /var/log/app
        
        # Sidecar: collect and forward logs
        - name: fluent-bit
          image: fluent/fluent-bit:2.2
          volumeMounts:
            - name: log-volume
              mountPath: /var/log/app
              readOnly: true
            - name: fluent-bit-config
              mountPath: /fluent-bit/etc
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
            limits:
              cpu: 200m
              memory: 128Mi
      
      volumes:
        - name: log-volume
          emptyDir: {}
        - name: fluent-bit-config
          configMap:
            name: fluent-bit-sidecar-config

---
# Sidecar: Envoy as transparent proxy
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-with-proxy
spec:
  template:
    spec:
      containers:
        - name: app
          image: myapp:1.0
          ports:
            - containerPort: 8080
        
        - name: envoy
          image: envoyproxy/envoy:v1.28
          ports:
            - containerPort: 9901  # Admin
            - containerPort: 10000 # Proxy
          volumeMounts:
            - name: envoy-config
              mountPath: /etc/envoy
          command: [envoy, -c, /etc/envoy/envoy.yaml]
      
      volumes:
        - name: envoy-config
          configMap:
            name: envoy-config
```

---

## Step 783: Ambassador Pattern

```yaml
# Ambassador: proxy that handles retries, circuit breaking
# for calls TO external services

apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-service
  namespace: production
spec:
  template:
    spec:
      containers:
        - name: app
          image: payment-service:1.0
          env:
            # App calls localhost instead of external API
            - name: STRIPE_API_URL
              value: http://localhost:8888/stripe
          ports:
            - containerPort: 8080
        
        # Ambassador: handles external Stripe API calls
        - name: stripe-ambassador
          image: nginx:alpine
          ports:
            - containerPort: 8888
          volumeMounts:
            - name: nginx-config
              mountPath: /etc/nginx/nginx.conf
              subPath: nginx.conf
      
      volumes:
        - name: nginx-config
          configMap:
            name: stripe-ambassador-config

---
apiVersion: v1
kind: ConfigMap
metadata:
  name: stripe-ambassador-config
  namespace: production
data:
  nginx.conf: |
    events {}
    http {
      upstream stripe {
        server api.stripe.com:443;
        keepalive 16;
      }
      server {
        listen 8888;
        location /stripe/ {
          proxy_pass https://stripe/;
          proxy_ssl_server_name on;
          proxy_ssl_name api.stripe.com;
          proxy_next_upstream error timeout http_500 http_502;
          proxy_next_upstream_tries 3;
          proxy_connect_timeout 5s;
          proxy_read_timeout 30s;
        }
      }
    }
```

---

## Step 784: Circuit Breaker with Istio

```yaml
# Circuit Breaker via Istio DestinationRule
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: payment-circuit-breaker
  namespace: production
spec:
  host: payment-service
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        http1MaxPendingRequests: 100
        maxRequestsPerConnection: 1
        maxRetries: 3
    
    # Circuit breaker: outlier detection
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 30s
      baseEjectionTime: 30s
      maxEjectionPercent: 50
      minHealthPercent: 30
      splitExternalLocalOriginErrors: true

---
# Retry Policy with exponential backoff
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: payment-retry
  namespace: production
spec:
  hosts:
    - payment-service
  http:
    - route:
        - destination:
            host: payment-service
      timeout: 10s
      retries:
        attempts: 3
        perTryTimeout: 3s
        retryOn: "5xx,reset,connect-failure,retriable-4xx"
        retryRemoteLocalities: true
```

---

## Step 785: Blue-Green Deployment

```yaml
# Blue deployment (current)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-blue
  namespace: production
  labels:
    app: myapp
    version: blue
spec:
  replicas: 5
  selector:
    matchLabels:
      app: myapp
      version: blue
  template:
    metadata:
      labels:
        app: myapp
        version: blue
    spec:
      containers:
        - name: app
          image: myapp:1.0.0

---
# Green deployment (new version)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-green
  namespace: production
  labels:
    app: myapp
    version: green
spec:
  replicas: 5
  selector:
    matchLabels:
      app: myapp
      version: green
  template:
    metadata:
      labels:
        app: myapp
        version: green
    spec:
      containers:
        - name: app
          image: myapp:2.0.0

---
# Service points to BLUE initially
apiVersion: v1
kind: Service
metadata:
  name: myapp
  namespace: production
spec:
  selector:
    app: myapp
    version: blue  # Change to "green" to switch
  ports:
    - port: 80

# Switch: kubectl patch service myapp -n production \
#   -p '{"spec":{"selector":{"version":"green"}}}'
```

---

## Step 786: Canary Deployment with Argo Rollouts

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: myapp-rollout
  namespace: production
spec:
  replicas: 10
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: app
          image: myapp:1.0
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
  
  strategy:
    canary:
      analysis:
        templates:
          - templateName: success-rate
        args:
          - name: service-name
            value: myapp-canary
      
      steps:
        - setWeight: 5
        - pause: {duration: 5m}
        - setWeight: 20
        - pause: {duration: 5m}
        - setWeight: 50
        - pause: {}    # Manual approval
        - setWeight: 100
      
      canaryService: myapp-canary
      stableService: myapp-stable
      trafficRouting:
        istio:
          virtualService:
            name: myapp
            routes:
              - primary

---
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: success-rate
  namespace: production
spec:
  args:
    - name: service-name
  metrics:
    - name: success-rate
      interval: 2m
      successCondition: result[0] >= 0.95
      failureLimit: 3
      provider:
        prometheus:
          address: http://prometheus.monitoring.svc:9090
          query: |
            sum(rate(http_requests_total{service="{{args.service-name}}",status_code!~"5.."}[2m]))
            /
            sum(rate(http_requests_total{service="{{args.service-name}}"}[2m]))
```

---

## Step 787: Saga Pattern

```yaml
# Saga Orchestration with Argo Workflows
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  name: order-saga
  namespace: orders
spec:
  entrypoint: order-flow
  arguments:
    parameters:
      - name: order-id
      - name: customer-id
      - name: amount

  templates:
    - name: order-flow
      dag:
        tasks:
          - name: reserve-inventory
            template: call-service
            arguments:
              parameters:
                - name: service
                  value: inventory-service
                - name: action
                  value: reserve
          
          - name: charge-payment
            template: call-service
            dependencies: [reserve-inventory]
            arguments:
              parameters:
                - name: service
                  value: payment-service
                - name: action
                  value: charge
          
          - name: confirm-order
            template: call-service
            dependencies: [charge-payment]
            arguments:
              parameters:
                - name: service
                  value: order-service
                - name: action
                  value: confirm
          
          # Compensating transaction on failure
          - name: release-inventory-on-failure
            template: call-service
            dependencies: [charge-payment]
            when: "{{tasks.charge-payment.status}} == Failed"
            arguments:
              parameters:
                - name: service
                  value: inventory-service
                - name: action
                  value: release
    
    - name: call-service
      inputs:
        parameters:
          - name: service
          - name: action
      container:
        image: curlimages/curl:latest
        command:
          - curl
          - -f
          - http://{{inputs.parameters.service}}/{{inputs.parameters.action}}
```

---

## Step 788: Feature Flags with OpenFeature

```yaml
apiVersion: core.openfeature.dev/v1alpha1
kind: FeatureFlagConfiguration
metadata:
  name: app-feature-flags
  namespace: production
spec:
  featureFlagSpec:
    flags:
      new-checkout-flow:
        state: ENABLED
        variants:
          "on": true
          "off": false
        defaultVariant: "off"
        targeting:
          if:
            - in:
                - {var: userId}
                - [user1, user2, user3]
            - "on"
            - "off"
      
      dark-mode:
        state: ENABLED
        variants:
          enabled: true
          disabled: false
        defaultVariant: disabled
      
      max-items-per-page:
        state: ENABLED
        variants:
          large: 100
          medium: 50
          small: 20
        defaultVariant: medium

---
apiVersion: v1
kind: Pod
metadata:
  name: myapp
  annotations:
    openfeature.dev/enabled: "true"
    openfeature.dev/flagsourceconfiguration: "app-feature-flags"
spec:
  containers:
    - name: app
      image: myapp:1.0
      env:
        - name: OPENFEATURE_PROVIDER
          value: flagd
        - name: FLAGD_HOST
          value: localhost
        - name: FLAGD_PORT
          value: "8013"
```

---

## Step 789: Event-Driven Architecture with Kafka

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: production-cluster
  namespace: kafka
spec:
  kafka:
    version: 3.6.0
    replicas: 3
    listeners:
      - name: plain
        port: 9092
        type: internal
        tls: false
      - name: tls
        port: 9093
        type: internal
        tls: true
        authentication:
          type: tls
    config:
      offsets.topic.replication.factor: 3
      transaction.state.log.replication.factor: 3
      min.insync.replicas: 2
      default.replication.factor: 3
    storage:
      type: jbod
      volumes:
        - id: 0
          type: persistent-claim
          size: 100Gi
          class: gp3
          deleteClaim: false
  
  zookeeper:
    replicas: 3
    storage:
      type: persistent-claim
      size: 10Gi
      class: gp3

---
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaTopic
metadata:
  name: order-events
  namespace: kafka
  labels:
    strimzi.io/cluster: production-cluster
spec:
  partitions: 10
  replicas: 3
  config:
    retention.ms: 604800000
    cleanup.policy: delete

---
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: order-processor-keda
  namespace: production
spec:
  scaleTargetRef:
    name: order-processor
  minReplicaCount: 1
  maxReplicaCount: 20
  triggers:
    - type: kafka
      metadata:
        bootstrapServers: production-cluster-kafka-bootstrap.kafka.svc:9092
        consumerGroup: order-processor-group
        topic: order-events
        lagThreshold: "100"
```

---

## Step 790: Workshop - Cloud-Native Patterns Summary

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: cloud-native-patterns-guide
  namespace: production
data:
  patterns.yaml: |
    when_to_use:
      sidecar:
        - Logging/metrics collection
        - TLS termination
        - Configuration reload (Vault agent)
        - Service mesh proxy injection
      
      ambassador:
        - Connection pooling to external databases
        - Protocol translation (gRPC -> HTTP/1.1)
        - Retry logic for legacy external services
      
      circuit_breaker:
        - Calls to slow/unreliable downstream services
        - Prevent cascade failures
        - Give time for service recovery
      
      blue_green:
        - Zero-downtime requirement
        - Instant rollback capability
        - Extra cost: 2x resources during deploy
      
      canary:
        - Risk reduction for new features
        - Gradual performance validation
        - A/B testing
        - Requires metrics-based automation
      
      saga:
        - Distributed transactions across microservices
        - Cannot use database transactions
        - Always design compensating transactions
      
      event_driven:
        - Decouple producers from consumers
        - Handle traffic spikes (buffer in Kafka)
        - Async processing of non-critical tasks

---
# Flagger automated canary
apiVersion: flagger.app/v1beta1
kind: Canary
metadata:
  name: myapp
  namespace: production
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  progressDeadlineSeconds: 600
  service:
    port: 80
  analysis:
    interval: 1m
    threshold: 5
    maxWeight: 50
    stepWeight: 10
    metrics:
      - name: request-success-rate
        min: 99
        interval: 1m
      - name: request-duration
        max: 500
        interval: 1m
```

---

## 📊 สรุป Part 82

| Pattern | Problem Solved | Kubernetes Tool |
|---------|---------------|------------------|
| Sidecar | Cross-cutting concerns | Pod multi-container |
| Ambassador | External service proxy | Pod sidecar + nginx |
| Circuit Breaker | Cascade failures | Istio DestinationRule |
| Blue-Green | Zero-downtime deploy | Service selector switch |
| Canary | Safe gradual rollout | Argo Rollouts, Flagger |
| Saga | Distributed transactions | Argo Workflows |
| Feature Flags | Runtime feature control | OpenFeature + flagd |
| Event-Driven | Async decoupling | Kafka + KEDA |

---

## 🔗 ต่อไป
- [Part 83: Developer Experience and Tooling](./part-83-developer-experience.md)

---
*Part 82 | Steps 781-790 | Cloud-Native Patterns | Educational Content*
