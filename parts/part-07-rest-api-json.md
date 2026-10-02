# Part 07: REST API Design ด้วย JSON
## Steps 61-70: สร้าง API ระดับ Production

---

## Step 61: REST API Fundamentals

### REST Principles
```
REST (Representational State Transfer) มี 6 หลักการ:

1. Client-Server Separation
   - Frontend และ Backend แยกกัน
   - Communication ผ่าน API เท่านั้น

2. Stateless
   - Server ไม่เก็บ client state ระหว่าง requests
   - ทุก request ต้องมีข้อมูลครบ

3. Cacheable
   - Response ต้องบอกได้ว่า cacheable หรือไม่
   - Cache-Control, ETag headers

4. Uniform Interface
   - Resource-based URLs
   - HTTP methods ตาม semantics
   - Standard representations (JSON, XML)

5. Layered System
   - Client ไม่รู้ว่า connect กับ server ตรง หรือผ่าน proxy

6. Code on Demand (Optional)
   - Server อาจส่ง executable code
```

### HTTP Methods
```
GET    → Read/Retrieve (idempotent, safe)
POST   → Create (not idempotent)
PUT    → Replace entire resource (idempotent)
PATCH  → Partial update (not necessarily idempotent)
DELETE → Delete (idempotent)
HEAD   → Same as GET but no body
OPTIONS→ What methods are allowed

Idempotent = รัน N ครั้งได้ผลเหมือนรัน 1 ครั้ง
```

### Status Codes
```
2xx Success:
200 OK                  → สำเร็จ (GET, PUT, PATCH)
201 Created             → สร้างสำเร็จ (POST)
204 No Content          → สำเร็จแต่ไม่มี response body (DELETE)
206 Partial Content     → Streaming/Range requests

3xx Redirection:
301 Moved Permanently   → URL เปลี่ยนถาวร
302 Found               → Temporary redirect
304 Not Modified        → Cache ยังใช้ได้

4xx Client Errors:
400 Bad Request         → ข้อมูลผิด format
401 Unauthorized        → ยังไม่ได้ authenticate
403 Forbidden           → ไม่มีสิทธิ์
404 Not Found           → Resource ไม่มี
409 Conflict            → State conflict (duplicate)
422 Unprocessable Entity→ Validation error
429 Too Many Requests   → Rate limited

5xx Server Errors:
500 Internal Server Error → Bug ใน server
502 Bad Gateway          → Upstream error
503 Service Unavailable  → Server หยุดชั่วคราว
```

---

## Step 62: URL Design

### Resource Naming
```
# ✓ Good URLs
GET  /users              → list users
GET  /users/123          → get user 123
POST /users              → create user
PUT  /users/123          → replace user 123
PATCH /users/123         → partial update
DELETE /users/123        → delete user 123

# Nested resources
GET /users/123/orders        → user's orders
GET /users/123/orders/456    → specific order
POST /users/123/orders       → create order for user

# Filtering/Sorting/Pagination
GET /users?status=active&sort=name&order=asc&page=1&limit=20
GET /products?category=electronics&price_min=100&price_max=500

# Search
GET /users/search?q=john&fields=name,email

# Actions (non-CRUD)
POST /users/123/activate      → activate user
POST /users/123/reset-password
POST /orders/456/cancel
```

---

## Step 63: JSON Response Formats

### Standard Response
```json
{
  "success": true,
  "data": {
    "id": 123,
    "email": "user@example.com",
    "name": "John Doe",
    "createdAt": "2024-01-15T10:30:00Z"
  },
  "meta": {
    "requestId": "req-abc123",
    "timestamp": "2024-01-15T10:30:01Z"
  }
}
```

### Error Response
```json
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "ข้อมูลไม่ถูกต้อง",
    "details": [
      {
        "field": "email",
        "message": "รูปแบบ email ไม่ถูกต้อง",
        "code": "INVALID_FORMAT"
      }
    ]
  }
}
```

### Pagination Response
```json
{
  "success": true,
  "data": [],
  "pagination": {
    "page": 2,
    "limit": 20,
    "totalItems": 150,
    "totalPages": 8,
    "hasNext": true,
    "hasPrev": true
  }
}
```

---

## Step 64: FastAPI Implementation

```python
from fastapi import FastAPI, HTTPException, Query, Depends
from pydantic import BaseModel, Field, EmailStr
from typing import Optional, List, Any
from datetime import datetime
import uuid, math

app = FastAPI(title="User Management API", version="1.0.0")

class UserCreate(BaseModel):
    email: EmailStr
    name: str = Field(min_length=2, max_length=100)
    role: str = Field(default="user", pattern="^(admin|user|moderator)$")

class UserResponse(BaseModel):
    id: str
    email: str
    name: str
    role: str
    is_active: bool
    created_at: datetime

users_db = {}

def get_db():
    return users_db

@app.get("/users")
async def list_users(
    page: int = Query(default=1, ge=1),
    limit: int = Query(default=20, ge=1, le=100),
    db: dict = Depends(get_db)
):
    all_users = list(db.values())
    total = len(all_users)
    start = (page - 1) * limit
    page_data = all_users[start:start + limit]
    return {
        "success": True,
        "data": page_data,
        "pagination": {
            "page": page, "limit": limit,
            "total_items": total,
            "total_pages": math.ceil(total / limit) if total > 0 else 1
        }
    }

@app.post("/users", status_code=201)
async def create_user(user: UserCreate, db: dict = Depends(get_db)):
    if any(u["email"] == user.email for u in db.values()):
        raise HTTPException(status_code=409, detail={"code": "DUPLICATE_EMAIL"})
    
    user_id = str(uuid.uuid4())
    now = datetime.utcnow()
    user_data = {
        "id": user_id, "email": user.email, "name": user.name,
        "role": user.role, "is_active": True,
        "created_at": now, "updated_at": now
    }
    db[user_id] = user_data
    return {"success": True, "data": user_data}

@app.get("/users/{user_id}")
async def get_user(user_id: str, db: dict = Depends(get_db)):
    if user_id not in db:
        raise HTTPException(status_code=404, detail={"code": "USER_NOT_FOUND"})
    return {"success": True, "data": db[user_id]}

@app.delete("/users/{user_id}", status_code=204)
async def delete_user(user_id: str, db: dict = Depends(get_db)):
    if user_id not in db:
        raise HTTPException(status_code=404, detail={"code": "USER_NOT_FOUND"})
    del db[user_id]
```

---

## Step 65: JWT Authentication

```python
# pip install python-jose[cryptography] passlib[bcrypt]

from datetime import datetime, timedelta
from jose import JWTError, jwt
from passlib.context import CryptContext
from fastapi import Depends, HTTPException
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials

SECRET_KEY = "your-secret-key-change-in-production"
ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE = 30  # minutes

pwd_context = CryptContext(schemes=["bcrypt"])
security = HTTPBearer()

def create_access_token(data: dict):
    to_encode = data.copy()
    expire = datetime.utcnow() + timedelta(minutes=ACCESS_TOKEN_EXPIRE)
    to_encode.update({"exp": expire})
    return jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)

async def get_current_user(
    credentials: HTTPAuthorizationCredentials = Depends(security)
):
    token = credentials.credentials
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        user_id: str = payload.get("sub")
        if not user_id:
            raise HTTPException(status_code=401, detail="Invalid token")
        return {"user_id": user_id, "role": payload.get("role")}
    except JWTError:
        raise HTTPException(status_code=401, detail="Invalid token")

@app.post("/auth/login")
async def login(email: str, password: str):
    user = {"id": "123", "email": email, "role": "user"}
    return {
        "access_token": create_access_token({"sub": user["id"], "role": user["role"]}),
        "token_type": "bearer",
        "expires_in": ACCESS_TOKEN_EXPIRE * 60
    }

@app.get("/me")
async def get_me(current_user = Depends(get_current_user)):
    return {"user": current_user}
```

---

## Step 66: Rate Limiting

```python
# pip install slowapi
from slowapi import Limiter, _rate_limit_exceeded_handler
from slowapi.util import get_remote_address
from slowapi.errors import RateLimitExceeded
from fastapi import Request

limiter = Limiter(key_func=get_remote_address)
app.state.limiter = limiter
app.add_exception_handler(RateLimitExceeded, _rate_limit_exceeded_handler)

@app.get("/public/data")
@limiter.limit("100/minute")
async def public_endpoint(request: Request):
    return {"data": "public"}

@app.get("/api/search")
@limiter.limit("10/minute;1000/hour")
async def search(request: Request, q: str):
    return {"results": []}
```

---

## Step 67: API Versioning

```python
from fastapi import FastAPI, APIRouter

app = FastAPI()

v1_router = APIRouter(prefix="/v1", tags=["v1"])
v2_router = APIRouter(prefix="/v2", tags=["v2"])

@v1_router.get("/users")
async def list_users_v1():
    return {"version": "v1", "data": []}

@v2_router.get("/users")
async def list_users_v2():
    return {
        "version": "v2",
        "data": [],
        "pagination": {"page": 1, "total": 0}
    }

app.include_router(v1_router)
app.include_router(v2_router)
```

---

## Step 68-70: OpenAPI Docs & Testing

```python
from fastapi import FastAPI
from fastapi.openapi.utils import get_openapi

app = FastAPI(
    title="My API",
    description="""API สำหรับจัดการระบบ\n\n## Authentication\nใช้ Bearer token""",
    version="2.0.0"
)

# Custom OpenAPI
def custom_openapi():
    if app.openapi_schema:
        return app.openapi_schema
    openapi_schema = get_openapi(title="My API", version="2.0.0", routes=app.routes)
    openapi_schema["components"]["securitySchemes"] = {
        "BearerAuth": {"type": "http", "scheme": "bearer", "bearerFormat": "JWT"}
    }
    app.openapi_schema = openapi_schema
    return app.openapi_schema

app.openapi = custom_openapi

# Testing with pytest + httpx
import pytest
from httpx import AsyncClient

@pytest.mark.asyncio
async def test_create_user():
    async with AsyncClient(app=app, base_url="http://test") as client:
        response = await client.post("/users", json={
            "email": "test@example.com",
            "name": "Test User",
            "role": "user"
        })
    assert response.status_code == 201
    data = response.json()
    assert data["success"] is True
    assert "id" in data["data"]

# API Client
import httpx
from dataclasses import dataclass

@dataclass
class APIConfig:
    base_url: str
    api_key: str = None

class UserAPIClient:
    def __init__(self, config: APIConfig):
        self.client = httpx.AsyncClient(
            base_url=config.base_url,
            headers={"Authorization": f"Bearer {config.api_key}"} if config.api_key else {}
        )
    
    async def list_users(self, page: int = 1, limit: int = 20) -> dict:
        response = await self.client.get("/users", params={"page": page, "limit": limit})
        response.raise_for_status()
        return response.json()
    
    async def __aenter__(self):
        return self
    
    async def __aexit__(self, *args):
        await self.client.aclose()
```

---

## 📊 สรุป Part 07

| HTTP Method | Action | Status Code |
|------------|--------|-------------|
| GET /resources | List | 200 |
| POST /resources | Create | 201 |
| GET /resources/{id} | Read | 200 |
| PATCH /resources/{id} | Update | 200 |
| DELETE /resources/{id} | Delete | 204 |

---
*Part 07 | Steps 61-70 | ระดับพื้นฐาน*
