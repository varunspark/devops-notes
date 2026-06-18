## Kubernetes Day 4 — RBAC, Service Accounts, Network Policies & Security Context

---

### Why This is Critical

Your cluster is running real applications now. Three security gaps remain:

```
Problem 1 — Who can do what?
Any developer with kubectl access could delete production deployments,
read all Secrets, or create resources in any namespace
Need fine-grained permission control

Problem 2 — Pod-to-pod traffic:
By default, EVERY pod can talk to EVERY other pod in the cluster
A compromised frontend pod could directly hit your database pod
Need network-level restrictions

Problem 3 — Container privileges:
By default, containers can run as root, write to the host filesystem,
escalate privileges
A compromised container could break out and harm the host
Need to lock down what a container is allowed to do
```

RBAC solves who-can-do-what. Network Policies solve pod-to-pod traffic control. Security Context solves container-level privilege restriction.

---

### What You Will Learn Today

- RBAC — Roles, RoleBindings, ClusterRoles, ClusterRoleBindings
- Service Accounts — identity for pods
- How pods authenticate to the API server
- Network Policies — controlling pod-to-pod traffic
- Security Context — pod and container level
- Pod Security Standards
- Real DevOps security patterns

---

## RBAC — Role-Based Access Control

---

### What is RBAC?

RBAC controls who can perform which actions on which resources in a Kubernetes cluster. Every request to the API server is checked against RBAC rules before it's allowed.

```
RBAC answers:
WHO    → User, Group, or ServiceAccount
CAN DO WHAT  → get, list, create, update, delete, watch
ON WHICH RESOURCE  → pods, deployments, secrets, configmaps
IN WHICH SCOPE  → specific namespace, or cluster-wide
```

```
Four RBAC objects:
Role               → permissions within ONE namespace
RoleBinding        → grants a Role to a user/group/ServiceAccount
ClusterRole        → permissions across the WHOLE cluster (or reusable)
ClusterRoleBinding → grants a ClusterRole cluster-wide
```

---

### Role — Namespace-Scoped Permissions

```yaml
# role.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: production
rules:
- apiGroups: [""]                     # "" = core API group
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["pods/log"]
  verbs: ["get"]
```

```bash
# apiGroups reference:
# ""                → core resources: pods, services, configmaps, secrets
# "apps"            → deployments, statefulsets, replicasets
# "networking.k8s.io" → ingress, networkpolicies
# "rbac.authorization.k8s.io" → roles, rolebindings

# Common verbs:
# get, list, watch    → read operations
# create, update, patch → write operations
# delete, deletecollection → remove operations
```

---

### RoleBinding — Granting the Role

```yaml
# rolebinding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods-binding
  namespace: production
subjects:
- kind: User
  name: alice
  apiGroup: rbac.authorization.k8s.io
- kind: ServiceAccount
  name: myapp-sa
  namespace: production
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

```bash
kubectl apply -f role.yaml
kubectl apply -f rolebinding.yaml

# Verify what a user/serviceaccount can do:
kubectl auth can-i get pods --namespace production --as alice
kubectl auth can-i delete deployments --namespace production --as alice
kubectl auth can-i list secrets --as system:serviceaccount:production:myapp-sa
```

---

### ClusterRole — Cluster-Wide Permissions

```yaml
# clusterrole.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-reader
rules:
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get", "list", "watch"]

---
# Reusable ClusterRole — applied per-namespace via RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: deployment-manager
rules:
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "create", "update", "delete"]
```

```bash
# ClusterRole used two ways:
# 1. ClusterRoleBinding → permission applies cluster-wide
# 2. RoleBinding (in a specific namespace) → permission applies
#    only in that namespace, even though it's a "ClusterRole"
#    (this is how you reuse one set of rules across many namespaces)
```

---

### ClusterRoleBinding — Cluster-Wide Grant

```yaml
# clusterrolebinding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: node-reader-binding
subjects:
- kind: Group
  name: sre-team
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: node-reader
  apiGroup: rbac.authorization.k8s.io
```

---

### Built-In ClusterRoles

```bash
# K8s ships with default ClusterRoles — don't reinvent these:
# cluster-admin  → full access to everything, every namespace
# admin          → full access within a namespace (not cluster resources)
# edit           → read/write most objects in a namespace, not RBAC itself
# view           → read-only access to most objects in a namespace

kubectl get clusterroles
kubectl describe clusterrole admin
```

---

### Principle of Least Privilege — Real Example

```yaml
# CI/CD pipeline ServiceAccount — only what it needs, nothing more
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: ci-deployer
  namespace: production
rules:
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "update", "patch"]     # can deploy, NOT delete
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]                # can check rollout status
# Notice: NO access to secrets, NO delete verb, NO other namespaces
```

---

## Service Accounts

---

### What is a Service Account?

A Service Account is an identity for processes running inside pods — not for humans. Every pod authenticates to the API server using a Service Account, even if you never explicitly assign one.

```bash
# Every namespace has a "default" ServiceAccount automatically
kubectl get serviceaccounts
kubectl get sa -n production

# Pods use "default" SA unless you specify otherwise
# Best practice: create a dedicated SA per application, with
# only the RBAC permissions that application actually needs
```

---

### Creating and Using a Service Account

```bash
kubectl create serviceaccount myapp-sa -n production
```

```yaml
# deployment.yaml — attach the ServiceAccount to pods
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: production
spec:
  template:
    spec:
      serviceAccountName: myapp-sa     # pods use this identity
      containers:
      - name: myapp
        image: varun/myapp:1.0.0
```

---

### How Pods Authenticate to the API Server

```bash
# Kubernetes automatically mounts a token for the pod's ServiceAccount:
/var/run/secrets/kubernetes.io/serviceaccount/token
/var/run/secrets/kubernetes.io/serviceaccount/ca.crt
/var/run/secrets/kubernetes.io/serviceaccount/namespace

# Any app code inside the pod (e.g. a K8s client library) reads this
# token automatically to make authenticated calls back to the API server
# This is how apps like Helm controllers, operators, or your own app
# can call "kubectl"-like actions from inside the cluster
```

```yaml
# Disable auto-mounting if the pod doesn't need API access at all
# (reduces attack surface):
spec:
  serviceAccountName: myapp-sa
  automountServiceAccountToken: false
```

---

### Real Scenario — App That Reads K8s Secrets at Runtime

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: secret-reader-sa
  namespace: production

---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: secret-reader-role
  namespace: production
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "list"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: secret-reader-binding
  namespace: production
subjects:
- kind: ServiceAccount
  name: secret-reader-sa
  namespace: production
roleRef:
  kind: Role
  name: secret-reader-role
  apiGroup: rbac.authorization.k8s.io
```

---

## Network Policies

---

### The Default Behavior — No Restrictions

```bash
# By default in Kubernetes:
# EVERY pod can send traffic to EVERY other pod, any namespace
# No firewall rules exist between pods unless you create them

# This means a compromised low-privilege pod (e.g. a public-facing
# frontend) can freely connect to your database pod, internal admin
# APIs, or anything else in the cluster
```

A NetworkPolicy is a firewall rule for pod traffic, enforced by the CNI plugin (e.g. Calico, Cilium — not all CNIs support NetworkPolicy).

---

### Default Deny — Starting Point

```yaml
# deny-all.yaml — blocks ALL traffic in/out for pods in this namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}            # applies to ALL pods in namespace
  policyTypes:
  - Ingress
  - Egress
```

```bash
# Best practice: apply default-deny first, then explicitly allow
# only the traffic your application actually needs
```

---

### Allow Specific Traffic — Frontend to Backend

```yaml
# allow-frontend-to-backend.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend              # this policy applies to backend pods
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend          # only frontend pods can reach backend
    ports:
    - protocol: TCP
      port: 5000
```

---

### Allow Traffic from a Specific Namespace

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-monitoring-namespace
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: myapp
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: monitoring
    ports:
    - protocol: TCP
      port: 9090
```

---

### Restrict Egress — Database Only Talks to Backend

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: mysql-allow-backend-only
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: mysql
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: backend
    ports:
    - protocol: TCP
      port: 3306
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: backend
```

```bash
# Allow DNS lookups — almost always needed alongside Egress rules:
egress:
- to:
  - namespaceSelector: {}
  ports:
  - protocol: UDP
    port: 53
```

---

### Useful NetworkPolicy Commands

```bash
kubectl get networkpolicy -n production
kubectl describe networkpolicy default-deny-all -n production

# Test connectivity between pods:
kubectl exec -it frontend-pod -- curl backend-service:5000
kubectl exec -it other-pod -- curl backend-service:5000   # should fail if denied
```

---

## Security Context

---

### What is a Security Context?

Security Context defines privilege and access control settings for a Pod or Container — what user it runs as, what it can or can't do at the OS level.

```
Pod-level securityContext  → applies to all containers in the pod
Container-level securityContext → overrides pod-level, per container
```

---

### Pod-Level Security Context

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  securityContext:
    runAsUser: 1000          # run as non-root UID
    runAsGroup: 3000
    fsGroup: 2000             # group ownership for mounted volumes
    runAsNonRoot: true         # refuse to start if image wants root
  containers:
  - name: myapp
    image: varun/myapp:1.0.0
```

---

### Container-Level Security Context

```yaml
spec:
  containers:
  - name: myapp
    image: varun/myapp:1.0.0
    securityContext:
      runAsNonRoot: true
      readOnlyRootFilesystem: true        # container can't write to its own fs
      allowPrivilegeEscalation: false      # can't gain more privileges than parent
      privileged: false                    # never grant full host access
      capabilities:
        drop:
        - ALL                              # drop all Linux capabilities
        add:
        - NET_BIND_SERVICE                 # add back only what's needed
```

```bash
# Linux capabilities reference (common ones):
# NET_BIND_SERVICE → bind to ports below 1024
# CHOWN            → change file ownership
# SYS_ADMIN        → broad system admin powers (avoid granting this)
# Drop ALL, then add back only the specific ones the app needs
```

---

### readOnlyRootFilesystem — Real Pattern

```yaml
# If app needs to write somewhere (logs, temp files), mount a volume
# specifically for that, keep everything else read-only:
spec:
  containers:
  - name: myapp
    image: varun/myapp:1.0.0
    securityContext:
      readOnlyRootFilesystem: true
    volumeMounts:
    - name: tmp-volume
      mountPath: /tmp
  volumes:
  - name: tmp-volume
    emptyDir: {}
```

---

### Why privileged: false Matters

```bash
# privileged: true gives a container nearly all capabilities of
# the host machine — full device access, ability to load kernel
# modules, bypass most isolation
# This should almost NEVER be used in production
# Common legitimate exception: certain CNI/storage plugins that
# genuinely need host-level access — everything else, avoid it
```

---

### Pod Security Standards (replaces old PodSecurityPolicy)

```bash
# Three built-in levels, enforced via namespace labels:
# privileged → no restrictions (avoid in production)
# baseline   → blocks known privilege escalations, minimal restriction
# restricted → heavily locked down — non-root, no privilege escalation,
#              dropped capabilities, no hostPath volumes

kubectl label namespace production \
    pod-security.kubernetes.io/enforce=restricted \
    pod-security.kubernetes.io/audit=restricted \
    pod-security.kubernetes.io/warn=restricted
```

```bash
# enforce → blocks non-compliant pods from being created
# audit   → logs violations, doesn't block
# warn    → shows warning to user, doesn't block
```

---

### Full Summary — K8s Day 4

| Concept | Key point |
|---|---|
| Role | Permissions scoped to one namespace |
| RoleBinding | Grants a Role to a user/group/ServiceAccount |
| ClusterRole | Permissions across the cluster, or reusable across namespaces |
| ClusterRoleBinding | Grants a ClusterRole cluster-wide |
| ServiceAccount | Identity for pods/processes, not humans |
| Default SA | Every namespace has one; create dedicated SAs per app |
| NetworkPolicy | Firewall rules between pods — enforced by CNI plugin |
| Default-deny | Best practice starting point — block all, allow specifically |
| Security Context | Controls user, privileges, filesystem access for pods/containers |
| readOnlyRootFilesystem | Prevents writes to container filesystem |
| privileged: false | Never give full host access unless absolutely required |
| Pod Security Standards | privileged / baseline / restricted enforcement levels |

---

### Interview Questions — K8s Day 4

**Q1. What is the difference between a Role and a ClusterRole?**
A Role defines permissions scoped to a single namespace and can only be bound within that namespace. A ClusterRole defines permissions that can either apply cluster-wide (via a ClusterRoleBinding) or be reused within a specific namespace (via a RoleBinding referencing the ClusterRole). ClusterRoles are also required for cluster-scoped resources like nodes, which don't belong to any namespace.

**Q2. What is a Service Account and why does every pod have one?**
A Service Account is an identity used by processes running in pods to authenticate to the Kubernetes API server, as opposed to User accounts which represent humans. Every pod is automatically assigned the "default" Service Account in its namespace unless one is explicitly specified, and Kubernetes mounts an authentication token into the pod so it can make API calls with that identity.

**Q3. What is the default network behavior between pods, and how do NetworkPolicies change that?**
By default, all pods in a cluster can communicate with all other pods with no restrictions, regardless of namespace. A NetworkPolicy acts as a firewall, restricting which pods can send or receive traffic to/from a given set of pods, identified via podSelector and optionally namespaceSelector. Best practice is to apply a default-deny-all policy first, then add specific allow rules.

**Q4. What is the difference between runAsNonRoot and privileged in a Security Context?**
runAsNonRoot ensures the container's process runs as a non-root user, refusing to start the container if the image is configured to run as root. privileged is a much broader setting that, if set to true, grants the container nearly full access to the host machine's devices and kernel capabilities, effectively disabling most container isolation. privileged should be false in virtually all production workloads.

**Q5. Why is readOnlyRootFilesystem considered a security best practice?**
Setting readOnlyRootFilesystem to true prevents a container from writing to its own filesystem at runtime, which limits the damage an attacker can do if the application is compromised, since they can't modify application binaries or drop malicious files into the container's filesystem. Applications that need to write temporary data use a mounted volume, such as an emptyDir, for that specific path instead.

**Q6. What are Linux capabilities in the context of container security, and why drop them?**
Linux capabilities are granular permissions (like NET_BIND_SERVICE or SYS_ADMIN) that break up what would otherwise be all-or-nothing root privileges. Containers by default run with a set of capabilities that are often more than they need. Best practice is to drop all capabilities and explicitly add back only the ones the application genuinely requires, minimizing what a compromised container could do.

**Q7. What are Pod Security Standards and what are the three levels?**
Pod Security Standards are built-in Kubernetes policies, applied via namespace labels, that restrict what kinds of pods can be created. The three levels are privileged (no restrictions), baseline (blocks known privilege escalation paths with minimal restriction), and restricted (enforces strong hardening like non-root, dropped capabilities, and no hostPath volumes). They can be set to enforce, audit, or warn modes.

---

### Homework — Before K8s Day 5

```bash
# 1. Create a Role and RoleBinding limiting a ServiceAccount to
#    read-only access on pods in one namespace, then verify:
kubectl auth can-i list pods --as system:serviceaccount:production:myapp-sa
kubectl auth can-i delete pods --as system:serviceaccount:production:myapp-sa

# 2. Create a dedicated ServiceAccount for your app deployment
#    and attach it via serviceAccountName

# 3. Apply a default-deny-all NetworkPolicy to a namespace, then
#    add a specific allow rule so your frontend can still reach
#    your backend — verify with kubectl exec + curl

# 4. Update your deployment.yaml to include:
#    - runAsNonRoot: true
#    - readOnlyRootFilesystem: true
#    - capabilities: drop ALL
#    Redeploy and confirm the pod starts successfully

# 5. Label a namespace with Pod Security Standards "restricted"
#    in "warn" mode first, see what warnings appear, then fix
#    your manifests before switching to "enforce"
```

---

### Your Progress

```
Linux  ████████████████████  ✅ COMPLETE
Git    ████████████████████  ✅ COMPLETE
Docker ████████████████████  ✅ COMPLETE
AWS    ████████████████████  ✅ COMPLETE
CI/CD  ████████████████████  ✅ COMPLETE
K8s    ████████░░░░░░░░░░░░  Day 4 of 6
Final  ░░░░░░░░░░░░░░░░░░░░  Interview prep
```

K8s Day 5 we learn **Monitoring with Prometheus & Grafana, Logging, and Health Checks in depth** — how to actually observe what's happening inside your cluster in production. 💪

Say **"K8s Day 5"** whenever you are ready!
