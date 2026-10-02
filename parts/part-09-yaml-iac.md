# Part 09: YAML สำหรับ Infrastructure as Code
## Steps 81-90: Ansible, Terraform, Pulumi

---

## Step 81: Infrastructure as Code คืออะไร?

```
Infrastructure as Code (IaC):
การจัดการ infrastructure โดยใช้ code/config files
แทนที่จะทำ manual

เครื่องมือ:
- Ansible (Configuration Management)
- Terraform (Cloud Infrastructure Provisioning)
- Pulumi (IaC with programming languages)
- CloudFormation (AWS specific)
- Helm (Kubernetes Package Manager)
- Kustomize (Kubernetes Config Management)
```

---

## Step 82: Ansible Fundamentals

```yaml
# inventory.yaml
all:
  children:
    webservers:
      hosts:
        web1.example.com:
          ansible_user: ubuntu
          ansible_ssh_private_key_file: ~/.ssh/id_rsa
        web2.example.com:
          ansible_user: ubuntu
    databases:
      hosts:
        db1.example.com:
          ansible_user: ubuntu
          db_port: 5432
  vars:
    ansible_python_interpreter: /usr/bin/python3
    environment: production
```

### Playbook
```yaml
# setup-webserver.yaml
---
- name: Setup Web Server
  hosts: webservers
  become: yes
  vars:
    app_port: 8080
    node_version: "18"
    
  tasks:
    - name: Update apt cache
      apt:
        update_cache: yes
        cache_valid_time: 3600
    
    - name: Install required packages
      apt:
        name: [nginx, curl, git, python3-pip]
        state: present
    
    - name: Start nginx
      service:
        name: nginx
        state: started
        enabled: yes
    
    - name: Create app directory
      file:
        path: /opt/myapp
        state: directory
        owner: www-data
        mode: '0755'
    
    - name: Configure nginx
      template:
        src: templates/nginx.conf.j2
        dest: /etc/nginx/sites-available/myapp
      notify: Reload nginx
  
  handlers:
    - name: Reload nginx
      service:
        name: nginx
        state: reloaded
```

---

## Step 83: Ansible Roles

```
roles/webapp/
├── tasks/
│   ├── main.yaml
│   ├── install.yaml
│   └── configure.yaml
├── handlers/main.yaml
├── templates/nginx.conf.j2
├── vars/main.yaml
└── defaults/main.yaml
```

```yaml
# roles/webapp/defaults/main.yaml
---
webapp_user: www-data
webapp_dir: /opt/webapp
webapp_port: 8080
webapp_packages: [python3, python3-pip, nginx]
webapp_debug: false
webapp_log_level: info

# roles/webapp/tasks/install.yaml
---
- name: Install application dependencies
  apt:
    name: "{{ webapp_packages }}"
    state: present

- name: Create application user
  user:
    name: "{{ webapp_user }}"
    system: yes
    shell: /sbin/nologin

- name: Create directories
  file:
    path: "{{ item }}"
    state: directory
    owner: "{{ webapp_user }}"
    mode: '0755'
  loop:
    - "{{ webapp_dir }}"
    - "{{ webapp_dir }}/logs"
```

```yaml
# deploy.yaml - ใช้ roles
---
- name: Deploy Web Application
  hosts: webservers
  become: yes
  
  roles:
    - role: webapp
      vars:
        webapp_port: 8080
        webapp_debug: false
    - role: monitoring
      vars:
        monitoring_interval: 30
  
  post_tasks:
    - name: Verify application is running
      uri:
        url: "http://localhost:{{ webapp_port }}/health"
        status_code: 200
      retries: 5
      delay: 10
```

---

## Step 84: Ansible Variables และ Conditionals

```yaml
tasks:
  - name: Install on Ubuntu only
    apt:
      name: curl
    when: ansible_distribution == "Ubuntu"
  
  - name: Create multiple users
    user:
      name: "{{ item.name }}"
      uid: "{{ item.uid }}"
    loop:
      - { name: alice, uid: 1001 }
      - { name: bob, uid: 1002 }
  
  - name: Check if file exists
    stat:
      path: /opt/myapp/.installed
    register: install_marker
  
  - name: Run setup only if not installed
    script: setup.sh
    when: not install_marker.stat.exists
```

---

## Step 85: Ansible Vault

```bash
ansible-vault create secrets.yaml
ansible-vault edit secrets.yaml
ansible-vault encrypt config.yaml
ansible-playbook deploy.yaml --ask-vault-pass
ansible-playbook deploy.yaml --vault-password-file ~/.vault_pass
```

```yaml
# secrets.yaml (เขียน encrypted)
---
db_password: "super-secret-password"
api_key: "abc123xyz"

# playbook.yaml
---
- hosts: databases
  vars_files:
    - secrets.yaml
  tasks:
    - name: Configure database
      template:
        src: db.conf.j2
        dest: /etc/postgresql/postgresql.conf
```

---

## Step 86: Terraform HCL + YAML

```hcl
# main.tf - อ่าน config จาก YAML
locals {
  config = yamldecode(file("${path.module}/config.yaml"))
}

resource "aws_instance" "web" {
  count         = local.config.instance_count
  ami           = local.config.ami_id
  instance_type = local.config.instance_type
  tags          = local.config.tags
}
```

```yaml
# config.yaml
instance_count: 3
ami_id: "ami-0c55b159cbfafe1f0"
instance_type: "t3.micro"
tags:
  Environment: production
  Team: backend
```

### Pulumi + Python + YAML
```python
import pulumi, pulumi_aws as aws, yaml

with open("infrastructure.yaml") as f:
    infra_config = yaml.safe_load(f)

vpc = aws.ec2.Vpc(
    "main-vpc",
    cidr_block=infra_config["vpc"]["cidr"],
    enable_dns_hostnames=True,
    tags={"Name": infra_config["vpc"]["name"]}
)

for i, subnet_config in enumerate(infra_config["subnets"]):
    aws.ec2.Subnet(
        f"subnet-{i}",
        vpc_id=vpc.id,
        cidr_block=subnet_config["cidr"],
        availability_zone=subnet_config["az"]
    )

pulumi.export("vpc_id", vpc.id)
```

---

## Step 87: AWS CloudFormation YAML

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: 'Web Application Stack'

Parameters:
  Environment:
    Type: String
    AllowedValues: [dev, staging, prod]
    Default: dev
  InstanceType:
    Type: String
    Default: t3.micro

Mappings:
  RegionAMI:
    ap-southeast-1:
      AMI: ami-0c55b159cbfafe1f0

Conditions:
  IsProd: !Equals [!Ref Environment, prod]

Resources:
  VPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 10.0.0.0/16
      EnableDnsHostnames: true
      Tags:
        - Key: Name
          Value: !Sub '${Environment}-vpc'
  
  WebServer:
    Type: AWS::EC2::Instance
    Properties:
      ImageId: !FindInMap [RegionAMI, !Ref AWS::Region, AMI]
      InstanceType: !Ref InstanceType
      UserData: !Base64
        'Fn::Sub': |
          #!/bin/bash
          yum update -y
          yum install -y nginx
          systemctl start nginx

Outputs:
  WebServerURL:
    Value: !Sub 'http://${WebServer.PublicDnsName}'
    Export:
      Name: !Sub '${Environment}-WebServerURL'
```

---

## Step 88: Helm Charts

```yaml
# Chart.yaml
apiVersion: v2
name: myapp
description: My Application Helm Chart
type: application
version: 1.2.3
appVersion: "2.0.0"
dependencies:
  - name: postgresql
    version: "12.x.x"
    repository: https://charts.bitnami.com/bitnami
    condition: postgresql.enabled

---
# values.yaml
replicaCount: 2
image:
  repository: myregistry/myapp
  tag: "latest"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80
  targetPort: 8080

ingress:
  enabled: true
  className: nginx
  hosts:
    - host: myapp.example.com
      paths: [{path: /, pathType: Prefix}]
  tls:
    - secretName: myapp-tls
      hosts: [myapp.example.com]

resources:
  limits: {cpu: 500m, memory: 512Mi}
  requests: {cpu: 100m, memory: 128Mi}

autoscaling:
  enabled: false
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 80
```

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "myapp.fullname" . }}
  labels: {{- include "myapp.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels: {{- include "myapp.selectorLabels" . | nindent 6 }}
  template:
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          ports:
            - containerPort: {{ .Values.service.targetPort }}
          resources: {{- toYaml .Values.resources | nindent 12 }}
          livenessProbe:
            httpGet: {path: /health, port: http}
            initialDelaySeconds: 30
```

---

## Step 89: Kustomize

```yaml
# k8s/base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
  - service.yaml
  - configmap.yaml
commonLabels:
  app: myapp
  managed-by: kustomize

---
# k8s/overlays/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
bases:
  - ../../base
namespace: production
replicas:
  - name: myapp
    count: 5
images:
  - name: myregistry/myapp
    newTag: v2.0.0
patches:
  - path: patch-resources.yaml
    target: {kind: Deployment, name: myapp}
configMapGenerator:
  - name: myapp-config
    literals:
      - NODE_ENV=production
      - LOG_LEVEL=warn
    behavior: merge
```

---

## Step 90: IaC Best Practices

```yaml
# ✓ ใช้ Variables สำหรับ repeated values
vars:
  app_name: myapp
  app_version: "2.0.0"
  app_port: 8080

# ✓ Validate inputs
- name: Validate required vars
  assert:
    that:
      - app_name is defined
      - app_name | length > 0
    fail_msg: "app_name must be defined and non-empty"

# ✓ Idempotent tasks
- name: Create directory
  file:
    path: /opt/myapp
    state: directory  # ไม่ล้มเหลวถ้ามีอยู่แล้ว

# ✓ ใช้ tags
- name: Install packages
  apt: {name: "{{ packages }}"}
  tags: [packages, install]

- name: Configure app
  template: {src: app.conf.j2, dest: /etc/myapp/app.conf}
  tags: [config, configure]
```

```bash
# Dry run
ansible-playbook --check deploy.yaml
# Run specific tags
ansible-playbook deploy.yaml --tags packages,config
```

---

## 📊 สรุป Part 09

| Tool | Use Case | Language |
|------|----------|----------|
| Ansible | Config Management | YAML |
| Terraform | Cloud Provisioning | HCL + YAML |
| Pulumi | IaC Programmatic | Python/TS/Go |
| CloudFormation | AWS-specific | YAML/JSON |
| Helm | Kubernetes Packages | YAML + Go templates |
| Kustomize | K8s Config Overlays | YAML |

---
*Part 09 | Steps 81-90 | ระดับพื้นฐาน/กลาง*
