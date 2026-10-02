# Part 66: Red Team / Blue Team Operations
## Steps 621-630: Kubernetes Adversarial Exercises

---

> **⚠️**: เนื้อหาด้าน security ในหลักสูตรนี้มีไว้เพื่อการศึกษาและการทดสอบระบบที่ได้รับอนุญาตเท่านั้น

## Step 621: Red Team / Blue Team Framework

```
Red Team / Blue Team in Kubernetes:

Red Team (Attacker simulation):
  Goal: Find gaps before real attackers do
  Scope: Authorized test environment only
  Methods: MITRE ATT&CK techniques
  Output: Findings report + remediation

Blue Team (Defender):
  Goal: Detect and respond to red team actions
  Tools: Falco, Prometheus, Audit logs, Loki
  Output: Detection rules, playbooks

Purple Team (Collaboration):
  Red + Blue work together
  Red demonstrates attack
  Blue improves detection
  Both improve overall security posture

Exercise Types:
  1. Tabletop: discussion-only
  2. CTF: structured challenges
  3. Red Team exercise: full adversary simulation
  4. Purple Team: collaborative detection improvement
  5. Chaos Engineering: reliability + security
```

---

## Step 622: CTF Challenge Setup (Authorized Lab)

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: ctf-lab
  labels:
    purpose: security-training
    pod-security.kubernetes.io/enforce: baseline

---
# CTF Challenge 1: Find the misconfigured RBAC
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: ctf-challenge-1
  namespace: ctf-lab
  labels:
    ctf.challenge: "1"
    ctf.description: "Find what is wrong with this RBAC"
rules:
  # VULNERABLE: wildcards on everything (for CTF)
  - apiGroups: ["*"]
    resources: ["*"]
    verbs: ["*"]

---
# CTF Challenge 2: Exposed secret in ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: ctf-challenge-2
  namespace: ctf-lab
  labels:
    ctf.challenge: "2"
data:
  app.properties: |
    database.host=postgres
    database.user=app
    database.password=CTF_FLAG_TRAINING_ONLY

---
# Solution: Kyverno policy that would catch it
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: ctf-solution-no-wildcard-rbac
spec:
  validationFailureAction: Audit
  rules:
    - name: no-wildcard-verbs
      match:
        any:
          - resources:
              kinds: [ClusterRole, Role]
      validate:
        message: "Wildcard verbs not allowed in RBAC"
        deny:
          conditions:
            any:
              - key: "{{ request.object.rules[].verbs[] | contains(@, '*') }}"
                operator: Equals
                value: true
```

---

## Step 623: Red Team - Reconnaissance Detection

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: falco-recon-detection
  namespace: falco
data:
  recon-rules.yaml: |
    - rule: Kubernetes Recon - List All Namespaces
      desc: Detect attempt to list all namespaces
      condition: >
        ka.verb = "list" and
        ka.target.resource = "namespaces" and
        not ka.user.name startswith "system:"
      output: >
        Namespace enumeration attempt
        (user=%ka.user.name src=%ka.source.ip)
      priority: WARNING
      source: k8s_audit
      tags: [recon, discovery]
    
    - rule: Kubernetes Recon - Secrets List
      desc: Detect attempt to list secrets
      condition: >
        ka.verb = "list" and
        ka.target.resource = "secrets" and
        not ka.user.name startswith "system:"
      output: >
        Secret enumeration attempt
        (user=%ka.user.name ns=%ka.target.namespace src=%ka.source.ip)
      priority: HIGH
      source: k8s_audit
      tags: [recon, credential-access]
    
    - rule: kubectl exec in Production
      desc: kubectl exec used in production namespace
      condition: >
        ka.verb = "create" and
        ka.target.subresource = "exec" and
        ka.target.namespace = "production"
      output: >
        kubectl exec in production
        (user=%ka.user.name pod=%ka.target.name)
      priority: CRITICAL
      source: k8s_audit
      tags: [execution]
```

---

## Step 624: Red Team - SA Token Exploitation Scenario

```yaml
# Detection for SA token abuse
apiVersion: v1
kind: ConfigMap
metadata:
  name: falco-sa-token-rules
  namespace: falco
data:
  sa-token.yaml: |
    - rule: Service Account Token Read
      desc: Process reads SA token file
      condition: >
        open_read and
        fd.name startswith "/var/run/secrets/kubernetes.io/serviceaccount/" and
        not proc.name in (java, python, node, ruby, go)
      output: >
        SA token read by unexpected process
        (proc=%proc.name file=%fd.name pod=%k8s.pod.name)
      priority: HIGH
      tags: [credential-access]
    
    - rule: Curl Kubernetes API from Pod
      desc: Detect curl to Kubernetes API from inside pod
      condition: >
        spawned_process and
        proc.name in (curl, wget) and
        proc.args contains "kubernetes.default"
      output: >
        Kubernetes API access attempt from pod
        (proc=%proc.name args=%proc.args pod=%k8s.pod.name)
      priority: CRITICAL
      tags: [discovery, credential-access]
```

---

## Step 625: Blue Team - Detection Playbook

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: blue-team-playbook
  namespace: security
data:
  detection-playbook.yaml: |
    alert: "Service Account Token Read"
    severity: HIGH
    
    immediate_triage:
      time_limit: 5m
      steps:
        1: "Is this a known application process?"
        2: "Check recent deployments"
        3: "Check pod's RBAC"
    
    if_confirmed_threat:
      time_limit: 15m
      steps:
        1: "Collect evidence: kubectl logs, describe pod"
        2: "Isolate: kubectl label pod security.status=compromised"
        3: "Apply isolation NetworkPolicy (zero egress)"
        4: "Revoke SA: kubectl delete serviceaccount"
        5: "Notify security team via PagerDuty"
    
    investigation:
      time_limit: 1h
      steps:
        1: "What data was the pod accessing before incident?"
        2: "Did the pod make any outbound API calls?"
        3: "Check audit log for SA token usage in Loki"
        4: "Scan all pods for same image"
    
    remediation:
      steps:
        1: "Add automountServiceAccountToken: false"
        2: "Add NetworkPolicy blocking API server access"
        3: "Update Falco rule to catch earlier"
        4: "Write post-incident report"

---
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: blue-team-metrics
  namespace: monitoring
spec:
  groups:
    - name: blue-team
      rules:
        - record: blue_team:mean_time_to_detect
          expr: avg(falco_event_to_alert_seconds)
        
        - alert: MTTDExceedsSLA
          expr: blue_team:mean_time_to_detect > 300
          annotations:
            summary: "Mean time to detect exceeds 5 min SLA"
          labels:
            severity: warning
```

---

## Step 626: Purple Team Exercise

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: purple-team-exercise
  namespace: security
data:
  exercise.yaml: |
    exercise_name: "Lateral Movement Detection Improvement"
    date: "2024-01-20"
    
    scenario:
      name: "Pod-to-pod lateral movement"
      attack_steps:
        1: "Get RCE in frontend pod (simulated)"
        2: "Run nmap from inside pod"
        3: "Connect to database service"
        4: "Attempt to read sensitive data"
      
      blue_team_starting_detection:
        - falco_rule: "Nmap in container"
        - prometheus_alert: "Unexpected database connections"
      
      exercise_outcome:
        detected_at_step: 2
        detection_time_seconds: 45
      
      improvements:
        - "Add Falco rule for port scanning tools"
        - "Add Cilium L7 policy between frontend and database"
        - "Add PrometheusRule for connection spike to db"
```

---

## Step 627: Kubernetes CTF Lab Architecture

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ctf-vulnerable-app
  namespace: ctf-lab
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ctf-vulnerable
  template:
    metadata:
      labels:
        app: ctf-vulnerable
    spec:
      automountServiceAccountToken: true
      serviceAccountName: ctf-sa
      containers:
        - name: app
          image: vulnerables/web-dvwa:latest
          ports:
            - containerPort: 80
      nodeSelector:
        ctf-node: "true"

---
apiVersion: v1
kind: ConfigMap
metadata:
  name: ctf-challenges
  namespace: ctf-lab
data:
  challenges.yaml: |
    challenges:
      - id: CTF-001
        title: "Find the over-permissive RBAC"
        points: 100
        hint: "Use kubectl auth can-i --list"
      
      - id: CTF-002
        title: "Extract the hardcoded credential"
        points: 150
        hint: "Check ConfigMaps in ctf-lab namespace"
      
      - id: CTF-003
        title: "Access the secret via SA token"
        points: 200
        hint: "SA tokens are mounted automatically here"
      
      - id: CTF-004
        title: "Write a Falco rule to detect CTF-003"
        points: 300
        hint: "Detect reading of SA token by unexpected process"
```

---

## Step 628: Chaos Engineering for Security

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: chaos-security-tests
  namespace: security
data:
  tests.yaml: |
    test_1_networkpolicy_under_node_failure:
      description: "Verify NetworkPolicy enforced even after node restart"
      steps:
        1: "Apply default-deny NetworkPolicy"
        2: "Verify pod-to-pod communication blocked"
        3: "Drain and reboot a node"
        4: "After node joins: verify NetworkPolicy still blocks traffic"
      pass_criteria: "Traffic still blocked after node rejoin"
    
    test_2_falco_agent_failure:
      description: "What happens when Falco agent crashes?"
      steps:
        1: "Kill Falco DaemonSet pod on one node"
        2: "Perform attack on that node's pods"
        3: "Verify: alert fires (or failsafe alert triggers)"
      pass_criteria: "Alert fires within 5 min"
      improvement: "Add Prometheus alert: FalcoAgentDown"
    
    test_3_networkpolicy_removal:
      description: "Verify GitOps restores NetworkPolicy if deleted"
      steps:
        1: "Delete production default-deny NetworkPolicy"
        2: "Verify ArgoCD restores within selfHeal timeout"
        3: "Verify alert fires for NetworkPolicy deletion"
      pass_criteria: "Restored within 30 seconds"
```

---

## Step 629: Security Metrics and KPIs

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: security-kpis
  namespace: monitoring
spec:
  groups:
    - name: security-kpis
      rules:
        - record: security:mttd_seconds
          expr: avg(falco_event_to_pagerduty_seconds)
        
        - record: security:mttc_seconds
          expr: avg(incident_to_networkpolicy_applied_seconds)
        
        - record: security:critical_cve_patch_days
          expr: |
            avg(
              (container_image_critical_cve_fixed_timestamp -
               container_image_critical_cve_found_timestamp)
              / 86400
            )
        
        - record: security:policy_compliance_ratio
          expr: |
            1 - (
              sum(gatekeeper_violations_total) /
              count(kube_pod_info{namespace="production"})
            )
        
        - alert: SecurityKPIDegraded
          expr: |
            security:policy_compliance_ratio < 0.99 or
            security:mttd_seconds > 300 or
            security:mttc_seconds > 900
          annotations:
            summary: "Security KPI below target"
          labels:
            severity: warning
```

---

## Step 630: Workshop - Red/Blue Team Exercise Plan

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: redblue-exercise-plan
  namespace: security
data:
  plan.yaml: |
    exercise: "Quarterly Red/Blue Team Exercise"
    quarter: Q1-2024
    
    red_team_scenarios:
      scenario_1:
        name: "Supply chain attack simulation"
        technique: T1195
        steps:
          - Deploy image with known CVE
          - Exploit vulnerability for RCE
          - Read SA token
          - Enumerate cluster
        time_limit: 2h
      
      scenario_2:
        name: "Insider threat simulation"
        technique: T1078
        steps:
          - Use legitimate developer credentials
          - Attempt privilege escalation via RBAC
          - Try to access production secrets
        time_limit: 2h
    
    blue_team_objectives:
      - Detect each red team action within SLA
      - Document detection gaps
      - Propose new detection rules
    
    success_criteria:
      - All scenarios detected within 15 minutes
      - No production impact
      - Post-exercise report within 5 days
    
    schedule:
      kickoff: "2024-01-20 09:00"
      red_team_active: "2024-01-20 10:00 - 16:00"
      debrief: "2024-01-20 16:00 - 17:00"
      report_due: "2024-01-25"
```

---

## 📊 สรุป Part 66

| Activity | Tool | Goal |
|----------|------|------|
| Red Team recon | kubectl, audit logs | Find gaps |
| Blue Team detection | Falco, Loki | Improve rules |
| Purple Team | Both | Collaborative improvement |
| CTF Lab | DVWA, kyverno | Training |
| Chaos Engineering | LitmusChaos | Resilience testing |

---

## 🔗 ต่อไป
- [Part 67: Security Hardening Automation](./part-67-hardening-automation.md)

---
*Part 66 | Steps 621-630 | ระดับผู้เชี่ยวชาญ | Educational Use Only*
