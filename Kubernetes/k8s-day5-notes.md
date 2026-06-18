## Kubernetes Day 5 — Monitoring, Logging & Observability

---

### Why This is Critical

Your cluster is running, secured, scaling automatically. But one question remains unanswered:

```
3am — something is wrong:
Is the app slow? Is it down? Which pod? Which node?
Is it a memory leak? A bad deploy? A database issue?

"kubectl get pods" shows Running — but users are complaining
"kubectl logs" only shows ONE pod, and it might already be gone

You need to SEE what's happening across the whole cluster,
historically and in real time — not guess
```

Monitoring tells you WHAT is wrong (metrics — CPU, memory, request rate, error rate). Logging tells you WHY it's wrong (the actual log lines from the app). Health checks tell Kubernetes WHEN to act automatically.

---

### What You Will Learn Today

- The three pillars of observability — metrics, logs, traces
- Prometheus — metrics collection
- Grafana — visualization and dashboards
- PromQL basics
- Alerting with Alertmanager
- Centralized logging patterns
- EFK / ELK stack overview
- Health checks in depth — startup, liveness, readiness
- Real DevOps observability patterns

---

## The Three Pillars of Observability

```
Metrics  → numbers over time (CPU %, request count, error rate)
           Good for: dashboards, alerting, trends
           Tool: Prometheus + Grafana

Logs     → discrete events with detail (error stack traces, request logs)
           Good for: debugging a specific incident
           Tool: Fluentd/Fluent Bit + Elasticsearch/Loki + Kibana/Grafana

Traces   → the path a single request took across microservices
           Good for: finding which service in a chain is slow
           Tool: Jaeger, Tempo, OpenTelemetry
```

This lesson focuses on metrics and logs, the two most commonly tested in interviews and most immediately useful day-to-day.

---

## Prometheus — Metrics Collection

---

### What is Prometheus?

Prometheus is a time-series database and monitoring system that pulls (scrapes) metrics from your applications and infrastructure at regular intervals, then stores them so you can query and alert on them.

```
How Prometheus works:
1. Your app/exporter exposes metrics at /metrics endpoint (plain text)
2. Prometheus scrapes that endpoint every N seconds (e.g. 15s)
3. Stores each metric as a time series (value + timestamp + labels)
4. You query historical/current data using PromQL
5. Alertmanager fires alerts when query conditions are met
```

```bash
# Example of what a /metrics endpoint looks like:
http_requests_total{method="GET", status="200"} 15203
http_requests_total{method="GET", status="500"} 12
node_memory_usage_bytes 1073741824
```

---

### Installing Prometheus via Helm

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# kube-prometheus-stack bundles Prometheus + Grafana + Alertmanager
helm install monitoring prometheus-community/kube-prometheus-stack \
    --namespace monitoring --create-namespace

kubectl get pods -n monitoring
kubectl get svc -n monitoring
```

---

### How Prometheus Discovers What to Scrape

```bash
# Kubernetes Service Discovery — Prometheus auto-discovers targets:
# - Pods with the right annotations
# - ServiceMonitor / PodMonitor custom resources (kube-prometheus-stack)

# Example: annotate a pod so Prometheus scrapes it automatically:
metadata:
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "9090"
    prometheus.io/path: "/metrics"
```

---

### ServiceMonitor — The Modern Way

```yaml
# servicemonitor.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: myapp-monitor
  namespace: monitoring
  labels:
    release: monitoring
spec:
  selector:
    matchLabels:
      app: myapp              # matches the Service's labels
  namespaceSelector:
    matchNames:
    - production
  endpoints:
  - port: metrics              # name of the port in the Service spec
    interval: 15s
    path: /metrics
```

```bash
# Your Service must expose a port named "metrics" for this to work:
spec:
  ports:
  - name: metrics
    port: 9090
    targetPort: 9090
```

---

### Exposing Application Metrics

```bash
# Most languages have a Prometheus client library:
# Python: prometheus_client
# Node.js: prom-client
# Go: client_golang
# Java: micrometer

# Example exposed metrics from a typical web app:
http_requests_total              # counter — total requests
http_request_duration_seconds    # histogram — latency distribution
process_resident_memory_bytes    # gauge — current memory usage

# Metric types:
# Counter   → only goes up (total requests, total errors)
# Gauge     → goes up and down (current memory, active connections)
# Histogram → buckets of observed values (request duration)
# Summary   → similar to histogram, calculated client-side
```

---

### PromQL — Querying Metrics

```bash
# Basic queries (run in Prometheus UI or Grafana):

# Current value of a metric:
node_memory_usage_bytes

# Filter by label:
http_requests_total{status="500"}

# Rate of increase over 5 minutes (for counters):
rate(http_requests_total[5m])

# CPU usage percentage per pod:
sum(rate(container_cpu_usage_seconds_total{namespace="production"}[5m])) by (pod)

# Memory usage per pod:
sum(container_memory_usage_bytes{namespace="production"}) by (pod)

# Error rate as a percentage:
sum(rate(http_requests_total{status="500"}[5m]))
/
sum(rate(http_requests_total[5m])) * 100

# Pods restarting (a key health signal):
kube_pod_container_status_restarts_total
```

---

### Alertmanager — Alerting Rules

```yaml
# alertrule.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: myapp-alerts
  namespace: monitoring
  labels:
    release: monitoring
spec:
  groups:
  - name: myapp.rules
    rules:
    - alert: HighErrorRate
      expr: |
        sum(rate(http_requests_total{status="500"}[5m]))
        /
        sum(rate(http_requests_total[5m])) > 0.05
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "Error rate above 5% for 5 minutes"

    - alert: PodCrashLooping
      expr: increase(kube_pod_container_status_restarts_total[15m]) > 3
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "Pod {{ $labels.pod }} restarting frequently"

    - alert: HighMemoryUsage
      expr: |
        container_memory_usage_bytes / container_spec_memory_limit_bytes > 0.9
      for: 10m
      labels:
        severity: warning
      annotations:
        summary: "Pod {{ $labels.pod }} using over 90% of memory limit"
```

```bash
# Alertmanager routes alerts to:
# Slack, PagerDuty, email, webhook — configured separately
# in Alertmanager's own config, grouped by labels like severity
```

---

## Grafana — Visualization

---

### What is Grafana?

Grafana connects to data sources like Prometheus and renders the metrics as dashboards — graphs, gauges, tables, alerts — for humans to actually look at.

```bash
# Access Grafana (after installing kube-prometheus-stack):
kubectl port-forward svc/monitoring-grafana 3000:80 -n monitoring

# Default login (check chart docs — often):
# username: admin
# password: kubectl get secret -n monitoring monitoring-grafana \
#           -o jsonpath="{.data.admin-password}" | base64 --decode

# Open browser: http://localhost:3000
```

---

### Common Dashboards to Import

```bash
# kube-prometheus-stack ships with many pre-built dashboards:
# - Kubernetes / Compute Resources / Namespace (Pods)
# - Kubernetes / Compute Resources / Node (Pods)
# - Node Exporter / Nodes
# - Kubernetes / Persistent Volumes

# Or import community dashboards by ID from grafana.com/dashboards
# e.g. Node Exporter Full = dashboard ID 1860
```

---

### Building a Simple Panel

```bash
# In Grafana UI:
# 1. New Dashboard → Add Panel
# 2. Data source: Prometheus
# 3. Query: sum(rate(http_requests_total[5m])) by (status)
# 4. Visualization: Time series
# 5. Set thresholds/alerts directly on the panel if needed
```

---

## Centralized Logging

---

### The Logging Problem in Kubernetes

```bash
# Without centralized logging:
kubectl logs myapp-7d9f-abc123    # only works if the pod still exists
# Pod crashes and gets replaced → old logs GONE forever
# 10 replicas → checking logs one pod at a time is impossible at scale
```

Centralized logging ships every container's logs to one searchable place, regardless of which pod produced them or whether that pod still exists.

---

### EFK Stack — Elasticsearch, Fluentd/Fluent Bit, Kibana

```
Flow:
Container writes logs to stdout/stderr
        ↓
Fluent Bit (DaemonSet — runs on every node) collects logs
        ↓
Forwards to Elasticsearch (stores and indexes logs)
        ↓
Kibana (UI) — search, filter, visualize logs
```

```yaml
# Fluent Bit runs as a DaemonSet so it's on every node automatically:
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluent-bit
  namespace: logging
spec:
  selector:
    matchLabels:
      app: fluent-bit
  template:
    metadata:
      labels:
        app: fluent-bit
    spec:
      containers:
      - name: fluent-bit
        image: fluent/fluent-bit:latest
        volumeMounts:
        - name: varlog
          mountPath: /var/log
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
```

```bash
# Install via Helm in practice (much simpler than raw YAML):
helm repo add elastic https://helm.elastic.co
helm install elasticsearch elastic/elasticsearch -n logging --create-namespace
helm install kibana elastic/kibana -n logging
helm install fluent-bit fluent/fluent-bit -n logging
```

---

### Loki — Lightweight Alternative to Elasticsearch

```bash
# Loki (by Grafana Labs) is a simpler, cheaper logging backend
# Designed to integrate directly into Grafana — same UI as your metrics

helm repo add grafana https://grafana.github.io/helm-charts
helm install loki grafana/loki-stack \
    --namespace logging --create-namespace \
    --set grafana.enabled=false   # reuse existing Grafana from monitoring stack

# Loki indexes only labels (not full text like Elasticsearch) —
# cheaper to run, and queries use LogQL (similar style to PromQL)
```

```bash
# Example LogQL queries in Grafana:
{namespace="production", pod="myapp-7d9f"}
{namespace="production"} |= "error"
{app="myapp"} |= "error" | json | status_code = "500"
```

---

### Why You Never Store Logs Inside the Container

```bash
# Logs written to a file inside a container:
# - Lost when the container restarts
# - Not visible to centralized tooling unless explicitly shipped
# - Fills up the container's writable layer (especially bad with
#   readOnlyRootFilesystem: true from Day 4)

# Best practice — apps log to stdout/stderr only:
# Kubernetes captures stdout/stderr automatically
# Fluent Bit/Fluentd picks it up from the node's container runtime logs
# No application code changes needed beyond "log to console"
```

---

## Health Checks in Depth

---

### Three Types of Probes

```
Liveness Probe  → "Is this container alive?"
                  Fails → kubelet KILLS and restarts the container

Readiness Probe → "Is this container ready for traffic?"
                  Fails → pod removed from Service endpoints,
                          NOT restarted, traffic stops flowing to it

Startup Probe   → "Has this container finished starting up?"
                  Disables liveness/readiness checks until it passes
                  Prevents slow-starting apps from being killed
                  before they even finish booting
```

---

### Startup Probe — Solving the Slow-Boot Problem

```bash
# Problem without startup probe:
# App takes 60 seconds to fully initialize
# Liveness probe starts checking immediately, fails repeatedly
# kubelet thinks app is broken → kills it → restart loop forever

# Solution: startup probe covers the slow boot, THEN liveness takes over
```

```yaml
spec:
  containers:
  - name: myapp
    image: varun/myapp:1.0.0
    startupProbe:
      httpGet:
        path: /health
        port: 5000
      failureThreshold: 30      # 30 attempts...
      periodSeconds: 10          # ...every 10 seconds = 5 min max startup time
    livenessProbe:
      httpGet:
        path: /health
        port: 5000
      periodSeconds: 10
      failureThreshold: 3
    readinessProbe:
      httpGet:
        path: /ready
        port: 5000
      periodSeconds: 5
      failureThreshold: 3
```

---

### Three Probe Mechanisms

```yaml
# 1. httpGet — most common, checks an HTTP endpoint
livenessProbe:
  httpGet:
    path: /health
    port: 5000
    httpHeaders:
    - name: Custom-Header
      value: probe

# 2. tcpSocket — just checks if a port is open (no HTTP needed)
livenessProbe:
  tcpSocket:
    port: 3306

# 3. exec — runs a command inside the container; success = exit code 0
livenessProbe:
  exec:
    command:
    - cat
    - /tmp/healthy
```

---

### Probe Tuning Parameters

```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 5000
  initialDelaySeconds: 15    # wait before first check
  periodSeconds: 20           # check every 20 seconds
  timeoutSeconds: 5            # fail if no response within 5s
  successThreshold: 1           # 1 success = considered healthy
  failureThreshold: 3            # 3 consecutive failures = act
```

```bash
# Common mistake: making readiness too strict so a momentary blip
# (e.g. a slow DB query) removes a healthy pod from rotation
# unnecessarily — tune failureThreshold and periodSeconds to your
# app's real behavior, not arbitrary defaults
```

---

### Designing a Good /health Endpoint

```bash
# A good /health endpoint should check things that actually mean
# "this process can serve traffic correctly," not just "the process
# is running" — for example:

/health (liveness)  → process responds at all, no deadlock
/ready (readiness)  → can the app actually reach its DB, cache,
                       and any required downstream dependency

# Anti-pattern: making liveness check the database
# If the DB goes down, EVERY pod's liveness fails → K8s restarts
# all pods → restart storm makes the outage worse, not better
# Database connectivity belongs in READINESS, not liveness
```

---

### Debugging with Health Check Failures

```bash
kubectl describe pod myapp-7d9f-abc
# Look for "Liveness probe failed" / "Readiness probe failed" events

kubectl get events --sort-by='.lastTimestamp' -n production

kubectl logs myapp-7d9f-abc --previous
# --previous shows logs from BEFORE the last restart — critical
# for debugging why a liveness probe killed the container
```

---

### Full Summary — K8s Day 5

| Concept | Key point |
|---|---|
| Metrics | Numbers over time — Prometheus scrapes, stores, queries |
| Prometheus | Pull-based time-series metrics database |
| ServiceMonitor | Tells Prometheus what to scrape, via Service labels |
| PromQL | Query language for metrics — rate(), sum(), by() |
| Alertmanager | Routes alerts (Slack, PagerDuty) based on PromQL rules |
| Grafana | Visualizes Prometheus (and Loki) data as dashboards |
| Logs | Discrete events — centralized so they survive pod death |
| EFK / ELK | Fluentd/Fluent Bit + Elasticsearch + Kibana logging stack |
| Loki | Lightweight log backend, integrates natively with Grafana |
| Liveness probe | Restarts container if it fails |
| Readiness probe | Removes pod from traffic if it fails, no restart |
| Startup probe | Delays liveness/readiness checks during slow boot |
| stdout/stderr logging | Apps should log to console, not files, inside containers |

---

### Interview Questions — K8s Day 5

**Q1. What is the difference between monitoring and logging, and why do you need both?**
Monitoring (metrics) gives you aggregated numerical data over time, such as CPU usage, request rate or error rate, which is ideal for dashboards, trend analysis and alerting on thresholds. Logging gives you detailed discrete event records, such as a specific error stack trace, which is essential for debugging exactly why something went wrong. Metrics tell you something is wrong and roughly where; logs tell you precisely why.

**Q2. How does Prometheus collect metrics, and what is a ServiceMonitor?**
Prometheus uses a pull-based model — it scrapes a `/metrics` HTTP endpoint exposed by applications or exporters at a configured interval, rather than applications pushing data to it. A ServiceMonitor is a Kubernetes custom resource, used by the Prometheus Operator, that tells Prometheus which Services to scrape by matching their labels, along with which port and path to use, removing the need to manually configure scrape targets.

**Q3. What is the difference between a liveness probe and a readiness probe, and what's a common mistake with them?**
A liveness probe checks if a container is alive; if it fails, kubelet kills and restarts the container. A readiness probe checks if a container can currently serve traffic; if it fails, the pod is removed from the Service's endpoints without being restarted. A common mistake is making the liveness probe check downstream dependencies like a database — if the database goes down, this causes every pod to fail liveness and restart simultaneously, turning a database outage into an unnecessary application-wide restart storm.

**Q4. What problem does a startup probe solve?**
A startup probe is meant for containers that take a long time to initialize. Without it, a liveness probe might start checking immediately and repeatedly fail before the application has even finished booting, causing kubelet to kill and restart the container in a loop before it ever gets a chance to become healthy. A startup probe disables liveness and readiness checks until it succeeds, giving slow-starting applications enough time to boot.

**Q5. Why is centralized logging necessary in Kubernetes instead of just using `kubectl logs`?**
`kubectl logs` only works while the specific pod still exists, and once a pod is deleted or replaced, its logs are gone. At scale, with many replicas across many nodes, checking logs pod by pod is impractical. Centralized logging tools, such as Fluent Bit shipping logs to Elasticsearch or Loki, collect logs from every container as they're produced and store them in one searchable location, so logs survive pod restarts and can be searched across the whole cluster at once.

**Q6. Why should containerized applications log to stdout/stderr instead of writing to log files?**
Kubernetes and the underlying container runtime automatically capture stdout and stderr output, which log shippers like Fluent Bit can pick up directly without any extra application configuration. Writing logs to a file inside the container instead means those logs are lost on restart, aren't automatically visible to centralized logging tools, and can conflict with security hardening such as a read-only root filesystem.

**Q7. What does the PromQL query `rate(http_requests_total[5m])` actually calculate?**
`http_requests_total` is a counter metric that only ever increases. `rate()` calculates the per-second average rate of increase of that counter over the specified time window, in this case 5 minutes, which converts a constantly growing raw total into a meaningful "requests per second" value that can be graphed or alerted on.

---

### Homework — Before K8s Day 6

```bash
# 1. Install kube-prometheus-stack via Helm:
helm install monitoring prometheus-community/kube-prometheus-stack \
    -n monitoring --create-namespace

# 2. Port-forward Grafana and log in, explore the pre-built
#    Kubernetes dashboards:
kubectl port-forward svc/monitoring-grafana 3000:80 -n monitoring

# 3. Write a PromQL query in the Prometheus UI to show:
#    - Total pod restarts in the last 15 minutes
#    - CPU usage by pod in your namespace

# 4. Create a PrometheusRule that alerts when a pod restarts
#    more than 3 times in 15 minutes

# 5. Install Loki (or EFK) and view logs for a deployment with
#    multiple replicas across one searchable interface

# 6. Add a startup probe to a deployment with a deliberately
#    slow-starting container (sleep 30 on boot) and observe
#    how it behaves with vs without the startup probe
```

---

### Your Progress

```
Linux  ████████████████████  ✅ COMPLETE
Git    ████████████████████  ✅ COMPLETE
Docker ████████████████████  ✅ COMPLETE
AWS    ████████████████████  ✅ COMPLETE
CI/CD  ████████████████████  ✅ COMPLETE
K8s    ██████████░░░░░░░░░░  Day 5 of 6
Final  ░░░░░░░░░░░░░░░░░░░░  Interview prep
```

K8s Day 6 — the final day — we cover **Helm in production patterns, cluster upgrades, disaster recovery (etcd backup/restore), and a full end-to-end project tying everything together**. 💪

Say **"K8s Day 6"** whenever you are ready!
