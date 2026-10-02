# Part 05: YAML/JSON Tools & Validation
## Steps 41-50: เครื่องมือที่ต้องรู้จัก

---

## Step 41: yamllint - ตรวจสอบ YAML

```bash
pip install yamllint
brew install yamllint  # macOS
apt-get install yamllint  # Ubuntu
```

```bash
# ตรวจไฟล์เดียว
yamllint config.yaml

# ตรวจ directory
yamllint .

# รูปแบบ output
yamllint -f parsable config.yaml   # machine-readable
yamllint -f github config.yaml     # GitHub Actions
```

### Custom Configuration
```yaml
# .yamllint.yaml
---
extends: default

rules:
  line-length:
    max: 120
    level: warning
  indentation:
    spaces: 2
    indent-sequences: true
  key-duplicates: enable
  quoted-strings:
    quote-type: double
    required: only-when-needed
  trailing-spaces: enable
  empty-lines:
    max: 1
  truthy:
    allowed-values: ["true", "false", "yes", "no"]
    check-keys: false
  document-start:
    present: always
```

### Pre-commit Hook
```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/adrienverge/yamllint
    rev: v1.32.0
    hooks:
      - id: yamllint
        args: ["-c", ".yamllint.yaml"]
```

---

## Step 42: jq - JSON Command Line Tool

```bash
brew install jq        # macOS
apt-get install jq     # Ubuntu
```

### Basic Usage
```bash
cat data.json | jq '.'
cat data.json | jq '.name'
cat data.json | jq '.user.email'
cat data.json | jq '.items[0]'
cat data.json | jq -r '.name'  # raw string
```

### Filters
```bash
# Map
echo '{"items": [1,2,3]}' | jq '.items | map(. * 2)'
# [2,4,6]

# Select/filter
echo '{"users": [{"name":"A","active":true},{"name":"B","active":false}]}' | \
  jq '.users[] | select(.active == true)'

# keys, values
echo '{"a":1,"b":2}' | jq 'keys'    # ["a","b"]
echo '{"a":1,"b":2}' | jq 'to_entries'
# [{"key":"a","value":1},{"key":"b","value":2}]

# Conditional
echo '{"score": 85}' | jq 'if .score >= 90 then "A" elif .score >= 80 then "B" else "C" end'

# Sort and unique
echo '[3,1,4,1,5]' | jq 'sort | unique'
# [1,3,4,5]

# Group by
echo '[{"type":"A","v":1},{"type":"B","v":2},{"type":"A","v":3}]' | \
  jq 'group_by(.type) | map({type: .[0].type, values: map(.v)})'
```

### jq Script
```bash
#!/bin/bash
API_RESPONSE=$(cat api_response.json)

echo "Total users: $(echo "$API_RESPONSE" | jq '.users | length')"
echo "Active: $(echo "$API_RESPONSE" | jq '[.users[] | select(.active)] | length')"
echo "$API_RESPONSE" | jq -r '.users | sort_by(.last_login) | reverse | .[0:5] | .[] | "\(.name) - \(.last_login)"'
```

---

## Step 43: yq - YAML Processor

```bash
# ติดตั้ง
wget https://github.com/mikefarah/yq/releases/download/v4.40.5/yq_linux_amd64 -O /usr/bin/yq
chmod +x /usr/bin/yq
brew install yq  # macOS
```

```bash
# Read YAML
yq '.name' config.yaml

# Write/Update inplace
yq -i '.version = "2.0"' config.yaml
yq -i 'del(.oldField)' config.yaml

# Merge
yq '. *= load("override.yaml")' base.yaml

# YAML → JSON
yq -o=json '.' config.yaml > config.json

# JSON → YAML
yq -P '.' data.json > data.yaml

# Kubernetes YAML
yq '.spec.template.spec.containers[].image' deployment.yaml
yq -i '.spec.template.spec.containers[0].image = "myapp:v2.0"' deployment.yaml
```

---

## Step 44: Python Libraries

### PyYAML
```python
import yaml

with open("config.yaml") as f:
    config = yaml.safe_load(f)

data = {"name": "test", "version": 1}
yaml_str = yaml.dump(data, default_flow_style=False, allow_unicode=True)
```

### ruamel.yaml (Preserves comments)
```python
from ruamel.yaml import YAML

yaml = YAML()
yaml.preserve_quotes = True

with open("config.yaml") as f:
    data = yaml.load(f)

data["version"] = "2.0"
with open("config.yaml", "w") as f:
    yaml.dump(data, f)
```

### orjson (Fast JSON)
```python
import orjson
from datetime import datetime
from decimal import Decimal

data = {"timestamp": datetime.now(), "price": Decimal("99.99")}
json_bytes = orjson.dumps(data, option=orjson.OPT_INDENT_2)
obj = orjson.loads(json_bytes)
# orjson เร็วกว่า json มาตรฐาน 3-10x
```

---

## Step 45: JSON Patch

```python
# pip install jsonpatch
import jsonpatch, json

original = {"name": "John", "age": 30, "hobbies": ["reading"]}

patch = jsonpatch.JsonPatch([
    {"op": "replace", "path": "/name", "value": "Jane"},
    {"op": "add", "path": "/email", "value": "jane@example.com"},
    {"op": "remove", "path": "/age"},
    {"op": "add", "path": "/hobbies/-", "value": "gaming"}
])

result = patch.apply(original)
print(json.dumps(result, indent=2))

# Generate patch from diff
patch = jsonpatch.make_patch(original, result)
print(patch.to_string())
```

### JSON Merge Patch (RFC 7396)
```python
from copy import deepcopy

def merge_patch(target: dict, patch: dict) -> dict:
    result = deepcopy(target)
    for key, value in patch.items():
        if value is None:
            result.pop(key, None)  # null = delete
        elif isinstance(value, dict) and isinstance(result.get(key), dict):
            result[key] = merge_patch(result[key], value)
        else:
            result[key] = value
    return result

original = {"a": "foo", "b": "bar", "c": {"x": 1, "y": 2}, "d": [1, 2, 3]}
patch = {"b": "baz", "c": {"x": 10}, "d": None, "e": "new"}
result = merge_patch(original, patch)
# {"a": "foo", "b": "baz", "c": {"x": 10, "y": 2}, "e": "new"}
```

---

## Step 46: VS Code Extensions

```
Essential Extensions:
1. YAML (redhat.vscode-yaml) - Schema validation, Autocomplete
2. Prettier (esbenp.prettier-vscode) - Format JSON/YAML
3. REST Client (humao.rest-client) - Test APIs
```

```json
// settings.json
{
  "yaml.schemas": {
    "https://json.schemastore.org/github-workflow.json": ".github/workflows/*.yaml",
    "https://json.schemastore.org/docker-compose.json": "docker-compose*.yaml",
    "https://json.schemastore.org/kubernetes": "k8s/**/*.yaml"
  },
  "yaml.validate": true
}
```

---

## Step 47: Prettier

```bash
npm install -g prettier
prettier --write "**/*.json"
prettier --write "**/*.yaml"
prettier --check "**/*.json"  # CI check
```

```json
// .prettierrc
{
  "tabWidth": 2,
  "printWidth": 120,
  "overrides": [
    { "files": "*.yaml", "options": { "tabWidth": 2 } },
    { "files": "*.json", "options": { "printWidth": 80 } }
  ]
}
```

---

## Step 48: JSON/YAML ใน CI/CD

```yaml
name: Validate YAML/JSON

on:
  pull_request:
    paths: ['**/*.yaml', '**/*.json']

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Lint YAML
        run: |
          pip install yamllint
          yamllint -c .yamllint.yaml .
      - name: Check JSON
        run: |
          python -c "
import json, glob, sys
errors = []
for f in glob.glob('**/*.json', recursive=True):
    try:
        json.load(open(f))
    except Exception as e:
        errors.append(f'{f}: {e}')
if errors:
    for e in errors: print(f'✗ {e}')
    sys.exit(1)
print(f'✓ All JSON files valid')
"
      - name: Validate K8s YAML
        run: |
          curl -L https://github.com/yannh/kubeconform/releases/download/v0.6.3/kubeconform-linux-amd64.tar.gz | tar xz
          ./kubeconform -summary k8s/
```

---

## Step 49: JSON Streaming

```python
# pip install ijson
import ijson

def process_large_json(file_path: str, max_records: int = 1000):
    count = 0
    with open(file_path, 'rb') as f:
        for item in ijson.items(f, 'item'):
            print(f"Processing: {item.get('id', 'unknown')}")
            count += 1
            if count >= max_records:
                break
    return count
```

### JSON Lines (ndjson)
```python
import json

def read_jsonl(file_path: str):
    with open(file_path) as f:
        for line_num, line in enumerate(f, 1):
            line = line.strip()
            if not line:
                continue
            try:
                yield json.loads(line)
            except json.JSONDecodeError as e:
                print(f"Error on line {line_num}: {e}")

def write_jsonl(file_path: str, records: list):
    with open(file_path, 'w') as f:
        for record in records:
            f.write(json.dumps(record, ensure_ascii=False) + '\n')
```

---

## Step 50: Best Practices

### YAML Best Practices
```yaml
# ✓ Good YAML
---
application:
  name: my-app
  version: "1.2.3"  # Quote versions

database:
  host: localhost
  port: 5432
  password: ${DB_PASSWORD}  # Use env vars for secrets

logging:
  level: info
  format: json
```

```yaml
# ✗ Bad YAML
application: {name: my-app, version: 1.2}  # Flow style
DB_PASSWORD: secret123  # Hardcoded secret
name: yes  # yes = true!
```

### JSON Best Practices
```json
{
  "data": {
    "id": "user-123",
    "createdAt": "2024-01-15T10:30:00Z"
  },
  "meta": { "requestId": "req-abc123" }
}
```

### Naming Conventions
```
JSON:   camelCase     → userId, firstName, createdAt
YAML:   snake_case    → user_id, first_name, created_at
K8s:    kebab-case    → my-deployment, config-map
```

---

## 📊 สรุป Part 05

| Tool | ใช้สำหรับ | Command |
|------|----------|---------|
| yamllint | Lint YAML | `yamllint config.yaml` |
| jq | Process JSON | `cat file.json \| jq '.'` |
| yq | Process YAML | `yq '.key' file.yaml` |
| prettier | Format | `prettier --write "**/*.json"` |
| jsonschema | Validate | Python library |

---
*Part 05 | Steps 41-50 | ระดับพื้นฐาน*
