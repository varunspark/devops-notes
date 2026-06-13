# Docker Notes

---

## Docker Day 2 — Dockerfile & Building Custom Images

---

### Why This is Critical for DevOps

Day 1 you ran other people's images — nginx, mysql, hello-world. That is useful but not enough.

In real DevOps work you need to:
- Package YOUR application into a Docker image
- Add your code, config and dependencies into the image
- Build it once — run it anywhere — dev, staging, production
- Push it to a registry so servers can pull and run it

**The Dockerfile is how you do all of this. Every DevOps engineer writes Dockerfiles daily.**

---

### What You Will Learn Today

- What a Dockerfile is
- Every Dockerfile instruction explained
- Building images with `docker build`
- Build context and `.dockerignore`
- Layer caching — how to write efficient Dockerfiles
- Multi-stage builds — production best practice
- Real application Dockerfiles
- Common Dockerfile mistakes

---

### What is a Dockerfile?

A Dockerfile is a text file with instructions that tells Docker how to build your image. Each instruction creates a layer.

```
Dockerfile instructions
        ↓
   docker build
        ↓
Docker Image (your custom image)
        ↓
   docker run
        ↓
Container (running application)
```

---

### Your First Dockerfile

```bash
mkdir docker_practice && cd docker_practice
```

Create the Dockerfile:

```dockerfile
FROM ubuntu:22.04
RUN apt-get update && apt-get install -y curl
CMD ["echo", "Hello from my first Docker image!"]
```

Build it:

```bash
docker build -t my-first-image .
```

Run it:

```bash
docker run my-first-image
# Output: Hello from my first Docker image!
```

---

## Every Dockerfile Instruction Explained

### FROM — Base Image

Every Dockerfile must start with `FROM`. It sets the base image you build on top of.

```dockerfile
FROM ubuntu:22.04            # Ubuntu base
FROM debian:bookworm-slim    # Debian slim — smaller
FROM python:3.11-slim        # Python pre-installed
FROM node:18-alpine          # Node.js on Alpine — very small
FROM nginx:latest            # nginx web server
FROM scratch                 # empty — for minimal Go/C binaries
```

**Choosing the right base image:**

| Image type | Size | Use case |
|---|---|---|
| `ubuntu:22.04` | ~77 MB | Full, complete |
| `python:3.11-slim` | ~45 MB | Reduced packages |
| `node:18-alpine` | ~170 MB | Minimal Linux |
| `gcr.io/distroless/python` | Tiny | No shell, most secure |

---

### RUN — Execute Commands During Build

`RUN` executes commands while building the image. Used to install packages, create directories, set permissions.

```dockerfile
# Multiple commands — WRONG way (creates multiple layers):
RUN apt-get update
RUN apt-get install -y nginx
RUN apt-get install -y curl

# Multiple commands — RIGHT way (one layer, one cache entry):
RUN apt-get update && \
    apt-get install -y \
    nginx \
    curl \
    wget \
    && rm -rf /var/lib/apt/lists/*
```

**Why `rm -rf /var/lib/apt/lists/*`?**

After installing packages, apt stores package lists that are only needed during installation. Deleting them in the SAME RUN command reduces image size. Cleaning in a separate RUN does NOT reduce size — the files already exist in the previous layer.

```dockerfile
# CORRECT — clean in SAME RUN command:
RUN apt-get update && \
    apt-get install -y nginx && \
    rm -rf /var/lib/apt/lists/*

# WRONG — too late, layer already created with the files:
RUN apt-get update && apt-get install -y nginx
RUN rm -rf /var/lib/apt/lists/*
```

---

### COPY — Copy Files Into Image

Copies files from your local machine into the image.

```dockerfile
COPY app.py /app/app.py                         # copy single file
COPY . /app/                                    # copy everything from current directory
COPY config/ /etc/myapp/config/                 # copy directory
COPY requirements.txt package.json /app/        # copy multiple files
```

---

### ADD — Copy With Extra Powers

Similar to COPY but with extra features:

```dockerfile
ADD app.tar.gz /app/                # automatically extracts archives
ADD https://example.com/file /app/  # downloads from URL
```

**Use COPY for simple file copying — it is more explicit and predictable. Use ADD only when you need auto-extraction.**

---

### WORKDIR — Set Working Directory

Sets the working directory for all following instructions. Creates the directory if it does not exist.

```dockerfile
# WRONG:
RUN cd /app && npm install    # only affects this RUN command

# CORRECT:
WORKDIR /app
RUN npm install               # runs in /app and all subsequent commands too
```

Always use `WORKDIR` instead of `RUN cd /app`.

---

### ENV — Environment Variables

Sets environment variables inside the image. Available during build AND at runtime.

```dockerfile
ENV APP_PORT=8080
ENV DB_HOST=localhost
ENV ENVIRONMENT=production

# Use in Dockerfile:
EXPOSE $APP_PORT
```

**Override at runtime:**

```bash
docker run -e APP_PORT=9090 myapp        # overrides the Dockerfile ENV
docker run --env-file .env myapp         # load from .env file
```

---

### EXPOSE — Document Port

Documents which port the container listens on. Does NOT actually publish the port — it is informational.

```dockerfile
EXPOSE 80     # nginx uses port 80
EXPOSE 8080   # application uses 8080
EXPOSE 3306   # MySQL uses 3306
```

You still need `-p 8080:80` in `docker run` to actually map the port.

---

### CMD — Default Command

Specifies the default command when container starts. Only the LAST CMD is used.

```dockerfile
CMD ["nginx", "-g", "daemon off;"]    # exec form — preferred
CMD nginx -g "daemon off;"            # shell form — runs via /bin/sh -c
```

Can be overridden at runtime:

```bash
docker run myimage          # runs CMD
docker run myimage bash     # overrides CMD — opens bash instead
```

---

### ENTRYPOINT — Fixed Command

Like CMD but cannot be overridden by normal `docker run` arguments. Arguments passed to `docker run` are APPENDED to ENTRYPOINT.

```dockerfile
ENTRYPOINT ["nginx"]
CMD ["-g", "daemon off;"]      # default arguments to ENTRYPOINT
```

```bash
docker run myimage             # runs: nginx -g "daemon off;"
docker run myimage -t          # runs: nginx -t (test config)
```

---

### CMD vs ENTRYPOINT — The Key Difference

| | CMD | ENTRYPOINT |
|---|---|---|
| Overridable | Yes — `docker run image other_cmd` | No — arguments appended |
| Use for | Default arguments | Fixed executable |
| Common use | Define defaults | Define the main program |

**Best practice — combine both:**

```dockerfile
ENTRYPOINT ["python3"]    # always runs python
CMD ["app.py"]            # default file — overridable

# docker run myimage          → python3 app.py
# docker run myimage test.py  → python3 test.py
```

---

### ARG — Build-Time Variables

Variables only available DURING the build — not in the running container.

```dockerfile
ARG APP_VERSION=1.0.0
ARG BUILD_DATE

RUN echo "Building version $APP_VERSION"
```

Pass at build time:

```bash
docker build --build-arg APP_VERSION=2.0.0 -t myapp .
```

---

### VOLUME — Declare Mount Points

Declares a directory as a mount point for persistent data.

```dockerfile
VOLUME /var/lib/mysql    # database data persists here
VOLUME /var/log/nginx    # logs persist here
VOLUME ["/app/data", "/app/logs"]    # multiple volumes
```

---

### USER — Run As Non-Root

Specifies which user to run the container as. Critical for security.

```dockerfile
# Create user and switch to it:
RUN useradd -r -s /bin/false appuser
USER appuser
# All following commands run as appuser not root
```

**Never run production containers as root.** If a container is compromised, running as root gives the attacker root access to the host system.

---

### HEALTHCHECK — Monitor Container Health

Tells Docker how to check if the container is healthy.

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
    CMD curl -f http://localhost:8080/health || exit 1
```

| Flag | Meaning |
|---|---|
| `--interval` | How often to check |
| `--timeout` | How long to wait for response |
| `--retries` | How many failures before unhealthy |

```bash
docker ps    # STATUS column shows: healthy / unhealthy / starting
```

---

### LABEL — Metadata

Adds metadata to image — author, version, description.

```dockerfile
LABEL maintainer="varun@company.com"
LABEL version="1.0.0"
LABEL description="Production nginx server"
```

```bash
docker inspect myimage | grep Labels    # view labels
```

---

## Building Images

### docker build Command

```bash
docker build -t myapp:1.0 .
```

| Part | Meaning |
|---|---|
| `-t myapp:1.0` | Tag — name and version of image |
| `.` | Build context — current directory |

**Useful build flags:**

```bash
docker build -t myapp .                     # basic build
docker build -t myapp:1.0 .                 # with version tag
docker build -t myapp:1.0 -t myapp:latest . # multiple tags
docker build -f Dockerfile.prod -t myapp .  # use specific Dockerfile
docker build --no-cache -t myapp .          # force rebuild — ignore cache
docker build --build-arg VERSION=2.0 -t myapp .  # pass build argument
```

---

### .dockerignore — Exclude From Build Context

Works exactly like `.gitignore` but for Docker builds.

```
# Dependencies — reinstalled during build anyway
node_modules/
vendor/
__pycache__/

# Git history — not needed in image
.git/
.gitignore

# Test files — not needed in production
tests/
*.test.js
*.spec.py

# Logs — not needed
*.log
logs/

# Secrets — NEVER in image
.env
*.pem
*.key

# Docker files themselves
Dockerfile
docker-compose.yml
.dockerignore

# OS files
.DS_Store
Thumbs.db
```

---

## Layer Caching — Write Efficient Dockerfiles

Docker caches each layer. If a layer's instruction has not changed — Docker uses the cache instead of rebuilding. When any layer changes — all subsequent layers rebuild from scratch.

### The Golden Rule of Dockerfile Order

**Put things that change LEAST at the top. Put things that change MOST at the bottom.**

```dockerfile
# WRONG order — slow builds:
FROM node:18-alpine
COPY . /app/               # code changes every time
WORKDIR /app
RUN npm install            # reinstalls ALL deps every build

# CORRECT order — fast builds with caching:
FROM node:18-alpine
WORKDIR /app
COPY package.json package-lock.json ./    # only changes when deps change
RUN npm install                           # cached unless deps change
COPY . .                                  # code changes often — at bottom
```

**Visualizing layer cache:**

```
Build 1 (first time):          Build 2 (only code changed):
FROM node:18-alpine  → BUILD   FROM node:18-alpine   → CACHED ✅
WORKDIR /app         → BUILD   WORKDIR /app           → CACHED ✅
COPY package.json ./  → BUILD  COPY package.json ./   → CACHED ✅
RUN npm install      → BUILD   RUN npm install        → CACHED ✅ (not downloaded again!)
COPY . .             → BUILD   COPY . .               → REBUILD (code changed)
```

Build 2 is seconds instead of minutes. This is why layer order matters enormously.

---

## Multi-Stage Builds — Production Best Practice

### The Problem With Single-Stage Builds

```dockerfile
FROM node:18
WORKDIR /app
COPY . .
RUN npm install
RUN npm run build
CMD ["node", "dist/app.js"]
```

This image includes Node.js runtime, npm and all build tools, all development dependencies, source code AND compiled output — everything used only during build. Result: 900 MB image in production.

### Multi-Stage Build Solution

```dockerfile
# ── Stage 1: Build ─────────────────────────────────
FROM node:18 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build

# ── Stage 2: Production ────────────────────────────
FROM node:18-alpine AS production
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/package*.json ./
RUN npm install --production
USER node
EXPOSE 3000
CMD ["node", "dist/app.js"]
```

**Result: 150 MB instead of 900 MB** — build tools never enter the production image.

```
Stage 1 (builder):          Stage 2 (production):
Full node:18 image          Tiny node:18-alpine
+ source code               + only compiled output
+ all dev dependencies      + only prod dependencies
+ build tools               + no build tools
        |                           |
        |    COPY --from=builder    |
        └──────────────────────────>
                    ↓
          Final image — small and clean
```

Only the LAST stage becomes the final image. All earlier stages are discarded.

### Multi-Stage for Python Application

```dockerfile
# ── Stage 1: Build dependencies ───────────────────
FROM python:3.11 AS builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --user -r requirements.txt

# ── Stage 2: Production ───────────────────────────
FROM python:3.11-slim AS production
WORKDIR /app
COPY --from=builder /root/.local /root/.local
COPY . .
ENV PATH=/root/.local/bin:$PATH
USER nobody
EXPOSE 8080
CMD ["python3", "app.py"]
```

---

## Real Dockerfile Examples

### Example 1 — Static Website with Nginx

```dockerfile
FROM nginx:alpine

# Remove default nginx config
RUN rm /etc/nginx/conf.d/default.conf

# Copy custom nginx config
COPY nginx.conf /etc/nginx/conf.d/

# Copy website files
COPY html/ /usr/share/nginx/html/

EXPOSE 80

HEALTHCHECK --interval=30s --timeout=3s \
    CMD curl -f http://localhost/ || exit 1
```

```bash
docker build -t my-website .
docker run -d -p 8080:80 --name website my-website
curl http://localhost:8080
```

---

### Example 2 — Python Flask Application

```dockerfile
FROM python:3.11-slim

LABEL maintainer="varun@company.com"
LABEL version="1.0.0"

# Create non-root user
RUN useradd -r -s /bin/false flaskuser

WORKDIR /app

# Install dependencies first (caching optimization)
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application
COPY . .

# Switch to non-root user
USER flaskuser

EXPOSE 5000

ENV FLASK_APP=app.py
ENV FLASK_ENV=production

HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
    CMD curl -f http://localhost:5000/health || exit 1

CMD ["python3", "-m", "flask", "run", "--host=0.0.0.0"]
```

---

### Example 3 — DevOps Tools Image

```dockerfile
FROM ubuntu:22.04

LABEL maintainer="varun@company.com"
LABEL description="DevOps tools image"

ENV DEBIAN_FRONTEND=noninteractive

RUN apt-get update && \
    apt-get install -y \
    curl \
    wget \
    git \
    vim \
    jq \
    unzip \
    python3 \
    python3-pip \
    awscli \
    && rm -rf /var/lib/apt/lists/*

# Install Terraform
ARG TERRAFORM_VERSION=1.5.0
RUN wget https://releases.hashicorp.com/terraform/${TERRAFORM_VERSION}/terraform_${TERRAFORM_VERSION}_linux_amd64.zip && \
    unzip terraform_${TERRAFORM_VERSION}_linux_amd64.zip && \
    mv terraform /usr/local/bin/ && \
    rm terraform_${TERRAFORM_VERSION}_linux_amd64.zip

WORKDIR /workspace

CMD ["/bin/bash"]
```

```bash
docker build -t devops-tools .
docker run -it --rm -v $(pwd):/workspace devops-tools
# Now you are inside container with all tools available
```

---

## Common Dockerfile Mistakes

| Mistake | Fix |
|---|---|
| Running as root | Add `USER appuser` before CMD |
| Secrets in ENV | Pass with `-e` at runtime — never bake in |
| No `.dockerignore` | Always create one |
| Using `latest` tag | Pin specific version: `node:18.17.0` |
| Wrong layer order | Dependencies before source code |

---

## Full Summary — Docker Day 2

**Dockerfile instructions:**

| Instruction | What it does |
|---|---|
| `FROM image:tag` | Set base image — every Dockerfile starts here |
| `RUN command` | Execute command during build |
| `COPY src dst` | Copy files into image |
| `ADD src dst` | Copy with auto-extract and URL support |
| `WORKDIR /path` | Set working directory |
| `ENV KEY=value` | Set environment variable |
| `EXPOSE port` | Document which port app uses |
| `CMD ["cmd"]` | Default command — overridable |
| `ENTRYPOINT ["cmd"]` | Fixed command — args appended |
| `ARG name=default` | Build-time variable |
| `VOLUME /path` | Declare persistent mount point |
| `USER username` | Switch to non-root user |
| `HEALTHCHECK` | Define container health check |
| `LABEL key=value` | Add metadata |

**Build commands:**

| Command | What it does |
|---|---|
| `docker build -t name .` | Build image from Dockerfile |
| `docker build --no-cache -t name .` | Force full rebuild |
| `docker build -f Dockerfile.prod .` | Use specific Dockerfile |
| `docker history image` | See all layers |

---

### Interview Questions — Docker Day 2

**Q1. What is a Dockerfile?**
A text file with instructions that tells Docker how to build a custom image. Each instruction creates a layer. Docker reads it top to bottom and builds the image by executing each instruction.

**Q2. What is the difference between CMD and ENTRYPOINT?**
CMD sets the default command that can be overridden when running the container. ENTRYPOINT sets a fixed command that always runs — arguments from docker run are appended to it. Best practice is to combine both — ENTRYPOINT for the executable, CMD for default arguments.

**Q3. What is the difference between COPY and ADD?**
COPY simply copies files from local machine to image. ADD does the same but also automatically extracts tar archives and supports downloading from URLs. Use COPY for simple file copying — it is more predictable. Use ADD only when you need auto-extraction.

**Q4. Why should you combine RUN commands with `&&` in Dockerfile?**
Each RUN instruction creates a new layer. Multiple RUN commands create multiple layers increasing image size. Combining with `&&` creates one layer. Also clean up package caches in the SAME RUN command — cleaning in a separate RUN does not reduce size because the files already exist in the previous layer.

**Q5. What is layer caching and why does instruction order matter?**
Docker caches each layer. When a layer changes all subsequent layers rebuild. Instructions that change least should be at the top — like installing dependencies. Instructions that change most go at the bottom — like copying source code. This way changing code does not invalidate the dependency cache.

**Q6. What is a multi-stage build and why use it?**
A Dockerfile with multiple FROM statements where each stage builds on the previous. Used to separate build environment from production environment. Build tools, dev dependencies and source code stay in the build stage. Only compiled output and production dependencies go into the final stage. Results in much smaller and more secure production images.

**Q7. Why should you never run containers as root?**
If a container running as root is compromised, the attacker gains root-level access. Create a dedicated non-root user with `useradd` in the Dockerfile and use `USER` instruction to switch to it before CMD. Principle of least privilege.

**Q8. What is `.dockerignore` and why is it important?**
A file that tells Docker which files to exclude from the build context. Without it, large directories like `node_modules` or `.git` are sent to Docker daemon unnecessarily — slowing builds and potentially including secrets. Always exclude `node_modules`, `.git`, `.env` and test files.

---

### Homework — Before Docker Day 3

1. Write a Dockerfile for a simple Python script that prints "Hello DevOps"
2. Build it: `docker build -t hello-devops .`
3. Run it: `docker run hello-devops`
4. Create a `.dockerignore` file
5. Write a multi-stage Dockerfile — stage 1 installs something, stage 2 copies only what is needed
6. Run `docker history hello-devops` — see all layers
7. Try `docker build --no-cache -t hello-devops .` — watch it rebuild everything

---

*Next — Docker Day 3: Docker Volumes and Networking — how containers store data permanently and how they communicate with each other.
