# Argocd_Canary

> **Argo CD itself does GitOps reconciliation; Argo Rollouts provides the progressive/canary rollout controller.**

So the common architecture is:

```text
                    Git
                     |
                     v
                Argo CD
             (GitOps sync)
                     |
                     v
              Rollout resource
                     |
                     v
              Argo Rollouts
             (progressive delivery)
                     |
          +----------+----------+
          |                     |
          v                     v
       Stable                Canary
        v1.0                  v2.0
          |                     |
       Pods                    Pods
          |                     |
          +----------+----------+
                     |
                     v
              Traffic Router
          (ALB / NGINX / Istio /
           Gateway API, etc.)
                     |
                     v
                   Users
```

Let's build a complete example.

---

# 1. Normal Deployment vs Canary Rollout

With a normal Deployment:

```text
v1.0
 ↓
Pod replaced
 ↓
v2.0
 ↓
Pod replaced
 ↓
v2.0
```

Kubernetes gradually replaces Pods, but you're not necessarily controlling **exactly how much user traffic** goes to v2.

With Canary:

```text
                    Traffic
                       |
             +---------+---------+
             |                   |
            90%                 10%
             |                   |
             v                   v
          Stable              Canary
           v1.0                v2.0
```

Then:

```text
90/10
  ↓
75/25
  ↓
50/50
  ↓
25/75
  ↓
0/100
```

---

# 2. What Argo Rollouts adds

A standard Kubernetes Deployment has:

```yaml
kind: Deployment
```

Argo Rollouts introduces:

```yaml
kind: Rollout
```

Instead of:

```text
Deployment
```

you have:

```text
Rollout
```

which understands:

```text
Canary
Blue-Green
Analysis
Traffic shifting
Pause
Promotion
Rollback
```

---

# 3. Architecture

Here's the production architecture I'd remember:

```text
                         Git
                          |
                          | commit v2.0
                          v
                    +-----------+
                    |  Argo CD  |
                    +-----+-----+
                          |
                          | Sync
                          v
                 +-------------------+
                 | Argo Rollout      |
                 | Controller        |
                 +---------+---------+
                           |
              +------------+------------+
              |                         |
              v                         v
        Stable Service            Canary Service
             |                         |
             v                         v
        +---------+                +---------+
        | v1.0    |                | v2.0    |
        | Pods    |                | Pods    |
        +---------+                +---------+
              \                         /
               \                       /
                +----------+----------+
                           |
                           v
                  Traffic Management
                 ALB / Istio / NGINX /
                    Gateway API
                           |
                           v
                         Users
```

---

# 4. Install Argo Rollouts

You need the Argo Rollouts controller installed in the cluster.

Then you normally use the Argo Rollouts CRD:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
```

You can manage that Rollout through Argo CD.

---

# 5. Simple Canary example

Let's say:

```text
Application = payment-api

Current = v1.0
New      = v2.0
```

Rollout:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout

metadata:
  name: payment-api

spec:

  replicas: 10

  revisionHistoryLimit: 3

  selector:
    matchLabels:
      app: payment-api

  template:
    metadata:
      labels:
        app: payment-api

    spec:
      containers:
        - name: payment-api
          image: company/payment-api:v2.0
          ports:
            - containerPort: 8080

  strategy:

    canary:

      canaryService: payment-api-canary
      stableService: payment-api-stable

      steps:

        - setWeight: 10

        - pause:
            duration: 5m

        - setWeight: 25

        - pause:
            duration: 5m

        - setWeight: 50

        - pause:
            duration: 10m

        - setWeight: 100
```

---

# 6. What does `setWeight` mean?

This is extremely important.

```yaml
- setWeight: 10
```

means:

> Send approximately **10% of traffic to the Canary** and keep the rest on Stable, assuming a compatible traffic-routing integration is configured.

So:

```text
Stable = 90%
Canary = 10%
```

Then:

```yaml
- setWeight: 25
```

becomes:

```text
Stable = 75%
Canary = 25%
```

Then:

```text
Stable = 50%
Canary = 50%
```

Finally:

```text
Stable = 0%
Canary = 100%
```

---

# 7. Why do we need two Services?

You'll commonly see:

```yaml
canaryService: payment-api-canary
stableService: payment-api-stable
```

These identify the stable and canary ReplicaSets/Pods.

Architecture:

```text
                    Rollout
                       |
            +----------+----------+
            |                     |
            v                     v
      Stable Service        Canary Service
            |                     |
            v                     v
         v1.0 Pods              v2.0 Pods
```

The Rollouts controller manages which ReplicaSet is considered:

```text
Stable
```

and which is:

```text
Canary
```

---

# 8. But Services alone don't give exact traffic percentages

This is an important interview point.

Suppose:

```text
Stable Service → 9 Pods
Canary Service → 1 Pod
```

That does **not** automatically mean:

```text
90% traffic → Stable
10% traffic → Canary
```

Pod count isn't the same thing as traffic percentage.

For precise traffic shifting, Argo Rollouts can integrate with traffic-management systems.

For example:

```text
ALB
NGINX
Istio
Traefik
Gateway API implementations
```

---

# 9. Example with Istio

Architecture:

```text
                     Users
                       |
                       v
                    Gateway
                       |
                       v
                  Istio / Envoy
                       |
              +--------+--------+
              |                 |
             90%               10%
              |                 |
              v                 v
           Stable            Canary
            v1.0              v2.0
```

Argo Rollouts changes the traffic weights.

For example:

```text
VirtualService:

stable = 90
canary = 10
```

Then:

```text
stable = 75
canary = 25
```

etc.

---

# 10. What happens when you push v2?

Initially:

```text
Stable
v1.0
10 Pods

Traffic:
100%
```

You modify Git:

```yaml
image: company/payment-api:v2.0
```

Argo CD detects:

```text
Git desired state
       |
       v
Argo CD
       |
       v
Rollout changed
```

Argo Rollouts then creates a new ReplicaSet.

```text
Old ReplicaSet
     |
     +-- v1.0

New ReplicaSet
     |
     +-- v2.0
```

---

# 11. Step 1 — 10% Canary

Argo Rollouts creates enough v2 Pods and shifts:

```text
                    Traffic
                       |
                 +-----+-----+
                 |           |
                90%         10%
                 |           |
                 v           v
               v1.0        v2.0
             Stable       Canary
```

Now you monitor:

```text
5xx
latency
CPU
memory
business errors
payment failures
```

---

# 12. Pause

Your rollout says:

```yaml
- pause:
    duration: 5m
```

So Rollout pauses for five minutes.

You can also have:

```yaml
- pause: {}
```

which means:

> Pause indefinitely until somebody promotes it.

For example:

```bash
kubectl argo rollouts promote payment-api
```

This is very useful for production approvals.

---

# 13. Step 2 — 25%

If the first stage looks good:

```text
90% stable
10% canary
```

becomes:

```text
75% stable
25% canary
```

Architecture:

```text
                Traffic
                   |
          +--------+--------+
          |                 |
         75%               25%
          |                 |
          v                 v
       v1.0               v2.0
```

---

# 14. Step 3 — 50%

Then:

```text
50% stable
50% canary
```

Now half your traffic is using v2.

This is a higher-risk stage, so you might have:

```yaml
- setWeight: 50

- pause:
    duration: 15m
```

and run deeper analysis.

---

# 15. Final promotion

Finally:

```yaml
- setWeight: 100
```

means:

```text
0% → Stable
100% → Canary
```

The canary revision becomes the new stable revision.

Conceptually:

```text
Before:

Stable = v1.0
Canary = v2.0

After promotion:

Stable = v2.0
Canary = none
```

---

# 16. What happens if v2 fails?

This is where Canary becomes really powerful.

Suppose:

```text
10% traffic → v2
```

and monitoring detects:

```text
HTTP 5xx = 8%
```

instead of:

```text
HTTP 5xx = 0.2%
```

You don't proceed to:

```text
25%
50%
100%
```

Instead:

```text
STOP
```

The rollout remains at the current step or aborts depending on configuration.

Then:

```text
Traffic
   |
   v
100% → v1.0
```

The old stable version remains available.

---

# 17. Automated rollback with Analysis

This is where Argo Rollouts becomes much more powerful.

You can define an:

```text
AnalysisTemplate
```

which asks:

> "Is this canary healthy?"

For example:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate

metadata:
  name: payment-api-analysis

spec:

  metrics:

    - name: error-rate

      interval: 1m

      count: 5

      successCondition: result[0] < 0.01

      provider:

        prometheus:
          address: http://prometheus.monitoring.svc:9090

          query: |
            sum(rate(http_requests_total{
              app="payment-api",
              status=~"5.."
            }[1m]))
            /
            sum(rate(http_requests_total{
              app="payment-api"
            }[1m]))
```

Conceptually:

```text
Prometheus
    |
    | metrics
    v
AnalysisTemplate
    |
    v
Argo Rollouts
    |
    +---- healthy → continue
    |
    +---- unhealthy → abort
```

---

# 18. Automated Canary architecture

Now we have the full production architecture:

```text
                         Git
                          |
                          v
                     +---------+
                     | Argo CD |
                     +----+----+
                          |
                          | Sync
                          v
                  +---------------+
                  | Argo Rollouts |
                  +-------+-------+
                          |
              +-----------+-----------+
              |                       |
              v                       v
        Stable Service          Canary Service
              |                       |
            v1.0                    v2.0
              |                       |
              +-----------+-----------+
                          |
                          v
                  Traffic Router
                 /   ALB / Istio  \
                /                  \
               v                    v
            Users              Production
                                  Traffic

                          ^
                          |
                     Metrics
                          |
                          v
                     Prometheus
                          |
                          v
                  AnalysisTemplate
                          |
                          v
                  Argo Rollouts
                    /         \
                   /           \
              PASS              FAIL
                |                 |
                v                 v
            Continue          Abort/Rollback
```

This is the architecture you should be able to draw in an interview.

---

# 19. Complete example with Analysis

A more realistic Rollout:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout

metadata:
  name: payment-api

spec:

  replicas: 10

  selector:
    matchLabels:
      app: payment-api

  template:

    metadata:
      labels:
        app: payment-api

    spec:
      containers:
        - name: payment-api
          image: company/payment-api:v2.0
          ports:
            - containerPort: 8080

  strategy:

    canary:

      stableService: payment-api-stable
      canaryService: payment-api-canary

      steps:

        - setWeight: 10

        - pause:
            duration: 5m

        - analysis:
            templates:
              - templateName: payment-api-analysis

        - setWeight: 25

        - pause:
            duration: 10m

        - analysis:
            templates:
              - templateName: payment-api-analysis

        - setWeight: 50

        - pause:
            duration: 15m

        - analysis:
            templates:
              - templateName: payment-api-analysis

        - setWeight: 100
```

---

# 20. The rollout lifecycle

You can memorize this:

```text
                  Git
                   |
                   v
                Argo CD
                   |
                   v
              Rollout v2
                   |
                   v
            Create Canary
                   |
                   v
                 10%
                   |
                   v
             Run Analysis
              /       \
            PASS       FAIL
             |           |
             v           v
            25%        ABORT
             |
             v
          Analysis
             |
            PASS
             |
             v
            50%
             |
             v
          Analysis
             |
            PASS
             |
             v
           100%
             |
             v
          PROMOTE
```

---

# 21. Argo CD's role vs Argo Rollouts' role

This distinction is **very important**.

### Argo CD

Responsible for:

```text
Git
 ↓
Desired state
 ↓
Kubernetes
```

It answers:

> **"What should be deployed?"**

---

### Argo Rollouts

Responsible for:

```text
How should the new version be released?
```

It answers:

> **"How do I safely move production traffic from v1 to v2?"**

So:

```text
Argo CD
   |
   | deploy desired Rollout configuration
   v
Argo Rollouts
   |
   | progressive delivery
   v
Production
```

---

# 22. Argo CD + Rollouts in Git

Your GitOps repository might look like:

```text
gitops/
│
├── applications/
│   └── payment.yaml
│
└── payment/
    │
    ├── rollout.yaml
    ├── service-stable.yaml
    ├── service-canary.yaml
    └── analysis-template.yaml
```

Argo CD Application:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application

metadata:
  name: payment-api

spec:

  source:
    repoURL: https://github.com/company/gitops.git
    path: payment
    targetRevision: main

  destination:
    server: https://kubernetes.default.svc
    namespace: payment
```

Then:

```text
Git
 |
 +-- rollout.yaml
 +-- services
 +-- analysis
       |
       v
     Argo CD
       |
       v
 Argo Rollouts
       |
       v
 Progressive deployment
```

---

# 23. Manual vs automated promotion

You can implement:

### Automatic

```text
10%
 ↓
5 minutes
 ↓
Analysis
 ↓
PASS
 ↓
25%
```

Everything happens automatically.

---

### Manual approval

```yaml
steps:
  - setWeight: 10

  - pause: {}
```

Then the rollout waits.

You inspect:

```text
Grafana
Prometheus
Dynatrace
Logs
Business metrics
```

Then:

```bash
kubectl argo rollouts promote payment-api
```

and it continues.

For a production payment application, this can be a good safety gate.

---

# 24. What if the canary fails?

There are two useful concepts:

### Abort

The rollout stops.

```text
v1.0 = stable
v2.0 = failed canary
```

Traffic returns/remains on stable according to the configured traffic routing.

### Rollback

The desired state is changed back to the previous version, typically through GitOps.

For example:

```text
Git:

v2.0
 ↓
rollback
 ↓
v1.0
```

Then Argo CD reconciles the desired state.

---

# 25. Canary vs Blue-Green in Argo Rollouts

Argo Rollouts supports both.

### Canary

```text
90% → v1
10% → v2

75% → v1
25% → v2

50% → v1
50% → v2

0% → v1
100% → v2
```

### Blue-Green

```text
BLUE = v1
GREEN = v2

100% → BLUE

        ↓ switch

100% → GREEN
```

The Rollout resource's strategy determines which model you use.

---

# 26. One subtle interview question

**Interviewer:**

> "If I use `setWeight: 10`, does Argo Rollouts simply create 1 Pod out of 10?"

**Answer:**

> "Not necessarily. Replica count and traffic weight are different concepts. Without a traffic-routing integration, Argo Rollouts can use ReplicaSet scaling as an approximation, but for precise traffic percentages I would integrate Rollouts with a traffic router such as Istio, NGINX, ALB or a Gateway API implementation. Then `setWeight: 10` represents traffic weighting rather than simply Pod count."

That's a **Senior-level answer**.

---

# 27. Another interview question

> **"Where does Argo CD stop and Argo Rollouts start?"**

Answer:

> "Argo CD is responsible for reconciling the desired Rollout resource from Git. Argo Rollouts watches that Rollout and controls the progressive delivery process—creating ReplicaSets, scaling stable/canary versions, shifting traffic, running analysis and deciding whether to continue, pause or abort."

---

# 28. Final mental model

Remember these four components:

```text
             ┌──────────────┐
             │     Git      │
             └──────┬───────┘
                    │
                    v
             ┌──────────────┐
             │   Argo CD    │
             │  "WHAT?"     │
             └──────┬───────┘
                    │
                    v
             ┌──────────────┐
             │    Rollout   │
             │  "HOW?"      │
             └──────┬───────┘
                    │
           +--------+--------+
           |                 |
           v                 v
        Stable             Canary
         v1.0               v2.0
           |                 |
           +--------+--------+
                    |
                    v
             Traffic Router
                    |
                    v
                  Users
                    |
                    v
                Metrics
                    |
                    v
               Prometheus
                    |
                    v
             AnalysisTemplate
                    |
                    v
              PASS / FAIL
```

### The one-liner to remember for your interview:

> **Argo CD manages the desired Rollout configuration from Git; Argo Rollouts performs the progressive delivery by gradually shifting traffic between stable and canary versions, optionally using traffic routing and automated metric analysis to decide whether to promote or abort the release.**
