# Fluent Bit

## 1. Explain Fluent Bit

**Fluent Bit is a lightweight log collector/shipper.**

In Kubernetes, it is commonly deployed as a **DaemonSet**, so one Fluent Bit pod runs on each node and collects logs from that node.

```text
Application
    ↓
stdout / stderr
    ↓
/var/log/containers/*.log
    ↓
Fluent Bit
    ↓
Dynatrace
```

It mainly does:

```text
Collect → Parse → Enrich/Filter → Buffer → Forward
```



---

# 2. Why is Fluent Bit used?

Without Fluent Bit, every application would need its own logging integration.

Instead:

```text
                 EKS
      +----------+----------+
      |          |          |
     Pod        Pod        Pod
      |          |          |
      +----------+----------+
                 |
          Node log files
                 |
                 v
           Fluent Bit
                 |
        +--------+--------+
        |                 |
    Dynatrace          S3/etc.
```

### Why DaemonSet?

Logs are stored on the **node**, so we run one Fluent Bit per node to collect them.

```text
Node 1 → Fluent Bit
Node 2 → Fluent Bit
Node 3 → Fluent Bit
```



---

# 3. Architecture

```text
                     EKS CLUSTER

 Node 1                         Node 2
┌──────────────────┐           ┌──────────────────┐
│ App Pod          │           │ App Pod          │
│ stdout/stderr    │           │ stdout/stderr    │
└────────┬─────────┘           └────────┬─────────┘
         ↓                              ↓
 /var/log/containers/*.log      /var/log/containers/*.log
         ↓                              ↓
┌──────────────────┐           ┌──────────────────┐
│ Fluent Bit       │           │ Fluent Bit       │
│ DaemonSet        │           │ DaemonSet        │
│                  │           │                  │
│ Tail             │           │ Tail             │
│ Parser           │           │ Parser           │
│ K8s Filter       │           │ K8s Filter       │
│ Buffer            │           │ Buffer            │
└────────┬─────────┘           └────────┬─────────┘
         │                              │
         └──────────────┬───────────────┘
                        ↓
                    Dynatrace
```

The core Fluent Bit pipeline is:

```text
INPUT
  ↓
PARSER
  ↓
FILTER
  ↓
BUFFER
  ↓
OUTPUT
```



---

Yes. The important thing is to understand **what happens to one log record as it travels through Fluent Bit**. Once that is clear, `INPUT`, `PARSER`, `FILTER`, `BUFFER`, `OUTPUT`, `MULTILINE`, `GREP/Regex`, etc. become easy.

# 4. Fluent Bit Configuration — Detailed but Easy

Think of Fluent Bit as a pipeline:

```text
Application
     │
     │ stdout/stderr
     ▼
┌──────────────┐
│    INPUT     │  ← Where do I get logs?
│     Tail     │
└──────┬───────┘
       ▼
┌──────────────┐
│    PARSER    │  ← What does this log mean?
│  CRI / JSON  │
└──────┬───────┘
       ▼
┌──────────────┐
│    FILTER    │  ← Modify/enrich/filter
│ Kubernetes   │
└──────┬───────┘
       ▼
┌──────────────┐
│    BUFFER    │  ← Temporarily hold
└──────┬───────┘
       ▼
┌──────────────┐
│    OUTPUT    │  ← Where should it go?
│  Dynatrace   │
└──────────────┘
```

The core terms and their roles are also summarized in your source. 

---

# 1. `[SERVICE]`

`SERVICE` contains **global settings for Fluent Bit itself**.

```ini
[SERVICE]
    Flush        5
    Log_Level    info
```

It does **not** define where logs come from or where they go.

Think:

> "How should Fluent Bit itself operate?"

---

## `Flush 5`

```ini
Flush 5
```

Means Fluent Bit attempts to flush/process pending records every **5 seconds**.

Conceptually:

```text
Logs arrive
   ↓
Fluent Bit
   ↓
every 5 sec
   ↓
send pending records
```

If you make it very small:

```text
Flush 1
```

you can get more frequent sending, but potentially more overhead.

If you make it larger:

```text
Flush 30
```

logs may wait longer before being sent.

For an interview:

> **Flush controls the interval at which Fluent Bit flushes data to outputs.**

---

## `Log_Level`

```ini
Log_Level info
```

Controls **Fluent Bit's own logs**, not your application's logs.

Typical levels:

```text
error
warn
info
debug
trace
```

For example:

```ini
Log_Level debug
```

is useful while troubleshooting.

You might see:

```text
[debug] [input:tail:tail.0]
[debug] [filter:kubernetes]
[debug] [output:http]
```

But you normally don't want excessive debug logging permanently.

---

# 2. `[INPUT]`

This answers:

> **Where should Fluent Bit collect logs from?**

For Kubernetes:

```ini
[INPUT]
    Name              tail
    Path              /var/log/containers/*.log
    Parser            cri
    Tag               kube.*
    Mem_Buf_Limit     50MB
    Skip_Long_Lines   On
```

The source specifically uses `tail` for Kubernetes container log files. 

---

# 3. `Name tail`

```ini
Name tail
```

`tail` means:

> Read and continuously follow a file.

Similar to:

```bash
tail -f application.log
```

For example:

```text
application writes:
ERROR payment failed
```

Fluent Bit is continuously watching the file and picks up the new line.

---

# 4. `Path`

```ini
Path /var/log/containers/*.log
```

This tells Fluent Bit **which files to read**.

`*` means:

> Match multiple files.

So:

```text
/var/log/containers/*.log
```

could match:

```text
payment-api-xxx.log
order-api-xxx.log
frontend-xxx.log
auth-api-xxx.log
```

Architecture:

```text
Node
│
├── /var/log/containers/
│      ├── payment-api.log
│      ├── order-api.log
│      └── frontend.log
│
└── Fluent Bit
       │
       └── Tail reads these files
```

---

# 5. `Parser cri`

```ini
Parser cri
```

This is extremely important in Kubernetes.

The container runtime can produce something like:

```text
2026-08-19T10:30:15.123Z stdout F {"level":"INFO","message":"Payment successful"}
```

That's not simply:

```json
{"level":"INFO"}
```

There is container-runtime information around the application log.

The `cri` parser understands the **Container Runtime Interface log format**.

Conceptually:

```text
Raw container log
       ↓
CRI Parser
       ↓
timestamp
stream = stdout
flag
actual log message
```

Then Fluent Bit can process the actual application payload.

---

# 6. `Tag`

```ini
Tag kube.*
```

A **tag is an internal label used for routing records through Fluent Bit**.

Think:

```text
Record
  |
  +-- Tag = kube.xxx
```

Then:

```ini
Match kube.*
```

means:

> Process records whose tag matches `kube.*`.

For example:

```text
INPUT
Tag = kube.*
       ↓
FILTER
Match = kube.*
       ↓
OUTPUT
Match = kube.*
```

So the same Kubernetes logs travel through the pipeline.

Your source describes tags as an internal routing/classification identifier. 

---

# 7. `Mem_Buf_Limit`

```ini
Mem_Buf_Limit 50MB
```

This controls how much **memory buffering** the input can use.

Imagine:

```text
Application
  ↓
Fluent Bit
  ↓
Dynatrace is slow
```

Logs start accumulating.

```text
Log
Log
Log
Log
Log
 ↓
Memory buffer
```

You don't want Fluent Bit to consume unlimited RAM.

So:

```ini
Mem_Buf_Limit 50MB
```

puts a limit on that memory buffer.

---

# 8. `Skip_Long_Lines`

```ini
Skip_Long_Lines On
```

Suppose an application produces an extremely large log line:

```text
10 MB single log line
```

Instead of allowing a huge line to cause problems, Fluent Bit can skip lines that exceed the configured limits.

For interviews:

> **Skip_Long_Lines prevents oversized log lines from causing input processing problems.**

---

# 9. `[PARSER]`

Parser answers:

> **How do I interpret the raw log?**

Example JSON:

```json
{
  "timestamp": "2026-08-19T10:30:00Z",
  "level": "ERROR",
  "message": "Database failed"
}
```

A JSON parser can turn this into structured fields:

```text
timestamp = ...
level     = ERROR
message   = Database failed
```

Example:

```ini
[PARSER]
    Name        app_json
    Format      json
    Time_Key    timestamp
    Time_Format %Y-%m-%dT%H:%M:%SZ
```

Your source describes parsers as converting raw log text into structured fields. 

---

# 10. CRI Parser vs JSON Parser

This confuses people.

They can be used for **different layers**.

Suppose the actual file contains:

```text
2026-08-19T10:30:00Z stdout F {"level":"ERROR","message":"DB failed"}
```

First:

```text
CRI parser
     ↓
{"level":"ERROR","message":"DB failed"}
```

Then the application payload can be parsed as JSON.

Conceptually:

```text
Container log
     ↓
CRI Parser
     ↓
Application JSON
     ↓
JSON Parser
     ↓
Structured fields
```

---

# 11. `[FILTER]`

Filter answers:

> **Now that I have the record, what should I do with it?**

Filters can:

```text
add fields
remove fields
modify fields
rename fields
drop records
enrich records
```

Your source lists these filter capabilities. 

---

# 12. Kubernetes Filter

Example:

```ini
[FILTER]
    Name       kubernetes
    Match      kube.*
    Merge_Log  On
    Keep_Log   Off
```

This is very important for Kubernetes.

Suppose application sends:

```json
{
  "level": "ERROR",
  "message": "Payment failed"
}
```

Fluent Bit can enrich it with Kubernetes information:

```text
namespace = payment
pod        = payment-api-abc
container  = payment
labels     = app=payment
```

So the record becomes conceptually:

```json
{
  "level": "ERROR",
  "message": "Payment failed",

  "kubernetes": {
    "namespace": "payment",
    "pod": "payment-api-abc",
    "container": "payment"
  }
}
```

---

# 13. `Merge_Log On`

Suppose the application sends:

```json
{"level":"ERROR","message":"Database failed"}
```

Without merging, you might effectively have:

```text
log = '{"level":"ERROR","message":"Database failed"}'
```

With:

```ini
Merge_Log On
```

the JSON fields can be merged into the record:

```text
level   = ERROR
message = Database failed
```

Your source demonstrates this exact conceptual difference. 

---

# 14. `Keep_Log Off`

```ini
Keep_Log Off
```

If Fluent Bit successfully extracts the JSON fields from the original `log` field, this tells it not to unnecessarily keep the original log field.

Conceptually:

### Before

```text
log = '{"level":"ERROR","message":"DB failed"}'
```

### After

```text
level = ERROR
message = DB failed
```

This avoids duplicate data.

---

# 15. `GREP` / Regex Filter

This is one of the other important settings you mentioned.

Suppose logs are:

```text
INFO Payment successful
WARN Payment retry
ERROR Database failed
```

You only want ERROR logs.

You can use a grep filter:

```ini
[FILTER]
    Name   grep
    Match  kube.*
    Regex  level ERROR
```

Conceptually:

```text
INFO
 ↓
DROP

WARN
 ↓
DROP

ERROR
 ↓
KEEP
```

Your source gives the same pattern. 

### But be careful

In production, don't blindly filter logs just to reduce volume.

You might accidentally remove information needed during troubleshooting.

---

# 16. `Modify` Filter

Another common filter:

```ini
[FILTER]
    Name   modify
    Match  kube.*
    Add    environment production
```

Now every matching record gets:

```text
environment=production
```

Useful for adding:

```text
environment
team
region
application
cluster
```

Your source shows this pattern. 

---

# 17. MULTILINE

This is **very important** for Java/Python/.NET stack traces.

Suppose Java produces:

```text
Exception: Database connection failed
    at PaymentService.process(PaymentService.java:50)
    at PaymentService.handle(PaymentService.java:30)
    at Controller.handle(Controller.java:20)
```

Without multiline:

```text
Log 1 → Exception...
Log 2 → at PaymentService...
Log 3 → at PaymentService...
Log 4 → at Controller...
```

That's bad.

With multiline:

```text
             ONE LOG EVENT
                  │
        ┌─────────┴─────────┐
        │                   │
 Exception               stack trace
                           │
                 ├── at PaymentService
                 ├── at PaymentService
                 └── at Controller
```

Fluent Bit's multiline processing combines related lines into a single event. 

### Typical use cases

```text
Java exceptions
Python tracebacks
Go panic
.NET stack traces
```

---

# 18. BUFFER

Buffer answers:

> **What do I do with logs when the destination can't keep up?**

Normal:

```text
Application
    ↓
Fluent Bit
    ↓
Dynatrace
```

Dynatrace becomes unavailable:

```text
Application
    ↓
Fluent Bit
    ↓
BUFFER
    ↓
Dynatrace ❌
```

Later:

```text
Dynatrace becomes available
          ↓
      BUFFER
          ↓
      Dynatrace
```

Fluent Bit supports memory and filesystem buffering. 

---

# 19. Memory vs Filesystem Buffer

### Memory

```text
Fluent Bit
    ↓
RAM
    ↓
Dynatrace
```

Fast, but if the Fluent Bit process/node fails, buffered data can be lost.

### Filesystem

```text
Fluent Bit
    ↓
Disk
    ↓
Dynatrace
```

More resilient for temporary outages/restarts, assuming sufficient disk capacity.

For production, filesystem buffering is often considered when you need stronger durability.

---

# 20. `[OUTPUT]`

Output answers:

> **Where should Fluent Bit send the logs?**

Example:

```ini
[OUTPUT]
    Name    http
    Match   kube.*
    Host    <dynatrace-endpoint>
    Port    443
    URI     <log-ingest-path>
    Format  json
    tls     On
```

---

## `Name`

```ini
Name http
```

Use the HTTP output plugin.

Other destinations/plugins can include systems such as:

```text
Elasticsearch
S3
Kafka
HTTP endpoints
```

depending on the supported Fluent Bit output.

---

## `Match`

```ini
Match kube.*
```

Only records with matching tags are sent.

Remember:

```text
INPUT:
Tag kube.*

        ↓

FILTER:
Match kube.*

        ↓

OUTPUT:
Match kube.*
```

---

## `Host`

```ini
Host <dynatrace-endpoint>
```

Destination hostname.

---

## `Port`

```ini
Port 443
```

HTTPS:

```text
443
```

---

## `URI`

```ini
URI <log-ingest-path>
```

The API path where logs are submitted.

The exact Dynatrace endpoint depends on the ingestion method you're using.

---

## `Format`

```ini
Format json
```

Send records as JSON.

---

## `tls On`

```ini
tls On
```

Enables TLS encryption.

So:

```text
Fluent Bit
     |
     | HTTPS/TLS
     v
Dynatrace
```

---

# 21. Authentication

Don't do:

```ini
Header Authorization Bearer my-real-token
```

inside Git.

Instead:

```text
Kubernetes Secret
       ↓
Fluent Bit
       ↓
Dynatrace
```

For example:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: dynatrace-token
type: Opaque
stringData:
  DT_API_TOKEN: "REPLACE_ME"
```

Then inject the secret into the Fluent Bit pod.

Your source specifically recommends Secret/centralized secret management instead of hardcoding tokens. 

---

# 22. Putting everything together

Now the entire configuration makes sense:

```text
[SERVICE]
    ↓
How Fluent Bit behaves

[INPUT]
    ↓
Where logs come from
    ↓
Tail /var/log/containers/*.log

[PARSER]
    ↓
Understand log format
    ↓
CRI / JSON

[FILTER]
    ↓
Modify/enrich/filter
    ↓
Kubernetes metadata
    ↓
Merge JSON
    ↓
Grep / Regex
    ↓
Modify

[MULTILINE]
    ↓
Combine stack traces

[BUFFER]
    ↓
Hold logs during destination problems

[OUTPUT]
    ↓
Where logs go
    ↓
Dynatrace
```

---

# 23. The settings you should know for interviews

Don't try to memorize every Fluent Bit option. Know these:

| Area           | Important settings/concepts                                         |
| -------------- | ------------------------------------------------------------------- |
| **SERVICE**    | `Flush`, `Log_Level`                                                |
| **INPUT**      | `Name`, `Path`, `Parser`, `Tag`, `Mem_Buf_Limit`, `Skip_Long_Lines` |
| **PARSER**     | JSON, CRI, Regex, time parsing                                      |
| **FILTER**     | Kubernetes, Modify, Grep                                            |
| **Kubernetes** | `Match`, `Merge_Log`, `Keep_Log`                                    |
| **MULTILINE**  | Stack traces                                                        |
| **BUFFER**     | Memory vs filesystem, retry/backpressure                            |
| **OUTPUT**     | `Name`, `Match`, `Host`, `Port`, `URI`, `Format`, TLS               |
| **Security**   | Secrets, TLS, don't hardcode tokens                                 |

---

# 24. One example to remember

Suppose your Java application produces:

```text
ERROR Database failed
    at PaymentService.java:50
    at Controller.java:20
```

Fluent Bit does:

```text
1. INPUT
   Tail
   ↓
2. PARSER
   CRI
   ↓
3. MULTILINE
   Combine 3 lines → 1 event
   ↓
4. FILTER
   Kubernetes metadata
   ↓
5. FILTER
   JSON / Modify / Regex if required
   ↓
6. BUFFER
   Hold temporarily
   ↓
7. OUTPUT
   HTTPS → Dynatrace
```

That's the **core Fluent Bit pipeline** you should have in your head.

---

# 5. Full demo — End to End

We'll create:

```text
logging namespace
 ├── Secret
 ├── ConfigMap
 └── Fluent Bit DaemonSet

demo namespace
 └── Application
```

Flow:

```text
Demo App
   ↓
stdout
   ↓
Container runtime
   ↓
/var/log/containers/*.log
   ↓
Fluent Bit
   ↓
Tail → CRI Parser → K8s Filter → Buffer
   ↓
Dynatrace
```



---

## Step 1 — Namespaces

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: logging
---
apiVersion: v1
kind: Namespace
metadata:
  name: demo
```

```bash
kubectl apply -f namespaces.yaml
```

---

## Step 2 — Dynatrace Secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: dynatrace-logs
  namespace: logging
type: Opaque
stringData:
  DT_API_TOKEN: "REPLACE_WITH_REAL_TOKEN"
```

**Never commit the real token to Git.**

In production, use your organization's secret-management solution. 

---

## Step 3 — Fluent Bit ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluent-bit-config
  namespace: logging
data:

  fluent-bit.conf: |
    [SERVICE]
        Flush        5
        Log_Level    info

    [INPUT]
        Name              tail
        Path              /var/log/containers/*.log
        Parser            cri
        Tag               kube.*
        Mem_Buf_Limit     50MB
        Skip_Long_Lines   On

    [FILTER]
        Name                kubernetes
        Match               kube.*
        Merge_Log           On
        Keep_Log            Off

    [OUTPUT]
        Name        http
        Match       kube.*
        Host        <dynatrace-endpoint>
        Port        443
        URI         <log-ingest-path>
        Format      json
        tls         On

  parsers.conf: |
    [PARSER]
        Name        app_json
        Format      json
```

The important part to understand for the interview is:

```text
Tail
 ↓
reads /var/log/containers/*.log

CRI
 ↓
understands Kubernetes container log format

Kubernetes filter
 ↓
adds pod/namespace/container metadata

HTTP output
 ↓
sends logs to Dynatrace
```

The exact Dynatrace endpoint/output configuration depends on the ingestion method used in your environment. 

---

## Step 4 — Fluent Bit DaemonSet

```yaml
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

      serviceAccountName: fluent-bit

      containers:
        - name: fluent-bit
          image: fluent/fluent-bit:latest

          volumeMounts:

            - name: varlog
              mountPath: /var/log

            - name: config
              mountPath: /fluent-bit/etc

      volumes:

        - name: varlog
          hostPath:
            path: /var/log

        - name: config
          configMap:
            name: fluent-bit-config
```

The important piece is:

```yaml
- name: varlog
  hostPath:
    path: /var/log
```

This gives Fluent Bit access to the node's log files. 

---

## Step 5 — RBAC

Fluent Bit needs permission to query Kubernetes metadata.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: fluent-bit
  namespace: logging
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: fluent-bit
rules:
  - apiGroups: [""]
    resources:
      - pods
      - namespaces
    verbs:
      - get
      - list
      - watch
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: fluent-bit
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: fluent-bit
subjects:
  - kind: ServiceAccount
    name: fluent-bit
    namespace: logging
```

Flow:

```text
Fluent Bit
    ↓
ServiceAccount
    ↓
RBAC
    ↓
Kubernetes API
    ↓
Pod metadata
```



---

## Step 6 — Deploy Fluent Bit

```bash
kubectl apply -f rbac.yaml
kubectl apply -f fluent-bit-config.yaml
kubectl apply -f fluent-bit-daemonset.yaml
```

Check:

```bash
kubectl get pods -n logging -o wide
```

If you have:

```text
3 nodes
```

you should see approximately:

```text
fluent-bit-xxxxx   Running   node1
fluent-bit-yyyyy   Running   node2
fluent-bit-zzzzz   Running   node3
```

Because it is a **DaemonSet**.

---

## Step 7 — Create demo application

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

```bash
kubectl apply -f app.yaml
```

---

## Step 8 — Verify application logs

```bash
kubectl get pods -n demo
```

Then:

```bash
kubectl logs -n demo deployment/log-demo
```

You should see:

```json
{"level":"INFO","message":"Payment processed","request_id":"req-0","service":"payment-api"}
{"level":"INFO","message":"Payment processed","request_id":"req-1","service":"payment-api"}
```

The application simply writes to **stdout**; it doesn't know anything about Fluent Bit. 

---

## Step 9 — What happens internally?

```text
Application
    ↓
stdout
    ↓
Container runtime
    ↓
/var/log/containers/*.log
    ↓
Fluent Bit Tail
    ↓
CRI Parser
    ↓
Kubernetes Filter
    ↓
JSON processing
    ↓
Buffer
    ↓
Dynatrace
```



The final record can contain both application data and Kubernetes metadata:

```json
{
  "level": "INFO",
  "message": "Payment processed",
  "request_id": "req-123",
  "service": "payment-api",

  "kubernetes": {
    "namespace_name": "demo",
    "pod_name": "log-demo-xxxxx",
    "container_name": "app",
    "node_name": "worker-node"
  }
}
```

---

## Step 10 — Troubleshooting

If logs don't reach Dynatrace, follow this exact chain:

```text
1. kubectl logs
       ↓
2. /var/log/containers
       ↓
3. Fluent Bit pod logs
       ↓
4. INPUT / Tail
       ↓
5. Parser
       ↓
6. Kubernetes Filter / RBAC
       ↓
7. Buffer / Retry
       ↓
8. OUTPUT / TLS / Authentication
       ↓
9. Dynatrace ingestion
```

Useful commands:

```bash
kubectl get pods -n logging

kubectl logs -n logging daemonset/fluent-bit

kubectl get configmap -n logging

kubectl describe daemonset -n logging fluent-bit

kubectl logs -n demo deployment/log-demo
```

---

## Interview mental model

Just remember this:

```text
          FLUENT BIT

Application
     ↓
stdout/stderr
     ↓
/var/log/containers
     ↓
   INPUT
   Tail
     ↓
  PARSER
  CRI/JSON
     ↓
  FILTER
  Kubernetes
     ↓
  BUFFER
  memory/disk
     ↓
  OUTPUT
  Dynatrace
```

**One-line interview answer:**

> **"I run Fluent Bit as a DaemonSet on each Kubernetes node. It tails container logs from `/var/log/containers`, parses the CRI format, enriches them with Kubernetes metadata using the Kubernetes filter, buffers them for reliability, and forwards them securely to Dynatrace."**
