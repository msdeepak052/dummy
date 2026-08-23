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
---
## Reconcilation file like nginx.conf analogy

## 1. First, the exact analogy

With NGINX Ingress:

```text
Ingress YAML
     │
     ▼
NGINX Ingress Controller
     │
     ▼
nginx.conf
     │
     ▼
NGINX
```

You can inspect `nginx.conf` and see what the controller actually generated.

With Istio:

```text
VirtualService
DestinationRule
Gateway
ServiceEntry
     │
     ▼
   Istiod
     │
     │  xDS
     ▼
Envoy sidecar / Istio Ingress Gateway
     │
     ▼
Listeners
Routes
Clusters
Endpoints
```

**There isn't one `istio.conf` file.**

Istiod dynamically generates and pushes Envoy configuration through **xDS**.

So the equivalent of:

```text
nginx.conf
```

is conceptually:

```text
Envoy's dynamic xDS configuration
```

And you inspect it with `istioctl`.

---

# 2. Let's build a real example

Suppose you have:

```text
                 Internet
                    │
                    ▼
             Istio Ingress Gateway
                    │
                    ▼
             app.example.com
                    │
              ┌─────┴─────┐
              │           │
              ▼           ▼
          frontend      api
                           │
                    /orders
                    /users
```

You create this `VirtualService`:

```yaml
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: app
  namespace: prod
spec:
  hosts:
    - app.example.com

  gateways:
    - istio-system/prod-gateway

  http:
    - match:
        - uri:
            prefix: /orders
      route:
        - destination:
            host: order-service
            port:
              number: 8080

    - match:
        - uri:
            prefix: /users
      route:
        - destination:
            host: user-service
            port:
              number: 8080
```

Conceptually:

```text
app.example.com/orders
          │
          ▼
     Istio Gateway
          │
          ▼
       Istiod
          │
          ▼
      Envoy route
          │
          ▼
    order-service:8080
```

And:

```text
app.example.com/users
          │
          ▼
     Istio Gateway
          │
          ▼
       Istiod
          │
          ▼
      Envoy route
          │
          ▼
     user-service:8080
```

---

# 3. Where does Istiod store this?

This is the important part.

You **do not** get:

```text
/etc/istio/istio.conf
```

Instead, Istiod watches Kubernetes resources:

```text
VirtualService
DestinationRule
Gateway
Service
Endpoints
ServiceEntry
...
```

and builds an internal model.

Then it sends configuration to Envoy.

```text
             Kubernetes API
                   │
                   │
        ┌──────────┼───────────┐
        │          │           │
 VirtualService DestinationRule Gateway
        │          │           │
        └──────────┼───────────┘
                   ▼
                 Istiod
                   │
          Configuration model
                   │
                 xDS
                   │
        ┌──────────┴──────────┐
        ▼                     ▼
 Envoy sidecar          Istio Gateway
```

---

# 4. What is xDS?

xDS is essentially the mechanism through which Istiod tells Envoy:

> "Here is how you should behave."

There are several important pieces:

```text
LDS → Listener Discovery Service
RDS → Route Discovery Service
CDS → Cluster Discovery Service
EDS → Endpoint Discovery Service
SDS → Secret Discovery Service
```

For your VirtualService example:

```text
VirtualService
       │
       ▼
     Istiod
       │
       ├── RDS → HTTP routes
       │
       ├── CDS → upstream clusters
       │
       └── EDS → actual endpoints
       │
       ▼
     Envoy
```

---

# 5. Now let's do the equivalent of checking nginx.conf

With NGINX you might do:

```bash
kubectl exec nginx-ingress-controller -n ingress-nginx -- \
  nginx -T
```

and see:

```text
server {
    server_name app.example.com;

    location /orders {
        proxy_pass http://order-service;
    }

    location /users {
        proxy_pass http://user-service;
    }
}
```

With Istio, instead of looking at one config file, you use:

```bash
istioctl proxy-config routes <pod> -n prod
```

For example:

```bash
istioctl proxy-config routes product-api-7d8f9c7d6f-xk92m -n prod
```

You might get something conceptually like:

```text
NAME          DOMAINS                MATCH                 VIRTUAL SERVICE
http.8080     app.example.com        /orders               app.prod
http.8080     app.example.com        /users                app.prod
```

Now you've confirmed:

```text
VirtualService
      ↓
Istiod
      ↓
RDS
      ↓
Envoy
```

---

# 6. Inspect the actual routes

You can get more detail:

```bash
istioctl proxy-config routes \
  product-api-7d8f9c7d6f-xk92m \
  -n prod \
  -o json
```

Conceptually you'll see something like:

```json
{
  "virtualHosts": [
    {
      "domains": [
        "app.example.com"
      ],
      "routes": [
        {
          "match": {
            "prefix": "/orders"
          },
          "route": {
            "cluster": "outbound|8080||order-service.prod.svc.cluster.local"
          }
        },
        {
          "match": {
            "prefix": "/users"
          },
          "route": {
            "cluster": "outbound|8080||user-service.prod.svc.cluster.local"
          }
        }
      ]
    }
  ]
}
```

This is **very similar conceptually to opening `nginx.conf`**.

You're asking:

> "What configuration did the controller actually give to the proxy?"

---

# 7. Now let's create the problem you were talking about

This is where the analogy becomes really useful.

Suppose engineer A creates:

```yaml
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: orders
  namespace: prod
spec:
  hosts:
    - app.example.com

  gateways:
    - istio-system/prod-gateway

  http:
    - match:
        - uri:
            prefix: /orders
      route:
        - destination:
            host: order-service
```

Later engineer B creates:

```yaml
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: payments
  namespace: prod
spec:
  hosts:
    - app.example.com

  gateways:
    - istio-system/prod-gateway

  http:
    - match:
        - uri:
            prefix: /orders
      route:
        - destination:
            host: payment-service
```

Now we have:

```text
VirtualService: orders
Host: app.example.com
Path: /orders

VirtualService: payments
Host: app.example.com
Path: /orders
```

This is the equivalent of the kind of **duplicate host/path ownership problem** you're familiar with from NGINX Ingress.

---

# 8. But Istio handles this differently

This is an important distinction.

Don't think:

> "Istiod simply concatenates all VirtualServices into one file."

It doesn't.

Istiod **merges/configures the Envoy model** based on the applicable resources.

The resulting route configuration can become ambiguous or surprising depending on how the VirtualServices overlap.

For example:

```text
app.example.com
       │
       ├── /orders → order-service
       │
       └── /orders → payment-service
```

Which one should win?

That's exactly the kind of configuration collision you want to investigate.

---

# 9. How do you troubleshoot it?

First:

```bash
kubectl get virtualservice -A
```

Example:

```text
NAMESPACE   NAME       GATEWAYS                    HOSTS
prod        orders     prod-gateway                app.example.com
prod        payments   prod-gateway                app.example.com
```

Then:

```bash
kubectl describe virtualservice orders -n prod
```

and:

```bash
kubectl describe virtualservice payments -n prod
```

Look for:

```text
hosts
gateways
http.match
http.route
```

---

# 10. Then inspect what Istiod actually gave Envoy

This is the **most important debugging step**.

Find the gateway pod:

```bash
kubectl get pods -n istio-system
```

For example:

```text
istio-ingressgateway-7d9f6f8b7c-x2k9p
```

Then:

```bash
istioctl proxy-config routes \
  istio-ingressgateway-7d9f6f8b7c-x2k9p \
  -n istio-system
```

You might see:

```text
NAME          DOMAINS          MATCH
http.80       app.example.com  /orders
http.80       app.example.com  /orders
```

Now you know:

```text
Kubernetes objects
       ↓
Istiod
       ↓
Generated Envoy configuration
       ↓
PROBLEM EXISTS HERE
```

---

# 11. Then inspect clusters

Suppose the route points to:

```text
order-service
```

Check:

```bash
istioctl proxy-config clusters \
  istio-ingressgateway-7d9f6f8b7c-x2k9p \
  -n istio-system
```

You might see:

```text
outbound|8080||order-service.prod.svc.cluster.local
outbound|8080||payment-service.prod.svc.cluster.local
```

This tells you that Istiod has created Envoy clusters for both destinations.

---

# 12. Then inspect endpoints

Now:

```bash
istioctl proxy-config endpoints \
  istio-ingressgateway-7d9f6f8b7c-x2k9p \
  -n istio-system
```

Example:

```text
CLUSTER                                      ENDPOINT
outbound|8080||order-service.prod.svc...     10.10.1.25:8080
                                               10.10.1.26:8080

outbound|8080||payment-service.prod.svc...   10.10.2.15:8080
                                               10.10.2.16:8080
```

Now you can trace the entire thing:

```text
VirtualService
       │
       ▼
     Istiod
       │
       ├──── RDS ────► Routes
       │
       ├──── CDS ────► Clusters
       │
       └──── EDS ────► Endpoints
                         │
                         ▼
                       Envoy
```

---

# 13. What about DestinationRule?

DestinationRule is slightly different.

Suppose:

```yaml
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
  name: order-service
  namespace: prod
spec:
  host: order-service

  subsets:
    - name: v1
      labels:
        version: v1

    - name: v2
      labels:
        version: v2
```

Then your VirtualService could say:

```yaml
route:
  - destination:
      host: order-service
      subset: v1
```

Now the flow is:

```text
VirtualService
       │
       │ "send /orders to
       │  order-service v1"
       ▼
     Istiod
       │
       ├───────────────┐
       │               │
       ▼               ▼
DestinationRule     Kubernetes
       │               │
       │               ▼
       │            Endpoints
       │
       ▼
     Envoy
```

And you can inspect:

```bash
istioctl proxy-config clusters <pod> -n prod
```

You might see:

```text
outbound|8080|v1|order-service.prod.svc.cluster.local
outbound|8080|v2|order-service.prod.svc.cluster.local
```

That's the equivalent of saying:

> "Show me what Istiod actually translated my DestinationRule into."

---

# 14. Your NGINX comparison is therefore very good

Think of it like this:

### NGINX

```text
Ingress
   ↓
Ingress Controller
   ↓
nginx.conf
   ↓
NGINX
```

You debug:

```bash
kubectl get ingress
kubectl describe ingress
kubectl exec ... -- nginx -T
```

### Istio

```text
VirtualService
DestinationRule
Gateway
   ↓
Istiod
   ↓
xDS
   ↓
Envoy configuration
```

You debug:

```bash
kubectl get virtualservice -A
kubectl describe virtualservice ...
kubectl get destinationrule -A
kubectl describe destinationrule ...
istioctl proxy-config routes ...
istioctl proxy-config clusters ...
istioctl proxy-config endpoints ...
```

---

# 15. And this becomes VERY important when Istiod is down

This is another place where your analogy needs a slight adjustment.

Suppose:

```text
              Istiod
                X
             DOWN
                │
                │
        Existing Envoy config
             remains
                │
                ▼
             Envoy
                │
                ▼
             Traffic
```

**Existing Envoy proxies generally continue using their last successfully received configuration.**

So:

> **Istiod being down does not automatically mean all Istio traffic immediately stops.**

Existing Envoy configuration is already loaded into Envoy.

However:

```text
Istiod DOWN
   │
   ├── Existing config → continues
   │
   ├── New VirtualService → won't be propagated
   │
   ├── Changed DestinationRule → won't be propagated
   │
   ├── New endpoints/config → won't be propagated
   │
   └── New proxy → may not receive required config
```

That's a **very important production troubleshooting concept**.

So if Istiod goes down:

```text
Existing configuration
        ↓
       Envoy
        ↓
   traffic can continue
```

But:

```text
New configuration
        ↓
      Istiod ❌
        ↓
   Envoy doesn't receive it
```

---

## The one sentence to remember

> **NGINX Ingress Controller reconciles Kubernetes Ingress objects into `nginx.conf`; Istiod reconciles Istio networking resources into Envoy's dynamic xDS configuration.**

And when you're troubleshooting Istio, don't look for an `istio.conf`.

Think:

```text
"Let me see what configuration Istiod actually pushed to this Envoy."
```

and use:

```bash
istioctl proxy-config routes <pod> -n <namespace>
istioctl proxy-config clusters <pod> -n <namespace>
istioctl proxy-config endpoints <pod> -n <namespace>
```

That is the **Istio equivalent of checking the generated `nginx.conf`**.

---

You're right. I mixed unrelated Prometheus/HPA context into your Istio question. I'll keep this **strictly to Istio/Istiod + VirtualService/DestinationRule**, using the NGINX Ingress analogy.

## 1. First, the exact analogy

With NGINX Ingress:

```text
Ingress YAML
     │
     ▼
NGINX Ingress Controller
     │
     ▼
nginx.conf
     │
     ▼
NGINX
```

You can inspect `nginx.conf` and see what the controller actually generated.

With Istio:

```text
VirtualService
DestinationRule
Gateway
ServiceEntry
     │
     ▼
   Istiod
     │
     │  xDS
     ▼
Envoy sidecar / Istio Ingress Gateway
     │
     ▼
Listeners
Routes
Clusters
Endpoints
```

**There isn't one `istio.conf` file.**

Istiod dynamically generates and pushes Envoy configuration through **xDS**.

So the equivalent of:

```text
nginx.conf
```

is conceptually:

```text
Envoy's dynamic xDS configuration
```

And you inspect it with `istioctl`.

---

# 2. Let's build a real example

Suppose you have:

```text
                 Internet
                    │
                    ▼
             Istio Ingress Gateway
                    │
                    ▼
             app.example.com
                    │
              ┌─────┴─────┐
              │           │
              ▼           ▼
          frontend      api
                           │
                    /orders
                    /users
```

You create this `VirtualService`:

```yaml
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: app
  namespace: prod
spec:
  hosts:
    - app.example.com

  gateways:
    - istio-system/prod-gateway

  http:
    - match:
        - uri:
            prefix: /orders
      route:
        - destination:
            host: order-service
            port:
              number: 8080

    - match:
        - uri:
            prefix: /users
      route:
        - destination:
            host: user-service
            port:
              number: 8080
```

Conceptually:

```text
app.example.com/orders
          │
          ▼
     Istio Gateway
          │
          ▼
       Istiod
          │
          ▼
      Envoy route
          │
          ▼
    order-service:8080
```

And:

```text
app.example.com/users
          │
          ▼
     Istio Gateway
          │
          ▼
       Istiod
          │
          ▼
      Envoy route
          │
          ▼
     user-service:8080
```

---

# 3. Where does Istiod store this?

This is the important part.

You **do not** get:

```text
/etc/istio/istio.conf
```

Instead, Istiod watches Kubernetes resources:

```text
VirtualService
DestinationRule
Gateway
Service
Endpoints
ServiceEntry
...
```

and builds an internal model.

Then it sends configuration to Envoy.

```text
             Kubernetes API
                   │
                   │
        ┌──────────┼───────────┐
        │          │           │
 VirtualService DestinationRule Gateway
        │          │           │
        └──────────┼───────────┘
                   ▼
                 Istiod
                   │
          Configuration model
                   │
                 xDS
                   │
        ┌──────────┴──────────┐
        ▼                     ▼
 Envoy sidecar          Istio Gateway
```

---

# 4. What is xDS?

xDS is essentially the mechanism through which Istiod tells Envoy:

> "Here is how you should behave."

There are several important pieces:

```text
LDS → Listener Discovery Service
RDS → Route Discovery Service
CDS → Cluster Discovery Service
EDS → Endpoint Discovery Service
SDS → Secret Discovery Service
```

For your VirtualService example:

```text
VirtualService
       │
       ▼
     Istiod
       │
       ├── RDS → HTTP routes
       │
       ├── CDS → upstream clusters
       │
       └── EDS → actual endpoints
       │
       ▼
     Envoy
```

---

# 5. Now let's do the equivalent of checking nginx.conf

With NGINX you might do:

```bash
kubectl exec nginx-ingress-controller -n ingress-nginx -- \
  nginx -T
```

and see:

```text
server {
    server_name app.example.com;

    location /orders {
        proxy_pass http://order-service;
    }

    location /users {
        proxy_pass http://user-service;
    }
}
```

With Istio, instead of looking at one config file, you use:

```bash
istioctl proxy-config routes <pod> -n prod
```

For example:

```bash
istioctl proxy-config routes product-api-7d8f9c7d6f-xk92m -n prod
```

You might get something conceptually like:

```text
NAME          DOMAINS                MATCH                 VIRTUAL SERVICE
http.8080     app.example.com        /orders               app.prod
http.8080     app.example.com        /users                app.prod
```

Now you've confirmed:

```text
VirtualService
      ↓
Istiod
      ↓
RDS
      ↓
Envoy
```

---

# 6. Inspect the actual routes

You can get more detail:

```bash
istioctl proxy-config routes \
  product-api-7d8f9c7d6f-xk92m \
  -n prod \
  -o json
```

Conceptually you'll see something like:

```json
{
  "virtualHosts": [
    {
      "domains": [
        "app.example.com"
      ],
      "routes": [
        {
          "match": {
            "prefix": "/orders"
          },
          "route": {
            "cluster": "outbound|8080||order-service.prod.svc.cluster.local"
          }
        },
        {
          "match": {
            "prefix": "/users"
          },
          "route": {
            "cluster": "outbound|8080||user-service.prod.svc.cluster.local"
          }
        }
      ]
    }
  ]
}
```

This is **very similar conceptually to opening `nginx.conf`**.

You're asking:

> "What configuration did the controller actually give to the proxy?"

---

# 7. Now let's create the problem you were talking about

This is where the analogy becomes really useful.

Suppose engineer A creates:

```yaml
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: orders
  namespace: prod
spec:
  hosts:
    - app.example.com

  gateways:
    - istio-system/prod-gateway

  http:
    - match:
        - uri:
            prefix: /orders
      route:
        - destination:
            host: order-service
```

Later engineer B creates:

```yaml
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: payments
  namespace: prod
spec:
  hosts:
    - app.example.com

  gateways:
    - istio-system/prod-gateway

  http:
    - match:
        - uri:
            prefix: /orders
      route:
        - destination:
            host: payment-service
```

Now we have:

```text
VirtualService: orders
Host: app.example.com
Path: /orders

VirtualService: payments
Host: app.example.com
Path: /orders
```

This is the equivalent of the kind of **duplicate host/path ownership problem** you're familiar with from NGINX Ingress.

---

# 8. But Istio handles this differently

This is an important distinction.

Don't think:

> "Istiod simply concatenates all VirtualServices into one file."

It doesn't.

Istiod **merges/configures the Envoy model** based on the applicable resources.

The resulting route configuration can become ambiguous or surprising depending on how the VirtualServices overlap.

For example:

```text
app.example.com
       │
       ├── /orders → order-service
       │
       └── /orders → payment-service
```

Which one should win?

That's exactly the kind of configuration collision you want to investigate.

---

# 9. How do you troubleshoot it?

First:

```bash
kubectl get virtualservice -A
```

Example:

```text
NAMESPACE   NAME       GATEWAYS                    HOSTS
prod        orders     prod-gateway                app.example.com
prod        payments   prod-gateway                app.example.com
```

Then:

```bash
kubectl describe virtualservice orders -n prod
```

and:

```bash
kubectl describe virtualservice payments -n prod
```

Look for:

```text
hosts
gateways
http.match
http.route
```

---

# 10. Then inspect what Istiod actually gave Envoy

This is the **most important debugging step**.

Find the gateway pod:

```bash
kubectl get pods -n istio-system
```

For example:

```text
istio-ingressgateway-7d9f6f8b7c-x2k9p
```

Then:

```bash
istioctl proxy-config routes \
  istio-ingressgateway-7d9f6f8b7c-x2k9p \
  -n istio-system
```

You might see:

```text
NAME          DOMAINS          MATCH
http.80       app.example.com  /orders
http.80       app.example.com  /orders
```

Now you know:

```text
Kubernetes objects
       ↓
Istiod
       ↓
Generated Envoy configuration
       ↓
PROBLEM EXISTS HERE
```

---

# 11. Then inspect clusters

Suppose the route points to:

```text
order-service
```

Check:

```bash
istioctl proxy-config clusters \
  istio-ingressgateway-7d9f6f8b7c-x2k9p \
  -n istio-system
```

You might see:

```text
outbound|8080||order-service.prod.svc.cluster.local
outbound|8080||payment-service.prod.svc.cluster.local
```

This tells you that Istiod has created Envoy clusters for both destinations.

---

# 12. Then inspect endpoints

Now:

```bash
istioctl proxy-config endpoints \
  istio-ingressgateway-7d9f6f8b7c-x2k9p \
  -n istio-system
```

Example:

```text
CLUSTER                                      ENDPOINT
outbound|8080||order-service.prod.svc...     10.10.1.25:8080
                                               10.10.1.26:8080

outbound|8080||payment-service.prod.svc...   10.10.2.15:8080
                                               10.10.2.16:8080
```

Now you can trace the entire thing:

```text
VirtualService
       │
       ▼
     Istiod
       │
       ├──── RDS ────► Routes
       │
       ├──── CDS ────► Clusters
       │
       └──── EDS ────► Endpoints
                         │
                         ▼
                       Envoy
```

---

# 13. What about DestinationRule?

DestinationRule is slightly different.

Suppose:

```yaml
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
  name: order-service
  namespace: prod
spec:
  host: order-service

  subsets:
    - name: v1
      labels:
        version: v1

    - name: v2
      labels:
        version: v2
```

Then your VirtualService could say:

```yaml
route:
  - destination:
      host: order-service
      subset: v1
```

Now the flow is:

```text
VirtualService
       │
       │ "send /orders to
       │  order-service v1"
       ▼
     Istiod
       │
       ├───────────────┐
       │               │
       ▼               ▼
DestinationRule     Kubernetes
       │               │
       │               ▼
       │            Endpoints
       │
       ▼
     Envoy
```

And you can inspect:

```bash
istioctl proxy-config clusters <pod> -n prod
```

You might see:

```text
outbound|8080|v1|order-service.prod.svc.cluster.local
outbound|8080|v2|order-service.prod.svc.cluster.local
```

That's the equivalent of saying:

> "Show me what Istiod actually translated my DestinationRule into."

---

# 14. Your NGINX comparison is therefore very good

Think of it like this:

### NGINX

```text
Ingress
   ↓
Ingress Controller
   ↓
nginx.conf
   ↓
NGINX
```

You debug:

```bash
kubectl get ingress
kubectl describe ingress
kubectl exec ... -- nginx -T
```

### Istio

```text
VirtualService
DestinationRule
Gateway
   ↓
Istiod
   ↓
xDS
   ↓
Envoy configuration
```

You debug:

```bash
kubectl get virtualservice -A
kubectl describe virtualservice ...
kubectl get destinationrule -A
kubectl describe destinationrule ...
istioctl proxy-config routes ...
istioctl proxy-config clusters ...
istioctl proxy-config endpoints ...
```

---

# 15. And this becomes VERY important when Istiod is down

This is another place where your analogy needs a slight adjustment.

Suppose:

```text
              Istiod
                X
             DOWN
                │
                │
        Existing Envoy config
             remains
                │
                ▼
             Envoy
                │
                ▼
             Traffic
```

**Existing Envoy proxies generally continue using their last successfully received configuration.**

So:

> **Istiod being down does not automatically mean all Istio traffic immediately stops.**

Existing Envoy configuration is already loaded into Envoy.

However:

```text
Istiod DOWN
   │
   ├── Existing config → continues
   │
   ├── New VirtualService → won't be propagated
   │
   ├── Changed DestinationRule → won't be propagated
   │
   ├── New endpoints/config → won't be propagated
   │
   └── New proxy → may not receive required config
```

That's a **very important production troubleshooting concept**.

So if Istiod goes down:

```text
Existing configuration
        ↓
       Envoy
        ↓
   traffic can continue
```

But:

```text
New configuration
        ↓
      Istiod ❌
        ↓
   Envoy doesn't receive it
```

---

## The one sentence to remember

> **NGINX Ingress Controller reconciles Kubernetes Ingress objects into `nginx.conf`; Istiod reconciles Istio networking resources into Envoy's dynamic xDS configuration.**

And when you're troubleshooting Istio, don't look for an `istio.conf`.

Think:

```text
"Let me see what configuration Istiod actually pushed to this Envoy."
```

and use:

```bash
istioctl proxy-config routes <pod> -n <namespace>
istioctl proxy-config clusters <pod> -n <namespace>
istioctl proxy-config endpoints <pod> -n <namespace>
```

That is the **Istio equivalent of checking the generated `nginx.conf`**.

---
Absolutely. Let's do **one complete Istio Canary Deployment demo**, from deploying v1/v2 → creating the Istio objects → sending traffic → seeing 90/10 distribution → changing it to 50/50.

I'll keep the focus strictly on **Istio + Envoy + Istiod + Kubernetes**.

# Demo 2 — Canary Deployment using Istio

## 1. What we want to build

We'll deploy:

```text
                    Client
                      │
                      │ HTTP
                      ▼
              Istio Ingress Gateway
                      │
                      ▼
                VirtualService
                  90% / 10%
                 ┌────┴────┐
                90%       10%
                 │          │
                 ▼          ▼
              app-v1     app-v2
```

Both versions use the same Kubernetes Service:

```text
app-service
    │
    ├── v1 Pods
    │
    └── v2 Pods
```

But Istio will control the traffic split.

---

# 2. End-to-end architecture

![Image](https://images.openai.com/static-rsc-4/-XM4IL9vqwNXLsRFaUA8aeyRECuUhdL9mvjjDe3l2TghXZ-rmJpbg2w31dUtnty7d1Cjf9qh40n9_OFzoklegxBJPgyd51NdKIocs_T6yCyY6I9m1WANXSVShe3f7yxOBJfRyJJOV18ZGoRpgN7wfsv2_eAMGtiri-UCKLFP5bwpGs_uU0adhNpC3RtIsi3Z?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/irfZZoK02NprDB62zUEdrInTwYm-gJhQ3EjwHJlgdgpzGXmPypYJFGKl82x9vnl1T2iIAxsk-O_r131ek5zq9JitfpNNZdTSpOt3ZUiyu546FjjUymVKlKGl1ZcQgVaEALSs1V8k3h8WBWaEy2w6YaPQAy6iWjmSWKnqY9ZHSz8oCju_SalG-Yl31Q8MmvAf?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/cAbU28hzEXJ3aCqhgLKX8s6IZjFL-q05rtsZIssLGTQz0qJayqdB2m4HdzWQnms5sik-4qJkAiT0B7wZGXKj41ErlGLjnf3zKtoVuV5aIodtI2bvTWraUE-jDxUwhcvaThZjNmHosC6E-6nUn9ky8jtz8ppqQwpFeLjndAyWhIac2t9zLUS8W_gfXhHzx99v?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/FRI9hyQA9SihXjOGl_32RsRfu1hGdSdK7WZ2UbGAjCnnopr_n0vI3slfzDKWLIbkwRKhXJN1By4_VMKD0GJQBIx81kjEE85wi_dmnffmcYWXGE7YBcF29gzVSuJaXd0kUW8c6OKpq0FvMhl3a0-KqS_rsTohk3_oZXDeokJ2fXU8w14gdiYVWQJukTGgFxSQ?purpose=fullsize)

The complete flow is:

```text
                    User
                     │
                     │ GET /
                     ▼
          ┌──────────────────────┐
          │ Istio IngressGateway │
          │       Envoy          │
          └──────────┬───────────┘
                     │
                     │ HTTP request
                     ▼
              VirtualService
                     │
             90%     │     10%
              ┌──────┴──────┐
              ▼             ▼
          subset:v1      subset:v2
              │             │
              ▼             ▼
        app-v1 Pods      app-v2 Pods
```

Now let's build it.

---

# 3. Prerequisites

You need:

```bash
kubectl
istioctl
```

and an Istio installation with an ingress gateway.

Check:

```bash
istioctl version
```

```bash
kubectl get pods -n istio-system
```

You should see something similar to:

```text
NAME                                    READY   STATUS
istiod-7c8f6c7d6f-x2k9p                1/1     Running
istio-ingressgateway-5d6f7c8f9d-x7k2m  1/1     Running
```

---

# 4. Create namespace

Create:

```bash
kubectl create namespace canary
```

Enable automatic sidecar injection:

```bash
kubectl label namespace canary istio-injection=enabled
```

Verify:

```bash
kubectl get namespace canary --show-labels
```

You should see:

```text
istio-injection=enabled
```

This means newly created pods will get:

```text
Application container
        +
Envoy sidecar
```

---

# 5. Deploy application v1

Create:

```yaml id="f5x8pj"
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-v1
  namespace: canary
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      version: v1
  template:
    metadata:
      labels:
        app: myapp
        version: v1
    spec:
      containers:
        - name: myapp
          image: hashicorp/http-echo:1.0
          args:
            - "-text=Hello from VERSION 1"
          ports:
            - containerPort: 5678
```

Apply:

```bash
kubectl apply -f myapp-v1.yaml
```

Check:

```bash
kubectl get pods -n canary -o wide
```

You should see:

```text
NAME                        READY
myapp-v1-7d8f9c7d6f-a1b2c   2/2
myapp-v1-7d8f9c7d6f-d4e5f   2/2
myapp-v1-7d8f9c7d6f-f6g7h   2/2
```

Why `2/2`?

```text
myapp container
+
Envoy sidecar
```

---

# 6. Deploy application v2

Now deploy the canary.

```yaml id="ub6sjy"
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-v2
  namespace: canary
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
      version: v2
  template:
    metadata:
      labels:
        app: myapp
        version: v2
    spec:
      containers:
        - name: myapp
          image: hashicorp/http-echo:1.0
          args:
            - "-text=Hello from VERSION 2"
          ports:
            - containerPort: 5678
```

Apply:

```bash
kubectl apply -f myapp-v2.yaml
```

Now:

```bash
kubectl get pods -n canary
```

Example:

```text
NAME                        READY
myapp-v1-7d8f9c7d6f-a1b2c   2/2
myapp-v1-7d8f9c7d6f-d4e5f   2/2
myapp-v1-7d8f9c7d6f-f6g7h   2/2
myapp-v2-5c9f7d8c8d-x8y9z   2/2
```

---

# 7. Create the Kubernetes Service

This Service selects:

```text
app: myapp
```

Notice that it does **NOT** select `version`.

```yaml id="ffas9o"
apiVersion: v1
kind: Service
metadata:
  name: myapp
  namespace: canary
spec:
  selector:
    app: myapp
  ports:
    - name: http
      port: 80
      targetPort: 5678
```

Apply:

```bash
kubectl apply -f service.yaml
```

Now Kubernetes sees:

```text
                  myapp Service
                       │
             selector: app=myapp
                       │
           ┌───────────┴───────────┐
           │                       │
           ▼                       ▼
       v1 Pods                  v2 Pod
      10.0.1.10                 10.0.2.20
      10.0.1.11
      10.0.1.12
```

This is important.

**Kubernetes Service itself isn't doing the 90/10 canary split.**

Istio will do that.

---

# 8. Create DestinationRule

Now we tell Istio that this service has two logical subsets.

```yaml id="bq2jti"
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
  name: myapp
  namespace: canary
spec:
  host: myapp.canary.svc.cluster.local

  subsets:
    - name: v1
      labels:
        version: v1

    - name: v2
      labels:
        version: v2
```

Apply:

```bash
kubectl apply -f destination-rule.yaml
```

Now Istiod understands:

```text
myapp
 │
 ├── subset v1
 │      └── version=v1
 │
 └── subset v2
        └── version=v2
```

---

# 9. Create Gateway

Now we need an entry point from outside.

```yaml id="9i4l2r"
apiVersion: networking.istio.io/v1
kind: Gateway
metadata:
  name: myapp-gateway
  namespace: canary
spec:
  selector:
    istio: ingressgateway

  servers:
    - port:
        number: 80
        name: http
        protocol: HTTP

      hosts:
        - myapp.example.com
```

Apply:

```bash
kubectl apply -f gateway.yaml
```

The Gateway says:

> Istio Ingress Gateway should accept HTTP traffic for `myapp.example.com`.

---

# 10. Create VirtualService — the actual canary logic

This is where the **90/10 traffic split** happens.

```yaml id="1e1mkt"
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: myapp
  namespace: canary
spec:
  hosts:
    - myapp.example.com

  gateways:
    - myapp-gateway

  http:
    - route:
        - destination:
            host: myapp.canary.svc.cluster.local
            subset: v1
          weight: 90

        - destination:
            host: myapp.canary.svc.cluster.local
            subset: v2
          weight: 10
```

Apply:

```bash
kubectl apply -f virtual-service.yaml
```

Now your architecture is:

```text
                         Client
                           │
                           │ myapp.example.com
                           ▼
                 Istio Ingress Gateway
                           │
                           ▼
                    VirtualService
                           │
                    ┌──────┴──────┐
                    │             │
                  90%            10%
                    │             │
                    ▼             ▼
                 subset v1     subset v2
                    │             │
                    ▼             ▼
                 v1 Pods        v2 Pod
```

---

# 11. Now understand WHO calls WHAT

This is the most important part.

When you run:

```bash
kubectl apply -f virtual-service.yaml
```

the flow is:

```text
kubectl
   │
   ▼
Kubernetes API Server
   │
   ▼
VirtualService stored in etcd
   │
   ▼
Istiod watches Kubernetes API
   │
   ▼
Istiod reads VirtualService
+
DestinationRule
+
Gateway
+
Services
+
Endpoints
   │
   ▼
Istiod builds Envoy configuration
   │
   ▼
xDS
   │
   ▼
Istio Ingress Gateway Envoy
```

**Your application does NOT call Istiod.**

Istiod pushes configuration **to Envoy**.

---

# 12. What happens when a request arrives?

Suppose you run:

```bash
curl http://myapp.example.com/
```

The request flow is:

```text
curl
 │
 ▼
Load Balancer
 │
 ▼
Istio Ingress Gateway
 │
 ▼
Envoy
 │
 │ checks route configuration
 ▼
VirtualService-derived route
 │
 ├──────── 90% ────────► subset v1
 │
 └──────── 10% ────────► subset v2
```

Suppose this request is selected for v2:

```text
Client
  │
  ▼
Ingress Gateway Envoy
  │
  │ 10%
  ▼
subset v2
  │
  ▼
myapp-v2
```

The application returns:

```text
Hello from VERSION 2
```

---

# 13. Verify the Envoy route

This is where the **NGINX `nginx.conf` analogy** becomes useful.

Find the gateway:

```bash
kubectl get pods -n istio-system
```

Then:

```bash
istioctl proxy-config routes \
  <istio-ingressgateway-pod> \
  -n istio-system
```

You should see a route associated with:

```text
myapp.example.com
```

For detailed output:

```bash
istioctl proxy-config routes \
  <istio-ingressgateway-pod> \
  -n istio-system \
  -o json
```

Conceptually, the generated Envoy route contains:

```text
myapp.example.com
       │
       ▼
 weighted clusters
       │
       ├── 90 → myapp subset v1
       │
       └── 10 → myapp subset v2
```

This is what **Istiod has translated your VirtualService into**.

---

# 14. Verify the clusters

Run:

```bash
istioctl proxy-config clusters \
  <istio-ingressgateway-pod> \
  -n istio-system
```

You should find clusters conceptually like:

```text
outbound|80|v1|myapp.canary.svc.cluster.local
outbound|80|v2|myapp.canary.svc.cluster.local
```

So:

```text
VirtualService
      │
      ▼
    RDS
      │
      ▼
Weighted routes

DestinationRule
      │
      ▼
    CDS
      │
      ▼
v1/v2 clusters
```

---

# 15. Verify the endpoints

Now:

```bash
istioctl proxy-config endpoints \
  <istio-ingressgateway-pod> \
  -n istio-system
```

Conceptually:

```text
CLUSTER                                      ENDPOINT

outbound|80|v1|myapp.canary.svc...          10.10.1.10:5678
                                             10.10.1.11:5678
                                             10.10.1.12:5678

outbound|80|v2|myapp.canary.svc...          10.10.2.10:5678
```

Now you have proved the entire chain:

```text
VirtualService
       │
       │ 90/10
       ▼
     Istiod
       │
       ├── RDS → weighted routes
       │
       ├── CDS → v1/v2 clusters
       │
       └── EDS → pod endpoints
       │
       ▼
Ingress Gateway Envoy
       │
       ├── 90% → v1
       │
       └── 10% → v2
```

---

# 16. Test the canary

Send 100 requests.

```bash
for i in {1..100}; do
  curl -s http://myapp.example.com/
  echo
done
```

You might see:

```text
Hello from VERSION 1
Hello from VERSION 1
Hello from VERSION 1
Hello from VERSION 2
Hello from VERSION 1
Hello from VERSION 1
...
```

Approximately:

```text
VERSION 1 → ~90 requests
VERSION 2 → ~10 requests
```

**It won't necessarily be exactly 90/10 for only 100 requests.** It's a traffic weighting, not a promise that every 10 requests gives exactly one v2 request.

---

# 17. Now increase the canary to 50%

Change:

```yaml id="ry3k6n"
http:
  - route:
      - destination:
          host: myapp.canary.svc.cluster.local
          subset: v1
        weight: 50

      - destination:
          host: myapp.canary.svc.cluster.local
          subset: v2
        weight: 50
```

Apply:

```bash
kubectl apply -f virtual-service.yaml
```

Now:

```text
                 VirtualService
                       │
                ┌──────┴──────┐
               50%           50%
                │              │
                ▼              ▼
             subset v1      subset v2
```

Istiod detects the change:

```text
VirtualService changed
       │
       ▼
Kubernetes API
       │
       ▼
     Istiod
       │
       ▼
new RDS configuration
       │
       ▼
Envoy Gateway
```

**You don't restart the Envoy gateway.**

That's one of the major benefits of Istio's dynamic configuration.

---

# 18. Finally move 100% to v2

Once you've validated the canary:

```yaml id="8n3i3a"
http:
  - route:
      - destination:
          host: myapp.canary.svc.cluster.local
          subset: v2
        weight: 100
```

Apply:

```bash
kubectl apply -f virtual-service.yaml
```

Now:

```text
Client
  │
  ▼
Istio Gateway
  │
  ▼
VirtualService
  │
  │ 100%
  ▼
subset v2
  │
  ▼
v2 Pods
```

Then you can remove v1:

```bash
kubectl delete deployment myapp-v1 -n canary
```

---

# 19. What exactly is happening inside Istiod?

This is the part I'd focus on for a **Senior Platform/DevOps interview**.

You have:

```text
VirtualService
DestinationRule
Gateway
Service
Endpoints
```

Istiod watches these resources.

It constructs the desired configuration for each Envoy proxy.

For our example:

```text
                   Istiod
                     │
       ┌─────────────┼──────────────┐
       │             │              │
       ▼             ▼              ▼
      RDS            CDS            EDS
       │             │              │
       ▼             ▼              ▼
   HTTP Routes     Clusters       Endpoints
       │             │              │
       └─────────────┼──────────────┘
                     │
                     ▼
                 Envoy Gateway
```

### RDS

Tells Envoy:

```text
myapp.example.com
/orders
/etc...

90% → cluster v1
10% → cluster v2
```

### CDS

Tells Envoy:

```text
cluster v1
cluster v2
```

### EDS

Tells Envoy:

```text
v1 → 10.10.1.10
      10.10.1.11
      10.10.1.12

v2 → 10.10.2.10
```

---

# 20. The entire demo in one picture

```text
                         USER
                          │
                          │ HTTP
                          ▼
               ┌─────────────────────┐
               │ Istio LoadBalancer   │
               │ Ingress Gateway      │
               │      Envoy           │
               └──────────┬──────────┘
                          │
                          │ RDS
                          ▼
                 ┌─────────────────┐
                 │ VirtualService  │
                 │                 │
                 │  v1 = 90%       │
                 │  v2 = 10%       │
                 └───────┬─────────┘
                         │
                 ┌───────┴────────┐
                 │                │
                90%              10%
                 │                │
                 ▼                ▼
        ┌────────────────┐ ┌────────────────┐
        │ DestinationRule│ │ DestinationRule│
        │    subset v1   │ │    subset v2   │
        └───────┬────────┘ └───────┬────────┘
                │                  │
                ▼                  ▼
          myapp-v1 Pods       myapp-v2 Pod
             3 replicas          1 replica


                    CONTROL PLANE
                         │
                         ▼
                       Istiod
                         │
          ┌──────────────┼──────────────┐
          │              │              │
          ▼              ▼              ▼
         RDS            CDS            EDS
       Routes         Clusters       Endpoints
          │              │              │
          └──────────────┼──────────────┘
                         │
                         │ xDS
                         ▼
                  Envoy Gateway
```

## The key interview answer

If they ask:

> **"How does Istio perform a canary deployment?"**

Say:

> "We deploy multiple versions of the application with different labels, define those versions as subsets using a DestinationRule, and use a VirtualService to assign traffic weights to those subsets. Istiod watches these resources, translates them into Envoy xDS configuration—primarily RDS for routing, CDS for clusters and EDS for endpoints—and pushes that configuration dynamically to the Envoy proxy. The proxy then performs the actual 90/10 or 50/50 traffic distribution."

And if they ask:

> **"Where can you verify that the configuration actually reached Envoy?"**

Use:

```bash
istioctl proxy-config routes <gateway-pod> -n istio-system
istioctl proxy-config clusters <gateway-pod> -n istio-system
istioctl proxy-config endpoints <gateway-pod> -n istio-system
```

That is the **Istio equivalent of checking the generated `nginx.conf`** in your NGINX Ingress troubleshooting workflow.

---

### One-line interview answer

> **"Istio is a Kubernetes service mesh that uses Envoy proxies as the data plane and Istiod as the control plane to provide service-to-service traffic management, mTLS, retries, timeouts, observability and authorization without requiring these capabilities to be implemented directly in applications."**
