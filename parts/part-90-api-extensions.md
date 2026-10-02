# Part 90: Kubernetes API Extensions
## Steps 861-870: CRDs, Webhooks, Aggregation Layer, Controllers

---

## Step 861: Kubernetes API Extension Overview

```
Kubernetes API Extension Methods:

1. Custom Resource Definitions (CRDs):
   - Most common: add new resource types
   - Stored in etcd like built-in resources
   - Served by the main API server
   - Watch with kubectl, use RBAC, events
   - Operators = CRD + controller

2. Aggregation Layer (AA):
   - Separate API server (extension API server)
   - Registered via APIService resource
   - Gets proxied traffic from main API server
   - Use when: streaming APIs, custom storage
   - Examples: metrics-server, kube-aggregator

3. Admission Webhooks:
   - Mutating: modify objects before persistence
   - Validating: reject invalid objects
   - Triggered by: create, update, delete
   - Must return in < 30s (configurable timeout)

4. Custom Schedulers:
   - Extend pod scheduling logic
   - Run alongside default scheduler
   - Pods opt-in via schedulerName field

Architecture Flow:
  kubectl apply -f crd.yaml
  kubectl apply -f myresource.yaml
  -> API server validates vs CRD schema
  -> Mutating webhooks run (if registered)
  -> Validating webhooks run (if registered)
  -> Stored to etcd
  -> Controller receives watch event
  -> Controller reconciles actual state
```

---

## Step 862: Complete CRD with Schema Validation

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: applicationsets.platform.mycompany.com
spec:
  group: platform.mycompany.com
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          required: [spec]
          properties:
            spec:
              type: object
              required: [image, replicas]
              properties:
                image:
                  type: string
                  pattern: "^[a-z0-9/._:-]+$"
                replicas:
                  type: integer
                  minimum: 1
                  maximum: 100
                environment:
                  type: string
                  enum: [development, staging, production]
                  default: development
                resources:
                  type: object
                  properties:
                    cpu:
                      type: string
                      pattern: "^[0-9]+(m|[0-9]*)$"
                    memory:
                      type: string
                      pattern: "^[0-9]+(Mi|Gi)$"
                autoscaling:
                  type: object
                  properties:
                    enabled:
                      type: boolean
                      default: false
                    minReplicas:
                      type: integer
                      minimum: 1
                    maxReplicas:
                      type: integer
                      maximum: 100
            status:
              type: object
              x-kubernetes-preserve-unknown-fields: true
      
      subresources:
        status: {}
        scale:
          specReplicasPath: .spec.replicas
          statusReplicasPath: .status.replicas
      
      additionalPrinterColumns:
        - name: Image
          type: string
          jsonPath: .spec.image
        - name: Replicas
          type: integer
          jsonPath: .spec.replicas
        - name: Environment
          type: string
          jsonPath: .spec.environment
        - name: Age
          type: date
          jsonPath: .metadata.creationTimestamp
  
  scope: Namespaced
  names:
    plural: applicationsets
    singular: applicationset
    kind: ApplicationSet
    shortNames: [appset]
    categories: [platform]
```

---

## Step 863: Custom Resource Example

```yaml
apiVersion: platform.mycompany.com/v1
kind: ApplicationSet
metadata:
  name: payment-service
  namespace: production
  labels:
    team: payments
    cost-center: cc-1234
spec:
  image: ghcr.io/myorg/payment-service:1.5.2
  replicas: 3
  environment: production
  resources:
    cpu: 500m
    memory: 512Mi
  autoscaling:
    enabled: true
    minReplicas: 3
    maxReplicas: 20
    targetCPUUtilizationPercentage: 70

---
# RBAC for custom resources
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: appset-deployer
  namespace: production
rules:
  - apiGroups: [platform.mycompany.com]
    resources: [applicationsets]
    verbs: [get, list, watch, create, update, patch]
  - apiGroups: [platform.mycompany.com]
    resources: [applicationsets/status]
    verbs: [get, update, patch]
  - apiGroups: [platform.mycompany.com]
    resources: [applicationsets/scale]
    verbs: [get, update, patch]
```

---

## Step 864: Controller Reconcile Loop

```go
// Controller reconcile loop (Go pseudo-code)
// Uses controller-runtime library

func (r *ApplicationSetReconciler) Reconcile(
    ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    
    // 1. Fetch the custom resource
    appset := &platformv1.ApplicationSet{}
    if err := r.Get(ctx, req.NamespacedName, appset); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }
    
    // 2. Build desired Deployment
    deploy := &appsv1.Deployment{
        ObjectMeta: metav1.ObjectMeta{
            Name:      appset.Name,
            Namespace: appset.Namespace,
        },
    }
    
    // 3. CreateOrUpdate: idempotent reconcile
    _, err := controllerutil.CreateOrUpdate(ctx, r.Client, deploy, func() error {
        deploy.Spec.Replicas = &appset.Spec.Replicas
        deploy.Spec.Template.Spec.Containers[0].Image = appset.Spec.Image
        return controllerutil.SetControllerReference(appset, deploy, r.Scheme)
    })
    
    // 4. Create HPA if autoscaling enabled
    if appset.Spec.Autoscaling.Enabled {
        hpa := buildHPA(appset)
        _, err = controllerutil.CreateOrUpdate(ctx, r.Client, hpa, func() error {
            return controllerutil.SetControllerReference(appset, hpa, r.Scheme)
        })
    }
    
    // 5. Update status
    appset.Status.Replicas = deploy.Status.ReadyReplicas
    appset.Status.Phase = "Running"
    if err := r.Status().Update(ctx, appset); err != nil {
        return ctrl.Result{}, err
    }
    
    return ctrl.Result{RequeueAfter: 5 * time.Minute}, nil
}
```

---

## Step 865: Mutating Admission Webhook

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingWebhookConfiguration
metadata:
  name: platform-defaults-injector
  annotations:
    cert-manager.io/inject-ca-from: platform-system/platform-webhook-cert
spec:
  webhooks:
    - name: inject-defaults.platform.mycompany.com
      admissionReviewVersions: [v1]
      clientConfig:
        service:
          name: platform-webhook
          namespace: platform-system
          path: /mutate-platform-mycompany-com-v1-applicationset
        caBundle: ""
      rules:
        - apiGroups: [platform.mycompany.com]
          apiVersions: [v1]
          resources: [applicationsets]
          operations: [CREATE, UPDATE]
      sideEffects: None
      failurePolicy: Fail
      timeoutSeconds: 10
      namespaceSelector:
        matchLabels:
          platform-webhooks: enabled
```

---

## Step 866: Validating Admission Webhook

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: platform-validator
  annotations:
    cert-manager.io/inject-ca-from: platform-system/platform-webhook-cert
spec:
  webhooks:
    - name: validate.platform.mycompany.com
      admissionReviewVersions: [v1]
      clientConfig:
        service:
          name: platform-webhook
          namespace: platform-system
          path: /validate-platform-mycompany-com-v1-applicationset
      rules:
        - apiGroups: [platform.mycompany.com]
          apiVersions: [v1]
          resources: [applicationsets]
          operations: [CREATE, UPDATE]
      sideEffects: None
      failurePolicy: Fail

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: platform-webhook
  namespace: platform-system
spec:
  replicas: 2
  template:
    spec:
      containers:
        - name: webhook
          image: ghcr.io/myorg/platform-webhook:1.0
          ports:
            - containerPort: 9443
              name: webhook-server
          volumeMounts:
            - name: cert
              mountPath: /tmp/k8s-webhook-server/serving-certs
              readOnly: true
      volumes:
        - name: cert
          secret:
            secretName: platform-webhook-cert
```

---

## Step 867: Aggregation Layer API Server

```yaml
apiVersion: apiregistration.k8s.io/v1
kind: APIService
metadata:
  name: v1beta1.metrics.k8s.io
spec:
  service:
    name: metrics-server
    namespace: kube-system
    port: 443
  group: metrics.k8s.io
  version: v1beta1
  insecureSkipTLSVerify: false
  caBundle: base64-ca-bundle-here
  groupPriorityMinimum: 100
  versionPriority: 100

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: custom-apiserver
  namespace: platform-system
spec:
  replicas: 2
  template:
    spec:
      serviceAccountName: custom-apiserver
      containers:
        - name: apiserver
          image: ghcr.io/myorg/custom-apiserver:1.0
          args:
            - --etcd-servers=https://etcd.platform-system.svc:2379
            - --tls-cert-file=/certs/tls.crt
            - --tls-private-key-file=/certs/tls.key
          ports:
            - containerPort: 443
          volumeMounts:
            - name: certs
              mountPath: /certs
      volumes:
        - name: certs
          secret:
            secretName: custom-apiserver-cert
```

---

## Step 868: Finalizers and Owner References

```yaml
# Finalizer: prevent deletion until cleanup done
apiVersion: platform.mycompany.com/v1
kind: ApplicationSet
metadata:
  name: payment-service
  namespace: production
  finalizers:
    - platform.mycompany.com/cleanup
    # When kubectl delete is called:
    # 1. DeletionTimestamp is set
    # 2. Controller detects DeletionTimestamp
    # 3. Controller runs cleanup
    # 4. Controller removes finalizer
    # 5. Object is deleted from etcd

---
# Owner Reference auto-GC
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-service
  namespace: production
  ownerReferences:
    - apiVersion: platform.mycompany.com/v1
      kind: ApplicationSet
      name: payment-service
      uid: abc123-def456
      controller: true
      blockOwnerDeletion: true
```

---

## Step 869: Custom Metrics API

```yaml
# Prometheus Adapter: serve custom metrics to HPA
# values.yaml
prometheus:
  url: http://prometheus.monitoring.svc
  port: 9090

rules:
  default: false
  custom:
    - seriesQuery: 'http_requests_total{namespace!="",pod!=""}'
      resources:
        overrides:
          namespace:
            resource: namespace
          pod:
            resource: pod
      name:
        matches: "^(.*)_total"
        as: "${1}_per_second"
      metricsQuery: 'sum(rate(<<.Series>>{<<.LabelMatchers>>}[2m])) by (<<.GroupBy>>)'

---
# HPA using custom metrics
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-custom-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 2
  maxReplicas: 50
  metrics:
    - type: Pods
      pods:
        metric:
          name: http_requests_per_second
        target:
          type: AverageValue
          averageValue: 100
```

---

## Step 870: Workshop - Operator Monitoring

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: operator-alerts
  namespace: monitoring
spec:
  groups:
    - name: operators
      rules:
        - alert: OperatorReconcileErrors
          expr: |
            rate(controller_runtime_reconcile_errors_total[5m]) > 0.1
          for: 5m
          annotations:
            summary: "Controller {{ $labels.controller }} reconcile error rate > 0.1/s"
          labels:
            severity: warning

        - alert: WebhookHighLatency
          expr: |
            histogram_quantile(0.99,
              rate(controller_runtime_webhook_latency_seconds_bucket[5m])
            ) > 1
          for: 5m
          annotations:
            summary: "Webhook {{ $labels.webhook }} P99 latency > 1s"
          labels:
            severity: warning

        - alert: CustomResourcePending
          expr: |
            count(kube_customresource_status_phase{phase="Pending"}) by (kind, namespace) > 5
          for: 15m
          annotations:
            summary: "{{ $labels.kind }} resources pending in {{ $labels.namespace }} for 15m"
          labels:
            severity: warning
```

---

## 📊 สรุป Part 90

| Component | Purpose | Tool |
|-----------|---------|------|
| CRD | New resource types | apiextensions/v1 |
| Controller | Reconcile loop | controller-runtime |
| Mutating webhook | Inject defaults | cert-manager + webhook server |
| Validating webhook | Policy enforcement | OPA / custom webhook |
| Aggregation layer | Custom API server | APIService |
| Finalizers | Safe deletion | controller-runtime |
| Custom metrics | Custom HPA scaling | Prometheus Adapter |

---

## 🔗 ต่อไป
- [Part 91: Kubernetes Security Advanced](./part-91-security-advanced.md)

---
*Part 90 | Steps 861-870 | Kubernetes API Extensions | Educational Content*
