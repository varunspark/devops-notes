## Kubernetes Day 2 — ConfigMaps, Secrets, Volumes & Resource Management

---

### Why This is Critical

Your Pod is running. But how does it get its configuration? Database hostname, API keys, environment-specific settings?

```
Bad way:
Hardcode in Docker image → rebuild image for every config change
Store password in Dockerfile → everyone with image access sees it

Good way (Kubernetes):
ConfigMap  → non-sensitive config (DB host, port, log level)
Secret     → sensitive data (passwords, API keys, certificates)
Volume     → persistent storage (database files, uploads)
Resources  → CPU and memory limits (prevent one pod killing the node)
```

---

### What You Will Learn Today

- ConfigMaps — store and inject configuration
- Secrets — store sensitive data securely
- Environment variables vs volume mounts
- Persistent Volumes and Persistent Volume Claims
- Storage Classes
- Resource requests and limits
- LimitRange and ResourceQuota
- Real DevOps configuration patterns

---

## ConfigMaps

---

### What is a ConfigMap?

A ConfigMap stores **non-sensitive configuration data** as key-value pairs. Keeps config separate from the container image.

```yaml
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: myapp-config
  namespace: default
data:
  # Simple key-value pairs:
  APP_ENV: production
  LOG_LEVEL: info
  DB_HOST: mysql-service
  DB_PORT: "3306"
  DB_NAME: myappdb

  # Multi-line value (config file):
  nginx.conf: |
    server {
      listen 80;
      location / {
        proxy_pass http://localhost:5000;
      }
    }

  # JSON value:
  app-settings.json: |
    {
      "maxConnections": 100,
      "timeout": 30,
      "retries": 3
    }
```

```bash
# Create ConfigMap:
kubectl apply -f configmap.yaml

# Create from literal values:
kubectl create configmap myapp-config \
    --from-literal=APP_ENV=production \
    --from-literal=LOG_LEVEL=info \
    --from-literal=DB_HOST=mysql-service

# Create from file:
kubectl create configmap nginx-config \
    --from-file=nginx.conf

# Create from directory (all files become keys):
kubectl create configmap app-configs \
    --from-file=./config/

# View ConfigMap:
kubectl get configmaps
kubectl describe configmap myapp-config
kubectl get configmap myapp-config -o yaml
```

---

### Using ConfigMap — Method 1: Environment Variables

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 2
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
        image: varun/myapp:1.0.0

        # Method 1a — inject all keys as env vars:
        envFrom:
        - configMapRef:
            name: myapp-config

        # Method 1b — inject specific keys:
        env:
        - name: DATABASE_HOST
          valueFrom:
            configMapKeyRef:
              name: myapp-config
              key: DB_HOST
        - name: LOG_LEVEL
          valueFrom:
            configMapKeyRef:
              name: myapp-config
              key: LOG_LEVEL
```

---

### Using ConfigMap — Method 2: Volume Mount (Files)

```yaml
spec:
  containers:
  - name: myapp
    image: varun/myapp:1.0.0
    volumeMounts:
    - name: config-volume
      mountPath: /etc/config        # files appear here
      readOnly: true

  volumes:
  - name: config-volume
    configMap:
      name: myapp-config
      # Each key becomes a file at /etc/config/KEY_NAME
      # nginx.conf → /etc/config/nginx.conf
      # app-settings.json → /etc/config/app-settings.json
```

---

### Env Vars vs Volume Mount — When to Use Which

```bash
# Use environment variables when:
# → Simple key=value config
# → App reads from environment (os.environ in Python)
# → Config values are short strings

# Use volume mount when:
# → Config file format required (nginx.conf, app.yaml)
# → Large config data
# → App reads from file path
# → Config needs to be updated without pod restart
#    (volume mounts update automatically — env vars don't)

# Key difference:
# env vars: fixed at pod start — need pod restart to update
# volume mount: updated automatically when ConfigMap changes
#               (within ~60 seconds — no pod restart needed)
```

---

## Secrets

---

### What is a Secret?

A Secret stores **sensitive data** — passwords, API keys, certificates, SSH keys. Similar to ConfigMap but:

```bash
# Differences from ConfigMap:
# - Base64 encoded (NOT encrypted by default — just encoded)
# - Not shown in kubectl describe (values masked)
# - Can be encrypted at rest with KMS (encryption config)
# - Access controlled separately with RBAC
# - Stored in etcd — encrypt etcd for real security
```

---

### Creating Secrets

```bash
# From literal values (most common):
kubectl create secret generic myapp-secrets \
    --from-literal=DB_PASSWORD=supersecret123 \
    --from-literal=API_KEY=abc123xyz \
    --from-literal=SECRET_KEY=mysecretkey

# From file (for certificates, SSH keys):
kubectl create secret generic tls-certs \
    --from-file=tls.crt=./server.crt \
    --from-file=tls.key=./server.key

# TLS secret (special type for HTTPS):
kubectl create secret tls myapp-tls \
    --cert=server.crt \
    --key=server.key

# Docker registry secret (pull from private registry):
kubectl create secret docker-registry ecr-secret \
    --docker-server=123456789.dkr.ecr.ap-south-1.amazonaws.com \
    --docker-username=AWS \
    --docker-password=$(aws ecr get-login-password)
```

---

### Secret YAML

```yaml
# secret.yaml — values must be base64 encoded
apiVersion: v1
kind: Secret
metadata:
  name: myapp-secrets
type: Opaque
data:
  DB_PASSWORD: c3VwZXJzZWNyZXQxMjM=    # base64 of "supersecret123"
  API_KEY: YWJjMTIzeHl6             # base64 of "abc123xyz"

# Encode: echo -n "supersecret123" | base64
# Decode: echo "c3VwZXJzZWNyZXQxMjM=" | base64 --decode
```

---

### Using Secrets in Pod

```yaml
spec:
  containers:
  - name: myapp
    image: varun/myapp:1.0.0

    # Method 1 — all keys as env vars:
    envFrom:
    - secretRef:
        name: myapp-secrets

    # Method 2 — specific keys:
    env:
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: myapp-secrets
          key: DB_PASSWORD

    # Method 3 — mount as files:
    volumeMounts:
    - name: secrets-volume
      mountPath: /run/secrets
      readOnly: true

  volumes:
  - name: secrets-volume
    secret:
      secretName: myapp-secrets
      defaultMode: 0400    # read only by owner
```

---

### ⚠️ Secret Security Reality

```bash
# Secrets are base64 encoded — NOT encrypted:
kubectl get secret myapp-secrets -o yaml
# Shows base64 values — anyone with kubectl access can decode

# For real security:
# 1. Enable encryption at rest in etcd (K8s encryption config)
# 2. Use External Secrets Operator + AWS Secrets Manager
# 3. Use Vault by HashiCorp
# 4. Restrict RBAC — who can read secrets

# External Secrets Operator (production best practice):
# Syncs from AWS Secrets Manager → K8s Secret automatically
# Rotation happens in Secrets Manager → K8s Secret updates
# No secrets stored in Git or etcd
```

---

### Pull Image from ECR Using Secret

```yaml
spec:
  imagePullSecrets:
  - name: ecr-secret          # uses the docker-registry secret
  containers:
  - name: myapp
    image: 123456789.dkr.ecr.ap-south-1.amazonaws.com/myapp:1.0.0
```

---

## Persistent Volumes

---

### The Storage Problem

```bash
# Container storage is ephemeral:
kubectl exec -it mypod -- bash
echo "important data" > /data/file.txt
exit
kubectl delete pod mypod
# Pod recreated — /data/file.txt is GONE

# Solution: Persistent Volumes
# Storage that exists independently of pods
# Pod can be deleted and recreated — data survives
```

---

### Three Storage Objects

```
PersistentVolume (PV):
  Actual storage resource — provisioned by admin or dynamically
  Like a physical hard drive in the cluster
  Has capacity, access mode, reclaim policy

PersistentVolumeClaim (PVC):
  Developer's request for storage
  "I need 5GB of ReadWriteOnce storage"
  K8s finds matching PV and binds them

StorageClass:
  Template for dynamic provisioning
  "Create AWS EBS volumes automatically when PVC requested"
  On AWS: gp2, gp3, io1, io2
```

---

### Storage Class — Dynamic Provisioning

```yaml
# storageclass.yaml — AWS EBS gp3
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
  encrypted: "true"
reclaimPolicy: Retain          # keep EBS volume when PVC deleted
allowVolumeExpansion: true     # allow resizing
volumeBindingMode: WaitForFirstConsumer  # create when pod scheduled
```

---

### PersistentVolumeClaim

```yaml
# pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-data-pvc
spec:
  storageClassName: fast-ssd     # use the StorageClass above
  accessModes:
  - ReadWriteOnce                # one node can read+write
  resources:
    requests:
      storage: 20Gi
```

```bash
# Access modes:
# ReadWriteOnce (RWO): one node read+write — EBS, most block storage
# ReadWriteMany (RWX): many nodes read+write — EFS, NFS
# ReadOnlyMany (ROX):  many nodes read only
```

---

### Using PVC in Deployment

```yaml
# mysql-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secrets
              key: ROOT_PASSWORD
        - name: MYSQL_DATABASE
          value: myappdb
        ports:
        - containerPort: 3306
        volumeMounts:
        - name: mysql-storage
          mountPath: /var/lib/mysql    # MySQL data directory

      volumes:
      - name: mysql-storage
        persistentVolumeClaim:
          claimName: mysql-data-pvc    # binds to our PVC
```

---

### StatefulSet — For Stateful Applications

```yaml
# For databases — use StatefulSet not Deployment:
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: mysql-headless
  replicas: 3
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql

  # StatefulSet creates PVC per pod automatically:
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      storageClassName: fast-ssd
      accessModes: [ReadWriteOnce]
      resources:
        requests:
          storage: 20Gi

  # Each pod gets: mysql-0, mysql-1, mysql-2
  # Each gets its own PVC: data-mysql-0, data-mysql-1, data-mysql-2
  # Stable network identity — pods always get same hostname
```

---

### StatefulSet vs Deployment

| Feature | Deployment | StatefulSet |
|---|---|---|
| Pod names | Random — myapp-7d9f-abc | Ordered — mysql-0, mysql-1 |
| Storage | Shared or no persistence | Each pod gets own PVC |
| Scaling | All pods identical | Ordered scale up/down |
| Use for | Stateless apps | Databases, queues, Kafka |

---

## Resource Management

---

### Resource Requests and Limits

```yaml
spec:
  containers:
  - name: myapp
    image: varun/myapp:1.0.0
    resources:
      requests:               # minimum guaranteed resources
        memory: "128Mi"       # 128 megabytes RAM
        cpu: "250m"           # 250 millicores = 0.25 CPU
      limits:                 # maximum allowed
        memory: "256Mi"       # if exceeds → OOMKilled
        cpu: "500m"           # if exceeds → throttled (not killed)
```

---

### Requests vs Limits — Critical Difference

```bash
# REQUESTS — what the scheduler uses:
# K8s scheduler looks at requests to decide which node to place pod
# "This node has 1 CPU free — pod needs 250m — fits!"
# Node is NOT overloaded if actual usage < requests

# LIMITS — what the runtime enforces:
# Memory limit: if pod exceeds → OOMKilled (killed immediately)
# CPU limit: if pod exceeds → throttled (slowed down, not killed)

# Best practice:
# Set requests = what app normally uses
# Set limits = what app should never exceed
# Ratio limits/requests = 2:1 is common

# Memory in K8s:
# Mi = Mebibytes (1Mi = 1,048,576 bytes)
# M  = Megabytes (1M  = 1,000,000 bytes)

# CPU in K8s:
# 1 CPU = 1000m (millicores)
# 500m = 0.5 CPU
# 100m = 0.1 CPU
```

---

### What Happens Without Resource Limits

```bash
# Without limits:
# One buggy pod consumes all node memory
# → Other pods OOMKilled
# → Node becomes unstable
# → Entire application down

# With limits:
# Buggy pod exceeds memory limit → OOMKilled (only that pod)
# Other pods unaffected
# K8s restarts the killed pod

# ALWAYS set resource limits in production
```

---

### LimitRange — Default Limits for Namespace

```yaml
# limitrange.yaml
# Sets default requests/limits for pods that don't specify
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: production
spec:
  limits:
  - type: Container
    default:                  # limit if not specified
      cpu: "500m"
      memory: "256Mi"
    defaultRequest:           # request if not specified
      cpu: "100m"
      memory: "128Mi"
    max:                      # maximum allowed in this namespace
      cpu: "2"
      memory: "2Gi"
    min:                      # minimum required
      cpu: "50m"
      memory: "64Mi"
```

---

### ResourceQuota — Limit Total Namespace Usage

```yaml
# resourcequota.yaml
# Limits total resources consumed in a namespace
apiVersion: v1
kind: ResourceQuota
metadata:
  name: production-quota
  namespace: production
spec:
  hard:
    # Compute:
    requests.cpu: "10"           # total CPU requested across all pods
    requests.memory: "20Gi"      # total memory requested
    limits.cpu: "20"             # total CPU limits
    limits.memory: "40Gi"        # total memory limits

    # Objects:
    pods: "50"                   # max number of pods
    services: "20"               # max services
    persistentvolumeclaims: "10" # max PVCs
    secrets: "30"                # max secrets
    configmaps: "30"             # max configmaps
```

---

### Checking Resource Usage

```bash
# Node resource usage:
kubectl top nodes

# Pod resource usage:
kubectl top pods
kubectl top pods -n production
kubectl top pods --sort-by=memory    # sort by memory usage
kubectl top pods --sort-by=cpu       # sort by CPU usage

# Describe node — see allocatable vs allocated:
kubectl describe node <node-name>
# Shows:
# Allocatable: CPU 2, Memory 4Gi
# Allocated:   CPU 1.5 (75%), Memory 3Gi (75%)

# Check if quota exceeded:
kubectl describe resourcequota -n production
```

---

## Complete Example — Full Application Config

```yaml
# Full stack: ConfigMap + Secret + Deployment + Service

---
# 1. ConfigMap for non-sensitive config
apiVersion: v1
kind: ConfigMap
metadata:
  name: myapp-config
  namespace: production
data:
  DB_HOST: mysql-service
  DB_PORT: "3306"
  DB_NAME: myappdb
  LOG_LEVEL: info
  APP_ENV: production

---
# 2. Secret for sensitive data
apiVersion: v1
kind: Secret
metadata:
  name: myapp-secrets
  namespace: production
type: Opaque
data:
  DB_PASSWORD: c3VwZXJzZWNyZXQxMjM=
  API_KEY: YWJjMTIzeHl6

---
# 3. Deployment using both
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: production
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
        image: varun/myapp:1.0.0
        ports:
        - containerPort: 5000
        envFrom:
        - configMapRef:
            name: myapp-config       # all ConfigMap keys as env vars
        - secretRef:
            name: myapp-secrets      # all Secret keys as env vars
        resources:
          requests:
            memory: "128Mi"
            cpu: "250m"
          limits:
            memory: "256Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health
            port: 5000
          initialDelaySeconds: 15
          periodSeconds: 20
        readinessProbe:
          httpGet:
            path: /health
            port: 5000
          initialDelaySeconds: 5
          periodSeconds: 10

---
# 4. Service
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
  namespace: production
spec:
  selector:
    app: myapp
  ports:
  - port: 80
    targetPort: 5000
  type: ClusterIP
```

---

### Full Summary — K8s Day 2

| Object | Purpose | Key point |
|---|---|---|
| ConfigMap | Non-sensitive config | env vars or volume mount |
| Secret | Sensitive data | base64 encoded — use KMS for real encryption |
| PVC | Request for storage | K8s finds/creates matching PV |
| StorageClass | Dynamic storage template | AWS EBS gp3 — auto-provision |
| StatefulSet | Stateful apps | Ordered pods, own PVC per pod |
| Resource requests | Scheduler uses this | Minimum guaranteed resources |
| Resource limits | Runtime enforces | Memory = kill, CPU = throttle |
| LimitRange | Namespace defaults | Applies to pods that don't specify |
| ResourceQuota | Namespace maximum | Total resource cap |

---

### Interview Questions — K8s Day 2

**Q1. What is the difference between a ConfigMap and a Secret?**
Both store configuration data as key-value pairs. ConfigMap is for non-sensitive config — DB hostnames, log levels, feature flags. Secret is for sensitive data — passwords, API keys, certificates — values are base64 encoded and masked in describe output. Secrets can be encrypted at rest. Access controlled separately via RBAC.

**Q2. What is the difference between resource requests and limits?**
Requests are what the scheduler uses to find a node — the minimum guaranteed resources. Limits are what the runtime enforces — maximum allowed. If memory exceeds limit the pod is OOMKilled. If CPU exceeds limit the pod is throttled but not killed. Always set both in production to prevent one bad pod from starving others.

**Q3. What is the difference between a Deployment and a StatefulSet?**
Deployment manages stateless pods — all identical, random names, no persistent identity. StatefulSet manages stateful applications like databases — pods get stable ordered names (mysql-0, mysql-1), stable network identities and each pod gets its own PersistentVolumeClaim. Use StatefulSet for databases, Kafka, Elasticsearch.

**Q4. What is a PersistentVolumeClaim?**
A request for storage by a developer — "I need 20GB of ReadWriteOnce storage." Kubernetes finds or dynamically provisions a matching PersistentVolume and binds them. The pod uses the PVC — not the PV directly. When pod is deleted data persists in the PV. Use StorageClass for automatic provisioning on AWS EBS.

**Q5. How do you update a ConfigMap without restarting pods?**
When ConfigMap is mounted as a volume — changes propagate automatically within about 60 seconds. No pod restart needed. When ConfigMap is used as environment variables — pod must be restarted because env vars are set at startup. For zero-downtime config updates use volume mounts instead of env vars.

**Q6. What is a LimitRange and why use it?**
LimitRange sets default resource requests and limits for containers in a namespace that don't specify their own. Without it pods can run with no limits — one pod could consume all node resources. LimitRange also sets maximum and minimum values for the namespace — prevents accidentally requesting too much or too little.

---

### Homework — Before K8s Day 3

```bash
# 1. Create ConfigMap and use as env vars:
kubectl create configmap myapp-config \
    --from-literal=APP_ENV=production \
    --from-literal=LOG_LEVEL=debug

kubectl run myapp --image=nginx:alpine \
    --env-from=configmap/myapp-config

kubectl exec -it myapp -- env | grep -E "APP_ENV|LOG_LEVEL"

# 2. Create Secret:
kubectl create secret generic myapp-secrets \
    --from-literal=DB_PASSWORD=secret123

# 3. Write a complete deployment.yaml that uses:
#    - ConfigMap (envFrom)
#    - Secret (env specific key)
#    - Resource requests and limits
#    - Liveness and readiness probes

# 4. Apply and verify:
kubectl apply -f deployment.yaml
kubectl describe pod <pod-name>
kubectl top pods

# 5. Create PVC and attach to pod:
kubectl apply -f pvc.yaml
kubectl get pvc   # should show Bound
```

---

### Your Progress

```
Linux  ████████████████████  ✅ COMPLETE
Git    ████████████████████  ✅ COMPLETE
Docker ████████████████████  ✅ COMPLETE
AWS    ████████████████████  ✅ COMPLETE
CI/CD  ████████████████████  ✅ COMPLETE
K8s    ████░░░░░░░░░░░░░░░░  Day 2 of 6
Final  ░░░░░░░░░░░░░░░░░░░░  Interview prep
```

K8s Day 3 we learn **Ingress, Services deep dive, Horizontal Pod Autoscaler and Helm** — how real production traffic flows into a K8s cluster, automatic scaling based on metrics and packaging applications for easy deployment. 💪

Say **"K8s Day 3"** whenever you are ready!
