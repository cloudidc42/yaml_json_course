# Part 63: Compliance - CIS/NIST/SOC2/PCI
## Steps 591-600: Kubernetes Compliance Frameworks

---

> **⚠️**: เนื้อหาด้าน security ในหลักสูตรนี้มีไว้เพื่อการศึกษาและการทดสอบระบบที่ได้รับอนุญาตเท่านั้น

## Step 591: Compliance Frameworks Overview

```
Key compliance frameworks for Kubernetes:

CIS Kubernetes Benchmark:
  - CIS K8s v1.27 (latest)
  - 5 sections: Control Plane, Worker Nodes, Policies, etc.
  - Tool: kube-bench

NIST SP 800-190 (Container Security):
  - Image vulnerabilities
  - Configuration defects
  - Build pipeline compromises
  - Container runtime threats
  - Host OS vulnerabilities

SOC 2 Type II:
  - Security (CC6-CC9)
  - Availability (A1)
  - Confidentiality (C1)
  - Processing Integrity (PI1)
  - Privacy (P1-P8)

PCI DSS v4.0:
  - Requirement 2: Secure configurations
  - Requirement 6: Secure software
  - Requirement 8: Access control
  - Requirement 10: Logging & monitoring
  - Requirement 12: Security policies
```

---

## Step 592: CIS Benchmark Automation

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: kube-bench
  namespace: security
spec:
  template:
    metadata:
      labels:
        app: kube-bench
    spec:
      hostPID: true
      containers:
        - name: kube-bench
          image: aquasec/kube-bench:latest
          command: ["kube-bench"]
          args:
            - "--benchmark"
            - "cis-1.8"
            - "--json"
          volumeMounts:
            - name: var-lib-etcd
              mountPath: /var/lib/etcd
              readOnly: true
            - name: etc-kubernetes
              mountPath: /etc/kubernetes
              readOnly: true
            - name: var-lib-kubelet
              mountPath: /var/lib/kubelet
              readOnly: true
      volumes:
        - name: var-lib-etcd
          hostPath:
            path: /var/lib/etcd
        - name: etc-kubernetes
          hostPath:
            path: /etc/kubernetes
        - name: var-lib-kubelet
          hostPath:
            path: /var/lib/kubelet
      restartPolicy: Never

---
apiVersion: batch/v1
kind: CronJob
metadata:
  name: cis-benchmark-weekly
  namespace: security
spec:
  schedule: "0 2 * * 1"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: kube-bench
              image: aquasec/kube-bench:latest
              command:
                - /bin/sh
                - -c
                - |
                  kube-bench --json > /tmp/results.json
                  aws s3 cp /tmp/results.json \
                    s3://compliance-reports/cis/$(date +%Y-%m-%d)-results.json
          restartPolicy: OnFailure
```

---

## Step 593: NIST Container Security Controls

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: nist-image-vulnerability
spec:
  validationFailureAction: Enforce
  rules:
    - name: require-image-scan
      match:
        any:
          - resources:
              kinds: [Pod]
              namespaces: [production]
      validate:
        message: "Images must be from approved, scanned registry"
        pattern:
          spec:
            containers:
              - image: "myregistry.io/*"
    
    - name: no-latest-tag
      match:
        any:
          - resources:
              kinds: [Pod]
      validate:
        message: "Image tag 'latest' not allowed - use specific version"
        pattern:
          spec:
            containers:
              - image: "!*:latest"

---
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

---

## Step 594: SOC2 - Access Controls (CC6)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: soc2-developer-role
  annotations:
    soc2.control: "CC6.1"
    description: "Least privilege role for developers"
rules:
  - apiGroups: [""]
    resources: ["pods", "services", "configmaps", "events"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["apps"]
    resources: ["deployments", "replicasets"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["pods/log"]
    verbs: ["get"]

---
# SOC2 CC6.3: Quarterly access review
apiVersion: v1
kind: ConfigMap
metadata:
  name: access-review-schedule
  namespace: security
data:
  schedule.yaml: |
    frequency: quarterly
    responsible: security-team
    process:
      1. Export all RBAC bindings
      2. Review with team leads
      3. Remove stale access
      4. Document changes
    automation:
      schedule: "0 9 1 */3 *"
      script: |
        kubectl get rolebindings,clusterrolebindings \
          --all-namespaces -o json | \
          jq '.items[] | {name: .metadata.name,
                           ns: .metadata.namespace,
                           role: .roleRef.name,
                           subjects: .subjects}'
```

---

## Step 595: SOC2 - Monitoring (CC7)

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: soc2-monitoring
  namespace: monitoring
  annotations:
    soc2.control: "CC7.1"
spec:
  groups:
    - name: soc2-cc71
      rules:
        - alert: SOC2AuthenticationFailures
          expr: |
            increase(apiserver_authentication_attempts_total{result="failure"}[5m]) > 5
          labels:
            severity: warning
            soc2_control: "CC7.1"
          annotations:
            summary: "Multiple authentication failures"
        
        - alert: SOC2PrivilegedContainerStarted
          expr: |
            increase(falco_events_total{rule="Launch Privileged Container"}[5m]) > 0
          labels:
            severity: critical
            soc2_control: "CC7.1"
          annotations:
            summary: "Privileged container started - SOC2 CC7.1 event"

---
# SOC2: Log retention 1 year
apiVersion: v1
kind: ConfigMap
metadata:
  name: loki-retention
  namespace: logging
data:
  loki.yaml: |
    limits_config:
      retention_period: 8760h
      retention_stream:
        - selector: '{job="k8s-audit"}'
          priority: 1
          period: 8760h
```

---

## Step 596: PCI DSS Compliance

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: pci-dss-r2-secure-config
  annotations:
    pci-dss.requirement: "2.2"
spec:
  validationFailureAction: Enforce
  rules:
    - name: no-unnecessary-ports
      match:
        any:
          - resources:
              kinds: [Pod]
              namespaces: [pci-scope]
      validate:
        message: "PCI DSS R2.2: No unnecessary ports"
        pattern:
          spec:
            containers:
              - ports:
                  - containerPort: ">0"

---
# PCI DSS Requirement 10: Logging
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  - level: RequestResponse
    namespaces: ["pci-scope"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
    resources:
      - group: ""
        resources: ["*"]
```

---

## Step 597: Compliance as Code

```yaml
# CIS 5.2.1: Minimize privileged containers
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: cis-5.2.1-no-privileged
  annotations:
    cis.benchmark: "5.2.1"
spec:
  validationFailureAction: Enforce
  rules:
    - name: no-privileged
      match:
        any:
          - resources:
              kinds: [Pod]
      validate:
        message: "CIS 5.2.1: Privileged containers not allowed"
        pattern:
          spec:
            containers:
              - =(securityContext):
                  =(privileged): "false"

---
# CIS 5.2.2: No root containers
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: cis-5.2.2-no-root
  annotations:
    cis.benchmark: "5.2.2"
spec:
  validationFailureAction: Enforce
  rules:
    - name: no-root-containers
      match:
        any:
          - resources:
              kinds: [Pod]
      validate:
        message: "CIS 5.2.2: Root containers not allowed"
        anyPattern:
          - spec:
              securityContext:
                runAsNonRoot: true
          - spec:
              containers:
                - securityContext:
                    runAsUser: ">0"

---
# CIS 5.2.6: Drop NET_RAW
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: cis-5.2.6-no-net-raw
  annotations:
    cis.benchmark: "5.2.6"
spec:
  validationFailureAction: Enforce
  rules:
    - name: drop-net-raw
      match:
        any:
          - resources:
              kinds: [Pod]
      validate:
        message: "CIS 5.2.6: NET_RAW must be dropped"
        pattern:
          spec:
            containers:
              - securityContext:
                  capabilities:
                    drop:
                      - NET_RAW | ALL
```

---

## Step 598: Compliance Reporting

```yaml
# Polaris configuration
apiVersion: v1
kind: ConfigMap
metadata:
  name: polaris-config
  namespace: polaris
data:
  config.yaml: |
    checks:
      hostIPCSet: error
      hostPIDSet: error
      hostNetworkSet: warning
      notReadOnlyRootFilesystem: warning
      privilegeEscalationAllowed: error
      runAsRootAllowed: warning
      runAsPrivileged: error
      cpuRequestsMissing: warning
      cpuLimitsMissing: warning
      memoryRequestsMissing: warning
      memoryLimitsMissing: warning
      tagNotSpecified: error
      latestTagUsed: error
      pullPolicyNotAlways: warning

---
apiVersion: batch/v1
kind: CronJob
metadata:
  name: compliance-report
  namespace: security
spec:
  schedule: "0 6 * * 1"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: reporter
              image: aquasec/kube-bench:latest
              command: [sh, -c]
              args:
                - |
                  kube-bench --json > /tmp/cis-results.json
                  PASS=$(jq '[.[] | .tests[].results[] | select(.status=="PASS")] | length' /tmp/cis-results.json)
                  FAIL=$(jq '[.[] | .tests[].results[] | select(.status=="FAIL")] | length' /tmp/cis-results.json)
                  echo "CIS Results: PASS=$PASS FAIL=$FAIL"
                  aws s3 cp /tmp/cis-results.json \
                    s3://compliance-reports/weekly/$(date +%Y-%m-%d).json
          restartPolicy: OnFailure
```

---

## Step 599: Compliance Drift Detection

```yaml
apiVersion: config.gatekeeper.sh/v1alpha1
kind: Config
metadata:
  name: config
  namespace: gatekeeper-system
spec:
  sync:
    syncOnly:
      - group: ""
        version: "v1"
        kind: "Namespace"
      - group: ""
        version: "v1"
        kind: "Pod"
      - group: "apps"
        version: "v1"
        kind: "Deployment"
      - group: "rbac.authorization.k8s.io"
        version: "v1"
        kind: "ClusterRoleBinding"

---
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: compliance-score
  namespace: monitoring
spec:
  groups:
    - name: compliance
      rules:
        - record: compliance:violations_total
          expr: sum(gatekeeper_violations_total)
        
        - record: compliance:policy_pass_rate
          expr: |
            1 - (sum(gatekeeper_violations_total) /
                 sum(gatekeeper_audit_last_run_total))
        
        - alert: ComplianceScoreDegraded
          expr: compliance:policy_pass_rate < 0.95
          for: 15m
          annotations:
            summary: "Compliance score below 95% - review violations"
            description: "Current compliance rate: {{ $value | humanizePercentage }}"
          labels:
            severity: warning
```

---

## Step 600: Workshop - Compliance Dashboard

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: compliance-status
  namespace: security
data:
  cis-1.8.yaml: |
    last_run: 2024-01-15
    score: 87%
    critical_failures:
      - "1.2.1: API server anonymous auth enabled"
      - "4.2.6: kubelet protect-kernel-defaults not set"
    remediation_in_progress:
      - "1.2.1: Disabling anonymous auth - PR #245"
  
  soc2.yaml: |
    audit_period: Q4 2023 - Q3 2024
    controls_tested: 47
    controls_passed: 45
    exceptions:
      - control: "CC6.2"
        description: "MFA not enforced for break-glass accounts"
        remediation: "Configure OIDC MFA for all accounts by Q1 2024"
  
  pci-dss.yaml: |
    scope: cardholder data environment
    last_qsa_assessment: 2023-11-01
    next_assessment: 2024-11-01
    open_findings:
      - requirement: "10.5.1"
        description: "Log retention policy documented but not automated"
        due_date: "2024-02-01"

---
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: compliance-sla
  namespace: monitoring
spec:
  groups:
    - name: compliance-sla
      rules:
        - alert: CISCriticalFindingUnremediated
          expr: |
            (time() - compliance_finding_opened_timestamp{severity="critical"}) > 604800
          annotations:
            summary: "Critical CIS finding open more than 7 days (SLA breach)"
          labels:
            severity: critical
```

---

## 📊 สรุป Part 63

| Framework | Key Tool | Automation |
|-----------|----------|------------|
| CIS Benchmark | kube-bench | Weekly CronJob |
| NIST 800-190 | Kyverno policies | Admission control |
| SOC2 | Prometheus/Loki | Continuous monitoring |
| PCI DSS | Audit policy | Quarterly report |
| General | OPA/Polaris | Drift detection |

---

## 🔗 ต่อไป
- [Part 64: DevSecOps Pipeline](./part-64-devsecops.md)

---
*Part 63 | Steps 591-600 | ระดับผู้เชี่ยวชาญ | Educational Use Only*
