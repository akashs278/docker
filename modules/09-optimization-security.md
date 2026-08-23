# Module 09 — Production Optimization, Multi-Stage Builds & Security Hardening

> **🏠 [← Back to Main Course](../README.md)** | **[← Module 08](./08-docker-compose.md)** | **Next: [Module 10 — Orchestration & Capstone →](./10-orchestration-capstone.md)**

---

## 📋 Table of Contents
- [9.1 The Need for Image Optimization](#91-the-need-for-image-optimization)
- [9.2 Multi-Stage Builds Explained](#92-multi-stage-builds-explained)
- [9.3 Real-World Multi-Stage Examples (Go & Node.js)](#93-real-world-multi-stage-examples-go--nodejs)
- [9.4 Base Image Selection: Alpine vs Distroless vs Scratch](#94-base-image-selection-alpine-vs-distroless-vs-scratch)
- [9.5 Production Security Hardening Checklist](#95-production-security-hardening-checklist)
- [9.6 Resource Caps & Container Limits](#96-resource-caps--container-limits)

---

## 9.1 The Need for Image Optimization

Standard development images contain full compilers, C software libraries, package managers, and text editors.
* **Large Images (1GB+):** Slow network deployment, high cloud storage costs, slow auto-scaling.
* **Security Vulnerabilities:** Every unnecessary package inside your image increases the attack surface for hackers.

---

## 9.2 Multi-Stage Builds Explained

A **Multi-Stage Build** allows you to use multiple `FROM` statements in a single Dockerfile.
* **Stage 1 (Builder):** Uses a heavy image containing SDKs and compilers to build your source code.
* **Stage 2 (Production):** Starts from a tiny, clean runtime image and copies **ONLY** the compiled binary/artifacts from Stage 1.

```mermaid
graph LR
    subgraph Stage1 [Stage 1: Heavy Builder Image (~1GB)]
        B1[FROM golang:1.20] --> B2[Compile Go Binary]
    end

    subgraph Stage2 [Stage 2: Minimal Production Image (~10MB)]
        R1[FROM scratch] --> R2[COPY --from=builder /app/server .]
    end

    B2 -->|Extract Only Binary| R2
```

---

## 9.3 Real-World Multi-Stage Examples (Go & Node.js)

### Go Application (Shrinking from 850MB -> 12MB!):

```dockerfile
# Stage 1: Build binary
FROM golang:1.20-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
# Build statically linked binary without debug symbols (-ldflags "-s -w")
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o server .

# Stage 2: Minimal Runtime
FROM scratch
WORKDIR /app
# Copy CA certificates for HTTPS requests
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
# Copy binary from builder stage
COPY --from=builder /app/server .

EXPOSE 8080
ENTRYPOINT ["./server"]
```

### Node.js / React Application:

```dockerfile
# Stage 1: Build React static bundle
FROM node:20-alpine AS build-stage
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Stage 2: Serve via Nginx
FROM nginx:alpine AS prod-stage
# Copy custom Nginx config and compiled static dist files
COPY --from=build-stage /app/dist /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

---

## 9.4 Base Image Selection: Alpine vs Distroless vs Scratch

| Base Image | Size | Contains Shell? | Ideal Use Case |
| :--- | :--- | :--- | :--- |
| **`node:20` (Full)** | ~1.1 GB | Yes (`bash`) | Local development only |
| **`node:20-alpine`** | ~170 MB | Yes (`sh`, `apk`) | General production apps needing minimal tools |
| **`gcr.io/distroless`** | ~20 MB | ❌ No | High-security enterprise apps (Language runtime only) |
| **`scratch`** | 0 MB (Empty) | ❌ No | Statically compiled Go/Rust/C binaries |

---

## 9.5 Production Security Hardening Checklist

### 1. Never Run as Root (`USER` directive)
By default, Docker containers run commands as `root`. If an attacker escapes a container, they gain root access to your host system!

```dockerfile
# Create non-privileged user and group
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser
```

### 2. Add Healthchecks
Allow Docker and orchestrators to automatically detect and restart frozen or dead containers:

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD curl -f http://localhost:8080/health || exit 1
```

### 3. Vulnerability Scanning with Trivy / Docker Scan
Scan images for CVE security flaws before deploying:

```bash
# Scan image using Trivy
trivy image my-app:v1
```

### 4. Read-Only Filesystem
Prevent attackers from modifying application code inside a running container:

```bash
docker run -d --read-only --tmpfs /tmp my-app:v1
```

---

## 9.6 Resource Caps & Container Limits

Prevent a single container from starving other services of CPU or RAM:

```bash
# Limit container to 512MB RAM and 1.5 CPU cores
docker run -d --name throttled-app --memory="512m" --cpus="1.5" my-app:v1
```

### In `compose.yaml`:
```yaml
services:
  api:
    image: my-app:v1
    deploy:
      resources:
        limits:
          cpus: '1.0'
          memory: 512M
        reservations:
          cpus: '0.25'
          memory: 128M
```

---

> **👉 Next Step:** Learn orchestration concepts, CI/CD automation, and deploy the Capstone Microservices stack in **[Module 10 — Orchestration & Capstone](./10-orchestration-capstone.md)**.
