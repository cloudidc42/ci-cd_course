# Part 12: Docker Compose & Multi-Container Apps

## สารบัญ

1. [Docker Compose คืออะไร?](#docker-compose-คืออะไร)
2. [ติดตั้ง Docker Compose](#ติดตั้ง-docker-compose)
3. [docker-compose.yml Syntax](#docker-composeyml-syntax)
4. [Services, Networks, Volumes](#services-networks-volumes)
5. [ตัวอย่าง: Webapp + Database + Redis](#ตัวอย่าง-webapp--database--redis)
6. [Environment Variables](#environment-variables)
7. [Depends_on และ Health Checks](#depends_on-และ-health-checks)
8. [Scaling Services](#scaling-services)
9. [Override Files](#override-files)
10. [Profiles](#profiles)
11. [Docker Compose ใน CI/CD Pipeline](#docker-compose-ใน-cicd-pipeline)
12. [Testing ด้วย Docker Compose](#testing-ด้วย-docker-compose)
13. [Commands สำคัญ](#commands-สำคัญ)
14. [Best Practices](#best-practices)
15. [แบบฝึกหัด](#แบบฝึกหัด)

---

## Docker Compose คืออะไร?

Docker Compose คือ tool สำหรับ define และ run **multi-container Docker applications** ด้วย YAML file เดียว

### ทำไมต้องใช้ Docker Compose?

แอปพลิเคชันจริงๆ มักประกอบด้วยหลาย services เช่น:
- Web application
- Database (PostgreSQL, MySQL)
- Cache (Redis, Memcached)
- Message queue (RabbitMQ, Kafka)
- Load balancer (Nginx)

การรัน containers ทั้งหมดด้วย `docker run` ด้วยตนเองนั้น:
- ยุ่งยาก
- Error-prone
- ยากต่อการ share กับทีม

**Docker Compose แก้ปัญหาด้วยการ:**
- Define ทุก services ใน `docker-compose.yml` เดียว
- รัน environment ทั้งหมดด้วย `docker compose up`
- จัดการ networks และ volumes อัตโนมัติ
- ง่ายต่อการ reproduce environment

### Docker Compose v1 vs v2

```bash
# v1 (deprecated) - ติดตั้งแยก
docker-compose up

# v2 (current) - built-in กับ Docker
docker compose up
```

> **หมายเหตุ:** บทนี้ใช้ Docker Compose v2 (subcommand ของ docker)

---

## ติดตั้ง Docker Compose

### Linux

```bash
# Docker Compose v2 มาพร้อมกับ Docker Engine 20.10+
# ถ้า install Docker ด้วย apt/yum จะมาพร้อมกันแล้ว

# ตรวจสอบ version
docker compose version

# ถ้ายังไม่มี ให้ install plugin
sudo apt-get install docker-compose-plugin

# หรือ download binary โดยตรง
DOCKER_CONFIG=${DOCKER_CONFIG:-$HOME/.docker}
mkdir -p $DOCKER_CONFIG/cli-plugins
curl -SL "https://github.com/docker/compose/releases/latest/download/docker-compose-linux-x86_64" \
    -o $DOCKER_CONFIG/cli-plugins/docker-compose
chmod +x $DOCKER_CONFIG/cli-plugins/docker-compose
```

### macOS และ Windows

Docker Desktop มา Docker Compose v2 พร้อมกันแล้ว

```bash
# ตรวจสอบ
docker compose version
# Docker Compose version v2.24.0
```

---

## docker-compose.yml Syntax

### โครงสร้างพื้นฐาน

```yaml
# docker-compose.yml
version: "3.9"  # compose file format version (optional ใน v2)

services:        # container definitions
  web:
    image: nginx:alpine
    ports:
      - "80:80"
  
  db:
    image: postgres:15
    environment:
      POSTGRES_PASSWORD: password

volumes:         # named volumes
  db_data:

networks:        # custom networks
  app_network:
```

### Services Configuration

```yaml
services:
  webapp:
    # Image ที่จะใช้
    image: nginx:alpine
    
    # หรือ build จาก Dockerfile
    build:
      context: ./app
      dockerfile: Dockerfile
      target: production  # multi-stage target
      args:
        - NODE_ENV=production
      cache_from:
        - myapp:latest
    
    # ชื่อ container
    container_name: my-webapp
    
    # Port mapping (host:container)
    ports:
      - "80:80"
      - "443:443"
      - "127.0.0.1:8080:8080"  # bind to specific interface
    
    # Environment variables
    environment:
      NODE_ENV: production
      PORT: 3000
    
    # หรือโหลดจาก file
    env_file:
      - .env
      - .env.production
    
    # Volumes
    volumes:
      - ./html:/usr/share/nginx/html:ro
      - app_data:/app/data
    
    # Network
    networks:
      - frontend
      - backend
    
    # Dependencies
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started
    
    # Restart policy
    restart: always
    # unless-stopped, on-failure, no
    
    # Resource limits
    deploy:
      resources:
        limits:
          cpus: "0.5"
          memory: 512M
        reservations:
          cpus: "0.25"
          memory: 256M
    
    # Health check
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    
    # Logging
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
    
    # Labels
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.webapp.rule=Host(`example.com`)"
    
    # Command override
    command: ["node", "server.js"]
    
    # Entrypoint override
    entrypoint: ["/bin/sh", "-c"]
    
    # Working directory
    working_dir: /app
    
    # User
    user: "1000:1000"
    
    # Privileged mode (ระวัง!)
    privileged: false
    
    # Read-only filesystem
    read_only: true
    
    # Temporary filesystems
    tmpfs:
      - /tmp
      - /run
    
    # Expose ports (ไม่ publish, แค่ document)
    expose:
      - "3000"
    
    # Extra hosts
    extra_hosts:
      - "host.docker.internal:host-gateway"
    
    # DNS settings
    dns:
      - 8.8.8.8
      - 8.8.4.4
    
    # Profiles (เปิดใช้แค่บาง environment)
    profiles:
      - development
```

### Networks Configuration

```yaml
networks:
  # Default bridge network
  frontend:
  
  # Custom network settings
  backend:
    driver: bridge
    ipam:
      config:
        - subnet: "172.20.0.0/16"
          gateway: "172.20.0.1"
  
  # External network (สร้างไว้ก่อน)
  existing-network:
    external: true
  
  # Overlay network (สำหรับ Swarm)
  swarm-network:
    driver: overlay
    attachable: true
```

### Volumes Configuration

```yaml
volumes:
  # Named volume แบบง่าย
  db_data:
  
  # Named volume พร้อม driver options
  app_data:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: /host/path
  
  # External volume (สร้างไว้ก่อน)
  existing-volume:
    external: true
  
  # NFS volume
  nfs_data:
    driver: local
    driver_opts:
      type: nfs
      o: "addr=192.168.1.100,rw"
      device: ":/nfs/path"
```

---

## Services, Networks, Volumes

### ตัวอย่าง Complete Stack

```yaml
# docker-compose.yml - Complete web application stack
version: "3.9"

services:
  # Nginx Reverse Proxy
  nginx:
    image: nginx:1.25-alpine
    container_name: nginx-proxy
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/conf.d:/etc/nginx/conf.d:ro
      - ./nginx/ssl:/etc/nginx/ssl:ro
      - static_files:/usr/share/nginx/html:ro
    networks:
      - frontend
    depends_on:
      - webapp
    restart: always
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "5"

  # Node.js Web Application
  webapp:
    build:
      context: .
      target: production
    container_name: webapp
    environment:
      NODE_ENV: production
      PORT: 3000
      DATABASE_URL: postgresql://appuser:${DB_PASSWORD}@postgres:5432/appdb
      REDIS_URL: redis://redis:6379
    volumes:
      - static_files:/app/public
    networks:
      - frontend
      - backend
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_started
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s

  # PostgreSQL Database
  postgres:
    image: postgres:15-alpine
    container_name: postgres-db
    environment:
      POSTGRES_DB: appdb
      POSTGRES_USER: appuser
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      PGDATA: /var/lib/postgresql/data/pgdata
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./postgres/init:/docker-entrypoint-initdb.d:ro
    networks:
      - backend
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U appuser -d appdb"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s
    logging:
      driver: "json-file"
      options:
        max-size: "10m"

  # Redis Cache
  redis:
    image: redis:7-alpine
    container_name: redis-cache
    command: redis-server --appendonly yes --requirepass ${REDIS_PASSWORD}
    volumes:
      - redis_data:/data
    networks:
      - backend
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "${REDIS_PASSWORD}", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  # Database Admin (Development only)
  adminer:
    image: adminer:latest
    container_name: db-admin
    ports:
      - "8080:8080"
    networks:
      - backend
    depends_on:
      - postgres
    restart: unless-stopped
    profiles:
      - tools

volumes:
  postgres_data:
    driver: local
  redis_data:
    driver: local
  static_files:
    driver: local

networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
    internal: true  # ไม่มี external access
```

---

## ตัวอย่าง: Webapp + Database + Redis

### โครงสร้าง Project

```
myapp/
├── docker-compose.yml
├── docker-compose.override.yml
├── docker-compose.prod.yml
├── .env
├── .env.example
├── Dockerfile
├── src/
│   └── index.js
├── nginx/
│   └── conf.d/
│       └── default.conf
└── postgres/
    └── init/
        └── 01-schema.sql
```

### Application Code

```javascript
// src/index.js
const express = require('express');
const { Pool } = require('pg');
const Redis = require('ioredis');

const app = express();
const PORT = process.env.PORT || 3000;

// PostgreSQL connection
const pool = new Pool({
    connectionString: process.env.DATABASE_URL
});

// Redis connection
const redis = new Redis(process.env.REDIS_URL);

app.use(express.json());

// Health check endpoint
app.get('/health', async (req, res) => {
    try {
        // Test DB
        await pool.query('SELECT 1');
        
        // Test Redis
        await redis.ping();
        
        res.json({ 
            status: 'healthy',
            db: 'connected',
            cache: 'connected',
            timestamp: new Date().toISOString()
        });
    } catch (error) {
        res.status(503).json({ 
            status: 'unhealthy',
            error: error.message 
        });
    }
});

// Get all users (with Redis cache)
app.get('/users', async (req, res) => {
    try {
        // Check cache
        const cached = await redis.get('users');
        if (cached) {
            return res.json({ 
                data: JSON.parse(cached), 
                source: 'cache' 
            });
        }
        
        // Query DB
        const result = await pool.query('SELECT * FROM users ORDER BY id');
        
        // Store in cache (10 minutes)
        await redis.setex('users', 600, JSON.stringify(result.rows));
        
        res.json({ data: result.rows, source: 'database' });
    } catch (error) {
        res.status(500).json({ error: error.message });
    }
});

// Create user
app.post('/users', async (req, res) => {
    const { name, email } = req.body;
    
    try {
        const result = await pool.query(
            'INSERT INTO users (name, email) VALUES ($1, $2) RETURNING *',
            [name, email]
        );
        
        // Invalidate cache
        await redis.del('users');
        
        res.status(201).json(result.rows[0]);
    } catch (error) {
        res.status(500).json({ error: error.message });
    }
});

app.listen(PORT, () => {
    console.log(`Server running on port ${PORT}`);
});
```

### Database Schema

```sql
-- postgres/init/01-schema.sql
CREATE TABLE IF NOT EXISTS users (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Insert sample data
INSERT INTO users (name, email) VALUES
    ('Alice Johnson', 'alice@example.com'),
    ('Bob Smith', 'bob@example.com'),
    ('Charlie Brown', 'charlie@example.com')
ON CONFLICT DO NOTHING;
```

### Nginx Configuration

```nginx
# nginx/conf.d/default.conf
upstream webapp {
    server webapp:3000;
    keepalive 32;
}

server {
    listen 80;
    server_name localhost;
    
    # Static files
    location /static {
        alias /usr/share/nginx/html;
        expires 1y;
        add_header Cache-Control "public, immutable";
    }
    
    # Health check
    location /nginx-health {
        return 200 "healthy\n";
        add_header Content-Type text/plain;
    }
    
    # Proxy to Node.js
    location / {
        proxy_pass http://webapp;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_cache_bypass $http_upgrade;
        
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }
}
```

### docker-compose.yml สมบูรณ์

```yaml
version: "3.9"

services:
  nginx:
    image: nginx:1.25-alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx/conf.d:/etc/nginx/conf.d:ro
      - static_files:/usr/share/nginx/html:ro
    depends_on:
      webapp:
        condition: service_healthy
    networks:
      - frontend
    restart: unless-stopped

  webapp:
    build:
      context: .
      target: production
    expose:
      - "3000"
    environment:
      NODE_ENV: production
      PORT: 3000
      DATABASE_URL: postgresql://${DB_USER:-appuser}:${DB_PASSWORD}@postgres:5432/${DB_NAME:-appdb}
      REDIS_URL: redis://:${REDIS_PASSWORD}@redis:6379
    volumes:
      - static_files:/app/public
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    networks:
      - frontend
      - backend
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s

  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: ${DB_NAME:-appdb}
      POSTGRES_USER: ${DB_USER:-appuser}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      PGDATA: /var/lib/postgresql/data/pgdata
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./postgres/init:/docker-entrypoint-initdb.d:ro
    networks:
      - backend
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER:-appuser} -d ${DB_NAME:-appdb}"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s

  redis:
    image: redis:7-alpine
    command: redis-server --requirepass ${REDIS_PASSWORD} --appendonly yes
    volumes:
      - redis_data:/data
    networks:
      - backend
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "${REDIS_PASSWORD}", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  postgres_data:
  redis_data:
  static_files:

networks:
  frontend:
  backend:
    internal: true
```

---

## Environment Variables

### .env File

```bash
# .env
# Database
DB_NAME=appdb
DB_USER=appuser
DB_PASSWORD=supersecretpassword123

# Redis
REDIS_PASSWORD=redissecretpassword456

# Application
NODE_ENV=production
APP_SECRET_KEY=yoursecretkeyhere

# Email (optional)
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=noreply@example.com
SMTP_PASS=emailpassword
```

### .env.example สำหรับ Documentation

```bash
# .env.example (commit ไปใน git)
# Copy เป็น .env และแก้ไขค่า

# Database Configuration
DB_NAME=appdb
DB_USER=appuser
DB_PASSWORD=CHANGE_ME

# Redis Configuration
REDIS_PASSWORD=CHANGE_ME

# Application Configuration
NODE_ENV=production
APP_SECRET_KEY=CHANGE_ME

# Email Configuration (optional)
SMTP_HOST=
SMTP_PORT=587
SMTP_USER=
SMTP_PASS=
```

### การใช้ Environment Variables ใน docker-compose.yml

```yaml
services:
  webapp:
    environment:
      # ใช้ค่าจาก .env โดยตรง
      DB_PASSWORD: ${DB_PASSWORD}
      
      # ใส่ default value
      PORT: ${PORT:-3000}
      NODE_ENV: ${NODE_ENV:-development}
      
      # ค่าคงที่ (ไม่ใช้ .env)
      APP_NAME: MyApplication
    
    # โหลดทั้ง .env file
    env_file:
      - .env
    
    # โหลดหลาย env files
    env_file:
      - .env
      - .env.${COMPOSE_ENV:-development}
```

### Variable Interpolation

```yaml
# docker-compose.yml
services:
  db:
    image: postgres:${POSTGRES_VERSION:-15}-alpine
    environment:
      POSTGRES_DB: ${DB_NAME}
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - ${DATA_PATH:-./data}/postgres:/var/lib/postgresql/data
```

---

## Depends_on และ Health Checks

### ปัญหาของ depends_on แบบง่าย

```yaml
# ❌ depends_on ธรรมดา จะรอแค่ container start ไม่ใช่ ready
services:
  webapp:
    depends_on:
      - postgres  # รอแค่ container start, postgres อาจยังไม่ ready!
```

### depends_on พร้อม Health Check

```yaml
# ✅ รอจนกว่า service จะ healthy
services:
  webapp:
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
      worker:
        condition: service_started  # แค่ started ไม่ต้อง healthy
  
  postgres:
    image: postgres:15-alpine
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER} -d ${DB_NAME}"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s  # รอก่อนเริ่ม health check
  
  redis:
    image: redis:7-alpine
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5
```

### Health Check Options

```yaml
healthcheck:
  # Command to run
  test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
  # หรือ shell form
  test: ["CMD-SHELL", "curl -f http://localhost:3000/health || exit 1"]
  
  # ตรวจสอบทุก 30 วินาที
  interval: 30s
  
  # Timeout ถ้าไม่ตอบใน 10 วินาที = fail
  timeout: 10s
  
  # ลองใหม่ 3 ครั้งถึงจะ unhealthy
  retries: 3
  
  # รอ 40 วินาทีก่อนเริ่ม check (startup time)
  start_period: 40s
```

### Health Check สำหรับ Services ต่างๆ

```yaml
# PostgreSQL
postgres:
  healthcheck:
    test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER:-postgres}"]
    interval: 10s
    timeout: 5s
    retries: 5

# MySQL
mysql:
  healthcheck:
    test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-u", "root", "-p${MYSQL_ROOT_PASSWORD}"]
    interval: 10s
    timeout: 5s
    retries: 5

# Redis
redis:
  healthcheck:
    test: ["CMD", "redis-cli", "ping"]
    interval: 5s
    timeout: 3s
    retries: 5

# MongoDB
mongo:
  healthcheck:
    test: ["CMD", "mongosh", "--eval", "db.adminCommand('ping')"]
    interval: 10s
    timeout: 5s
    retries: 5

# RabbitMQ
rabbitmq:
  healthcheck:
    test: ["CMD", "rabbitmq-diagnostics", "ping"]
    interval: 10s
    timeout: 5s
    retries: 5

# HTTP service
webapp:
  healthcheck:
    test: ["CMD-SHELL", "curl -f http://localhost:3000/health || exit 1"]
    interval: 30s
    timeout: 10s
    retries: 3
    start_period: 40s

# Node.js with wget (Alpine)
webapp-alpine:
  healthcheck:
    test: ["CMD-SHELL", "wget -qO- http://localhost:3000/health || exit 1"]
    interval: 30s
    timeout: 10s
    retries: 3
```

### Wait-for-it Script (Alternative)

```dockerfile
# Dockerfile
FROM node:18-alpine
RUN apk add --no-cache bash

# Copy wait script
COPY wait-for-it.sh /usr/local/bin/wait-for-it
RUN chmod +x /usr/local/bin/wait-for-it

COPY . /app
WORKDIR /app
RUN npm ci --only=production

CMD ["wait-for-it", "postgres:5432", "--", "node", "src/index.js"]
```

---

## Scaling Services

### Scale ด้วย --scale flag

```bash
# Scale webapp ขึ้นเป็น 3 instances
docker compose up -d --scale webapp=3

# ดู containers
docker compose ps
# NAME         IMAGE     SERVICE    STATUS    PORTS
# myapp-webapp-1   ...   webapp   running   3000/tcp
# myapp-webapp-2   ...   webapp   running   3000/tcp
# myapp-webapp-3   ...   webapp   running   3000/tcp
```

### Scale ใน docker-compose.yml

```yaml
services:
  webapp:
    image: myapp:latest
    deploy:
      replicas: 3
      resources:
        limits:
          cpus: "0.5"
          memory: 512M
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
```

### Load Balancer สำหรับ Scaled Services

```yaml
version: "3.9"

services:
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - webapp

  webapp:
    build: .
    expose:
      - "3000"
    environment:
      PORT: 3000
    deploy:
      replicas: 3
```

```nginx
# nginx.conf
events {
    worker_connections 1024;
}

http {
    upstream webapp_cluster {
        # Load balancing (round-robin by default)
        server webapp:3000;
        
        # หรือใช้ least connections
        # least_conn;
    }
    
    server {
        listen 80;
        
        location / {
            proxy_pass http://webapp_cluster;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }
    }
}
```

---

## Override Files

Docker Compose รองรับการ override configuration สำหรับ environments ต่างๆ

### docker-compose.override.yml

ถ้ามีไฟล์ `docker-compose.override.yml` จะถูก merge อัตโนมัติ:

```yaml
# docker-compose.yml (base)
version: "3.9"

services:
  webapp:
    image: myapp:latest
    environment:
      NODE_ENV: production
    ports:
      - "3000:3000"
  
  postgres:
    image: postgres:15
    environment:
      POSTGRES_PASSWORD: ${DB_PASSWORD}
```

```yaml
# docker-compose.override.yml (development overrides)
version: "3.9"

services:
  webapp:
    build: .           # build แทน pull image
    volumes:
      - .:/app         # mount source code
      - /app/node_modules
    environment:
      NODE_ENV: development
      DEBUG: "app:*"
    command: ["npm", "run", "dev"]  # hot reload
  
  postgres:
    ports:
      - "5432:5432"    # expose port สำหรับ local tools
  
  # เพิ่ม service ที่ใช้แค่ development
  mailhog:
    image: mailhog/mailhog
    ports:
      - "1025:1025"
      - "8025:8025"
```

```yaml
# docker-compose.prod.yml (production overrides)
version: "3.9"

services:
  webapp:
    image: myregistry.com/myapp:${TAG:-latest}
    restart: always
    deploy:
      replicas: 3
      resources:
        limits:
          memory: 1G
  
  postgres:
    # ไม่ expose port ใน production
    volumes:
      - /data/postgres:/var/lib/postgresql/data
```

### การใช้ Override Files

```bash
# Development (default: ใช้ docker-compose.yml + docker-compose.override.yml)
docker compose up

# Production (ไม่ใช้ override)
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d

# Staging
docker compose -f docker-compose.yml -f docker-compose.staging.yml up -d

# ดูผลลัพธ์ merged config
docker compose config
docker compose -f docker-compose.yml -f docker-compose.prod.yml config
```

### ตัวอย่าง Merge Logic

```yaml
# Base
services:
  webapp:
    image: myapp
    environment:
      KEY1: value1
      KEY2: value2
    ports:
      - "3000:3000"

# Override
services:
  webapp:
    environment:
      KEY2: overridden  # override
      KEY3: new         # add new
    ports:
      - "8080:3000"     # add port

# Result
services:
  webapp:
    image: myapp
    environment:
      KEY1: value1      # kept
      KEY2: overridden  # overridden
      KEY3: new         # added
    ports:
      - "3000:3000"     # kept
      - "8080:3000"     # added
```

---

## Profiles

Profiles ช่วยให้ define services ที่ start เฉพาะบาง environment

```yaml
version: "3.9"

services:
  # Services หลัก (รันเสมอ)
  webapp:
    build: .
    ports:
      - "3000:3000"
  
  postgres:
    image: postgres:15
    environment:
      POSTGRES_PASSWORD: password
  
  # Development tools (รันแค่ตอน dev)
  adminer:
    image: adminer
    ports:
      - "8080:8080"
    profiles:
      - development
      - tools
  
  # Monitoring (รันแค่ตอน monitoring)
  prometheus:
    image: prom/prometheus
    ports:
      - "9090:9090"
    profiles:
      - monitoring
  
  grafana:
    image: grafana/grafana
    ports:
      - "3001:3000"
    profiles:
      - monitoring
  
  # Testing tools
  mailhog:
    image: mailhog/mailhog
    ports:
      - "1025:1025"
      - "8025:8025"
    profiles:
      - development
      - testing
  
  # CI/CD testing
  test-runner:
    build:
      context: .
      target: test
    command: ["npm", "test"]
    profiles:
      - testing
```

### การใช้ Profiles

```bash
# รัน default services เท่านั้น
docker compose up -d

# รัน development tools ด้วย
docker compose --profile development up -d

# รัน monitoring ด้วย
docker compose --profile monitoring up -d

# รันหลาย profiles
docker compose --profile development --profile monitoring up -d

# ตั้งค่าด้วย COMPOSE_PROFILES
export COMPOSE_PROFILES=development,monitoring
docker compose up -d
```

---

## Docker Compose ใน CI/CD Pipeline

### GitHub Actions Integration

```yaml
# .github/workflows/ci.yml
name: CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Copy env file
        run: cp .env.example .env
      
      - name: Create test environment
        run: |
          echo "DB_PASSWORD=testpassword" >> .env
          echo "REDIS_PASSWORD=testredis" >> .env
      
      - name: Start test environment
        run: |
          docker compose -f docker-compose.yml \
                          -f docker-compose.test.yml \
                          up -d --build
      
      - name: Wait for services
        run: |
          # รอให้ทุก services healthy
          timeout 120 bash -c '
            until docker compose ps | grep -v "health: starting" | grep -q "healthy"; do
              echo "Waiting for services..."
              sleep 5
            done
          '
      
      - name: Run tests
        run: |
          docker compose run --rm \
            -e CI=true \
            test-runner npm test
      
      - name: Run integration tests
        run: |
          docker compose run --rm \
            -e CI=true \
            test-runner npm run test:integration
      
      - name: Collect test results
        if: always()
        run: |
          mkdir -p test-results
          docker compose cp test-runner:/app/coverage ./test-results/ || true
      
      - name: Upload test coverage
        uses: codecov/codecov-action@v3
        with:
          directory: ./test-results/coverage
      
      - name: Stop environment
        if: always()
        run: |
          docker compose -f docker-compose.yml \
                          -f docker-compose.test.yml \
                          down -v
      
      - name: Show logs on failure
        if: failure()
        run: docker compose logs --no-color

  build:
    runs-on: ubuntu-latest
    needs: test
    if: github.ref == 'refs/heads/main'
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      
      - name: Login to registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          target: production
          push: true
          tags: |
            ghcr.io/${{ github.repository }}:latest
            ghcr.io/${{ github.repository }}:${{ github.sha }}
```

### docker-compose.test.yml

```yaml
# docker-compose.test.yml
version: "3.9"

services:
  webapp:
    environment:
      NODE_ENV: test
      DATABASE_URL: postgresql://testuser:testpassword@postgres:5432/testdb
      REDIS_URL: redis://redis:6379
    command: ["npm", "start"]
  
  postgres:
    environment:
      POSTGRES_DB: testdb
      POSTGRES_USER: testuser
      POSTGRES_PASSWORD: testpassword
    # ไม่ใช้ volume เพื่อให้ clean ทุก test run
    tmpfs:
      - /var/lib/postgresql/data
  
  redis:
    command: redis-server  # ไม่ต้องมี password ใน test
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
  
  test-runner:
    build:
      context: .
      target: test
    environment:
      NODE_ENV: test
      DATABASE_URL: postgresql://testuser:testpassword@postgres:5432/testdb
      REDIS_URL: redis://redis:6379
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    volumes:
      - ./coverage:/app/coverage
    profiles:
      - testing
```

---

## Testing ด้วย Docker Compose

### Unit Tests

```yaml
# สำหรับ unit tests (ไม่ต้องการ external services)
services:
  unit-test:
    build:
      context: .
      target: test
    command: ["npm", "run", "test:unit"]
    environment:
      NODE_ENV: test
    profiles:
      - unit-test
```

### Integration Tests

```yaml
services:
  integration-test:
    build:
      context: .
      target: test
    command: ["npm", "run", "test:integration"]
    environment:
      NODE_ENV: test
      DATABASE_URL: postgresql://user:pass@postgres:5432/testdb
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    profiles:
      - integration-test
  
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: testdb
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
    tmpfs:
      - /var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user -d testdb"]
      interval: 5s
      timeout: 3s
      retries: 5
  
  redis:
    image: redis:7-alpine
    tmpfs:
      - /data
```

### E2E Tests

```yaml
services:
  e2e-test:
    image: cypress/included:latest
    depends_on:
      webapp:
        condition: service_healthy
    environment:
      CYPRESS_baseUrl: http://webapp:3000
    volumes:
      - ./cypress:/cypress
      - ./cypress/videos:/cypress/videos
      - ./cypress/screenshots:/cypress/screenshots
    profiles:
      - e2e
```

### การรัน Tests ใน CI

```bash
# Unit tests
docker compose --profile unit-test run --rm unit-test

# Integration tests
docker compose --profile integration-test up -d postgres redis
docker compose --profile integration-test run --rm integration-test
docker compose --profile integration-test down -v

# E2E tests
docker compose --profile e2e up -d webapp postgres redis
docker compose --profile e2e run --rm e2e-test
docker compose --profile e2e down -v
```

---

## Commands สำคัญ

### docker compose up/down

```bash
# รัน services
docker compose up
docker compose up -d              # detached mode (background)
docker compose up --build         # build images ก่อน
docker compose up --force-recreate  # recreate containers แม้ไม่มีการเปลี่ยนแปลง
docker compose up --no-deps webapp  # รัน webapp เท่านั้น ไม่รัน dependencies

# หยุด services
docker compose down
docker compose down -v           # ลบ volumes ด้วย
docker compose down --rmi all    # ลบ images ด้วย
docker compose down --remove-orphans  # ลบ orphan containers

# Start/Stop โดยไม่ recreate
docker compose start
docker compose stop
docker compose restart
docker compose restart webapp    # restart service เดียว
```

### docker compose logs

```bash
# ดู logs ทุก services
docker compose logs

# Follow (real-time)
docker compose logs -f

# ดู service เดียว
docker compose logs webapp
docker compose logs -f webapp

# แสดง timestamp
docker compose logs -t

# แสดง 100 บรรทัดล่าสุด
docker compose logs --tail 100

# ดูหลาย services
docker compose logs webapp postgres

# ดู logs ตั้งแต่เวลาที่กำหนด
docker compose logs --since 1h
```

### docker compose exec

```bash
# รัน command ใน running container
docker compose exec webapp sh
docker compose exec webapp bash
docker compose exec webapp node -e "console.log('hello')"

# รัน ด้วย specific user
docker compose exec -u root webapp bash

# รัน ใน container เฉพาะถ้า scale > 1
docker compose exec --index=2 webapp bash
```

### docker compose run

```bash
# สร้าง container ชั่วคราวและรัน command
docker compose run --rm webapp sh
docker compose run --rm webapp npm test
docker compose run --rm webapp npm run migrate

# ไม่รัน dependencies
docker compose run --no-deps webapp sh

# Override port
docker compose run --rm -p 3001:3000 webapp node server.js

# Override environment
docker compose run --rm -e DEBUG=true webapp npm start
```

### docker compose ps

```bash
# ดู status ของ services
docker compose ps
docker compose ps --all    # รวม stopped containers
docker compose ps webapp   # service เดียว

# ผลลัพธ์
# NAME            IMAGE      SERVICE   CREATED         STATUS         PORTS
# myapp-webapp-1  myapp:v1   webapp    2 minutes ago   Up 2 minutes   3000/tcp
# myapp-postgres  postgres   postgres  2 minutes ago   Up 2 minutes   5432/tcp
```

### docker compose build

```bash
# Build images
docker compose build
docker compose build webapp     # build service เดียว
docker compose build --no-cache  # ไม่ใช้ cache
docker compose build --pull      # pull base images ใหม่ก่อน build

# Build พร้อม push
docker compose push
docker compose push webapp
```

### docker compose pull

```bash
# Pull images
docker compose pull
docker compose pull webapp postgres
```

### docker compose config

```bash
# ดู merged config
docker compose config

# Validate config
docker compose config --quiet

# ดู services
docker compose config --services

# ดู volumes
docker compose config --volumes
```

### docker compose top

```bash
# ดู processes ใน containers
docker compose top
docker compose top webapp
```

### docker compose cp

```bash
# Copy files
docker compose cp webapp:/app/logs ./logs
docker compose cp ./config.json webapp:/app/config.json
```

---

## Best Practices

### 1. ใช้ Named Volumes แทน Bind Mounts ใน Production

```yaml
# ❌ Bind mount (ขึ้นกับ host filesystem)
volumes:
  - /host/data:/container/data

# ✅ Named volume (Docker จัดการ)
volumes:
  - db_data:/var/lib/postgresql/data
```

### 2. ระบุ Version สำหรับ Images

```yaml
# ❌ ไม่ระบุ version
image: nginx

# ✅ ระบุ specific version
image: nginx:1.25.3-alpine
```

### 3. ใช้ .env File และไม่ commit secrets

```bash
# .gitignore
.env
.env.local
.env.production
*.env
```

### 4. Health Checks สำหรับทุก Services

```yaml
services:
  postgres:
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER}"]
      interval: 10s
      timeout: 5s
      retries: 5
```

### 5. จำกัด Resources

```yaml
services:
  webapp:
    deploy:
      resources:
        limits:
          cpus: "1"
          memory: 1G
        reservations:
          cpus: "0.5"
          memory: 512M
```

### 6. ใช้ Internal Networks

```yaml
networks:
  frontend:  # accessible จาก outside
  backend:
    internal: true  # ไม่มี external access
```

### 7. Log Rotation

```yaml
services:
  webapp:
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "5"
        compress: "true"
```

### 8. Explicit depends_on พร้อม Conditions

```yaml
services:
  webapp:
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
```

### 9. Read-only Filesystem

```yaml
services:
  webapp:
    read_only: true
    tmpfs:
      - /tmp
      - /run
    volumes:
      - logs_data:/app/logs  # เฉพาะที่ต้องเขียน
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: สร้าง Basic Stack

สร้าง docker-compose.yml สำหรับ:
- Node.js API (build จาก Dockerfile)
- PostgreSQL database
- Redis cache
- Adminer (db admin tool)

ต้องการ:
1. API ต้องรอให้ PostgreSQL healthy ก่อน
2. Environment variables ต้องอยู่ใน .env file
3. Database data ต้อง persist ด้วย named volume
4. Adminer ต้อง accessible ที่ port 8080

### แบบฝึกหัดที่ 2: Development vs Production

สร้างไฟล์สำหรับสองสภาพแวดล้อม:

**Development:**
- Hot reload สำหรับ Node.js
- Source code mounted เป็น volume
- MailHog สำหรับ test email
- Exposed database port

**Production:**
- Pull image จาก registry
- ไม่ mount source code
- ไม่มี debug tools
- Internal network สำหรับ database

### แบบฝึกหัดที่ 3: Scale และ Load Balancing

สร้าง configuration สำหรับ:
- Nginx load balancer
- 3 instances ของ Node.js API
- Shared PostgreSQL
- Shared Redis

### เฉลย แบบฝึกหัดที่ 1

```yaml
# docker-compose.yml
version: "3.9"

services:
  api:
    build: .
    ports:
      - "${API_PORT:-3000}:3000"
    environment:
      NODE_ENV: ${NODE_ENV:-development}
      DATABASE_URL: postgresql://${DB_USER}:${DB_PASSWORD}@postgres:5432/${DB_NAME}
      REDIS_URL: redis://redis:6379
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    networks:
      - app_net
    restart: unless-stopped

  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: ${DB_NAME}
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - app_net
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER} -d ${DB_NAME}"]
      interval: 10s
      timeout: 5s
      retries: 5
    restart: unless-stopped

  redis:
    image: redis:7-alpine
    volumes:
      - redis_data:/data
    networks:
      - app_net
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5
    restart: unless-stopped

  adminer:
    image: adminer:latest
    ports:
      - "8080:8080"
    networks:
      - app_net
    depends_on:
      - postgres
    restart: unless-stopped

volumes:
  postgres_data:
  redis_data:

networks:
  app_net:
    driver: bridge
```

```bash
# .env
DB_NAME=appdb
DB_USER=appuser
DB_PASSWORD=secretpassword
API_PORT=3000
NODE_ENV=development
```

### เฉลย แบบฝึกหัดที่ 2

```yaml
# docker-compose.yml (base)
version: "3.9"

services:
  api:
    environment:
      NODE_ENV: production
      DATABASE_URL: postgresql://${DB_USER}:${DB_PASSWORD}@postgres:5432/${DB_NAME}
    depends_on:
      postgres:
        condition: service_healthy
    networks:
      - frontend
      - backend
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "wget -qO- http://localhost:3000/health || exit 1"]
      interval: 30s
      timeout: 10s
      retries: 3
  
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: ${DB_NAME}
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - backend
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER}"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  postgres_data:

networks:
  frontend:
  backend:
    internal: true
```

```yaml
# docker-compose.override.yml (development)
version: "3.9"

services:
  api:
    build: .
    environment:
      NODE_ENV: development
      DEBUG: "api:*"
    volumes:
      - .:/app
      - /app/node_modules
    command: ["npm", "run", "dev"]
  
  postgres:
    ports:
      - "5432:5432"  # expose ใน dev
  
  mailhog:
    image: mailhog/mailhog
    ports:
      - "1025:1025"
      - "8025:8025"
    networks:
      - frontend
```

```yaml
# docker-compose.prod.yml (production)
version: "3.9"

services:
  api:
    image: ghcr.io/myorg/myapp:${TAG:-latest}
    deploy:
      replicas: 2
      resources:
        limits:
          memory: 512M
```

---

## สรุป

ในบทนี้คุณได้เรียนรู้:

1. **Docker Compose** - Tool สำหรับจัดการ multi-container apps
2. **docker-compose.yml Syntax** - Services, Networks, Volumes configuration
3. **Health Checks** - รอให้ services ready ก่อน start dependencies
4. **Environment Variables** - .env files, variable interpolation
5. **Override Files** - แยก config ตาม environment
6. **Profiles** - เปิด/ปิด services ตาม use case
7. **CI/CD Integration** - ใช้ Docker Compose ใน pipeline
8. **Testing** - Unit, Integration, E2E tests

### สิ่งที่ควรจำ

- ใช้ **depends_on + healthcheck** ไม่ใช่แค่ depends_on
- เก็บ secrets ใน **.env** และ gitignore มัน
- ใช้ **named volumes** สำหรับ persistent data
- แยก environments ด้วย **override files**
- ใช้ **profiles** สำหรับ optional services
- เสมอ specify **image version** อย่างชัดเจน

---

*บทถัดไป: [Part 13: Container Registry & Image Management](part-13-container-registry.md)*
