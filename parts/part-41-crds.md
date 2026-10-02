# Part 41: CRDs และ API Extensions
## Steps 391-400: Custom Resources Deep Dive

---

## 📖 บทนำ

Custom Resource Definitions (CRDs) ช่วยขยาย Kubernetes API ด้วย resource types ของเราเอง ทำให้ Kubernetes กลายเป็น platform ที่สามารถ model domain-specific concepts ได้

---

## Step 391: CRD Versioning

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: applications.app.example.com
spec:
  group: app.example.com
  versions:
    - name: v1alpha1
      served: true
      storage: false
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                image:
                  type: string
                replicas:
                  type: integer
    
    - name: v1beta1
      served: true
      storage: false
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                image:
                  type: string
                replicas:
                  type: integer
                resources:
                  type: object
    
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              required: ["image"]
              properties:
                image:
                  type: string
                replicas:
                  type: integer
                  minimum: 1
                  maximum: 100
                  default: 1
                resources:
                  type: object
                config:
                  type: object
                  x-kubernetes-preserve-unknown-fields: true
  
  conversion:
    strategy: Webhook
    webhook:
      conversionReviewVersions: ["v1", "v1beta1"]
      clientConfig:
        service:
          namespace: default
          name: crd-conversion-webhook
          path: /convert
  
  scope: Namespaced
  names:
    plural: applications
    singular: application
    kind: Application
    shortNames: ["app"]
    categories: ["all"]
```

---

## Step 392: Schema Validation Advanced

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: pipelines.ci.example.com
spec:
  group: ci.example.com
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              required: ["stages"]
              properties:
                stages:
                  type: array
                  minItems: 1
                  maxItems: 20
                  items:
                    type: object
                    required: ["name", "steps"]
                    properties:
                      name:
                        type: string
                        pattern: "^[a-z][a-z0-9-]*$"
                      dependsOn:
                        type: array
                        items:
                          type: string
                      steps:
                        type: array
                        items:
                          type: object
                          required: ["name", "image"]
                          properties:
                            name:
                              type: string
                            image:
                              type: string
                            command:
                              type: array
                              items:
                                type: string
                            env:
                              type: array
                              items:
                                type: object
                                required: ["name"]
                                properties:
                                  name:
                                    type: string
                                  value:
                                    type: string
                timeout:
                  type: string
                  pattern: "^[0-9]+(s|m|h)$"
                  default: "1h"
                trigger:
                  type: object
                  properties:
                    type:
                      type: string
                      enum: ["push", "tag", "manual", "schedule"]
                    branch:
                      type: string
```

---

## Step 393: CEL Validation Rules

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: quotas.billing.example.com
spec:
  group: billing.example.com
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                maxCPU:
                  type: string
                maxMemory:
                  type: string
                maxPods:
                  type: integer
                hardLimit:
                  type: boolean
              x-kubernetes-validations:
                - rule: "self.maxCPU.matches('^[0-9]+(m)?$')"
                  message: "maxCPU must be in millicores (e.g. '500m') or cores (e.g. '2')"
                - rule: "self.maxPods > 0"
                  message: "maxPods must be positive"
                - rule: "!self.hardLimit || (has(self.maxCPU) && has(self.maxMemory))"
                  message: "hardLimit requires maxCPU and maxMemory"
  scope: Namespaced
  names:
    plural: quotas
    singular: quota
    kind: Quota
```

---

## Step 394: Aggregated API Server

```yaml
apiVersion: apiregistration.k8s.io/v1
kind: APIService
metadata:
  name: v1alpha1.metrics.example.com
spec:
  group: metrics.example.com
  version: v1alpha1
  service:
    name: custom-metrics-apiserver
    namespace: monitoring
    port: 443
  groupPriorityMinimum: 100
  versionPriority: 100
  caBundle: <base64-ca>
  insecureSkipTLSVerify: false

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: custom-metrics-apiserver
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: custom-metrics-apiserver
  template:
    metadata:
      labels:
        app: custom-metrics-apiserver
    spec:
      serviceAccountName: custom-metrics-apiserver
      containers:
        - name: server
          image: myregistry.io/custom-metrics:1.0
          args:
            - --secure-port=6443
            - --tls-cert-file=/var/run/serving-cert/tls.crt
            - --tls-private-key-file=/var/run/serving-cert/tls.key
          ports:
            - containerPort: 6443
          volumeMounts:
            - name: serving-cert
              mountPath: /var/run/serving-cert
              readOnly: true
      volumes:
        - name: serving-cert
          secret:
            secretName: cm-adapter-serving-certs
```

---

## Step 395: Webhook Conversion

```python
# CRD Version conversion webhook (Python)
from flask import Flask, request, jsonify

app = Flask(__name__)

@app.route('/convert', methods=['POST'])
def convert():
    conversion_request = request.json
    uid = conversion_request["request"]["uid"]
    desired_api_version = conversion_request["request"]["desiredAPIVersion"]
    objects = conversion_request["request"]["objects"]
    
    converted_objects = [convert_object(obj, desired_api_version) for obj in objects]
    
    return jsonify({
        "apiVersion": "apiextensions.k8s.io/v1",
        "kind": "ConversionReview",
        "response": {
            "uid": uid,
            "result": {"status": "Success"},
            "convertedObjects": converted_objects
        }
    })

def convert_object(obj: dict, desired_version: str) -> dict:
    current_version = obj["apiVersion"].split("/")[-1]
    
    if current_version == desired_version:
        return obj
    
    if current_version == "v1alpha1" and desired_version == "app.example.com/v1":
        return {
            **obj,
            "apiVersion": desired_version,
            "spec": {
                "image": obj["spec"]["image"],
                "replicas": obj["spec"].get("replicas", 1),
                "resources": {
                    "requests": {"cpu": "100m", "memory": "128Mi"},
                    "limits": {"cpu": "500m", "memory": "512Mi"}
                }
            }
        }
    
    return obj
```

---

## Step 396: Category and PrinterColumns

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: webapps.web.example.com
spec:
  group: web.example.com
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                url:
                  type: string
                replicas:
                  type: integer
            status:
              type: object
              properties:
                availableReplicas:
                  type: integer
                health:
                  type: string
      
      additionalPrinterColumns:
        - name: URL
          type: string
          jsonPath: .spec.url
        - name: Replicas
          type: integer
          jsonPath: .spec.replicas
        - name: Available
          type: integer
          jsonPath: .status.availableReplicas
        - name: Health
          type: string
          jsonPath: .status.health
        - name: Age
          type: date
          jsonPath: .metadata.creationTimestamp
      
      subresources:
        status: {}
        scale:
          specReplicasPath: .spec.replicas
          statusReplicasPath: .status.availableReplicas
  
  scope: Namespaced
  names:
    plural: webapps
    singular: webapp
    kind: WebApp
    categories:
      - all
      - web
```

---

## Step 397-398: Defaulting และ Validation

```go
// Defaulting
func (r *Application) Default() {
    if r.Spec.Replicas == 0 {
        r.Spec.Replicas = 1
    }
    if r.Spec.Resources.Requests == nil {
        r.Spec.Resources.Requests = corev1.ResourceList{
            corev1.ResourceCPU:    resource.MustParse("100m"),
            corev1.ResourceMemory: resource.MustParse("128Mi"),
        }
    }
}

// Validation
func (r *Application) ValidateCreate() (admission.Warnings, error) {
    return r.validateApplication()
}

func (r *Application) ValidateUpdate(old runtime.Object) (admission.Warnings, error) {
    oldApp := old.(*Application)
    if oldApp.Spec.Engine != r.Spec.Engine {
        return nil, fmt.Errorf("spec.engine is immutable")
    }
    return r.validateApplication()
}

func (r *Application) validateApplication() (admission.Warnings, error) {
    var errs field.ErrorList
    if !strings.Contains(r.Spec.Image, "/") {
        errs = append(errs, field.Invalid(
            field.NewPath("spec").Child("image"),
            r.Spec.Image,
            "image must include registry prefix",
        ))
    }
    if len(errs) > 0 {
        return nil, errs.ToAggregate()
    }
    return nil, nil
}
```

---

## Step 399-400: Workshop

```yaml
# MicroService CRD
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: microservices.platform.example.com
spec:
  group: platform.example.com
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              required: ["image", "ports"]
              properties:
                image:
                  type: string
                replicas:
                  type: object
                  properties:
                    min:
                      type: integer
                      minimum: 1
                    max:
                      type: integer
                      minimum: 1
                  x-kubernetes-validations:
                    - rule: "self.min <= self.max"
                      message: "replicas.min must be <= replicas.max"
                ports:
                  type: array
                  minItems: 1
                  items:
                    type: object
                    required: ["name", "port"]
                    properties:
                      name:
                        type: string
                      port:
                        type: integer
                        minimum: 1
                        maximum: 65535
                slo:
                  type: object
                  properties:
                    availability:
                      type: number
                      minimum: 0
                      maximum: 100
                    latencyP99Ms:
                      type: integer
            status:
              type: object
              properties:
                phase:
                  type: string
                readyReplicas:
                  type: integer
  scope: Namespaced
  names:
    plural: microservices
    singular: microservice
    kind: MicroService
    shortNames: ["ms"]
    categories: ["all"]
```

---

## 📊 สรุป Part 41

| Feature | รายละเอียด |
|---------|----------|
| CRD Versioning | หลาย versions พร้อม conversion |
| Schema Validation | OpenAPI v3 + CEL rules |
| Aggregated API | Custom API servers |
| Printer Columns | Custom kubectl output |
| Defaulting Webhook | Set default values |
| Validation Webhook | Validate on create/update |
| Subresources | /status /scale endpoints |

---

## 🔗 ต่อไป
- [Part 42: Kubernetes API Server Internals](./part-42-api-server.md)

---
*Part 41 | Steps 391-400 | ระดับสูง*
