# Prometheus
For a **Senior DevOps / Platform Engineer interview**, I would learn Prometheus in this order:

```text
Prometheus fundamentals
        ↓
Metrics model
        ↓
Architecture & components
        ↓
Scraping / Service Discovery
        ↓
Exporters & instrumentation
        ↓
PromQL
        ↓
Recording rules
        ↓
Alerting + Alertmanager
        ↓
Kubernetes monitoring
        ↓
Prometheus Operator / ServiceMonitor
        ↓
Prometheus Adapter
        ↓
HPA with custom metrics
        ↓
Scaling / HA / Remote Write / Thanos
```

I'll cover the whole chain and then build a **complete Kubernetes + Prometheus + Prometheus Adapter + HPA demo**.

---

# 1. What is Prometheus?

**Prometheus is a monitoring and alerting system that collects numerical metrics, stores them as time-series data, and provides PromQL to query and analyze those metrics.**

Prometheus primarily uses a **pull model**:

```text
Prometheus
    |
    | HTTP GET /metrics
    |
    v
Application / Exporter
```

The target exposes metrics, and Prometheus periodically scrapes them. Prometheus stores samples with timestamps and labels. ([Prometheus][1])

Example:

```text
http_requests_total{method="GET",status="200"} 15234
```

Prometheus might collect that every 15 seconds:

```text
10:00:00 → 15000
10:00:15 → 15100
10:00:30 → 15234
```

That creates a **time series**.

---

# 2. The Prometheus mental model

Think:

```text
                 PROMETHEUS
                     |
        +------------+------------+
        |            |            |
        v            v            v
     Collect       Store        Query
        |            |            |
        v            v            v
    /metrics      TSDB          PromQL
                     |
                     v
                  Rules
                 /     \
                v       v
         Recording     Alerting
            Rules        Rules
                         |
                         v
                   Alertmanager
```

And:

```text
Grafana
   |
   | PromQL
   v
Prometheus
```

---

# 3. Prometheus architecture

A typical Kubernetes architecture:

```text
                         Kubernetes
                              |
             +----------------+----------------+
             |                |                |
             v                v                v
        Application       Node Exporter    kube-state-metrics
          /metrics            /metrics          /metrics
             |                  |                  |
             +------------------+------------------+
                                |
                                | scrape
                                v
                       +----------------+
                       |  Prometheus    |
                       |    Server     |
                       +-------+--------+
                               |
                +--------------+--------------+
                |              |              |
                v              v              v
              TSDB         PromQL          Rules
                |                             |
                |                       +-----+-----+
                |                       |           |
                |                       v           v
                |                  Alerting      Recording
                |                    Rules         Rules
                |                       |
                |                       v
                |                 Alertmanager
                |                       |
                |               +-------+-------+
                |               |       |       |
                |              Slack   Email   PagerDuty
                |
                v
              Grafana
```

Prometheus itself includes the server, storage, query engine and rule evaluation; Alertmanager and exporters are separate components in the ecosystem. ([Prometheus][1])

---

# 4. Major Prometheus components

You should know these for interviews.

| Component           | Purpose                                                                    |
| ------------------- | -------------------------------------------------------------------------- |
| Prometheus Server   | Scrapes and stores metrics                                                 |
| TSDB                | Local time-series storage                                                  |
| PromQL              | Query language                                                             |
| Exporters           | Expose metrics from systems                                                |
| Client libraries    | Instrument applications                                                    |
| Service Discovery   | Find scrape targets                                                        |
| Recording Rules     | Precompute queries                                                         |
| Alerting Rules      | Generate alerts                                                            |
| Alertmanager        | Group, route, silence alerts                                               |
| Pushgateway         | Metrics for short-lived jobs                                               |
| Grafana             | Visualization                                                              |
| Prometheus Operator | Kubernetes-native management                                               |
| Prometheus Adapter  | Exposes Prometheus metrics through Kubernetes Custom/External Metrics APIs |

Prometheus documentation describes exporters as a way to expose metrics from systems that aren't directly instrumented, while client libraries are used for direct application instrumentation. ([Prometheus][2])

---

# 5. Prometheus uses a pull model

This is one of the first interview questions.

Suppose:

```text
Application
10.0.1.20:8080
```

exposes:

```text
GET /metrics
```

Prometheus does:

```text
Prometheus
    |
    | GET http://10.0.1.20:8080/metrics
    |
    v
Application
    |
    | metrics response
    v
Prometheus
```

Example response:

```text
# HELP http_requests_total Total HTTP requests
# TYPE http_requests_total counter

http_requests_total{method="GET",status="200"} 1500
http_requests_total{method="GET",status="500"} 20
```

---

# 6. Why pull instead of push?

Prometheus can determine:

```text
Is target alive?
Is target reachable?
How long did scrape take?
Did scrape fail?
```

For example:

```text
up{job="payment-api"} 1
```

means the target was successfully scraped.

```text
up{job="payment-api"} 0
```

means the scrape failed.

---

# 7. Pushgateway

You may hear:

> "Prometheus is pull based, so how do batch jobs send metrics?"

For short-lived jobs:

```text
Batch Job
   |
   | push
   v
Pushgateway
   |
   | scrape
   v
Prometheus
```

But **don't use Pushgateway as a general replacement for Prometheus scraping**. Its intended use is mainly for short-lived batch jobs. Prometheus documents Pushgateway specifically as a mechanism for persisting the latest push from batch jobs so Prometheus can scrape it. ([Prometheus][3])

---

# 8. Metrics data model

This is extremely important.

Prometheus stores:

```text
metric name
+
labels
+
timestamp
+
value
```

Example:

```text
http_requests_total{
    method="GET",
    status="200",
    service="payment"
}
```

Each unique combination of metric + labels is a **unique time series**. ([Prometheus][4])

---

# 9. Labels

Suppose:

```text
http_requests_total
```

with:

```text
method="GET"
```

and:

```text
method="POST"
```

These are two different series:

```text
http_requests_total{method="GET"}

http_requests_total{method="POST"}
```

Add:

```text
status="200"
```

and you get additional series.

This is powerful but creates the **cardinality problem**.

---

# 10. Cardinality

Suppose you have:

```text
10 services
×
5 HTTP methods
×
5 status codes
×
100 pods
```

That's:

```text
10 × 5 × 5 × 100
=
25,000 series
```

Now imagine adding:

```text
user_id
```

with:

```text
10 million users
```

You could explode your cardinality.

### Never casually use labels like:

```text
user_id
request_id
transaction_id
session_id
```

for high-volume metrics.

Instead:

```text
service
namespace
pod
method
status
endpoint
```

with care.

---

# 11. Four important metric types

You should know:

```text
Counter
Gauge
Histogram
Summary
```

---

# 12. Counter

Counter only increases, except when the process restarts.

Example:

```text
http_requests_total
```

```text
100
110
125
140
```

You don't normally query a counter directly to calculate request rate.

Use:

```promql
rate(http_requests_total[5m])
```

---

# 13. Gauge

Gauge can increase and decrease.

Examples:

```text
memory_usage_bytes
active_connections
queue_depth
temperature
```

Example:

```text
100
120
80
60
```

Use directly:

```promql
queue_depth
```

---

# 14. Histogram

Histogram is extremely important for latency.

Suppose:

```text
http_request_duration_seconds
```

Histogram creates buckets such as:

```text
≤ 0.1 sec
≤ 0.25 sec
≤ 0.5 sec
≤ 1 sec
≤ 2.5 sec
...
```

You get:

```text
http_request_duration_seconds_bucket
http_request_duration_seconds_sum
http_request_duration_seconds_count
```

Then you can calculate percentiles using:

```promql
histogram_quantile(
  0.95,
  rate(http_request_duration_seconds_bucket[5m])
)
```

This gives approximate **p95 latency**.

---

# 15. Summary

Summary also calculates quantiles, but the important interview distinction is:

```text
Histogram
    |
    +-- buckets
    +-- can aggregate across instances
    +-- quantile calculated by Prometheus

Summary
    |
    +-- quantiles calculated client-side
    +-- generally cannot aggregate quantiles meaningfully
```

For Kubernetes fleet-wide latency analysis, histograms are generally more useful.

---

# 16. Exporters

Suppose Linux doesn't expose metrics in Prometheus format.

You deploy:

```text
node-exporter
```

Architecture:

```text
Linux
  |
  | OS metrics
  v
Node Exporter
  |
  | /metrics
  v
Prometheus
```

Node Exporter exposes metrics such as:

```text
node_cpu_seconds_total
node_memory_MemAvailable_bytes
node_filesystem_avail_bytes
```

---

# 17. kube-state-metrics

This is very important in Kubernetes.

`kube-state-metrics` exposes Kubernetes **object state**.

For example:

```text
Deployment replicas
Pod status
DaemonSet status
StatefulSet status
Node conditions
```

Architecture:

```text
Kubernetes API
       |
       v
kube-state-metrics
       |
       | /metrics
       v
Prometheus
```

Example:

```promql
kube_deployment_status_replicas_available
```

This is different from node resource metrics.

---

# 18. kube-state-metrics vs metrics-server

Another common interview question.

### metrics-server

Primarily provides resource usage metrics to Kubernetes APIs, such as:

```text
CPU
Memory
```

and is commonly used by HPA/VPA and `kubectl top`.

### kube-state-metrics

Exposes Kubernetes object's **desired/current state** as Prometheus metrics.

```text
Deployment replicas
Pod phase
Node conditions
DaemonSet status
```

So:

```text
metrics-server
    ↓
resource usage API

kube-state-metrics
    ↓
Kubernetes object state → Prometheus
```

---

# 19. Service Discovery

How does Prometheus know what to scrape?

In Kubernetes, you don't want:

```yaml
targets:
  - 10.0.1.20
  - 10.0.1.21
  - 10.0.1.22
```

because Pods change.

Instead:

```text
Kubernetes API
      |
      | service discovery
      v
Prometheus
      |
      v
Current Pods/Services
```

Prometheus dynamically discovers targets through service discovery mechanisms. ([Prometheus][1])

---

# 20. Prometheus configuration

A simplified configuration:

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:

  - job_name: prometheus

    static_configs:
      - targets:
          - localhost:9090
```

Meaning:

```text
every 15 seconds
      ↓
scrape target
```

---

# 21. Scrape interval vs evaluation interval

### `scrape_interval`

How frequently Prometheus collects metrics.

```yaml
scrape_interval: 15s
```

### `evaluation_interval`

How frequently recording/alerting rules are evaluated.

```yaml
evaluation_interval: 15s
```

They are independent concepts.

---

# 22. PromQL — the most important interview area

You should be comfortable with:

```text
Selectors
rate
irate
increase
sum
avg
min/max
count
by
without
topk
histogram_quantile
rate + aggregation
label matching
absent
predict_linear
```

Let's go through the important ones.

---

# 23. Basic metric query

```promql
http_requests_total
```

All series.

Filter:

```promql
http_requests_total{status="500"}
```

Multiple labels:

```promql
http_requests_total{
  service="payment",
  status="500"
}
```

Regex:

```promql
http_requests_total{
  status=~"5.."
}
```

Negative:

```promql
http_requests_total{
  status!="200"
}
```

---

# 24. `rate()`

For counters:

```promql
rate(http_requests_total[5m])
```

Meaning:

> Average per-second increase over the last five minutes.

For example:

```text
1000 requests
1100
1200
```

Prometheus calculates the rate.

Use:

```promql
rate(counter[5m])
```

for most dashboards/alerts involving counters.

---

# 25. `increase()`

```promql
increase(http_requests_total[1h])
```

means:

> How much did the counter increase during the last hour?

Example:

```text
10,000 requests/hour
```

---

# 26. `irate()`

```promql
irate(http_requests_total[5m])
```

Uses the most recent samples to estimate an instant rate.

Use carefully.

Typical guidance:

```text
rate()  → alerts / dashboards
irate() → rapidly changing graphs
```

---

# 27. Aggregation

Suppose:

```text
http_requests_total
```

exists for:

```text
pod-1
pod-2
pod-3
```

Total:

```promql
sum(rate(http_requests_total[5m]))
```

Per service:

```promql
sum by (service) (
  rate(http_requests_total[5m])
)
```

Per namespace:

```promql
sum by (namespace) (
  rate(http_requests_total[5m])
)
```

---

# 28. Error rate

A very common interview PromQL:

```promql
sum(rate(http_requests_total{status=~"5.."}[5m]))
/
sum(rate(http_requests_total[5m]))
```

This gives:

```text
5xx request rate / total request rate
```

For percentage:

```promql
100 *
sum(rate(http_requests_total{status=~"5.."}[5m]))
/
sum(rate(http_requests_total[5m]))
```

---

# 29. CPU utilization

A common node CPU query:

```promql
100 *
(
  1 -
  avg by(instance) (
    rate(node_cpu_seconds_total{
      mode="idle"
    }[5m])
  )
)
```

Conceptually:

```text
1 - idle
=
CPU used
```

---

# 30. Memory utilization

Conceptually:

```promql
100 *
(
  1 -
  node_memory_MemAvailable_bytes
  /
  node_memory_MemTotal_bytes
)
```

---

# 31. Top CPU-consuming pods

A common pattern:

```promql
topk(
  10,
  sum by (pod) (
    rate(container_cpu_usage_seconds_total[5m])
  )
)
```

Meaning:

```text
Calculate CPU
   ↓
Group by pod
   ↓
Take top 10
```

---

# 32. Pod restarts

```promql
increase(
  kube_pod_container_status_restarts_total[1h]
)
```

This tells you how much the restart counter increased in the last hour.

---

# 33. Pending pods

Depending on metric labels/version:

```promql
sum(
  kube_pod_status_phase{
    phase="Pending"
  }
)
```

---

# 34. Recording rules

Suppose this query is expensive:

```promql
sum by (service) (
  rate(http_requests_total[5m])
)
```

and Grafana runs it every few seconds.

Create:

```yaml
groups:
  - name: application
    rules:

      - record: service:http_requests_per_second:rate5m

        expr: |
          sum by (service) (
            rate(http_requests_total[5m])
          )
```

Prometheus periodically evaluates it and stores the result as a new time series. Recording rules are specifically designed to precompute expensive/frequently used expressions. ([Prometheus][5])

Then Grafana queries:

```promql
service:http_requests_per_second:rate5m
```

instead of recomputing the entire expression.

---

# 35. Alerting rules

Example:

```yaml
groups:

  - name: payment-alerts

    rules:

      - alert: HighErrorRate

        expr: |
          (
            sum(rate(http_requests_total{
              service="payment",
              status=~"5.."
            }[5m]))
            /
            sum(rate(http_requests_total{
              service="payment"
            }[5m]))
          ) > 0.05

        for: 5m

        labels:
          severity: critical

        annotations:
          summary: "Payment API error rate is high"
```

Meaning:

```text
Error rate > 5%
        |
       5 min
        |
        v
Alert fires
```

---

# 36. Alertmanager

Prometheus doesn't directly handle all notification routing.

Architecture:

```text
Prometheus
    |
    | alerting rule
    v
Alert
    |
    v
Alertmanager
    |
    +---- Slack
    |
    +---- Email
    |
    +---- PagerDuty
```

Alertmanager handles grouping, deduplication, silencing, inhibition and notification routing. ([Prometheus][6])

---

# 37. Kubernetes Prometheus Operator

In Kubernetes, manually writing:

```text
prometheus.yml
scrape_configs
```

for everything becomes painful.

Prometheus Operator gives Kubernetes CRDs such as:

```text
Prometheus
ServiceMonitor
PodMonitor
PrometheusRule
Alertmanager
```

Architecture:

```text
Kubernetes
     |
     v
Prometheus Operator
     |
     +---- Prometheus
     |
     +---- Alertmanager
     |
     +---- ServiceMonitor
     |
     +---- PrometheusRule
```

---

# 38. ServiceMonitor

Suppose:

```text
payment-service
```

exposes:

```text
/metrics
```

You can create:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor

metadata:
  name: payment

spec:
  selector:
    matchLabels:
      app: payment

  endpoints:
    - port: metrics
      path: /metrics
      interval: 15s
```

Conceptually:

```text
Service
   |
   | selected by ServiceMonitor
   v
Prometheus Operator
   |
   v
Prometheus scrape config
   |
   v
/metrics
```

---

# 39. Now the important part: Prometheus Adapter

This is where Prometheus connects to Kubernetes HPA using custom metrics.

The problem:

Prometheus understands:

```text
PromQL
```

But HPA doesn't simply query Prometheus directly.

HPA expects Kubernetes Metrics APIs.

For example:

```text
metrics.k8s.io
custom.metrics.k8s.io
external.metrics.k8s.io
```

Prometheus Adapter implements Kubernetes Custom, Resource and External Metrics APIs and can be used with HPA. ([GitHub][7])

---

# 40. Why do we need Prometheus Adapter?

Suppose your application exposes:

```text
http_requests_total
```

Prometheus stores it.

You want:

> "Scale payment-api when requests per second per pod exceed 20."

HPA needs access to that metric.

The architecture becomes:

```text
Application
    |
    | /metrics
    v
Prometheus
    |
    | PromQL
    v
Prometheus Adapter
    |
    | Kubernetes Custom Metrics API
    v
HPA
    |
    v
Deployment
    |
    v
More Pods
```

This is the key diagram.

---

# 41. Complete architecture

```text
                    +------------------+
                    |   Application    |
                    | payment-api      |
                    +--------+---------+
                             |
                          /metrics
                             |
                             v
                    +------------------+
                    |   Prometheus     |
                    |                  |
                    | TSDB + PromQL    |
                    +--------+---------+
                             |
                         PromQL query
                             |
                             v
                  +----------------------+
                  | Prometheus Adapter   |
                  |                      |
                  | Custom Metrics API   |
                  +----------+-----------+
                             |
                /apis/custom.metrics.k8s.io
                             |
                             v
                    +----------------+
                    |      HPA       |
                    +-------+--------+
                            |
                     desired replicas
                            |
                            v
                    +---------------+
                    | Deployment    |
                    +-------+-------+
                            |
                            v
                         Pods
```

---

# 42. What does the Adapter actually do?

This is the most important concept.

Prometheus has:

```text
http_requests_total{
  namespace="payment",
  pod="payment-abc"
}
```

Adapter configuration says:

```text
Find this Prometheus series
        ↓
Associate it with Kubernetes Pod
        ↓
Rename/expose it as:
http_requests_per_second
```

Then Kubernetes can query:

```text
custom.metrics.k8s.io
```

---

# 43. Adapter configuration

The Prometheus Adapter configuration contains rules like:

```yaml
rules:
  custom:

    - seriesQuery: |
        http_requests_total{
          namespace!="",
          pod!=""
        }

      resources:
        overrides:
          namespace:
            resource: namespace
          pod:
            resource: pod

      name:
        matches: "^(.*)_total$"
        as: "${1}_per_second"

      metricsQuery: |
        sum(
          rate(
            http_requests_total{
              <<.LabelMatchers>>
            }[2m]
          )
        ) by (
          <<.GroupBy>>
        )
```

Don't memorize every syntax character.

Understand the three major sections.

---

# 44. `seriesQuery`

```yaml
seriesQuery: |
  http_requests_total{
    namespace!="",
    pod!=""
  }
```

Means:

> Find Prometheus series matching this metric and labels.

---

# 45. `resources`

```yaml
resources:
  overrides:
    namespace:
      resource: namespace

    pod:
      resource: pod
```

This tells the Adapter:

> These Prometheus labels correspond to Kubernetes resources.

So:

```text
Prometheus:

namespace="payment"
pod="payment-abc"
```

maps to:

```text
Kubernetes:

namespace/payment
pod/payment-abc
```

This mapping is critical.

The adapter documentation explicitly warns that consumers such as HPA don't perform this association themselves; the adapter needs the series labels and configuration to associate metrics with Kubernetes resources. ([GitHub][7])

---

# 46. `name`

```yaml
name:
  matches: "^(.*)_total$"
  as: "${1}_per_second"
```

This transforms:

```text
http_requests_total
```

into something exposed as:

```text
http_requests_per_second
```

---

# 47. `metricsQuery`

This is where the actual PromQL comes from.

```yaml
metricsQuery: |
  sum(
    rate(
      http_requests_total{
        <<.LabelMatchers>>
      }[2m]
    )
  ) by (
    <<.GroupBy>>
  )
```

The Adapter substitutes:

```text
<<.LabelMatchers>>
```

and:

```text
<<.GroupBy>>
```

based on the Kubernetes resource request.

---

# 48. HPA

Now create:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler

metadata:
  name: payment-api

spec:

  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: payment-api

  minReplicas: 2
  maxReplicas: 10

  metrics:

    - type: Pods

      pods:
        metric:
          name: http_requests_per_second

        target:
          type: AverageValue
          averageValue: "20"
```

This means:

> Keep average requests/sec around 20 per Pod.

---

# 49. Complete HPA flow

Suppose:

```text
2 Pods
```

Traffic:

```text
100 requests/sec
```

Then:

```text
50 req/sec/pod
```

HPA target:

```text
20 req/sec/pod
```

So HPA calculates approximately:

```text
desired replicas
=
current replicas × current metric / target metric
```

Conceptually:

```text
2 × 50 / 20
=
5 replicas
```

So:

```text
2 Pods
  ↓
5 Pods
```

---

# 50. The complete chain

This is the **single most important diagram** for your question:

```text
                         REQUESTS
                            |
                            v
                    +---------------+
                    | payment-api   |
                    | Pod           |
                    +-------+-------+
                            |
                         /metrics
                            |
                            v
                  +-------------------+
                  |    Prometheus     |
                  |                   |
                  | http_requests_    |
                  | total             |
                  +---------+---------+
                            |
                          PromQL
                            |
                            v
                +-----------------------+
                | Prometheus Adapter    |
                |                       |
                | seriesQuery           |
                | resources             |
                | metricsQuery          |
                +-----------+-----------+
                            |
                  Custom Metrics API
                            |
                            v
                     +-------------+
                     |     HPA     |
                     +------+------+
                            |
                    desired replicas
                            |
                            v
                     +-------------+
                     | Deployment  |
                     +------+------+
                            |
                            v
                    +---------------+
                    | Pods increase |
                    +---------------+
```

---

# 51. Complete demo

Let's build a small application.

Application exposes:

```text
http_requests_total
```

with:

```text
namespace
pod
```

labels.

For the demo, imagine the metric looks like:

```text
http_requests_total{
  namespace="demo",
  pod="demo-app-123"
} 5000
```

---

# 52. Install Prometheus

For a real Kubernetes setup, the easiest interview/demo approach is generally **kube-prometheus-stack** via Helm.

Conceptually:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

helm repo update
```

Install:

```bash
helm install monitoring \
  prometheus-community/kube-prometheus-stack \
  -n monitoring \
  --create-namespace
```

You now have the monitoring stack.

---

# 53. Verify

```bash
kubectl get pods -n monitoring
```

You'll typically see components such as:

```text
prometheus
grafana
alertmanager
kube-state-metrics
node-exporter
operator
```

The exact names depend on chart/version/release naming.

---

# 54. Application

Example Deployment:

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: demo-app
  namespace: demo

spec:
  replicas: 2

  selector:
    matchLabels:
      app: demo-app

  template:

    metadata:
      labels:
        app: demo-app

    spec:

      containers:
        - name: app
          image: myrepo/demo-app:v1

          ports:
            - name: http
              containerPort: 8080

            - name: metrics
              containerPort: 9090
```

---

# 55. Service

```yaml
apiVersion: v1
kind: Service

metadata:
  name: demo-app
  namespace: demo

  labels:
    app: demo-app

spec:

  selector:
    app: demo-app

  ports:

    - name: http
      port: 80
      targetPort: http

    - name: metrics
      port: 9090
      targetPort: metrics
```

---

# 56. ServiceMonitor

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor

metadata:
  name: demo-app
  namespace: monitoring

spec:

  namespaceSelector:
    matchNames:
      - demo

  selector:
    matchLabels:
      app: demo-app

  endpoints:

    - port: metrics
      path: /metrics
      interval: 15s
```

Now:

```text
demo-app
   |
   | /metrics
   v
Service
   |
   v
ServiceMonitor
   |
   v
Prometheus
```

---

# 57. Verify Prometheus is scraping

Query:

```promql
up
```

Then:

```promql
up{namespace="demo"}
```

Depending on your scrape labels/configuration.

You can also check:

```promql
http_requests_total
```

If the metric exists, you're good.

---

# 58. Test the metric

Generate traffic:

```bash
for i in {1..1000}; do
  curl http://demo-app
done
```

Now:

```promql
rate(http_requests_total[2m])
```

You might get:

```text
12.3
```

meaning approximately:

```text
12.3 requests/sec
```

for that series.

---

# 59. Install Prometheus Adapter

Using Helm:

```bash
helm repo add prometheus-community \
  https://prometheus-community.github.io/helm-charts

helm repo update
```

Install:

```bash
helm install prometheus-adapter \
  prometheus-community/prometheus-adapter \
  -n monitoring
```

You must configure the adapter to point to your Prometheus service and define the custom metric rule.

The official chart supports `rules.custom` for custom metric mappings. ([GitHub][8])

---

# 60. Adapter values

For example:

```yaml
prometheus:
  url: http://monitoring-kube-prometheus-prometheus.monitoring.svc
  port: 9090

rules:

  default: false

  custom:

    - seriesQuery: |
        http_requests_total{
          namespace!="",
          pod!=""
        }

      resources:
        overrides:

          namespace:
            resource: namespace

          pod:
            resource: pod

      name:
        matches: "^(.*)_total$"
        as: "${1}_per_second"

      metricsQuery: |
        sum(
          rate(
            http_requests_total{
              <<.LabelMatchers>>
            }[2m]
          )
        ) by (
          <<.GroupBy>>
        )
```

Install/upgrade:

```bash
helm upgrade --install prometheus-adapter \
  prometheus-community/prometheus-adapter \
  -n monitoring \
  -f adapter-values.yaml
```

---

# 61. Check the Custom Metrics API

Run:

```bash
kubectl get --raw \
/apis/custom.metrics.k8s.io/v1beta1
```

You should see your exposed metrics if discovery/configuration is correct.

The Prometheus Adapter documentation specifically recommends this API endpoint when troubleshooting whether metrics are being exposed. ([GitHub][7])

---

# 62. Query metric for Pods

You can inspect something like:

```bash
kubectl get --raw \
"/apis/custom.metrics.k8s.io/v1beta1/namespaces/demo/pods/*/http_requests_per_second"
```

Now you're no longer talking directly to Prometheus.

You're talking:

```text
kubectl
  |
  v
Kubernetes API
  |
  v
Prometheus Adapter
  |
  v
Prometheus
```

---

# 63. HPA

Now:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler

metadata:
  name: demo-app
  namespace: demo

spec:

  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: demo-app

  minReplicas: 2
  maxReplicas: 10

  behavior:

    scaleUp:
      stabilizationWindowSeconds: 0

    scaleDown:
      stabilizationWindowSeconds: 300

  metrics:

    - type: Pods

      pods:

        metric:
          name: http_requests_per_second

        target:
          type: AverageValue
          averageValue: "20"
```

---

# 64. HPA working

Suppose:

```text
Current:
2 Pods

Metric:
50 req/sec/pod

Target:
20 req/sec/pod
```

HPA sees:

```text
50 > 20
```

and increases replicas.

```text
2 Pods
   ↓
5 Pods
```

After scaling:

```text
Traffic = 100 req/sec

5 Pods

≈20 req/sec/pod
```

HPA stabilizes around the target.

---

# 65. Full request flow

Let's trace **one request** from beginning to end.

```text
User
 |
 | HTTP request
 v
Service
 |
 v
Pod
 |
 | application handles request
 |
 +----> http_requests_total++
 |
 v
/metrics
```

Prometheus:

```text
Prometheus
 |
 | every 15 sec
 |
 | GET /metrics
 v
Pod
 |
 v
Prometheus TSDB
```

Then Adapter:

```text
HPA
 |
 | "Give me http_requests_per_second for Pods"
 v
Kubernetes Custom Metrics API
 |
 v
Prometheus Adapter
 |
 | executes PromQL
 v
Prometheus
 |
 | result
 v
Adapter
 |
 v
HPA
```

Then:

```text
HPA
 |
 | desired replicas = 5
 v
Deployment
 |
 v
5 Pods
```

---

# 66. Why not connect HPA directly to Prometheus?

Because Kubernetes HPA is designed to consume Kubernetes metrics APIs rather than arbitrary PromQL.

The adapter acts as the bridge:

```text
Prometheus world
       |
       | PromQL
       v
Prometheus Adapter
       |
       | Kubernetes Metrics API
       v
Kubernetes world
       |
       v
HPA
```

This is the conceptual reason for the Adapter.

---

# 67. Custom Metrics vs External Metrics

Another important interview question.

### Custom metrics

Metrics associated with Kubernetes resources.

Example:

```text
requests per pod
```

```text
Pod
 ↓
http_requests_per_second
```

HPA:

```yaml
type: Pods
```

---

### External metrics

Metrics that are external to Kubernetes resources.

Examples:

```text
AWS SQS queue depth
Kafka lag
CloudWatch metric
External API queue
```

Conceptually:

```text
SQS
 |
 | queue depth = 5000
 v
Prometheus
 |
 v
Adapter
 |
 v
HPA
```

Then HPA can say:

> Scale workers based on queue depth.

---

# 68. Custom metric example

```text
http_requests_per_second
```

belongs to:

```text
Pod
```

So:

```yaml
type: Pods
```

---

# 69. External metric example

Suppose:

```text
queue_messages = 5000
```

You want:

```text
target = 100 messages/pod
```

HPA might use:

```yaml
metrics:

  - type: External

    external:

      metric:
        name: queue_messages

      target:
        type: AverageValue
        averageValue: "100"
```

The exact adapter query/configuration depends on where the metric comes from.

---

# 70. Prometheus Adapter troubleshooting

This is **very interview-relevant**.

Suppose:

```text
HPA:
unknown metric
```

Don't randomly restart things.

Check:

### Step 1

Prometheus has the metric:

```promql
http_requests_total
```

---

### Step 2

PromQL query works:

```promql
sum(
  rate(http_requests_total[2m])
)
```

---

### Step 3

Adapter sees the series.

Check:

```bash
kubectl logs deployment/prometheus-adapter -n monitoring
```

---

### Step 4

Custom Metrics API:

```bash
kubectl get --raw \
/apis/custom.metrics.k8s.io/v1beta1
```

---

### Step 5

Check metric for the specific Pod:

```bash
kubectl get --raw \
"/apis/custom.metrics.k8s.io/v1beta1/namespaces/demo/pods/*/http_requests_per_second"
```

---

### Step 6

Check HPA:

```bash
kubectl describe hpa demo-app -n demo
```

---

# 71. Very common Adapter problem

You have:

```text
http_requests_total
```

but it doesn't have:

```text
namespace
pod
```

Then the Adapter can't properly map it to:

```text
Pod
```

You might have:

```text
http_requests_total{service="payment"}
```

but HPA asks:

```text
Give me this metric for Pod X
```

The Adapter needs an appropriate resource mapping.

That's why this section matters:

```yaml
resources:
  overrides:
    namespace:
      resource: namespace

    pod:
      resource: pod
```

---

# 72. Another common problem: metric disappears

Prometheus Adapter has discovery/relist settings.

The adapter documentation notes that `metrics-max-age` should generally be at least as large as the scrape interval, otherwise metrics can intermittently disappear from the adapter. ([GitHub][7])

So if:

```text
Prometheus scrape = 30s
```

don't configure discovery freshness so aggressively that a metric disappears before the next scrape is visible.

---

# 73. Recording rule + Adapter

For production, you may not want the Adapter to execute a complicated query repeatedly.

Instead:

```text
Prometheus
   |
   | expensive PromQL
   v
Recording Rule
   |
   v
payment:http_requests_per_second
   |
   v
Adapter
   |
   v
HPA
```

Example:

```yaml
groups:

  - name: payment

    rules:

      - record: payment:http_requests_per_second

        expr: |
          sum by (namespace, pod) (
            rate(
              http_requests_total{
                namespace="payment"
              }[2m]
            )
          )
```

Then Adapter can expose the recording rule.

This is often cleaner and more efficient.

---

# 74. Prometheus storage

Prometheus stores data locally in its TSDB.

Architecture:

```text
Prometheus
    |
    v
WAL
    |
    v
TSDB blocks
```

WAL helps protect recently ingested data and supports recovery.

Prometheus local storage is intentionally single-node; for longer retention or larger-scale durability, Prometheus provides remote storage integrations such as Remote Write/Remote Read. ([Prometheus][9])

---

# 75. Remote Write

Suppose:

```text
Prometheus
```

needs long-term storage.

Architecture:

```text
Prometheus
     |
     | remote_write
     v
Long-term backend
     |
     +-- Thanos
     +-- Cortex/Mimir
     +-- other compatible systems
```

Remote Write sends samples to a compatible receiver. ([Prometheus][10])

---

# 76. Why Thanos/Mimir?

Suppose you have:

```text
100 Kubernetes clusters
```

and don't want:

```text
100 isolated Prometheus servers
```

You can have:

```text
Cluster A
  Prometheus
       |
       |
Cluster B
  Prometheus
       |
       +--------> Long-term / global metrics backend
       |
Cluster C
  Prometheus
```

This provides a broader/global view and long-term storage depending on architecture.

---

# 77. Prometheus HA

A common misconception:

> "Prometheus is automatically a distributed cluster."

No.

A normal Prometheus server is standalone.

You can run:

```text
Prometheus-1
Prometheus-2
```

with the same scrape configuration for HA, but then you need a strategy for deduplication/querying at scale, commonly using systems such as Thanos/Mimir or equivalent architectures.

---

# 78. Prometheus Operator architecture

For Kubernetes interviews, remember:

```text
                 Git
                  |
                  v
              Argo CD
                  |
                  v
       Prometheus Operator
                  |
       +----------+----------+
       |          |          |
       v          v          v
 Prometheus  Alertmanager  ServiceMonitor
       |                     |
       |                     |
       +----------+----------+
                  |
                  v
             Kubernetes
```

---

# 79. The complete Kubernetes monitoring stack

A very realistic EKS platform:

```text
                           EKS
                            |
       +--------------------+--------------------+
       |                    |                    |
       v                    v                    v
 Applications          Kubernetes            Nodes
       |                    |                    |
       | /metrics           |                    |
       v                    v                    v
   ServiceMonitor    kube-state-metrics    node-exporter
       |                    |                    |
       +--------------------+--------------------+
                            |
                            v
                      Prometheus
                            |
              +-------------+-------------+
              |             |             |
              v             v             v
           Grafana       Rules       Adapter
                            |             |
                            v             v
                      Alertmanager       HPA
                            |
                            v
                    Slack/PagerDuty
```

---

# 80. Where Dynatrace fits

Since you're familiar with Dynatrace, don't confuse the roles.

You can have:

```text
Application
   |
   +---- Prometheus metrics
   |
   +---- Dynatrace telemetry
```

Prometheus is especially strong for:

```text
Kubernetes metrics
Infrastructure metrics
PromQL
SRE alerting
HPA custom metrics
```

Dynatrace provides broader observability capabilities depending on deployment.

In an interview, don't claim that Prometheus replaces every observability platform.

---

# 81. Prometheus interview questions you should know

### Fundamentals

1. What is Prometheus?
2. Why pull instead of push?
3. What is a time series?
4. What are labels?
5. What is cardinality?
6. Counter vs Gauge?
7. Histogram vs Summary?
8. What is an exporter?
9. What is Service Discovery?
10. What is `/metrics`?

### Architecture

11. Prometheus architecture?
12. What is TSDB?
13. What is WAL?
14. What is Alertmanager?
15. What is Pushgateway?
16. What is remote write?
17. How do you achieve long-term storage?
18. How do you scale Prometheus?
19. How do you achieve HA?

### PromQL

20. `rate()` vs `irate()`?
21. `increase()`?
22. `sum by()`?
23. `sum without()`?
24. `histogram_quantile()`?
25. How do you calculate error rate?
26. How do you calculate CPU?
27. How do you find top pods?
28. How do you calculate p95 latency?
29. Recording rules?
30. Alerting rules?

### Kubernetes

31. Prometheus Operator?
32. ServiceMonitor?
33. PodMonitor?
34. PrometheusRule?
35. kube-state-metrics?
36. metrics-server vs Prometheus?
37. How does Prometheus discover Pods?

### HPA

38. Why can't HPA directly use arbitrary PromQL?
39. What is Prometheus Adapter?
40. Custom metrics vs external metrics?
41. How does Adapter map Prometheus labels to Kubernetes resources?
42. How does HPA use `custom.metrics.k8s.io`?
43. How would you troubleshoot an HPA custom metric?
44. How do you scale based on Kafka lag/SQS queue depth?

---

# 82. The HPA interview answer you should memorize

If they ask:

> **"Explain how you use Prometheus custom metrics with HPA."**

Say:

> "The application exposes a Prometheus metric such as `http_requests_total`. Prometheus scrapes and stores it. Because HPA consumes Kubernetes Metrics APIs rather than arbitrary PromQL directly, I deploy Prometheus Adapter. I configure the Adapter with a `seriesQuery` to discover the Prometheus series, resource mappings to associate labels such as namespace and pod with Kubernetes resources, and a `metricsQuery` containing the PromQL needed to calculate the metric. The Adapter exposes the result through `custom.metrics.k8s.io`. HPA queries that API, calculates the desired replica count based on the current metric versus the target, and updates the Deployment's replica count."

Then draw:

```text
App
 ↓
/metrics
 ↓
Prometheus
 ↓
PromQL
 ↓
Prometheus Adapter
 ↓
custom.metrics.k8s.io
 ↓
HPA
 ↓
Deployment
 ↓
Pods
```

That answer demonstrates that you understand **the entire data path**, not just YAML.

---

# 83. PromQL cheat sheet

Keep these ready for interviews:

```promql
# Current metric
http_requests_total
```

```promql
# Filter
http_requests_total{status="500"}
```

```promql
# Regex
http_requests_total{status=~"5.."}
```

```promql
# Request rate
rate(http_requests_total[5m])
```

```promql
# Increase
increase(http_requests_total[1h])
```

```promql
# Total request rate
sum(rate(http_requests_total[5m]))
```

```promql
# By service
sum by(service) (
  rate(http_requests_total[5m])
)
```

```promql
# Error rate
sum(rate(http_requests_total{status=~"5.."}[5m]))
/
sum(rate(http_requests_total[5m]))
```

```promql
# Top 10 pods
topk(
  10,
  sum by(pod) (
    rate(container_cpu_usage_seconds_total[5m])
  )
)
```

```promql
# p95 latency
histogram_quantile(
  0.95,
  sum by(le) (
    rate(http_request_duration_seconds_bucket[5m])
  )
)
```

```promql
# Memory utilization
1 -
(
  node_memory_MemAvailable_bytes
  /
  node_memory_MemTotal_bytes
)
```

```promql
# CPU utilization
1 -
avg by(instance) (
  rate(node_cpu_seconds_total{mode="idle"}[5m])
)
```

---

# 84. The entire Prometheus ecosystem in one diagram

This is the **one diagram I'd memorize for your interview**:

```text
                              USERS
                                |
                                v
                         APPLICATIONS
                                |
                         +------+------+
                         |             |
                      /metrics       Logs
                         |
                         v
                  +--------------+
                  |  Prometheus  |
                  |              |
                  |   Scrape     |
                  |   TSDB       |
                  |   PromQL     |
                  |   Rules      |
                  +------+-------+
                         |
          +--------------+--------------+
          |              |              |
          v              v              v
      Recording       Alerting       Remote Write
       Rules           Rules             |
          |              |               v
          |              v          Long-term Store
          |         Alertmanager
          |              |
          |       +------+------+ 
          |       |      |      |
          |     Slack   Email  PagerDuty
          |
          v
       Grafana


KUBERNETES SIDE
────────────────────────────────────────────────────

             Kubernetes API
                    |
          +---------+---------+
          |                   |
          v                   v
   kube-state-metrics     Service Discovery
          |                   |
          |                   |
          +---------+---------+
                    |
                    v
                Prometheus
                    |
                    |
                    v
           Prometheus Adapter
                    |
           custom.metrics.k8s.io
                    |
                    v
                   HPA
                    |
                    v
               Deployment
                    |
                    v
                   Pods
```

---

# 85. Finally: what I'd prioritize for your interview

With your **6 YOE / Senior Platform Engineer** profile, don't spend equal time on every Prometheus feature.

I'd prioritize:

### Must know extremely well

```text
1. Prometheus architecture
2. Pull model
3. Metrics / labels / cardinality
4. Counter / Gauge / Histogram
5. rate / increase / histogram_quantile
6. sum by / aggregation
7. Error-rate PromQL
8. CPU/memory PromQL
9. Exporters
10. Service Discovery
11. kube-state-metrics
12. ServiceMonitor / PodMonitor
13. Recording rules
14. Alerting + Alertmanager
15. Prometheus Adapter
16. Custom Metrics API
17. HPA custom metrics
18. Troubleshooting HPA + Adapter
```

### Senior-level topics

```text
19. Cardinality management
20. Prometheus HA
21. Remote Write
22. Long-term storage
23. Thanos/Mimir architecture
24. Federation
25. Prometheus performance
26. Scrape interval/evaluation interval
27. Relabeling
28. Recording rule design
29. Alert design
30. SLO/error-budget queries
```

### The most important architecture to be able to draw from memory

```text
Application
    |
    | /metrics
    v
Prometheus
    |
    +---- PromQL ----> Grafana
    |
    +---- Alert Rules --> Alertmanager --> PagerDuty/Slack
    |
    +---- Recording Rules
    |
    +---- Remote Write --> Long-term Storage
    |
    +---- Prometheus Adapter
                    |
                    v
             Custom Metrics API
                    |
                    v
                   HPA
                    |
                    v
                Deployment
                    |
                    v
                   Pods
```

The **Prometheus → Adapter → Custom Metrics API → HPA** path is particularly worth practicing hands-on. It is the area where a lot of candidates can explain Prometheus dashboards but struggle to explain how **Prometheus actually becomes an autoscaling signal**.

[1]: https://prometheus.io/docs/introduction/overview/?utm_source=chatgpt.com "Overview | Prometheus"
[2]: https://prometheus.io/docs/instrumenting/exporters/?utm_source=chatgpt.com "Exporters and integrations | Prometheus"
[3]: https://prometheus.io/docs/introduction/glossary/?utm_source=chatgpt.com "Glossary | Prometheus"
[4]: https://prometheus.io/docs/concepts/data_model/?utm_source=chatgpt.com "Data model | Prometheus"
[5]: https://prometheus.io/docs/prometheus/latest/configuration/recording_rules/?utm_source=chatgpt.com "Defining recording rules | Prometheus"
[6]: https://prometheus.io/docs/alerting/latest/overview/?utm_source=chatgpt.com "Alerting overview | Prometheus"
[7]: https://github.com/kubernetes-sigs/prometheus-adapter/blob/master/README.md?utm_source=chatgpt.com "prometheus-adapter/README.md at master · kubernetes-sigs/prometheus-adapter · GitHub"
[8]: https://github.com/prometheus-community/helm-charts/blob/main/charts/prometheus-adapter/README.md?utm_source=chatgpt.com "helm-charts/charts/prometheus-adapter/README.md at main · prometheus-community/helm-charts · GitHub"
[9]: https://prometheus.io/docs/prometheus/latest/storage/?utm_source=chatgpt.com "Storage | Prometheus"
[10]: https://prometheus.io/docs/specs/prw/remote_write_spec/?utm_source=chatgpt.com "Prometheus Remote-Write 1.0 specification | Prometheus"
