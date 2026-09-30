# Part 11: Docker Fundamentals สำหรับ CI/CD

## สารบัญ

1. [Docker คืออะไร?](#docker-คืออะไร)
2. [Container vs Virtual Machine](#container-vs-virtual-machine)
3. [ติดตั้ง Docker](#ติดตั้ง-docker)
4. [Docker CLI Commands พื้นฐาน](#docker-cli-commands-พื้นฐาน)
5. [Dockerfile Syntax](#dockerfile-syntax)
6. [Multi-Stage Builds](#multi-stage-builds)
7. [Docker Layers และ Caching](#docker-layers-และ-caching)
8. [.dockerignore](#dockerignore)
9. [Build Optimizations](#build-optimizations)
10. [ตัวอย่าง Dockerfile สำหรับแต่ละภาษา](#ตัวอย่าง-dockerfile-สำหรับแต่ละภาษา)
11. [Debugging Containers](#debugging-containers)
12. [Docker Networking Basics](#docker-networking-basics)
13. [Docker Volumes](#docker-volumes)
14. [Docker ใน CI/CD Pipeline](#docker-ใน-cicd-pipeline)
15. [แบบฝึกหัด](#แบบฝึกหัด)

---

## Docker คืออะไร?

Docker คือ platform สำหรับการพัฒนา, ส่งมอบ, และรันแอปพลิเคชันในรูปแบบ **container** ซึ่งเป็น isolated environment ที่รวม code, runtime, libraries, และ configuration ทั้งหมดไว้ด้วยกัน

### ทำไมต้องใช้ Docker?

ปัญหาที่พบบ่อยในการพัฒนาซอฟต์แวร์คือ "It works on my machine!" ซึ่งหมายความว่าโค้ดทำงานได้บนเครื่องนักพัฒนา แต่ไม่ทำงานบน server หรือเครื่องของคนอื่น

**สาเหตุของปัญหา:**
- Version ของ library ที่แตกต่างกัน
- Configuration ของ OS ที่แตกต่างกัน
- Environment variables ที่ขาดหาย
- Dependencies ที่ไม่ครบถ้วน

**Docker แก้ปัญหาเหล่านี้ด้วยการ:**
- Package แอปพลิเคชันพร้อม dependencies ทั้งหมดไว้ใน container
- Guarantee ว่า container จะทำงานเหมือนกันทุกที่
- ทำให้ environment สอดคล้องกันตั้งแต่ development จนถึง production

### Docker Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Docker Host                          │
│                                                         │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐             │
│  │Container │  │Container │  │Container │             │
│  │  App A   │  │  App B   │  │  App C   │             │
│  └──────────┘  └──────────┘  └──────────┘             │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │              Docker Engine (daemon)              │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │              Host Operating System               │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │              Physical Hardware                   │   │
│  └─────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

**องค์ประกอบหลักของ Docker:**

1. **Docker Daemon (`dockerd`)** - Background service ที่จัดการ containers, images, networks, volumes
2. **Docker CLI** - Command-line interface สำหรับ interact กับ Docker Daemon
3. **Docker Registry** - ที่เก็บ Docker images (เช่น Docker Hub)
4. **Docker Image** - Blueprint หรือ template สำหรับสร้าง container
5. **Docker Container** - Instance ที่รันได้ของ Docker Image

---

## Container vs Virtual Machine

การทำความเข้าใจความแตกต่างระหว่าง Container และ VM เป็นสิ่งสำคัญ

### Virtual Machine Architecture

```
┌──────────────────────────────────────────────┐
│                  VM Host                     │
│                                              │
│  ┌──────────────┐    ┌──────────────┐       │
│  │     VM 1     │    │     VM 2     │       │
│  │  ┌────────┐  │    │  ┌────────┐  │       │
│  │  │  App   │  │    │  │  App   │  │       │
│  │  ├────────┤  │    │  ├────────┤  │       │
│  │  │  Bins/ │  │    │  │  Bins/ │  │       │
│  │  │  Libs  │  │    │  │  Libs  │  │       │
│  │  ├────────┤  │    │  ├────────┤  │       │
│  │  │  Guest │  │    │  │  Guest │  │       │
│  │  │   OS   │  │    │  │   OS   │  │       │
│  └──┴────────┘  │    └──┴────────┘  │       │
│                 │                   │       │
│  └──────────────┘    └──────────────┘       │
│                                              │
│  ┌──────────────────────────────────────┐   │
│  │           Hypervisor                 │   │
│  └──────────────────────────────────────┘   │
│                                              │
│  ┌──────────────────────────────────────┐   │
│  │         Host Operating System        │   │
│  └──────────────────────────────────────┘   │
│                                              │
│  ┌──────────────────────────────────────┐   │
│  │         Physical Hardware            │   │
│  └──────────────────────────────────────┘   │
└──────────────────────────────────────────────┘
```

### Container Architecture

```
┌──────────────────────────────────────────────┐
│               Docker Host                    │
│                                              │
│  ┌───────────┐  ┌───────────┐  ┌─────────┐ │
│  │Container 1│  │Container 2│  │Container│ │
│  │  ┌──────┐ │  │  ┌──────┐ │  │   3     │ │
│  │  │ App  │ │  │  │ App  │ │  │ ┌─────┐ │ │
│  │  ├──────┤ │  │  ├──────┤ │  │ │ App │ │ │
│  │  │Bins/ │ │  │  │Bins/ │ │  │ ├─────┤ │ │
│  │  │Libs  │ │  │  │Libs  │ │  │ │Bins/│ │ │
│  └──┴──────┘─┘  └──┴──────┘─┘  │ │Libs │ │ │
│                                  └─┴─────┘─┘ │
│                                              │
│  ┌──────────────────────────────────────┐   │
│  │         Docker Engine                │   │
│  └──────────────────────────────────────┘   │
│                                              │
│  ┌──────────────────────────────────────┐   │
│  │         Host Operating System        │   │
│  └──────────────────────────────────────┘   │
│                                              │
│  ┌──────────────────────────────────────┐   │
│  │         Physical Hardware            │   │
│  └──────────────────────────────────────┘   │
└──────────────────────────────────────────────┘
```

### การเปรียบเทียบ

| คุณสมบัติ | Container | Virtual Machine |
|-----------|-----------|-----------------|
| **Size** | MB (เล็กกว่า) | GB (ใหญ่กว่า) |
| **Boot Time** | วินาที | นาที |
| **OS** | Share host OS kernel | ต้องมี Guest OS เต็มรูปแบบ |
| **Isolation** | Process-level isolation | Full hardware isolation |
| **Performance** | Near-native | Overhead จาก hypervisor |
| **Portability** | สูงมาก | สูง แต่ image ใหญ่กว่า |
| **Security** | น้อยกว่า VM | แยกกันสมบูรณ์กว่า |
| **Resource Usage** | ประหยัด | ใช้มากกว่า |

**เมื่อไหรควรใช้อะไร:**
- **Container** - Modern application development, microservices, CI/CD pipelines
- **VM** - Workloads ที่ต้องการ full OS isolation, legacy applications, mixed OS environments

---

## ติดตั้ง Docker

### ติดตั้งบน Ubuntu/Debian

```bash
# อัปเดต package index
sudo apt-get update

# ติดตั้ง dependencies
sudo apt-get install -y \
    ca-certificates \
    curl \
    gnupg \
    lsb-release

# เพิ่ม Docker's official GPG key
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
    sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

# ตั้งค่า repository
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# ติดตั้ง Docker Engine
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

# เพิ่ม user เข้ากลุ่ม docker (ไม่ต้องใช้ sudo)
sudo usermod -aG docker $USER
newgrp docker

# ตรวจสอบการติดตั้ง
docker --version
docker run hello-world
```

### ติดตั้งบน macOS

```bash
# วิธีที่ 1: ดาวน์โหลด Docker Desktop จาก https://www.docker.com/products/docker-desktop

# วิธีที่ 2: ใช้ Homebrew
brew install --cask docker

# ตรวจสอบ
docker --version
docker compose version
```

### ติดตั้งบน Windows

```powershell
# วิธีที่ 1: ดาวน์โหลด Docker Desktop จาก https://www.docker.com/products/docker-desktop
# ต้องเปิดใช้งาน WSL2 หรือ Hyper-V

# วิธีที่ 2: ใช้ winget
winget install Docker.DockerDesktop

# ตรวจสอบใน PowerShell
docker --version
```

### Docker Post-Installation

```bash
# เริ่ม Docker service
sudo systemctl start docker
sudo systemctl enable docker  # เริ่มอัตโนมัติตอน boot

# ตรวจสอบ status
sudo systemctl status docker

# ทดสอบ
docker run hello-world

# ดู Docker info
docker info

# ดู Docker version (รายละเอียด)
docker version
```

---

## Docker CLI Commands พื้นฐาน

### Image Commands

```bash
# ดึง image จาก registry
docker pull nginx
docker pull nginx:latest
docker pull nginx:1.25-alpine
docker pull ubuntu:22.04

# แสดง images ที่มีในเครื่อง
docker images
docker image ls

# ลบ image
docker rmi nginx
docker image rm nginx:latest

# ลบ images ที่ไม่ได้ใช้
docker image prune

# ค้นหา image ใน Docker Hub
docker search nginx

# ดู layers ของ image
docker image history nginx
docker history nginx
```

### Container Lifecycle

```bash
# รัน container (และสร้างใหม่)
docker run nginx
docker run -d nginx                     # background (detached mode)
docker run -it ubuntu bash              # interactive + terminal
docker run --name my-nginx nginx        # ตั้งชื่อ container
docker run -p 8080:80 nginx             # map port (host:container)
docker run -p 8080:80 -d nginx          # background + port mapping

# ดู running containers
docker ps
docker container ls

# ดู ALL containers (รวม stopped)
docker ps -a
docker ps --all

# หยุด container
docker stop my-nginx
docker stop $(docker ps -q)            # หยุดทั้งหมด

# เริ่ม container ที่หยุดไว้
docker start my-nginx

# Restart container
docker restart my-nginx

# Pause/Unpause
docker pause my-nginx
docker unpause my-nginx

# ลบ container
docker rm my-nginx
docker rm -f my-nginx                   # force remove (แม้ยังรันอยู่)
docker rm $(docker ps -aq)             # ลบทั้งหมด

# รัน container แล้วลบอัตโนมัติเมื่อหยุด
docker run --rm nginx echo "hello"
```

### ตัวอย่างการใช้งาน docker run

```bash
# รัน web server
docker run -d \
    --name webserver \
    -p 8080:80 \
    -v $(pwd)/html:/usr/share/nginx/html \
    nginx:alpine

# รัน database
docker run -d \
    --name postgres-db \
    -p 5432:5432 \
    -e POSTGRES_PASSWORD=mypassword \
    -e POSTGRES_USER=myuser \
    -e POSTGRES_DB=mydb \
    -v postgres_data:/var/lib/postgresql/data \
    postgres:15

# รัน Redis
docker run -d \
    --name redis-cache \
    -p 6379:6379 \
    redis:7-alpine

# รัน MySQL
docker run -d \
    --name mysql-db \
    -p 3306:3306 \
    -e MYSQL_ROOT_PASSWORD=rootpassword \
    -e MYSQL_DATABASE=mydb \
    -e MYSQL_USER=user \
    -e MYSQL_PASSWORD=password \
    mysql:8.0
```

### Container Information

```bash
# ดู logs
docker logs my-nginx
docker logs -f my-nginx                 # follow (real-time)
docker logs --tail 100 my-nginx         # แสดง 100 บรรทัดล่าสุด
docker logs --since 1h my-nginx         # logs ใน 1 ชั่วโมงที่ผ่านมา

# Execute command ใน container ที่กำลังรัน
docker exec my-nginx ls /etc/nginx
docker exec -it my-nginx bash           # เข้า interactive shell
docker exec -it my-nginx sh             # สำหรับ Alpine ที่ไม่มี bash

# ดู processes ใน container
docker top my-nginx

# ดู stats (CPU, Memory, Network, Disk)
docker stats
docker stats my-nginx
docker stats --no-stream                # snapshot ครั้งเดียว

# ดู details ของ container
docker inspect my-nginx
docker inspect my-nginx | grep -i ip    # ดู IP address

# Copy files
docker cp my-nginx:/etc/nginx/nginx.conf ./nginx.conf   # container -> host
docker cp ./nginx.conf my-nginx:/etc/nginx/nginx.conf   # host -> container
```

### System Commands

```bash
# ดู disk usage
docker system df

# ลบ resources ที่ไม่ได้ใช้
docker system prune                     # containers, networks, images
docker system prune -a                  # รวม unused images ด้วย
docker system prune --volumes           # รวม volumes ด้วย

# ดู events
docker events
docker events --since 1h

# Login/Logout registry
docker login
docker login ghcr.io
docker logout

# Tag image
docker tag nginx:latest myrepo/nginx:1.0

# Push image
docker push myrepo/nginx:1.0
```

---

## Dockerfile Syntax

Dockerfile คือ text file ที่มี instructions สำหรับ build Docker image

### โครงสร้างพื้นฐาน

```dockerfile
# Dockerfile
# Comment จะขึ้นต้นด้วย #

# Instruction ARGUMENTS
FROM ubuntu:22.04
RUN apt-get update && apt-get install -y python3
COPY . /app
WORKDIR /app
CMD ["python3", "app.py"]
```

### Instruction ทั้งหมด

#### FROM - Base Image

```dockerfile
# รูปแบบพื้นฐาน
FROM ubuntu:22.04

# ใช้ latest (ไม่แนะนำใน production)
FROM node:latest

# Alpine Linux (เบามาก ~5MB)
FROM alpine:3.18

# Scratch (empty image สำหรับ compiled binaries)
FROM scratch

# Multi-stage: ตั้งชื่อ stage
FROM node:18 AS builder
FROM nginx:alpine AS production

# Build argument ใน FROM
ARG NODE_VERSION=18
FROM node:${NODE_VERSION}
```

#### RUN - Execute Commands

```dockerfile
# Shell form (รันผ่าน /bin/sh -c)
RUN apt-get update

# Exec form (แนะนำ)
RUN ["apt-get", "update"]

# รวมหลาย commands (ลด layers)
RUN apt-get update && \
    apt-get install -y \
        curl \
        wget \
        git && \
    rm -rf /var/lib/apt/lists/*

# สำหรับ Alpine
RUN apk add --no-cache \
    curl \
    wget \
    git

# สำหรับ Node.js
RUN npm ci --only=production

# สำหรับ Python
RUN pip install --no-cache-dir -r requirements.txt
```

#### COPY - Copy Files

```dockerfile
# Copy file เดียว
COPY package.json /app/package.json

# Copy directory
COPY src/ /app/src/

# Copy หลาย files
COPY package.json package-lock.json /app/

# Copy ทุกอย่างใน current directory
COPY . /app/

# Copy ด้วย permissions
COPY --chown=node:node . /app/

# Copy จาก build stage
COPY --from=builder /app/dist /usr/share/nginx/html
```

#### ADD - เพิ่ม Files (มีความสามารถเพิ่มเติม)

```dockerfile
# แตก tar อัตโนมัติ
ADD archive.tar.gz /app/

# ดาวน์โหลดจาก URL (ไม่แนะนำ ใช้ RUN curl/wget แทน)
ADD https://example.com/file.txt /app/

# ส่วนใหญ่ควรใช้ COPY แทน ADD
```

#### WORKDIR - Working Directory

```dockerfile
# ตั้ง working directory
WORKDIR /app

# ถ้า path ไม่มี จะสร้างให้อัตโนมัติ
WORKDIR /usr/src/app

# สามารถใช้หลายครั้ง
WORKDIR /app
WORKDIR src
# ผลลัพธ์คือ /app/src
```

#### ENV - Environment Variables

```dockerfile
# ตั้ง single variable
ENV NODE_ENV=production

# ตั้งหลาย variables
ENV PORT=3000 \
    NODE_ENV=production \
    LOG_LEVEL=info

# ใช้ใน build process
ENV APP_HOME=/app
WORKDIR ${APP_HOME}
```

#### ARG - Build Arguments

```dockerfile
# ประกาศ build argument
ARG VERSION=latest
ARG REGISTRY=docker.io

# ใช้ใน build
FROM node:${VERSION}
RUN echo "Building from ${REGISTRY}"

# ส่งค่าตอน build
# docker build --build-arg VERSION=18 .

# ARG ที่ประกาศก่อน FROM ใช้ได้แค่ใน FROM
ARG BASE_IMAGE=node:18
FROM ${BASE_IMAGE}

# ARG หลัง FROM จะ scope อยู่แค่ stage นั้น
ARG NODE_ENV=production
ENV NODE_ENV=${NODE_ENV}
```

#### EXPOSE - Expose Ports

```dockerfile
# Expose port (แค่ documentation เท่านั้น)
EXPOSE 3000
EXPOSE 80 443
EXPOSE 8080/tcp
EXPOSE 53/udp

# ต้องใช้ -p หรือ -P เมื่อรัน container จริงๆ
# docker run -p 3000:3000 myapp
# docker run -P myapp  # map ทุก exposed ports อัตโนมัติ
```

#### CMD - Default Command

```dockerfile
# Exec form (แนะนำ)
CMD ["node", "app.js"]
CMD ["python3", "-m", "flask", "run"]
CMD ["nginx", "-g", "daemon off;"]

# Shell form
CMD node app.js

# CMD ถูก override ได้ตอนรัน container
# docker run myapp node other.js

# ถ้ามี ENTRYPOINT, CMD จะเป็น default arguments
ENTRYPOINT ["node"]
CMD ["app.js"]  # default, สามารถ override ได้
```

#### ENTRYPOINT - Container Entry Point

```dockerfile
# Exec form (แนะนำ)
ENTRYPOINT ["docker-entrypoint.sh"]
ENTRYPOINT ["node"]

# Shell form
ENTRYPOINT node app.js

# ตัวอย่างการใช้ร่วมกับ CMD
ENTRYPOINT ["python3"]
CMD ["app.py"]
# รัน: python3 app.py
# Override: docker run myapp other.py -> python3 other.py

# ENTRYPOINT override ด้วย --entrypoint flag
# docker run --entrypoint /bin/bash myapp
```

#### USER - Set User

```dockerfile
# ใช้ root (default)
USER root

# ใช้ non-root user (แนะนำด้าน security)
RUN groupadd -r appgroup && useradd -r -g appgroup appuser
USER appuser

# ใช้UID/GID
USER 1000:1000

# สำหรับ Node.js (มี user node built-in)
USER node
```

#### VOLUME - Declare Volumes

```dockerfile
# ประกาศ volume mount point
VOLUME /data
VOLUME ["/data", "/logs"]

# ตัวอย่าง
VOLUME /var/lib/postgresql/data
VOLUME /var/log/nginx
```

#### LABEL - Metadata

```dockerfile
LABEL maintainer="developer@example.com"
LABEL version="1.0"
LABEL description="My application"

# OCI standard labels
LABEL org.opencontainers.image.title="My App"
LABEL org.opencontainers.image.description="A sample application"
LABEL org.opencontainers.image.version="1.0.0"
LABEL org.opencontainers.image.created="2024-01-01"
LABEL org.opencontainers.image.source="https://github.com/user/repo"
LABEL org.opencontainers.image.authors="developer@example.com"
```

#### HEALTHCHECK - Container Health

```dockerfile
# ตรวจสอบด้วย HTTP
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:3000/health || exit 1

# ตรวจสอบด้วย command
HEALTHCHECK --interval=30s CMD pg_isready -U postgres || exit 1

# ปิด healthcheck
HEALTHCHECK NONE
```

#### ONBUILD - Trigger Instructions

```dockerfile
# Instructions จะรันเมื่อ image นี้ถูกใช้เป็น base image
ONBUILD COPY package.json /app/
ONBUILD RUN npm install
ONBUILD COPY . /app/
```

#### STOPSIGNAL - Stop Signal

```dockerfile
# ส่ง signal นี้เมื่อหยุด container
STOPSIGNAL SIGTERM
STOPSIGNAL 15
```

---

## Multi-Stage Builds

Multi-stage builds ช่วยลดขนาด image ในขั้นตอน production โดยแยก build environment กับ runtime environment

### ปัญหาของ Single-Stage Build

```dockerfile
# ❌ ไม่ดี: image ใหญ่เกินไป
FROM node:18
WORKDIR /app
COPY package*.json ./
RUN npm install         # รวม devDependencies ด้วย
COPY . .
RUN npm run build       # build ด้วย webpack/tsc
EXPOSE 3000
CMD ["node", "dist/index.js"]
# ผล: image ~800MB รวม node_modules ทั้งหมด
```

### Multi-Stage Build

```dockerfile
# ✅ ดีกว่า: แยก build กับ runtime
# Stage 1: Build
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci               # install ทุกอย่างรวม devDependencies
COPY . .
RUN npm run build        # build

# Stage 2: Production
FROM node:18-alpine AS production
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production  # install แค่ production deps
COPY --from=builder /app/dist ./dist  # copy แค่ build output
EXPOSE 3000
USER node
CMD ["node", "dist/index.js"]
# ผล: image ~150MB
```

### Multi-Stage Build สำหรับ React/Frontend

```dockerfile
# Stage 1: Build React app
FROM node:18-alpine AS builder
WORKDIR /app

# Install dependencies
COPY package*.json ./
RUN npm ci

# Copy source และ build
COPY . .
RUN npm run build

# Stage 2: Serve ด้วย Nginx
FROM nginx:alpine AS production
COPY --from=builder /app/build /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

**nginx.conf สำหรับ React:**
```nginx
server {
    listen 80;
    server_name localhost;
    root /usr/share/nginx/html;
    index index.html;

    # Handle React Router
    location / {
        try_files $uri $uri/ /index.html;
    }

    # Cache static assets
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }
}
```

### Multi-Stage Build สำหรับ Go

```dockerfile
# Stage 1: Build
FROM golang:1.21-alpine AS builder
WORKDIR /app

# Install dependencies
COPY go.mod go.sum ./
RUN go mod download

# Build binary
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o main .

# Stage 2: Minimal runtime
FROM scratch AS production
COPY --from=builder /app/main /main
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
EXPOSE 8080
ENTRYPOINT ["/main"]
# ผล: image ขนาดเล็กมาก ~10MB!
```

### Multi-Stage Build สำหรับ Java/Spring Boot

```dockerfile
# Stage 1: Build Maven project
FROM maven:3.9-eclipse-temurin-17 AS builder
WORKDIR /app

# Download dependencies ก่อน (ใช้ cache)
COPY pom.xml .
RUN mvn dependency:go-offline -B

# Build application
COPY src ./src
RUN mvn package -DskipTests

# Stage 2: Runtime
FROM eclipse-temurin:17-jre-alpine AS production
WORKDIR /app
COPY --from=builder /app/target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### การ Build Specific Stage

```bash
# Build จนถึง stage ที่ระบุ
docker build --target builder -t myapp:builder .
docker build --target production -t myapp:latest .

# Build พร้อม build args
docker build \
    --target production \
    --build-arg NODE_ENV=production \
    -t myapp:latest .
```

---

## Docker Layers และ Caching

### ทำความเข้าใจ Layer System

Docker image ประกอบด้วย layers ที่ซ้อนกัน แต่ละ instruction ใน Dockerfile สร้าง layer ใหม่

```
┌─────────────────────────┐
│    CMD ["node", "app"]  │  Layer 5 (writable)
├─────────────────────────┤
│    COPY . /app          │  Layer 4
├─────────────────────────┤
│    RUN npm install      │  Layer 3
├─────────────────────────┤
│    COPY package.json    │  Layer 2
├─────────────────────────┤
│    FROM node:18         │  Layer 1 (base)
└─────────────────────────┘
```

### Layer Caching

Docker cache layers ไว้ ถ้า instruction ไม่เปลี่ยน จะใช้ cache แทนการรัน build ใหม่

```bash
# ตัวอย่าง build output
Step 1/6 : FROM node:18-alpine
 ---> abc123def456
Step 2/6 : WORKDIR /app
 ---> Using cache          ← ใช้ cache
 ---> xyz789abc123
Step 3/6 : COPY package*.json ./
 ---> Using cache          ← ใช้ cache
 ---> def456ghi789
Step 4/6 : RUN npm ci
 ---> Using cache          ← ใช้ cache (เพราะ package.json ไม่เปลี่ยน)
 ---> ghi789jkl012
Step 5/6 : COPY . .        ← cache ถูก invalidate ถ้า source เปลี่ยน
 ---> abc456def789
Step 6/6 : CMD ["node", "app.js"]
 ---> Running in 1234abcd5678
```

### Optimization: จัด Order ให้ถูกต้อง

```dockerfile
# ❌ ไม่ดี: source code เปลี่ยน = ต้อง re-install dependencies ทุกครั้ง
FROM node:18-alpine
WORKDIR /app
COPY . .                    # ← copy ทุกอย่างก่อน
RUN npm install             # ← ต้อง run ทุกครั้ง

# ✅ ดีกว่า: แยก copy package.json ออกมาก่อน
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./       # ← copy แค่ package files
RUN npm ci                  # ← cache ถ้า package files ไม่เปลี่ยน
COPY . .                    # ← copy source code
```

### ดู Layers ด้วย docker history

```bash
# ดู layers ของ image
docker history nginx
docker history --no-trunc nginx  # แสดง commands เต็มๆ

# ผลลัพธ์ตัวอย่าง
IMAGE          CREATED        CREATED BY                                      SIZE
a6bd71f48f68   3 weeks ago    /bin/sh -c #(nop)  CMD ["nginx" "-g" "daemon…   0B
<missing>      3 weeks ago    /bin/sh -c #(nop)  STOPSIGNAL SIGQUIT           0B
<missing>      3 weeks ago    /bin/sh -c #(nop)  EXPOSE 80                    0B
...
```

### Cache Busting

บางครั้งต้องการ invalidate cache โดยเจตนา

```dockerfile
# วิธีที่ 1: ใช้ ARG
ARG CACHE_BUST=1
RUN apt-get update

# Build ด้วย: docker build --build-arg CACHE_BUST=$(date +%s) .

# วิธีที่ 2: ใช้ --no-cache
# docker build --no-cache -t myapp .

# วิธีที่ 3: ใส่ timestamp ใน ENV
ARG BUILD_DATE
ENV BUILD_DATE=${BUILD_DATE}
```

---

## .dockerignore

`.dockerignore` คือ file ที่ระบุว่าไม่ต้อง copy อะไรเข้าใน Docker build context เพื่อลดขนาดและเพิ่มความเร็ว

### ตัวอย่าง .dockerignore

```
# .dockerignore

# Git
.git
.gitignore

# Node.js
node_modules
npm-debug.log
yarn-error.log

# Build outputs
dist
build
.next
out

# Test files
coverage
*.test.js
*.spec.js
__tests__
.jest

# Documentation
docs
*.md
*.txt

# IDE files
.vscode
.idea
*.swp
*.swo

# OS files
.DS_Store
Thumbs.db

# Environment files
.env
.env.local
.env.*.local
*.env

# Temporary files
tmp
temp
*.tmp
*.log

# Docker files
Dockerfile*
docker-compose*
.dockerignore
```

### ทดสอบ .dockerignore

```bash
# ดูว่าไฟล์ไหนจะถูก include ใน build context
docker build --dry-run . 2>&1 | head -20

# หรือดู build context size
docker build . --no-cache 2>&1 | grep "Sending build context"
```

---

## Build Optimizations

### 1. ใช้ Alpine Images

```dockerfile
# ❌ image ใหญ่
FROM node:18          # ~900MB

# ✅ เล็กกว่ามาก
FROM node:18-alpine   # ~170MB
FROM node:18-slim     # ~250MB
```

### 2. เรียง Instructions จาก น้อยเปลี่ยน ไป เปลี่ยนบ่อย

```dockerfile
FROM node:18-alpine

# ติดตั้ง OS packages ก่อน (เปลี่ยนน้อย)
RUN apk add --no-cache dumb-init

WORKDIR /app

# Copy และ install dependencies (เปลี่ยนบางครั้ง)
COPY package*.json ./
RUN npm ci --only=production

# Copy source code (เปลี่ยนบ่อย)
COPY src/ ./src/

EXPOSE 3000
CMD ["dumb-init", "node", "src/index.js"]
```

### 3. รวม RUN Commands

```dockerfile
# ❌ สร้าง 4 layers
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get install -y wget
RUN rm -rf /var/lib/apt/lists/*

# ✅ สร้าง 1 layer
RUN apt-get update && \
    apt-get install -y \
        curl \
        wget && \
    rm -rf /var/lib/apt/lists/*
```

### 4. ลบ files ชั่วคราวใน RUN command เดียวกัน

```dockerfile
# ❌ file ยังอยู่ใน layer ก่อนหน้า
RUN apt-get update && apt-get install -y build-essential
RUN make install
RUN rm -rf /tmp/*    # ช้าเกินไป: layer แรกมี files อยู่แล้ว

# ✅ ลบในคำสั่งเดียวกัน
RUN apt-get update && \
    apt-get install -y build-essential && \
    make install && \
    apt-get remove -y build-essential && \
    apt-get autoremove -y && \
    rm -rf /var/lib/apt/lists/* /tmp/*
```

### 5. ใช้ BuildKit (Docker 18.09+)

```bash
# เปิดใช้งาน BuildKit
export DOCKER_BUILDKIT=1
docker build -t myapp .

# หรือใช้ docker buildx
docker buildx build -t myapp .
```

**BuildKit Syntax ใน Dockerfile:**

```dockerfile
# syntax=docker/dockerfile:1

FROM node:18-alpine

# Mount cache สำหรับ package manager
RUN --mount=type=cache,target=/root/.npm \
    npm ci

# Mount secret ไม่ให้ติดอยู่ใน layer
RUN --mount=type=secret,id=github_token \
    GITHUB_TOKEN=$(cat /run/secrets/github_token) \
    npm install

# Build ด้วย secret
# docker build --secret id=github_token,src=.github_token .
```

---

## ตัวอย่าง Dockerfile สำหรับแต่ละภาษา

### Node.js Application

```dockerfile
# syntax=docker/dockerfile:1
FROM node:18-alpine AS base

# ติดตั้ง dependencies จำเป็น
RUN apk add --no-cache dumb-init

# -------- Development Stage --------
FROM base AS development
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
EXPOSE 3000
CMD ["npm", "run", "dev"]

# -------- Builder Stage --------
FROM base AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# -------- Production Stage --------
FROM base AS production
ENV NODE_ENV=production
WORKDIR /app

# Create non-root user
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

# Install only production dependencies
COPY package*.json ./
RUN npm ci --only=production && npm cache clean --force

# Copy build output
COPY --from=builder /app/dist ./dist

# Set ownership
RUN chown -R appuser:appgroup /app
USER appuser

EXPOSE 3000
HEALTHCHECK --interval=30s --timeout=3s \
    CMD wget -qO- http://localhost:3000/health || exit 1

ENTRYPOINT ["dumb-init", "--"]
CMD ["node", "dist/index.js"]
```

### Python/FastAPI Application

```dockerfile
# syntax=docker/dockerfile:1
FROM python:3.11-slim AS base

# ป้องกัน Python bytecode
ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1

# -------- Development Stage --------
FROM base AS development
WORKDIR /app

COPY requirements-dev.txt requirements.txt ./
RUN pip install --no-cache-dir -r requirements-dev.txt

COPY . .
EXPOSE 8000
CMD ["uvicorn", "app.main:app", "--reload", "--host", "0.0.0.0", "--port", "8000"]

# -------- Production Stage --------
FROM base AS production

# ติดตั้ง build dependencies
RUN apt-get update && apt-get install -y \
    gcc \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app

# ติดตั้ง Python dependencies
COPY requirements.txt ./
RUN pip install --no-cache-dir -r requirements.txt

# Copy application
COPY app/ ./app/

# สร้าง non-root user
RUN addgroup --system appgroup && adduser --system --group appuser
USER appuser

EXPOSE 8000
HEALTHCHECK --interval=30s --timeout=10s \
    CMD python -c "import requests; requests.get('http://localhost:8000/health')" || exit 1

CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000", "--workers", "4"]
```

**requirements.txt:**
```
fastapi==0.104.1
uvicorn[standard]==0.24.0
pydantic==2.5.0
sqlalchemy==2.0.23
alembic==1.13.0
```

### Java Spring Boot Application

```dockerfile
# syntax=docker/dockerfile:1

# -------- Build Stage --------
FROM eclipse-temurin:17-jdk-alpine AS builder
WORKDIR /app

# ติดตั้ง Maven
RUN apk add --no-cache maven

# Download dependencies (ใช้ cache)
COPY pom.xml .
RUN mvn dependency:go-offline -B

# Build
COPY src ./src
RUN mvn package -DskipTests -B

# -------- Layer Extraction --------
FROM eclipse-temurin:17-jre-alpine AS extract
WORKDIR /extracted
COPY --from=builder /app/target/*.jar app.jar
RUN java -Djarmode=layertools -jar app.jar extract

# -------- Production Stage --------
FROM eclipse-temurin:17-jre-alpine AS production

# Create non-root user
RUN addgroup -S spring && adduser -S spring -G spring

WORKDIR /app

# Copy layers (leverages Docker caching)
COPY --from=extract /extracted/dependencies/ ./
COPY --from=extract /extracted/spring-boot-loader/ ./
COPY --from=extract /extracted/snapshot-dependencies/ ./
COPY --from=extract /extracted/application/ ./

USER spring:spring

EXPOSE 8080
HEALTHCHECK --interval=30s --timeout=10s --start-period=60s \
    CMD wget -qO- http://localhost:8080/actuator/health || exit 1

ENTRYPOINT ["java", "org.springframework.boot.loader.launch.JarLauncher"]
```

### Golang Application

```dockerfile
# syntax=docker/dockerfile:1

# -------- Build Stage --------
FROM golang:1.21-alpine AS builder
WORKDIR /app

# ติดตั้ง dependencies
RUN apk add --no-cache git ca-certificates tzdata

# Download modules (ใช้ cache)
COPY go.mod go.sum ./
RUN go mod download && go mod verify

# Build
COPY . .
RUN CGO_ENABLED=0 \
    GOOS=linux \
    GOARCH=amd64 \
    go build \
    -ldflags='-w -s -extldflags "-static"' \
    -a \
    -o /go/bin/app \
    ./cmd/main.go

# -------- Production Stage --------
FROM scratch AS production

# Copy necessary files
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
COPY --from=builder /usr/share/zoneinfo /usr/share/zoneinfo
COPY --from=builder /go/bin/app /app

EXPOSE 8080
ENTRYPOINT ["/app"]
```

---

## Debugging Containers

### เข้า Container เพื่อ Debug

```bash
# เข้า shell ของ container ที่รันอยู่
docker exec -it container_name bash
docker exec -it container_name sh     # สำหรับ Alpine

# รัน container ใหม่พร้อม shell
docker run -it myimage bash
docker run -it --rm myimage sh

# Override entrypoint เพื่อ debug
docker run -it --entrypoint /bin/sh myimage

# ดู environment variables
docker exec container_name env

# ดู filesystem
docker exec container_name ls -la /app
```

### Debug Build Problems

```bash
# Build ด้วย verbose output
docker build --progress=plain .

# หยุด build ที่ specific stage
docker build --target builder .

# รัน container จาก intermediate stage
docker build --target builder -t myapp:debug .
docker run -it myapp:debug sh

# ดู build cache
docker buildx du
docker system df
```

### Docker Logs

```bash
# ดู logs
docker logs container_name

# Follow logs real-time
docker logs -f container_name

# แสดง timestamp
docker logs -t container_name

# แสดง 50 บรรทัดล่าสุด
docker logs --tail 50 container_name

# Logs ตั้งแต่เวลาที่กำหนด
docker logs --since "2024-01-01T00:00:00" container_name
docker logs --since 30m container_name
docker logs --until 1h container_name
```

### Inspect Container

```bash
# ดู full container info
docker inspect container_name

# ดู IP address
docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' container_name

# ดู port mappings
docker inspect -f '{{json .NetworkSettings.Ports}}' container_name | python3 -m json.tool

# ดู environment variables
docker inspect -f '{{range .Config.Env}}{{println .}}{{end}}' container_name

# ดู mounted volumes
docker inspect -f '{{json .Mounts}}' container_name | python3 -m json.tool
```

---

## Docker Networking Basics

### Network Types

```bash
# ดู networks
docker network ls

# สร้าง network
docker network create my-network
docker network create --driver bridge my-network
docker network create --driver overlay my-swarm-network

# ดู network details
docker network inspect my-network

# ลบ network
docker network rm my-network
docker network prune  # ลบ networks ที่ไม่ได้ใช้
```

### Network Drivers

1. **bridge** (default) - สร้าง virtual network บน host, containers สามารถ communicate กันผ่านชื่อ container ได้
2. **host** - ใช้ network ของ host โดยตรง, ไม่มี isolation
3. **none** - ปิด networking ทั้งหมด
4. **overlay** - ใช้ใน Docker Swarm, container ต่าง hosts สื่อสารกันได้
5. **macvlan** - Assign MAC address ให้ container ดูเหมือน physical device

### ตัวอย่างการใช้ Networks

```bash
# สร้าง custom network
docker network create app-network

# รัน containers ใน network เดียวกัน
docker run -d \
    --name postgres \
    --network app-network \
    -e POSTGRES_PASSWORD=password \
    postgres:15

docker run -d \
    --name webapp \
    --network app-network \
    -p 3000:3000 \
    -e DATABASE_URL=postgres://postgres:password@postgres:5432/mydb \
    myapp:latest

# webapp สามารถ connect ไปที่ postgres ด้วยชื่อ "postgres"
# เพราะอยู่ใน network เดียวกัน
```

### Port Mapping

```bash
# Map single port
docker run -p 8080:80 nginx

# Map หลาย ports
docker run -p 8080:80 -p 8443:443 nginx

# Bind to specific interface
docker run -p 127.0.0.1:8080:80 nginx   # แค่ localhost
docker run -p 0.0.0.0:8080:80 nginx     # ทุก interfaces

# Map ทุก exposed ports อัตโนมัติ
docker run -P nginx
```

---

## Docker Volumes

### Volume Types

1. **Named Volume** - Docker จัดการเอง, persistent, แนะนำสำหรับ production
2. **Bind Mount** - Mount directory จาก host, ดีสำหรับ development
3. **tmpfs Mount** - Mount ใน memory, ข้อมูลหายเมื่อ container หยุด

### Named Volumes

```bash
# สร้าง volume
docker volume create my-data

# ดู volumes
docker volume ls

# ดู volume details
docker volume inspect my-data

# ใช้ volume กับ container
docker run -d \
    --name postgres \
    -v my-data:/var/lib/postgresql/data \
    postgres:15

# ลบ volume
docker volume rm my-data
docker volume prune  # ลบ volumes ที่ไม่ได้ใช้
```

### Bind Mounts

```bash
# Mount current directory
docker run -v $(pwd):/app myapp

# Mount specific directory
docker run -v /host/path:/container/path myapp

# Read-only mount
docker run -v $(pwd)/config:/app/config:ro myapp

# ตัวอย่าง development setup
docker run -d \
    --name dev-app \
    -p 3000:3000 \
    -v $(pwd):/app \
    -v /app/node_modules \
    node:18-alpine \
    npm run dev
```

### ตัวอย่าง Volume ใน Development

```bash
# Node.js development ด้วย hot reload
docker run -d \
    --name node-dev \
    -p 3000:3000 \
    -v $(pwd):/app \
    -v /app/node_modules \    # anonymous volume สำหรับ node_modules
    -e NODE_ENV=development \
    node:18-alpine \
    sh -c "npm install && npm run dev"
```

---

## Docker ใน CI/CD Pipeline

### GitHub Actions ตัวอย่าง

```yaml
# .github/workflows/docker-build.yml
name: Docker Build and Test

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Cache Docker layers
        uses: actions/cache@v3
        with:
          path: /tmp/.buildx-cache
          key: ${{ runner.os }}-buildx-${{ github.sha }}
          restore-keys: |
            ${{ runner.os }}-buildx-

      - name: Build Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          target: production
          push: false
          tags: myapp:test
          cache-from: type=local,src=/tmp/.buildx-cache
          cache-to: type=local,dest=/tmp/.buildx-cache-new,mode=max

      - name: Test Docker image
        run: |
          docker run --rm myapp:test npm test

      - name: Security scan
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: myapp:test
          format: table
          exit-code: 1

      # Move cache
      - name: Move cache
        run: |
          rm -rf /tmp/.buildx-cache
          mv /tmp/.buildx-cache-new /tmp/.buildx-cache
```

### GitLab CI ตัวอย่าง

```yaml
# .gitlab-ci.yml
stages:
  - build
  - test
  - deploy

variables:
  DOCKER_IMAGE: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA

build:
  stage: build
  image: docker:latest
  services:
    - docker:dind
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
  script:
    - docker build -t $DOCKER_IMAGE .
    - docker push $DOCKER_IMAGE
  only:
    - main
    - develop

test:
  stage: test
  image: docker:latest
  services:
    - docker:dind
  script:
    - docker run --rm $DOCKER_IMAGE npm test
  needs: [build]

deploy:
  stage: deploy
  script:
    - echo "Deploy to production"
  only:
    - main
  needs: [test]
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: สร้าง Dockerfile พื้นฐาน

สร้าง Dockerfile สำหรับ Node.js application ง่ายๆ:

```javascript
// app.js
const http = require('http');

const server = http.createServer((req, res) => {
    res.writeHead(200, { 'Content-Type': 'application/json' });
    res.end(JSON.stringify({
        message: 'Hello from Docker!',
        hostname: process.env.HOSTNAME,
        nodeVersion: process.version
    }));
});

const PORT = process.env.PORT || 3000;
server.listen(PORT, () => {
    console.log(`Server running on port ${PORT}`);
});
```

**คำสั่ง:**
```bash
# สร้าง Dockerfile
# Build image
docker build -t my-nodejs-app .

# รัน container
docker run -d -p 3000:3000 --name test-app my-nodejs-app

# ทดสอบ
curl http://localhost:3000

# ดู logs
docker logs test-app

# ลบ
docker stop test-app && docker rm test-app
```

### แบบฝึกหัดที่ 2: Multi-Stage Build

สร้าง TypeScript application แล้ว build ด้วย multi-stage build

```typescript
// src/index.ts
import express from 'express';

const app = express();
const PORT = process.env.PORT || 3000;

app.get('/health', (req, res) => {
    res.json({ status: 'healthy', timestamp: new Date().toISOString() });
});

app.get('/', (req, res) => {
    res.json({ message: 'Hello TypeScript Docker!' });
});

app.listen(PORT, () => {
    console.log(`Server running on port ${PORT}`);
});
```

```json
// package.json
{
  "name": "ts-docker-app",
  "version": "1.0.0",
  "scripts": {
    "build": "tsc",
    "start": "node dist/index.js",
    "dev": "ts-node-dev src/index.ts"
  },
  "dependencies": {
    "express": "^4.18.2"
  },
  "devDependencies": {
    "@types/express": "^4.17.21",
    "@types/node": "^20.10.0",
    "typescript": "^5.3.2",
    "ts-node-dev": "^2.0.0"
  }
}
```

```json
// tsconfig.json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "commonjs",
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true
  }
}
```

**งาน:** สร้าง Dockerfile ที่:
1. Build TypeScript ใน stage แรก
2. สร้าง production image ที่เล็กที่สุดใน stage ที่สอง
3. ใช้ non-root user
4. มี HEALTHCHECK

### แบบฝึกหัดที่ 3: Docker Networking

```bash
# สร้าง custom network
docker network create app-net

# รัน PostgreSQL
docker run -d \
    --name db \
    --network app-net \
    -e POSTGRES_DB=testdb \
    -e POSTGRES_USER=user \
    -e POSTGRES_PASSWORD=password \
    postgres:15-alpine

# รัน แอพที่ต้องการ connect กับ db
docker run -d \
    --name app \
    --network app-net \
    -p 3000:3000 \
    -e DATABASE_URL="postgresql://user:password@db:5432/testdb" \
    myapp:latest

# ทดสอบ network
docker exec app ping db
docker exec app wget -qO- http://db:5432 || true

# ดู network
docker network inspect app-net
```

### แบบฝึกหัดที่ 4: Optimize Dockerfile

ปรับปรุง Dockerfile นี้ให้ดีขึ้น:

```dockerfile
# ❌ Dockerfile ที่ต้องปรับปรุง
FROM ubuntu:latest
RUN apt-get update
RUN apt-get install -y nodejs npm
RUN apt-get install -y curl
COPY . /app
WORKDIR /app
RUN npm install
RUN npm run build
EXPOSE 3000
CMD node dist/index.js
```

**เป้าหมาย:**
- ลดขนาด image
- เพิ่ม build speed ด้วย caching
- ใช้ multi-stage build
- เพิ่ม security ด้วย non-root user
- เพิ่ม .dockerignore

### เฉลย แบบฝึกหัดที่ 1

```dockerfile
# Dockerfile
FROM node:18-alpine

WORKDIR /app

COPY package*.json ./
RUN npm ci --only=production

COPY app.js .

EXPOSE 3000

USER node

CMD ["node", "app.js"]
```

### เฉลย แบบฝึกหัดที่ 2

```dockerfile
# syntax=docker/dockerfile:1

# Stage 1: Build
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY tsconfig.json .
COPY src/ ./src/
RUN npm run build

# Stage 2: Production
FROM node:18-alpine AS production
WORKDIR /app

# Create non-root user
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

# Install production deps only
COPY package*.json ./
RUN npm ci --only=production && npm cache clean --force

# Copy build output
COPY --from=builder /app/dist ./dist

# Set ownership
RUN chown -R appuser:appgroup /app
USER appuser

EXPOSE 3000

HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD wget -qO- http://localhost:3000/health || exit 1

CMD ["node", "dist/index.js"]
```

### เฉลย แบบฝึกหัดที่ 4

```dockerfile
# syntax=docker/dockerfile:1

# Stage 1: Build
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Stage 2: Production
FROM node:18-alpine AS production
ENV NODE_ENV=production
WORKDIR /app

RUN addgroup -S appgroup && adduser -S appuser -G appgroup

COPY package*.json ./
RUN npm ci --only=production && npm cache clean --force

COPY --from=builder /app/dist ./dist

RUN chown -R appuser:appgroup /app
USER appuser

EXPOSE 3000

HEALTHCHECK --interval=30s --timeout=3s \
    CMD wget -qO- http://localhost:3000/ || exit 1

CMD ["node", "dist/index.js"]
```

**.dockerignore:**
```
node_modules
dist
.git
.gitignore
*.md
.env
.env.*
npm-debug.log
coverage
.nyc_output
```

---

## สรุป

ในบทนี้คุณได้เรียนรู้:

1. **Docker Architecture** - Daemon, CLI, Registry, Images, Containers
2. **Container vs VM** - ความแตกต่างและข้อดีข้อเสีย
3. **Docker CLI** - Commands สำคัญสำหรับจัดการ images และ containers
4. **Dockerfile** - Instructions ทั้งหมดพร้อมตัวอย่างการใช้งาน
5. **Multi-Stage Builds** - เทคนิคลดขนาด image สำหรับ production
6. **Layer Caching** - การเรียง instructions เพื่อ maximize cache hits
7. **Build Optimizations** - Alpine images, รวม RUN commands, BuildKit
8. **Networking** - Bridge, Host, Custom networks
9. **Volumes** - Named volumes, Bind mounts

### สิ่งที่ควรจำ

- ใช้ **Alpine images** เพื่อลดขนาด
- ใช้ **Multi-stage builds** สำหรับ production
- **จัด order** instructions จากเปลี่ยนน้อยไปเปลี่ยนบ่อย
- ใช้ **non-root user** เสมอ
- สร้าง **.dockerignore** เพื่อลด build context
- เพิ่ม **HEALTHCHECK** สำหรับ production containers
- ใช้ **specific tags** ไม่ใช่ `latest`

---

*บทถัดไป: [Part 12: Docker Compose & Multi-Container Apps](part-12-docker-compose.md)*
