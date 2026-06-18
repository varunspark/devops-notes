## Kubernetes Day 3 — Ingress, Services Deep Dive, HPA & Helm

---

### Why This is Critical

You can deploy pods, configure them, give them storage and resource limits. But two real production problems remain:

```
Problem 1 — Traffic routing:
You have 10 microservices, each needs its own URL path or domain
api.myapp.com      → backend service
myapp.com          → frontend service
admin.myapp.com    → admin service
One LoadBalancer per service = expensive, hard to manage

Problem 2 — Scaling on demand:
Traffic spikes at 9am, drops at midnight
Manually running "kubectl scale" every time = not sustainable
Need pods to scale themselves based on real CPU/memory usage

Problem 3 — Deployment complexity:
Every app needs ConfigMap + Secret + Deployment + Service + Ingress YAML
Copy-pasting and editing YAML for every environment = error prone
Need a way to package and template all of this
```

Ingress solves traffic routing. HPA solves automatic scaling. Helm solves packaging and templating.

---

### What You Will Learn Today

- Services deep dive — how kube-proxy actually routes traffic
- Ingress and Ingress Controllers
- Path-based and host-based routing
- TLS/HTTPS with Ingress
- Horizontal Pod Autoscaler (HPA)
- Metrics Server
- Helm — package manager for Kubernetes
- Helm charts, values, templates
- Real DevOps production patterns

---

## Services Deep Dive

---

### How Services Actually Work

A Service is not a process or a pod — it's a virtual IP plus routing rules maintained by kube-proxy on every node.

```
1. You create a Service with selector app=myapp
2. K8s control plane finds all pods matching that label
3. Creates an "Endpoints" object — list of pod IPs:ports
4. kube-proxy on every node updates iptables/IPVS rules
5. Any traffic to the Service's virtual IP gets load-balanced
   across the pod IPs in the Endpoints list
6. As pods die/start, Endpoints updates automatically
```

```bash
# See the Endpoints behind a Service:
kubectl get endpoints myapp-service

# See which pods a Service is currently routing to:
kubectl describe service myapp-service
```

---

### Service Discovery — DNS

Every Service automatically gets a DNS name inside the cluster:

```
Format: <service-name>.<namespace>.svc.cluster.local

Examples:
myapp-service.default.svc.cluster.local
mysql-service.production.svc.cluster.local

Shortcuts (within same namespace):
myapp-service                    # same namespace
myapp-service.production         # different namespace, short form
```

```yaml
# Pod in another namespace connecting to a service:
env:
- name: DB_HOST
  value: mysql-service.production.svc.cluster.local
```

---

### Headless Services

A normal Service load-balances and gives one virtual IP. A headless Service returns the IPs of all individual pods directly — used by StatefulSets so each pod gets a stable, addressable DNS name.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql-headless
spec:
  clusterIP: None        # this makes it headless
  selector:
    app: mysql
  ports:
  - port: 3306
```

```bash
# DNS resolves directly to each pod:
mysql-0.mysql-headless.default.svc.cluster.local
mysql-1.mysql-headless.default.svc.cluster.local
mysql-2.mysql-headless.default.svc.cluster.local
```

---

### Multi-Port Services

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  selector:
    app: myapp
  ports:
  - name: http
    protocol: TCP
    port: 80
    targetPort: 5000
  - name: metrics
    protocol: TCP
    port: 9090
    targetPort: 9090
```

---

## Ingress

---

### What is Ingress?

A Service of type LoadBalancer gives you one external IP per service — expensive and hard to manage at scale. Ingress lets one single load balancer route to many services based on hostname or URL path.

```
Without Ingress:
api.myapp.com    → LoadBalancer 1 → backend service    ($$$)
myapp.com        → LoadBalancer 2 → frontend service   ($$$)
admin.myapp.com  → LoadBalancer 3 → admin service       ($$$)

With Ingress:
api.myapp.com    ┐
myapp.com        ├─→ ONE LoadBalancer → Ingress Controller → routes to correct service
admin.myapp.com  ┘
```

---

### Ingress Needs an Ingress Controller

Ingress is just a set of routing rules — it does nothing by itself. You need an Ingress Controller (a pod running a reverse proxy like NGINX) that reads those rules and actually does the routing.

```bash
# Most common Ingress Controllers:
# - ingress-nginx (most widely used, open source)
# - AWS Load Balancer Controller (native ALB integration on EKS)
# - Traefik
# - HAProxy

# Install ingress-nginx via Helm (common method):
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm install ingress-nginx ingress-nginx/ingress-nginx \
    --namespace ingress-nginx --create-namespace

# Verify controller is running:
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx   # note the external IP
```

---

### Host-Based Routing

```yaml
# ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: myapp.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80
  - host: api.myapp.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: backend-service
            port:
              number: 80
```

---

### Path-Based Routing

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
spec:
  ingressClassName: nginx
  rules:
  - host: myapp.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: backend-service
            port:
              number: 80
      - path: /admin
        pathType: Prefix
        backend:
          service:
            name: admin-service
            port:
              number: 80
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80
```

```bash
# pathType options:
# Exact   → matches the exact path only
# Prefix  → matches the path and everything under it
# ImplementationSpecific → controller decides
```

---

### TLS / HTTPS with Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - myapp.com
    secretName: myapp-tls-secret     # must contain tls.crt and tls.key
  rules:
  - host: myapp.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80
```

```bash
# cert-manager automates TLS certificate creation/renewal
# from Let's Encrypt — install it via Helm, then reference
# the ClusterIssuer in your Ingress annotations

helm repo add jetstack https://charts.jetstack.io
helm install cert-manager jetstack/cert-manager \
    --namespace cert-manager --create-namespace \
    --set installCRDs=true
```

---

### Useful Ingress Commands

```bash
kubectl get ingress
kubectl describe ingress myapp-ingress
kubectl get ingress -A

# Test routing locally (edit /etc/hosts to point domain to minikube/cluster IP)
minikube ip
echo "192.168.49.2 myapp.com" | sudo tee -a /etc/hosts
```

---

## Horizontal Pod Autoscaler (HPA)

---

### What is HPA?

HPA automatically increases or decreases the number of pod replicas in a Deployment based on observed metrics like CPU or memory usage.

```
Without HPA:
Traffic spikes → app slows down or crashes → you manually
                 run "kubectl scale" at 2am

With HPA:
Traffic spikes → CPU usage crosses threshold → HPA automatically
                 adds more pods → traffic drops → HPA removes pods
```

---

### Prerequisite — Metrics Server

HPA needs real-time CPU/memory data, which comes from the Metrics Server.

```bash
# Install Metrics Server:
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Verify it is working:
kubectl top nodes
kubectl top pods
# If these commands return data, Metrics Server is working
```

---

### Creating an HPA

```bash
# Imperative command:
kubectl autoscale deployment myapp \
    --cpu-percent=50 \
    --min=2 \
    --max=10

# This means:
# Keep average CPU usage across pods at 50% of requested CPU
# Never go below 2 pods
# Never go above 10 pods
```

```yaml
# hpa.yaml — declarative version
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 70
```

---

### ⚠️ HPA Needs Resource Requests Set

```bash
# HPA calculates percentage based on requests, not limits
# No requests set → HPA cannot calculate utilization → won't work

spec:
  containers:
  - name: myapp
    resources:
      requests:
        cpu: "250m"        # HPA target % is based on THIS value
      limits:
        cpu: "500m"
```

---

### Watching HPA in Action

```bash
kubectl get hpa
kubectl get hpa myapp-hpa --watch

kubectl describe hpa myapp-hpa
# Shows current vs target metrics, and recent scaling events

# Simulate load to test scaling (run inside a temporary pod):
kubectl run load-generator --image=busybox --rm -it -- /bin/sh
# Inside: while true; do wget -q -O- http://myapp-service; done
```

---

### HPA Scaling Behavior

```yaml
# Control how fast HPA scales up/down (avoid flapping):
spec:
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0       # scale up immediately
      policies:
      - type: Pods
        value: 4
        periodSeconds: 60                 # max 4 pods added per minute
    scaleDown:
      stabilizationWindowSeconds: 300      # wait 5 min before scaling down
      policies:
      - type: Percent
        value: 50
        periodSeconds: 60                 # remove max 50% of pods per minute
```

```bash
# Why scaleDown is more cautious than scaleUp:
# Scaling up fast = handle traffic spike, no harm if a bit excessive
# Scaling down fast = risk killing pods needed seconds later
# Default stabilization window protects against "flapping"
```

---

### Vertical Pod Autoscaler (VPA) — Quick Note

```bash
# HPA changes the NUMBER of pods
# VPA changes the CPU/memory REQUESTS of existing pods
# VPA is a separate add-on, less commonly used in production
# (resizing requires pod restart in most setups)
# HPA + VPA together for CPU/memory is generally NOT recommended
# (they can fight each other)
```

---

## Helm

---

### What is Helm?

Helm is the package manager for Kubernetes — like apt for Ubuntu or npm for Node.js. It lets you package ConfigMaps, Secrets, Deployments, Services and Ingress into one reusable, versioned, templated unit called a Chart.

```
Without Helm:
Maintain separate deployment.yaml, service.yaml, configmap.yaml,
ingress.yaml for every environment (dev/staging/prod)
Copy-paste and manually edit values for each — error prone

With Helm:
One chart, different values.yaml per environment
helm install myapp ./mychart -f values-prod.yaml
One command deploys everything, consistently
```

---

### Installing Helm

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

helm version

# Add a public chart repository:
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# Search for charts:
helm search repo nginx
helm search hub wordpress
```

---

### Installing a Public Chart

```bash
# Install nginx from Bitnami repo:
helm install my-nginx bitnami/nginx

# Install with custom values:
helm install my-nginx bitnami/nginx \
    --set service.type=LoadBalancer \
    --set replicaCount=3

# List installed releases:
helm list
helm list -A

# Uninstall a release:
helm uninstall my-nginx
```

---

### Chart Directory Structure

```bash
# Create a new chart:
helm create mychart

mychart/
├── Chart.yaml          # chart metadata — name, version, description
├── values.yaml         # default configuration values
├── charts/             # dependency charts (sub-charts)
├── templates/          # YAML templates with placeholders
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── configmap.yaml
│   ├── _helpers.tpl    # reusable template snippets
│   └── NOTES.txt       # shown to user after install
└── .helmignore
```

---

### Chart.yaml

```yaml
apiVersion: v2
name: myapp
description: A Helm chart for myapp
type: application
version: 1.0.0          # chart version
appVersion: "1.0.0"     # version of the app it deploys
```

---

### values.yaml — Default Configuration

```yaml
# values.yaml
replicaCount: 3

image:
  repository: varun/myapp
  tag: "1.0.0"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80
  targetPort: 5000

resources:
  requests:
    cpu: 250m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 256Mi

ingress:
  enabled: true
  host: myapp.com

env:
  APP_ENV: production
  LOG_LEVEL: info
```

---

### Template — deployment.yaml

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-myapp
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Release.Name }}-myapp
  template:
    metadata:
      labels:
        app: {{ .Release.Name }}-myapp
    spec:
      containers:
      - name: myapp
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        ports:
        - containerPort: {{ .Values.service.targetPort }}
        resources:
          requests:
            cpu: {{ .Values.resources.requests.cpu }}
            memory: {{ .Values.resources.requests.memory }}
          limits:
            cpu: {{ .Values.resources.limits.cpu }}
            memory: {{ .Values.resources.limits.memory }}
        env:
        {{- range $key, $value := .Values.env }}
        - name: {{ $key }}
          value: "{{ $value }}"
        {{- end }}
```

```bash
# {{ .Values.X }}       → pulls value from values.yaml
# {{ .Release.Name }}   → name given during "helm install <name>"
# {{- range }} {{- end }} → loops over a list/map
# Conditionals: {{- if .Values.ingress.enabled }} ... {{- end }}
```

---

### Installing Your Own Chart

```bash
# Validate templates render correctly without installing:
helm template myapp ./mychart

# Dry run — simulate install, see what would be created:
helm install myapp ./mychart --dry-run --debug

# Actually install:
helm install myapp ./mychart

# Install with custom values file (per environment):
helm install myapp ./mychart -f values-prod.yaml

# Override individual value from CLI:
helm install myapp ./mychart --set replicaCount=5
```

---

### Upgrading and Rolling Back Releases

```bash
# Upgrade after changing values.yaml or templates:
helm upgrade myapp ./mychart

# Upgrade with new values:
helm upgrade myapp ./mychart --set image.tag=2.0.0

# See release history:
helm history myapp

# Rollback to previous release:
helm rollback myapp

# Rollback to specific revision:
helm rollback myapp 2

# See what would change before upgrading:
helm diff upgrade myapp ./mychart    # requires helm-diff plugin
```

---

### Helm Values Per Environment — Real Pattern

```bash
mychart/
├── values.yaml           # shared defaults
├── values-dev.yaml       # dev overrides
├── values-staging.yaml   # staging overrides
└── values-prod.yaml      # prod overrides
```

```yaml
# values-prod.yaml
replicaCount: 5
image:
  tag: "1.0.0"
resources:
  requests:
    cpu: 500m
    memory: 256Mi
```

```bash
# Deploy to each environment:
helm install myapp-dev ./mychart -f values-dev.yaml -n dev
helm install myapp-staging ./mychart -f values-staging.yaml -n staging
helm install myapp-prod ./mychart -f values-prod.yaml -n production
```

---

### Essential Helm Commands

```bash
# ── Repos ─────────────────────────────────────
helm repo add <name> <url>
helm repo update
helm search repo <keyword>

# ── Install/Upgrade/Uninstall ───────────────────
helm install <release-name> <chart>
helm upgrade <release-name> <chart>
helm uninstall <release-name>

# ── Inspect ──────────────────────────────────────
helm list
helm list -A
helm status <release-name>
helm get values <release-name>
helm get manifest <release-name>

# ── Debug ─────────────────────────────────────────
helm template <chart>
helm install <release> <chart> --dry-run --debug
helm lint ./mychart            # check chart for errors

# ── History/Rollback ─────────────────────────────
helm history <release-name>
helm rollback <release-name> <revision>
```

---

### Full Summary — K8s Day 3

| Concept | Key point |
|---|---|
| Service (deep dive) | Virtual IP + kube-proxy rules, backed by Endpoints |
| Headless Service | No cluster IP — DNS resolves to individual pod IPs |
| Ingress | Routes HTTP/HTTPS traffic to services by host/path |
| Ingress Controller | Actual proxy (e.g. NGINX) that implements Ingress rules |
| TLS on Ingress | cert-manager automates Let's Encrypt certificates |
| HPA | Auto-scales pod replica count based on CPU/memory |
| Metrics Server | Required for HPA — provides live resource metrics |
| VPA | Adjusts pod resource requests, not replica count |
| Helm | Package manager for K8s — Charts bundle all manifests |
| values.yaml | Configuration injected into chart templates |
| helm upgrade/rollback | Versioned deploys with easy rollback |

---

### Interview Questions — K8s Day 3

**Q1. What is the difference between a Service and an Ingress?**
A Service provides a stable internal (or external, via LoadBalancer/NodePort) endpoint to reach a set of pods using IP-level routing — Layer 4. An Ingress operates at Layer 7 (HTTP/HTTPS) and routes traffic to different services based on hostname or URL path, using a single external entry point. Ingress requires an Ingress Controller to actually function.

**Q2. What is an Ingress Controller and why is it required?**
An Ingress resource is just a set of routing rules stored in the K8s API — it does nothing on its own. An Ingress Controller is a running pod (commonly NGINX) that watches Ingress resources and configures an actual reverse proxy to implement that routing. Without an Ingress Controller installed, Ingress resources have no effect.

**Q3. How does the Horizontal Pod Autoscaler work?**
HPA periodically checks resource metrics (CPU/memory) from the Metrics Server and compares them against a target utilization percentage defined relative to the pod's resource requests. If usage exceeds the target, HPA increases replica count up to maxReplicas; if usage drops, it decreases replicas down to minReplicas. It requires resource requests to be set on the container, since percentages are calculated from requests.

**Q4. Why does HPA need resource requests to be configured?**
HPA's target utilization is a percentage of the requested resource value, not the limit. Without a CPU or memory request defined on the container, K8s has no baseline to calculate "current usage as a percentage of requested," so the HPA cannot compute a target and scaling won't trigger correctly.

**Q5. What is Helm and what problem does it solve?**
Helm is a package manager for Kubernetes. It bundles related manifests (Deployment, Service, ConfigMap, Ingress, etc.) into a single versioned unit called a Chart, with a values.yaml file controlling configuration. This avoids maintaining and manually editing multiple raw YAML files per environment, supports templating, and provides built-in versioning, upgrade and rollback capabilities via helm upgrade and helm rollback.

**Q6. What is the difference between a headless Service and a normal Service?**
A normal Service is assigned a single virtual ClusterIP, and traffic to it is load-balanced across matching pods. A headless Service (clusterIP: None) has no virtual IP — DNS queries for it return the IP addresses of all matching pods directly. This is used by StatefulSets so that each pod gets its own stable, individually addressable DNS name.

**Q7. How do you safely roll back a Helm release?**
`helm history <release-name>` shows all previous revisions of a release. `helm rollback <release-name> <revision-number>` reverts the release to that specific revision's chart and values, recreating the Kubernetes resources accordingly. Running `helm rollback <release-name>` without a revision number reverts to the immediately previous release.

---

### Homework — Before K8s Day 4

```bash
# 1. Install ingress-nginx via Helm and verify it's running:
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm install ingress-nginx ingress-nginx/ingress-nginx \
    --namespace ingress-nginx --create-namespace
kubectl get pods -n ingress-nginx

# 2. Create an Ingress with host-based routing for two services
#    of your choice, and test it locally via /etc/hosts

# 3. Install Metrics Server and verify:
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
kubectl top pods

# 4. Create an HPA for your myapp deployment:
kubectl autoscale deployment myapp --cpu-percent=50 --min=2 --max=6
kubectl get hpa --watch
# Generate load and observe scaling

# 5. Create your own Helm chart from scratch:
helm create mychart
# Edit values.yaml and templates/deployment.yaml to deploy your app
helm install myapp ./mychart --dry-run --debug
helm install myapp ./mychart

# 6. Practice an upgrade and a rollback:
helm upgrade myapp ./mychart --set image.tag=2.0.0
helm rollback myapp
```

---

### Your Progress

```
Linux  ████████████████████  ✅ COMPLETE
Git    ████████████████████  ✅ COMPLETE
Docker ████████████████████  ✅ COMPLETE
AWS    ████████████████████  ✅ COMPLETE
CI/CD  ████████████████████  ✅ COMPLETE
K8s    ██████░░░░░░░░░░░░░░  Day 3 of 6
Final  ░░░░░░░░░░░░░░░░░░░░  Interview prep
```

K8s Day 4 we learn **RBAC, Service Accounts, Network Policies and Security Context** — how to control who can do what in a cluster, restrict pod-to-pod traffic and lock down container privileges for production-grade security. 💪

Say **"K8s Day 4"** whenever you are ready!
