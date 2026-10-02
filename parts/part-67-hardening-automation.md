# Part 67: Security Hardening Automation
## Steps 631-640: Automated Security Posture Management

---

> **⚠️**: เนื้อหาด้าน security ในหลักสูตรนี้มีไว้เพื่อการศึกษาและการทดสอบระบบที่ได้รับอนุญาตเท่านั้น

## Step 631: Hardening Automation Overview

```
Security Hardening Automation Stack:

Infrastructure Level:
  - Terraform: secure defaults for EKS/GKE/AKS
  - Karpenter: secure node profiles
  - Ansible: node-level hardening

Kubernetes Level:
  - Kyverno: policy-as-code enforcement
  - OPA Gatekeeper: constraint templates
  - PSA (Pod Security Admission): namespace enforcement

Workload Level:
  - Helm security values templates
  - Kubescape: posture scoring
  - Starboard/Trivy Operator: continuous scanning

Continuous Assessment:
  - kube-bench: CIS benchmark
  - Polaris: workload best practices
  - Kubescape: NSA/CISA framework
```

---

## Step 632: Terraform - Secure EKS Defaults

```hcl
# terraform/modules/eks-secure/main.tf

module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 20.0"

  cluster_name    = var.cluster_name
  cluster_version = "1.29"

  # SECURITY: private endpoint only
  cluster_endpoint_public_access  = false
  cluster_endpoint_private_access = true

  # SECURITY: enable all control plane logging
  cluster_enabled_log_types = [
    "api", "audit", "authenticator",
    "controllerManager", "scheduler"
  ]

  # SECURITY: encryption for secrets
  cluster_encryption_config = {
    resources        = ["secrets"]
    provider_key_arn = aws_kms_key.eks.arn
  }

  enable_irsa = true

  eks_managed_node_groups = {
    main = {
      ami_type       = "BOTTLEROCKET_x86_64"
      instance_types = ["m5.large"]

      # IMDSv2 required, hop limit 1
      metadata_options = {
        http_endpoint               = "enabled"
        http_tokens                 = "required"
        http_put_response_hop_limit = 1
      }

      block_device_mappings = {
        xvda = {
          device_name = "/dev/xvda"
          ebs = {
            volume_size = 50
            volume_type = "gp3"
            encrypted   = true
            kms_key_id  = aws_kms_key.ebs.arn
          }
        }
      }
    }
  }

  tags = var.tags
}

resource "aws_kms_key" "eks" {
  description             = "EKS secrets encryption"
  deletion_window_in_days = 7
  enable_key_rotation     = true
  tags                    = var.tags
}

resource "aws_kms_key" "ebs" {
  description             = "EBS encryption for EKS nodes"
  deletion_window_in_days = 7
  enable_key_rotation     = true
  tags                    = var.tags
}
```

---

## Step 633: Kubescape - Automated Posture Scoring

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: kubescape-weekly
  namespace: security
spec:
  schedule: "0 3 * * 1"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: kubescape
              image: quay.io/kubescape/kubescape:latest
              command:
                - /bin/sh
                - -c
                - |
                  kubescape scan framework nsa \
                    --format json \
                    --output /tmp/nsa-results.json
                  
                  kubescape scan framework mitre \
                    --format json \
                    --output /tmp/mitre-results.json
                  
                  aws s3 cp /tmp/nsa-results.json \
                    s3://security-reports/kubescape/nsa-$(date +%Y-%m-%d).json
                  
                  SCORE=$(jq '.summaryDetails.frameworks[].score' /tmp/nsa-results.json)
                  echo "NSA Framework Score: $SCORE"
          restartPolicy: OnFailure
```

---

## Step 634: Trivy Operator - Continuous Image Scanning

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: trivy-alerts
  namespace: monitoring
spec:
  groups:
    - name: trivy
      rules:
        - alert: CriticalVulnerabilityFound
          expr: |
            sum by (namespace, pod, container, image) (
              trivy_image_vulnerabilities{
                severity="CRITICAL",
                namespace="production"
              }
            ) > 0
          for: 1h
          annotations:
            summary: "Critical vulnerability in {{ $labels.image }}"
            description: "Container {{ $labels.container }} in {{ $labels.pod }} has critical CVEs"
          labels:
            severity: critical
        
        - alert: HighVulnerabilityUnpatched
          expr: |
            sum by (namespace, pod) (
              trivy_image_vulnerabilities{
                severity="HIGH",
                namespace="production",
                fixed_version!=""
              }
            ) > 5
          for: 72h
          annotations:
            summary: "Multiple patchable HIGH vulnerabilities unaddressed for 3+ days"
          labels:
            severity: warning
```

---

## Step 635: Automated Hardening Pipeline

```yaml
# .github/workflows/hardening-check.yml
name: Hardening Check

on:
  pull_request:
    paths:
      - 'k8s/**'
      - 'helm/**'

jobs:
  kubescape:
    name: Kubescape Scan
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Kubescape scan
        uses: kubescape/github-action@main
        with:
          files: k8s/
          frameworks: NSA,MITRE
          severity-threshold: high
          fail-threshold: 10
  
  polaris:
    name: Polaris Check
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Run Polaris audit
        run: |
          polaris audit \
            --audit-path k8s/ \
            --format json \
            --output-file polaris-results.json
          
          SCORE=$(jq '.score' polaris-results.json)
          if (( $(echo "$SCORE < 80" | bc -l) )); then
            echo "Polaris score below 80 - failing"
            exit 1
          fi
```

---

## Step 636: Node Hardening via DaemonSet

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-hardening
  namespace: kube-system
spec:
  selector:
    matchLabels:
      app: node-hardening
  template:
    metadata:
      labels:
        app: node-hardening
    spec:
      hostPID: true
      hostNetwork: true
      initContainers:
        - name: harden
          image: alpine:3.19
          securityContext:
            privileged: true
          command:
            - /bin/sh
            - -c
            - |
              nsenter --mount=/host /bin/sh -c '
                sysctl -w net.ipv4.conf.all.send_redirects=0
                sysctl -w net.ipv4.conf.all.accept_redirects=0
                sysctl -w net.ipv4.icmp_echo_ignore_broadcasts=1
                sysctl -w kernel.randomize_va_space=2
              '
              echo "Node hardening complete"
          volumeMounts:
            - name: host-root
              mountPath: /host
      containers:
        - name: pause
          image: gcr.io/google_containers/pause:3.9
      volumes:
        - name: host-root
          hostPath:
            path: /
      tolerations:
        - operator: Exists
```

---

## Step 637: Automated RBAC Cleanup

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: rbac-cleanup
  namespace: security
spec:
  schedule: "0 1 * * 0"
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
                  echo "=== RBAC Audit Report ===" > /tmp/rbac-report.txt
                  echo "Date: $(date)" >> /tmp/rbac-report.txt
                  
                  echo "--- ClusterRoleBindings to cluster-admin ---" >> /tmp/rbac-report.txt
                  kubectl get clusterrolebindings -o json | \
                    jq -r '.items[] | select(.roleRef.name == "cluster-admin") |
                      "\(.metadata.name): \(.subjects[].name)"' >> /tmp/rbac-report.txt
                  
                  aws s3 cp /tmp/rbac-report.txt \
                    s3://security-reports/rbac/$(date +%Y-%m-%d).txt
                  cat /tmp/rbac-report.txt
          restartPolicy: OnFailure

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: rbac-reader
rules:
  - apiGroups: ["rbac.authorization.k8s.io"]
    resources: ["clusterrolebindings", "rolebindings", "clusterroles", "roles"]
    verbs: ["get", "list"]
  - apiGroups: [""]
    resources: ["serviceaccounts", "pods"]
    verbs: ["get", "list"]
```

---

## Step 638: Automated NetworkPolicy Generation

```yaml
# Generated NetworkPolicy from traffic observation
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: auto-generated-frontend
  namespace: production
  annotations:
    generated-by: network-observer
    generated-date: "2024-01-15"
    observed-period: "7-days"
spec:
  podSelector:
    matchLabels:
      app: frontend
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: ingress-nginx
      ports:
        - port: 8080
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: api
      ports:
        - port: 3000
    - ports:
        - port: 53
          protocol: UDP

---
apiVersion: batch/v1
kind: CronJob
metadata:
  name: networkpolicy-coverage
  namespace: security
spec:
  schedule: "0 4 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: checker
              image: bitnami/kubectl:latest
              command:
                - /bin/sh
                - -c
                - |
                  TOTAL=$(kubectl get pods -n production --no-headers | wc -l)
                  LABELED=$(kubectl get pods -n production -o json | \
                    jq '[.items[] | select(.metadata.labels | has("app"))] | length')
                  echo "Coverage: $LABELED / $TOTAL pods have app label for NetworkPolicy"
          restartPolicy: OnFailure
```

---

## Step 639: Security Scorecard Dashboard

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: security-score
  namespace: monitoring
spec:
  groups:
    - name: security-score
      rules:
        - record: security:overall_score
          expr: |
            (
              security:policy_compliance_ratio * 30 +
              (1 - clamp_max(devsecops:critical_cves_in_production, 1)) * 30 +
              devsecops:signed_image_ratio * 20 +
              (1 - clamp_max(security:mttd_seconds / 300, 1)) * 20
            )
        
        - alert: SecurityScoreBelow80
          expr: security:overall_score < 80
          for: 1h
          annotations:
            summary: "Overall security score below 80"
          labels:
            severity: warning
```

---

## Step 640: Workshop - Full Hardening Checklist

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: full-hardening-checklist
  namespace: security
data:
  hardening.yaml: |
    api_server:
      - anonymous_auth: disabled
      - audit_logging: enabled
      - encryption_config: enabled (AES-GCM + KMS)
      - authorization_mode: Node,RBAC
      - admission_controllers: [NodeRestriction, PSA]
    
    etcd:
      - client_cert_auth: enabled
      - tls configured
      - encryption: enabled
    
    kubelet:
      - anonymous_auth: disabled
      - read_only_port: 0
      - protect_kernel_defaults: true
      - authorization_mode: Webhook
    
    cluster_policies:
      - default_deny_networkpolicy: in all namespaces
      - psa_restricted: in production namespaces
      - kyverno_policies: 20+ security policies active
      - opa_constraints: applied
    
    supply_chain:
      - image_scanning: trivy in CI/CD
      - image_signing: cosign keyless
      - sbom: generated for all images
      - registry_allowlist: Kyverno enforced
    
    monitoring:
      - falco: running with custom rules
      - audit_logs: shipped to Loki (1 year retention)
      - prometheus_alerts: all critical rules active
      - incident_runbooks: tested quarterly
    
    compliance:
      - cis_benchmark: last_run < 30 days
      - kubescape_score: > 80%
      - polaris_score: > 80%
      - rbac_review: last_run < 90 days
```

---

## 📊 สรุป Part 67

| Area | Tool | Automation |
|------|------|------------|
| Posture Assessment | Kubescape, Polaris | Weekly CronJob |
| Image Scanning | Trivy Operator | Continuous |
| RBAC Audit | Custom CronJob | Weekly |
| Node Hardening | DaemonSet, Bottlerocket | At node launch |
| IaC Security | Checkov, Terraform | On PR |

---

## 🔗 ต่อไป
- [Part 68: CKS Exam Preparation](./part-68-cks-exam.md)

---
*Part 67 | Steps 631-640 | ระดับผู้เชี่ยวชาญ | Educational Use Only*
