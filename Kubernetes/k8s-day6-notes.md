## Kubernetes Day 6 — Production Patterns, Upgrades, Disaster Recovery & Final Project

---

### Why This is Critical

You can deploy, configure, secure, scale and observe applications. Three things remain before you can call yourself production-ready:

```
Problem 1 — Helm in the real world:
You know "helm install" — but production teams manage dozens of
charts, across environments, with dependencies and CI/CD pipelines
Need real Helm patterns, not just toy examples

Problem 2 — The cluster itself needs maintenance:
Kubernetes versions get deprecated, nodes need patching,
control plane components need upgrading — with zero downtime

Problem 3 — Disaster recovery:
If etcd is lost, the ENTIRE cluster state is gone — every
deployment, secret, config, everything
You need a tested backup and restore process, not hope
```

This is the final day. We close with a full end-to-end project tying together everything from Linux through Kubernetes.

---

### What You Will Learn Today

- Helm in production — dependencies, hooks, library charts, CI/CD
- Cluster upgrade strategy — control plane and nodes
- Node draining and maintenance
- etcd backup and restore
- Disaster recovery planning
- Multi-cluster and GitOps overview
- Final end-to-end project
- Complete K8s interview preparation summary

---

## Helm in Production

---

### Chart Dependencies

Real applications often depend on other charts — for example, your app might need Redis or PostgreSQL alongside it.

```yaml
# Chart.yaml
apiVersion: v2
name: myapp
version: 1.0.0
dependencies:
- name: redis
  version: "18.x.x"
  repository: "https://charts.bitnami.com/bitnami"
  condition: redis.enabled
- name: postgresql
  version: "13.x.x"
  repository: "https://charts.bitnami.com/bitnami"
  condition: postgresql.enabled
```

```bash
# Download dependency charts into ./charts/:
helm dependency update ./mychart
helm dependency build ./mychart

# Enable/disable dependencies via values.yaml:
redis:
  enabled: true
postgresql:
  enabled: false
```

---

### Helm Hooks — Run Jobs at Specific Lifecycle Points

```yaml
# templates/db-migration-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ .Release.Name }}-db-migrate
  annotations:
    "helm.sh/hook": pre-install,pre-upgrade
    "helm.sh/hook-weight": "0"
    "helm.sh/hook-delete-policy": hook-succeeded
spec:
  template:
    spec:
      containers:
      - name: migrate
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        command: ["python", "manage.py", "migrate"]
      restartPolicy: Never
```

```bash
# Common hook points:
# pre-install, post-install
# pre-upgrade, post-upgrade
# pre-delete, post-delete
# pre-rollback, post-rollback

# hook-weight controls execution order when multiple hooks exist
# (lower numbers run first)
```

---

### Library Charts — Shared Templates Across Teams

```bash
# A library chart contains reusable template snippets but
# creates no Kubernetes resources of its own — other charts
# import from it to avoid duplicating boilerplate

# Chart.yaml of the library chart:
apiVersion: v2
name: common
type: library          # this is the key difference
version: 1.0.0
```

```yaml
# Other charts depend on it:
dependencies:
- name: common
  version: "1.0.0"
  repository: "file://../common"
```

```bash
# Why teams use this: standardize labels, resource naming,
# security context defaults etc. across 20+ microservice charts
# without copy-pasting the same YAML into every chart
```

---

### Helm in CI/CD Pipelines

```yaml
# Example GitHub Actions step (conceptual)
- name: Deploy with Helm
  run: |
    helm upgrade --install myapp ./charts/myapp \
      --namespace production \
      --set image.tag=${{ github.sha }} \
      --values ./charts/myapp/values-prod.yaml \
      --wait --timeout 5m \
      --atomic
```

```bash
# Key flags for CI/CD safety:
# --wait     → blocks until all resources report Ready
# --timeout  → fails the pipeline if not ready within this time
# --atomic   → automatically rolls back on failure
# --install  → "upgrade --install" creates if it doesn't exist yet,
#               upgrades if it does — idempotent, safe to rerun
```

---

### Helm Chart Testing and Linting

```bash
# Lint a chart for syntax/best-practice issues:
helm lint ./mychart

# Run chart tests (defined as Jobs with a special annotation):
helm test myapp

# helm-unittest plugin for actual unit tests on templates:
helm plugin install https://github.com/helm-unittest/helm-unittest
helm unittest ./mychart
```

---

## GitOps Overview

---

### What is GitOps?

GitOps means your cluster's desired state lives entirely in a Git repository, and an automated controller continuously syncs the live cluster to match what's in Git — rather than engineers running `kubectl apply` or `helm upgrade` manually.

```
Traditional CI/CD:
Pipeline runs → kubectl apply / helm upgrade → cluster changes
(push-based — pipeline pushes changes INTO the cluster)

GitOps:
Git repo changes → controller INSIDE cluster notices the diff
                  → pulls and applies it automatically
(pull-based — controller running in-cluster pulls FROM Git)
```

```bash
# Most common GitOps tools:
# Argo CD  → most widely adopted, has a UI, supports Helm/Kustomize
# Flux     → CNCF project, lighter weight, very Git-native

# Why teams adopt GitOps:
# - Git becomes the single source of truth and audit log
# - Rollback = git revert
# - No one needs direct kubectl/helm access to prod — only
#   permission to merge to the Git repo
```

```bash
# Conceptual Argo CD Application pointing at a Helm chart in Git:
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
  namespace: argocd
spec:
  source:
    repoURL: https://github.com/myorg/myapp-charts.git
    path: charts/myapp
    targetRevision: main
    helm:
      valueFiles:
      - values-prod.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

---

## Cluster Upgrades

---

### Why Upgrades Matter

```bash
# Kubernetes releases a new minor version roughly every 4 months
# Each version is generally supported (with patches) for about
# 14 months before it stops receiving security fixes

# Staying too far behind means:
# - No security patches
# - Managed services (EKS, GKE, AKS) may force-upgrade you
#   on their own schedule with less control
# - Bigger version jumps later = riskier upgrades
```

---

### Upgrade Order — Control Plane First

```bash
# Correct order, always:
# 1. Upgrade Control Plane (API server, etcd, scheduler, controller manager)
# 2. Upgrade kubectl on your local machine to match
# 3. Upgrade Worker Nodes (kubelet, kube-proxy) — one at a time

# Why this order: the API server must always be the same version
# or NEWER than kubelet on any node — never older
# A newer kubelet talking to an older API server can break things
```

---

### On Managed Kubernetes (EKS/GKE/AKS) — Control Plane

```bash
# EKS example — control plane upgrade is mostly automatic via console/CLI:
aws eks update-cluster-version \
    --name my-cluster \
    --kubernetes-version 1.31

# This only upgrades the control plane — node groups must be
# upgraded SEPARATELY afterward
```

---

### Upgrading Worker Nodes — Rolling, Zero Downtime

```bash
# Step 1 — Cordon the node (stop new pods from being scheduled there):
kubectl cordon node-1

# Step 2 — Drain the node (safely evict existing pods elsewhere):
kubectl drain node-1 --ignore-daemonsets --delete-emptydir-data

# Step 3 — Upgrade kubelet/kube-proxy on that node, or replace
# the node entirely (common pattern on managed K8s — just
# terminate the old node and let a new, upgraded one join)

# Step 4 — Uncordon once healthy, allowing scheduling again:
kubectl uncordon node-1

# Repeat one node at a time — never drain multiple nodes at once
# unless you have enough spare capacity to absorb all those pods
```

```bash
# What cordon vs drain actually do:
# cordon → marks node unschedulable, EXISTING pods keep running
# drain  → cordons AND evicts existing pods (respecting PodDisruptionBudgets)
```

---

### PodDisruptionBudget — Protecting Availability During Drains

```yaml
# pdb.yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: myapp-pdb
spec:
  minAvailable: 2          # always keep at least 2 pods running
  selector:
    matchLabels:
      app: myapp
```

```bash
# Without a PDB, "kubectl drain" could evict ALL replicas of myapp
# on one node simultaneously if they're all scheduled there,
# causing an outage during routine maintenance

# With a PDB, drain will respect minAvailable and wait/retry
# rather than violate it

# Alternative: maxUnavailable instead of minAvailable
spec:
  maxUnavailable: 1
```

---

### Checking Cluster and Node Versions

```bash
kubectl version --short
kubectl get nodes -o wide        # shows kubelet version per node

# Check what API versions/resources will be deprecated in the
# NEXT release before upgrading:
kubectl convert -f old-manifest.yaml --output-version apps/v1
```

---

## Disaster Recovery — etcd Backup & Restore

---

### Why etcd is the Single Most Critical Component

```bash
# etcd stores EVERYTHING:
# - Every Deployment, Pod spec, Service, ConfigMap, Secret
# - All RBAC rules
# - The entire desired and observed state of the cluster

# If etcd is lost with no backup:
# The cluster doesn't just go down — it forgets EVERYTHING
# you ever deployed. Pods/nodes might keep running briefly,
# but nothing can be managed, scaled, or recovered properly
```

---

### Taking an etcd Backup

```bash
# Run on a control plane node, using etcdctl:
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-snapshot-$(date +%F).db \
    --endpoints=https://127.0.0.1:2379 \
    --cacert=/etc/kubernetes/pki/etcd/ca.crt \
    --cert=/etc/kubernetes/pki/etcd/server.crt \
    --key=/etc/kubernetes/pki/etcd/server.key

# Verify the snapshot:
ETCDCTL_API=3 etcdctl snapshot status /backup/etcd-snapshot-2026-06-18.db \
    --write-out=table
```

```bash
# Automate this with a CronJob (running on a control plane node
# or via a privileged pod with access to etcd certs), and always
# copy snapshots OFF the cluster — to S3, GCS, or similar —
# a backup stored only on the same disk as etcd protects against
# nothing
```

---

### Restoring etcd from a Snapshot

```bash
# Restore creates a NEW etcd data directory from the snapshot:
ETCDCTL_API=3 etcdctl snapshot restore /backup/etcd-snapshot-2026-06-18.db \
    --data-dir=/var/lib/etcd-restored

# Then update the etcd static pod manifest (usually
# /etc/kubernetes/manifests/etcd.yaml) to point at the new
# data directory, and restart etcd

# On a multi-node control plane, this must be done carefully
# across all etcd members — consult your specific K8s
# distribution's documented restore procedure before attempting
# this in a real incident
```

```bash
# Practice this BEFORE you need it:
# A backup you've never test-restored is not a real backup
# Run a full restore drill in a non-production cluster at
# least once so the process isn't unfamiliar during an actual outage
```

---

### Application-Level Backup — Don't Forget This

```bash
# etcd backup protects cluster CONFIGURATION, not application DATA
# Your MySQL/PostgreSQL data living in a PersistentVolume needs
# its OWN backup strategy — etcd backup does not cover this

# Common approaches:
# - Velero — backs up both cluster resources AND PV data (via
#   snapshots) to object storage like S3
# - Database-native backup tools (mysqldump, pg_dump, WAL archiving)
# - Cloud-native EBS/disk snapshots scheduled independently
```

```bash
# Velero quick install reference:
helm repo add vmware-tanzu https://vmware-tanzu.github.io/helm-charts
helm install velero vmware-tanzu/velero -n velero --create-namespace \
    --set configuration.provider=aws \
    --set configuration.backupStorageLocation.bucket=my-backup-bucket

# Backup an entire namespace:
velero backup create production-backup --include-namespaces production

# Restore it:
velero restore create --from-backup production-backup
```

---

## Final End-to-End Project

---

### Project Brief

Deploy a complete production-style application stack that uses everything learned across all six days.

```
Stack:
- Flask/Node backend API (your own image, pushed to a registry)
- MySQL database with persistent storage
- Redis cache (via Helm dependency)
- Frontend served by nginx

Requirements covering each day:
Day 1 → Deployment, Service, correct replica count
Day 2 → ConfigMap for app config, Secret for DB credentials,
         PVC for MySQL data, resource requests/limits set
Day 3 → Ingress with host-based routing, HPA on the backend,
         entire stack packaged as a Helm chart
Day 4 → Dedicated ServiceAccount with least-privilege RBAC,
         NetworkPolicy so only backend can reach MySQL,
         non-root securityContext on all containers
Day 5 → ServiceMonitor exposing backend metrics, a Grafana
         dashboard, an alert rule for high error rate, liveness/
         readiness/startup probes correctly separated
Day 6 → values-dev.yaml and values-prod.yaml, a PodDisruptionBudget,
         and a tested etcd/Velero backup of the namespace
```

---

### Suggested Build Order

```bash
# 1. Containerize and push your app images (Docker Day 1-2 skills)
docker build -t yourname/myapp-backend:1.0.0 .
docker push yourname/myapp-backend:1.0.0

# 2. Scaffold the Helm chart:
helm create myapp-stack
# Edit values.yaml, add MySQL/Redis as dependencies

# 3. Write templates: deployment, service, configmap, secret,
#    pvc, ingress, hpa, serviceaccount, role, rolebinding,
#    networkpolicy, servicemonitor, pdb

# 4. Validate before installing:
helm lint ./myapp-stack
helm template myapp-stack ./myapp-stack | less

# 5. Install to a dev namespace first:
helm install myapp ./myapp-stack -f values-dev.yaml -n dev --create-namespace

# 6. Verify everything end-to-end:
kubectl get all -n dev
kubectl get ingress -n dev
kubectl get hpa -n dev
kubectl auth can-i list secrets --as system:serviceaccount:dev:myapp-sa
kubectl exec -it <frontend-pod> -n dev -- curl backend-service:5000/health

# 7. Promote to "prod" namespace with different values:
helm install myapp ./myapp-stack -f values-prod.yaml -n production --create-namespace
```

---

### Self-Check Before Calling It Done

```bash
# Can you answer "yes" to all of these about your project?

# - If a pod dies, does it come back automatically? (Deployment)
# - If MySQL pod restarts, is data still there? (PVC)
# - Can you reach the app from outside via a domain name? (Ingress)
# - Does the backend scale under load and shrink after? (HPA)
# - Can the frontend reach the backend, but NOT reach MySQL directly? (NetworkPolicy)
# - Does the app run as non-root with a read-only filesystem? (Security Context)
# - Can you see live metrics and an error-rate dashboard? (Prometheus/Grafana)
# - If you delete a node, does a PDB prevent a full outage? (PodDisruptionBudget)
# - Could you restore this whole namespace from a backup right now? (Velero)
# - Is the whole thing one "helm install" away from being rebuilt
#   from scratch? (Helm chart completeness)
```

---

### Full Summary — K8s Day 6

| Concept | Key point |
|---|---|
| Chart dependencies | Sub-charts (Redis, PostgreSQL) declared in Chart.yaml |
| Helm hooks | Run Jobs at lifecycle points (pre-install, post-upgrade) |
| Library charts | Shared template snippets across many charts, no resources of own |
| GitOps | Git is source of truth; controller pulls and syncs cluster to match |
| Argo CD / Flux | Most common GitOps controllers |
| Upgrade order | Control plane first, then kubectl, then worker nodes |
| cordon | Marks node unschedulable, doesn't evict existing pods |
| drain | Cordons AND evicts pods safely, respecting PDBs |
| PodDisruptionBudget | Guarantees minimum availability during voluntary disruptions |
| etcd | Stores entire cluster state — losing it loses everything |
| etcdctl snapshot save/restore | Backup and recovery commands for etcd |
| Velero | Backs up cluster resources AND PersistentVolume data |
| Application data backup | Separate from etcd backup — needs its own strategy |

---

### Interview Questions — K8s Day 6

**Q1. What is GitOps and how does it differ from a traditional CI/CD deployment?**
GitOps treats a Git repository as the single source of truth for the cluster's desired state. An in-cluster controller, such as Argo CD or Flux, continuously compares the live cluster state against what's declared in Git and automatically syncs any differences — a pull-based model. Traditional CI/CD is push-based: a pipeline runs commands like `kubectl apply` or `helm upgrade` to push changes into the cluster from outside.

**Q2. In what order should you upgrade a Kubernetes cluster, and why?**
The control plane (API server, etcd, scheduler, controller manager) is upgraded first, followed by kubectl on local machines, and worker nodes (kubelet, kube-proxy) are upgraded last, one at a time. This order matters because the API server must always be running a version equal to or newer than every kubelet in the cluster; a kubelet newer than the API server can cause compatibility issues.

**Q3. What is the difference between cordoning and draining a node?**
Cordoning a node marks it as unschedulable so no new pods will be placed there, but any pods already running on it keep running undisturbed. Draining a node does both: it cordons the node and also safely evicts the existing pods so they get rescheduled elsewhere, while respecting any PodDisruptionBudgets defined for those pods.

**Q4. What is a PodDisruptionBudget and why is it important during cluster maintenance?**
A PodDisruptionBudget specifies the minimum number of pods (or maximum number that can be unavailable) for an application during voluntary disruptions, such as node drains for maintenance. Without one, draining a node could evict all replicas of an application that happen to be scheduled there simultaneously, causing an outage even though the application as a whole still has healthy capacity elsewhere in the cluster.

**Q5. Why is backing up etcd considered critical, and what does an etcd backup NOT cover?**
etcd stores the entire state of the Kubernetes cluster, including every Deployment, Service, ConfigMap, Secret and RBAC rule. Losing etcd without a backup means losing the ability to manage or reconstruct the cluster's configuration. However, an etcd backup does not include application data stored in PersistentVolumes, such as a database's actual records — that requires a separate backup strategy, for example using a tool like Velero or database-native backup tools.

**Q6. What is the purpose of a Helm hook, and give an example use case?**
A Helm hook lets you run a Kubernetes Job at a specific point in a chart's lifecycle, such as pre-install, post-install, pre-upgrade, or pre-delete, rather than as a normal long-running resource. A common use case is running a database migration Job before a new application version starts receiving traffic, ensuring the schema is up to date before the upgraded pods come online.

**Q7. What is the purpose of a library chart in Helm?**
A library chart provides reusable template snippets, such as standardized labels, naming conventions, or default security contexts, that other charts can import as a dependency. Unlike a normal chart, it doesn't create any Kubernetes resources on its own — its only purpose is to be referenced by other charts, which helps large organizations keep many microservice charts consistent without duplicating the same boilerplate YAML across each one.

---

### Final Interview Prep — Quick Recall Table (All 6 Days)

| Day | Core Topics | One-Line Recall |
|---|---|---|
| 1 | Pod, Deployment, Service, Namespace | Deployment manages Pods; Service gives them a stable address |
| 2 | ConfigMap, Secret, PVC, Resources | Config separate from image; requests=scheduler, limits=runtime |
| 3 | Ingress, HPA, Helm | Ingress routes HTTP by host/path; HPA needs resource requests to work |
| 4 | RBAC, ServiceAccount, NetworkPolicy, SecurityContext | Least privilege everywhere — default-deny network, non-root containers |
| 5 | Prometheus, Grafana, Logging, Probes | Liveness restarts, readiness removes from traffic — never mix the two up |
| 6 | Helm production, upgrades, etcd DR | Control plane upgrades first; etcd backup ≠ application data backup |

---

### Your Progress

```
Linux  ████████████████████  ✅ COMPLETE
Git    ████████████████████  ✅ COMPLETE
Docker ████████████████████  ✅ COMPLETE
AWS    ████████████████████  ✅ COMPLETE
CI/CD  ████████████████████  ✅ COMPLETE
K8s    ████████████████████  ✅ COMPLETE (Day 6 of 6)
Final  ██████░░░░░░░░░░░░░░  Interview prep — in progress
```

You've now covered the full path from Linux fundamentals through a production-grade Kubernetes deployment. The final stretch is consolidated interview prep — drilling the Q&A across all topics, practicing whiteboard-style architecture explanations, and mock scenario questions (e.g. "a pod is CrashLooping, walk me through your debugging steps").

Say **"Interview Prep"** whenever you are ready to begin that final stage. 💪
