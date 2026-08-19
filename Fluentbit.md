# Fluentbit

```text
Application
    ↓
Container stdout/stderr
    ↓
Kubernetes node
    ↓
/var/log/containers/*.log
    ↓
Fluent Bit DaemonSet
    ↓
Parse / enrich / filter
    ↓
Dynatrace
    ↓
Logs / dashboards / alerts
```

One important distinction up front:

> **Fluent Bit is the log shipper/collector. Dynatrace is the observability backend.** Fluent Bit does not replace Dynatrace; it collects, processes and forwards logs to it.

---

# 1. What is Fluent Bit?

**Fluent Bit is a lightweight, high-performance telemetry agent/collector used to collect, parse, filter, enrich and forward logs and other telemetry.**

For Kubernetes, one of the most common patterns is:

```text
Fluent Bit = node-level log collector
```

You normally run it as a:

```text
DaemonSet
```

so that:

```text
1 Kubernetes node
        ↓
1 Fluent Bit pod
```

Architecture:

```text
                    EKS NODE
┌─────────────────────────────────────────────┐
│                                             │
│   Pod A          Pod B          Pod C       │
│     │              │              │         │
│ stdout/stderr   stdout/stderr   stdout      │
│     │              │              │         │
│     └──────────────┴──────────────┘         │
│                    │                        │
│                    v                        │
│          /var/log/containers/*.log          │
│                    │                        │
│                    v                        │
│            Fluent Bit DaemonSet             │
│                    │                        │
└────────────────────┼────────────────────────┘
                     │
                     │ HTTPS
                     v
                 Dynatrace
```

---

# 2. Why do we need Fluent Bit?

Suppose you have:

```text
EKS
 |
 +-- Node 1
 |    +-- payment-api
 |    +-- order-api
 |
 +-- Node 2
 |    +-- frontend
 |    +-- auth-api
 |
 +-- Node 3
      +-- worker
```

Each application writes:

```text
stdout
stderr
```

Kubernetes/container runtime stores those logs on the node.

You don't want applications individually doing:

```text
Application
   |
   +--> Dynatrace
   |
   +--> Elasticsearch
   |
   +--> Splunk
```

Instead:

```text
Applications
     |
     v
Node log files
     |
     v
Fluent Bit
     |
     +------> Dynatrace
     |
     +------> Elasticsearch
     |
     +------> S3
```

This gives you a centralized logging pipeline.

---

# 3. Fluent Bit architecture

The basic Fluent Bit architecture is:

```text
              +------------------+
              |     INPUT        |
              |                  |
              | Tail / Systemd   |
              | TCP / Forward    |
              +--------+---------+
                       |
                       v
              +------------------+
              |     PARSER       |
              |                  |
              | JSON / Regex     |
              +--------+---------+
                       |
                       v
              +------------------+
              |     FILTER       |
              |                  |
              | Kubernetes       |
              | Modify / grep    |
              +--------+---------+
                       |
                       v
              +------------------+
              |      BUFFER      |
              |                  |
              | Memory / Disk     |
              +--------+---------+
                       |
                       v
              +------------------+
              |     OUTPUT       |
              |                  |
              | Dynatrace        |
              | Elasticsearch    |
              | S3               |
              +------------------+
```

The four words you should immediately remember:

```text
INPUT
  ↓
FILTER
  ↓
BUFFER
  ↓
OUTPUT
```

Parsers are used to interpret the incoming records, while filters transform/enrich them between input and output.

---

# 4. Major Fluent Bit components

| Component         | Purpose                                      |
| ----------------- | -------------------------------------------- |
| Input             | Where logs come from                         |
| Parser            | Converts raw log text into structured fields |
| Filter            | Modify/enrich/drop records                   |
| Buffer            | Temporarily stores records                   |
| Output            | Sends records to destination                 |
| Service           | Global/runtime configuration                 |
| Kubernetes filter | Adds Kubernetes metadata                     |
| Tail input        | Reads files                                  |
| Forward input     | Receives Fluent Forward protocol             |
| Systemd input     | Reads systemd journal                        |
| HTTP input        | Receives HTTP                                |
| Multiline parser  | Handles stack traces/multiline logs          |

For your Kubernetes use case:

```text
Tail
 ↓
Kubernetes Filter
 ↓
Parser
 ↓
Multiline
 ↓
Output
```

---

# 5. Why DaemonSet?

This is an important Kubernetes interview question.

Suppose:

```text
Node 1 → 50 Pods
Node 2 → 70 Pods
Node 3 → 40 Pods
```

You want one Fluent Bit per node:

```text
Node 1                    Node 2                    Node 3
+----------------+        +----------------+        +----------------+
| Pods            |        | Pods            |        | Pods            |
|                |        |                |        |                |
| Fluent Bit     |        | Fluent Bit     |        | Fluent Bit     |
+-------+--------+        +-------+--------+        +-------+--------+
        |                         |                         |
        +-------------------------+-------------------------+
                                  |
                                  v
                              Dynatrace
```

Why?

Because the log files are local to the node.

If Fluent Bit were just one Deployment:

```text
Fluent Bit
    |
    X
Node 1 logs
Node 2 logs
Node 3 logs
```

it wouldn't naturally have access to all node-local log files.

A DaemonSet ensures:

> **Every node gets a log collector.**

---

# 6. Kubernetes pod logging flow

This is critical.

Application:

```text
Python / Java / Go / Node
           |
           |
      stdout/stderr
           |
           v
    Container runtime
           |
           v
Kubernetes node filesystem
```

Typically you'll see:

```text
/var/log/containers/
```

which contains links to container log files.

You may also encounter:

```text
/var/log/pods/
```

The exact runtime/file layout matters, but the key interview concept is:

> Fluent Bit tails node-local container log files rather than reading application internals directly.

---

# 7. Example application log

Suppose:

```text
payment-api
```

prints:

```json
{"timestamp":"2026-08-19T10:30:15Z","level":"INFO","message":"Payment successful","order_id":"ORD-1001"}
```

Fluent Bit reads it.

Initially:

```text
raw log
```

Then parser:

```text
timestamp
level
message
order_id
```

Then Kubernetes filter adds:

```text
namespace
pod_name
container_name
container_id
labels
```

Then:

```text
Fluent Bit
       |
       v
Dynatrace
```

---

# 8. What does Fluent Bit actually see?

Suppose the raw container log is:

```text
2026-08-19T10:30:15.123456789Z stdout F {"level":"INFO","message":"Payment successful"}
```

The important pieces are:

```text
timestamp
stdout
F
application payload
```

Fluent Bit can parse the container runtime format and then process the application payload.

---

# 9. Kubernetes metadata enrichment

This is one of Fluent Bit's biggest advantages in Kubernetes.

Suppose application says:

```json
{
  "level":"ERROR",
  "message":"Payment failed"
}
```

Fluent Bit can enrich it with:

```text
namespace = payment
pod_name = payment-api-7d6c9
container_name = payment
node_name = ip-10-0-1-25
labels.app = payment-api
```

So Dynatrace can show:

```text
Payment failed

namespace:
payment

pod:
payment-api-7d6c9

container:
payment

node:
ip-10-0-1-25
```

This is extremely useful for troubleshooting.

---

# 10. Fluent Bit input

The most important input for Kubernetes pod logs:

```text
tail
```

Example:

```ini
[INPUT]
    Name              tail
    Path              /var/log/containers/*.log
    Parser            cri
    Tag               kube.*
    Mem_Buf_Limit     50MB
    Skip_Long_Lines   On
```

Meaning:

```text
Name
 ↓
tail files

Path
 ↓
container log files

Parser
 ↓
interpret container log format

Tag
 ↓
identify records
```

---

# 11. What is a Tag?

You might see:

```ini
Tag kube.*
```

Think of a tag as an internal routing/classification identifier.

Example:

```text
kube.var.log.containers.payment-api-xxx.log
```

Then filters/output can use:

```text
kube.*
```

to process Kubernetes logs.

---

# 12. Kubernetes filter

Example:

```ini
[FILTER]
    Name                kubernetes
    Match               kube.*
    Merge_Log           On
    Keep_Log            Off
    K8S-Logging.Parser  On
    K8S-Logging.Exclude On
```

This filter:

```text
Fluent Bit
    |
    | pod log
    v
Kubernetes filter
    |
    +--> namespace
    +--> pod
    +--> container
    +--> labels
    +--> annotations
```

---

# 13. `Merge_Log`

Suppose application writes:

```json
{"level":"ERROR","message":"Database connection failed"}
```

Without appropriate processing, the entire JSON may remain as one field:

```text
log = '{"level":"ERROR","message":"Database connection failed"}'
```

With:

```ini
Merge_Log On
```

Fluent Bit can merge parsed JSON fields into the record.

Conceptually:

```text
Before:

log = JSON string


After:

level   = ERROR
message = Database connection failed
```

---

# 14. Parser

Suppose application emits JSON.

You can use JSON parsing:

```ini
[PARSER]
    Name        app_json
    Format      json
    Time_Key    timestamp
    Time_Format %Y-%m-%dT%H:%M:%SZ
```

Now:

```text
raw string
   ↓
JSON parser
   ↓
structured record
```

---

# 15. Filter

Filters can:

```text
add fields
remove fields
rename fields
modify fields
drop records
grep records
parse fields
```

Example:

```ini
[FILTER]
    Name   modify
    Match  kube.*
    Add    environment production
```

Now:

```text
environment=production
```

is added.

---

# 16. Grep filter

Suppose you only want ERROR logs.

```ini
[FILTER]
    Name    grep
    Match   kube.*
    Regex   level ERROR
```

Flow:

```text
INFO
 ↓
X dropped

WARN
 ↓
X dropped

ERROR
 ↓
✓ forwarded
```

Be careful with this in production—you normally want to preserve useful logs rather than accidentally filtering away evidence.

---

# 17. Multiline logs

Very important for Java.

Application:

```text
Exception in thread "main" java.lang.Exception
    at com.example.PaymentService.process(PaymentService.java:45)
    at com.example.PaymentService.handle(PaymentService.java:22)
    at ...
```

If Fluent Bit treats every line independently:

```text
Record 1
Exception...

Record 2
at...

Record 3
at...
```

That's bad.

Multiline processing combines them:

```text
One log event
    |
    +-- Exception
    +-- at ...
    +-- at ...
```

This is particularly important for:

```text
Java
Python
Go panic
.NET stack traces
```

---

# 18. Buffering

What happens if Dynatrace is temporarily unavailable?

You don't want:

```text
Application
    ↓
Fluent Bit
    ↓
Dynatrace X
    ↓
LOG LOST
```

Buffering helps:

```text
Application
    ↓
Fluent Bit
    ↓
Buffer
    ↓
Dynatrace X

          later

Dynatrace available
    ↓
Buffered logs sent
```

Fluent Bit supports memory and filesystem buffering mechanisms; filesystem buffering is particularly useful when you need logs to survive pressure/restarts more robustly than memory-only buffering.

---

# 19. Backpressure

Senior-level question:

> "What happens if Dynatrace becomes slow?"

Suppose:

```text
Incoming:
10,000 logs/sec

Dynatrace:
2,000 logs/sec
```

Now:

```text
10,000 in
   ↓
Fluent Bit
   ↓
Buffer grows
   ↓
2,000 out
```

If the buffer fills, you need a policy.

Possible outcomes depend on configuration:

```text
buffer growth
     ↓
retry
     ↓
filesystem buffering
     ↓
eventual delivery
```

Eventually, if the destination remains unavailable and storage is exhausted, logs can be dropped.

This is why production logging needs capacity planning.

---

# 20. Fluent Bit → Dynatrace

Now your actual target.

There are two important architectural approaches you may encounter:

### Approach A — Fluent Bit sends through Dynatrace's supported log ingestion endpoint

```text
Fluent Bit
    |
    | HTTPS
    v
Dynatrace Log Ingest API
    |
    v
Dynatrace
```

### Approach B — Use an intermediate collector/observability gateway

```text
Fluent Bit
    |
    v
Collector/Gateway
    |
    v
Dynatrace
```

Which one you use depends on your Dynatrace architecture and organization standards.

For your interview, understand **direct log ingestion first**.

---

# 21. Dynatrace authentication

Your Fluent Bit needs credentials.

Conceptually:

```text
Fluent Bit
    |
    | Authorization
    v
Dynatrace
```

You should **not** hardcode tokens into:

```text
fluent-bit.conf
```

Instead:

```text
Kubernetes Secret
       |
       v
Fluent Bit
       |
       v
Dynatrace
```

Example:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: dynatrace-logs
  namespace: logging
type: Opaque
stringData:
  DT_API_TOKEN: "REPLACE_ME"
```

In real production:

> Prefer your organization's secret-management solution rather than committing this manifest to Git with an actual token.

---

# 22. End-to-end architecture

Here's the architecture you should be able to draw in an interview:

```text
                         EKS CLUSTER
┌────────────────────────────────────────────────────────────┐
│                                                            │
│   NODE 1                     NODE 2                        │
│                                                            │
│ ┌──────────────┐          ┌──────────────┐                │
│ │ payment-api  │          │ order-api    │                │
│ │ stdout/stderr│          │ stdout/stderr│                │
│ └──────┬───────┘          └──────┬───────┘                │
│        │                         │                         │
│        v                         v                         │
│ /var/log/containers/*.log      /var/log/containers/*.log │
│        │                         │                         │
│        v                         v                         │
│ ┌──────────────┐          ┌──────────────┐                │
│ │ Fluent Bit  │          │ Fluent Bit  │                │
│ │ DaemonSet   │          │ DaemonSet   │                │
│ │             │          │             │                │
│ │ Tail        │          │ Tail        │                │
│ │ Parser      │          │ Parser      │                │
│ │ K8s Filter  │          │ K8s Filter  │                │
│ │ Buffer      │          │ Buffer      │                │
│ └──────┬───────┘          └──────┬───────┘                │
│        │                         │                         │
└────────┼─────────────────────────┼─────────────────────────┘
         │                         │
         └────────────┬────────────┘
                      │ HTTPS
                      v
             ┌───────────────────┐
             │     Dynatrace     │
             │                   │
             │ Logs              │
             │ Search            │
             │ Dashboards        │
             │ Alerts            │
             └───────────────────┘
```

---

# 23. Complete demo architecture

We'll build:

```text
Namespace: logging
        |
        +-- Fluent Bit
        |
        +-- Secret

Namespace: demo
        |
        +-- Application
        +-- Service
```

Flow:

```text
demo-app
   |
   | stdout
   v
container log
   |
   v
Fluent Bit
   |
   +--> Tail
   |
   +--> CRI parser
   |
   +--> Kubernetes metadata
   |
   +--> JSON parsing
   |
   +--> Buffer
   |
   +--> Dynatrace output
   |
   v
Dynatrace
```

---

# 24. Install Fluent Bit

For Kubernetes, use the official Fluent Bit Helm chart rather than manually creating every DaemonSet object.

Conceptually:

```bash
helm repo add fluent https://fluent.github.io/helm-charts
helm repo update
```

Create:

```bash
kubectl create namespace logging
```

Then install:

```bash
helm upgrade --install fluent-bit \
  fluent/fluent-bit \
  -n logging
```

The exact chart values should be pinned/managed through your organization's GitOps process in production.

---

# 25. Verify DaemonSet

```bash
kubectl get daemonset -n logging
```

You want:

```text
DESIRED
CURRENT
READY
```

to correspond to your node count.

For example:

```text
DESIRED   CURRENT   READY
3         3         3
```

Then:

```bash
kubectl get pods -n logging -o wide
```

You should see one Fluent Bit pod per node.

---

# 26. Mount node log directories

This is essential.

Fluent Bit needs access to:

```text
/var/log/containers
/var/log/pods
```

The DaemonSet uses hostPath mounts conceptually like:

```yaml
volumeMounts:
  - name: varlog
    mountPath: /var/log

volumes:
  - name: varlog
    hostPath:
      path: /var/log
```

This is how:

```text
Node filesystem
       |
       | hostPath
       v
Fluent Bit container
```

gets access to logs.

---

# 27. RBAC

The Kubernetes filter needs Kubernetes metadata.

So Fluent Bit generally requires permissions to query relevant Kubernetes API objects.

Conceptually:

```text
Fluent Bit ServiceAccount
       |
       v
Role/ClusterRole
       |
       v
Kubernetes API
       |
       v
Pod metadata
```

Typical permissions involve resources such as:

```text
pods
namespaces
```

and potentially related resources depending on configuration.

---

# 28. Basic Fluent Bit config

Conceptually:

```ini
[SERVICE]
    Flush        5
    Log_Level    info

[INPUT]
    Name         tail
    Path         /var/log/containers/*.log
    Parser       cri
    Tag          kube.*
    Mem_Buf_Limit 50MB
    Skip_Long_Lines On

[FILTER]
    Name         kubernetes
    Match        kube.*
    Merge_Log    On
    Keep_Log     Off

[OUTPUT]
    Name         <Dynatrace output>
    Match        kube.*
    ...
```

The exact Dynatrace output/plugin configuration depends on the Dynatrace ingestion method and the Fluent Bit plugin/output supported by your environment.

---

# 29. Why I don't want you memorizing a fake Dynatrace output block

This is important for an interview.

Dynatrace ingestion endpoints, authentication, and supported Fluent Bit integration options can vary by Dynatrace deployment/version and organization architecture.

So don't memorize something like:

```ini
[OUTPUT]
Name dynatrace
...
```

unless you have verified that **your organization's actual Fluent Bit integration uses that output plugin**.

The architecture is stable:

```text
Fluent Bit
   |
   | HTTPS + token
   v
Dynatrace ingestion
```

but the exact configuration should be taken from the current Dynatrace integration documentation/environment.

---

# 30. A generic HTTP output example

If your chosen Dynatrace ingestion path is HTTP-compatible, the conceptual configuration is:

```ini
[OUTPUT]
    Name        http
    Match       kube.*
    Host        <dynatrace-endpoint>
    Port        443
    URI         <log-ingest-path>
    Format      json
    tls         On

    Header      Authorization Bearer <token>
```

**Don't put a real token directly in this config.**

Use environment variables / secret injection or the supported Helm Secret mechanism.

---

# 31. Secret-based authentication

Conceptually:

```text
Kubernetes Secret
       |
       v
Fluent Bit Pod
       |
       | environment / mounted secret
       v
Dynatrace output
```

Example:

```yaml
env:
  - name: DT_API_TOKEN
    valueFrom:
      secretKeyRef:
        name: dynatrace-logs
        key: DT_API_TOKEN
```

Then your configuration references the injected credential using the mechanism supported by your output plugin/chart.

---

# 32. Application demo

Let's create an application that emits JSON logs.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: log-demo
  namespace: demo

spec:
  replicas: 2

  selector:
    matchLabels:
      app: log-demo

  template:
    metadata:
      labels:
        app: log-demo

    spec:
      containers:
        - name: app
          image: busybox:1.36

          command:
            - /bin/sh
            - -c
            - |
              i=0
              while true; do
                echo "{\"level\":\"INFO\",\"message\":\"Payment processed\",\"request_id\":\"req-$i\",\"service\":\"payment-api\"}"
                i=$((i+1))
                sleep 5
              done
```

Create namespace:

```bash
kubectl create namespace demo
```

Apply:

```bash
kubectl apply -f app.yaml
```

---

# 33. Check application logs

```bash
kubectl logs -n demo deployment/log-demo
```

You'll see:

```text
{"level":"INFO","message":"Payment processed","request_id":"req-0","service":"payment-api"}
{"level":"INFO","message":"Payment processed","request_id":"req-1","service":"payment-api"}
```

The application knows nothing about Fluent Bit.

That's important.

```text
Application
    |
    | stdout
    X
No logging SDK required
```

---

# 34. What happens next?

Container runtime writes:

```text
/var/log/containers/log-demo-....log
```

Fluent Bit sees:

```text
new line
   |
   v
Tail input
```

Then:

```text
CRI parser
```

Then:

```text
Kubernetes filter
```

Then:

```text
JSON parsing
```

Then:

```text
buffer
```

Then:

```text
Dynatrace
```

---

# 35. Final structured log

Conceptually, the record arriving at Dynatrace can contain:

```json
{
  "timestamp": "2026-08-19T10:30:00Z",
  "level": "INFO",
  "message": "Payment processed",
  "request_id": "req-123",
  "service": "payment-api",

  "kubernetes": {
    "namespace_name": "demo",
    "pod_name": "log-demo-7d6c8",
    "container_name": "app",
    "node_name": "ip-10-0-1-20"
  }
}
```

The exact field names depend on the Fluent Bit/Dynatrace integration.

---

# 36. Test failure scenario

This is a great interview demonstration.

Modify application:

```text
ERROR
Payment database connection failed
```

Fluent Bit sends:

```text
ERROR
namespace=payment
pod=payment-api
node=worker-1
```

Dynatrace lets you correlate:

```text
ERROR logs
      |
      +-- pod
      +-- namespace
      +-- node
      +-- service
```

Now you can investigate:

```text
Pod restart
     +
Error logs
     +
Node issue
```

---

# 37. Important: stdout vs file logging

For Kubernetes:

### Preferred

```text
Application
    |
 stdout/stderr
    |
 Kubernetes
    |
 Fluent Bit
```

rather than:

```text
Application
    |
 /app/logs/application.log
```

Why?

Kubernetes naturally captures stdout/stderr.

You avoid:

```text
sidecar per application
```

for basic logging.

---

# 38. Sidecar vs DaemonSet

Interview question:

> "Why not use a Fluent Bit sidecar in every Pod?"

You can, but it is often unnecessary.

### DaemonSet

```text
Node
 |
 +-- Pod A
 +-- Pod B
 +-- Pod C
 |
 +-- Fluent Bit
```

Advantages:

```text
low overhead
one collector/node
central configuration
```

### Sidecar

```text
Pod
 |
 +-- Application
 |
 +-- Fluent Bit
```

Advantages:

```text
specialized log processing
isolated application pipeline
```

Disadvantages:

```text
more containers
more CPU/memory
more operational complexity
```

For normal Kubernetes stdout/stderr collection:

> **DaemonSet is usually the default pattern.**

---

# 39. Fluent Bit vs Fluentd

Another interview question.

Both are from the Fluent ecosystem.

### Fluent Bit

```text
lightweight
low memory
high performance
C
edge/agent
```

### Fluentd

```text
heavier
Ruby-based
more complex processing
aggregation
```

A common architecture historically was:

```text
Fluent Bit
    |
    v
Fluentd
    |
    v
Backend
```

But Fluent Bit can often send directly to the destination.

---

# 40. Fluent Bit vs Filebeat

Common comparison:

```text
Fluent Bit
    |
    +-- lightweight
    +-- Kubernetes friendly
    +-- CNCF ecosystem
    +-- high performance
```

Filebeat:

```text
Elastic ecosystem
    |
    +-- Elasticsearch
    +-- Logstash
    +-- Kibana
```

For your environment:

```text
EKS
+
Dynatrace
```

Fluent Bit is a very natural lightweight node-level collector.

---

# 41. Important production concerns

As a Senior Platform Engineer, don't stop at:

> "I deployed Fluent Bit."

Talk about:

```text
1. Buffering
2. Backpressure
3. Retry
4. Multiline
5. Cardinality
6. Sensitive data
7. Resource limits
8. Log volume
9. Destination failures
10. DaemonSet upgrades
11. Configuration management
12. Security
13. TLS
14. Authentication
15. Data loss strategy
```

---

# 42. PII/secrets

Very important.

Imagine:

```text
password=abc123
credit_card=4111111111111111
authorization=Bearer ...
```

You don't want:

```text
Application
   ↓
Fluent Bit
   ↓
Dynatrace
```

to blindly forward them.

Use filters/parsers/redaction strategies to remove sensitive fields.

Conceptually:

```text
Raw log
   |
   v
Fluent Bit
   |
   | redact
   v
Safe log
   |
   v
Dynatrace
```

---

# 43. Resource limits

Fluent Bit itself consumes CPU and memory.

Example:

```yaml
resources:
  requests:
    cpu: 100m
    memory: 100Mi

  limits:
    cpu: 500m
    memory: 500Mi
```

But don't blindly use these values.

You should size based on:

```text
nodes
logs/sec
average log size
burst rate
filters
parsing complexity
buffer size
destination throughput
```

---

# 44. Log loss scenario

Suppose:

```text
Application
100 MB/min
```

Dynatrace goes down:

```text
30 minutes
```

You potentially have:

```text
3 GB
```

of logs to buffer.

If Fluent Bit has:

```text
200 MB
```

of buffering capacity:

```text
200 MB < 3 GB
```

You cannot retain everything locally.

This is why buffer sizing is an engineering decision.

---

# 45. Fluent Bit troubleshooting flow

This is an excellent interview answer.

Suppose:

> "Logs from one namespace aren't reaching Dynatrace."

I would troubleshoot layer by layer:

```text
1. Application
       ↓
kubectl logs

2. Node log file
       ↓
/var/log/containers

3. Fluent Bit
       ↓
kubectl logs fluent-bit

4. Input
       ↓
Tail configuration

5. Parser
       ↓
CRI/JSON/multiline

6. Kubernetes filter
       ↓
metadata enrichment

7. Buffer
       ↓
backpressure/retries

8. Output
       ↓
Dynatrace endpoint/auth/TLS

9. Dynatrace
       ↓
ingestion/query
```

---

# 46. Commands I'd use

Check Fluent Bit:

```bash
kubectl get pods -n logging
```

Logs:

```bash
kubectl logs -n logging daemonset/fluent-bit
```

Describe:

```bash
kubectl describe pod -n logging <fluent-bit-pod>
```

Check node:

```bash
kubectl get nodes
```

Check application:

```bash
kubectl logs -n demo <pod>
```

Check Fluent Bit config:

```bash
kubectl get configmap -n logging
```

Check DaemonSet:

```bash
kubectl describe daemonset -n logging fluent-bit
```

---

# 47. Debug architecture

```text
                 Logs missing
                      |
                      v
              kubectl logs
                      |
                 Application?
                      |
             +--------+--------+
             |                 |
             NO                YES
             |                 |
         Fix app       Check node log
                               |
                               v
                         Fluent Bit?
                               |
                         +-----+-----+
                         |           |
                        NO          YES
                         |           |
                     Check RBAC   Check parser
                                     |
                                     v
                                Check buffer
                                     |
                                     v
                                Check output
                                     |
                                     v
                               Dynatrace API
```

---

# 48. The senior-level architecture I would recommend

For your EKS environment:

```text
                        EKS
                         |
        +----------------+----------------+
        |                |                |
       Node             Node             Node
        |                |                |
    Applications     Applications     Applications
        |                |                |
        +----------------+----------------+
                         |
                    stdout/stderr
                         |
                         v
                 /var/log/containers
                         |
                         v
              Fluent Bit DaemonSet
                         |
          +--------------+--------------+
          |              |              |
          v              v              v
       Tail          Kubernetes      Multiline
                       Filter          Parser
          |              |              |
          +--------------+--------------+
                         |
                         v
                      Buffer
                         |
                         v
                       HTTPS
                         |
                         v
                    Dynatrace
                         |
              +----------+----------+
              |          |          |
             Logs     Dashboards   Alerts
```

---

# 49. The interview answer: "Explain Fluent Bit"

You can answer:

> "Fluent Bit is a lightweight telemetry collector that I can deploy as a DaemonSet on Kubernetes nodes. For pod logs, applications write to stdout/stderr, Kubernetes/container runtime writes those logs to node-local container log files, and Fluent Bit uses the Tail input to collect them. I use the CRI or appropriate container parser to interpret the container log format, then the Kubernetes filter enriches each record with namespace, pod, container and other metadata. Additional filters can parse JSON, redact sensitive fields or handle multiline stack traces. Fluent Bit buffers records and forwards them over TLS to our observability backend, in our case Dynatrace. If the backend is unavailable, buffering and retry mechanisms provide resilience, subject to configured storage limits."

That's a **Senior-level answer**.

---

# 50. The Dynatrace interview answer

If asked:

> **"How do you ship Kubernetes pod logs to Dynatrace?"**

Say:

> "I deploy Fluent Bit as a DaemonSet so each Kubernetes node has a collector. It mounts the node's container log directories and uses the Tail input to read pod stdout/stderr logs. I configure the appropriate CRI parser, Kubernetes filter for metadata enrichment, JSON/multiline processing where required, buffering for destination failures, and a Dynatrace output/HTTP ingestion path over TLS. Authentication is provided through a Kubernetes Secret or our centralized secret-management mechanism rather than hardcoding credentials. The resulting records are ingested into Dynatrace with Kubernetes context such as namespace, pod and container, allowing us to correlate logs with the workload and infrastructure."

---

# 51. The five things you absolutely need to remember

For your interview, remember this:

```text
                FLUENT BIT

                 INPUT
                   |
              Tail / Systemd
                   |
                   v
                PARSER
                   |
            CRI / JSON / Regex
                   |
                   v
                FILTER
                   |
       Kubernetes / Modify / Grep
                   |
                   v
                BUFFER
                   |
             memory/filesystem
                   |
                   v
                OUTPUT
                   |
                   v
              Dynatrace
```

And Kubernetes:

```text
Pod
 ↓
stdout/stderr
 ↓
/var/log/containers
 ↓
Fluent Bit DaemonSet
 ↓
Tail
 ↓
CRI Parser
 ↓
Kubernetes Filter
 ↓
JSON/Multiline
 ↓
Buffer
 ↓
TLS
 ↓
Dynatrace
```

And the three most important **Senior-level failure scenarios**:

```text
Dynatrace DOWN
    ↓
Buffer + Retry + Backpressure
```

```text
Java stack trace
    ↓
Multiline parser
```

```text
Sensitive data
    ↓
Filter/redaction
    ↓
Dynatrace
```

The next thing I would practice hands-on is **the exact Fluent Bit Helm `values.yaml` for EKS + DaemonSet + CRI parsing + Kubernetes metadata + multiline + filesystem buffering + Secret-based Dynatrace authentication + Dynatrace output**, because that is where the conceptual knowledge above turns into the production configuration an interviewer is likely to ask you to design.
