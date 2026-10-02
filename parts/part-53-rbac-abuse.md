# Part 53: RBAC Abuse & Privilege Escalation
## Steps 501-510: Kubernetes RBAC Security

---

## 📖 บทนำ

> **⚠️ หมายเหตุ**: เนื้อหาด้าน security ในหลักสูตรนี้มีไว้เพื่อการศึกษาและการทดสอบระบบที่ได้รับอนุญาตเท่านั้น การนำไปใช้โจมตีระบบที่ไม่ได้รับอนุญาตเป็นสิ่งผิดกฎหมาย

RBAC (Role-Based Access Control) เป็น authorization mechanism หลักใน Kubernetes การ misconfigure RBAC อาจนำไปสู่ privilege escalation ได้

---

## Step 501: RBAC Architecture

```
Kubernetes RBAC Flow:

Request → API Server → Authentication → Authorization → Admission

RBAC Components:
  Subject (Who)       Role/ClusterRole (What)    Binding (Map)
  User                Role (namespace)           RoleBinding
  Group               ClusterRole (cluster)      ClusterRoleBinding
  ServiceAccount      Rules: verbs+resources

Escalation Paths (for defenders):
  1. bind verb → can bind ClusterRole to self
  2. escalate verb → can grant higher privileges
  3. impersonate → can impersonate other users
  4. create/patch RBAC → can modify permissions
  5. create pods with SA → inherit SA permissions
  6. exec into privileged pod → escape to host
```

---

## Step 502: Dangerous RBAC Permissions

```yaml
# Permissions that allow privilege escalation:

# 1. wildcards (*) - overly broad
# - apiGroups: ["*"]
#   resources: ["*"]
#   verbs: ["*"]
# -> Can do ANYTHING

# 2. secrets get/list - credential theft
# - resources: ["secrets"]
#   verbs: ["get", "list"]
# -> Read all secrets

# 3. pods/exec - code execution
# - resources: ["pods/exec"]
#   verbs: ["create"]
# -> Exec into privileged pods

# 4. nodes proxy - access node APIs
# - resources: ["nodes/proxy"]
#   verbs: ["get"]
# -> Kubelet API access

# 5. bind/escalate meta-permission
# - apiGroups: ["rbac.authorization.k8s.io"]
#   resources: ["clusterrolebindings"]
#   verbs: ["create", "bind", "escalate"]
# -> Grant cluster-admin to self

---
# ValidatingAdmissionPolicy: block wildcards
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicy
metadata:
  name: restrict-rbac-wildcards
spec:
  failurePolicy: Fail
  matchConstraints:
    resourceRules:
      - apiGroups: ["rbac.authorization.k8s.io"]
        apiVersions: ["v1"]
        operations: ["CREATE", "UPDATE"]
        resources: ["roles", "clusterroles"]
  validations:
    - expression: |
        !object.rules.exists(rule,
          rule.verbs.exists(v, v == "*") ||
          rule.resources.exists(r, r == "*") ||
          rule.apiGroups.exists(g, g == "*")
        )
      message: "Wildcard permissions (*) are not allowed in Roles"
```

---

## Step 503: Least Privilege RBAC

```yaml
# Minimal permissions for application
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: app-role
  namespace: production
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get"]
    resourceNames: ["my-app-config"]  # specific resource only
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get"]
    resourceNames: ["my-app-secret"]

---
# ServiceAccount: disable auto-mount
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app
  namespace: production
automountServiceAccountToken: false

---
# Pod: disable SA token mount
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  serviceAccountName: my-app
  automountServiceAccountToken: false
  containers:
    - name: app
      image: myregistry.io/my-app:1.0
```

---

## Step 504: RBAC Audit Tools

```yaml
# Audit commands:
# kubectl auth can-i get secrets --as=system:serviceaccount:prod:my-app
# kubectl auth can-i create pods --as=system:serviceaccount:default:default

# kubectl-who-can plugin
# kubectl-who-can get secrets
# kubectl-who-can exec pods
# kubectl-who-can create clusterrolebindings

# rbac-lookup plugin
# kubectl rbac-lookup my-serviceaccount -o wide

# Find cluster-admin bindings
# kubectl get clusterrolebindings -o json | \
#   jq '.items[] | select(.roleRef.name=="cluster-admin") | .subjects'

# Falco: RBAC change detection
# - rule: RBAC manipulation
#   condition: >
#     ka.target.resource in (clusterroles,clusterrolebindings,roles,rolebindings)
#     and not ka.user.name startswith "system:"
#   output: RBAC modified (user=%ka.user.name resource=%ka.target.name)
#   priority: WARNING
```

---

## Step 505: ServiceAccount Token Security

```yaml
# Projected SA Token (short-lived, scoped)
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  serviceAccountName: my-app
  automountServiceAccountToken: false
  volumes:
    - name: token
      projected:
        sources:
          - serviceAccountToken:
              path: token
              expirationSeconds: 3600  # 1 hour
              audience: my-app-audience
          - configMap:
              name: kube-root-ca.crt
              items:
                - key: ca.crt
                  path: ca.crt
  containers:
    - name: app
      image: myregistry.io/my-app:1.0
      volumeMounts:
        - name: token
          mountPath: /var/run/secrets/tokens
          readOnly: true

---
# IRSA (IAM Roles for Service Accounts) on EKS
apiVersion: v1
kind: ServiceAccount
metadata:
  name: aws-app
  namespace: production
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789:role/my-app-role
    eks.amazonaws.com/token-expiration: "3600"
automountServiceAccountToken: false
```

---

## Step 506: Node Authorization

```yaml
# NodeRestriction admission plugin
# --enable-admission-plugins=NodeRestriction

# NodeRestriction prevents:
# - Node can only modify its own Node object
# - Node can only read/modify Pods bound to it
# - Node cannot read secrets for other nodes
# - Node cannot modify other nodes' configmaps

# ValidatingAdmissionPolicy: protect node labels
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicy
metadata:
  name: protect-node-labels
spec:
  failurePolicy: Fail
  matchConstraints:
    resourceRules:
      - apiGroups: [""]
        apiVersions: ["v1"]
        operations: ["UPDATE"]
        resources: ["nodes"]
  validations:
    - expression: |
        oldObject.metadata.labels == object.metadata.labels ||
        request.userInfo.groups.exists(g, g == "system:masters")
      message: "Node labels can only be modified by cluster admins"
```

---

## Step 507: Impersonation Security

```yaml
# Limited impersonation role
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: limited-impersonation
rules:
  - apiGroups: [""]
    resources: ["users"]
    verbs: ["impersonate"]
    resourceNames: ["specific-user-only"]
  - apiGroups: [""]
    resources: ["serviceaccounts"]
    verbs: ["impersonate"]
    resourceNames: ["specific-sa"]

# Audit impersonation:
# jq '.items[] | select(.annotations."authorization.k8s.io/decision" == "allow")
#   | select(.userAgent | contains("impersonate"))' audit.log
```

---

## Step 508: Pod Security Standards

```yaml
# Enable PSS per namespace
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: v1.29
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/audit: restricted

# PSS restricted requires:
# - runAsNonRoot: true
# - allowPrivilegeEscalation: false
# - seccompProfile: RuntimeDefault
# - capabilities.drop: [ALL]
# - No privileged, no hostPath, no hostNetwork/PID/IPC

---
# Kyverno: custom PSS additions
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: enhanced-pod-security
spec:
  validationFailureAction: Enforce
  rules:
    - name: require-resource-limits
      match:
        any:
          - resources:
              kinds: [Pod]
      validate:
        message: "Resource limits required"
        pattern:
          spec:
            containers:
              - resources:
                  limits:
                    memory: "?*"
                    cpu: "?*"
```

---

## Step 509: RBAC Testing

```yaml
# Test RBAC policies:
# kubectl auth can-i get secrets --as=system:serviceaccount:production:my-app
# kubectl auth can-i "*" "*" --as=attacker  # should return 'no'

# Conftest RBAC policy (policy/rbac.rego):
# package main
#
# deny[msg] {
#   input.kind == "Role"
#   rule := input.rules[_]
#   rule.verbs[_] == "*"
#   msg := sprintf("Role %v uses wildcard verbs", [input.metadata.name])
# }
#
# deny[msg] {
#   input.kind == "ClusterRoleBinding"
#   input.roleRef.name == "cluster-admin"
#   subject := input.subjects[_]
#   subject.kind == "ServiceAccount"
#   msg := sprintf("SA %v/%v has cluster-admin", [subject.namespace, subject.name])
# }
```

---

## Step 510: Workshop - RBAC Hardening

```yaml
# Minimal operator ServiceAccount
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-operator
  namespace: operators
automountServiceAccountToken: false

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: my-operator
rules:
  - apiGroups: ["mygroup.io"]
    resources: ["myresources"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
  - apiGroups: [""]
    resources: ["services", "configmaps"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
  # NO secrets, NO pods/exec, NO RBAC modification

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: my-operator
subjects:
  - kind: ServiceAccount
    name: my-operator
    namespace: operators
roleRef:
  kind: ClusterRole
  name: my-operator
  apiGroup: rbac.authorization.k8s.io

---
# Weekly RBAC audit CronJob
apiVersion: batch/v1
kind: CronJob
metadata:
  name: rbac-audit
  namespace: security
spec:
  schedule: "0 0 * * 1"
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: rbac-auditor
          restartPolicy: OnFailure
          containers:
            - name: audit
              image: bitnami/kubectl:latest
              command:
                - /bin/sh
                - -c
                - |
                  echo "=== Cluster Admin Bindings ==="
                  kubectl get clusterrolebindings -o json | \
                    jq -r '.items[] | select(.roleRef.name=="cluster-admin") |
                    "\(.metadata.name): \(.subjects)"'
```

---

## 📊 สรุป Part 53

| RBAC Risk | Mitigation |
|-----------|------------|
| Wildcards (*) | ValidatingAdmissionPolicy deny |
| cluster-admin bindings | Audit + remove unnecessary |
| SA token auto-mount | automountServiceAccountToken: false |
| Secrets access | Limit with resourceNames |
| pods/exec permission | Only for debugging, time-limited |
| RBAC modification | Only for platform team |

---

## 🔗 ต่อไป
- [Part 54: Secrets Exposure & Protection](./part-54-secrets-exposure.md)

---
*Part 53 | Steps 501-510 | ระดับผู้เชี่ยวชาญ | Educational Use Only*
