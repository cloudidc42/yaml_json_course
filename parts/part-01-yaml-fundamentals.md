# Part 01: YAML Fundamentals - พื้นฐาน YAML
## Steps 1-10: เริ่มต้นกับ YAML

---

## 📖 บทนำ

YAML (YAML Ain't Markup Language) เป็น data serialization format ที่มนุษย์อ่านได้ง่าย ถูกใช้งานอย่างแพร่หลายในการเขียน configuration files, CI/CD pipelines, Infrastructure as Code, และ Kubernetes manifests

### ทำไมต้องเรียน YAML?
- **Kubernetes** ใช้ YAML 100% สำหรับ configuration
- **Docker Compose** ใช้ YAML สำหรับ multi-container apps
- **GitHub Actions, GitLab CI, Jenkins** ใช้ YAML สำหรับ pipeline
- **Ansible** ใช้ YAML สำหรับ automation
- **Helm Charts** ใช้ YAML template

---

## Step 1: YAML คืออะไร และประวัติศาสตร์

### ประวัติ YAML
- **2001**: Clark Evans สร้าง YAML ครั้งแรก
- **YAML 1.0**: 2004
- **YAML 1.1**: 2005 (ใช้อยู่ใน Docker, Kubernetes เป็นส่วนใหญ่)
- **YAML 1.2**: 2009 (compatible กับ JSON)
- **YAML 1.3**: 2021 (แก้ไข ambiguity หลายส่วน)

### YAML ย่อมาจากอะไร?
- เดิม: "Yet Another Markup Language"
- ปัจจุบัน: "YAML Ain't Markup Language" (recursive acronym)

```yaml
# ตัวอย่าง YAML ง่ายๆ
# นี่คือ comment ใน YAML
name: John Doe
age: 30
city: Bangkok
```

---

## Step 2: กฎพื้นฐานของ YAML

```yaml
# กฎข้อที่ 1: Indentation ใช้ Spaces ไม่ใช้ Tabs
# ✅ ถูกต้อง - ใช้ spaces
person:
  name: Alice
  age: 25

# ✔️ กฎข้อที่ 2: Case Sensitive
Name: Alice    # ต่างจาก
name: Alice    # นี่
NAME: Alice    # และนี่

# ✔️ กฎข้อที่ 3: Document Separator
---
# เริ่ม document ใหม่
name: Document 1
---
name: Document 2
...
# จบ document
```

---

## Step 3: ประเภทข้อมูลพื้นฐาน (Scalar Types)

```yaml
# String (ข้อความ)
simple_string: Hello World
thai_text: สวัสดีชาวโลก
quoted_string: "Hello, World!"
with_special: "This has: colon and # hash"
single_quoted: 'It''s a single quote inside'

# Multiline string - Literal Block (|)
multiline_literal: |
  บรรทัดที่ 1
  บรรทัดที่ 2
  บรรทัดที่ 3

# Multiline string - Folded Block (>)
multiline_folded: >
  บรรทัดนี้จะ
  ถูกรวมเป็น
  บรรทัดเดียว

# Integer
positive_int: 42
negative_int: -17
octal: 0o755
hex: 0xFF

# Float
positive_float: 3.14159
scientific: 6.022e23

# Boolean
true_value: true
false_value: false

# Null
explicit_null: null
tilde_null: ~

# Date
date: 2024-01-15
datetime: 2024-01-15T10:30:00Z
```

---

## Step 4: Collections - Mappings (Key-Value Pairs)

```yaml
# Simple key-value
name: Alice
age: 30
city: Bangkok

# Nested mapping
person:
  name: Alice
  age: 30
  address:
    street: "123 Main St"
    city: Bangkok
    country: Thailand

# Inline mapping (flow style)
coordinates: {x: 10, y: 20, z: 30}

# ตัวอย่างการใช้งานจริง
database:
  host: localhost
  port: 5432
  name: mydb
  credentials:
    username: dbuser
    password: secret123

server:
  host: 0.0.0.0
  port: 8080
  ssl:
    enabled: true
    cert_file: /etc/ssl/cert.pem
    key_file: /etc/ssl/key.pem
```

---

## Step 5: Collections - Sequences (Lists/Arrays)

```yaml
# Block sequence
fruits:
  - apple
  - banana
  - orange
  - mango

# Inline sequence (flow style)
colors: [red, green, blue]

# List of objects
people:
  - name: Alice
    age: 30
    role: developer
  - name: Bob
    age: 25
    role: designer
  - name: Charlie
    age: 35
    role: manager

# Complex nesting
projects:
  - name: Project Alpha
    team:
      - Alice
      - Bob
    technologies:
      - Python
      - Docker
      - Kubernetes
    status: active
  - name: Project Beta
    team:
      - Charlie
      - Dave
    technologies:
      - Node.js
      - React
      - AWS
    status: planning
```

---

## Step 6: Anchors และ Aliases (การ reuse)

```yaml
# Anchor: &anchor_name
# Alias: *anchor_name

# กำหนด anchor
defaults: &defaults
  timeout: 30
  retries: 3
  log_level: info

# ใช้ alias
development:
  <<: *defaults    # Merge key - ดึงทุก key จาก defaults
  debug: true
  host: localhost

production:
  <<: *defaults    # ดึงทุก key จาก defaults
  host: prod.example.com
  log_level: error  # Override log_level

---
# Docker Compose example
version: '3.8'

x-common-env: &common-env
  RAILS_ENV: production
  NODE_ENV: production
  TZ: Asia/Bangkok

x-common-resources: &common-resources
  deploy:
    resources:
      limits:
        cpus: '0.5'
        memory: 512M

services:
  web:
    image: myapp:latest
    environment:
      <<: *common-env
      PORT: 3000
    <<: *common-resources
    
  worker:
    image: myapp:latest
    command: bundle exec sidekiq
    environment:
      <<: *common-env
      QUEUE: default
    <<: *common-resources
```

---

## Step 7: Comments ใน YAML

```yaml
# นี่คือ comment บรรทัดเดียว

name: Alice  # นี่คือ inline comment

# Block comment
# สามารถเขียนหลายบรรทัด
# โดยขึ้นต้นแต่ละบรรทัดด้วย #

database:
  host: new-server.com
  port: 5432

# ใน YAML ไม่มี multi-line comment แบบ /* */ ของ C
# ต้องใช้ # ทุกบรรทัด
```

---

## Step 8: Special Characters และ Quoting

```yaml
# ต้องใช้ quotes เมื่อ value มี special characters
with_colon: "key: value"
with_hash: "text # not a comment"
with_brackets: "[this is not a list]"
with_braces: "{this is not a map}"

# ตัวเลขที่ต้องการเป็น string
zip_code: "10110"
phone: "0812345678"
version: "1.0"

# Boolean strings
yes_string: "yes"
no_string: "no"
true_string: "true"
null_string: "null"

# Double quotes - รองรับ escape sequences
double_quoted: "Hello\nWorld"     # มี newline
with_tab: "Hello\tWorld"          # มี tab
unicode: "A"                  # 'A'

# Single quotes - ไม่มี escape sequences
single_quoted: 'Hello\nWorld'     # \n เป็น literal string

# Block Scalars
# Literal block scalar (|) - รักษา newlines
literal: |
  line 1
  line 2
  line 3
# ผลลัพธ์: "line 1\nline 2\nline 3\n"

# Folded block scalar (>) - แปลง newline เป็น space
folded: >
  This is a long
  sentence that spans
  multiple lines.
# ผลลัพธ์: "This is a long sentence that spans multiple lines.\n"

# Chomping indicators
literal_clip: |     # default - เก็บ newline ท้าย 1 ตัว
  hello
literal_strip: |-   # ลบ newlines ท้ายทั้งหมด
  hello
literal_keep: |+    # เก็บ newlines ท้ายทั้งหมด
  hello
```

---

## Step 9: YAML Document Structure

```yaml
---
# document start indicator (optional)
name: Simple Document
version: 1.0
...
# document end indicator (optional)

---
name: Document 1
type: config
---
name: Document 2
type: template
```

### Kubernetes ใช้ multiple documents
```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: my-app
  ports:
    - port: 80
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: app
          image: my-image:latest
```

---

## Step 10: การตรวจสอบ YAML (Validation)

```bash
# ติดตั้ง yamllint
pip install yamllint

# ตรวจสอบไฟล์
yamllint myfile.yaml
yamllint -d relaxed myfile.yaml
```

```python
import yaml
import json

def validate_yaml(content):
    try:
        data = yaml.safe_load(content)
        print("✅ Valid YAML!")
        return True
    except yaml.YAMLError as e:
        print(f"❌ Invalid YAML: {e}")
        return False

def yaml_to_json(yaml_content: str) -> str:
    data = yaml.safe_load(yaml_content)
    return json.dumps(data, indent=2, ensure_ascii=False)

def json_to_yaml(json_content: str) -> str:
    data = json.loads(json_content)
    return yaml.dump(data, default_flow_style=False, allow_unicode=True)

def get_nested_value(data: dict, key_path: str, default=None):
    """dot notation: get_nested_value(data, 'database.host')"""
    keys = key_path.split('.')
    current = data
    for key in keys:
        if isinstance(current, dict) and key in current:
            current = current[key]
        else:
            return default
    return current

def deep_merge(base: dict, override: dict) -> dict:
    result = base.copy()
    for key, value in override.items():
        if key in result and isinstance(result[key], dict) and isinstance(value, dict):
            result[key] = deep_merge(result[key], value)
        else:
            result[key] = value
    return result

# ทดสอบ
if __name__ == "__main__":
    yaml_content = """
name: Alice
age: 30
hobbies:
  - Python
  - Kubernetes
  - Photography
address:
  city: Bangkok
  country: Thailand
"""
    data = yaml.safe_load(yaml_content)
    print(f"Name: {data['name']}")
    print(f"Hobbies: {', '.join(data['hobbies'])}")
    print(f"City: {data['address']['city']}")
    
    print("\n--- YAML to JSON ---")
    print(yaml_to_json(yaml_content))
    
    print("\n--- Nested Access ---")
    print(f"City: {get_nested_value(data, 'address.city')}")
    print(f"Missing: {get_nested_value(data, 'address.zip', 'N/A')}")
```

### Common YAML Errors
```yaml
# Error 1: Tabs แทน spaces
bad_indent:
	- item1    # TabError!

# Error 2: Missing space after colon
bad_colon:key: value    # ต้องมี space หลัง :

# Error 3: Duplicate keys
duplicate:
  key: value1
  key: value2    # Parser จะใช้ค่าสุดท้าย แต่บางตัว error

# Error 4: Kubernetes ที่มี error
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app:my-app    # Error: ต้องมี space
    spec:
      containers:
      - name: app
        image: my-image:latest
        ports:
          - containerPort:8080    # Error: ต้องมี space
```

---

## 📊 สรุป Part 1

| หัวข้อ | สิ่งที่เรียน |
|--------|------------|
| YAML คืออะไร | ประวัติ, ความสำคัญ, use cases |
| กฎพื้นฐาน | Indentation, case sensitivity, document separator |
| Scalar Types | String, Integer, Float, Boolean, Null, Date |
| Collections | Mappings (key-value), Sequences (lists) |
| Anchors/Aliases | การ reuse data ด้วย & และ * |
| Comments | # ใช้สำหรับ comment |
| Special Characters | เมื่อไหรต้องใส่ quotes |
| Block Scalars | \| สำหรับ literal, > สำหรับ folded |
| Document Structure | ---, ... สำหรับ document boundary |
| Validation | yamllint, Python yaml module |

---

## 🔗 ต่อไป
- [Part 02: YAML Advanced Syntax](./part-02-yaml-advanced-syntax.md)

---
*Part 01 | Steps 1-10 | ระดับพื้นฐาน*
