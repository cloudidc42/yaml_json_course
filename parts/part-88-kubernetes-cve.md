# Part 88: Kubernetes CVE Analysis & Exploitation
## Steps 871-880: การวิเคราะห์และทดสอบ CVE

---

> ⚠️ **คำเตือนสำคัญ**: เนื้อหานี้มีไว้เพื่อการศึกษา การป้องกัน และการทดสอบใน authorized environments เท่านั้น

---

## 📖 บทนำ

การศึกษา CVE (Common Vulnerabilities and Exposures) ใน Kubernetes ช่วยให้:
- เข้าใจ attack vectors จริงๆ
- Patch และ upgrade อย่างถูกต้องตามลำดับความสำคัญ
- ป้องกันระบบก่อนที่ CVE จะถูก exploit

---

## Step 871: CVE Research Methodology

### แหล่งข้อมูล CVE
```
NVD (National Vulnerability Database): https://nvd.nist.gov
CVE Mitre: https://cve.mitre.org
Kubernetes Security Advisories: https://kubernetes.io/docs/reference/issues-security/security/
GitHub Security Advisories: https://github.com/advisories

CVSS Score:
0.0     → None
0.1-3.9 → Low
4.0-6.9 → Medium
7.0-8.9 → High
9.0-10.0 → Critical
```

### ตรวจสอบ Kubernetes Version
```bash
kubectl version --short

# ตรวจสอบ CVEs ด้วย Trivy
trivy kubernetes --scanners vuln cluster

# ด้วย kube-bench
kubectl apply -f https://raw.githubusercontent.com/aquasecurity/kube-bench/main/job.yaml
kubectl logs job/kube-bench -n default

kubectl version -o json | jq '.serverVersion'
```

---

## Step 872: CVE-2018-1002105 - Privilege Escalation

### Overview
```
CVE: CVE-2018-1002105
CVSS: 9.8 (Critical)
Kubernetes: < 1.10.11, < 1.11.5, < 1.12.3
Description: Unauthorized API access via aggregated API server

เนื่องจาก bug ใน API proxy, attacker สามารถส่ง requests ไปยัง
 backend APIs ผ่าน Kubernetes API server แม้ไม่มีสิทธิ์
```

### วิธี Detect
```bash
kubectl version --short
# ถ้า < 1.10.11, < 1.11.5, < 1.12.3 → vulnerable

# ตรวจสอบ aggregated API servers
kubectl get apiservices | grep -v kube-system
kubectl get apiservices -o json | jq '.items[] | select(.status.conditions[].type == "Available")'
```

### Mitigation
```bash
# อัพเกรด Kubernetes เป็น version ที่ปลอดภัย
# 1.10.11+, 1.11.5+, 1.12.3+

kubectl delete apiservice <name>  # ลบ APIService ที่ไม่จำเป็น

# Monitor ด้วย audit logs
kubectl get events --all-namespaces | grep "aggregated"
```

---

## Step 873: CVE-2019-11247 - RBAC Bypass

### Overview
```
CVE: CVE-2019-11247
CVSS: 8.1 (High)
Kubernetes: 1.13.x before 1.13.9, 1.14.x before 1.14.5, 1.15.x before 1.15.2
Description: Users with access to cluster-scoped resources can bypass namespace restrictions
```

### Mitigation
```bash
# อัพเกรด Kubernetes
# ตรวจสอบ RBAC configurations
kubectl get clusterrolebindings -o yaml | grep -A5 "kind: ClusterRoleBinding"

# Audit ด้วย script
kubectl get clusterroles -o json | jq '
  .items[] |
  select(.rules[]?.verbs[]? | contains("*")) |
  {name: .metadata.name, rules: .rules}'
```

---

## Step 874: CVE-2021-25741 - Symlink Attacks

### Overview
```
CVE: CVE-2021-25741  
CVSS: 8.8 (High)
Kubernetes: all versions before 1.19.15, 1.20.x before 1.20.11, 1.21.x before 1.21.5, 1.22.x before 1.22.2
Description: Symlink Exchange Race Condition

Pod สามารถอ่านไฟล์ sensitive จาก host filesystem ผ่าน
subPath volume mount ที่มี symlink
```

### Detection and Mitigation
```bash
# ตรวจหา pods ที่ใช้ subPath
kubectl get pods -A -o json | jq '
  .items[] |
  select(.spec.containers[].volumeMounts[]? | .subPath != null) |
  {name: .metadata.name, namespace: .metadata.namespace}'

# อัพเกรด Kubernetes
# ใช้ OPA/Gatekeeper ป้องกัน subPath usage ที่อันตราย

# Gatekeeper policy
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sDenySubPath
metadata:
  name: deny-subpath
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
```

---

## Step 875: CVE-2022-3294 - Node Address Override

### Overview
```
CVE: CVE-2022-3294
CVSS: 8.8 (High)
Kubernetes: < 1.25.4, < 1.24.8, < 1.23.14, < 1.22.16
Description: Node authorizer allows overriding node addresses
```

### Mitigation
```bash
# Enable --node-restriction admission plugin
# kube-apiserver configuration
# --enable-admission-plugins=...,NodeRestriction,...

# ตรวจสอบว่า NodeRestriction เปิดอยู่
kubectl describe pod kube-apiserver -n kube-system | grep "enable-admission-plugins"
```

---

## Step 876: CVE-2023-5528 - Windows Node Privilege Escalation

```bash
# ตรวจสอบว่ามี Windows nodes
kubectl get nodes -o json | jq '.items[] | select(.status.nodeInfo.operatingSystem == "windows") | .metadata.name'

# ถ้ามี Windows nodes → update immediately
```

---

## Step 877: Log4Shell ใน Kubernetes Context

### Overview  
```
CVE-2021-44228 (Log4Shell)
CVSS: 10.0 (Critical)
Log4j: 2.0-2.14.1
Description: Remote Code Execution via JNDI injection

ผลกระทบในระบบ K8s:
- Apps ใน pods ที่ใช้ Log4j → RCE
- อาจ escape container
- อาจขโมย service account tokens
```

### Detection ใน Kubernetes
```bash
# ค้นหา pods ที่อาจใช้ Java/Log4j
kubectl get pods -A -o json | jq '
  .items[] |
  {name: .metadata.name, 
   namespace: .metadata.namespace,
   images: [.spec.containers[].image]}'

# Scan images ด้วย Trivy
trivy image --scanners vuln <image-name> | grep -i log4

# ตรวจสอบ apps Java
kubectl get deployments -A -o json | jq '
  .items[] |
  select(.spec.template.spec.containers[].image | test("java|elasticsearch|kafka|solr|logstash")) |
  {name: .metadata.name, namespace: .metadata.namespace}'
```

### Mitigation
```yaml
# Kubernetes NetworkPolicy เพื่อป้องกัน outbound JNDI connections
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-external-jndi
  namespace: affected-namespace
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    # Allow only internal cluster traffic
    - to:
        - podSelector: {}
    # Block external LDAP/DNS callbacks
    # (LDAP port 389, 636, 1389 → block external)
```

---

## Step 878: Container Escape via runc (CVE-2019-5736)

### Overview
```
CVE: CVE-2019-5736
CVSS: 8.6 (High)
runc: < 1.0-rc6
Description: runc overwrite vulnerability

Attacker ภายใน malicious container สามารถ overwrite
host runc binary และ gain root access บน host
```

### Detection
```bash
# ตรวจสอบ runc version
docker info | grep -i runc
sudo runc --version

# ตรวจสอบ container runtime
kubectl get nodes -o json | jq '.items[].status.nodeInfo.containerRuntimeVersion'
```

### Mitigation
```bash
# 1. อัพเกรด runc เป็น >= 1.0-rc6
apt-get update && apt-get install -y runc

# 2. Enable Seccomp ใน pod spec:
# securityContext:
#   seccompProfile:
#     type: RuntimeDefault

# 3. Enable AppArmor:
# annotations:
#   container.apparmor.security.beta.kubernetes.io/<container>: runtime/default
```

---

## Step 879: Kubernetes Secrets Exposure CVEs

### CVE-2022-1471 - SnakeYAML Constructor RCE
```
CVE: CVE-2022-1471
CVSS: 9.8 (Critical)
SnakeYAML: < 2.0
Description: Deserialization RCE via SnakeYAML Constructor

ผลกระทบต่อ Kubernetes:
- Tools ที่ใช้ SnakeYAML อาจ vulnerable
- ArgoCD, Flux, custom controllers
```

```bash
# ตรวจสอป tools ที่ใช้ Java + SnakeYAML
kubectl get pods -A -o json | jq '
  .items[] |
  select(.spec.containers[].image | test("argocd|flux|operator")) |
  {name: .metadata.name, ns: .metadata.namespace, 
   image: .spec.containers[0].image}'
```

### CVE-2023-2727 - Image Policy Bypass
```
CVE: CVE-2023-2727
CVSS: 6.5 (Medium)
Kubernetes: 1.24.0-1.24.14, 1.25.0-1.25.10, 1.26.0-1.26.5
Description: Bypass of policies enforcing image digest requirements
ผ่าน ephemeral containers สามารถ bypass imagePullPolicy
```

```bash
# ตรวจสอบ version
kubectl version --short

# ตรวจสอบ ephemeral containers
kubectl get pods -A -o json | jq '
  .items[] |
  select(.spec.ephemeralContainers != null) |
  {name: .metadata.name, ns: .metadata.namespace}'
```

---

## Step 880: CVE Monitoring และ Response

### Automated CVE Scanning
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: trivy-cluster-scan
  namespace: security
spec:
  schedule: "0 2 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: trivy-scanner
          containers:
            - name: trivy
              image: aquasec/trivy:latest
              command:
                - /bin/sh
                - -c
                - |
                  trivy kubernetes --report summary cluster \
                    --format json > /reports/$(date +%Y%m%d)-cluster-scan.json
                  
                  CRITICAL=$(cat /reports/$(date +%Y%m%d)-cluster-scan.json | \
                    jq '.Vulnerabilities[] | select(.Severity == "CRITICAL")' | wc -l)
                  
                  if [ "$CRITICAL" -gt "0" ]; then
                    echo "CRITICAL CVEs found: $CRITICAL"
                    exit 1
                  fi
              volumeMounts:
                - name: reports
                  mountPath: /reports
          volumes:
            - name: reports
              persistentVolumeClaim:
                claimName: security-reports
          restartPolicy: Never
```

### CVE Response Playbook
```bash
#!/bin/bash
# cve-response.sh

assess_impact() {
    local cve=$1
    echo "=== Assessing impact of $cve ==="
    
    K8S_VERSION=$(kubectl version --short | grep "Server Version" | awk '{print $3}')
    echo "Kubernetes version: $K8S_VERSION"
    
    kubectl get pods -n kube-system
    trivy kubernetes --scanners vuln cluster 2>/dev/null | grep "$cve" || echo "Not found by Trivy"
}

apply_mitigations() {
    local cve=$1
    
    case "$cve" in
        "CVE-2021-25741")
            echo "Applying subPath restrictions..."
            kubectl apply -f deny-subpath.yaml
            ;;
        "CVE-2022-3294")
            echo "Verifying NodeRestriction is enabled..."
            kubectl describe pod kube-apiserver-$(kubectl get nodes -o name | head -1 | cut -d/ -f2) \
                -n kube-system | grep "NodeRestriction"
            ;;
        *)
            echo "No automated mitigation for $cve - manual action required"
            ;;
    esac
}

main() {
    if [ -z "$1" ]; then
        echo "Usage: $0 <CVE-ID>"
        exit 1
    fi
    
    assess_impact "$1"
    apply_mitigations "$1"
    
    echo "Next steps:"
    echo "1. Review patch notes for $1"
    echo "2. Test in non-production"
    echo "3. Schedule upgrade window"
    echo "4. Apply patch"
}

main "$@"
```

---

## 📊 สรุป Part 88

| CVE | ประเภท | CVSS | Status |
|----|--------|------|--------|
| CVE-2018-1002105 | API Auth Bypass | 9.8 | Patched 1.10.11+ |
| CVE-2019-5736 | Container Escape | 8.6 | Patched (runc 1.0-rc6+) |
| CVE-2019-11247 | RBAC Bypass | 8.1 | Patched 1.13.9+ |
| CVE-2021-25741 | Symlink Escape | 8.8 | Patched 1.19.15+ |
| CVE-2021-44228 | Log4Shell | 10.0 | App-level patch |
| CVE-2022-3294 | Node Override | 8.8 | Patched 1.22.16+ |
| CVE-2022-1471 | SnakeYAML RCE | 9.8 | App-level patch |
| CVE-2023-2727 | Image Policy Bypass | 6.5 | Patched 1.24.15+ |

---

## 🛡️ CVE Prevention Best Practices
1. **Regular upgrades** - stay within 2 versions of latest
2. **Automated scanning** - Trivy in CI/CD
3. **SBOM** - Software Bill of Materials
4. **Monitoring** - subscribe to Kubernetes security advisories
5. **Patch quickly** - critical CVEs ควร patch ภายใน 72 ชั่วโมง

---
*Part 88 | Steps 871-880 | ระดับมืออาชีพ/โลก*
