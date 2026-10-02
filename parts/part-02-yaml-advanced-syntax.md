# Part 02: YAML Advanced Syntax
## Steps 11-20: YAML ขั้นสูง

---

## Step 11: Tags และ Type Coercion

### YAML Tags คืออะไร?
Tags ใช้บอก YAML parser ว่าควร interpret ค่านี้เป็น type อะไร

```yaml
# Explicit type tags
string_explicit: !!str 42          # integer 42 → string "42"
int_explicit: !!int "42"           # string "42" → integer 42
float_explicit: !!float "3.14"     # string → float
bool_explicit: !!bool "true"       # string → boolean
null_explicit: !!null ""           # empty string → null

# Binary data
binary_data: !!binary |
  R0lGODlhDAAMAIQAAAAAAHh4eH9/f4iIiJCQ
  kJiYmKCgoKioqLCwsLi4uMDAwMjIyNDQ0NjY
  2ODg4Ojoy/xPAAAh+QQJCAAQACwAAAAADAAM
  
# Timestamp
created_at: !!timestamp 2024-01-15T10:30:00Z

# Ordered mapping (!!omap)
ordered: !!omap
  - key1: value1
  - key2: value2
  - key3: value3
```

### Custom Tags
```yaml
# Python-specific tags (ระวัง! ไม่ safe ถ้าใช้ yaml.load แทน yaml.safe_load)
# ตัวอย่างนี้แสดงให้เห็นว่า tags สามารถ inject ได้
dangerous: !!python/object/apply:os.system
  args: ['echo hello']

# ใช้ safe_load เสมอ!
```

---

## Step 12: Flow Style vs Block Style

### Flow Style (inline)
```yaml
# Flow mapping
point: {x: 10, y: 20}
nested_flow: {outer: {inner: value}}

# Flow sequence
numbers: [1, 2, 3, 4, 5]
mixed: [true, 42, "text", null, 3.14]

# Nested flow
matrix: [[1,2,3], [4,5,6], [7,8,9]]

# Mixed styles
config:
  server: {host: localhost, port: 8080}
  allowed_ips: [192.168.1.1, 10.0.0.1]
```

### Block Style
```yaml
# Block mapping
point:
  x: 10
  y: 20

# Block sequence
numbers:
  - 1
  - 2
  - 3

# Compact block sequence notation
person:
  name: Alice
  hobbies:
  - reading    # ไม่ต้อง indent เพิ่ม
  - coding
```

### เมื่อไหรควรใช้ Style ไหน?

| Style | ใช้เมื่อ |
|-------|------|
| Flow | ข้อมูลสั้น, simple values, coordinates |
| Block | ข้อมูลซับซ้อน, human-readable, Kubernetes config |

---

## Step 13: Multiline Strings ขั้นสูง

### Literal Block (|) - ทุกบรรทัด
```yaml
# รักษา newlines
script: |
  #!/bin/bash
  set -e
  echo "Starting deployment..."
  kubectl apply -f deployment.yaml
  kubectl rollout status deployment/myapp
  echo "Done!"

# ผลลัพธ์จะมี newline ทุกบรรทัด
```

### Folded Block (>) - รวมบรรทัด
```yaml
# รวม newlines เป็น spaces (ยกเว้น blank lines)
description: >
  This is a long description
  that spans multiple lines
  but will be folded into
  a single paragraph.
  
  This blank line creates a new paragraph.
  So this is the second paragraph.
```

### Chomping Modifiers
```yaml
# | (clip) - เก็บ newline สุดท้าย 1 ตัว (default)
clip: |
  hello
  

# |- (strip) - ตัด newlines ท้ายออกทั้งหมด
strip: |-
  hello
  

# |+ (keep) - เก็บ newlines ท้ายทั้งหมด
keep: |+
  hello
  

# |2 - บอกว่า content ถูก indent 2 ตัว
indented: |2
    This content is indented
    by 2 extra spaces
```

### Kubernetes ConfigMap Example
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  start.sh: |
    #!/bin/bash
    set -euo pipefail
    
    echo "Starting application..."
    
    until pg_isready -h "$DB_HOST" -p "$DB_PORT"; do
      echo "Waiting for database..."
      sleep 2
    done
    
    python manage.py migrate
    exec gunicorn app:wsgi --bind 0.0.0.0:8000 --workers 4
  
  nginx.conf: |
    server {
        listen 80;
        server_name _;
        
        location / {
            proxy_pass http://app:8000;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }
    }
```

---

## Step 14: Complex Anchors และ Merge Keys

```yaml
# Base configurations
base_container: &base_container
  image: ubuntu:22.04
  imagePullPolicy: IfNotPresent
  securityContext:
    runAsNonRoot: true
    readOnlyRootFilesystem: true
  resources:
    limits:
      cpu: 500m
      memory: 512Mi
    requests:
      cpu: 100m
      memory: 128Mi

# Override specific fields
app_container:
  <<: *base_container
  name: app
  image: myapp:1.0.0
  resources:
    limits:
      cpu: 1000m
      memory: 1Gi
    requests:
      cpu: 200m
      memory: 256Mi

# Multiple merges (ลำดับสำคัญ - ท้ายสุด override)
config_a: &config_a
  key1: value_a
  key2: value_a

config_b: &config_b
  key2: value_b
  key3: value_b

merged_config:
  <<: [*config_a, *config_b]
  key4: value_d
```

### Real-world Kubernetes กับ Anchors
```yaml
---
x-common-labels: &common-labels
  app.kubernetes.io/name: myapp
  app.kubernetes.io/version: "1.0.0"
  app.kubernetes.io/managed-by: Helm

x-common-env: &common-env
  - name: APP_ENV
    value: production
  - name: LOG_LEVEL
    value: info
  - name: DB_HOST
    valueFrom:
      secretKeyRef:
        name: db-secret
        key: host
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: db-secret
        key: password

x-standard-resources: &standard-resources
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 100m
    memory: 128Mi

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  labels:
    <<: *common-labels
    tier: web
spec:
  replicas: 3
  selector:
    matchLabels:
      app.kubernetes.io/name: myapp
      tier: web
  template:
    metadata:
      labels:
        <<: *common-labels
        tier: web
    spec:
      containers:
        - name: web
          image: myapp-web:1.0.0
          env:
            - *common-env
          resources:
            <<: *standard-resources
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: worker
  labels:
    <<: *common-labels
    tier: worker
spec:
  replicas: 2
  selector:
    matchLabels:
      app.kubernetes.io/name: myapp
      tier: worker
  template:
    metadata:
      labels:
        <<: *common-labels
        tier: worker
    spec:
      containers:
        - name: worker
          image: myapp-worker:1.0.0
          command: ["python", "-m", "celery", "worker"]
          env:
            - *common-env
          resources:
            <<: *standard-resources
            limits:
              cpu: 1000m
```

---

## Step 15: YAML Sets

```yaml
# YAML Set (unique values)
fruits: !!set
  ? apple
  ? banana
  ? cherry

# Ordered Pairs
coordinates: !!omap
  - x: 10
  - y: 20
  - z: 30

# Pairs (allows duplicates)
entries: !!pairs
  - name: Alice
  - name: Bob
  - role: admin
  - role: user
```

---

## Step 16: YAML Directives

```yaml
# YAML version directive
%YAML 1.2
---
name: document with version

# TAG directive - ลดการพิมพ์ tag ยาวๆ
%TAG ! tag:example.com,2024:
%TAG !! tag:yaml.org,2002:
---
config: !app/config
  version: 1.0
```

---

## Step 17: Handling Special Values

```yaml
# ⚠️ Country codes ที่ถูก parse เป็น boolean (YAML 1.1)
country_no: NO     # ⚠️ false (Norway abbreviation!)
country_yes: YES   # ⚠️ true

# Fix: ใส่ quotes เสมอ
country_no: "NO"
country_yes: "YES"

# ⚠️ Version numbers
version: 1.0       # อาจถูก parse เป็น float
version: "1.0"     # safe - เป็น string

# ⚠️ Port numbers
port: 080          # ⚠️ Octal! = 64 ใน decimal
port: "080"        # safe
port: 80           # safe (ไม่มี leading zero)

# ⚠️ Timestamps
date: 2024-01-15   # ถูก parse เป็น date object
date: "2024-01-15" # เป็น string

# ⚠️ Null coercion
value1:            # null
value2: ~          # null
value3: null       # null
value4: ""         # empty string (ไม่ใช่ null!)

# ⚠️ Octal, Hex
octal: 0o777       # 511 ใน decimal
hex: 0xFF          # 255 ใน decimal

# ✅ Safe practice
safe:
  version: "1.0"
  port: "080"
  country: "NO"
  date: "2024-01-15"
  yes_value: "yes"
  no_value: "no"
```

### Kubernetes ที่พบปัญหา
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
  labels:
    version: "1.0"    # ✅ string
spec:
  containers:
    - name: app
      env:
        - name: API_VERSION
          value: "1.0"     # ✅ string
        - name: ENABLED
          value: "true"    # ✅ string
        - name: DEBUG
          value: "yes"     # ✅ string
```

---

## Step 18: YAML Schema Validation

### JSON Schema สำหรับ YAML
```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "required": ["name", "version", "database"],
  "properties": {
    "name": {
      "type": "string",
      "minLength": 1,
      "maxLength": 100
    },
    "version": {
      "type": "string",
      "pattern": "^\\d+\\.\\d+\\.\\d+$"
    },
    "database": {
      "type": "object",
      "required": ["host", "port"],
      "properties": {
        "host": {"type": "string"},
        "port": {
          "type": "integer",
          "minimum": 1,
          "maximum": 65535
        }
      }
    }
  }
}
```

### Python Validation Script
```python
import yaml
import json
import jsonschema


def validate_yaml_against_schema(yaml_file: str, schema_file: str) -> bool:
    with open(schema_file) as f:
        schema = json.load(f)
    
    with open(yaml_file) as f:
        data = yaml.safe_load(f)
    
    try:
        jsonschema.validate(data, schema)
        print(f"✅ {yaml_file} is valid!")
        return True
    except jsonschema.ValidationError as e:
        print(f"❌ Validation error in {yaml_file}:")
        print(f"   Path: {' -> '.join(str(p) for p in e.path)}")
        print(f"   Error: {e.message}")
        return False
```

---

## Step 19: YAML Template Engines

### Jinja2 กับ YAML (ใช้ใน Ansible, Helm)
```yaml
# Ansible playbook ที่ใช้ Jinja2
---
- name: Deploy application
  hosts: "{{ target_hosts }}"
  vars:
    app_name: myapp
    app_version: "1.0.0"
    env: production
    
  tasks:
    - name: Create config file
      template:
        src: config.yaml.j2
        dest: /etc/myapp/config.yaml
      
    - name: Set environment
      when: env == 'production'
      debug:
        msg: "Deploying {{ app_name }} v{{ app_version }} to production"
```

```yaml
# config.yaml.j2 - Jinja2 template
app:
  name: {{ app_name }}
  version: {{ app_version }}
  
{% if env == 'production' %}
server:
  workers: 8
  debug: false
{% else %}
server:
  workers: 2
  debug: true
{% endif %}

database:
  host: {{ db_host | default('localhost') }}
  port: {{ db_port | default(5432) }}
  name: {{ app_name }}_{{ env }}
```

### Helm Chart Templates
```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "myapp.fullname" . }}
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "myapp.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "myapp.selectorLabels" . | nindent 8 }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          ports:
            - name: http
              containerPort: {{ .Values.service.port }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
```

```yaml
# values.yaml
replicaCount: 3

image:
  repository: myapp
  pullPolicy: IfNotPresent
  tag: ""

service:
  type: ClusterIP
  port: 8080

resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 100m
    memory: 128Mi

autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 80
```

---

## Step 20: YAML Best Practices ขั้นสูง

### 1. Consistency
```yaml
# ✅ สม่ำเสมอ: ใช้ 2 spaces indent
server:
  host: localhost
  port: 8080
```

### 2. Quote Strings ที่อาจ Ambiguous
```yaml
# ✅ Safe
version: "1.0.0"
enabled: "true"
country: "NO"
port: "8080"
```

### 3. Comments ที่มีประโยชน์
```yaml
database:
  # Connection pool size - ปรับตาม load
  pool_size: 20
  
  # Timeout เป็น seconds
  connection_timeout: 30
  
  # ตั้งตาม hardware: max_connections ใน postgresql.conf
  max_overflow: 10
```

### 4. Validation Script
```python
#!/usr/bin/env python3
import yaml

def check_yaml_pitfalls(data, path=""):
    issues = []
    
    if isinstance(data, dict):
        for key, value in data.items():
            current_path = f"{path}.{key}" if path else str(key)
            
            if isinstance(value, bool):
                issues.append({
                    'path': current_path,
                    'issue': f"Boolean value '{value}'. Is this intentional?",
                    'tip': f"If you want string, use '\"{ str(value).lower()}\"'"
                })
            
            if value is None:
                issues.append({
                    'path': current_path,
                    'issue': "Null value. Is this intentional?",
                })
            
            if isinstance(value, float) and str(value).count('.') == 1:
                issues.append({
                    'path': current_path,
                    'issue': f"Float value '{value}'. Could this be a version string?",
                    'tip': f"If version, use '\"{ value}\"'"
                })
            
            if isinstance(value, (dict, list)):
                issues.extend(check_yaml_pitfalls(value, current_path))
    
    elif isinstance(data, list):
        for i, item in enumerate(data):
            if isinstance(item, (dict, list)):
                issues.extend(check_yaml_pitfalls(item, f"{path}[{i}]"))
    
    return issues


test_yaml = """
version: 1.0
debug: true
enabled: yes
country: NO
database:
  host: localhost
  port: 5432
  password:
"""

data = yaml.safe_load(test_yaml)
issues = check_yaml_pitfalls(data)
print("Pitfalls found:")
for issue in issues:
    print(f"  - {issue['path']}: {issue['issue']}")
```

---

## 📊 สรุป Part 2

| หัวข้อ | สิ่งที่เรียน |
|--------|------------|
| Tags | !!str, !!int, !!float, !!bool, !!null |
| Flow vs Block Style | เมื่อไหรใช้ style ไหน |
| Multiline Strings | \|, >, \|-, \|+ |
| Complex Anchors | Merge keys, multiple merges |
| YAML Sets | !!set, !!omap, !!pairs |
| Special Values | Gotchas: boolean, octal, null |
| Schema Validation | JSON Schema กับ YAML |
| Template Engines | Jinja2, Helm |
| Best Practices | Consistency, quoting, comments |

---

## 🔗 ต่อไป
- [Part 03: JSON Fundamentals](./part-03-json-fundamentals.md)

---
*Part 02 | Steps 11-20 | ระดับพื้นฐาน*
