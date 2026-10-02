# Part 37: Admission Controllers
## Steps 351-360: Validating, Mutating Webhooks, OPA Gatekeeper

---

## 📖 บทนำ

Admission Controllers เป็น plugins ที่ intercept API requests ก่อน object ถูก create/update ใน etcd ช่วย enforce policies และ mutate resources

> **หมายเหตุ**: เนื้อหาด้าน security ในหลักสูตรนี้มีไว้เพื่อการศึกษาและการทดสอบระบบที่ได้รับอนุญาตเท่านั้น

---

## Step 351: Admission Control Flow

```
API Request Flow:
┌─────────────────────────────────────────────────────────┐
│  kubectl apply                                          │
│  ┌───────────────┐  ┌───────────────┐                 │
│  │ Authn          │  │ Authz          │                 │
│  └───────────────┘  └───────────────┘                 │
│  ┌──────────────────────────────────────────────┐ │
│  │ Admission Controllers                          │ │
│  │ 1. Mutating webhooks                           │ │
│  │ 2. Object validation                           │ │
│  │ 3. Validating webhooks                         │ │
│  └──────────────────────────────────────────────┘ │
│  ┌──────────────┐                                   │
│  │ etcd (persist)│                                   │
│  └──────────────┘                                   │
└─────────────────────────────────────────────────────────┘
```

---

## Step 352: MutatingAdmissionWebhook

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingWebhookConfiguration
metadata:
  name: resource-injector
webhooks:
  - name: inject-resources.example.com
    admissionReviewVersions: ["v1"]
    sideEffects: None
    clientConfig:
      service:
        name: admission-webhook
        namespace: webhook-system
        path: /mutate
      caBundle: <base64-encoded-CA-cert>
    rules:
      - operations: ["CREATE"]
        apiGroups: ["apps"]
        apiVersions: ["v1"]
        resources: ["deployments"]
    namespaceSelector:
      matchLabels:
        webhook.io/enabled: "true"
    failurePolicy: Ignore
    timeoutSeconds: 5
```

---

## Step 353: Webhook Server

```python
import base64
import json
from http.server import HTTPServer, BaseHTTPRequestHandler

DEFAULT_RESOURCES = {
    "requests": {"cpu": "100m", "memory": "128Mi"},
    "limits": {"cpu": "500m", "memory": "512Mi"}
}

DEFAULT_SECURITY = {
    "allowPrivilegeEscalation": False,
    "readOnlyRootFilesystem": True,
    "runAsNonRoot": True,
    "capabilities": {"drop": ["ALL"]}
}

class WebhookHandler(BaseHTTPRequestHandler):
    def do_POST(self):
        content_length = int(self.headers.get('Content-Length', 0))
        body = self.rfile.read(content_length)
        request = json.loads(body)
        
        if self.path == "/mutate":
            response = self.mutate(request)
        elif self.path == "/validate":
            response = self.validate(request)
        else:
            self.send_response(404)
            self.end_headers()
            return
        
        self.send_response(200)
        self.send_header("Content-Type", "application/json")
        self.end_headers()
        self.wfile.write(json.dumps(response).encode())
    
    def mutate(self, request: dict) -> dict:
        uid = request["request"]["uid"]
        obj = request["request"]["object"]
        patches = []
        containers = obj.get("spec", {}).get("template", {}).get("spec", {}).get("containers", [])
        for i, container in enumerate(containers):
            if not container.get("resources"):
                patches.append({"op": "add", "path": f"/spec/template/spec/containers/{i}/resources", "value": DEFAULT_RESOURCES})
            if not container.get("securityContext"):
                patches.append({"op": "add", "path": f"/spec/template/spec/containers/{i}/securityContext", "value": DEFAULT_SECURITY})
        patch_b64 = base64.b64encode(json.dumps(patches).encode()).decode()
        return {"apiVersion": "admission.k8s.io/v1", "kind": "AdmissionReview", "response": {"uid": uid, "allowed": True, "patchType": "JSONPatch", "patch": patch_b64}}
    
    def validate(self, request: dict) -> dict:
        uid = request["request"]["uid"]
        obj = request["request"]["object"]
        errors = []
        for c in obj.get("spec", {}).get("template", {}).get("spec", {}).get("containers", []):
            if not c.get("resources", {}).get("limits"):
                errors.append(f"Container '{c['name']}' missing resource limits")
            if c.get("securityContext", {}).get("privileged"):
                errors.append(f"Container '{c['name']}' must not be privileged")
        if errors:
            return {"apiVersion": "admission.k8s.io/v1", "kind": "AdmissionReview", "response": {"uid": uid, "allowed": False, "status": {"code": 400, "message": "; ".join(errors)}}}
        return {"apiVersion": "admission.k8s.io/v1", "kind": "AdmissionReview", "response": {"uid": uid, "allowed": True}}
```

---

## Step 354: ValidatingAdmissionWebhook

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: security-validator
webhooks:
  - name: validate-security.example.com
    admissionReviewVersions: ["v1"]
    sideEffects: None
    clientConfig:
      service:
        name: admission-webhook
        namespace: webhook-system
        path: /validate
      caBundle: <base64-encoded-CA-cert>
    rules:
      - operations: ["CREATE", "UPDATE"]
        apiGroups: ["apps"]
        apiVersions: ["v1"]
        resources: ["deployments", "daemonsets", "statefulsets"]
    failurePolicy: Fail
    namespaceSelector:
      matchExpressions:
        - key: kubernetes.io/metadata.name
          operator: NotIn
          values: ["kube-system", "monitoring"]
    timeoutSeconds: 10
```

---

## Step 355: ValidatingAdmissionPolicy (CEL)

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicy
metadata:
  name: require-resource-limits
spec:
  failurePolicy: Fail
  matchConstraints:
    resourceRules:
      - apiGroups: ["apps"]
        apiVersions: ["v1"]
        operations: ["CREATE", "UPDATE"]
        resources: ["deployments"]
  
  validations:
    - expression: |
        object.spec.template.spec.containers.all(
          c, has(c.resources) && has(c.resources.limits) &&
          has(c.resources.limits.cpu) &&
          has(c.resources.limits.memory)
        )
      message: "All containers must have CPU and memory limits"
    
    - expression: |
        object.spec.template.spec.containers.all(
          c, !has(c.securityContext) ||
          !has(c.securityContext.privileged) ||
          c.securityContext.privileged == false
        )
      message: "Privileged containers are not allowed"

---
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicyBinding
metadata:
  name: require-resource-limits-binding
spec:
  policyName: require-resource-limits
  validationActions: [Deny]
  matchResources:
    namespaceSelector:
      matchLabels:
        policy: enforced
```

---

## Step 356: OPA Gatekeeper

```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequiredlabels
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredLabels
      validation:
        openAPIV3Schema:
          type: object
          properties:
            labels:
              type: array
              items:
                type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequiredlabels
        violation[{"msg": msg, "details": {"missing_labels": missing}}] {
          provided := {label | input.review.object.metadata.labels[label]}
          required := {label | label := input.parameters.labels[_]}
          missing := required - provided
          count(missing) > 0
          msg := sprintf("Missing required labels: %v", [missing])
        }

---
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredLabels
metadata:
  name: require-team-label
spec:
  enforcementAction: deny
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Namespace"]
      - apiGroups: ["apps"]
        kinds: ["Deployment"]
    excludedNamespaces:
      - kube-system
      - gatekeeper-system
  parameters:
    labels:
      - team
      - owner
      - environment
```

---

## Step 357: Gatekeeper Policies

```yaml
# Allowed image registries
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8sallowedrepos
spec:
  crd:
    spec:
      names:
        kind: K8sAllowedRepos
      validation:
        openAPIV3Schema:
          type: object
          properties:
            repos:
              type: array
              items:
                type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8sallowedrepos
        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not starts_with_allowed(container.image)
          msg := sprintf("Container '%v' uses unauthorized image: %v", [container.name, container.image])
        }
        starts_with_allowed(image) {
          repo := input.parameters.repos[_]
          startswith(image, repo)
        }

---
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sAllowedRepos
metadata:
  name: allowed-repos
spec:
  enforcementAction: deny
  match:
    kinds:
      - apiGroups: ["apps"]
        kinds: ["Deployment", "StatefulSet"]
  parameters:
    repos:
      - "myregistry.io/"
      - "gcr.io/my-project/"
```

---

## Step 358-359: Webhook Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: admission-webhook
  namespace: webhook-system
spec:
  replicas: 2
  selector:
    matchLabels:
      app: admission-webhook
  template:
    metadata:
      labels:
        app: admission-webhook
    spec:
      serviceAccountName: admission-webhook
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        seccompProfile:
          type: RuntimeDefault
      containers:
        - name: webhook
          image: myregistry.io/admission-webhook:1.0
          ports:
            - containerPort: 8443
          volumeMounts:
            - name: tls-certs
              mountPath: /etc/certs
              readOnly: true
          livenessProbe:
            httpGet:
              path: /healthz
              port: 8080
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
            limits:
              cpu: 200m
              memory: 256Mi
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop: ["ALL"]
      volumes:
        - name: tls-certs
          secret:
            secretName: admission-webhook-tls
```

---

## Step 360: Workshop

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicy
metadata:
  name: no-latest-tag
spec:
  failurePolicy: Fail
  matchConstraints:
    resourceRules:
      - apiGroups: ["apps"]
        apiVersions: ["v1"]
        operations: ["CREATE", "UPDATE"]
        resources: ["deployments", "statefulsets"]
  validations:
    - expression: |
        object.spec.template.spec.containers.all(
          c, !c.image.endsWith(":latest") && c.image.contains(":")
        )
      message: "Image tag ':latest' or missing tag is not allowed"

---
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicyBinding
metadata:
  name: no-latest-tag-binding
spec:
  policyName: no-latest-tag
  validationActions: [Deny]
  matchResources:
    namespaceSelector:
      matchLabels:
        policy: strict
```

---

## 📊 สรุป Part 37

| Component | หน้าที่ |
|-----------|----------|
| MutatingWebhook | Modify objects |
| ValidatingWebhook | Reject bad config |
| ValidatingAdmissionPolicy | CEL-based (ไม่ต้องมี webhook) |
| OPA Gatekeeper | Rego policy engine |
| ConstraintTemplate | Policy template |
| Constraint | Policy instance |

---

## 🔗 ต่อไป
- [Part 38: Pod Security Standards](./part-38-pod-security.md)

---
*Part 37 | Steps 351-360 | ระดับสูง*
