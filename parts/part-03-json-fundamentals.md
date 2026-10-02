# Part 03: JSON Fundamentals - พื้นฐาน JSON
## Steps 21-30: เริ่มต้นกับ JSON

---

## 📖 บทนำ

JSON (JavaScript Object Notation) เป็น lightweight data-interchange format ที่ใช้งานกันอย่างแพร่หลายใน APIs, web applications, configuration files, และ databases

### ทำไมต้อง JSON?
- **REST APIs** ใช้ JSON เป็น default format
- **MongoDB, Firebase, CouchDB** เก็บข้อมูลเป็น JSON
- **npm/package.json** ใช้ JSON สำหรับ package management
- **Web Browsers** รองรับ JSON natively
- **แบทุก programming language** มี built-in JSON support

---

## Step 21: JSON Syntax พื้นฐาน

### กฎของ JSON
1. Data เป็น key-value pairs
2. Data คั่นด้วย commas
3. Curly braces `{}` สำหรับ objects
4. Square brackets `[]` สำหรับ arrays
5. Keys ต้องเป็น strings (double quotes เท่านั้น)
6. String values ต้องใช้ double quotes
7. ไม่มี trailing comma (,) หลัง element สุดท้าย
8. ไม่มี comments!

```json
{
  "name": "Alice",
  "age": 30,
  "isActive": true,
  "address": {
    "city": "Bangkok",
    "country": "Thailand"
  },
  "hobbies": ["reading", "coding", "hiking"],
  "spouse": null
}
```

---

## Step 22: JSON Data Types

### String
```json
{
  "simple": "Hello World",
  "with_unicode": "สวัสดีชาวโลก",
  "with_escapes": "Line 1\nLine 2\tTabbed",
  "with_quotes": "He said \"Hello\"",
  "with_backslash": "C:\\Users\\Alice",
  "empty": ""
}
```

### Number
```json
{
  "integer": 42,
  "negative": -17,
  "float": 3.14159,
  "scientific": 6.022e23,
  "zero": 0
}
```

**สิ่งที่ JSON ไม่รองรับ:**
```
Infinity, -Infinity, NaN, 0xFF, 0o77
```

### Boolean
```json
{
  "active": true,
  "deleted": false
}
```

### Null
```json
{
  "optional_field": null
}
```

### Array
```json
{
  "empty_array": [],
  "numbers": [1, 2, 3, 4, 5],
  "mixed": [1, "two", true, null, {"key": "value"}],
  "array_of_objects": [
    {"id": 1, "name": "Alice"},
    {"id": 2, "name": "Bob"}
  ]
}
```

---

## Step 23: JSON vs YAML เปรียบเทียบ

| Feature | JSON | YAML |
|---------|------|------|
| Comments | ❌ ไม่รองรับ | ✅ # |
| Quotes | ✅ Required สำหรับ keys | ❌ Optional |
| Trailing comma | ❌ ไม่รองรับ | N/A |
| Multiline strings | ❌ ต้องใช้ \n | ✅ Block scalars |
| Anchors | ❌ | ✅ & * |
| File size | Larger | Smaller |
| Human readable | Medium | High |
| Parsing speed | Faster | Slower |
| Browser support | ✅ Native | ❌ ต้องใช้ library |

---

## Step 24: JSON String Escape Characters

```json
{
  "newline": "Line 1\nLine 2",
  "tab": "Column 1\tColumn 2",
  "double_quote": "He said \"Hello\"",
  "backslash": "C:\\Program Files\\App",
  "unicode_4": "\u0041",
  "unicode_thai": "\u0E2A\u0E27\u0E31\u0E2A\u0E14\u0E35",
  "unicode_emoji": "\uD83D\uDE00"
}
```

---

## Step 25: JSON Objects ขั้นสูง

```json
{
  "company": {
    "name": "Tech Corp",
    "founded": 2010,
    "address": {
      "headquarters": {
        "country": "Thailand",
        "city": "Bangkok",
        "postal_code": "10110"
      },
      "branches": [
        {"city": "Chiang Mai", "employees": 50},
        {"city": "Phuket", "employees": 30}
      ]
    },
    "departments": {
      "engineering": {
        "head": "Alice Smith",
        "employees": 100,
        "teams": ["frontend", "backend", "devops", "qa"]
      },
      "marketing": {
        "head": "Bob Johnson",
        "employees": 30
      }
    }
  }
}
```

---

## Step 26: JSON Arrays ขั้นสูง

```json
{
  "users": [
    {
      "id": 1,
      "name": "Alice",
      "email": "alice@example.com",
      "role": "admin",
      "permissions": ["read", "write", "delete"],
      "metadata": {
        "last_login": "2024-01-15T08:30:00Z",
        "login_count": 145
      }
    },
    {
      "id": 2,
      "name": "Bob",
      "email": "bob@example.com",
      "role": "user",
      "permissions": ["read"]
    }
  ],
  "pagination": {
    "page": 1,
    "per_page": 20,
    "total": 2
  }
}
```

### Pagination Pattern
```json
{
  "data": [
    {"id": 1, "name": "Item 1"},
    {"id": 2, "name": "Item 2"}
  ],
  "meta": {
    "current_page": 1,
    "last_page": 10,
    "per_page": 10,
    "total": 100
  },
  "links": {
    "first": "https://api.example.com/items?page=1",
    "last": "https://api.example.com/items?page=10",
    "prev": null,
    "next": "https://api.example.com/items?page=2"
  }
}
```

---

## Step 27: JSON API Response Patterns

#### Success Response
```json
{
  "success": true,
  "data": {
    "id": 123,
    "name": "Alice",
    "email": "alice@example.com"
  },
  "message": "User created successfully"
}
```

#### Error Response
```json
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Validation failed",
    "details": [
      {"field": "email", "message": "Invalid email format"},
      {"field": "age", "message": "Must be at least 18"}
    ]
  }
}
```

### JSON:API Standard
```json
{
  "data": {
    "type": "articles",
    "id": "1",
    "attributes": {
      "title": "JSON:API paints my bikeshed!",
      "body": "The shortest article. Ever.",
      "created": "2015-05-22T14:56:29.000Z"
    },
    "relationships": {
      "author": {
        "data": {"id": "42", "type": "people"}
      }
    }
  },
  "included": [
    {
      "type": "people",
      "id": "42",
      "attributes": {"name": "John", "age": 80}
    }
  ]
}
```

---

## Step 28: JSON Schema

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://example.com/user.schema.json",
  "title": "User",
  "type": "object",
  "required": ["id", "name", "email"],
  "properties": {
    "id": {"type": "integer", "minimum": 1},
    "name": {"type": "string", "minLength": 1, "maxLength": 100},
    "email": {"type": "string", "format": "email"},
    "age": {"type": "integer", "minimum": 0, "maximum": 150},
    "role": {
      "type": "string",
      "enum": ["admin", "user", "moderator"],
      "default": "user"
    },
    "tags": {
      "type": "array",
      "items": {"type": "string"},
      "uniqueItems": true
    },
    "address": {
      "type": "object",
      "properties": {
        "street": {"type": "string"},
        "city": {"type": "string"},
        "country": {"type": "string", "pattern": "^[A-Z]{2}$"}
      }
    },
    "created_at": {"type": "string", "format": "date-time"}
  },
  "additionalProperties": false
}
```

### Advanced JSON Schema
```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "Config",
  "type": "object",
  "properties": {
    "server": {
      "type": "object",
      "oneOf": [
        {
          "properties": {
            "type": {"const": "http"},
            "port": {"type": "integer"}
          },
          "required": ["type", "port"]
        },
        {
          "properties": {
            "type": {"const": "https"},
            "port": {"type": "integer"},
            "cert": {"type": "string"},
            "key": {"type": "string"}
          },
          "required": ["type", "port", "cert", "key"]
        }
      ]
    },
    "features": {
      "type": "array",
      "items": {
        "type": "string",
        "enum": ["auth", "logging", "metrics", "caching"]
      },
      "uniqueItems": true
    },
    "limits": {
      "type": "object",
      "properties": {
        "max_connections": {"type": "integer", "minimum": 1, "maximum": 10000},
        "timeout_ms": {"type": "integer", "multipleOf": 100}
      },
      "if": {"properties": {"max_connections": {"minimum": 1000}}},
      "then": {"properties": {"timeout_ms": {"minimum": 5000}}}
    }
  }
}
```

---

## Step 29: JSON Manipulation ด้วย Python

```python
#!/usr/bin/env python3
import json
from typing import Any


def get_value(data: dict, path: str, default: Any = None) -> Any:
    """
    Get value using dot notation: get_value(data, "user.address.city")
    Supports array index: get_value(data, "users[0].name")
    """
    parts = path.split('.')
    current = data
    
    for part in parts:
        if '[' in part:
            key, idx_str = part.split('[', 1)
            idx = int(idx_str.rstrip(']'))
            
            if isinstance(current, dict) and key in current:
                current = current[key]
            else:
                return default
            
            if isinstance(current, list) and 0 <= idx < len(current):
                current = current[idx]
            else:
                return default
        else:
            if isinstance(current, dict) and part in current:
                current = current[part]
            else:
                return default
    
    return current


def flatten_dict(d: dict, separator: str = '.', prefix: str = '') -> dict:
    result = {}
    for key, value in d.items():
        new_key = f"{prefix}{separator}{key}" if prefix else key
        if isinstance(value, dict):
            result.update(flatten_dict(value, separator, new_key))
        else:
            result[new_key] = value
    return result


def deep_diff(a: Any, b: Any, path: str = '') -> list:
    diffs = []
    
    if type(a) != type(b):
        diffs.append({'path': path, 'type': 'type_change', 'from': type(a).__name__, 'to': type(b).__name__})
        return diffs
    
    if isinstance(a, dict):
        all_keys = set(a.keys()) | set(b.keys())
        for key in all_keys:
            current_path = f"{path}.{key}" if path else key
            if key not in a:
                diffs.append({'path': current_path, 'type': 'added', 'value': b[key]})
            elif key not in b:
                diffs.append({'path': current_path, 'type': 'removed', 'value': a[key]})
            else:
                diffs.extend(deep_diff(a[key], b[key], current_path))
    elif isinstance(a, list):
        for i, (item_a, item_b) in enumerate(zip(a, b)):
            diffs.extend(deep_diff(item_a, item_b, f"{path}[{i}]"))
    elif a != b:
        diffs.append({'path': path, 'type': 'value_change', 'from': a, 'to': b})
    
    return diffs


def camel_to_snake(name: str) -> str:
    import re
    s1 = re.sub('(.)([A-Z][a-z]+)', r'\1_\2', name)
    return re.sub('([a-z0-9])([A-Z])', r'\1_\2', s1).lower()


def transform_keys(data: Any, transform_fn) -> Any:
    if isinstance(data, dict):
        return {transform_fn(k): transform_keys(v, transform_fn) for k, v in data.items()}
    elif isinstance(data, list):
        return [transform_keys(item, transform_fn) for item in data]
    return data


if __name__ == "__main__":
    data = {
        "users": [
            {"id": 1, "name": "Alice", "role": "admin"},
            {"id": 2, "name": "Bob", "role": "user"}
        ],
        "config": {
            "server": {"host": "localhost", "port": 8080},
            "database": {"host": "db.example.com", "port": 5432}
        }
    }
    
    print(f"Server host: {get_value(data, 'config.server.host')}")
    print(f"First user: {get_value(data, 'users[0].name')}")
    
    flat = flatten_dict(data['config'])
    print(f"Flattened: {flat}")
    
    camel_data = {"firstName": "Alice", "emailAddress": "alice@example.com"}
    snake_data = transform_keys(camel_data, camel_to_snake)
    print(f"snake_case: {snake_data}")
    
    v1 = {"name": "app", "version": "1.0", "debug": False}
    v2 = {"name": "app", "version": "1.1", "debug": True, "new_feature": "enabled"}
    diffs = deep_diff(v1, v2)
    print(f"Diffs: {diffs}")
```

---

## Step 30: JSON ใน JavaScript/Node.js

```javascript
// JSON Operations in JavaScript/Node.js

const fs = require('fs');

// Parse and Stringify
const parsed = JSON.parse('{"name":"Alice","age":30}');
console.log(parsed.name); // "Alice"

const pretty = JSON.stringify({ name: "Alice", age: 30 }, null, 2);

// Custom reviver
const withReviver = JSON.parse('{"date":"2024-01-15"}', (key, value) => {
  if (key === 'date') return new Date(value);
  return value;
});

// Deep Operations
function deepClone(obj) {
  return JSON.parse(JSON.stringify(obj));
}

function deepMerge(target, source) {
  const result = { ...target };
  for (const key in source) {
    if (source[key] && typeof source[key] === 'object' && !Array.isArray(source[key])) {
      result[key] = deepMerge(result[key] || {}, source[key]);
    } else {
      result[key] = source[key];
    }
  }
  return result;
}

function getNestedValue(obj, path, defaultValue = undefined) {
  const parts = path.split('.');
  let current = obj;
  
  for (const part of parts) {
    if (current === null || current === undefined) return defaultValue;
    const arrayMatch = part.match(/^(\w+)\[(\d+)\]$/);
    if (arrayMatch) {
      const [, key, index] = arrayMatch;
      current = current[key]?.[parseInt(index)];
    } else {
      current = current[part];
    }
  }
  
  return current ?? defaultValue;
}

// Transformation
const camelToSnake = str => str.replace(/[A-Z]/g, letter => `_${letter.toLowerCase()}`);

function transformKeys(obj, transform) {
  if (Array.isArray(obj)) return obj.map(item => transformKeys(item, transform));
  if (obj !== null && typeof obj === 'object') {
    return Object.fromEntries(
      Object.entries(obj).map(([key, value]) => [transform(key), transformKeys(value, transform)])
    );
  }
  return obj;
}

const apiResponse = { userId: 1, firstName: "Alice", emailAddress: "alice@example.com" };
console.log(transformKeys(apiResponse, camelToSnake));
// { user_id: 1, first_name: "Alice", email_address: "..." }


// Validation
function validateJSON(str) {
  try {
    JSON.parse(str);
    return { valid: true };
  } catch (e) {
    return { valid: false, error: e.message };
  }
}

function validateSchema(data, schema) {
  const errors = [];
  
  if (schema.required) {
    for (const field of schema.required) {
      if (!(field in data)) errors.push(`Missing required field: ${field}`);
    }
  }
  
  if (schema.properties) {
    for (const [field, rules] of Object.entries(schema.properties)) {
      if (field in data) {
        const value = data[field];
        if (rules.type && typeof value !== rules.type)
          errors.push(`${field}: expected ${rules.type}, got ${typeof value}`);
        if (rules.minLength && typeof value === 'string' && value.length < rules.minLength)
          errors.push(`${field}: too short (min ${rules.minLength})`);
        if (rules.minimum && typeof value === 'number' && value < rules.minimum)
          errors.push(`${field}: too small (min ${rules.minimum})`);
      }
    }
  }
  
  return { valid: errors.length === 0, errors };
}


// Demo
const config = JSON.parse(`{
  "database": {
    "host": "localhost",
    "port": 5432
  },
  "features": ["auth", "logging"]
}`);

console.log('DB host:', getNestedValue(config, 'database.host'));
console.log('First feature:', getNestedValue(config, 'features[0]'));

const config2 = deepClone(config);
config2.database.host = 'prod-server';
console.log('Original:', config.database.host);  // still "localhost"
console.log('Clone:', config2.database.host);    // "prod-server"

const defaults = { timeout: 30, retries: 3 };
const overrides = { timeout: 60, debug: true };
console.log('Merged:', deepMerge(defaults, overrides));

const schema = {
  required: ['name', 'email'],
  properties: {
    name: { type: 'string', minLength: 1 },
    email: { type: 'string' },
    age: { type: 'number', minimum: 0 }
  }
};

console.log('Valid:', validateSchema({ name: 'Alice', email: 'alice@example.com', age: 30 }, schema));
console.log('Invalid:', validateSchema({ name: '', age: -5 }, schema));
```

---

## 📝 แบบฝึกหัด Part 3

### แบบฝึกหัดที่ 1: สร้าง REST API Response
```json
{
  "success": true,
  "data": {
    "order": {
      "id": "ORD-2024-12345",
      "status": "shipped",
      "customer": {
        "id": 567,
        "name": "สมชาย ใจดี",
        "email": "somchai@example.com"
      },
      "items": [
        {
          "sku": "LAPTOP-001",
          "name": "MacBook Pro 14",
          "quantity": 1,
          "unit_price": 75000.00,
          "total": 75000.00
        }
      ],
      "pricing": {
        "subtotal": 75000.00,
        "tax": 5250.00,
        "total": 80250.00,
        "currency": "THB"
      },
      "created_at": "2024-01-15T09:00:00+07:00"
    }
  }
}
```

### แบบฝึกหัดที่ 2: JSON Schema Validation
```python
import json
import jsonschema

order_schema = {
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "type": "object",
  "required": ["success", "data"],
  "properties": {
    "success": {"type": "boolean"},
    "data": {
      "type": "object",
      "required": ["order"],
      "properties": {
        "order": {
          "type": "object",
          "required": ["id", "status", "customer", "items", "pricing"],
          "properties": {
            "id": {"type": "string", "pattern": "^ORD-\\d{4}-\\d+$"},
            "status": {
              "type": "string",
              "enum": ["pending", "processing", "shipped", "delivered", "cancelled"]
            },
            "items": {
              "type": "array",
              "minItems": 1,
              "items": {
                "type": "object",
                "required": ["sku", "name", "quantity", "unit_price", "total"]
              }
            }
          }
        }
      }
    }
  }
}

try:
  jsonschema.validate(order_data, order_schema)
  print("✅ Valid order!")
except jsonschema.ValidationError as e:
  print(f"❌ Invalid: {e.message}")
```

---

## 📊 สรุป Part 3

| หัวข้อ | สิ่งที่เรียน |
|--------|------------|
| JSON Syntax | กฎพื้นฐาน, ห้ามมี comments/trailing comma |
| Data Types | String, Number, Boolean, Null, Object, Array |
| String Escapes | \n, \t, \", \\, \uXXXX |
| API Patterns | Success/Error responses, Pagination, JSON:API |
| JSON Schema | Validation, required, types, patterns |
| Python JSON | json module, parse, stringify, file ops |
| JavaScript JSON | JSON.parse/stringify, deep operations |
| Transformation | camelCase↔snake_case, flatten/unflatten |

---

## 🔗 ต่อไป
- [Part 04: JSON Schema และ Validation](./part-04-json-schema.md)

---
*Part 03 | Steps 21-30 | ระดับพื้นฐาน*
