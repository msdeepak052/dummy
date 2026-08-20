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

# 10. Istio Gateway

Istio also provides ingress traffic management.

```text
Internet
   |
   v
Istio Gateway
   |
   v
VirtualService
   |
   +------ payment
   |
   +------ order
```

Example:

```yaml
apiVersion: networking.istio.io/v1
kind: Gateway

metadata:
  name: app-gateway

spec:

  selector:
    istio: ingressgateway

  servers:

    - port:
        number: 80
        name: http
        protocol: HTTP

      hosts:
        - "example.com"
```

Then:

```yaml
apiVersion: networking.istio.io/v1
kind: VirtualService

metadata:
  name: payment

spec:

  hosts:
    - "example.com"

  gateways:
    - app-gateway

  http:

    - match:
        - uri:
            prefix: /payment

      route:
        - destination:
            host: payment
            port:
              number: 80
```

Flow:

```text
Internet
   ↓
Istio Gateway
   ↓
VirtualService
   ↓
/payment
   ↓
payment-service
```

---

# 11. Istio vs Ingress

Don't confuse them.

```text
Ingress
   ↓
Primarily north-south traffic
   ↓
Internet → Cluster
```

Istio:

```text
Istio
├── North-South
│      Internet → Cluster
│
└── East-West
       Service → Service
```

The big Istio value is **east-west service-to-service traffic**.

---

# 12. Most important Istio objects

For interviews, remember:

```text
Istiod
   ↓
Control plane

Envoy
   ↓
Data plane

VirtualService
   ↓
Traffic routing

DestinationRule
   ↓
Subsets + destination policies

Gateway
   ↓
Ingress traffic

PeerAuthentication
   ↓
mTLS

AuthorizationPolicy
   ↓
Who can communicate with whom
```

### One-line interview answer

> **"Istio is a Kubernetes service mesh that uses Envoy proxies as the data plane and Istiod as the control plane to provide service-to-service traffic management, mTLS, retries, timeouts, observability and authorization without requiring these capabilities to be implemented directly in applications."**
