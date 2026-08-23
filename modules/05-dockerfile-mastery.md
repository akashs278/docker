# Module 05 — Dockerfile Mastery & Building Custom Images

> **🏠 [← Back to Main Course](../README.md)** | **[← Module 04](./04-working-with-images.md)** | **Next: [Module 06 — Volumes & Storage →](./06-volumes-storage.md)**

---

## 📋 Table of Contents
- [5.1 What is a Dockerfile?](#51-what-is-a-dockerfile)
- [5.2 Complete Directives Reference](#52-complete-directives-reference)
- [5.3 Key Differences: `COPY` vs `ADD` & `CMD` vs `ENTRYPOINT`](#53-key-differences-copy-vs-add--cmd-vs-entrypoint)
- [5.4 Understanding `ARG` vs `ENV`](#54-understanding-arg-vs-env)
- [5.5 Practical Dockerfile Examples (Node.js & Python)](#55-practical-dockerfile-examples-nodejs--python)
- [5.6 The Importance of `.dockerignore`](#56-the-importance-of-dockerignore)
- [5.7 Building & Optimization Tips](#57-building--optimization-tips)

---

## 5.1 What is a Dockerfile?

A **Dockerfile** is a plain text file (named exactly `Dockerfile`, with no extension) containing a sequence of commands that Docker executes to automatically assemble a custom Docker Image.

```text
  Dockerfile Instructions
           │
           ▼
  [ docker build -t my-app . ]
           │
           ▼
     Docker Image
```

---

## 5.2 Complete Directives Reference

| Directive | Purpose | Example |
| :--- | :--- | :--- |
| `FROM` | Defines the starting base image | `FROM node:20-alpine` |
| `WORKDIR` | Sets the working directory inside container | `WORKDIR /app` |
| `COPY` | Copies files from host machine into container | `COPY package*.json ./` |
| `ADD` | Copies files OR downloads URLs OR extracts `.tar.gz` archives | `ADD app.tar.gz /app/` |
| `RUN` | Executes build commands to install dependencies | `RUN npm install --production` |
| `ENV` | Sets environment variables available at runtime | `ENV PORT=3000` |
| `ARG` | Sets variables available ONLY during build time | `ARG BUILD_VERSION=1.0` |
| `EXPOSE` | Documents which port the app listens on | `EXPOSE 3000` |
| `USER` | Switches execution user away from `root` | `USER node` |
| `CMD` | Default executable command when container starts | `CMD ["npm", "start"]` |
| `ENTRYPOINT` | Configures container to run as a fixed executable | `ENTRYPOINT ["python", "app.py"]` |

---

## 5.3 Key Differences: `COPY` vs `ADD` & `CMD` vs `ENTRYPOINT`

### 1. `COPY` vs `ADD`
- **Use `COPY` 99% of the time:** It explicitly copies local files/folders into the image. Safe and predictable.
- **Use `ADD` ONLY when:** You need to automatically unpack local `.tar.gz` archives into a container folder.

### 2. `CMD` vs `ENTRYPOINT`

| Directive | Overridable from CLI? | Primary Use Case |
| :--- | :--- | :--- |
| **`CMD`** | **Yes!** Easily overridden by appending arguments to `docker run`. | Default arguments or standalone server startup script. |
| **`ENTRYPOINT`** | **Harder to override.** Appended arguments from CLI become parameters to entrypoint. | Creating CLI command tool containers (e.g. `curl`, `ffmpeg`). |

#### Exec Form vs Shell Form:
* **Exec Form (RECOMMENDED):** `CMD ["node", "server.js"]` (Executes process directly as PID 1, receiving shutdown signals properly).
* **Shell Form (AVOID):** `CMD node server.js` (Wraps process inside `/bin/sh -c`, which blocks termination signals).

---

## 5.4 Understanding `ARG` vs `ENV`

```dockerfile
# 1. ARG: Build-time variable (Not included in final image runtime environment)
ARG NODE_ENV=production
RUN echo "Building for environment: $NODE_ENV"

# 2. ENV: Runtime variable (Persists inside the running container)
ENV PORT=8080
ENV DB_HOST=postgres-db
```

To override an `ARG` at build time:
```bash
docker build --build-arg NODE_ENV=development -t myapp:dev .
```

---

## 5.5 Practical Dockerfile Examples (Node.js & Python)

### Production Node.js Dockerfile:
```dockerfile
# Step 1: Base image
FROM node:20-alpine

# Step 2: Set working directory
WORKDIR /app

# Step 3: Copy dependency manifests first (Optimizes layer caching!)
COPY package*.json ./

# Step 4: Install production dependencies
RUN npm ci --only=production

# Step 5: Copy application source code
COPY . .

# Step 6: Set environment variables and expose port
ENV NODE_ENV=production
EXPOSE 3000

# Step 7: Security - Run as non-root user
USER node

# Step 8: Define startup command
CMD ["node", "src/server.js"]
```

### Production Python Flask Dockerfile:
```dockerfile
FROM python:3.11-slim

WORKDIR /app

# Prevent Python from writing .pyc files & buffer output logs
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 5000

USER www-data

CMD ["gunicorn", "--bind", "0.0.0.0:5000", "app:app"]
```

---

## 5.6 The Importance of `.dockerignore`

Just like `.gitignore`, a `.dockerignore` file prevents unnecessary, sensitive, or heavy files from being sent to the Docker daemon during `docker build`.

### Example `.dockerignore`:
```text
.git
.gitignore
node_modules
npm-debug.log
Dockerfile
.env
README.md
dist
coverage
```
> 🚀 **Benefit:** Skipping `node_modules` makes your `docker build` start instantly instead of uploading 500MB of unnecessary files!

---

## 5.7 Building & Optimization Tips

```bash
# Build an image tagged 'my-app:v1' using current directory (.) context
docker build -t my-app:v1 .

# Build without using cached layers (fresh rebuild)
docker build --no-cache -t my-app:v1 .
```

---

> **👉 Next Step:** Learn how to persist data permanently using Volumes and Bind Mounts in **[Module 06 — Volumes & Storage](./06-volumes-storage.md)**.
