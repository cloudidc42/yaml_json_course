# Part 99: Testing, Validation, and Quality Assurance
## Steps 951-960: Testkube, k6, Kyverno Testing, Policy Validation, Chaos

---

## Step 951: Testing Strategy Overview

```
Kubernetes Testing Pyramid:

Unit Tests (fast, many):
  - Test individual YAML manifests (yamllint, kubeval)
  - Test Helm charts (helm lint, helm template | kubeval)
  - Test Kustomize (kustomize build | kubeval)
  - Test OPA/Kyverno policies (rego unit tests)

Integration Tests (medium):
  - Deploy to ephemeral namespace (kind, k3d)
  - Test Service connectivity
  - Test RBAC permissions
  - Test NetworkPolicy enforcement
  - Tool: BATS, Ginkgo, Terratest

End-to-End Tests (slow, few):
  - Full user journey in staging cluster
  - Test via actual HTTP endpoints
  - Tools: Playwright, Cypress, Selenium
  - Kubernetes-native: Testkube

Performance Tests:
  - Load testing: k6, Locust, Gatling
  - Stress testing: KEDA + Chaos Mesh combination
  - Tools: k6 Operator (run k6 inside K8s)

Tools:
  yamllint: YAML syntax + style
  kubeval/kubeconform: validate against K8s schema
  helm lint: Helm chart static analysis
  conftest: OPA policy testing for YAML
  kube-score: Kubernetes best practices checker
  Testkube: Kubernetes-native test execution
  k6: load testing (k6 Operator for K8s)
```

---

## Step 952: YAML Validation Pipeline

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: ci-validation-script
  namespace: kube-system
data:
  validate.sh: |
    #!/bin/bash
    set -euo pipefail
    
    echo "=== yamllint ==="
    yamllint -c .yamllint.yaml manifests/
    
    echo "=== kubeconform ==="
    kubeconform -strict -summary \
      -schema-location default \
      manifests/
    
    echo "=== helm lint ==="
    for chart in helm/*/; do
      helm lint "$chart"
    done
    
    echo "=== kube-score ==="
    kube-score score manifests/*.yaml
    
    echo "=== conftest ==="
    conftest test manifests/ --policy policy/
    
    echo "All validations passed!"
```

---

## Step 953: Helm Chart Testing

```yaml
# Helm test: run test pods after deployment
apiVersion: v1
kind: Pod
metadata:
  name: "{{ include \"myapp.fullname\" . }}-test-connection"
  annotations:
    "helm.sh/hook": test
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
spec:
  containers:
    - name: wget
      image: busybox:1.36
      command: ['wget']
      args: ['--tries=3', '{{ include "myapp.fullname" . }}:{{ .Values.service.port }}/health']
  restartPolicy: Never

---
apiVersion: v2
name: myapp
description: A Helm chart for myapp
type: application
version: 1.2.3
appVersion: "2.0"
maintainers:
  - name: platform-team
    email: platform@mycompany.com
dependencies:
  - name: postgresql
    version: "~12.0"
    repository: https://charts.bitnami.com/bitnami
    condition: postgresql.enabled
```

---

## Step 954: Testkube

```yaml
apiVersion: tests.testkube.io/v3
kind: Test
metadata:
  name: payment-api-load-test
  namespace: testkube
spec:
  type: k6/script
  content:
    type: git
    repository:
      type: git
      uri: https://github.com/myorg/load-tests
      branch: main
      path: tests/payment-api/load.js
  executionRequest:
    variables:
      TARGET_URL:
        value: https://api.mycompany.com
        type: basic
      VUS:
        value: "50"
        type: basic
    activeDeadlineSeconds: 600

---
apiVersion: tests.testkube.io/v2
kind: TestSuite
metadata:
  name: pre-production-gate
  namespace: testkube
spec:
  description: "Run before promoting to production"
  steps:
    - execute:
        - test: payment-api-smoke-test
    - execute:
        - test: payment-api-load-test
        - test: checkout-api-load-test
    - execute:
        - test: e2e-checkout-flow
  schedule: "0 6 * * 1-5"

---
apiVersion: tests.testkube.io/v1
kind: TestTrigger
metadata:
  name: post-deploy-trigger
  namespace: testkube
spec:
  resource: pod
  resourceSelector:
    labelSelector:
      matchLabels:
        app: payment-api
  event: modified
  conditionSpec:
    conditions:
      - type: Ready
        status: "True"
  action: run
  execution: test
  testSelector:
    name: payment-api-smoke-test
    namespace: testkube
```

---

## Step 955: k6 Load Testing

```yaml
apiVersion: k6.io/v1alpha1
kind: TestRun
metadata:
  name: payment-api-load-test
  namespace: k6-tests
spec:
  parallelism: 5
  script:
    configMap:
      name: payment-api-k6-script
      file: script.js
  runner:
    image: grafana/k6:0.50.0
    resources:
      requests:
        cpu: 500m
        memory: 512Mi
    env:
      - name: TARGET_URL
        value: https://api.mycompany.com
      - name: K6_PROMETHEUS_RW_SERVER_URL
        value: http://prometheus-pushgateway.monitoring:9091/api/v1/write

---
apiVersion: v1
kind: ConfigMap
metadata:
  name: payment-api-k6-script
  namespace: k6-tests
data:
  script.js: |
    import http from 'k6/http';
    import { check, sleep } from 'k6';

    export const options = {
      stages: [
        { duration: '2m', target: 50 },
        { duration: '5m', target: 50 },
        { duration: '2m', target: 200 },
        { duration: '2m', target: 0 },
      ],
      thresholds: {
        http_req_duration: ['p(99)<1000'],
        http_req_failed: ['rate<0.01'],
      },
    };

    export default function () {
      const res = http.post(`${__ENV.TARGET_URL}/api/payments`,
        JSON.stringify({ amount: 100, currency: 'USD' }),
        { headers: { 'Content-Type': 'application/json' } }
      );
      check(res, { 'status 200': (r) => r.status === 200 });
      sleep(1);
    }
```

---

## Step 956: Kyverno Policy Testing

```yaml
apiVersion: cli.kyverno.io/v1alpha1
kind: Test
metadata:
  name: test-require-labels
spec:
  policies:
    - require-labels.yaml
  resources:
    - resources/
  results:
    - policy: require-labels
      rule: check-for-labels
      resource: deployment-with-labels
      namespace: default
      kind: Deployment
      result: pass
    
    - policy: require-labels
      rule: check-for-labels
      resource: deployment-without-labels
      namespace: default
      kind: Deployment
      result: fail
```

---

## Step 957: E2E Testing with Kind

```yaml
# kind cluster config
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
    kubeadmConfigPatches:
      - |
        kind: InitConfiguration
        nodeRegistration:
          kubeletExtraArgs:
            node-labels: "ingress-ready=true"
    extraPortMappings:
      - containerPort: 80
        hostPort: 8080
      - containerPort: 443
        hostPort: 8443
  - role: worker
  - role: worker
```

---

## Step 958: Chaos Engineering Testing

```yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: PodChaos
metadata:
  name: payment-api-pod-kill-test
  namespace: chaos-testing
spec:
  action: pod-kill
  mode: one
  selector:
    namespaces: [production]
    labelSelectors:
      app: payment-api
  scheduler:
    cron: "@every 5m"
  duration: 1m

---
apiVersion: chaos-mesh.org/v1alpha1
kind: NetworkChaos
metadata:
  name: database-network-delay-test
  namespace: chaos-testing
spec:
  action: delay
  mode: all
  selector:
    namespaces: [production]
    labelSelectors:
      app: payment-api
  delay:
    latency: 100ms
    jitter: 50ms
  direction: to
  target:
    selector:
      namespaces: [production]
      labelSelectors:
        app: postgres
  duration: 5m
```

---

## Step 959: Security Testing

```yaml
# Trivy Operator: continuous vulnerability scanning
# trivy image myorg/myapp:1.0

apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: security-scan-alerts
  namespace: monitoring
spec:
  groups:
    - name: security-scans
      rules:
        - alert: CriticalVulnerabilityFound
          expr: |
            trivy_vulnerability_id{severity="CRITICAL"} > 0
          for: 0m
          annotations:
            summary: "Critical vulnerability {{ $labels.vuln_id }} in {{ $labels.image }}"
          labels:
            severity: critical
        
        - alert: HighVulnerabilityCount
          expr: |
            count(trivy_vulnerability_id{severity="HIGH"}) by (image) > 10
          for: 1h
          annotations:
            summary: "{{ $labels.image }} has > 10 HIGH vulnerabilities"
          labels:
            severity: warning
```

---

## Step 960: Workshop - Testing Pipeline

```yaml
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: full-test-pipeline
  namespace: tekton-pipelines
spec:
  params:
    - name: image
      type: string
    - name: chart-version
      type: string
  
  tasks:
    - name: lint-yaml
      taskRef:
        name: yamllint-task
      params:
        - name: manifests-path
          value: manifests/
    
    - name: validate-schema
      taskRef:
        name: kubeconform-task
      runAfter: [lint-yaml]
    
    - name: security-scan
      taskRef:
        name: trivy-scan-task
      params:
        - name: image
          value: $(params.image)
      runAfter: [validate-schema]
    
    - name: deploy-to-staging
      taskRef:
        name: helm-deploy-task
      params:
        - name: chart-version
          value: $(params.chart-version)
        - name: environment
          value: staging
      runAfter: [security-scan]
    
    - name: load-test
      taskRef:
        name: k6-run-task
      params:
        - name: duration
          value: "5m"
      runAfter: [deploy-to-staging]
```

---

## 📊 สรุป Part 99

| Tool | Purpose | When to Use |
|------|---------|------------|
| yamllint + kubeconform | YAML validation | Every commit |
| kube-score | Best practices check | PR gate |
| Kyverno CLI | Policy unit tests | Policy changes |
| Testkube | K8s-native test runner | CI/CD |
| k6 Operator | Load testing in K8s | Pre-production |
| Chaos Mesh | Resilience testing | Weekly/scheduled |
| Trivy Operator | Continuous CVE scan | Always running |
| kind/k3d | Local integration test | PR gate |

---

## 🔗 ต่อไป
- [Part 100: Cloud Provider Integrations (EKS, GKE, AKS)](./part-100-cloud-providers.md)

---
*Part 99 | Steps 951-960 | Testing, Validation, and Quality Assurance | Educational Content*
