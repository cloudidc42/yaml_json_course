# Part 14: CI/CD Pipelines กับ YAML
## Steps 131-140: GitHub Actions, GitLab CI, และ Jenkins

---

## 📖 บทนำ

CI/CD (Continuous Integration/Continuous Delivery) pipelines ส่วนใหญ่ใช้ YAML สำหรับ configuration ทำให้ automation เป็นเรื่อง declarative และ version-controlled

---

## Step 131: GitHub Actions

### โครงสร้างพื้นฐาน
```yaml
# .github/workflows/ci.yml
name: CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 0 * * *'
  workflow_dispatch:

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build:
    runs-on: ubuntu-latest
    
    permissions:
      contents: read
      packages: write
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run tests
        run: npm test
      
      - name: Build
        run: npm run build
```

### Complete CI/CD Workflow
```yaml
name: Complete CI/CD

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  test:
    name: Test
    runs-on: ubuntu-latest
    
    strategy:
      matrix:
        node-version: [18, 20, 22]
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node ${{ matrix.node-version }}
        uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
          cache: npm
      
      - name: Install
        run: npm ci
      
      - name: Lint
        run: npm run lint
      
      - name: Test
        run: npm test -- --coverage
      
      - name: Upload coverage
        uses: codecov/codecov-action@v3
        if: matrix.node-version == '20'
        with:
          token: ${{ secrets.CODECOV_TOKEN }}
  
  build:
    name: Build Image
    runs-on: ubuntu-latest
    needs: [test]
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      
      - name: Login to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=ref,event=branch
            type=semver,pattern={{version}}
            type=sha,prefix=sha-
      
      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          platforms: linux/amd64,linux/arm64
          push: ${{ github.event_name != 'pull_request' }}
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
      
      - name: Scan image with Trivy
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:sha-${{ github.sha }}
          format: sarif
          output: trivy-results.sarif
  
  deploy-staging:
    name: Deploy Staging
    runs-on: ubuntu-latest
    needs: [build]
    if: github.ref == 'refs/heads/main'
    environment:
      name: staging
      url: https://staging.example.com
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ap-southeast-1
      
      - name: Update kubeconfig
        run: aws eks update-kubeconfig --name my-staging-cluster
      
      - name: Deploy to Kubernetes
        run: |
          kubectl set image deployment/my-app \
            app=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:sha-${{ github.sha }} \
            -n staging
          kubectl rollout status deployment/my-app -n staging --timeout=5m
      
      - name: Rollback on failure
        if: failure()
        run: kubectl rollout undo deployment/my-app -n staging
```

### Reusable Workflows
```yaml
# .github/workflows/deploy.yml (reusable workflow)
on:
  workflow_call:
    inputs:
      environment:
        required: true
        type: string
      image-tag:
        required: true
        type: string
    secrets:
      AWS_ACCESS_KEY_ID:
        required: true
      AWS_SECRET_ACCESS_KEY:
        required: true

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: ${{ inputs.environment }}
    steps:
      - name: Deploy ${{ inputs.image-tag }} to ${{ inputs.environment }}
        run: echo "Deploying ${{ inputs.image-tag }} to ${{ inputs.environment }}"
```

---

## Step 132: GitLab CI

```yaml
# .gitlab-ci.yml
stages:
  - test
  - build
  - deploy

variables:
  DOCKER_IMAGE: $CI_REGISTRY_IMAGE
  APP_ENV: production

default:
  image: node:20-alpine
  before_script:
    - npm ci --cache .npm
  cache:
    paths:
      - .npm/

test:unit:
  stage: test
  script:
    - npm test
  coverage: '/Statements\s*:\s*([^%]+)%/'
  artifacts:
    reports:
      coverage_report:
        coverage_format: cobertura
        path: coverage/cobertura-coverage.xml

test:security:
  stage: test
  script:
    - npm audit --audit-level=high

build:image:
  stage: build
  image: docker:24
  services:
    - docker:24-dind
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
  script:
    - docker build -t $DOCKER_IMAGE:$CI_COMMIT_SHA .
    - docker push $DOCKER_IMAGE:$CI_COMMIT_SHA
  only:
    - main

deploy:staging:
  stage: deploy
  environment:
    name: staging
    url: https://staging.example.com
  before_script:
    - apk add --no-cache curl kubectl
    - echo $KUBE_CONFIG | base64 -d > ~/.kube/config
  script:
    - kubectl set image deployment/app app=$DOCKER_IMAGE:$CI_COMMIT_SHA -n staging
    - kubectl rollout status deployment/app -n staging
  only:
    - main

deploy:production:
  stage: deploy
  environment:
    name: production
    url: https://example.com
  before_script:
    - echo $KUBE_CONFIG_PROD | base64 -d > ~/.kube/config
  script:
    - kubectl set image deployment/app app=$DOCKER_IMAGE:$CI_COMMIT_SHA -n production
    - kubectl rollout status deployment/app -n production
  only:
    - main
  when: manual
```

---

## Step 133: ArgoCD (GitOps)

### Application CRD
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  
  source:
    repoURL: https://github.com/my-org/my-app
    targetRevision: HEAD
    path: k8s/overlays/production
  
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ApplyOutOfSyncOnly=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
  
  ignoreDifferences:
    - group: apps
      kind: Deployment
      jsonPointers:
        - /spec/replicas
```

### Kustomize สำหรับ GitOps
```yaml
# k8s/base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml
  - service.yaml
  - configmap.yaml
  - ingress.yaml

commonLabels:
  app: my-app
  managed-by: argocd

---
# k8s/overlays/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

bases:
  - ../../base

namespace: production

images:
  - name: my-app
    newName: ghcr.io/my-org/my-app
    newTag: sha-abc123

patches:
  - path: replica-patch.yaml
  - path: resource-patch.yaml

configMapGenerator:
  - name: app-config
    literals:
      - APP_ENV=production
      - LOG_LEVEL=warning

---
# k8s/overlays/staging/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

bases:
  - ../../base

namespace: staging

images:
  - name: my-app
    newName: ghcr.io/my-org/my-app
    newTag: latest

patches:
  - patch: |
      apiVersion: apps/v1
      kind: Deployment
      metadata:
        name: my-app
      spec:
        replicas: 1
```

---

## Step 134: Jenkins Pipeline กับ YAML

```groovy
// Jenkinsfile
pipeline {
    agent {
        kubernetes {
            yaml '''
apiVersion: v1
kind: Pod
spec:
  containers:
    - name: node
      image: node:20-alpine
      command: [sleep, infinity]
    - name: docker
      image: docker:24-dind
      securityContext:
        privileged: true
'''
        }
    }
    
    environment {
        REGISTRY = 'registry.example.com'
        IMAGE = "${REGISTRY}/my-app"
    }
    
    stages {
        stage('Install') {
            steps {
                container('node') {
                    sh 'npm ci'
                }
            }
        }
        
        stage('Test') {
            parallel {
                stage('Unit Tests') {
                    steps {
                        container('node') {
                            sh 'npm test'
                        }
                    }
                }
                stage('Lint') {
                    steps {
                        container('node') {
                            sh 'npm run lint'
                        }
                    }
                }
            }
        }
        
        stage('Build') {
            steps {
                container('docker') {
                    sh """
                        docker build -t ${IMAGE}:${env.GIT_COMMIT} .
                        docker push ${IMAGE}:${env.GIT_COMMIT}
                    """
                }
            }
        }
    }
    
    post {
        failure {
            slackSend(
                color: 'danger',
                message: "Build failed: ${env.JOB_NAME} ${env.BUILD_NUMBER}"
            )
        }
    }
}
```

---

## Step 135: GitHub Actions สำหรับ Python

```yaml
name: Python CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
          POSTGRES_DB: testdb
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
      
      redis:
        image: redis:7
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    
    strategy:
      matrix:
        python-version: ['3.10', '3.11', '3.12']
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Python ${{ matrix.python-version }}
        uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}
          cache: pip
      
      - name: Install dependencies
        run: |
          pip install -r requirements.txt
          pip install -r requirements-dev.txt
      
      - name: Lint with ruff
        run: ruff check .
      
      - name: Type check
        run: mypy app/
      
      - name: Run tests
        env:
          DATABASE_URL: postgresql://test:test@localhost:5432/testdb
          REDIS_URL: redis://localhost:6379/0
        run: |
          pytest tests/ \
            --cov=app \
            --cov-report=xml \
            -v
      
      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          file: coverage.xml
```

---

## 📊 สรุป Part 14

| Platform | หัวข้อที่เรียน |
|----------|---------------|
| GitHub Actions | workflows, jobs, steps, matrix, environment |
| GitLab CI | stages, jobs, variables, artifacts |
| ArgoCD | GitOps, Application CRD, Kustomize |
| Jenkins | Declarative pipeline, parallel stages |
| Best Practices | Reusable workflows, conditions, notifications |

---

## 🔗 ต่อไป
- [Part 15: Observability กับ YAML](./part-15-observability-yaml.md)

---
*Part 14 | Steps 131-140 | ระดับพื้นฐาน*
