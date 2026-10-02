# Part 100: Cloud Provider Integrations (EKS, GKE, AKS)
## Steps 961-970: AWS EKS, Google GKE, Azure AKS Best Practices

---

## Step 961: Cloud Kubernetes Overview

```
Managed Kubernetes Comparison:

AWS EKS (Elastic Kubernetes Service):
  Control plane: AWS-managed, HA by default
  Worker nodes: EC2, Fargate, or Bottlerocket
  Networking: VPC CNI (aws-node) or Cilium
  Storage: EBS CSI, EFS CSI
  IAM: IRSA (IAM Roles for Service Accounts)
  Load Balancer: AWS Load Balancer Controller
  Addons: EKS Addons (CoreDNS, kube-proxy, VPC CNI)
  Cost: $0.10/hour per cluster + node costs
  
Google GKE (Google Kubernetes Engine):
  Control plane: Google-managed
  Worker nodes: Standard or Autopilot (serverless nodes)
  Networking: VPC-native (Alias IPs) or Dataplane V2 (eBPF)
  Storage: GCP Persistent Disk, Filestore
  IAM: Workload Identity (GCP SA bound to K8s SA)
  Load Balancer: GKE Ingress (GCLB), Gateway API
  Autopilot: fully managed node pools
  Cost: $0.10/hour per cluster + node costs

Azure AKS (Azure Kubernetes Service):
  Control plane: Azure-managed, free
  Worker nodes: VM ScaleSets
  Networking: Azure CNI, kubenet, or Cilium
  Storage: Azure Disk, Azure Files, Azure Blob CSI
  IAM: Workload Identity (pod-level AAD identity)
  Load Balancer: AGIC (Application Gateway), nginx
  Cost: Free control plane + node costs

Common Best Practices (all clouds):
  - Use Spot/Preemptible nodes for non-critical workloads
  - Enable cluster autoscaler or Karpenter (AWS)
  - Use managed node groups for lifecycle management
  - Enable private cluster (no public endpoint)
  - Store kubeconfig in AWS Secrets / GCP Secret Manager
```

---

## Step 962: AWS EKS Setup

```yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: production-us-east
  region: us-east-1
  version: "1.29"
  tags:
    Environment: production
    Team: platform

iam:
  withOIDC: true

vpc:
  clusterEndpoints:
    privateAccess: true
    publicAccess: false
  subnets:
    private:
      us-east-1a:
        id: subnet-xxxxx
      us-east-1b:
        id: subnet-yyyyy

managedNodeGroups:
  - name: system
    instanceType: m5.large
    minSize: 3
    maxSize: 6
    desiredCapacity: 3
    labels:
      role: system
    taints:
      - key: CriticalAddonsOnly
        value: "true"
        effect: NoSchedule
  
  - name: workers
    instanceType: m5.2xlarge
    minSize: 3
    maxSize: 50
    desiredCapacity: 10
  
  - name: spot-workers
    instanceTypes: [m5.2xlarge, m5.4xlarge, m4.2xlarge]
    spot: true
    minSize: 0
    maxSize: 100
    labels:
      workload-type: batch
    taints:
      - key: spot
        value: "true"
        effect: NoSchedule

addons:
  - name: vpc-cni
    version: latest
  - name: coredns
    version: latest
  - name: aws-ebs-csi-driver
    version: latest
    wellKnownPolicies:
      ebsCSIController: true
```

---

## Step 963: IRSA (IAM Roles for Service Accounts)

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: payment-api
  namespace: production
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/PaymentAPIRole
    eks.amazonaws.com/token-expiration: "3600"

---
# Karpenter: next-gen node autoscaling
apiVersion: karpenter.sh/v1beta1
kind: NodePool
metadata:
  name: default
spec:
  template:
    spec:
      nodeClassRef:
        apiVersion: karpenter.k8s.aws/v1beta1
        kind: EC2NodeClass
        name: default
      requirements:
        - key: karpenter.sh/capacity-type
          operator: In
          values: [on-demand, spot]
        - key: kubernetes.io/arch
          operator: In
          values: [amd64]
        - key: node.kubernetes.io/instance-type
          operator: In
          values: [m5.large, m5.xlarge, m5.2xlarge, m5.4xlarge]
  limits:
    cpu: 1000
    memory: 4000Gi
  disruption:
    consolidationPolicy: WhenUnderutilized
    expireAfter: 720h
```

---

## Step 964: AWS Load Balancer Controller

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: payment-api-alb
  namespace: production
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/ssl-policy: ELBSecurityPolicy-TLS13-1-2-2021-06
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS":443}]'
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:us-east-1:123456789:certificate/xxx
    alb.ingress.kubernetes.io/wafv2-acl-arn: arn:aws:wafv2:us-east-1:123456789:regional/webacl/xxx
spec:
  ingressClassName: alb
  rules:
    - host: api.mycompany.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: payment-api
                port:
                  number: 80
```

---

## Step 965: Google GKE Setup

```yaml
apiVersion: container.cnrm.cloud.google.com/v1beta1
kind: ContainerCluster
metadata:
  name: production
  namespace: config-connector
spec:
  description: Production GKE cluster
  location: us-central1
  initialNodeCount: 1
  removeDefaultNodePool: true
  workloadIdentityConfig:
    workloadPool: myproject.svc.id.goog
  privateClusterConfig:
    enablePrivateNodes: true
    enablePrivateEndpoint: false
    masterIpv4CidrBlock: 172.16.0.0/28
  addonsConfig:
    gcePersistentDiskCsiDriverConfig:
      enabled: true
    horizontalPodAutoscaling:
      disabled: false
```

---

## Step 966: GKE Workload Identity

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: payment-api
  namespace: production
  annotations:
    iam.gke.io/gcp-service-account: payment-api-sa@myproject.iam.gserviceaccount.com

---
# GKE Gateway API
apiVersion: gateway.networking.k8s.io/v1beta1
kind: Gateway
metadata:
  name: external-https
  namespace: production
spec:
  gatewayClassName: gke-l7-global-external-managed
  listeners:
    - name: https
      port: 443
      protocol: HTTPS
      tls:
        mode: Terminate
        certificateRefs:
          - kind: Secret
            name: api-mycompany-tls

---
apiVersion: gateway.networking.k8s.io/v1beta1
kind: HTTPRoute
metadata:
  name: payment-api-route
  namespace: production
spec:
  parentRefs:
    - name: external-https
      sectionName: https
  hostnames:
    - api.mycompany.com
  rules:
    - backendRefs:
        - name: payment-api
          port: 80
```

---

## Step 967: Azure AKS Setup

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: payment-api
  namespace: production
  annotations:
    azure.workload.identity/client-id: "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
    azure.workload.identity/tenant-id: "yyyyyyyy-yyyy-yyyy-yyyy-yyyyyyyyyyyy"

---
# AKS: Application Gateway Ingress Controller
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: payment-api-agic
  namespace: production
  annotations:
    kubernetes.io/ingress.class: azure/application-gateway
    appgw.ingress.kubernetes.io/ssl-redirect: "true"
    appgw.ingress.kubernetes.io/backend-protocol: http
spec:
  tls:
    - hosts:
        - api.mycompany.com
      secretName: api-tls-secret
  rules:
    - host: api.mycompany.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: payment-api
                port:
                  number: 80
```

---

## Step 968: Cost Optimization (Spot/Preemptible)

```yaml
# Karpenter: prefer spot instances for batch
apiVersion: karpenter.sh/v1beta1
kind: NodePool
metadata:
  name: spot-batch
spec:
  template:
    spec:
      requirements:
        - key: karpenter.sh/capacity-type
          operator: In
          values: [spot]
      taints:
        - key: spot
          value: "true"
          effect: NoSchedule
  disruption:
    consolidationPolicy: WhenUnderutilized
    expireAfter: 168h

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: batch-processor
  namespace: production
spec:
  template:
    spec:
      tolerations:
        - key: spot
          operator: Equal
          value: "true"
          effect: NoSchedule
      affinity:
        nodeAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 1
              preference:
                matchExpressions:
                  - key: karpenter.sh/capacity-type
                    operator: In
                    values: [spot]
      containers:
        - name: processor
          image: myorg/batch-processor:1.0
```

---

## Step 969: Cloud-Agnostic Tools

```yaml
# External DNS: auto DNS for all clouds
apiVersion: apps/v1
kind: Deployment
metadata:
  name: external-dns
  namespace: kube-system
spec:
  template:
    spec:
      serviceAccountName: external-dns
      containers:
        - name: external-dns
          image: registry.k8s.io/external-dns/external-dns:v0.14.0
          args:
            - --source=service
            - --source=ingress
            - --domain-filter=mycompany.com
            - --provider=aws
            - --policy=sync
            - --registry=txt
            - --txt-owner-id=production-cluster

---
# cert-manager: auto TLS all clouds
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: platform@mycompany.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
      - dns01:
          route53:
            region: us-east-1
            hostedZoneID: ZXXXXXXXXXXXXX
```

---

## Step 970: Workshop - Cloud Provider Summary

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: cloud-provider-alerts
  namespace: monitoring
spec:
  groups:
    - name: cloud
      rules:
        - alert: NodeGroupScalingFailed
          expr: cluster_autoscaler_errors_total > 0
          for: 5m
          annotations:
            summary: "Cluster autoscaler error: {{ $labels.function }}"
          labels:
            severity: warning

        - alert: SpotInstanceTermination
          expr: node:node_termination_imminent == 1
          for: 0m
          annotations:
            summary: "Spot instance {{ $labels.node }} will be terminated soon"
          labels:
            severity: warning

        - alert: ExternalDNSSyncFailed
          expr: |
            external_dns_controller_last_sync_timestamp_seconds == 0
          for: 10m
          annotations:
            summary: "ExternalDNS has not synced successfully"
          labels:
            severity: warning
```

---

## 📊 สรุป Part 100

| Feature | AWS EKS | Google GKE | Azure AKS |
|---------|---------|-----------|----------|
| Control plane cost | $0.10/hr | $0.10/hr | Free |
| Serverless nodes | Fargate | Autopilot | Virtual Nodes |
| Node autoscaling | Karpenter/CA | Autopilot/CA | CA |
| IAM integration | IRSA | Workload Identity | Workload Identity |
| Ingress | ALB Controller | GKE Gateway | AGIC |
| CNI | VPC CNI/Cilium | VPC-native/Cilium | Azure CNI/Cilium |

---

## 🔗 ต่อไป
- [Part 101: GitOps Advanced Patterns](./part-101-gitops-advanced.md)

---
*Part 100 | Steps 961-970 | Cloud Provider Integrations | Educational Content*
