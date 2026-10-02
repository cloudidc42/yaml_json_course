# Part 62: Incident Response Playbook
## Steps 581-590: Kubernetes Security Incident Response

---

> **⚠️**: เนื้อหาด้าน security ในหลักสูตรนี้มีไว้เพื่อการศึกษาและการทดสอบระบบที่ได้รับอนุญาตเท่านั้น

## Step 581: Incident Response Framework

```
Kubernetes IR Framework (NIST SP 800-61):

Phase 1: Preparation
  - IR runbooks in Argo Workflows
  - Forensics toolkit ready

Phase 2: Detection & Analysis
  - Alert fired (Falco/Prometheus)
  - Triage: false positive?
  - Scope: which pods/namespaces affected?

Phase 3: Containment
  - Isolate pod (NetworkPolicy)
  - Revoke SA token
  - Block image from registry

Phase 4: Eradication
  - Remove malicious workloads
  - Patch vulnerability
  - Rotate compromised credentials

Phase 5: Recovery
  - Restore from clean backup
  - Verify system integrity
  - Monitor for recurrence

Phase 6: Post-Incident
  - Root cause analysis
  - Detection rule improvements
```

## Step 582: Runbook - Compromised Pod

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: compromised-pod-response-
  namespace: security
spec:
  arguments:
    parameters:
      - name: namespace
        value: "production"
      - name: pod
        value: ""
      - name: incident-id
        value: ""
  
  entrypoint: response-dag
  
  templates:
    - name: response-dag
      dag:
        tasks:
          - name: collect-evidence
            template: collect-evidence
            arguments:
              parameters:
                - name: namespace
                  value: "{{workflow.parameters.namespace}}"
                - name: pod
                  value: "{{workflow.parameters.pod}}"
                - name: incident-id
                  value: "{{workflow.parameters.incident-id}}"
          
          - name: notify-team
            template: slack-notify
            arguments:
              parameters:
                - name: message
                  value: ":rotating_light: INCIDENT {{workflow.parameters.incident-id}} - Compromised pod: {{workflow.parameters.namespace}}/{{workflow.parameters.pod}}"
          
          - name: isolate-pod
            template: isolate-pod
            dependencies: [collect-evidence]
            arguments:
              parameters:
                - name: namespace
                  value: "{{workflow.parameters.namespace}}"
                - name: pod
                  value: "{{workflow.parameters.pod}}"
    
    - name: collect-evidence
      inputs:
        parameters:
          - name: namespace
          - name: pod
          - name: incident-id
      container:
        image: bitnami/kubectl:latest
        command: [sh, -c]
        args:
          - |
            BUCKET="s3://incident-evidence/{{inputs.parameters.incident-id}}"
            NS="{{inputs.parameters.namespace}}"
            POD="{{inputs.parameters.pod}}"
            
            kubectl get pod $POD -n $NS -o yaml > /tmp/pod-spec.yaml
            kubectl logs $POD -n $NS > /tmp/pod-logs.txt
            kubectl describe pod $POD -n $NS > /tmp/pod-describe.txt
            kubectl get events -n $NS > /tmp/events.txt
            
            aws s3 cp /tmp/pod-spec.yaml $BUCKET/pod-spec.yaml
            aws s3 cp /tmp/pod-logs.txt $BUCKET/pod-logs.txt
            aws s3 cp /tmp/pod-describe.txt $BUCKET/pod-describe.txt
            aws s3 cp /tmp/events.txt $BUCKET/events.txt
            echo "Evidence collected to $BUCKET"
    
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
            NS="{{inputs.parameters.namespace}}"
            POD="{{inputs.parameters.pod}}"
            kubectl label pod $POD -n $NS security.status=compromised
            kubectl apply -f - <<EOF
            apiVersion: networking.k8s.io/v1
            kind: NetworkPolicy
            metadata:
              name: isolate-compromised
              namespace: $NS
            spec:
              podSelector:
                matchLabels:
                  security.status: compromised
              policyTypes: [Ingress, Egress]
            EOF
            echo "Pod $NS/$POD isolated"
    
    - name: slack-notify
      inputs:
        parameters:
          - name: message
      container:
        image: curlimages/curl:latest
        command: [sh, -c]
        args:
          - curl -X POST -H 'Content-type: application/json' --data '{"text":"{{inputs.parameters.message}}"}' $SLACK_WEBHOOK_URL
        env:
          - name: SLACK_WEBHOOK_URL
            valueFrom:
              secretKeyRef:
                name: slack-credentials
                key: webhook-url
```

## Step 583: Runbook - RBAC Escalation

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: rbac-escalation-response-
  namespace: security
spec:
  entrypoint: rbac-response
  templates:
    - name: rbac-response
      dag:
        tasks:
          - name: audit-rbac
            template: audit-rbac
          - name: revoke-suspicious-bindings
            template: revoke-suspicious-bindings
            dependencies: [audit-rbac]
    
    - name: audit-rbac
      container:
        image: bitnami/kubectl:latest
        command: [sh, -c]
        args:
          - |
            echo '=== Cluster-admin bindings ==='
            kubectl get clusterrolebindings -o json | \
              jq '.items[] | select(.roleRef.name == "cluster-admin") |
                {name: .metadata.name, created: .metadata.creationTimestamp}'
            echo '=== Recent RBAC changes (past 1h) ==='
            kubectl get clusterrolebindings --sort-by=.metadata.creationTimestamp | tail -10
```

## Step 584: Runbook - Cryptominer

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: cryptominer-response-
  namespace: security
spec:
  entrypoint: cryptominer-response
  templates:
    - name: cryptominer-response
      dag:
        tasks:
          - name: identify-miner
            template: identify-miner
          - name: kill-miner-pods
            template: kill-miner-pods
            dependencies: [identify-miner]
          - name: block-miner-image
            template: block-miner-image
            dependencies: [kill-miner-pods]
    
    - name: identify-miner
      container:
        image: bitnami/kubectl:latest
        command: [sh, -c]
        args:
          - |
            kubectl top pods --all-namespaces --sort-by=cpu | head -20
            kubectl get pods --all-namespaces -o json | \
              jq '.items[] | select(.spec.containers[].image | test("xmrig|minerd")) |
                {ns: .metadata.namespace, pod: .metadata.name}'
```

## Step 585: Runbook - Data Exfiltration

```yaml
# Emergency egress block + secret rotation
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: exfiltration-response-
  namespace: security
spec:
  entrypoint: exfil-response
  templates:
    - name: exfil-response
      dag:
        tasks:
          - name: block-egress
            template: emergency-egress-block
          - name: rotate-secrets
            template: rotate-all-secrets
            dependencies: [block-egress]
    
    - name: emergency-egress-block
      container:
        image: bitnami/kubectl:latest
        command: [sh, -c]
        args:
          - |
            kubectl apply -f - <<EOF
            apiVersion: networking.k8s.io/v1
            kind: NetworkPolicy
            metadata:
              name: emergency-egress-block
              namespace: production
            spec:
              podSelector: {}
              policyTypes: [Egress]
              egress:
                - ports:
                    - port: 53
                      protocol: UDP
            EOF
            echo "Emergency egress block applied"
    
    - name: rotate-all-secrets
      container:
        image: bitnami/kubectl:latest
        command: [sh, -c]
        args:
          - kubectl annotate externalsecrets -n production --all force-sync="$(date +%s)"
```

## Step 586: Post-Incident Analysis Template

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: post-incident-template
  namespace: security
data:
  template.md: |
    # Post-Incident Report
    ## Incident ID: {INCIDENT_ID}
    ## Severity: {CRITICAL/HIGH/MEDIUM/LOW}
    
    ## Timeline
    - {TIME}: Alert fired
    - {TIME}: Containment applied
    - {TIME}: Root cause identified
    - {TIME}: Incident closed
    
    ## Root Cause
    {ROOT_CAUSE}
    
    ## Action Items
    | Action | Owner | Due Date |
    |--------|-------|----------|
    | {ACTION} | {OWNER} | {DATE} |
    
    ## Detection Improvements
    - New Falco rules: {LIST}
    - New Prometheus alerts: {LIST}
    - New Kyverno policies: {LIST}
```

## Step 587: Automated Response with FalcoSidekick

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: falcosidekick-config
  namespace: falco
data:
  config.yaml: |
    pagerduty:
      routingkey: "${PD_ROUTING_KEY}"
      minimumpriority: "critical"
    
    slack:
      webhookurl: "${SLACK_WEBHOOK}"
      channel: "#security-incidents"
      minimumpriority: "warning"
      messageformat: |
        :rotating_light: *Falco Alert* - {{ .Priority }}
        *Rule*: {{ .Rule }}
        *Pod*: {{ .OutputFields.k8s_pod_name }}
        *Namespace*: {{ .OutputFields.k8s_ns_name }}
        *Message*: {{ .Output }}
    
    webhook:
      address: "http://argo-workflows-server.argo:2746/api/v1/workflows/security"
      minimumpriority: "critical"
```

## Step 588: IR Drill Template

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: ir-drill-results
  namespace: security
data:
  drill-results.yaml: |
    date: 2024-01-15
    participants: [sre-alice, sre-bob, security-charlie]
    scenarios:
      cryptominer:
        detection_time_seconds: 25
        notification_time_seconds: 40
        containment_time_seconds: 180
        targets: {detection: 30, notification: 60, containment: 300}
        pass: true
      secret_theft:
        detection_time_seconds: 10
        pass: true
    action_items:
      - improve documentation for containment steps
      - add auto-rotation on secret theft detection
```

## Step 589: IR SLA Monitoring

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: ir-sla
  namespace: monitoring
spec:
  groups:
    - name: ir-sla
      rules:
        - alert: CriticalIncidentNotContained
          expr: |
            (time() - falco_events_created_timestamp{priority="CRITICAL"}) > 300
          annotations:
            summary: "Critical security incident not contained within 5 minutes (SLA breach)"
          labels:
            severity: critical
        
        - alert: HighIncidentNotContained
          expr: |
            (time() - falco_events_created_timestamp{priority="HIGH"}) > 900
          annotations:
            summary: "High severity incident not contained within 15 minutes"
          labels:
            severity: warning
```

## Step 590: Workshop - IR Playbook Builder

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: ir-playbook-cryptominer
  namespace: security
data:
  playbook.yaml: |
    name: Cryptominer Detection Response
    trigger: Falco alert "Crypto Miner Process Detected"
    severity: CRITICAL
    
    steps:
      1_triage:
        actions:
          - Verify alert is not false positive
          - kubectl top pods -n <namespace>
          - Check image origin
        time_limit: 5m
      
      2_contain:
        actions:
          - Run: argo submit cryptominer-response.yaml
          - Verify pod isolated
          - Notify management
        time_limit: 10m
      
      3_eradicate:
        actions:
          - Delete compromised pod
          - Block image in Kyverno policy
          - Scan all pods for same image
          - Check for persistence (CronJobs, DaemonSets)
        time_limit: 30m
      
      4_recover:
        actions:
          - Restore service from clean deployment
          - Verify application health
          - Monitor 24h for recurrence
        time_limit: 60m
      
      5_post_incident:
        actions:
          - Write post-incident report
          - Update Falco detection rules
          - Review admission policies
        time_limit: 48h
```

---

## 📊 สรุป Part 62

| IR Phase | Actions | Tools |
|----------|---------|-------|
| Detection | Falco/Prometheus alert | Falco, Alertmanager |
| Triage | Evidence collection | kubectl, aws s3 |
| Containment | Network isolation | NetworkPolicy, kubectl |
| Eradication | Remove + patch | kubectl, Kyverno |
| Recovery | Clean restore | Argo Rollouts |
| Post-incident | RCA + improvements | Confluence, Jira |

---

## 🔗 ต่อไป
- [Part 63: Compliance - CIS/NIST/SOC2](./part-63-compliance.md)

---
*Part 62 | Steps 581-590 | ระดับผู้เชี่ยวชาญ | Educational Use Only*
