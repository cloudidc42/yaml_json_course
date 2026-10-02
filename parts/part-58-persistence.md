# Part 58: Persistence Mechanisms & Detection
## Steps 541-550

> **⚠️**: เนื้อหาด้าน security ในหลักสูตรนี้มีไว้เพื่อการศึกษาและการทดสอบระบบที่ได้รับอนุญาตเท่านั้น การนำไปใช้โจมตีระบบที่ไม่ได้รับอนุญาตเป็นสิ่งผิดกฎหมาย

## Step 541: Persistence Techniques Overview

```
Kubernetes Persistence Mechanisms:

1. Backdoor ServiceAccount
   Create SA with ClusterAdmin -> survives pod deletion
   Detection: audit new ClusterRoleBindings

2. Malicious Webhook
   Create ValidatingWebhookConfiguration pointing to attacker server
   -> intercept/modify all API requests
   Detection: audit webhooks, check TLS certs

3. CronJob Backdoor
   Deploy CronJob that re-deploys malware every 5m
   -> survives manual pod deletion
   Detection: audit CronJobs in all namespaces

4. DaemonSet Backdoor
   DaemonSet on all nodes -> survives node rotation
   Detection: audit DaemonSets

5. Persistent Volume Access
   hostPath mount to node -> write to node filesystem
   Detection: Falco hostPath events

6. Node-level Persistence
   Write to /etc/cron.d, /etc/systemd, SSH keys
   Detection: Tetragon file write monitoring
```

## Step 542: Detecting Backdoor Service Accounts

```yaml
# Falco: detect new ClusterRoleBinding creation
- rule: New ClusterRoleBinding Created
  desc: Detect creation of ClusterRoleBinding
  condition: >
    ka.verb = create and
    ka.target.resource = clusterrolebindings
  output: >
    New ClusterRoleBinding (user=%ka.user.name
    binding=%ka.target.name)
  priority: WARNING
  tags: [persistence, privilege_escalation]

---
# Falco: detect binding to cluster-admin
- rule: Cluster Admin Binding Created
  desc: Detect new binding to cluster-admin role
  condition: >
    ka.verb = create and
    ka.target.resource in (clusterrolebindings, rolebindings) and
    ka.req.binding.role = cluster-admin
  output: >
    CRITICAL: cluster-admin binding created!
    (user=%ka.user.name binding=%ka.target.name)
  priority: CRITICAL
  tags: [persistence, privilege_escalation]

---
# RBAC monitoring CronJob
apiVersion: batch/v1
kind: CronJob
metadata:
  name: rbac-persistence-audit
  namespace: security
spec:
  schedule: "*/30 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: rbac-auditor
          containers:
            - name: auditor
              image: bitnami/kubectl:latest
              command:
                - /bin/sh
                - -c
                - |
                  echo '=== ClusterRoleBindings to cluster-admin ==='
                  kubectl get clusterrolebindings -o json | \
                    jq '.items[] | select(.roleRef.name == "cluster-admin") |
                      {name: .metadata.name, subjects: .subjects}'
                  echo '=== CronJobs ==='
                  kubectl get cronjobs --all-namespaces
                  echo '=== Webhooks ==='
                  kubectl get validatingwebhookconfigurations 2>/dev/null
          restartPolicy: OnFailure
```

## Step 543: Webhook Backdoor Detection

```yaml
# Kyverno: validate webhook configurations
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: audit-webhook-changes
spec:
  validationFailureAction: Audit
  background: false
  rules:
    - name: validate-webhook-url
      match:
        any:
          - resources:
              kinds:
                - ValidatingWebhookConfiguration
                - MutatingWebhookConfiguration
      validate:
        message: "Webhook URLs must use internal cluster services"
        deny:
          conditions:
            any:
              - key: "{{ request.object.webhooks[].clientConfig.url | to_array(@) | length(@) }}"
                operator: GreaterThan
                value: "0"
```

## Step 544: CronJob Backdoor Detection

```yaml
# Falco: detect unexpected CronJob creation
- rule: Suspicious CronJob Created
  desc: CronJob with short schedule in non-system namespace
  condition: >
    ka.verb = create and
    ka.target.resource = cronjobs and
    not ka.target.namespace in (kube-system, monitoring, security) and
    ka.req.object.spec.schedule contains "*/"
  output: >
    Suspicious CronJob created (user=%ka.user.name
    namespace=%ka.target.namespace job=%ka.target.name)
  priority: WARNING
  tags: [persistence, execution]

---
# ValidatingAdmissionPolicy: restrict CronJob creation
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicy
metadata:
  name: restrict-cronjob-creation
spec:
  failurePolicy: Fail
  matchConstraints:
    resourceRules:
      - apiGroups: ["batch"]
        apiVersions: ["v1"]
        operations: ["CREATE", "UPDATE"]
        resources: ["cronjobs"]
  validations:
    - expression: |
        has(object.metadata.labels) &&
        has(object.metadata.labels["security.approved"]) &&
        object.metadata.labels["security.approved"] == "true"
      message: "CronJobs require security.approved=true label"
```

## Step 545: DaemonSet Backdoor Detection

```yaml
# Kyverno: restrict DaemonSet to system namespaces
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: restrict-daemonset-namespaces
spec:
  validationFailureAction: Enforce
  rules:
    - name: daemonset-allowed-namespaces
      match:
        any:
          - resources:
              kinds: [DaemonSet]
      validate:
        message: "DaemonSets only allowed in system namespaces"
        deny:
          conditions:
            all:
              - key: "{{ request.object.metadata.namespace }}"
                operator: NotIn
                value:
                  - kube-system
                  - monitoring
                  - security
                  - logging

---
# Prometheus: alert on DaemonSet changes
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: persistence-alerts
  namespace: monitoring
spec:
  groups:
    - name: persistence
      rules:
        - alert: UnexpectedDaemonSetCreated
          expr: |
            increase(apiserver_audit_event_total{
              verb="create",
              resource="daemonsets"
            }[5m]) > 0
          annotations:
            summary: "New DaemonSet created - verify authorization"
          labels:
            severity: warning
        
        - alert: UnexpectedWebhookModified
          expr: |
            increase(apiserver_audit_event_total{
              verb=~"create|update|patch",
              resource=~"validatingwebhookconfigurations|mutatingwebhookconfigurations"
            }[5m]) > 0
          annotations:
            summary: "Webhook configuration modified"
          labels:
            severity: critical
```

## Step 546: Node-Level Persistence Detection

```yaml
# Tetragon: detect writes to sensitive node paths
apiVersion: cilium.io/v1alpha1
kind: TracingPolicy
metadata:
  name: detect-node-persistence
spec:
  kprobes:
    - call: "security_file_open"
      syscall: false
      args:
        - index: 0
          type: "file"
      selectors:
        - matchArgs:
            - index: 0
              operator: "Prefix"
              values:
                - "/host/etc/cron"
                - "/host/etc/systemd"
                - "/host/root/.ssh"
          matchActions:
            - action: Post

---
# Falco: detect writes to node persistence paths
- rule: Write to Node Persistence Path
  desc: Detect writes to cron, systemd, or SSH paths on node
  condition: >
    open_write and
    container and
    (fd.name startswith /host/etc/cron or
     fd.name startswith /host/etc/systemd or
     fd.name startswith /host/root/.ssh)
  output: >
    Write to node persistence path from container
    (container=%container.id image=%container.image.repository
    path=%fd.name proc=%proc.name)
  priority: CRITICAL
  tags: [persistence, node]
```

## Step 547: Immutable Infrastructure

```yaml
# Enforce immutable containers
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: enforce-immutable-containers
spec:
  validationFailureAction: Enforce
  rules:
    - name: readonly-rootfs
      match:
        any:
          - resources:
              kinds: [Pod]
              namespaces: [production]
      validate:
        message: "Containers must use readOnlyRootFilesystem: true"
        pattern:
          spec:
            containers:
              - securityContext:
                  readOnlyRootFilesystem: true

---
# Pod with read-only filesystem + tmpfs
apiVersion: v1
kind: Pod
metadata:
  name: immutable-app
  namespace: production
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 10001
    seccompProfile:
      type: RuntimeDefault
  containers:
    - name: app
      image: myregistry.io/app:1.0
      securityContext:
        readOnlyRootFilesystem: true
        allowPrivilegeEscalation: false
        capabilities:
          drop: [ALL]
      volumeMounts:
        - name: tmp
          mountPath: /tmp
        - name: cache
          mountPath: /app/cache
  volumes:
    - name: tmp
      emptyDir:
        medium: Memory
    - name: cache
      emptyDir:
        medium: Memory
        sizeLimit: 100Mi
```

## Step 548: Admission Control Defense

```yaml
# OPA Gatekeeper: persistence prevention
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: nopersistencerisks
spec:
  crd:
    spec:
      names:
        kind: NoPersistenceRisks
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package nopersistencerisks
        
        violation[{"msg": msg}] {
          input.review.object.kind == "ServiceAccount"
          input.review.object.metadata.namespace == "production"
          not input.review.object.automountServiceAccountToken == false
          msg := "ServiceAccounts in production must set automountServiceAccountToken: false"
        }
        
        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          mount := container.volumeMounts[_]
          sensitive_path(mount.mountPath)
          msg := sprintf("Container mounts sensitive path: %v", [mount.mountPath])
        }
        
        sensitive_path(path) {
          paths := ["/etc/kubernetes", "/var/lib/kubelet", "/.kube"]
          startswith(path, paths[_])
        }
```

## Step 549: Incident Response - Persistence Investigation

```yaml
# Argo Workflows: persistence investigation runbook
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: persistence-investigation-
  namespace: security
spec:
  entrypoint: investigate
  templates:
    - name: investigate
      dag:
        tasks:
          - name: audit-service-accounts
            template: kubectl-audit
            arguments:
              parameters:
                - name: cmd
                  value: "get serviceaccounts --all-namespaces -o json | jq '.items[] | select(.automountServiceAccountToken != false) | {ns: .metadata.namespace, name: .metadata.name}'"
          
          - name: audit-clusterrolebindings
            template: kubectl-audit
            arguments:
              parameters:
                - name: cmd
                  value: "get clusterrolebindings -o json | jq '.items[] | {name: .metadata.name, role: .roleRef.name}'"
          
          - name: audit-cronjobs
            template: kubectl-audit
            arguments:
              parameters:
                - name: cmd
                  value: "get cronjobs --all-namespaces"
    
    - name: kubectl-audit
      inputs:
        parameters:
          - name: cmd
      container:
        image: bitnami/kubectl:latest
        command: [sh, -c]
        args: ["kubectl {{inputs.parameters.cmd}}"]
```

## Step 550: Workshop - Persistence Detection Pipeline

```yaml
# Comprehensive persistence monitoring
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: persistence-comprehensive
  namespace: monitoring
spec:
  groups:
    - name: k8s-persistence
      rules:
        - alert: BackdoorSACreated
          expr: |
            increase(falco_events_total{rule="New ClusterRoleBinding Created"}[5m]) > 0
          labels:
            severity: critical
            category: persistence
          annotations:
            summary: "Possible backdoor SA/ClusterRoleBinding created"
            runbook: "https://wiki.internal/runbooks/k8s-persistence"
        
        - alert: WebhookBackdoor
          expr: |
            increase(falco_events_total{rule=~".*Webhook.*"}[5m]) > 0
          labels:
            severity: critical
            category: persistence
          annotations:
            summary: "Webhook configuration unexpectedly modified"
```

---

## 📊 สรุป Part 58

| Persistence Technique | Detection | Prevention |
|----------------------|-----------|------------|
| Backdoor SA | Falco RBAC rules | RBAC audit + Kyverno |
| Malicious webhook | Prometheus alert | OPA webhook validation |
| CronJob backdoor | Falco CronJob rules | ValidatingAdmissionPolicy |
| DaemonSet backdoor | Prometheus alert | Kyverno namespace restriction |
| Node filesystem write | Tetragon + Falco | readOnlyRootFilesystem |

---

## 🔗 ต่อไป
- [Part 59: Detection Engineering & Forensics](./part-59-detection-forensics.md)

---
*Part 58 | Steps 541-550 | ระดับผู้เชี่ยวชาญ | Educational Use Only*
