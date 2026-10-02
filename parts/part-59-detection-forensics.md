# Part 59: Detection Engineering & Forensics
## Steps 551-560: Kubernetes Security Detection & Digital Forensics

---

> **⚠️**: เนื้อหาด้าน security ในหลักสูตรนี้มีไว้เพื่อการศึกษาและการทดสอบระบบที่ได้รับอนุญาตเท่านั้น การนำไปใช้โจมตีระบบที่ไม่ได้รับอนุญาตเป็นสิ่งผิดกฎหมาย

## Step 551: Detection Architecture

```
Kubernetes Security Detection Stack:

Data Sources:
  kube-apiserver audit logs  -> who did what to which resource
  Container runtime (CRI)    -> process, file, network events
  Node syslogs               -> kernel events, auth logs
  Application logs           -> access logs, errors
  Network flow logs (CNI)    -> pod-to-pod communication

Detection Engines:
  Falco (runtime rules)
  Tetragon (eBPF)
  OPA/Kyverno (admission)
  Prometheus (metrics anomalies)

Storage & Analysis:
  Elasticsearch / Loki
  Grafana dashboards
  SIEM (Splunk, Sentinel)

Response:
  Alertmanager -> PagerDuty/Slack
  Argo Workflows (automated response)
```

## Step 552: Comprehensive Audit Policy

```yaml
apiVersion: audit.k8s.io/v1
kind: Policy
omitStages:
  - RequestReceived
rules:
  # Secrets: full request/response
  - level: RequestResponse
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
    resources:
      - group: ""
        resources: ["secrets"]
  
  # RBAC: full forensic trail
  - level: RequestResponse
    verbs: ["create", "update", "patch", "delete"]
    resources:
      - group: "rbac.authorization.k8s.io"
        resources:
          - clusterroles
          - clusterrolebindings
          - roles
          - rolebindings
  
  # Pod exec/attach
  - level: RequestResponse
    verbs: ["create"]
    resources:
      - group: ""
        resources:
          - pods/exec
          - pods/attach
          - pods/portforward
  
  # Webhooks
  - level: RequestResponse
    verbs: ["create", "update", "patch", "delete"]
    resources:
      - group: "admissionregistration.k8s.io"
        resources:
          - validatingwebhookconfigurations
          - mutatingwebhookconfigurations
  
  # Workloads
  - level: RequestResponse
    verbs: ["create", "update", "patch", "delete"]
    resources:
      - group: "apps"
        resources: ["deployments", "daemonsets", "statefulsets"]
      - group: "batch"
        resources: ["cronjobs", "jobs"]
  
  # Everything else
  - level: Metadata
```

## Step 553: Falco Detection Rules Library

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: falco-custom-rules
  namespace: falco
data:
  custom-rules.yaml: |
    # Crypto miner detection
    - rule: Crypto Miner Process Detected
      desc: Detect known crypto mining processes
      condition: >
        spawned_process and container and
        proc.name in (xmrig, minerd, cpuminer, ethminer, kswapd0)
      output: >
        Crypto miner detected (proc=%proc.name container=%container.id
        image=%container.image.repository)
      priority: CRITICAL
      tags: [cryptomining, malware]
    
    # Recon tools
    - rule: Container Running Recon Tool
      desc: Detect recon/scanning tools inside containers
      condition: >
        spawned_process and container and
        proc.name in (nmap, masscan, nikto, sqlmap, nuclei)
      output: >
        Recon tool in container (proc=%proc.name container=%container.id)
      priority: WARNING
      tags: [discovery, recon]
    
    # Exfiltration: bulk file deletion
    - rule: Container Deletes Multiple Files
      desc: Detect bulk file deletion (ransomware/wiper)
      condition: >
        container and
        evt.type = unlink and
        evt.count > 100
      output: >
        Bulk file deletion (container=%container.id count=%evt.count)
      priority: CRITICAL
      tags: [impact, ransomware]
```

## Step 554: SIEM Integration

```yaml
# Loki: aggregate security logs
apiVersion: v1
kind: ConfigMap
metadata:
  name: promtail-config
  namespace: monitoring
data:
  promtail.yaml: |
    server:
      http_listen_port: 9080
    
    clients:
      - url: http://loki:3100/loki/api/v1/push
    
    scrape_configs:
      - job_name: k8s-audit
        static_configs:
          - targets: [localhost]
            labels:
              job: k8s-audit
              __path__: /var/log/kubernetes/audit/*.log
        pipeline_stages:
          - json:
              expressions:
                verb: verb
                resource: objectRef.resource
                namespace: objectRef.namespace
                user: user.username
          - labels:
              verb:
              resource:
              namespace:
              user:
      
      - job_name: falco
        static_configs:
          - targets: [localhost]
            labels:
              job: falco
              __path__: /var/log/falco/*.log
        pipeline_stages:
          - json:
              expressions:
                priority: priority
                rule: rule
```

## Step 555: Digital Forensics - Container

```yaml
# Forensics Job: automated evidence collection
apiVersion: batch/v1
kind: Job
metadata:
  name: forensics-collect
  namespace: security
spec:
  template:
    spec:
      serviceAccountName: forensics-sa
      containers:
        - name: forensics
          image: gcr.io/security/forensics-toolkit:latest
          env:
            - name: TARGET_NAMESPACE
              value: "production"
            - name: TARGET_POD
              value: "suspicious-pod-xyz"
            - name: S3_BUCKET
              value: "incident-evidence-bucket"
          command:
            - /bin/sh
            - -c
            - |
              # Collect evidence without modifying target
              kubectl get pod $TARGET_POD -n $TARGET_NAMESPACE -o yaml > pod-spec.yaml
              kubectl logs $TARGET_POD -n $TARGET_NAMESPACE > pod-logs.txt
              kubectl logs $TARGET_POD -n $TARGET_NAMESPACE --previous > pod-logs-prev.txt 2>/dev/null
              
              # Upload to secure evidence bucket
              aws s3 cp pod-spec.yaml s3://$S3_BUCKET/incidents/$INCIDENT_ID/
              aws s3 cp pod-logs.txt s3://$S3_BUCKET/incidents/$INCIDENT_ID/
      restartPolicy: Never

# Manual forensics steps (run before killing compromised pod):
# 1. kubectl exec -n prod <pod> -- ls -la /proc/1/fd > open-files.txt
# 2. kubectl exec -n prod <pod> -- cat /proc/net/tcp > net-connections.txt
# 3. kubectl exec -n prod <pod> -- cat /proc/1/environ | tr '\0' '\n' > env.txt
# 4. kubectl cp -n prod <pod>:/ ./forensics-fs/
```

## Step 556: Audit Log Analysis Queries

```yaml
# Loki queries for common investigations:

# 1. All actions by suspicious user (past 24h):
# {job="k8s-audit"} | json | user_username = "suspicious-user"

# 2. All secrets accessed:
# {job="k8s-audit"} | json | resource = "secrets" | verb = "get"

# 3. Pod execs in production:
# {job="k8s-audit"} | json
#   | resource = "pods/exec"
#   | namespace = "production"

# 4. ClusterRoleBinding changes:
# {job="k8s-audit"} | json
#   | resource = "clusterrolebindings"
#   | verb =~ "create|update|delete"

# Elasticsearch index template
apiVersion: v1
kind: ConfigMap
metadata:
  name: es-audit-template
  namespace: logging
data:
  template.json: |
    {
      "index_patterns": ["k8s-audit-*"],
      "settings": {
        "number_of_shards": 3,
        "number_of_replicas": 1,
        "index.lifecycle.name": "30-day-policy"
      },
      "mappings": {
        "properties": {
          "@timestamp": {"type": "date"},
          "verb": {"type": "keyword"},
          "resource": {"type": "keyword"},
          "namespace": {"type": "keyword"},
          "user": {"type": "keyword"},
          "sourceIP": {"type": "ip"}
        }
      }
    }
```

## Step 557: Threat Intelligence Integration

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: falco-threat-intel
  namespace: falco
data:
  threat-intel-rules.yaml: |
    - list: known_c2_ips
      items:
        - 198.51.100.1
        - 203.0.113.50
    
    - rule: Connection to Known C2 IP
      desc: Detect outbound connection to known C2 infrastructure
      condition: >
        outbound and container and
        fd.rip in (known_c2_ips)
      output: >
        Connection to known C2 IP!
        (container=%container.id dest_ip=%fd.rip
        dest_port=%fd.rport proc=%proc.name)
      priority: CRITICAL
      tags: [c2, malware, threat_intel]
```

## Step 558: Automated Response

```yaml
# Argo Workflows: automated incident response
apiVersion: argoproj.io/v1alpha1
kind: WorkflowTemplate
metadata:
  name: auto-incident-response
  namespace: security
spec:
  templates:
    - name: respond-to-compromise
      inputs:
        parameters:
          - name: namespace
          - name: pod
          - name: severity
      dag:
        tasks:
          - name: collect-evidence
            template: collect-evidence
            arguments:
              parameters:
                - name: namespace
                  value: "{{inputs.parameters.namespace}}"
                - name: pod
                  value: "{{inputs.parameters.pod}}"
          
          - name: isolate-pod
            template: isolate-pod
            dependencies: [collect-evidence]
            when: "{{inputs.parameters.severity}} == 'critical'"
            arguments:
              parameters:
                - name: namespace
                  value: "{{inputs.parameters.namespace}}"
                - name: pod
                  value: "{{inputs.parameters.pod}}"
    
    - name: isolate-pod
      inputs:
        parameters:
          - name: namespace
          - name: pod
      container:
        image: bitnami/kubectl:latest
        command: [sh, -c]
        args:
          - |
            kubectl label pod {{inputs.parameters.pod}} \
              -n {{inputs.parameters.namespace}} \
              security.status=compromised
            
            kubectl apply -f - <<'POLICY'
            apiVersion: networking.k8s.io/v1
            kind: NetworkPolicy
            metadata:
              name: isolate-compromised
              namespace: {{inputs.parameters.namespace}}
            spec:
              podSelector:
                matchLabels:
                  security.status: compromised
              policyTypes:
                - Ingress
                - Egress
            POLICY
```

## Step 559: Grafana Security Dashboard

```yaml
# Grafana dashboard ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: security-dashboard
  namespace: monitoring
  labels:
    grafana_dashboard: "1"
data:
  security.json: |
    {
      "title": "Kubernetes Security Overview",
      "uid": "k8s-security",
      "panels": [
        {
          "title": "Critical Falco Events (24h)",
          "type": "stat",
          "fieldConfig": {
            "defaults": {"color": {"mode": "thresholds"},
              "thresholds": {"steps": [{"color": "green", "value": 0},
                {"color": "red", "value": 1}]}}
          },
          "targets": [{"expr": "sum(increase(falco_events_total{priority=\"CRITICAL\"}[24h]))"}]
        },
        {
          "title": "Pod Exec Sessions (rate)",
          "type": "timeseries",
          "targets": [{"expr": "sum(rate(apiserver_audit_event_total{verb=\"create\",resource=\"pods/exec\"}[5m]))"}]
        },
        {
          "title": "Secret Access Rate",
          "type": "timeseries",
          "targets": [{"expr": "sum by (namespace) (rate(apiserver_audit_event_total{verb=~\"get|list\",resource=\"secrets\"}[5m]))"}]
        }
      ]
    }
```

## Step 560: Workshop - Detection Rule Development

```yaml
# Detection coverage tracking
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: detection-coverage
  namespace: monitoring
spec:
  groups:
    - name: detection-quality
      rules:
        - record: detection:rules_firing:rate5m
          expr: |
            sum by (rule) (rate(falco_events_total[5m]))
        
        # Alert: Falco rule not firing in 24h (may indicate detection gap)
        - alert: DetectionRuleNotFiring
          expr: |
            absent(falco_events_total{
              rule=~"Service Account Token Read|Kubernetes Client Tool.*"
            })
          for: 24h
          annotations:
            summary: "Key detection rule has not fired - verify Falco is working"
          labels:
            severity: warning

# MITRE ATT&CK coverage matrix:
# TA0001 Initial Access:     Falco image rules
# TA0002 Execution:          Falco process rules, VAP
# TA0003 Persistence:        Falco RBAC/CronJob rules
# TA0004 Privilege Escalation: RBAC audit, Kyverno
# TA0005 Defense Evasion:    Falco log clearing rules
# TA0006 Credential Access:  Falco SA token rules
# TA0007 Discovery:          Falco recon tool rules
# TA0008 Lateral Movement:   NetworkPolicy, mTLS
# TA0009 Collection:         Falco file access rules
# TA0010 Exfiltration:       Egress gateway, Falco
# TA0040 Impact:             Falco file deletion rules
```

---

## 📊 สรุป Part 59

| Detection Layer | Tool | Coverage |
|----------------|------|----------|
| Admission | Kyverno/OPA/VAP | Policy violations |
| Runtime | Falco | Process, file, network |
| eBPF | Tetragon | Kernel-level events |
| Audit logs | Loki/Elasticsearch | API server actions |
| Network | Hubble/Cilium | East-west flows |
| Metrics | Prometheus | Anomaly detection |

---

## 🔗 ต่อไป
- [Part 60: Cloud Security EKS/GKE/AKS](./part-60-cloud-security.md)

---
*Part 59 | Steps 551-560 | ระดับผู้เชี่ยวชาญ | Educational Use Only*
