# Part 06: YAML/JSON กับ Python
## Steps 51-60: ใช้ Python จัดการ YAML และ JSON

---

## 📖 บทนำ

Python มี built-in support สำหรับ JSON และ library ยอดนิยม PyYAML/ruamel.yaml สำหรับ YAML ทำให้การทำงานกับ data formats เหล่านี้ง่ายมาก

---

## Step 51: การติดตั้ง Libraries

```bash
# PyYAML - library ยอดนิยม
pip install pyyaml

# ruamel.yaml - ดีกว่า PyYAML ในหลายด้าน
# - รองรับ YAML 1.2
# - รักษา comments
# - round-trip parsing
pip install ruamel.yaml

# json - built-in ไม่ต้องติดตั้ง

# jsonschema - validate JSON/YAML
pip install jsonschema

# pydantic - data validation ที่ทรงพลัง
pip install pydantic

# ujson - JSON ที่เร็วกว่า json built-in
pip install ujson

# orjson - เร็วที่สุด รองรับ datetime
pip install orjson
```

---

## Step 52: JSON กับ Python

### json module พื้นฐาน
```python
import json

# === Parse JSON ===
json_string = '{"name": "Alice", "age": 30, "active": true}'
data = json.loads(json_string)
print(data)  # {'name': 'Alice', 'age': 30, 'active': True}
print(type(data))  # <class 'dict'>

# === Serialize to JSON ===
python_obj = {
    "name": "Bob",
    "age": 25,
    "hobbies": ["Python", "Kubernetes"],
    "address": {"city": "Bangkok"}
}

# Compact
compact = json.dumps(python_obj)
print(compact)

# Pretty-printed
pretty = json.dumps(python_obj, indent=2, ensure_ascii=False)
print(pretty)

# Sort keys
sorted_json = json.dumps(python_obj, indent=2, sort_keys=True)
```

### JSON กับ Files
```python
import json

# อ่านจาก file
with open('config.json', 'r', encoding='utf-8') as f:
    config = json.load(f)

# เขียนไปยัง file
with open('output.json', 'w', encoding='utf-8') as f:
    json.dump(config, f, indent=2, ensure_ascii=False)
```

### Custom JSON Encoder/Decoder
```python
import json
from datetime import datetime, date
from decimal import Decimal
from pathlib import Path
from uuid import UUID


class CustomEncoder(json.JSONEncoder):
    """Custom JSON encoder สำหรับ types พิเศษ"""
    
    def default(self, obj):
        if isinstance(obj, datetime):
            return obj.isoformat()
        elif isinstance(obj, date):
            return obj.isoformat()
        elif isinstance(obj, Decimal):
            return float(obj)
        elif isinstance(obj, Path):
            return str(obj)
        elif isinstance(obj, UUID):
            return str(obj)
        elif isinstance(obj, bytes):
            return obj.decode('utf-8')
        elif hasattr(obj, '__dict__'):
            return obj.__dict__
        return super().default(obj)


def custom_decoder(dct):
    """Custom JSON decoder"""
    for key, value in dct.items():
        if isinstance(value, str):
            try:
                dct[key] = datetime.fromisoformat(value)
            except ValueError:
                pass
    return dct


data = {
    "name": "Alice",
    "created_at": datetime.now(),
    "birthdate": date(1994, 5, 15),
    "balance": Decimal("1234.56"),
    "user_id": UUID("550e8400-e29b-41d4-a716-446655440000")
}

json_str = json.dumps(data, cls=CustomEncoder, indent=2)
print(json_str)

decoded = json.loads(json_str, object_hook=custom_decoder)
print(type(decoded['created_at']))  # <class 'datetime.datetime'>
```

### JSON Streaming (Large Files)
```python
import json
from typing import Generator


def read_json_lines(filepath: str) -> Generator[dict, None, None]:
    """อ่าน JSON Lines format (newline-delimited JSON)"""
    with open(filepath, 'r', encoding='utf-8') as f:
        for line in f:
            line = line.strip()
            if line:
                yield json.loads(line)


def process_large_json_array(filepath: str, batch_size: int = 1000):
    """Process large JSON array แบบ streaming ด้วย ijson"""
    try:
        import ijson
    except ImportError:
        print("Install ijson: pip install ijson")
        return
    
    with open(filepath, 'rb') as f:
        parser = ijson.items(f, 'item')
        
        batch = []
        for item in parser:
            batch.append(item)
            if len(batch) >= batch_size:
                process_batch(batch)
                batch = []
        
        if batch:
            process_batch(batch)


def process_batch(batch: list) -> list:
    return [item for item in batch if item.get('active', True)]
```

---

## Step 53: YAML กับ Python (PyYAML)

### PyYAML พื้นฐาน
```python
import yaml

yaml_string = """
name: Alice
age: 30
hobbies:
  - Python
  - Kubernetes
address:
  city: Bangkok
  country: Thailand
"""

# ⚠️ ใช้ safe_load เสมอ! ไม่ใช้ load()
data = yaml.safe_load(yaml_string)
print(data)
print(type(data))  # <class 'dict'>

python_obj = {
    "name": "Bob",
    "servers": [
        {"host": "web-1", "port": 80},
        {"host": "web-2", "port": 80}
    ],
    "database": {
        "host": "db.example.com",
        "port": 5432
    }
}

yaml_output = yaml.dump(python_obj, 
                         default_flow_style=False,
                         allow_unicode=True,
                         sort_keys=False,
                         indent=2)
print(yaml_output)
```

### YAML Safety Issues
```python
import yaml

# ⚠️ DANGER: yaml.load() สามารถ execute arbitrary Python!
dangerous_yaml = """
!!python/object/apply:os.system
- "echo Arbitrary command execution!"
"""

# ❌ อย่าใช้! อันตราย!
# data = yaml.load(dangerous_yaml)  # จะ execute command!

# ✅ ปลอดภัย
try:
    data = yaml.safe_load(dangerous_yaml)
except yaml.YAMLError as e:
    print(f"Safe load refused dangerous YAML: {e}")

# ✅ ใช้ CSafeLoader สำหรับ performance ดีขึ้น
# data = yaml.load(yaml_string, Loader=yaml.CSafeLoader)
```

### YAML กับ Files
```python
import yaml

# อ่านไฟล์ YAML
with open('config.yaml', 'r', encoding='utf-8') as f:
    config = yaml.safe_load(f)

# อ่าน multiple documents
with open('k8s-resources.yaml', 'r', encoding='utf-8') as f:
    documents = list(yaml.safe_load_all(f))
    for doc in documents:
        print(f"Kind: {doc.get('kind')}, Name: {doc.get('metadata', {}).get('name')}")

# เขียนไปยัง file
with open('output.yaml', 'w', encoding='utf-8') as f:
    yaml.dump(config, f, 
              default_flow_style=False,
              allow_unicode=True,
              sort_keys=False)
```

---

## Step 54: ruamel.yaml - YAML กับ Comments

```python
from ruamel.yaml import YAML
import io

yaml_content = """
# Application Configuration
app:
  name: MyApp  # application name
  # Server settings
  server:
    host: localhost
    port: 8080  # HTTP port
  debug: false
"""

ryaml = YAML()
ryaml.preserve_quotes = True

data = ryaml.load(yaml_content)

# แก้ไขค่า
data['app']['server']['port'] = 9090

# เขียนกลับ - comments ยังอยู่!
output = io.StringIO()
ryaml.dump(data, output)
print(output.getvalue())
```

### Round-trip Parsing
```python
from ruamel.yaml import YAML

def update_yaml_preserve_comments(filepath: str, updates: dict):
    """
    อัพเดต YAML file โดยรักษา comments ไว้
    
    Args:
        filepath: path ไปยัง YAML file
        updates: {'app.server.port': 9090}
    """
    ryaml = YAML()
    ryaml.preserve_quotes = True
    
    with open(filepath, 'r') as f:
        data = ryaml.load(f)
    
    for path, value in updates.items():
        keys = path.split('.')
        current = data
        for key in keys[:-1]:
            current = current[key]
        current[keys[-1]] = value
    
    with open(filepath, 'w') as f:
        ryaml.dump(data, f)
    
    print(f"✅ Updated {filepath}")
```

---

## Step 55: YAML/JSON Conversion

```python
import json
import yaml


def yaml_to_json(yaml_input, indent: int = 2) -> str:
    """แปลง YAML เป็น JSON"""
    if isinstance(yaml_input, str):
        data = yaml.safe_load(yaml_input)
    else:
        data = yaml.safe_load(yaml_input)
    
    return json.dumps(data, indent=indent, ensure_ascii=False)


def json_to_yaml(json_input, sort_keys: bool = False) -> str:
    """แปลง JSON เป็น YAML"""
    if isinstance(json_input, str):
        data = json.loads(json_input)
    else:
        data = json.load(json_input)
    
    return yaml.dump(data, 
                     default_flow_style=False,
                     allow_unicode=True,
                     sort_keys=sort_keys,
                     indent=2)


def convert_file(input_file: str, output_file: str):
    """แปลงไฟล์ YAML↔JSON"""
    if input_file.endswith('.yaml') or input_file.endswith('.yml'):
        with open(input_file, 'r') as f:
            result = yaml_to_json(f)
        with open(output_file, 'w') as f:
            f.write(result)
        print(f"Converted YAML → JSON: {input_file} → {output_file}")
    
    elif input_file.endswith('.json'):
        with open(input_file, 'r') as f:
            result = json_to_yaml(f)
        with open(output_file, 'w') as f:
            f.write(result)
        print(f"Converted JSON → YAML: {input_file} → {output_file}")


yaml_example = """
name: MyApp
version: "1.0.0"
database:
  host: localhost
  port: 5432
features:
  - authentication
  - authorization
  - logging
"""

json_output = yaml_to_json(yaml_example)
print("YAML → JSON:")
print(json_output)

yaml_output = json_to_yaml(json_output)
print("\nJSON → YAML:")
print(yaml_output)
```

---

## Step 56: Pydantic สำหรับ Data Validation

```python
from pydantic import BaseModel, Field, validator, root_validator
from typing import Optional, List
from enum import Enum
import yaml
import json


class Environment(str, Enum):
    DEVELOPMENT = "development"
    STAGING = "staging"
    PRODUCTION = "production"


class DatabaseConfig(BaseModel):
    host: str = Field(..., description="Database hostname")
    port: int = Field(5432, ge=1, le=65535)
    name: str = Field(..., min_length=1)
    username: str
    password: str = Field(..., min_length=8)
    pool_size: int = Field(10, ge=1, le=100)


class ServerConfig(BaseModel):
    host: str = "0.0.0.0"
    port: int = Field(8080, ge=1, le=65535)
    workers: int = Field(4, ge=1, le=64)
    timeout: int = Field(30, ge=5)
    debug: bool = False


class AppConfig(BaseModel):
    name: str = Field(..., min_length=1, max_length=100)
    version: str = Field(..., regex=r'^\d+\.\d+\.\d+$')
    environment: Environment = Environment.DEVELOPMENT
    server: ServerConfig = ServerConfig()
    database: DatabaseConfig
    allowed_hosts: List[str] = []
    features: List[str] = []
    
    @root_validator
    def production_requires_allowed_hosts(cls, values):
        env = values.get('environment')
        hosts = values.get('allowed_hosts', [])
        if env == Environment.PRODUCTION and not hosts:
            raise ValueError("Production environment requires allowed_hosts")
        return values


def load_config_from_yaml(yaml_file: str) -> AppConfig:
    """โหลด config จาก YAML และ validate ด้วย Pydantic"""
    with open(yaml_file, 'r') as f:
        raw_config = yaml.safe_load(f)
    
    try:
        config = AppConfig(**raw_config)
        print(f"✅ Config loaded and validated successfully")
        return config
    except Exception as e:
        print(f"❌ Config validation failed: {e}")
        raise


config_yaml = """
name: MyApp
version: "1.0.0"
environment: development
server:
  host: "0.0.0.0"
  port: 8080
  workers: 4
  debug: true
database:
  host: localhost
  port: 5432
  name: myapp_dev
  username: developer
  password: "dev-password-123"
  pool_size: 10
features:
  - authentication
  - logging
"""

try:
    data = yaml.safe_load(config_yaml)
    config = AppConfig(**data)
    print(f"App: {config.name} v{config.version}")
    print(f"Environment: {config.environment}")
    print(f"Server: {config.server.host}:{config.server.port}")
    print(f"DB: {config.database.host}:{config.database.port}/{config.database.name}")
except Exception as e:
    print(f"Error: {e}")
```

---

## Step 57: Kubernetes YAML กับ Python

```python
#!/usr/bin/env python3
"""
Kubernetes YAML Generator
สร้าง Kubernetes manifests ด้วย Python
"""
import yaml
from typing import Optional, List, Dict


def generate_deployment(
    name: str,
    image: str,
    namespace: str = "default",
    replicas: int = 1,
    port: int = 8080,
    labels: Optional[Dict[str, str]] = None,
    env_vars: Optional[List[Dict]] = None,
    cpu_request: str = "100m",
    memory_request: str = "128Mi",
    cpu_limit: str = "500m",
    memory_limit: str = "512Mi"
) -> dict:
    """สร้าง Kubernetes Deployment manifest"""
    
    default_labels = {
        "app": name,
        "app.kubernetes.io/name": name,
        "app.kubernetes.io/managed-by": "python-generator"
    }
    if labels:
        default_labels.update(labels)
    
    deployment = {
        "apiVersion": "apps/v1",
        "kind": "Deployment",
        "metadata": {
            "name": name,
            "namespace": namespace,
            "labels": default_labels
        },
        "spec": {
            "replicas": replicas,
            "selector": {"matchLabels": {"app": name}},
            "strategy": {
                "type": "RollingUpdate",
                "rollingUpdate": {"maxSurge": 1, "maxUnavailable": 0}
            },
            "template": {
                "metadata": {"labels": default_labels},
                "spec": {
                    "containers": [
                        {
                            "name": name,
                            "image": image,
                            "ports": [{"containerPort": port, "protocol": "TCP"}],
                            "resources": {
                                "requests": {"cpu": cpu_request, "memory": memory_request},
                                "limits": {"cpu": cpu_limit, "memory": memory_limit}
                            },
                            "readinessProbe": {
                                "httpGet": {"path": "/healthz", "port": port},
                                "initialDelaySeconds": 10,
                                "periodSeconds": 5
                            },
                            "livenessProbe": {
                                "httpGet": {"path": "/healthz", "port": port},
                                "initialDelaySeconds": 30,
                                "periodSeconds": 30
                            }
                        }
                    ]
                }
            }
        }
    }
    
    if env_vars:
        deployment["spec"]["template"]["spec"]["containers"][0]["env"] = env_vars
    
    return deployment


def generate_service(
    name: str,
    port: int = 80,
    target_port: int = 8080,
    namespace: str = "default",
    service_type: str = "ClusterIP"
) -> dict:
    """สร้าง Kubernetes Service manifest"""
    return {
        "apiVersion": "v1",
        "kind": "Service",
        "metadata": {"name": name, "namespace": namespace, "labels": {"app": name}},
        "spec": {
            "selector": {"app": name},
            "ports": [{"name": "http", "port": port, "targetPort": target_port, "protocol": "TCP"}],
            "type": service_type
        }
    }


def save_manifests(manifests: List[dict], output_file: str):
    """บันทึก manifests ลงไฟล์ YAML"""
    with open(output_file, 'w') as f:
        yaml.dump_all(manifests, f,
                      default_flow_style=False,
                      allow_unicode=True,
                      sort_keys=False)
    print(f"Saved to {output_file}")


if __name__ == "__main__":
    deployment = generate_deployment(
        name="my-api",
        image="my-api:1.0.0",
        namespace="production",
        replicas=3,
        port=8080,
        env_vars=[
            {"name": "APP_ENV", "value": "production"},
            {"name": "DB_HOST", "valueFrom": {"secretKeyRef": {"name": "db-secret", "key": "host"}}}
        ]
    )
    
    service = generate_service(name="my-api", port=80, target_port=8080, namespace="production")
    
    print("=== Deployment ===")
    print(yaml.dump(deployment, default_flow_style=False))
    
    save_manifests([deployment, service], "k8s-manifests.yaml")
```

---

## Step 58: Kubernetes Python Client

```python
#!/usr/bin/env python3
"""
Kubernetes Python Client - interact กับ K8s API
pip install kubernetes
"""
from kubernetes import client, config
from kubernetes.client.rest import ApiException
import yaml


def setup_k8s_client():
    """ตั้งค่า Kubernetes client"""
    try:
        config.load_incluster_config()
        print("Using in-cluster config")
    except config.ConfigException:
        config.load_kube_config()
        print("Using local kubeconfig")


def list_pods(namespace: str = "default") -> list:
    """List pods ใน namespace"""
    v1 = client.CoreV1Api()
    pods = v1.list_namespaced_pod(namespace)
    result = []
    
    for pod in pods.items:
        result.append({
            "name": pod.metadata.name,
            "namespace": pod.metadata.namespace,
            "status": pod.status.phase,
            "ip": pod.status.pod_ip,
            "node": pod.spec.node_name,
        })
    
    return result


def create_deployment_from_yaml(yaml_file: str, namespace: str = "default"):
    """สร้าง Deployment จาก YAML file"""
    with open(yaml_file) as f:
        deployment_data = yaml.safe_load(f)
    
    apps_v1 = client.AppsV1Api()
    
    try:
        resp = apps_v1.create_namespaced_deployment(
            body=deployment_data, namespace=namespace
        )
        print(f"Deployment created: {resp.metadata.name}")
        return resp
    except ApiException as e:
        if e.status == 409:
            resp = apps_v1.patch_namespaced_deployment(
                name=deployment_data['metadata']['name'],
                namespace=namespace,
                body=deployment_data
            )
            print(f"Deployment updated: {resp.metadata.name}")
            return resp
        raise


def scale_deployment(name: str, replicas: int, namespace: str = "default"):
    """Scale deployment"""
    apps_v1 = client.AppsV1Api()
    body = {"spec": {"replicas": replicas}}
    apps_v1.patch_namespaced_deployment(name=name, namespace=namespace, body=body)
    print(f"Scaled {name} to {replicas} replicas")
```

---

## Step 59: Configuration Management Pattern

```python
#!/usr/bin/env python3
import os
import yaml
import json
from pathlib import Path
from typing import Any, Optional
from copy import deepcopy


class Config:
    """
    Layered configuration system
    Priority (high → low):
    1. Environment variables
    2. Environment-specific config file
    3. Base config file
    4. Default values
    """
    
    def __init__(self, config_dir: str = "config"):
        self.config_dir = Path(config_dir)
        self._data = {}
        self._load()
    
    def _load(self):
        base_config = self._load_file("base.yaml") or {}
        self._data = base_config
        
        env = os.getenv("APP_ENV", "development")
        env_config = self._load_file(f"{env}.yaml") or {}
        self._data = self._deep_merge(self._data, env_config)
        
        self._apply_env_vars()
        print(f"Config loaded for environment: {env}")
    
    def _load_file(self, filename: str) -> Optional[dict]:
        for ext in ['yaml', 'yml', 'json']:
            filepath = self.config_dir / filename.replace('.yaml', f'.{ext}')
            if filepath.exists():
                with open(filepath) as f:
                    if ext == 'json':
                        return json.load(f)
                    return yaml.safe_load(f)
        return None
    
    def _deep_merge(self, base: dict, override: dict) -> dict:
        result = deepcopy(base)
        for key, value in override.items():
            if key in result and isinstance(result[key], dict) and isinstance(value, dict):
                result[key] = self._deep_merge(result[key], value)
            else:
                result[key] = deepcopy(value)
        return result
    
    def _apply_env_vars(self):
        """Override config ด้วย env vars: APP__DATABASE__HOST → database.host"""
        prefix = "APP__"
        for key, value in os.environ.items():
            if key.startswith(prefix):
                config_path = key[len(prefix):].lower().split('__')
                self._set_nested(self._data, config_path, value)
    
    def _set_nested(self, data: dict, keys: list, value: str):
        for key in keys[:-1]:
            if key not in data:
                data[key] = {}
            data = data[key]
        data[keys[-1]] = self._parse_value(value)
    
    def _parse_value(self, value: str) -> Any:
        if value.lower() in ('true', 'yes', '1'):
            return True
        if value.lower() in ('false', 'no', '0'):
            return False
        try:
            return int(value)
        except ValueError:
            pass
        try:
            return float(value)
        except ValueError:
            pass
        try:
            return json.loads(value)
        except (json.JSONDecodeError, ValueError):
            pass
        return value
    
    def get(self, path: str, default: Any = None) -> Any:
        keys = path.split('.')
        current = self._data
        for key in keys:
            if isinstance(current, dict) and key in current:
                current = current[key]
            else:
                return default
        return current
```

---

## Step 60: Workshop - Config Management Tool

```python
#!/usr/bin/env python3
import argparse
import sys
import json
import yaml
from pathlib import Path


def cmd_validate(args):
    for filepath in args.files:
        try:
            with open(filepath, 'r') as f:
                content = f.read()
            
            if filepath.endswith('.json'):
                json.loads(content)
                print(f"✅ {filepath}: Valid JSON")
            else:
                yaml.safe_load(content)
                print(f"✅ {filepath}: Valid YAML")
        except (json.JSONDecodeError, yaml.YAMLError) as e:
            print(f"❌ {filepath}: Invalid - {e}")
            sys.exit(1)


def cmd_convert(args):
    with open(args.input, 'r') as f:
        content = f.read()
    
    if args.input.endswith('.json'):
        data = json.loads(content)
    else:
        data = yaml.safe_load(content)
    
    if args.output.endswith('.json'):
        output = json.dumps(data, indent=2, ensure_ascii=False)
    else:
        output = yaml.dump(data, default_flow_style=False, allow_unicode=True)
    
    with open(args.output, 'w') as f:
        f.write(output)
    
    print(f"✅ Converted: {args.input} → {args.output}")


def cmd_get(args):
    with open(args.file, 'r') as f:
        if args.file.endswith('.json'):
            data = json.load(f)
        else:
            data = yaml.safe_load(f)
    
    current = data
    for key in args.path.split('.'):
        if isinstance(current, dict) and key in current:
            current = current[key]
        else:
            print(f"Key not found: {args.path}")
            sys.exit(1)
    
    if args.json:
        print(json.dumps(current, indent=2, ensure_ascii=False))
    else:
        print(current)


def deep_merge(base: dict, override: dict) -> dict:
    from copy import deepcopy
    result = deepcopy(base)
    for key, value in override.items():
        if key in result and isinstance(result[key], dict) and isinstance(value, dict):
            result[key] = deep_merge(result[key], value)
        else:
            result[key] = deepcopy(value)
    return result


def main():
    parser = argparse.ArgumentParser(description='YAML/JSON Config Management Tool')
    subparsers = parser.add_subparsers(dest='command')
    
    validate_parser = subparsers.add_parser('validate', help='Validate YAML/JSON files')
    validate_parser.add_argument('files', nargs='+', help='Files to validate')
    validate_parser.set_defaults(func=cmd_validate)
    
    convert_parser = subparsers.add_parser('convert', help='Convert YAML↔JSON')
    convert_parser.add_argument('input', help='Input file')
    convert_parser.add_argument('output', help='Output file')
    convert_parser.set_defaults(func=cmd_convert)
    
    get_parser = subparsers.add_parser('get', help='Get value by path')
    get_parser.add_argument('file', help='YAML/JSON file')
    get_parser.add_argument('path', help='Dot-notation path')
    get_parser.add_argument('--json', action='store_true', help='Output as JSON')
    get_parser.set_defaults(func=cmd_get)
    
    args = parser.parse_args()
    
    if not args.command:
        parser.print_help()
        sys.exit(1)
    
    args.func(args)


if __name__ == '__main__':
    main()
```

---

## 📊 สรุป Part 6

| หัวข้อ | สิ่งที่เรียน |
|--------|------------|
| json module | loads/dumps, custom encoder/decoder, streaming |
| PyYAML | safe_load/dump, files, custom representer |
| ruamel.yaml | Round-trip, preserve comments |
| Conversion | YAML↔JSON |
| Pydantic | Data validation, type checking |
| K8s Generator | สร้าง manifests ด้วย Python |
| K8s Client | kubernetes library |
| Config System | Layered config management |

---

## 🔗 ต่อไป
- [Part 07: REST API กับ JSON](./part-07-rest-api-json.md)

---
*Part 06 | Steps 51-60 | ระดับพื้นฐาน*
