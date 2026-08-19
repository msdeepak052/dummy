# OpenTelemetry (OTel)

OpenTelemetry is a **vendor-neutral observability framework** used to generate, collect, process, and export **metrics, logs, and traces**.

For your DevOps interview, the easiest mental model is:

```text
Application
    ↓
OpenTelemetry instrumentation
    ↓
Telemetry
(metrics / traces / logs)
    ↓
OTel Collector
    ↓
Dynatrace / Prometheus / Jaeger / etc.
```

---

# 1. What is OpenTelemetry?

OpenTelemetry provides a common way to generate and collect:

```text
Traces  → What happened to a request?
Metrics → How much/how often?
Logs    → What happened?
```

The biggest advantage is **vendor neutrality**.

Without OTel:

```text
Application
    ↓
Dynatrace SDK
    ↓
Dynatrace
```

You become tightly coupled to Dynatrace.

With OTel:

```text
Application
    ↓
OpenTelemetry
    ↓
OTel Collector
    ↓
+------------+-------------+
|            |             |
Dynatrace   Jaeger      Prometheus
```

You can change the backend without completely changing application instrumentation.

---

# 2. Why is OpenTelemetry used?

Suppose you have:

```text
Frontend
   ↓
Payment API
   ↓
Order API
   ↓
PostgreSQL
```

A user makes:

```text
POST /payment
```

You want to know:

```text
Which service handled it?
How long did each service take?
Where did the request fail?
How many requests are coming?
What errors occurred?
```

OpenTelemetry gives you a standard way to generate that telemetry.

For example:

```text
Trace
  │
  ├── frontend       20ms
  │
  ├── payment-api    80ms
  │
  ├── order-api      40ms
  │
  └── PostgreSQL     30ms
```

---

# 3. Architecture

The architecture you should remember is:

```text
                    APPLICATION
                         │
             OpenTelemetry SDK/
             Auto Instrumentation
                         │
                         ▼
                  Telemetry Data
               ┌─────────┼─────────┐
               │         │         │
             Traces    Metrics    Logs
               │         │         │
               └─────────┼─────────┘
                         ▼
                 OTel Collector
                         │
              ┌──────────┼──────────┐
              │          │          │
              ▼          ▼          ▼
          Dynatrace   Prometheus   Jaeger
```

The **application generates telemetry**.

The **Collector receives/processes/exports telemetry**.

---

# 4. OTel Collector

This is the most important component from a Platform/DevOps perspective.

Think of the Collector as:

> **A centralized telemetry pipeline.**

Its pipeline is:

```text
RECEIVER
   ↓
PROCESSOR
   ↓
EXPORTER
```

Very similar to Fluent Bit:

```text
Fluent Bit:

INPUT
  ↓
FILTER
  ↓
OUTPUT


OTel Collector:

RECEIVER
  ↓
PROCESSOR
  ↓
EXPORTER
```

---

# 5. Receiver

Receiver answers:

> **Where does telemetry come from?**

Example:

```yaml
receivers:

  otlp:
    protocols:
      grpc:
      http:
```

This means:

```text
Application
    ↓
OTLP
    ↓
Collector Receiver
```

OTLP is the standard OpenTelemetry protocol.

It can transport:

```text
traces
metrics
logs
```

---

# 6. Processor

Processor answers:

> **What should I do with the telemetry before sending it?**

Example:

```yaml
processors:

  batch:

  memory_limiter:
    limit_mib: 512
```

Common processors:

```text
batch
memory_limiter
resource
attributes
filter
```

### `batch`

Instead of:

```text
send 1 record
send 1 record
send 1 record
```

it can batch:

```text
100 records
    ↓
send together
```

This improves efficiency.

### `memory_limiter`

Prevents the Collector from consuming unlimited memory.

---

# 7. Exporter

Exporter answers:

> **Where should the telemetry go?**

Example:

```yaml
exporters:

  otlp:
    endpoint: dynatrace-endpoint:443
```

Then:

```text
Application
    ↓
Receiver
    ↓
Processor
    ↓
Exporter
    ↓
Dynatrace
```

Other possible destinations include:

```text
Dynatrace
Prometheus
Jaeger
Kafka
OTel Collector
etc.
```

---

# 8. Complete Collector configuration

A simple configuration:

```yaml
receivers:

  otlp:
    protocols:
      grpc:
      http:


processors:

  memory_limiter:
    limit_mib: 512

  batch:


exporters:

  otlp:
    endpoint: <backend-endpoint>:443
    tls:
      insecure: false


service:

  pipelines:

    traces:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [otlp]

    metrics:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [otlp]

    logs:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [otlp]
```

The important part is:

```text
Receiver
   ↓
Processor
   ↓
Exporter
```

And you can have separate pipelines for:

```text
traces
metrics
logs
```

---

# 9. Trace — the most important concept

Suppose:

```text
User
 ↓
Frontend
 ↓
Payment API
 ↓
Database
```

One request creates a **trace**.

```text
Trace ID: abc123

Trace
│
├── Span: frontend
│      20ms
│
├── Span: payment-api
│      80ms
│
└── Span: PostgreSQL
       30ms
```

### Trace

Represents the **complete journey of a request**.

### Span

Represents **one operation inside that trace**.

Remember:

```text
Trace
  └── Spans
```

---

# 10. Trace example

User calls:

```text
GET /orders/123
```

Architecture:

```text
Frontend
   |
   | HTTP
   v
Order API
   |
   | SQL
   v
PostgreSQL
```

OTel creates:

```text
Trace ID = abc123

Span 1
Frontend
20ms

    ↓

Span 2
Order API
50ms

    ↓

Span 3
PostgreSQL
30ms
```

Now you can immediately identify:

```text
Total = 100ms
```

and see where the latency came from.

---

# 11. Context propagation

This is what connects spans together.

Without propagation:

```text
Frontend
  Trace A

Order API
  Trace B

Database
  Trace C
```

You can't easily connect them.

With OpenTelemetry context propagation:

```text
Frontend
  Trace ID = ABC
       ↓
Order API
  Trace ID = ABC
       ↓
Database
  Trace ID = ABC
```

So Dynatrace/your backend can show the entire request flow.

---

# 12. Metrics

OTel can also generate metrics.

Examples:

```text
http_requests_total
http_request_duration
cpu_usage
queue_size
```

Conceptually:

```text
Application
    ↓
OTel
    ↓
Metric
    ↓
Collector
    ↓
Prometheus/Dynatrace
```

Example:

```text
http_requests_total = 10000
```

---

# 13. Logs

OTel also supports logs.

```text
Application
    ↓
OTel
    ↓
Logs
    ↓
Collector
    ↓
Dynatrace
```

So you can have:

```text
Traces
Metrics
Logs
```

through one telemetry framework.

---

# 14. OTel vs Fluent Bit

This is important because you just learned Fluent Bit.

### Fluent Bit

Primarily:

```text
LOG COLLECTION
```

```text
Pod logs
   ↓
Fluent Bit
   ↓
Dynatrace
```

### OpenTelemetry

Broader observability:

```text
Metrics
Traces
Logs
```

```text
Application
   ↓
OpenTelemetry
   ↓
Collector
   ↓
Backend
```

So:

```text
Fluent Bit → primarily logs

OpenTelemetry → logs + metrics + traces
```

---

# 15. Full demo — Kubernetes

We'll create:

```text
Namespace
   ↓
OTel Collector
   ↓
Demo Application
```

Architecture:

```text
                 EKS

       ┌─────────────────────┐
       │     Demo App        │
       │                     │
       │ OTel SDK            │
       └─────────┬───────────┘
                 │
                 │ OTLP
                 ▼
       ┌─────────────────────┐
       │  OTel Collector     │
       │                     │
       │ Receiver            │
       │ Processor           │
       │ Exporter            │
       └─────────┬───────────┘
                 │
                 │ OTLP
                 ▼
             Dynatrace
```

---

# 16. Create namespace

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: observability
```

```bash
kubectl apply -f namespace.yaml
```

---

# 17. OTel Collector ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: otel-config
  namespace: observability

data:

  otel-collector.yaml: |

    receivers:

      otlp:
        protocols:
          grpc:
          http:


    processors:

      memory_limiter:
        limit_mib: 512

      batch:


    exporters:

      debug:


    service:

      pipelines:

        traces:
          receivers: [otlp]
          processors: [memory_limiter, batch]
          exporters: [debug]

        metrics:
          receivers: [otlp]
          processors: [memory_limiter, batch]
          exporters: [debug]

        logs:
          receivers: [otlp]
          processors: [memory_limiter, batch]
          exporters: [debug]
```

For this first demo, `debug` means:

> Don't send to Dynatrace yet; print the received telemetry so we can verify the pipeline.

---

# 18. Collector Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: otel-collector
  namespace: observability

spec:

  replicas: 1

  selector:
    matchLabels:
      app: otel-collector

  template:

    metadata:
      labels:
        app: otel-collector

    spec:

      containers:

        - name: otel-collector

          image: otel/opentelemetry-collector-contrib:latest

          args:
            - "--config=/etc/otel/otel-collector.yaml"

          ports:
            - containerPort: 4317
            - containerPort: 4318

          volumeMounts:
            - name: config
              mountPath: /etc/otel

      volumes:

        - name: config
          configMap:
            name: otel-config
```

---

# 19. Collector Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: otel-collector
  namespace: observability

spec:

  selector:
    app: otel-collector

  ports:

    - name: otlp-grpc
      port: 4317
      targetPort: 4317

    - name: otlp-http
      port: 4318
      targetPort: 4318
```

Now applications can send:

```text
otel-collector.observability.svc.cluster.local:4317
```

---

# 20. Application → Collector

Your application uses an OpenTelemetry SDK or auto-instrumentation.

Conceptually:

```text
Java application
      ↓
OTel Java Agent
      ↓
OTLP
      ↓
otel-collector:4317
```

For example:

```text
OTEL_SERVICE_NAME=payment-api

OTEL_EXPORTER_OTLP_ENDPOINT=
http://otel-collector.observability.svc.cluster.local:4317
```

Now the application knows:

```text
Where is my Collector?
```

It doesn't need to know about Dynatrace.

That's a major architectural benefit.

---

# 21. What happens when a request arrives?

Suppose:

```text
GET /payment
```

Application generates:

```text
Trace
 ↓
Span
 ↓
Metrics
```

Then:

```text
Application
    ↓
OTel SDK
    ↓
OTLP
    ↓
Collector
```

Collector:

```text
Receiver
   ↓
memory_limiter
   ↓
batch
   ↓
Exporter
```

Then:

```text
Dynatrace
```

---

# 22. Production architecture with Dynatrace

Replace the `debug` exporter with your Dynatrace-supported OTLP export configuration:

```text
Application
     │
     │ OTLP
     ▼
┌──────────────────┐
│ OTel Collector   │
│                  │
│ Receiver         │
│      ↓           │
│ Processor        │
│      ↓           │
│ Exporter         │
└────────┬─────────┘
         │
         │ OTLP/HTTPS
         ▼
    Dynatrace
```

Credentials should come from:

```text
Kubernetes Secret
```

rather than being hardcoded in the Collector configuration.

---

# 23. Why use a Collector instead of sending directly?

You **can** send telemetry directly:

```text
Application
     ↓
Dynatrace
```

But with Collector:

```text
Applications
     |
     v
OTel Collector
     |
     +---- Dynatrace
     +---- Jaeger
     +---- Prometheus
```

You get a central place for:

```text
batching
filtering
enrichment
memory limits
sampling
routing
export
```

This is particularly useful at platform scale.

---

# 24. OTel Collector deployment patterns

### Agent pattern

Collector runs close to workloads:

```text
Node
 |
 +-- Applications
 |
 +-- OTel Collector
```

Often deployed as:

```text
DaemonSet
```

### Gateway pattern

Centralized Collector:

```text
Applications
     |
     v
Collector Gateway
     |
     v
Dynatrace
```

You can also combine them:

```text
Application
    ↓
Node Collector
    ↓
Gateway Collector
    ↓
Dynatrace
```

For interviews, know:

> **Agent = close to workload. Gateway = centralized processing/routing.**

---

# 25. Important OTel terms to remember

| Term                     | Meaning                                                  |
| ------------------------ | -------------------------------------------------------- |
| **OTel SDK**             | Generates telemetry inside application                   |
| **Auto-instrumentation** | Automatically instruments supported libraries/frameworks |
| **OTLP**                 | Protocol used to transport OTel telemetry                |
| **Receiver**             | Receives telemetry                                       |
| **Processor**            | Modifies/batches/filters telemetry                       |
| **Exporter**             | Sends telemetry somewhere                                |
| **Collector**            | Runs receiver → processor → exporter pipeline            |
| **Trace**                | Complete request journey                                 |
| **Span**                 | One operation within a trace                             |
| **Context propagation**  | Connects spans across services                           |

---

# 26. The complete mental model

This is what I'd memorize for your interview:

```text
                 APPLICATION
                      │
              OTel SDK / Agent
                      │
          ┌───────────┼───────────┐
          │           │           │
        Trace       Metric       Log
          │           │           │
          └───────────┼───────────┘
                      │
                     OTLP
                      │
                      ▼
             ┌────────────────┐
             │ OTel Collector │
             │                │
             │   Receiver     │
             │       ↓        │
             │   Processor     │
             │       ↓        │
             │   Exporter     │
             └───────┬────────┘
                     │
                     ▼
                 Dynatrace
```

### One-line interview answer

> **"OpenTelemetry is a vendor-neutral observability framework for generating and collecting traces, metrics and logs. Applications use OTel SDKs or auto-instrumentation to generate telemetry and send it using OTLP to an OTel Collector. The Collector receives, processes, batches and exports that telemetry to backends such as Dynatrace, Prometheus or Jaeger."**
