# KEDA — Kubernetes Event-Driven Autoscaling

**KEDA (Kubernetes Event-driven Autoscaling)** is a Kubernetes autoscaler that scales workloads based on **external/event-driven metrics**, such as:

* Kafka lag
* AWS SQS queue length
* RabbitMQ messages
* Azure Service Bus
* Prometheus metrics
* Cron schedules
* Redis lists
* PostgreSQL queries

The key idea:

> **HPA is good at CPU/memory and Kubernetes metrics; KEDA is especially useful when scaling should depend on an event source such as queue depth or Kafka lag.**

---

# 1. Architecture

Example: scale workers based on an **AWS SQS queue**.

```text
                 AWS SQS
                    |
              messages = 5000
                    |
                    v
              +-----------+
              |   KEDA    |
              | Operator  |
              +-----+-----+
                    |
             calculates desired
                replicas
                    |
                    v
              +-----------+
              |    HPA    |
              +-----+-----+
                    |
                    v
              +-----------+
              | Deployment|
              |  workers  |
              +-----+-----+
                    |
             +------+------+ 
             |      |      |
            Pod    Pod    Pod
```

KEDA's operator watches the event source and works with Kubernetes HPA to scale the target workload.

---

# 2. Why KEDA?

Suppose you have:

```text
order-worker
```

and it processes messages from:

```text
AWS SQS
```

You have:

```text
Queue = 10,000 messages
```

CPU might still be:

```text
CPU = 20%
```

So a CPU-based HPA might say:

```text
CPU is low
→ don't scale
```

But KEDA says:

```text
Queue = 10,000
→ we need more workers
```

That's the important use case.

---

# 3. KEDA components

You mainly need to know these:

```text
KEDA Operator
KEDA Metrics Server
ScaledObject
Scaler
```

Architecture:

```text
             Event Source
          SQS / Kafka / Redis
                  |
                  v
             KEDA Scaler
                  |
                  v
           KEDA Operator
                  |
                  v
                 HPA
                  |
                  v
             Deployment
```

### KEDA Operator

Watches `ScaledObject` resources and manages autoscaling.

### Scaler

Connects to the external system.

Examples:

```text
SQS scaler
Kafka scaler
Prometheus scaler
Redis scaler
```

### KEDA Metrics Server

Exposes metrics to Kubernetes so HPA can consume them.

---

# 4. ScaledObject

This is the main KEDA CRD you should know.

Example:

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject

metadata:
  name: order-worker

spec:
  scaleTargetRef:
    name: order-worker

  minReplicaCount: 1
  maxReplicaCount: 10

  triggers:

    - type: aws-sqs-queue

      metadata:
        queueURL: https://sqs.us-east-1.amazonaws.com/123/order-queue
        queueLength: "20"
        awsRegion: us-east-1
```

Meaning:

```text
1 worker minimum
10 workers maximum

For every ~20 messages
→ scale workers
```

The exact authentication configuration should use your AWS identity/secret approach rather than putting credentials directly into the YAML.

---

# 5. Complete workflow

Suppose:

```text
SQS queue
   |
   | 1000 messages
   v
KEDA
```

KEDA checks the queue:

```text
Current queue = 1000
Target = 20 messages/pod
```

Desired replicas are conceptually:

```text
1000 / 20
= 50
```

But:

```text
maxReplicaCount = 10
```

so:

```text
Desired = 10
```

Then:

```text
KEDA
 ↓
HPA
 ↓
Deployment
 ↓
10 Pods
```

---

# 6. What happens when the queue decreases?

Suppose:

```text
Queue = 1000
```

then:

```text
10 Pods
```

Workers consume the messages:

```text
1000
 ↓
500
 ↓
200
 ↓
50
 ↓
10
 ↓
0
```

KEDA/HPA scales the workers down.

Eventually:

```text
10 Pods
 ↓
3 Pods
 ↓
1 Pod
```

depending on your configured minimum.

---

# 7. KEDA vs HPA

This is the important interview comparison.

### Normal HPA

```text
CPU / Memory
     |
     v
    HPA
     |
     v
Deployment
```

Example:

```yaml
metrics:
- type: Resource
  resource:
    name: cpu
```

---

### KEDA

```text
Kafka / SQS / Redis / Prometheus
             |
             v
            KEDA
             |
             v
            HPA
             |
             v
        Deployment
```

So:

> **KEDA doesn't replace HPA. It extends event-driven autoscaling and integrates with HPA.**

---

# 8. Kafka example

This is one of the most common real-world examples.

Suppose:

```text
Kafka
 |
 +-- orders topic
       |
       +-- consumer group
```

Current lag:

```text
Kafka lag = 5000
```

KEDA:

```text
Kafka lag
    |
    v
KEDA Kafka Scaler
    |
    v
HPA
    |
    v
order-consumer Deployment
```

Example:

```yaml
triggers:
  - type: kafka
    metadata:
      bootstrapServers: kafka:9092
      consumerGroup: order-consumer
      topic: orders
      lagThreshold: "100"
```

Meaning approximately:

> Try to maintain Kafka lag around 100 per scaling unit.

---

# 9. KEDA can scale to zero

This is one of KEDA's biggest advantages.

Normal HPA typically operates with a minimum replica count greater than zero.

KEDA can do:

```text
Queue = 0
     ↓
0 Pods
```

Then:

```text
New message arrives
     ↓
KEDA detects event
     ↓
Scale from 0
     ↓
Pod starts
     ↓
Process message
```

Architecture:

```text
             Queue
               |
          message arrives
               |
               v
             KEDA
               |
               v
          0 → 1 Pod
```

This is very useful for **worker/batch/event-driven workloads**.

---

# 10. Prometheus + KEDA

KEDA can also scale based on a Prometheus metric.

For example:

```text
Prometheus
    |
    | queue_depth
    v
   KEDA
    |
    v
   HPA
    |
    v
Workers
```

Example:

```yaml
triggers:
  - type: prometheus

    metadata:
      serverAddress: http://prometheus.monitoring:9090

      query: |
        sum(queue_depth)

      threshold: "100"
```

So you can use:

```text
Prometheus metric
        ↓
KEDA
        ↓
HPA
        ↓
Pods
```

---

# 11. KEDA vs Prometheus Adapter

Since you just learned Prometheus Adapter, this distinction is important.

### Prometheus Adapter

```text
Prometheus
    ↓
Prometheus Adapter
    ↓
Custom Metrics API
    ↓
HPA
```

Used when you want HPA to consume Prometheus metrics through Kubernetes custom metrics APIs.

### KEDA

```text
Prometheus / Kafka / SQS / Redis / etc.
                ↓
               KEDA
                ↓
               HPA
```

KEDA supports many event sources directly.

So if you have:

```text
Kafka
SQS
RabbitMQ
```

KEDA is usually much more natural than building a Prometheus-based solution just to expose queue metrics to HPA.

---

# 12. KEDA architecture to memorize

```text
                 EVENT SOURCE
          +--------+--------+
          |        |        |
         SQS     Kafka    Redis
          |        |        |
          +--------+--------+
                   |
                   v
             +-----------+
             |    KEDA   |
             |  Scalers  |
             +-----+-----+
                   |
                   v
             +-----------+
             |    HPA    |
             +-----+-----+
                   |
                   v
             +-----------+
             |Deployment |
             +-----+-----+
                   |
              +----+----+
              |         |
             Pod       Pod
```

### Remember:

**KEDA = Event → Scale**

Examples:

```text
SQS messages → workers
Kafka lag → consumers
Redis queue → workers
Prometheus metric → application
Cron → scheduled workload
```

For your interview, the most important KEDA topics are **ScaledObject, Scalers, scale-to-zero, Kafka lag/SQS queue examples, and KEDA vs HPA vs Prometheus Adapter**.


A **ServiceMonitor** is a Kubernetes Custom Resource Definition (CRD) introduced by the **Prometheus Operator**. Instead of manually writing scrape configurations in a static config file, Prometheus Operator watches for `ServiceMonitor` resources, matches target Kubernetes `Services` using label selectors, and automatically reloads Prometheus with dynamic scrape jobs targeting the underlying `Endpoints`.

---

### How the Scrape Discovery Pipeline Works

```
+-------------------------------------------------------------------------+
| 1. Prometheus CRD                                                       |
|    serviceMonitorSelector: { release: "prometheus-stack" }              |
+------------------------------------+------------------------------------+
                                     | (Matches labels on ServiceMonitor)
                                     v
+-------------------------------------------------------------------------+
| 2. ServiceMonitor CRD                                                   |
|    labels: { release: "prometheus-stack" }                              |
|    spec.selector.matchLabels: { app: "payment-api" }                    |
|    spec.endpoints: [ { port: "metrics", path: "/metrics" } ]            |
+------------------------------------+------------------------------------+
                                     | (Matches labels on K8s Service)
                                     v
+-------------------------------------------------------------------------+
| 3. Kubernetes Service (SVC)                                             |
|    metadata.labels: { app: "payment-api" }                              |
|    spec.ports: [ { name: "metrics", port: 8080 } ]                      |
|    spec.selector: { app: "payment-pod" }                                |
+------------------------------------+------------------------------------+
                                     | (Resolves Endpoints via Pod Selector)
                                     v
+-------------------------------------------------------------------------+
| 4. Running Pod(s) & Endpoints                                           |
|    Pod IP: 10.244.1.15:8080/metrics  <--- Prometheus Scrapes Directly   |
|    Pod IP: 10.244.2.22:8080/metrics  <--- Prometheus Scrapes Directly   |
+-------------------------------------------------------------------------+

```

---

### Complete YAML Implementation

**1. The Target Application Deployment & Service**
The Service must define a named port that matches the endpoint definition in the ServiceMonitor.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-api
  namespace: production
  labels:
    app: payment-api
spec:
  replicas: 2
  selector:
    matchLabels:
      app: payment-api
  template:
    metadata:
      labels:
        app: payment-api
    spec:
      containers:
        - name: web
          image: my-registry/payment-api:v1.0
          ports:
            - name: http-metrics
              containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: payment-api-svc
  namespace: production
  labels:
    app: payment-api
    monitor: "true"
spec:
  type: ClusterIP
  selector:
    app: payment-api
  ports:
    - name: http-metrics
      port: 8080
      targetPort: http-metrics

```

**2. The ServiceMonitor CRD**
Matches the service by label (`monitor: "true"`) and instructs Prometheus on the path, scrape interval, and port name.

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: payment-api-monitor
  namespace: production
  labels:
    release: prometheus-stack # Must match Prometheus spec.serviceMonitorSelector
spec:
  namespaceSelector:
    matchNames:
      - production
  selector:
    matchLabels:
      monitor: "true" # Matches Service metadata.labels
  endpoints:
    - port: http-metrics # Refers to the named port in the Service
      path: /metrics
      interval: 15s
      scrapeTimeout: 10s

```

**3. The Prometheus Instance Configuration**
Ensure the Prometheus instance selector matches the labels on your `ServiceMonitor`.

```yaml
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  name: k8s-prometheus
  namespace: monitoring
spec:
  serviceAccountName: prometheus-k8s
  serviceMonitorSelector:
    matchLabels:
      release: prometheus-stack # Triggers discovery of the ServiceMonitor
  serviceMonitorNamespaceSelector: {} # Allows scraping ServiceMonitors across all namespaces

```

---

### Troubleshooting Scrape Trigger Failures

* **Port Name Mismatch:** `spec.endpoints[].port` in the `ServiceMonitor` references the **name** of the port on the Kubernetes `Service` object (`http-metrics`), not the raw port number or container port name.
* **Selector Mismatch:** The `ServiceMonitor.spec.selector` matches against the **Service's `metadata.labels**`, not the Deployment's labels or Pod labels.
* **Namespace Isolation:** If the Service and Prometheus are in different namespaces, ensure `serviceMonitorNamespaceSelector: {}` is set on the `Prometheus` resource, or define explicit `namespaceSelector` rules in the `ServiceMonitor`.
* **No Healthy Endpoints:** A ServiceMonitor extracts targets from the Kubernetes `Endpoints` / `EndpointSlices` resource created by the Service. If no Pods pass their readiness probes, no scrape targets will be registered.
