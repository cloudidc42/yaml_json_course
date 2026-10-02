# Part 65: Threat Modeling for Kubernetes
## Steps 611-620: Systematic Threat Analysis

---

> **⚠️**: เนื้อหาด้าน security ในหลักสูตรนี้มีไว้เพื่อการศึกษาและการทดสอบระบบที่ได้รับอนุญาตเท่านั้น

## Step 611: Threat Modeling Frameworks

```
Threat Modeling for Kubernetes:

STRIDE Model:
  S - Spoofing: impersonate pod/service identity
  T - Tampering: modify container image/config
  R - Repudiation: deny malicious actions
  I - Information Disclosure: expose secrets/data
  D - Denial of Service: exhaust cluster resources
  E - Elevation of Privilege: escape container, gain admin

PASTA (Process for Attack Simulation and Threat Analysis):
  Stage 1: Define objectives (protect workload, data)
  Stage 2: Define technical scope (EKS, k8s 1.27)
  Stage 3: Application decomposition (DFD)
  Stage 4: Threat analysis (STRIDE)
  Stage 5: Vulnerability analysis (CVE scan)
  Stage 6: Attack modeling (attack trees)
  Stage 7: Risk analysis (CVSS scoring)

MITRE ATT&CK for Containers:
  Tactics: Initial Access, Execution, Persistence,
           Privilege Escalation, Defense Evasion,
           Credential Access, Discovery, Lateral Movement,
           Impact
```

---

## Step 612: Data Flow Diagram (DFD)

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: dfd-system-context
  namespace: security
data:
  description: |
    External Entities:
      [User Browser] --HTTPS--> [Ingress/WAF]
      [CI/CD System] --kubectl--> [API Server]
      [Admin] --kubectl--> [API Server]
    
    System (Kubernetes Cluster):
      [Ingress] -> [Frontend Pod]
      [Frontend Pod] -> [API Pod]
      [API Pod] -> [Database Pod]
      [API Pod] -> [Cache Pod]
    
    External Services:
      [API Pod] -> [External Payment API]
      [API Pod] -> [AWS S3] (IRSA)
    
    Trust Boundaries:
      TB1: Internet <-> Ingress (TLS termination)
      TB2: Ingress <-> Pod (mTLS via Istio)
      TB3: Pod <-> Pod (NetworkPolicy + mTLS)
      TB4: Pod <-> External (IRSA + TLS)
      TB5: Admin <-> API Server (RBAC + audit)
```

---

## Step 613: STRIDE Analysis - Kubernetes Components

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: stride-analysis
  namespace: security
data:
  api-server.yaml: |
    component: kube-apiserver
    threats:
      spoofing:
        threat: "Attacker impersonates legitimate user"
        attack: "Stolen kubeconfig or SA token"
        mitigation:
          - Short-lived tokens (1h expiry)
          - OIDC with MFA
          - Audit logging
        residual_risk: LOW
      
      tampering:
        threat: "Attacker modifies cluster state"
        attack: "Compromised admin account changes RBAC"
        mitigation:
          - Kyverno policies (audit ClusterRoleBinding)
          - GitOps (ArgoCD detects drift)
          - Alertmanager (RBAC change alert)
        residual_risk: MEDIUM
      
      information_disclosure:
        threat: "Secrets exposed via API"
        attack: "kubectl get secrets with over-permissive RBAC"
        mitigation:
          - EncryptionConfiguration (etcd encryption)
          - RBAC: no secrets list for developers
          - ExternalSecrets (never store in etcd)
        residual_risk: LOW
      
      elevation_of_privilege:
        threat: "Container escape to node"
        attack: "Privileged pod + hostPath mount"
        mitigation:
          - PSS restricted profile
          - Kyverno no-privileged
          - Falco (detect escape attempts)
        residual_risk: LOW
```

---

## Step 614: Attack Trees

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: attack-trees
  namespace: security
data:
  container-escape.yaml: |
    goal: "Escape container to node"
    
    OR:
      - "Privileged container escape"
        AND:
          - "Run privileged container" (blocked by PSS/Kyverno)
          - "Mount host filesystem" (blocked by Kyverno)
          - "Execute host commands"
        probability: LOW (mitigated)
      
      - "Kernel exploit escape"
        AND:
          - "Find kernel CVE"
          - "Execute exploit code in container"
        probability: LOW (seccomp + node patching)
      
      - "Misconfiguration escape"
        AND:
          - "hostPath: / mount"
          - "Write to /etc/cron.d"
        probability: MEDIUM if not using Kyverno
        mitigation: Kyverno restrict-hostpath
  
  credential-theft.yaml: |
    goal: "Steal cloud credentials"
    
    OR:
      - "SA token abuse"
        AND:
          - "Access SA token file"
          - "Use token against API server"
        mitigation: "automountServiceAccountToken: false"
      
      - "IMDS theft"
        AND:
          - "Access 169.254.169.254"
          - "Get node IAM role credentials"
        mitigation: "IMDSv2 hop limit=1 + NetworkPolicy"
      
      - "Secret from env var"
        AND:
          - "Read /proc/<pid>/environ"
          - "Parse credentials from env"
        mitigation: "Use volume mounts, not env vars for secrets"
```

---

## Step 615: MITRE ATT&CK Mapping

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mitre-attack-mapping
  namespace: security
data:
  techniques.yaml: |
    initial_access:
      T1190_exploit_public_application:
        description: "Exploit vulnerable app to get pod access"
        detection: "Falco: unexpected process/network"
        mitigation: "Trivy scan, WAF"
      
      T1078_valid_accounts:
        description: "Use stolen kubeconfig"
        detection: "Audit: unusual source IP"
        mitigation: "OIDC MFA, IP allowlist"
    
    persistence:
      T1525_implant_internal_image:
        description: "Modify container image in registry"
        detection: "Image digest verification"
        mitigation: "Cosign signing + Kyverno verify"
      
      T1053_scheduled_task:
        description: "Create malicious CronJob"
        detection: "Falco: Suspicious CronJob created"
        mitigation: "Kyverno restrict-cronjob"
    
    privilege_escalation:
      T1611_escape_to_host:
        description: "Container breakout"
        detection: "Falco: container escape attempt"
        mitigation: "PSS restricted + seccomp"
    
    lateral_movement:
      T1021_remote_services:
        description: "Access other pods via service"
        detection: "Cilium: L7 anomaly"
        mitigation: "mTLS + AuthorizationPolicy"
    
    exfiltration:
      T1041_exfil_c2:
        description: "Send data to C2 server"
        detection: "Falco: unexpected outbound"
        mitigation: "Egress NetworkPolicy + DNS filtering"
```

---

## Step 616: Risk Register

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: risk-register
  namespace: security
data:
  risks.yaml: |
    risks:
      - id: RISK-001
        title: "Container escape via privileged pod"
        category: "Privilege Escalation"
        likelihood: 2
        impact: 5
        risk_score: 10
        mitigations:
          - "PSS restricted profile"
          - "Kyverno no-privileged policy"
          - "Falco runtime detection"
        residual_score: 5
        owner: security-team
      
      - id: RISK-002
        title: "Secret exposure via over-permissive RBAC"
        category: "Information Disclosure"
        likelihood: 3
        impact: 4
        risk_score: 12
        mitigations:
          - "RBAC: developers cannot list secrets"
          - "ExternalSecrets (secrets not in etcd)"
          - "Audit logging on secret access"
        residual_score: 4
        owner: platform-team
      
      - id: RISK-003
        title: "Supply chain attack via compromised image"
        category: "Tampering"
        likelihood: 2
        impact: 5
        risk_score: 10
        mitigations:
          - "Cosign image signing"
          - "Kyverno verify-image-provenance"
          - "SLSA Level 3 provenance"
          - "Trivy scan in CI/CD"
        residual_score: 5
        owner: devsec-team
```

---

## Step 617: Threat Intelligence Integration

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: falco-threat-intel-rules
  namespace: falco
data:
  threat-intel.yaml: |
    - list: bad_ips
      items:
        - "185.220.101.42"
        - "185.220.102.42"
    
    - rule: Connection to Known Malicious IP
      desc: Detect connection to threat intel IP
      condition: >
        outbound and
        fd.sip in (bad_ips)
      output: >
        Connection to known malicious IP
        (proc=%proc.name ip=%fd.sip pod=%k8s.pod.name ns=%k8s.ns.name)
      priority: CRITICAL
      tags: [threat-intel, network]
    
    - list: bad_image_prefixes
      items:
        - "xmrig"
        - "nicehash"
        - "teamtnt"
    
    - rule: Known Malicious Container Image
      desc: Container image matches known malicious pattern
      condition: >
        container.image.repository pmatch (bad_image_prefixes)
      output: >
        Known malicious image started
        (image=%container.image.repository pod=%k8s.pod.name)
      priority: CRITICAL
      tags: [threat-intel, image]
```

---

## Step 618: Penetration Testing Scope

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: pentest-scope
  namespace: security
data:
  scope.yaml: |
    # Security testing scope - authorization required
    authorized_scope:
      environments:
        - name: staging
          cluster: staging-eks
          namespaces: [staging-app, staging-api]
          not_in_scope: [kube-system, monitoring]
      
      test_types:
        - rbac_enumeration
        - network_policy_bypass_attempt
        - image_pull_from_unauthorized_registry
        - resource_exhaustion_test
        - secret_access_attempt
      
      not_authorized:
        - production environment
        - etcd direct access
        - DoS attacks of any kind
        - Exfiltration of real customer data
      
      rules_of_engagement:
        - All tests logged in test-log.md
        - Stop immediately if production impact suspected
        - Findings reported within 24h
    
    contacts:
      security_lead: security@example.com
      emergency: ciso@example.com
```

---

## Step 619: Security Architecture Review

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: security-architecture-review
  namespace: security
data:
  checklist.yaml: |
    network_security:
      - Default deny NetworkPolicy in all namespaces
      - Egress restricted (no arbitrary internet access)
      - DNS filtering (CoreDNS blocklist)
      - mTLS for all service-to-service (Istio STRICT)
      - Ingress: WAF + rate limiting + TLS 1.3 only
    
    identity_and_access:
      - No cluster-admin for regular users
      - OIDC + MFA for all human access
      - IRSA/Workload Identity (no long-lived keys)
      - SA token: automount=false + short expiry
      - RBAC audit: quarterly review
    
    workload_security:
      - PSS restricted profile in production
      - All containers: non-root, readOnlyRootFilesystem
      - All containers: drop ALL capabilities
      - Resource limits on all containers
      - No hostNetwork/hostPID/hostIPC
    
    data_security:
      - etcd encrypted (AES-GCM + KMS v2)
      - Secrets via ESO/Vault (not hardcoded)
      - No secrets in ConfigMaps or env vars
    
    supply_chain:
      - Images signed (Cosign keyless)
      - SBOM generated for all images
      - Trivy scan in CI/CD (block on CRITICAL)
      - Base image: distroless or scratch
      - SLSA Level 2+ provenance
    
    monitoring:
      - Falco custom rules for environment
      - Audit policy: log RBAC + secrets + exec
      - Log retention: 1 year (compliance)
      - IR runbooks in Argo Workflows
```

---

## Step 620: Workshop - Threat Model Template

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: threat-model-template
  namespace: security
data:
  template.md: |
    # Threat Model: [Service Name]
    ## Date: [DATE]
    ## Author: [TEAM]
    
    ## 1. Service Description
    - Purpose: [what it does]
    - Data handled: [PII/financial/public]
    - External connections: [list APIs]
    
    ## 2. Trust Boundaries
    - TB1: Internet -> Ingress
    - TB2: Ingress -> Service
    - TB3: Service -> Database
    
    ## 3. STRIDE Analysis
    | Component | S | T | R | I | D | E |
    |-----------|---|---|---|---|---|---|
    | API endpoint | M | M | L | H | M | M |
    | Database | L | L | L | H | L | M |
    | SA token | H | L | L | M | L | H |
    
    ## 4. Top Risks
    1. [Risk description] - CVSS [score]
       Mitigation: [control]
    
    ## 5. Required Controls
    - NetworkPolicy
    - PSS restricted
    - Image signing
    - Secret management via ESO
    - Falco custom rule

---
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: threat-model-coverage
  namespace: monitoring
spec:
  groups:
    - name: threat-model
      rules:
        - alert: ServiceWithoutThreatModel
          expr: |
            count(kube_deployment_labels{namespace="production"}) >
            count(kube_configmap_labels{label_threat_model="true", namespace="security"})
          annotations:
            summary: "Some production services lack threat models"
          labels:
            severity: warning
```

---

## 📊 สรุป Part 65

| Framework | Use Case | Output |
|-----------|----------|--------|
| STRIDE | Component analysis | Threat list |
| PASTA | End-to-end process | Risk register |
| MITRE ATT&CK | Attack technique mapping | Detection rules |
| Attack Trees | Specific attack paths | Countermeasures |
| Risk Register | Prioritization | Remediation plan |

---

## 🔗 ต่อไป
- [Part 66: Red Team Operations](./part-66-red-team.md)

---
*Part 65 | Steps 611-620 | ระดับผู้เชี่ยวชาญ | Educational Use Only*
