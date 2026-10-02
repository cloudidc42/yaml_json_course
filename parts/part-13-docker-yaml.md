# Part 13: Docker Compose และ YAML
## Steps 121-130: Container Orchestration ด้วย Docker Compose

---

## 📖 บทนำ

Docker Compose ใช้ YAML เพื่อ define multi-container applications ทำให้การ run, build, และ manage containers ง่ายกว่าการใช้ docker commands แยกกัน

---

## Step 121: Docker Compose พื้นฐาน

### โครงสร้าง docker-compose.yml
```yaml
# docker-compose.yml
version: '3.8'

services:
  web:
    image: nginx:alpine
    ports:
      - "80:80"

  app:
    build: .        # build จาก Dockerfile ใน current directory
    depends_on:
      - db

  db:
    image: postgres:15

volumes:
  db_data:

networks:
  app_network:
```

### Commands พื้นฐาน
```bash
docker compose up -d
docker compose down
docker compose down -v
docker compose logs -f app
docker compose ps
docker compose exec app bash
docker compose up -d --scale app=3
docker compose build --no-cache
```

---

## Step 122: Service Configuration ครบถ้วน

```yaml
version: '3.8'

services:
  app:
    image: my-app:1.0.0
    build:
      context: .
      dockerfile: Dockerfile.prod
      args:
        NODE_ENV: production
      target: production
    
    container_name: my-app
    hostname: my-app
    
    ports:
      - "3000:3000"
      - "127.0.0.1:3001:3001"
    
    environment:
      - NODE_ENV=production
      - PORT=3000
      - DATABASE_URL=postgresql://user:pass@db:5432/mydb
    
    env_file:
      - .env
      - .env.production
    
    volumes:
      - ./src:/app/src
      - node_modules:/app/node_modules
    
    networks:
      - frontend
      - backend
    
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
        reservations:
          cpus: '0.25'
          memory: 256M
    
    restart: unless-stopped
    
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
    
    user: "1000:1000"
    read_only: true
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE
    
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started
    
    command: ["node", "server.js"]
```

---

## Step 123: Real-world Multi-service Application

```yaml
version: '3.8'

x-common-env: &common-env
  TZ: Asia/Bangkok
  NODE_ENV: production

x-logging: &logging
  logging:
    driver: "json-file"
    options:
      max-size: "10m"
      max-file: "5"

x-restart-policy: &restart-policy
  restart: unless-stopped

x-default-security: &default-security
  security_opt:
    - no-new-privileges:true
  cap_drop:
    - ALL

services:
  nginx:
    image: nginx:1.25-alpine
    <<: *restart-policy
    <<: *logging
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./ssl:/etc/nginx/ssl:ro
    networks:
      - frontend
    depends_on:
      - api

  api:
    build:
      context: ./api
      target: production
    <<: *restart-policy
    <<: *logging
    <<: *default-security
    environment:
      <<: *common-env
      PORT: 8000
      DATABASE_URL: postgresql://${DB_USER}:${DB_PASSWORD}@postgres:5432/${DB_NAME}
      REDIS_URL: redis://:${REDIS_PASSWORD}@redis:6379/0
      JWT_SECRET: ${JWT_SECRET}
    networks:
      - frontend
      - backend
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 30s

  postgres:
    image: postgres:15-alpine
    <<: *restart-policy
    <<: *logging
    environment:
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_DB: ${DB_NAME}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - backend
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER} -d ${DB_NAME}"]
      interval: 10s
      timeout: 5s
      retries: 5
    ports:
      - "127.0.0.1:5432:5432"

  redis:
    image: redis:7-alpine
    <<: *restart-policy
    <<: *logging
    command: >
      redis-server
        --requirepass ${REDIS_PASSWORD}
        --maxmemory 256mb
        --maxmemory-policy allkeys-lru
        --appendonly yes
    volumes:
      - redis_data:/data
    networks:
      - backend
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "${REDIS_PASSWORD}", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  postgres_data:
  redis_data:

networks:
  frontend:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/24
  backend:
    driver: bridge
    internal: true
    ipam:
      config:
        - subnet: 172.20.1.0/24
```

---

## Step 124: Environment Variables และ .env File

### .env file
```bash
# .env - ไม่ commit ไปยัง git!
DB_USER=myapp
DB_PASSWORD=supersecretpassword
DB_NAME=myapp_production

REDIS_PASSWORD=redissecretpassword

JWT_SECRET=very-long-random-jwt-secret-key-here
```

### Variable Substitution
```yaml
version: '3.8'

services:
  app:
    image: my-app:${APP_VERSION:-latest}
    environment:
      - DATABASE_URL=postgresql://${DB_USER}:${DB_PASSWORD}@${DB_HOST:-localhost}:${DB_PORT:-5432}/${DB_NAME}
      - DEBUG=${DEBUG:-false}
```

```bash
# โหลด env file ที่เฉพาะ
docker compose --env-file .env.production up -d
```

---

## Step 125: Networking

```yaml
version: '3.8'

services:
  frontend:
    networks:
      - public
  
  api:
    networks:
      - public
      - private
  
  database:
    networks:
      - private

networks:
  public:
    driver: bridge
  private:
    driver: bridge
    internal: true
```

---

## Step 126: Volumes ขั้นสูง

```yaml
version: '3.8'

services:
  app:
    volumes:
      - /host/path:/container/path
      - ./relative/path:/app/data
      - my_volume:/data
      - ./config:/app/config:ro
      - type: volume
        source: my_data
        target: /data
        volume:
          nocopy: true

volumes:
  my_volume:
    driver: local
    
  external_data:
    external: true
    name: existing_volume_name
  
  nfs_data:
    driver: local
    driver_opts:
      type: nfs
      o: addr=192.168.1.100,rw
      device: ":/nfs/share"
```

---

## Step 127: Health Checks ขั้นสูง

```yaml
version: '3.8'

services:
  postgres:
    image: postgres:15
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER:-postgres} -d ${POSTGRES_DB:-postgres}"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s

  redis:
    image: redis:7
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 3

  api:
    image: my-api:latest
    healthcheck:
      test: 
        - CMD
        - /bin/sh
        - -c
        - |
          curl -f http://localhost:8080/health \
          && curl -f http://localhost:8080/ready
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
```

---

## Step 128: Docker Compose Profiles

```yaml
version: '3.8'

services:
  api:
    image: my-api:latest
    networks:
      - app

  postgres:
    image: postgres:15
    networks:
      - app

  pgadmin:
    image: dpage/pgadmin4
    profiles: [dev]
    environment:
      PGADMIN_DEFAULT_EMAIL: admin@admin.com
      PGADMIN_DEFAULT_PASSWORD: admin
    ports:
      - "5050:80"
    networks:
      - app

  selenium:
    image: selenium/hub:latest
    profiles: [test]
    ports:
      - "4444:4444"
    networks:
      - app

networks:
  app:
```

```bash
docker compose --profile dev up -d
docker compose --profile dev --profile test up -d
COMPOSE_PROFILES=dev,test docker compose up -d
```

---

## Step 129: Development vs Production Setup

### Development
```yaml
# docker-compose.override.yml (auto-loaded)
version: '3.8'

services:
  api:
    build:
      target: development
    volumes:
      - .:/app
      - /app/node_modules
    environment:
      - NODE_ENV=development
      - DEBUG=*
    ports:
      - "3000:3000"
      - "9229:9229"

  postgres:
    ports:
      - "5432:5432"
    environment:
      POSTGRES_PASSWORD: devpassword

  mailhog:
    image: mailhog/mailhog
    ports:
      - "1025:1025"
      - "8025:8025"
    networks:
      - app
```

### Production
```yaml
# docker-compose.prod.yml
version: '3.8'

services:
  api:
    image: my-api:${VERSION}
    deploy:
      replicas: 3
      resources:
        limits:
          cpus: '1.0'
          memory: 512M
    restart: always

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.prod.conf:/etc/nginx/nginx.conf:ro
      - ./ssl:/etc/nginx/ssl:ro
    restart: always
```

```bash
# Run production
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```

---

## Step 130: Best Practices

### Security Best Practices
```yaml
version: '3.8'

services:
  app:
    image: my-app:1.0.0
    
    # ✅ Run as non-root
    user: "1000:1000"
    
    # ✅ Read-only filesystem
    read_only: true
    
    # ✅ Drop all capabilities
    cap_drop:
      - ALL
    
    # ✅ No new privileges
    security_opt:
      - no-new-privileges:true
    
    # ✅ Limit resources
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 256M
    
    # ✅ Internal network only
    networks:
      - internal
    
    # ✅ Secrets via secrets, not env vars
    secrets:
      - db_password
    
    volumes:
      - app_tmp:/tmp
      - app_logs:/var/log/app
    
    # ❌ อย่าทำแบบนี้
    # privileged: true
    # user: root

secrets:
  db_password:
    file: ./secrets/db_password.txt

networks:
  internal:
    internal: true
```

### .dockerignore
```
node_modules
.git
.gitignore
README.md
docker-compose*.yml
.env*
*.log
.DS_Store
coverage
```

---

## 📊 สรุป Part 13

| หัวข้อ | สิ่งที่เรียน |
|--------|------------|
| Compose Basics | Services, volumes, networks |
| Service Config | image/build, ports, env, volumes |
| Anchors | Reuse config กับ <<: *anchor |
| Networking | Bridge, internal, service discovery |
| Volumes | Bind mount, named, external |
| Health Checks | conditions สำหรับ depends_on |
| Profiles | dev/test/prod profiles |
| Dev vs Prod | override files, targets |
| Security | non-root, read-only, cap_drop |

---

## 🔗 ต่อไป
- [Part 14: CI/CD Pipelines กับ YAML](./part-14-cicd-yaml.md)

---
*Part 13 | Steps 121-130 | ระดับพื้นฐาน*
