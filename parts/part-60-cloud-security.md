# Part 60: Cloud Security - EKS/GKE/AKS
## Steps 561-570: Kubernetes Cloud Provider Security

---

> **⚠️**: เนื้อหาด้าน security ในหลักสูตรนี้มีไว้เพื่อการศึกษาและการทดสอบระบบที่ได้รับอนุญาตเท่านั้น การนำไปใช้โจมตีระบบที่ไม่ได้รับอนุญาตเป็นสิ่งผิดกฎหมาย

## Step 561: Cloud Kubernetes Security Overview

```
Cloud K8s Security Attack Surface:

EKS (AWS):
  IAM -> IRSA -> Pod permissions
  IMDS (169.254.169.254) -> node IAM role
  Public API endpoint

GKE (Google):
  Workload Identity -> GSA binding
  GCE metadata server (169.254.169.254)
  Binary Authorization

AKS (Azure):
  AAD Pod Identity / Workload Identity
  Azure RBAC integration
  ACR integration

Common Risks:
  Over-permissive pod IAM roles
  Public API server endpoint
  No audit logging
  Public container registry
```

## Step 562: EKS - IRSA Security

```yaml
# IRSA - least privilege
apiVersion: v1
kind: ServiceAccount
metadata:
  name: s3-reader
  namespace: production
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/s3-reader-role

---
# Block IMDS from pods (IMDSv2 + hop limit 1 on nodes)
# aws ec2 modify-instance-metadata-options \
#   --instance-id i-xxxx \
#   --http-put-response-hop-limit 1 \
#   --http-tokens required

---
# NetworkPolicy: block IMDS from pods
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: block-imds-access
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - to:
        - ipBlock:
            cidr: 0.0.0.0/0
            except:
              - 169.254.169.254/32
              - 169.254.170.2/32
    - ports:
        - port: 53
          protocol: UDP
```

## Step 563: EKS - aws-auth ConfigMap Security

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: aws-auth
  namespace: kube-system
data:
  mapRoles: |
    - rolearn: arn:aws:iam::123456789012:role/eks-node-role
      username: system:node:{{EC2PrivateDNSName}}
      groups:
        - system:bootstrappers
        - system:nodes
    
    - rolearn: arn:aws:iam::123456789012:role/sre-team-role
      username: sre-team
      groups:
        - sre-admins
    
    - rolearn: arn:aws:iam::123456789012:role/cicd-deploy-role
      username: cicd-system
      groups:
        - deployers
  
  mapUsers: |
    # Break-glass only
    - userarn: arn:aws:iam::123456789012:user/k8s-emergency-admin
      username: emergency-admin
      groups:
        - system:masters
```

## Step 564: GKE - Workload Identity

```yaml
# GKE Workload Identity
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-sa
  namespace: production
  annotations:
    iam.gke.io/gcp-service-account: app@project.iam.gserviceaccount.com

---
# Bind K8s SA to GCP SA:
# gcloud iam service-accounts add-iam-policy-binding \
#   app@project.iam.gserviceaccount.com \
#   --role="roles/iam.workloadIdentityUser" \
#   --member="serviceAccount:project.svc.id.goog[production/app-sa]"

# Block GCE metadata server
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: block-gce-metadata
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - to:
        - ipBlock:
            cidr: 0.0.0.0/0
            except:
              - 169.254.169.254/32
    - ports:
        - port: 53
          protocol: UDP
```

## Step 565: AKS - Workload Identity

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-sa
  namespace: production
  annotations:
    azure.workload.identity/client-id: "00000000-0000-0000-0000-000000000000"

---
# AKS: AAD Group ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: aad-sre-team
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: view
subjects:
  - apiGroup: rbac.authorization.k8s.io
    kind: Group
    name: "00000000-0000-0000-0000-aad-group-id"

---
# Block Azure IMDS
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: block-azure-imds
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - to:
        - ipBlock:
            cidr: 0.0.0.0/0
            except:
              - 169.254.169.254/32
    - ports:
        - port: 53
          protocol: UDP
```

## Step 566: Private Cluster Configuration

```yaml
# Kubelet configuration (secure defaults)
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
authentication:
  anonymous:
    enabled: false
  webhook:
    enabled: true
authorization:
  mode: Webhook
readOnlyPort: 0
protectKernelDefaults: true
tlsCipherSuites:
  - TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
  - TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256
streamingConnectionIdleTimeout: 5m
```

## Step 567: Node Security with Karpenter

```yaml
apiVersion: karpenter.sh/v1beta1
kind: NodePool
metadata:
  name: secure-nodepool
spec:
  template:
    spec:
      requirements:
        - key: kubernetes.io/arch
          operator: In
          values: [amd64]
        - key: karpenter.sh/capacity-type
          operator: In
          values: [on-demand]
      nodeClassRef:
        name: secure-nodeclass
  limits:
    cpu: 100

---
apiVersion: karpenter.k8s.aws/v1beta1
kind: EC2NodeClass
metadata:
  name: secure-nodeclass
spec:
  amiFamily: Bottlerocket
  role: KarpenterNodeRole
  subnetSelectorTerms:
    - tags:
        karpenter.sh/discovery: my-cluster
  securityGroupSelectorTerms:
    - tags:
        karpenter.sh/discovery: my-cluster
  metadataOptions:
    httpTokens: required
    httpPutResponseHopLimit: 1
  blockDeviceMappings:
    - deviceName: /dev/xvda
      ebs:
        volumeSize: 50Gi
        volumeType: gp3
        encrypted: true
        kmsKeyID: arn:aws:kms:us-east-1:123456789012:key/my-key
```

## Step 568: Container Registry Security

```yaml
# Kyverno: enforce ECR-only images
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-ecr-images
spec:
  validationFailureAction: Enforce
  rules:
    - name: check-ecr
      match:
        any:
          - resources:
              kinds: [Pod]
              namespaces: [production]
      validate:
        message: "Images must come from approved ECR registry"
        pattern:
          spec:
            containers:
              - image: "123456789012.dkr.ecr.us-east-1.amazonaws.com/*"
```

## Step 569: Cloud Audit Logging

```yaml
# PrometheusRule: cloud security alerts
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: cloud-security-alerts
  namespace: monitoring
spec:
  groups:
    - name: cloud-security
      rules:
        - alert: NodeIMDSAccessFromPod
          expr: |
            increase(falco_events_total{rule=~".*IMDS.*|.*metadata.*"}[5m]) > 0
          annotations:
            summary: "Pod accessing cloud IMDS - possible credential theft"
          labels:
            severity: critical
        
        - alert: PublicAPIServerDetected
          expr: |
            kube_cluster_info{endpoint_public_access="true"} > 0
          labels:
            severity: critical
          annotations:
            summary: "K8s API server has public endpoint exposed"
```

## Step 570: Workshop - Cloud Security Assessment

```yaml
# Cloud K8s security checklist:

# EKS:
# [x] API server endpoint: private only
# [x] aws-auth: no system:masters for regular users
# [x] IRSA: all pods use IRSA (not node IAM role)
# [x] IMDSv2: hop limit = 1 on all nodes
# [x] ECR: scanning enabled, immutable tags
# [x] CloudTrail: all K8s API calls logged
# [x] EKS Addons: updated to latest

# GKE:
# [x] Private cluster: enabled
# [x] Workload Identity: enabled
# [x] Binary Authorization: enforced
# [x] Shielded nodes: enabled

# AKS:
# [x] Private cluster: enabled
# [x] Workload Identity: enabled
# [x] Azure Defender for Containers: enabled
# [x] Network Policy: Calico or Azure

# Compliance scan with kube-bench (CIS Benchmark):
# kubectl run kube-bench --image=aquasec/kube-bench:latest \
#   --restart=Never -- node --version 1.27
```

---

## 📊 สรุป Part 60

| Cloud | Key Security Feature | Configuration |
|-------|---------------------|---------------|
| EKS | IRSA | SA annotation + IAM trust |
| EKS | Private endpoint | vpc_config block |
| GKE | Workload Identity | SA annotation + IAM binding |
| GKE | Binary Authorization | attestation policy |
| AKS | Workload Identity | federated credential |
| All | Block IMDS | NetworkPolicy + metadata options |

---

## 🔗 ต่อไป
- [Part 61: Zero Trust Architecture](./part-61-zero-trust.md)

---
*Part 60 | Steps 561-570 | ระดับผู้เชี่ยวชาญ | Educational Use Only*
