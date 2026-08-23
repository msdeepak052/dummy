# Istio — Short Interview Notes

## 1. What is Istio?

**Istio is a service mesh for Kubernetes.**

It manages **service-to-service communication** without requiring you to put networking logic inside every application.

It provides:

* Traffic management
* mTLS/security
* Retries and timeouts
* Observability
* Canary/traffic splitting
* Authorization policies

---

# 2. Basic Architecture

Istio mainly has:

```text
                Istiod
                  |
        Configuration / Certificates
                  |
       +----------+----------+
       |                     |
       v                     v
   Pod A                   Pod B
 +---------+             +---------+
 | App     |             | App     |
 |         |             |         |
 | Envoy   | <---------> | Envoy   |
 +---------+             +---------+
```

### Envoy

Each application pod gets an **Envoy sidecar**.

Traffic goes:

```text
App A
  ↓
Envoy A
  ↓
Envoy B
  ↓
App B
```

### Istiod

Istiod is the control plane.

It provides configuration to Envoy proxies and handles service-mesh control-plane functions such as certificate management.

---

# 3. Why do we need Istio?

Without Istio:

```text
App A
  ↓
Service B
```

The application itself may need to implement:

```text
retry
timeout
TLS
traffic routing
```

With Istio:

```text
App A
  ↓
Envoy
  ↓
Envoy
  ↓
App B
```

Istio handles many of these networking concerns outside the application.

---

# 4. Install/enable Istio injection

After installing Istio, label the namespace:

```bash
kubectl label namespace default istio-injection=enabled
```

Now a new pod created in that namespace gets an Envoy sidecar automatically.

Check:

```bash
kubectl get pods
```

You might see:

```text
NAME       READY
app-pod    2/2
```

Why `2/2`?

```text
1 container → application
1 container → Envoy
```

---

# 5. Demo 1 — Automatic mTLS

Suppose:

```text
payment → order
```

With Istio mTLS:

```text
payment
   |
 Envoy
   |
 encrypted mTLS
   |
 Envoy
   |
 order
```

You don't have to modify the application to implement TLS.

You can configure:

```yaml
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata:
  name: default
  namespace: default

spec:
  mtls:
    mode: STRICT
```

Now workloads must communicate using mTLS.

---

# 6. Demo 2 — Canary deployment

Suppose:

```text
payment-v1 → 90%
payment-v2 → 10%
```

Istio can control this without changing the application.

### DestinationRule

```yaml
apiVersion: networking.istio.io/v1
kind: DestinationRule

metadata:
  name: payment

spec:
  host: payment

  subsets:

    - name: v1
      labels:
        version: v1

    - name: v2
      labels:
        version: v2
```

Now tell Istio how to route traffic:

```yaml
apiVersion: networking.istio.io/v1
kind: VirtualService

metadata:
  name: payment

spec:

  hosts:
    - payment

  http:

    - route:

        - destination:
            host: payment
            subset: v1
          weight: 90

        - destination:
            host: payment
            subset: v2
          weight: 10
```

Architecture:

```text
                  payment-service
                        |
                  VirtualService
                    /       \
                  90%       10%
                   |          |
                  v1          v2
```

This is very useful for **canary deployments**.

---

# 7. VirtualService vs DestinationRule

Remember this:

### VirtualService

> **Where should traffic go?**

Example:

```text
90% → v1
10% → v2
```

### DestinationRule

> **What are the different versions/subsets of the destination?**

Example:

```text
payment
├── v1
└── v2
```

So:

```text
VirtualService
      ↓
Traffic routing

DestinationRule
      ↓
Destination policies/subsets
```

---

# 8. Demo 3 — Retry

Suppose payment service occasionally fails.

You can configure Istio:

```yaml
apiVersion: networking.istio.io/v1
kind: VirtualService

metadata:
  name: payment

spec:

  hosts:
    - payment

  http:

    - route:
        - destination:
            host: payment

      retries:
        attempts: 3
        perTryTimeout: 2s
```

Flow:

```text
App
 ↓
Envoy
 ↓
Payment
 ↓
FAIL
 ↓
Envoy retries
 ↓
Payment
```

The application doesn't need retry logic for this case.

---

# 9. Demo 4 — Timeout

```yaml
http:
  - route:
      - destination:
          host: payment

    timeout: 5s
```

Meaning:

```text
Request
   ↓
Payment
   ↓
5 seconds exceeded
   ↓
Timeout
```

This prevents requests from hanging indefinitely.

---

In an Istio service mesh, traffic routing decouples the Kubernetes Service concept from versioning. The **Kubernetes Service** acts as an internal DNS registry and port definition, the **VirtualService** defines *where and how* to route traffic (e.g., traffic splits, header matching), and the **DestinationRule** defines *subsets* (versions based on pod labels) and traffic policies (e.g., load balancing, TLS).

---

### End-to-End Traffic Flow Architecture

```
[ Internet Client ]
       │
       ▼
[ Istio Ingress Gateway ] ─── (Matches host / port)
       │
       ▼
[ Istio VirtualService ] ─── (Splits traffic: 80% weight to 'v1', 20% weight to 'v2')
       │
       ▼
[ Istio DestinationRule ] ── (Resolves subsets 'v1' and 'v2' to pod labels)
       │
       ├─────────────────────────┐
       ▼ (80% Traffic)           ▼ (20% Traffic)
[ Pods: app=my-app, version=v1 ] [ Pods: app=my-app, version=v2 ]

```

---

### Chronological Request Flow (Internet to Pod)

1. **Internet DNS Request:** The external client requests `app.example.com`, which resolves to the Public IP of the Cloud Load Balancer backing the `istio-ingressgateway`.
2. **Ingress Gateway (Envoy):** The gateway listens on port `80`/`443`, terminates external TLS, and evaluates the `Gateway` resource to determine if `app.example.com` is accepted.
3. **VirtualService Routing Evaluation:** The gateway matches the route to the `VirtualService` bound to it. The `VirtualService` reads the weight rules: **80%** to subset `v1` and **20%** to subset `v2`.
4. **DestinationRule Resolution:** The Envoy proxy looks up the `DestinationRule` for host `my-app-svc`. It maps subset `v1` to `version: v1` and subset `v2` to `version: v2`.
5. **Endpoint Routing (Envoy to Sidecar):** Istio's control plane (`istiod`) pushes the specific Pod IPs for each label subset directly into Envoy's cluster configurations. Envoy routes the connection directly to the target Pod's Envoy sidecar proxy, bypassing standard `kube-proxy` round-robin.
6. **Application Delivery:** The local Envoy sidecar proxy forwards the HTTP request to the container listening on port `8080`.

---

### Complete Kubernetes & Istio Manifests

#### 1. Deployments (v1 and v2)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-v1
  labels:
    app: my-app
    version: v1
spec:
  replicas: 2
  selector:
    matchLabels:
      app: my-app
      version: v1
  template:
    metadata:
      labels:
        app: my-app
        version: v1
    spec:
      containers:
      - name: my-app
        image: hashicorp/http-echo:latest
        args: ["-text=Hello from Version 1 (Stable)"]
        ports:
        - containerPort: 8080
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-v2
  labels:
    app: my-app
    version: v2
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-app
      version: v2
  template:
    metadata:
      labels:
        app: my-app
        version: v2
    spec:
      containers:
      - name: my-app
        image: hashicorp/http-echo:latest
        args: ["-text=Hello from Version 2 (Canary)"]
        ports:
        - containerPort: 8080

```

---

#### 2. Kubernetes Service

The Service targets the shared label `app: my-app`. It exposes an internal DNS name and port mapping.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app-svc
  labels:
    app: my-app
spec:
  ports:
  - name: http
    port: 80
    targetPort: 8080
  selector:
    app: my-app

```

---

#### 3. Istio Gateway

Configures the `istio-ingressgateway` to accept HTTP traffic for your domain.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: my-app-gateway
spec:
  selector:
    istio: ingressgateway # Selects default Istio ingress proxy
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "app.example.com"

```

---

#### 4. Istio DestinationRule (Defines Subsets)

The DestinationRule defines what "v1" and "v2" mean by mapping them to pod labels.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: my-app-dr
spec:
  host: my-app-svc.default.svc.cluster.local
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2

```

---

#### 5. Istio VirtualService (Canary 80/20 Traffic Split)

Splits incoming ingress requests across the subsets defined in the `DestinationRule`.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: my-app-vs
spec:
  hosts:
  - "app.example.com"
  gateways:
  - my-app-gateway
  http:
  - route:
    - destination:
        host: my-app-svc.default.svc.cluster.local
        subset: v1
      weight: 80
    - destination:
        host: my-app-svc.default.svc.cluster.local
        subset: v2
      weight: 20

```
