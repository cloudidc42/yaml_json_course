# Part 04: JSON Schema - การตรวจสอบข้อมูล
## Steps 31-40: JSON Schema ตั้งแต่พื้นฐานถึงขั้นสูง

---

## Step 31: JSON Schema คืออะไร?

JSON Schema คือ vocabulary ที่ใช้อธิบาย structure ของ JSON data — เหมือน contract ว่าข้อมูลควรมีรูปแบบอย่างไร

### ทำไมต้องใช้ JSON Schema?
```
ปัญหาที่แก้ได้:
1. API response ไม่มี type safety
2. Config files ไม่มี validation
3. Data migration ผิดพลาด
4. Documentation ล้าสมัย

ประโยชน์:
- Auto-validate input/output
- Generate documentation
- IDE autocomplete
- Type-safe code generation
- Contract testing
```

### Schema แรก
```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://example.com/user.schema.json",
  "title": "User",
  "description": "ข้อมูล user ในระบบ",
  "type": "object",
  "required": ["id", "email", "name"],
  "properties": {
    "id": {
      "type": "integer",
      "description": "User ID (auto-increment)",
      "minimum": 1
    },
    "email": {
      "type": "string",
      "format": "email",
      "description": "Email address ที่ unique"
    },
    "name": {
      "type": "string",
      "minLength": 2,
      "maxLength": 100,
      "description": "ชื่อ-นามสกุล"
    },
    "age": {
      "type": "integer",
      "minimum": 0,
      "maximum": 150,
      "description": "อายุ (optional)"
    },
    "role": {
      "type": "string",
      "enum": ["admin", "user", "moderator"],
      "default": "user"
    }
  },
  "additionalProperties": false
}
```

---

## Step 32: Types และ Validation Keywords

### Primitive Types
```json
{
  "type": "string",    // "hello"
  "type": "number",   // 42, 3.14
  "type": "integer",  // 42 (ไม่รับ 3.14)
  "type": "boolean",  // true, false
  "type": "null",     // null
  "type": "array",    // [1, 2, 3]
  "type": "object"    // {"key": "value"}
}
```

### String Validations
```json
{
  "type": "string",
  "minLength": 1,
  "maxLength": 255,
  "pattern": "^[a-zA-Z0-9_]+$",
  "format": "email"
}
```

### Format ที่ built-in
```
"date"       → "2024-01-15"
"time"        → "14:30:00"
"date-time"   → "2024-01-15T14:30:00Z"
"email"       → "user@example.com"
"hostname"    → "example.com"
"ipv4"        → "192.168.1.1"
"uri"         → "https://example.com"
"uuid"        → "550e8400-e29b-41d4-a716-446655440000"
```

### Number Validations
```json
{
  "type": "number",
  "minimum": 0,
  "maximum": 100,
  "exclusiveMinimum": 0,
  "exclusiveMaximum": 100,
  "multipleOf": 5
}
```

### Array Validations
```json
{
  "type": "array",
  "items": { "type": "string" },
  "minItems": 1,
  "maxItems": 10,
  "uniqueItems": true
}
```

### Object Validations
```json
{
  "type": "object",
  "properties": {
    "name": { "type": "string" },
    "age": { "type": "integer" }
  },
  "required": ["name"],
  "additionalProperties": false,
  "minProperties": 1,
  "maxProperties": 10,
  "propertyNames": {
    "pattern": "^[a-z_]+$"
  }
}
```

---

## Step 33: Combining Schemas

### allOf - ต้องตรงทุก schema
```json
{
  "allOf": [
    { "type": "object", "required": ["id"] },
    { "type": "object", "required": ["name"] }
  ]
}
```

### anyOf - ตรงอย่างน้อยหนึ่ง schema
```json
{
  "anyOf": [
    { "type": "string" },
    { "type": "number" },
    { "type": "boolean" }
  ]
}
```

### oneOf - ตรงแค่หนึ่ง schema เท่านั้น
```json
{
  "oneOf": [
    {
      "type": "object",
      "properties": {
        "type": { "const": "email" },
        "address": { "type": "string", "format": "email" }
      },
      "required": ["type", "address"]
    },
    {
      "type": "object",
      "properties": {
        "type": { "const": "phone" },
        "number": { "type": "string", "pattern": "^\\+?[0-9]{10,15}$" }
      },
      "required": ["type", "number"]
    }
  ]
}
```

### not - ต้องไม่ตรง schema
```json
{
  "not": {
    "type": "null"
  }
}
```

---

## Step 34: $ref และ Reusable Definitions

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  
  "$defs": {
    "Address": {
      "type": "object",
      "required": ["street", "city", "country"],
      "properties": {
        "street": { "type": "string" },
        "city": { "type": "string" },
        "country": {
          "type": "string",
          "minLength": 2,
          "maxLength": 2
        },
        "zipCode": {
          "type": "string",
          "pattern": "^[0-9]{5}(-[0-9]{4})?$"
        }
      }
    },
    "Money": {
      "type": "object",
      "required": ["amount", "currency"],
      "properties": {
        "amount": {
          "type": "number",
          "minimum": 0,
          "multipleOf": 0.01
        },
        "currency": {
          "type": "string",
          "enum": ["USD", "EUR", "THB", "JPY"]
        }
      }
    }
  },
  
  "type": "object",
  "properties": {
    "billingAddress": { "$ref": "#/$defs/Address" },
    "shippingAddress": { "$ref": "#/$defs/Address" },
    "total": { "$ref": "#/$defs/Money" }
  }
}
```

---

## Step 35: Conditional Schemas

### if/then/else
```json
{
  "type": "object",
  "properties": {
    "country": { "type": "string" },
    "zipCode": { "type": "string" },
    "province": { "type": "string" }
  },
  "required": ["country"],
  
  "if": {
    "properties": {
      "country": { "const": "TH" }
    }
  },
  "then": {
    "properties": {
      "zipCode": {
        "pattern": "^[0-9]{5}$"
      }
    },
    "required": ["zipCode", "province"]
  },
  "else": {
    "properties": {
      "zipCode": {
        "pattern": "^[A-Z0-9 -]{3,10}$"
      }
    }
  }
}
```

### dependentRequired
```json
{
  "type": "object",
  "properties": {
    "credit_card": { "type": "string" },
    "billing_address": { "type": "string" }
  },
  "dependentRequired": {
    "credit_card": ["billing_address"]
  }
}
```

---

## Step 36: Schema สำหรับ API

### Request Schema
```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "CreateProductRequest",
  "type": "object",
  "required": ["name", "price", "category"],
  "properties": {
    "name": {
      "type": "string",
      "minLength": 3,
      "maxLength": 200
    },
    "price": {
      "type": "number",
      "minimum": 0,
      "exclusiveMinimum": 0,
      "multipleOf": 0.01
    },
    "category": {
      "type": "string",
      "enum": ["electronics", "clothing", "food", "books", "other"]
    },
    "stock": {
      "type": "integer",
      "minimum": 0,
      "default": 0
    },
    "tags": {
      "type": "array",
      "items": { "type": "string", "minLength": 1, "maxLength": 50 },
      "maxItems": 10,
      "uniqueItems": true
    }
  },
  "additionalProperties": false
}
```

---

## Step 37: Python jsonschema Library

```python
# pip install jsonschema

import json
import jsonschema
from jsonschema import validate, ValidationError, Draft202012Validator

user_schema = {
    "$schema": "https://json-schema.org/draft/2020-12/schema",
    "type": "object",
    "required": ["id", "email", "name"],
    "properties": {
        "id": {"type": "integer", "minimum": 1},
        "email": {"type": "string", "format": "email"},
        "name": {"type": "string", "minLength": 2, "maxLength": 100},
        "role": {
            "type": "string",
            "enum": ["admin", "user", "moderator"],
            "default": "user"
        }
    },
    "additionalProperties": False
}

# Validate
valid_user = {"id": 1, "email": "user@example.com", "name": "John Doe", "role": "user"}
try:
    validate(instance=valid_user, schema=user_schema)
    print("✓ Valid user data")
except ValidationError as e:
    print(f"✗ {e.message}")

# Collect all errors
validator = Draft202012Validator(user_schema)
errors = list(validator.iter_errors({"id": "wrong", "email": "bad"}))
for error in errors:
    print(f"  - {'.'.join(str(p) for p in error.path)}: {error.message}")
```

### Schema Registry Pattern
```python
import jsonschema
from typing import Any, Dict

class SchemaRegistry:
    def __init__(self):
        self._schemas: Dict[str, dict] = {}
    
    def register(self, name: str, schema: dict):
        self._schemas[name] = schema
    
    def validate(self, name: str, data: Any) -> tuple[bool, list[str]]:
        if name not in self._schemas:
            raise ValueError(f"Schema '{name}' not found")
        validator = jsonschema.Draft202012Validator(self._schemas[name])
        errors = list(validator.iter_errors(data))
        if not errors:
            return True, []
        messages = [
            f"{'.'.join(str(p) for p in e.path) or 'root'}: {e.message}"
            for e in errors
        ]
        return False, messages

registry = SchemaRegistry()
registry.register("user", {
    "type": "object",
    "required": ["email", "name"],
    "properties": {
        "email": {"type": "string", "format": "email"},
        "name": {"type": "string", "minLength": 2}
    }
})

is_valid, errors = registry.validate("user", {"email": "bad", "name": "J"})
print(f"Valid: {is_valid}")
for err in errors:
    print(f"  - {err}")
```

---

## Step 38: JSON Schema ใน FastAPI

```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel, Field, EmailStr
from typing import Optional, List
from datetime import datetime
from decimal import Decimal
from enum import Enum
import uuid

app = FastAPI(title="Product API", version="1.0.0")

class CategoryEnum(str, Enum):
    ELECTRONICS = "electronics"
    CLOTHING = "clothing"
    FOOD = "food"
    BOOKS = "books"

class ProductCreate(BaseModel):
    name: str = Field(min_length=3, max_length=200)
    description: Optional[str] = Field(None, max_length=5000)
    price: Decimal = Field(gt=0, decimal_places=2)
    category: CategoryEnum
    stock: int = Field(default=0, ge=0)
    tags: List[str] = Field(default=[])

class ProductResponse(BaseModel):
    id: str = Field(default_factory=lambda: str(uuid.uuid4()))
    name: str
    price: Decimal
    category: str
    stock: int
    created_at: datetime = Field(default_factory=datetime.utcnow)

products_db = []

@app.post("/products", response_model=ProductResponse, status_code=201)
async def create_product(product: ProductCreate):
    new_product = ProductResponse(
        name=product.name,
        price=product.price,
        category=product.category,
        stock=product.stock
    )
    products_db.append(new_product)
    return new_product

@app.get("/products/{product_id}", response_model=ProductResponse)
async def get_product(product_id: str):
    for product in products_db:
        if product.id == product_id:
            return product
    raise HTTPException(status_code=404, detail="Product not found")
```

---

## Step 39: JSON Schema Generation

```python
from pydantic import BaseModel
import json

class PydanticUser(BaseModel):
    id: int
    email: str
    name: str

# Export JSON Schema
schema = PydanticUser.model_json_schema()
print(json.dumps(schema, indent=2))
# {
#   "title": "PydanticUser",
#   "type": "object",
#   "properties": {
#     "id": {"title": "Id", "type": "integer"},
#     "email": {"title": "Email", "type": "string"},
#     "name": {"title": "Name", "type": "string"}
#   },
#   "required": ["id", "email", "name"]
# }
```

---

## Step 40: Schema Testing

```python
import pytest
import jsonschema
import json
from pathlib import Path

schema_path = Path("schemas/user.schema.json")
with open(schema_path) as f:
    USER_SCHEMA = json.load(f)

class TestUserSchema:
    def test_valid_minimal_user(self):
        user = {"id": 1, "email": "test@example.com", "name": "Test User"}
        jsonschema.validate(user, USER_SCHEMA)
    
    def test_missing_required_field(self):
        user = {"id": 1, "name": "Test User"}  # missing email
        with pytest.raises(jsonschema.ValidationError) as exc:
            jsonschema.validate(user, USER_SCHEMA)
        assert "email" in exc.value.message
    
    def test_invalid_email_format(self):
        user = {"id": 1, "email": "not-an-email", "name": "Test User"}
        with pytest.raises(jsonschema.ValidationError):
            jsonschema.validate(user, USER_SCHEMA,
                format_checker=jsonschema.FormatChecker())
    
    def test_invalid_role(self):
        user = {"id": 1, "email": "test@example.com",
                "name": "Test", "role": "superadmin"}
        with pytest.raises(jsonschema.ValidationError) as exc:
            jsonschema.validate(user, USER_SCHEMA)
        assert "superadmin" in exc.value.message

# Property-based testing
from hypothesis import given, strategies as st

@given(
    user_id=st.integers(min_value=1),
    name=st.text(min_size=2, max_size=100)
)
def test_random_valid_users(user_id, name):
    user = {"id": user_id, "email": "test@example.com", "name": name}
    try:
        jsonschema.validate(user, USER_SCHEMA)
    except jsonschema.ValidationError as e:
        pytest.fail(f"Valid user rejected: {e.message}")
```

---

## 📊 สรุป Part 04

| Concept | ใช้สำหรับ |
|---------|----------|
| type | กำหนด data type |
| required | บังคับ fields |
| properties | define fields |
| pattern | regex validation |
| format | semantic validation |
| $ref | reuse schemas |
| if/then/else | conditional |
| oneOf/anyOf/allOf | composition |

---

## 🔗 ต่อไป
- [Part 05: YAML/JSON Tools](./part-05-yaml-json-tools.md)

---
*Part 04 | Steps 31-40 | ระดับพื้นฐาน*
