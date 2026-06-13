
## Docker Day 1 — Introduction & Core Concepts

---

### Why Docker is Non-Negotiable for DevOps

Here is the most common problem in software development before Docker:

**"It works on my machine."**

Developer builds an app on their laptop — works perfectly. They deploy to server — crashes immediately. The server has a different version of Python, different libraries, different OS settings.

Docker solves this completely. You package your application with everything it needs — code, runtime, libraries, config — into one unit called a container. It runs identically everywhere.

**Docker is in every DevOps job description. No exceptions.**

---

### What You Will Learn Today

- What Docker is and why it exists
- Containers vs Virtual Machines
- Docker architecture — images, containers, registry
- Installing Docker
- Your first container
- Core Docker commands
- Running real applications in containers
- Understanding Docker layers

---

### What is Docker?

Docker is a platform that lets you package, ship and run applications in containers.

Think of a container like a shipping container on a cargo ship:

```
Before shipping containers:  every ship loaded cargo differently, goods got damaged,
                             loading took days

After shipping containers:   one standard box, fits any ship, any truck, any crane —
                             worldwide
```

Docker containers work the same way for software. One container format — runs on any laptop, any server, any cloud.

---

### Containers vs Virtual Machines

| Feature | Virtual Machine | Container |
|---|---|---|
| Size | Several GB each | Tens of MB |
| Startup time | Minutes | Seconds |
| OS | Full OS per VM | Shares host OS kernel |
| Isolation | Complete | Process-level |
| Performance | More overhead | Near native |
| Use case | Full OS isolation | App packaging |

**Architecture diagram:**

```
Virtual Machine:                    Container:
┌─────────────────────┐             ┌─────────────────────┐
│ App A   │  App B    │             │ App A   │  App B    │
│─────────│───────────│             │─────────│───────────│
│Guest OS │ Guest OS  │             │Libs A   │  Libs B   │
│(full OS)│ (full OS) │             │─────────────────────│
│─────────────────────│             │    Docker Engine    │
│     Hypervisor      │             │─────────────────────│
│─────────────────────│             │  Host OS (one only) │
│      Host OS        │             │─────────────────────│
│─────────────────────│             │      Hardware       │
│      Hardware       │             └─────────────────────┘
└─────────────────────┘
Size: GB each, Boot: minutes        Size: MB each, Boot: seconds
```

---

### Docker Architecture — Three Key Concepts

| Concept | What it is | Real world analogy |
|---|---|---|
| Image | Read-only template — blueprint | Recipe / class definition |
| Container | Running instance of an image | Dish made from recipe / object |
| Registry | Storage for images (Docker Hub) | App store for images |

```
Developer                    Docker Hub (Registry)
(your machine)               (cloud storage for images)

docker build → Image → docker push → Registry
                                          ↑
                                     docker pull
                                          ↓
docker run → Container ← Image
```

---

### Installing Docker

**On Ubuntu / WSL:**

```bash
# Official method:
sudo apt update
sudo apt install -y ca-certificates curl gnupg

# Add Docker GPG key:
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

# Add Docker repository:
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] \
https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | \
sudo tee /etc/apt/sources.list.d/docker.list

# Install Docker:
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io

# Verify:
docker --version
sudo docker run hello-world
```

**Run Docker without sudo:**

```bash
sudo usermod -aG docker $USER    # add yourself to docker group
newgrp docker                    # apply without logout
docker run hello-world           # now works without sudo
```

---

### Your First Container

```bash
docker run hello-world
```

**What happens step by step:**

1. Docker looks for `hello-world` image locally — not found
2. Downloads it from Docker Hub automatically
3. Creates a container from that image
4. Runs it — prints a message
5. Container exits

You just ran your first container. The entire process took seconds.

---

### Running a Real Container — nginx

```bash
# Run nginx (no port mapping yet):
docker run nginx

# Stop with Ctrl+C, then run with port mapping:
docker run -p 8080:80 nginx
```

Now go to `http://localhost:8080` — nginx is running in a container!

**Key run flags:**

| Flag | Meaning |
|---|---|
| `-p 8080:80` | Map port 8080 on your machine to port 80 in container |
| `-d` | Detached mode — run in background |
| `-it` | Interactive terminal — stay inside container |
| `--name` | Give container a custom name |
| `--rm` | Auto-delete when stopped |
| `-e KEY=value` | Set environment variable |

**Run detached — background mode:**

```bash
docker run -d -p 8080:80 --name my-nginx nginx

# Check it is running:
docker ps
curl http://localhost:80
```

---

### Core Docker Commands

**Container lifecycle:**

```bash
# Run containers:
docker run nginx                        # run nginx
docker run -d nginx                     # run in background
docker run -it ubuntu bash              # run ubuntu and get shell inside
docker run -p 8080:80 nginx             # map ports
docker run --name my-app nginx          # custom name
docker run --rm nginx                   # auto-delete when stopped

# Managing running containers:
docker ps                               # list RUNNING containers
docker ps -a                            # list ALL containers including stopped
docker stop container_name              # stop gracefully
docker start container_name             # start stopped container
docker restart container_name           # restart

# Removing containers:
docker rm container_name                # remove stopped container
docker rm -f container_name             # force remove running container
docker rm $(docker ps -aq)              # remove ALL stopped containers
```

**Going inside a running container:**

```bash
docker exec -it my-nginx bash           # open bash shell inside container
docker exec -it my-nginx sh             # if bash not available, use sh
docker exec my-nginx ls /etc/nginx      # run single command in container
```

`-it` = interactive terminal. This is exactly like SSH-ing into the container.

**Viewing container logs:**

```bash
docker logs my-nginx                    # show all logs
docker logs my-nginx -f                 # follow logs live — like tail -f
docker logs my-nginx --tail 50          # last 50 lines
docker logs my-nginx --since 10m        # logs from last 10 minutes
```

**Container information:**

```bash
docker inspect my-nginx                 # detailed JSON info — IP, mounts, config
docker stats                            # live CPU and memory usage of all containers
docker stats my-nginx                   # stats for specific container
docker top my-nginx                     # processes running inside container
```

---

### Image Commands

```bash
# List and download:
docker images                           # list all local images
docker images -a                        # list all including intermediate
docker pull nginx                       # download image without running
docker pull nginx:1.24                  # download specific version
docker pull ubuntu:22.04                # Ubuntu specific version

# Remove images:
docker rmi nginx                        # remove image
docker rmi nginx:1.24                   # remove specific version
docker rmi $(docker images -q)          # remove ALL images
docker image prune                      # remove unused images
docker image prune -a                   # remove ALL unused images
```

**Image tags — versions:**

```bash
# Format: image_name:tag
nginx               # same as nginx:latest — always latest version
nginx:latest        # explicit latest
nginx:1.24          # specific version — USE THIS in production
ubuntu:22.04        # specific Ubuntu version
python:3.11-slim    # slim = smaller image
```

Always use specific versions in production. `latest` can change and break your application unexpectedly.

**Searching for images:**

```bash
docker search nginx                             # search Docker Hub
docker search --filter stars=100 nginx          # only popular images
```

---

### Understanding Docker Layers

Every Docker image is built from layers — each instruction in a Dockerfile creates one layer. Layers are cached and shared between images.

```
nginx image layers:
Layer 4: nginx config files
Layer 3: nginx binary
Layer 2: Ubuntu apt packages
Layer 1: Ubuntu base OS
```

When you download a second image that also uses Ubuntu — Docker reuses the cached Ubuntu layer. Saves disk space and download time.

**Seeing layers:**

```bash
docker history nginx              # see all layers of an image
docker history nginx --no-trunc   # see full commands
```

---

### Docker System Commands — Housekeeping

```bash
docker system df                    # show disk usage by Docker
docker system prune                 # remove all stopped containers, unused images
docker system prune -a              # remove everything not currently running
docker system prune --volumes       # also remove unused volumes

# Individual cleanup:
docker container prune              # remove all stopped containers
docker image prune                  # remove dangling images
docker volume prune                 # remove unused volumes
docker network prune                # remove unused networks
```

Run `docker system prune` regularly — Docker can use a lot of disk space over time.

---

### Real DevOps Scenario — Running a Full Application

**Run a database in Docker:**

```bash
docker run -d \
  --name mysql-db \
  -e MYSQL_ROOT_PASSWORD=secretpassword \
  -e MYSQL_DATABASE=myapp \
  -p 3306:3306 \
  mysql:8.0

# Verify it is running:
docker ps
docker logs mysql-db

# Connect to it:
docker exec -it mysql-db mysql -u root -p
```

You just ran a full MySQL database server in seconds. No installation. No configuration files. No dependencies. Just one command.

**Run a web application:**

```bash
docker run -d \
  --name my-webapp \
  -p 8080:80 \
  -v $(pwd)/html:/usr/share/nginx/html \
  nginx:latest

# -v = volume mount — connects your local folder to container folder
# Changes you make to ./html appear immediately in the container
```

---

### Full Commands Summary — Docker Day 1

| Command | What it does |
|---|---|
| `docker run image` | Create and start container |
| `docker run -d image` | Run in background |
| `docker run -it image bash` | Interactive shell |
| `docker run -p 8080:80 image` | Map ports |
| `docker run --name name image` | Custom container name |
| `docker run --rm image` | Auto-delete on stop |
| `docker ps` | List running containers |
| `docker ps -a` | List all containers |
| `docker stop name` | Stop container |
| `docker rm name` | Remove container |
| `docker logs name` | View logs |
| `docker logs -f name` | Follow logs live |
| `docker exec -it name bash` | Shell inside container |
| `docker images` | List local images |
| `docker pull image:tag` | Download image |
| `docker rmi image` | Remove image |
| `docker inspect name` | Detailed container info |
| `docker stats` | Live resource usage |
| `docker system prune` | Clean up unused resources |

---

### Interview Questions — Docker Day 1

**Q1. What is Docker and why is it used?**
Docker is a containerization platform that packages applications with all their dependencies into portable containers. Used to solve the "works on my machine" problem — containers run identically on any system. Makes deployment consistent, fast and repeatable.

**Q2. What is the difference between a container and a virtual machine?**
VMs run a full guest OS on a hypervisor — several GB each, minutes to start. Containers share the host OS kernel — tens of MB, start in seconds. Containers are lighter and faster but provide process-level isolation. VMs provide complete OS isolation.

**Q3. What is the difference between a Docker image and a container?**
An image is a read-only template — like a blueprint or class definition. A container is a running instance of an image — like an object created from a class. Multiple containers can run from the same image simultaneously.

**Q4. What is Docker Hub?**
Docker Hub is a public registry — cloud storage for Docker images. Official images for nginx, mysql, ubuntu, python and thousands of others are hosted there. `docker pull` downloads from Docker Hub. `docker push` uploads your images there.

**Q5. What does `docker run -d -p 8080:80 nginx` do?**
Runs nginx container in detached mode (background) and maps port 8080 on the host to port 80 inside the container. Accessing `localhost:8080` on your machine reaches nginx running inside the container.

**Q6. What are Docker layers?**
Docker images are built from stacked read-only layers — each Dockerfile instruction creates one layer. Layers are cached and shared between images. If two images use the same base OS layer, Docker stores it only once — saves disk space and speeds up downloads.

**Q7. How do you view logs of a running container?**
`docker logs container_name` shows all logs. `docker logs -f container_name` follows logs live — like `tail -f`. `docker logs --tail 50 container_name` shows last 50 lines.

**Q8. How do you get a shell inside a running container?**
`docker exec -it container_name bash` opens an interactive bash shell inside the running container. Use `sh` instead of `bash` if bash is not available (common in minimal images).

---

### Homework — Before Docker Day 2

```bash
# Practice all of these:
docker run hello-world
docker run -d -p 8080:80 --name my-nginx nginx
docker ps
curl http://localhost:80
docker logs my-nginx
docker exec -it my-nginx bash
# Inside container: ls /etc/nginx/ then exit
docker stats my-nginx
docker stop my-nginx
docker rm my-nginx
docker images
docker system df
docker system prune
```

---

*Next — Docker Day 2: Dockerfile — building your own custom images. This is where Docker goes from running other people's software to packaging YOUR OWN application.*