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
