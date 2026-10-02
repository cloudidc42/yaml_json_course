# Part 42: Kubernetes API Server Internals
## Steps 401-410: Understanding the API Server

---

## 📖 บทนำ

Kubernetes API Server เป็น core component ที่รับ requests จาก kubectl, controllers, และ external clients จัดการ authentication, authorization, admission control และเก็บข้อมูลใน etcd

> **หมายเหตุ**: เนื้อหาด้าน security ในหลักสูตรนี้มีไว้เพื่อการศึกษาและการทดสอบระบบที่ได้รับอนุญาตเท่านั้น

---

## Step 401: API Server Architecture

```
Kubernetes API Server Request Flow:
┌─────────────────────────────────────────────────────────────┐
│                    kube-apiserver                           │
│                                                             │
│  HTTP/HTTPS Request                                         │
│        │                                                    │
│  ┌─────▼─────┐  ┌──────────────┐  ┌──────────────────┐   │
│  │ TLS        │  │ Authn        │  │ Authz            │   │
│  │ Termination│─►│ (x509, JWT,  │─►│ (RBAC, ABAC,     │   │
│  └────────────┘  │  OIDC, SA)   │  │  Webhook, Node)  │   │
│                  └──────────────┘  └──────────────────┘   │
│                                            │                │
│  ┌─────────────────────────────────────────────▼─────┐    │
│  │              Admission Control                      │    │
│  │  1. Mutating webhooks                               │    │
│  │  2. Object schema validation                        │    │
│  │  3. Validating webhooks                             │    │
│  └────────────────────────────────────────────────────┐   │
│                                                    │   │
│  ┌─────────────────────────────────────────────▼─────┐   │
│  │              etcd                                   │   │
│  └───────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

---

## Step 402: API Server Configuration

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: kube-apiserver
  namespace: kube-system
spec:
  containers:
    - name: kube-apiserver
      image: registry.k8s.io/kube-apiserver:v1.28.0
      command:
        - kube-apiserver
        - --etcd-servers=https://127.0.0.1:2379
        - --etcd-cafile=/etc/kubernetes/pki/etcd/ca.crt
        - --etcd-certfile=/etc/kubernetes/pki/apiserver-etcd-client.crt
        - --etcd-keyfile=/etc/kubernetes/pki/apiserver-etcd-client.key
        - --tls-cert-file=/etc/kubernetes/pki/apiserver.crt
        - --tls-private-key-file=/etc/kubernetes/pki/apiserver.key
        - --client-ca-file=/etc/kubernetes/pki/ca.crt
        - --service-account-key-file=/etc/kubernetes/pki/sa.pub
        - --service-account-signing-key-file=/etc/kubernetes/pki/sa.key
        - --service-account-issuer=https://kubernetes.default.svc.cluster.local
        - --oidc-issuer-url=https://accounts.google.com
        - --oidc-client-id=my-cluster
        - --oidc-username-claim=email
        - --oidc-groups-claim=groups
        - --authorization-mode=Node,RBAC
        - --enable-admission-plugins=NodeRestriction,PodSecurity
        - --admission-control-config-file=/etc/kubernetes/admission/config.yaml
        - --audit-log-path=/var/log/kubernetes/audit/audit.log
        - --audit-log-maxage=30
        - --audit-policy-file=/etc/kubernetes/audit/policy.yaml
        - --feature-gates=ValidatingAdmissionPolicy=true
        - --anonymous-auth=false
        - --insecure-port=0
        - --secure-port=6443
```

---

## Step 403: Authentication Methods

```yaml
# 1. kubeconfig สำหรับ user (X.509)
apiVersion: v1
kind: Config
users:
  - name: john
    user:
      client-certificate: /path/to/john.crt
      client-key: /path/to/john.key

---
# 2. ServiceAccount projected token
apiVersion: v1
kind: Pod
spec:
  volumes:
    - name: token
      projected:
        sources:
          - serviceAccountToken:
              path: token
              expirationSeconds: 3600
              audience: my-service
  containers:
    - name: app
      volumeMounts:
        - name: token
          mountPath: /var/run/secrets/tokens

---
# 3. Bootstrap Token (สำหรับ node join)
apiVersion: v1
kind: Secret
metadata:
  name: bootstrap-token-07401b
  namespace: kube-system
type: bootstrap.kubernetes.io/token
stringData:
  token-id: "07401b"
  token-secret: "f395accd246ae52d"
  usage-bootstrap-authentication: "true"
  auth-extra-groups: system:bootstrappers:worker
  expiration: "2024-12-31T00:00:00Z"
```

---

## Step 404: OIDC Authentication

```yaml
# Dex ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: dex
  namespace: dex
data:
  config.yaml: |
    issuer: https://dex.example.com
    
    storage:
      type: kubernetes
      config:
        inCluster: true
    
    connectors:
      - type: github
        id: github
        name: GitHub
        config:
          clientID: $GITHUB_CLIENT_ID
          clientSecret: $GITHUB_CLIENT_SECRET
          redirectURI: https://dex.example.com/callback
          orgs:
            - name: my-org
    
    staticClients:
      - id: kubernetes
        name: Kubernetes
        redirectURIs:
          - http://localhost:8000
        secret: my-secret

---
# kubeconfig สำหรับ OIDC
users:
  - name: oidc-user
    user:
      exec:
        apiVersion: client.authentication.k8s.io/v1beta1
        command: kubectl
        args:
          - oidc-login
          - get-token
          - --oidc-issuer-url=https://dex.example.com
          - --oidc-client-id=kubernetes
          - --oidc-client-secret=my-secret
```

---

## Step 405: Audit Policy

```yaml
apiVersion: audit.k8s.io/v1
kind: Policy
omitStages:
  - RequestReceived

rules:
  - level: Metadata
    resources:
      - group: ""
        resources: ["secrets"]
  
  - level: RequestResponse
    resources:
      - group: ""
        resources: ["pods/exec", "pods/attach", "pods/portforward"]
  
  - level: RequestResponse
    verbs: ["create", "update", "patch"]
    resources:
      - group: "rbac.authorization.k8s.io"
        resources: ["*"]
  
  - level: None
    users:
      - system:kube-proxy
      - system:apiserver
    verbs: ["watch", "list", "get"]
  
  - level: None
    nonResourceURLs:
      - "/healthz*"
      - "/metrics"
  
  - level: Metadata
    omitStages:
      - RequestReceived
```

---

## Step 406: Server-Side Apply

```yaml
# Server-Side Apply (SSA) - ติดตาม field ownership
# kubectl apply --server-side -f deployment.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  managedFields:
    - manager: kubectl
      operation: Apply
      apiVersion: apps/v1
      fieldsType: FieldsV1
      fieldsV1:
        f:spec:
          f:replicas: {}
          f:template:
            f:spec:
              f:containers:
                k:{"name":"app"}:
                  f:image: {}
    - manager: horizontal-pod-autoscaler
      operation: Update
      apiVersion: apps/v1
      fieldsV1:
        f:spec:
          f:replicas: {}
```

---

## Step 407: Informers

```go
// Kubernetes Informer pattern
func main() {
    config, _ := clientcmd.BuildConfigFromFlags("", "~/.kube/config")
    clientset, _ := kubernetes.NewForConfig(config)
    
    factory := informers.NewSharedInformerFactory(clientset, 0)
    podInformer := factory.Core().V1().Pods()
    
    podInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
        AddFunc: func(obj interface{}) {
            pod := obj.(*corev1.Pod)
            fmt.Printf("Pod added: %s/%s\n", pod.Namespace, pod.Name)
        },
        UpdateFunc: func(oldObj, newObj interface{}) {
            pod := newObj.(*corev1.Pod)
            fmt.Printf("Pod updated: %s/%s\n", pod.Namespace, pod.Name)
        },
        DeleteFunc: func(obj interface{}) {
            pod := obj.(*corev1.Pod)
            fmt.Printf("Pod deleted: %s/%s\n", pod.Namespace, pod.Name)
        },
    })
    
    ctx := context.Background()
    factory.Start(ctx.Done())
    factory.WaitForCacheSync(ctx.Done())
    
    // Lister ใช้ local cache (ไม่ hit API server)
    pods, _ := podInformer.Lister().Pods("production").List(nil)
    fmt.Printf("Pods in production: %d\n", len(pods))
    <-ctx.Done()
}
```

---

## Step 408: API Priority and Fairness

```yaml
apiVersion: flowcontrol.apiserver.k8s.io/v1beta3
kind: PriorityLevelConfiguration
metadata:
  name: custom-high-priority
spec:
  type: Limited
  limited:
    nominalConcurrencyShares: 50
    limitResponse:
      type: Queue
      queuing:
        queues: 64
        handSize: 6
        queueLengthLimit: 50

---
apiVersion: flowcontrol.apiserver.k8s.io/v1beta3
kind: FlowSchema
metadata:
  name: my-controller-flow
spec:
  priorityLevelConfiguration:
    name: custom-high-priority
  matchingPrecedence: 1000
  distinguisherMethod:
    type: ByUser
  rules:
    - subjects:
        - kind: ServiceAccount
          serviceAccount:
            name: my-controller
            namespace: kube-system
      resourceRules:
        - verbs: ["get", "list", "watch"]
          apiGroups: ["*"]
          resources: ["*"]
```

---

## Step 409-410: Security Hardening

```yaml
# 1. Encryption at rest
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
      - configmaps
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: <base64-32-byte-key>
      - identity: {}

---
# 2. Admission plugin config
apiVersion: apiserver.config.k8s.io/v1
kind: AdmissionConfiguration
plugins:
  - name: PodSecurity
    configuration:
      apiVersion: pod-security.admission.config.k8s.io/v1
      kind: PodSecurityConfiguration
      defaults:
        enforce: baseline
        enforce-version: latest
        audit: restricted
        warn: restricted
      exemptions:
        namespaces:
          - kube-system
          - monitoring
```

---

## 📊 สรุป Part 42

| Component | หน้าที่ |
|-----------|------|
| Authentication | ตรวจสอบ identity |
| Authorization | ตรวจสอบ permission |
| Admission Control | Validate/mutate requests |
| etcd | Persistent storage |
| Audit Log | บันทึก API activities |
| APF | Rate limit ด้วย priority |
| Encryption at Rest | เข้ารหัส secrets ใน etcd |

---

## 🔗 ต่อไป
- [Part 43: etcd Security](./part-43-etcd-security.md)

---
*Part 42 | Steps 401-410 | ระดับสูง*
