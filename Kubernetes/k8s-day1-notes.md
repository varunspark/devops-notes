## Kubernetes Day 1 — Introduction & Core Concepts

---

### Why Kubernetes is the Final Piece

You have Docker running containers. You deployed them on EC2. That works for simple applications. But imagine:

- Your application has 10 microservices
- Each running in Docker containers
- Traffic spikes — you need 20 instances of the payment service
- But only 2 instances of the admin service
- One container crashes at 3am — needs to restart automatically
- You push new version — need rolling update with zero downtime
- You have 5 EC2 servers — pack containers efficiently across them

Doing all this manually → impossible at scale

**Kubernetes does all of this automatically**

Kubernetes (K8s) is the container orchestration platform used by every major tech company. Swiggy, Flipkart, Zomato, Amazon, Google — all run on Kubernetes.

---

### What You Will Learn Today

- What Kubernetes is and why it exists
- K8s architecture — master and worker nodes
- The core building blocks — Pod, Deployment, Service
- Namespaces
- kubectl — the K8s CLI
- Setting up a local K8s cluster (minikube)
- Your first Pod and Deployment
- Labels and selectors
- Real DevOps K8s concepts

---

### What is Kubernetes?

Kubernetes is a container orchestration platform — it manages containers at scale automatically.

**Without Kubernetes:**
- You manage: which server each container runs on
- You manage: restarting crashed containers
- You manage: scaling containers up and down
- You manage: rolling updates
- You manage: load balancing between containers
- You manage: resource allocation across servers

**With Kubernetes:**
```
You say: "I want 5 replicas of this container"
K8s does: places them across servers, monitors health,
          restarts failures, rolls updates, load balances
You sleep while K8s works
```

---

### Kubernetes vs Docker

```
Docker:
Runs ONE container on ONE machine
You decide everything manually

Docker Compose:
Runs MULTIPLE containers on ONE machine
Good for development

Kubernetes:
Runs MANY containers across MANY machines
Manages everything automatically
Production at scale
```

---

### K8s Architecture

```
Kubernetes Cluster

Control Plane (Master Node):
├── API Server        → entry point for all commands
├── etcd              → key-value store — cluster state/config
├── Scheduler         → decides which node to run pods on
└── Controller Manager → ensures desired state is maintained

Worker Nodes (where containers actually run):
├── Node 1
│   ├── kubelet       → talks to API server, manages pods
│   ├── kube-proxy    → networking rules
│   └── Pods          → containers running here
├── Node 2
│   └── Pods
└── Node 3
    └── Pods
```

---

### Control Plane Components — What They Do

**API Server:**
- Front door to the cluster
- Everything goes through here
- kubectl sends commands here
- Validates and processes requests

**etcd:**
- Distributed key-value store
- Stores ALL cluster state
- Which pods exist, their status, config
- If etcd dies — cluster loses its memory
- Always back up etcd in production

**Scheduler:**
- Watches for new pods with no node assigned
- Finds best node based on resources, constraints
- "This pod needs 2 CPU and 4GB RAM — put it on Node 3"

**Controller Manager:**
- Runs multiple controllers in one process
- Replication Controller: maintains desired pod count
- Node Controller: notices when nodes go down
- If you want 5 pods and 2 die → creates 2 new ones

---

### Worker Node Components

**kubelet:**
- Agent running on every worker node
- Talks to API server — "what should I be running?"
- Starts, stops, monitors pods on that node
- Reports node and pod status back to API server

**kube-proxy:**
- Networking agent on every node
- Maintains network rules
- Enables Service load balancing
- Routes traffic to correct pods

**Container Runtime:**
- Actually runs containers
- Usually containerd (Docker used to be default)
- Pulls images, starts/stops containers

---

### Core K8s Objects

#### Object 1 — Pod

The smallest deployable unit in Kubernetes. A Pod wraps one or more containers that share:
- Same network namespace (same IP address)
- Same storage volumes
- Always run on the same node

```yaml
# pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
spec:
  containers:
  - name: myapp
    image: varun/myapp:1.0.0
    ports:
    - containerPort: 5000
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
```

**⚠️ You Never Create Pods Directly**

```bash
# WRONG in production:
kubectl create pod myapp    # if pod dies → gone forever

# RIGHT — use Deployment:
kubectl create deployment myapp --image=varun/myapp:1.0.0
# if pod dies → Deployment creates new one automatically
```

Pods are ephemeral — they can die anytime. You manage them through higher-level objects like Deployments.

#### Object 2 — Deployment

A Deployment manages a set of identical Pods. It ensures the desired number of replicas are always running.

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-deployment
  labels:
    app: myapp
spec:
  replicas: 3                  # always keep 3 pods running
  selector:
    matchLabels:
      app: myapp               # manages pods with this label
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1              # max extra pods during update
      maxUnavailable: 1        # max pods down during update
  template:                    # pod template
    metadata:
      labels:
        app: myapp             # pods get this label
    spec:
      containers:
      - name: myapp
        image: varun/myapp:1.0.0
        ports:
        - containerPort: 5000
        resources:
          requests:
            memory: "128Mi"
            cpu: "250m"
          limits:
            memory: "256Mi"
            cpu: "500m"
        livenessProbe:         # restart if fails
          httpGet:
            path: /health
            port: 5000
          initialDelaySeconds: 15
          periodSeconds: 20
        readinessProbe:        # only send traffic when ready
          httpGet:
            path: /health
            port: 5000
          initialDelaySeconds: 5
          periodSeconds: 10
```

**Liveness vs Readiness Probe**

```
# Liveness Probe:
# Is the container alive?
# If fails → kubelet RESTARTS the container
# Use for: detecting deadlocks, infinite loops

# Readiness Probe:
# Is the container READY to receive traffic?
# If fails → pod removed from Service load balancer
#            (NOT restarted — just no traffic)
# Use for: app still starting up, temporary unavailability

# Both failing:
# Liveness fails → restart
# Readiness fails → remove from rotation until recovered

# Real scenario:
# App starts → readiness fails (not ready yet)
# → no traffic sent → app finishes starting
# → readiness passes → traffic starts flowing
```

#### Object 3 — Service

Pods have ephemeral IPs — they change every time a pod restarts. A Service provides a stable endpoint to reach pods.

```yaml
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  selector:
    app: myapp           # routes traffic to pods with this label
  ports:
  - protocol: TCP
    port: 80             # Service port (clients connect here)
    targetPort: 5000     # Pod port (traffic forwarded here)
  type: ClusterIP        # internal only
```

**Three Service Types**

```
# ClusterIP (default):
# Only accessible within the cluster
# Pods talk to each other using ClusterIP services
# Example: frontend → backend service

# NodePort:
# Exposes service on each node's IP at a static port (30000-32767)
# Access: http://node-ip:30080
# Not recommended for production (port range ugly)

# LoadBalancer:
# Creates a cloud load balancer (AWS ALB/ELB)
# Gets external IP automatically
# Production standard for internet-facing services
# Cost: one cloud load balancer per service

# Difference in YAML — just change type:
spec:
  type: ClusterIP    # internal
  type: NodePort     # node IP + high port
  type: LoadBalancer # cloud load balancer
```

#### Object 4 — Namespace

Namespaces provide logical isolation within a cluster:

```bash
# Default namespaces:
kubectl get namespaces
# default        → where your apps go without specifying
# kube-system    → K8s system components
# kube-public    → publicly readable config
# kube-node-lease → node heartbeats

# Create namespace:
kubectl create namespace production
kubectl create namespace staging

# Deploy to specific namespace:
kubectl apply -f deployment.yaml -n production
kubectl apply -f deployment.yaml -n staging

# Different config per namespace:
# staging/DB_HOST = staging-db
# production/DB_HOST = prod-db
```

---

### Labels and Selectors

**What are Labels?**

Labels are key-value pairs attached to K8s objects. They are how objects find and relate to each other.

```yaml
metadata:
  labels:
    app: myapp
    version: "1.0"
    environment: production
    tier: frontend
```

**Selectors — Finding Objects by Labels**

```bash
# List pods with specific label:
kubectl get pods -l app=myapp
kubectl get pods -l environment=production
kubectl get pods -l app=myapp,tier=frontend

# Service finds pods using selector:
spec:
  selector:
    app: myapp      # routes to all pods with app=myapp label

# Deployment manages pods using selector:
spec:
  selector:
    matchLabels:
      app: myapp    # manages all pods with app=myapp label
```

---

### Setting Up Local Kubernetes — Minikube

```bash
# Install minikube (local single-node K8s cluster):
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube

# Install kubectl (K8s CLI):
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# Start minikube:
minikube start --driver=docker

# Verify cluster is running:
kubectl cluster-info
kubectl get nodes
```

---

### Your First Deployment

```bash
# Create deployment:
kubectl create deployment myapp \
    --image=nginx:alpine \
    --replicas=3

# Check it:
kubectl get deployments
kubectl get pods
kubectl get pods -o wide   # shows which node each pod is on

# Watch pods in real time:
kubectl get pods --watch

# Describe a pod (detailed info):
kubectl describe pod <pod-name>

# View pod logs:
kubectl logs <pod-name>
kubectl logs <pod-name> -f    # follow live

# Execute command inside pod:
kubectl exec -it <pod-name> -- bash
kubectl exec -it <pod-name> -- sh   # if no bash
```

---

### Expose Deployment as Service

```bash
# Expose deployment:
kubectl expose deployment myapp \
    --type=NodePort \
    --port=80

# Check service:
kubectl get services

# Get URL in minikube:
minikube service myapp --url
# http://192.168.49.2:30080  ← open in browser

# Scale up:
kubectl scale deployment myapp --replicas=5
kubectl get pods   # now 5 pods

# Scale down:
kubectl scale deployment myapp --replicas=2
kubectl get pods   # back to 2 pods
```

---

### Applying YAML Files

```bash
# Create directory for manifests:
mkdir k8s-manifests
cd k8s-manifests

# deployment.yaml:
cat > deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  labels:
    app: myapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: nginx:alpine
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "64Mi"
            cpu: "250m"
          limits:
            memory: "128Mi"
            cpu: "500m"
EOF

# service.yaml:
cat > service.yaml << 'EOF'
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  selector:
    app: myapp
  ports:
  - port: 80
    targetPort: 80
  type: NodePort
EOF

# Apply both:
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml

# Apply entire directory:
kubectl apply -f .

# Delete resources:
kubectl delete -f .
```

---

### Rolling Update — Zero Downtime

```bash
# Update image version:
kubectl set image deployment/myapp \
    myapp=nginx:1.24 \
    --record

# Watch rollout:
kubectl rollout status deployment/myapp

# See rollout history:
kubectl rollout history deployment/myapp

# Rollback to previous version:
kubectl rollout undo deployment/myapp

# Rollback to specific revision:
kubectl rollout undo deployment/myapp --to-revision=2
```

---

### Essential kubectl Commands

```bash
# ── Get resources ───────────────────────────────
kubectl get pods                      # list pods
kubectl get pods -o wide              # with node info
kubectl get pods -A                   # all namespaces
kubectl get deployments
kubectl get services
kubectl get nodes
kubectl get all                       # everything

# ── Describe ─────────────────────────────────────
kubectl describe pod <name>           # detailed pod info
kubectl describe deployment <name>
kubectl describe node <name>

# ── Logs ─────────────────────────────────────────
kubectl logs <pod-name>
kubectl logs <pod-name> -f            # follow live
kubectl logs <pod-name> --tail=50
kubectl logs <pod-name> -c <container> # multi-container pod

# ── Execute ──────────────────────────────────────
kubectl exec -it <pod-name> -- bash
kubectl exec <pod-name> -- ls /app

# ── Apply/Delete ─────────────────────────────────
kubectl apply -f file.yaml
kubectl delete -f file.yaml
kubectl delete pod <name>
kubectl delete deployment <name>

# ── Scale ────────────────────────────────────────
kubectl scale deployment <name> --replicas=5

# ── Port forward ─────────────────────────────────
kubectl port-forward pod/<name> 8080:5000
kubectl port-forward service/<name> 8080:80
# Access on http://localhost:8080

# ── Context ──────────────────────────────────────
kubectl config get-contexts            # list clusters
kubectl config use-context <name>      # switch cluster
kubectl config current-context         # current cluster
```

---

### K8s vs Docker Compose Comparison

```yaml
# Docker Compose:
services:
  app:
    image: myapp:1.0
    ports:
      - "80:5000"
    replicas: 3

# Kubernetes equivalent:
# deployment.yaml:
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    spec:
      containers:
      - name: app
        image: myapp:1.0
        ports:
        - containerPort: 5000

# service.yaml:
apiVersion: v1
kind: Service
metadata:
  name: app
spec:
  selector:
    app: myapp
  ports:
  - port: 80
    targetPort: 5000
```

---

### Full Summary — K8s Day 1

| Concept | Key point |
|---|---|
| Kubernetes | Container orchestration — manages containers at scale |
| Control Plane | API Server, etcd, Scheduler, Controller Manager |
| Worker Node | kubelet, kube-proxy, container runtime |
| Pod | Smallest unit — wraps one or more containers |
| Deployment | Manages pods — ensures desired replica count |
| Service | Stable endpoint to reach pods — ClusterIP/NodePort/LB |
| Namespace | Logical isolation within cluster |
| Labels | Key-value tags on objects |
| Selectors | Find objects by labels |
| Liveness Probe | Restart container if fails |
| Readiness Probe | Remove from traffic if not ready |
| kubectl apply | Create or update from YAML |
| kubectl get | List resources |
| kubectl describe | Detailed resource info |
| kubectl logs | View container output |
| Rolling update | Zero-downtime image update |

---

### Interview Questions — K8s Day 1

**Q1. What is Kubernetes and why is it needed?**
Kubernetes is a container orchestration platform that automates deployment, scaling and management of containerized applications. Needed because Docker alone manages one container on one machine — Kubernetes manages thousands of containers across hundreds of machines. It handles automatic restarts when containers crash, scales based on demand, rolls updates with zero downtime and efficiently packs containers across servers.

**Q2. What is the difference between a Pod and a Deployment?**
A Pod is the smallest deployable unit — wraps one or more containers. If a Pod dies it is gone permanently. A Deployment is a higher-level object that manages a set of identical Pods — it ensures the desired number of replicas always run. If a Pod dies the Deployment automatically creates a replacement. Always use Deployments in production — never create Pods directly.

**Q3. What are the three types of Kubernetes Services?**
ClusterIP is internal only — pods communicate with each other within the cluster. NodePort exposes the service on each node's IP at a static port between 30000-32767 — accessible from outside but with an awkward port. LoadBalancer creates a cloud load balancer (AWS ALB) with an external IP — the production standard for internet-facing services. Each type is specified in the service YAML with the type field.

**Q4. What is the difference between liveness and readiness probes?**
Liveness probe checks if the container is alive — if it fails, kubelet restarts the container. Used to detect deadlocks or frozen processes. Readiness probe checks if the container is ready to receive traffic — if it fails the pod is removed from the Service load balancer but NOT restarted. Used during startup when the app needs time to initialize or temporarily when the app is under heavy load.

**Q5. What is etcd in Kubernetes?**
etcd is a distributed key-value store that stores the entire cluster state — all configurations, which pods exist, their status, secrets, config maps. It is the source of truth for the cluster. If etcd is lost the cluster loses all its state. Always back up etcd in production. It runs as part of the Control Plane.

**Q6. What is a Namespace in Kubernetes?**
A Namespace provides logical isolation within a cluster — like virtual clusters. Used to separate environments (dev, staging, production) within one physical cluster, or separate teams and applications. Resources in different namespaces are isolated — same name can exist in multiple namespaces. Default namespace is used when none is specified.

---

### Homework — Before K8s Day 2

```bash
# Install minikube and kubectl
minikube start --driver=docker

# Deploy your Flask app from CI/CD days:
kubectl create deployment myapp \
    --image=varun/myapp:latest

# Expose it:
kubectl expose deployment myapp \
    --type=NodePort \
    --port=5000

# Scale up and watch:
kubectl scale deployment myapp --replicas=5
kubectl get pods --watch

# Update image and watch rolling update:
kubectl set image deployment/myapp myapp=nginx:alpine
kubectl rollout status deployment/myapp

# Rollback:
kubectl rollout undo deployment/myapp

# Delete a pod manually — watch K8s recreate it:
kubectl delete pod <pod-name>
kubectl get pods --watch   # new pod appears automatically

# Write deployment.yaml and service.yaml from scratch
# Apply them: kubectl apply -f .
```

---

### Your Progress

```
Linux  ████████████████████  ✅ COMPLETE
Git    ████████████████████  ✅ COMPLETE
Docker ████████████████████  ✅ COMPLETE
AWS    ████████████████████  ✅ COMPLETE
CI/CD  ████████████████████  ✅ COMPLETE
K8s    ██░░░░░░░░░░░░░░░░░░  Day 1 of 6
Final  ░░░░░░░░░░░░░░░░░░░░  Interview prep — after K8s
```

*Next — K8s Day 2: ConfigMaps, Secrets, Persistent Volumes and Resource Management — how to configure applications properly in Kubernetes, store sensitive data securely and give containers exactly the right amount of resources.* 💪
