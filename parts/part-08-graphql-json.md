# Part 08: GraphQL และ JSON
## Steps 71-80: GraphQL API Development

---

## Step 71: GraphQL คืออะไร?

### เปรียบเทียบ REST vs GraphQL
```
REST:
  GET /users/123               → ได้ user ทั้งหมด (over-fetching)
  GET /users/123/posts         → ต้องเรียก 2 endpoints
  GET /users/123/friends       → ต้องเรียก 3 endpoints

GraphQL:
  query {
    user(id: "123") {
      name
      email
      posts(limit: 5) { title }
      friends { name }
    }
  }
  → ได้ทุกอย่างใน 1 request
```

### JSON ใน GraphQL
```json
{
  "query": "query GetUser($id: ID!) { user(id: $id) { name email } }",
  "variables": { "id": "123" },
  "operationName": "GetUser"
}
```

---

## Step 72: GraphQL Schema Definition Language

```graphql
scalar DateTime
scalar JSON

enum UserRole { ADMIN USER MODERATOR }

type User {
  id: ID!
  email: String!
  name: String!
  role: UserRole!
  isActive: Boolean!
  createdAt: DateTime!
  posts: [Post!]!
}

type Post {
  id: ID!
  title: String!
  content: String!
  published: Boolean!
  author: User!
  createdAt: DateTime!
}

input CreateUserInput {
  email: String!
  name: String!
  role: UserRole = USER
}

type PageInfo {
  page: Int!
  totalItems: Int!
  totalPages: Int!
  hasNext: Boolean!
}

type UserConnection {
  nodes: [User!]!
  pageInfo: PageInfo!
}

type Query {
  user(id: ID!): User
  users(page: Int = 1, limit: Int = 20, role: UserRole): UserConnection!
  me: User
}

type Mutation {
  createUser(input: CreateUserInput!): User!
  deleteUser(id: ID!): Boolean!
  login(email: String!, password: String!): AuthPayload!
}

type Subscription {
  userCreated: User!
  messageAdded(channelId: ID!): Message!
}

union SearchResult = User | Post
```

---

## Step 73: Strawberry Python

```python
# pip install strawberry-graphql[fastapi]

import strawberry
from strawberry.fastapi import GraphQLRouter
from fastapi import FastAPI
from typing import Optional, List
from datetime import datetime
from enum import Enum

@strawberry.enum
class UserRole(Enum):
    ADMIN = "admin"
    USER = "user"
    MODERATOR = "moderator"

@strawberry.type
class User:
    id: strawberry.ID
    email: str
    name: str
    role: UserRole
    is_active: bool
    created_at: datetime

@strawberry.input
class CreateUserInput:
    email: str
    name: str
    role: UserRole = UserRole.USER

_users_db = {}

@strawberry.type
class Query:
    @strawberry.field
    async def user(self, id: strawberry.ID) -> Optional[User]:
        u = _users_db.get(id)
        return User(**u) if u else None
    
    @strawberry.field
    async def me(self, info: strawberry.types.Info) -> Optional[User]:
        user_id = info.context.get("user_id")
        if not user_id:
            return None
        u = _users_db.get(user_id)
        return User(**u) if u else None

@strawberry.type
class Mutation:
    @strawberry.mutation
    async def create_user(self, input: CreateUserInput) -> User:
        import uuid
        if any(u["email"] == input.email for u in _users_db.values()):
            raise Exception("Email already exists")
        user_id = str(uuid.uuid4())
        user_data = {
            "id": user_id, "email": input.email, "name": input.name,
            "role": input.role, "is_active": True, "created_at": datetime.utcnow()
        }
        _users_db[user_id] = user_data
        return User(**user_data)
    
    @strawberry.mutation
    async def delete_user(self, id: strawberry.ID) -> bool:
        if id not in _users_db:
            raise Exception(f"User {id} not found")
        del _users_db[id]
        return True

schema = strawberry.Schema(query=Query, mutation=Mutation)
graphql_router = GraphQLRouter(schema)

app = FastAPI(title="GraphQL API")
app.include_router(graphql_router, prefix="/graphql")
```

---

## Step 74-75: Queries & Mutations

```graphql
# Basic query
query GetUser($userId: ID!) {
  user(id: $userId) {
    id name email role
    posts { title published }
  }
}

# Fragments
fragment UserFields on User {
  id name email role
}

query GetUsers {
  users {
    nodes { ...UserFields }
    pageInfo { page totalItems }
  }
}

# Inline fragments (unions)
query Search {
  search(query: "john") {
    ... on User { name email }
    ... on Post { title content }
  }
}

# Mutations
mutation CreateUserMutation($input: CreateUserInput!) {
  createUser(input: $input) {
    id name email
  }
}
```

---

## Step 76: N+1 Problem และ DataLoader

```python
# pip install aiodataloader
from aiodataloader import DataLoader
from typing import List, Any

class UserLoader(DataLoader):
    async def batch_load_fn(self, user_ids: List[str]) -> List[Any]:
        # Batch load แทน N individual queries
        users = await load_users_by_ids(user_ids)
        user_map = {u.id: u for u in users}
        return [user_map.get(uid) for uid in user_ids]

async def load_users_by_ids(ids: List[str]) -> List:
    # SELECT * FROM users WHERE id IN (...)
    return [
        User(id=uid, name=f"User {uid}", email=f"{uid}@test.com",
             role=UserRole.USER, is_active=True, created_at=datetime.utcnow())
        for uid in ids
    ]

from strawberry.fastapi import BaseContext
from fastapi import Request

class CustomContext(BaseContext):
    def __init__(self, request: Request):
        super().__init__()
        self.request = request
        self.user_loader = UserLoader()

async def get_context(request: Request) -> CustomContext:
    return CustomContext(request=request)

@strawberry.type
class Post:
    id: strawberry.ID
    title: str
    author_id: str
    
    @strawberry.field
    async def author(self, info: strawberry.types.Info) -> User:
        return await info.context.user_loader.load(self.author_id)

schema = strawberry.Schema(query=Query, mutation=Mutation)
graphql_router = GraphQLRouter(schema, context_getter=get_context)
```

---

## Step 77: GraphQL Subscriptions

```python
import asyncio
from typing import AsyncGenerator

_subscribers = {}

@strawberry.type
class Message:
    id: str
    content: str
    channel_id: str
    created_at: datetime

@strawberry.type
class Subscription:
    @strawberry.subscription
    async def message_added(self, channel_id: str) -> AsyncGenerator[Message, None]:
        queue = asyncio.Queue()
        _subscribers.setdefault(channel_id, []).append(queue)
        try:
            while True:
                message = await queue.get()
                yield message
        finally:
            _subscribers[channel_id].remove(queue)

@strawberry.type
class MutationWithSub:
    @strawberry.mutation
    async def send_message(self, channel_id: str, content: str) -> Message:
        import uuid
        msg = Message(
            id=str(uuid.uuid4()), content=content,
            channel_id=channel_id, created_at=datetime.utcnow()
        )
        for queue in _subscribers.get(channel_id, []):
            await queue.put(msg)
        return msg
```

---

## Step 78-79: Directives & Python Client

```graphql
# Built-in directives
query GetUser($showEmail: Boolean = true) {
  user(id: "123") {
    name
    email @include(if: $showEmail)
    role @skip(if: false)
  }
}
```

```python
# pip install gql[httpx]
from gql import gql, Client
from gql.transport.httpx import HTTPXTransport

transport = HTTPXTransport(
    url="http://localhost:8000/graphql",
    headers={"Authorization": "Bearer token123"}
)

async def graphql_query():
    async with Client(transport=transport) as session:
        query = gql("""
            query GetUser($id: ID!) {
                user(id: $id) { id name email role }
            }
        """)
        result = await session.execute(query, variable_values={"id": "123"})
        return result

# Subscription
from gql.transport.websockets import WebsocketsTransport

async def graphql_subscription():
    transport = WebsocketsTransport(url="ws://localhost:8000/graphql")
    async with Client(transport=transport) as session:
        subscription = gql("""
            subscription {
                messageAdded(channelId: "general") { id content createdAt }
            }
        """)
        async for result in session.subscribe(subscription):
            print(f"New message: {result['messageAdded']}")
```

---

## Step 80: GraphQL Best Practices

```graphql
# Connection pattern for pagination
type UserEdge { node: User! cursor: String! }
type UserConnection { edges: [UserEdge!]! pageInfo: PageInfo! totalCount: Int! }

# Mutation payload pattern
type CreateUserPayload {
  user: User
  errors: [UserError!]!
}
type UserError { field: String message: String! code: String! }
type Mutation { createUser(input: CreateUserInput!): CreateUserPayload! }
```

```python
# Query depth + complexity limiting
import strawberry
from strawberry.extensions import QueryDepthLimiter, MaxTokensLimiter

schema = strawberry.Schema(
    query=Query,
    extensions=[
        QueryDepthLimiter(max_depth=10),
        MaxTokensLimiter(max_token_count=1000)
    ]
)
```

---

## 📊 สรุป Part 08

| Feature | REST | GraphQL |
|---------|------|--------|
| Data fetching | Multiple endpoints | Single endpoint |
| Over-fetching | Common | Avoided |
| Real-time | SSE/WebSocket | Subscriptions |
| Learning curve | Low | Medium |

---
*Part 08 | Steps 71-80 | ระดับพื้นฐาน*
