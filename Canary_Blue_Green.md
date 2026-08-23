# **Canary vs Blue-Green** 

It is to focus on **how traffic moves between old and new versions**.

## Blue-Green

**Blue-Green deployment** is a release strategy where you run two identical production environments: one hosting the live version (**Blue**) and the other hosting the new version (**Green**). Once the Green environment is verified, a router or load balancer switches live user traffic from Blue to Green instantly.

**How the Workflow Works**

1. **Active State (Blue Live):** User traffic is routed entirely to the Blue environment running version `v1.0`.
2. **Deploy to Idle (Green):** Deploy version `v2.0` to the Green environment. Perform smoke tests, automated checks, and functional validation in isolation without impacting live users.
3. **Cutover:** Update the load balancer, ingress controller, or DNS to route incoming traffic from Blue to Green (`v2.0` becomes live).
4. **Monitoring & Rollback:** If errors spike, switch traffic back to Blue immediately with zero downtime. If stable, decommission or keep Blue on standby for the next cycle.

---
<img width="867" height="541" alt="image" src="https://github.com/user-attachments/assets/c1003e52-78f7-453d-8722-906593b29504" />

**Trade-offs & Considerations**

| Advantage | Trade-off / Challenge |
| --- | --- |
| **Zero Downtime:** Traffic switch is nearly instantaneous. | **Double Resource Cost:** Requires provisioning 2x infrastructure capacity during deployments. |
| **Instant Rollback:** Reverting to the old version takes a single routing update. | **Stateful / Database Schema Drift:** Schema migrations must maintain backward compatibility across both `v1` and `v2`. |
| **Production-Grade Testing:** Green environment can be tested with real production infrastructure before cutover. | **Session Handling:** Shared state or distributed caches (e.g., Redis) are needed to avoid dropping active sessions during the cutover. |

> **"Run the new version alongside the old version, then switch traffic."**

---

## Canary

You run old and new versions **at the same time**, but gradually shift traffic:

```text
                Load Balancer
                     |
          +----------+----------+
          |                     |
          v                     v
       v1.0                  v2.0
      90%                    10%
```

Then:

```text
90% / 10%
   ↓
75% / 25%
   ↓
50% / 50%
   ↓
10% / 90%
   ↓
0% / 100%
```

Canary is:

> **"Expose the new version to a small percentage of users first, observe it, then gradually increase traffic."**

---

# 2. Real-world example

Suppose you have:

```text
payment-api
```

Current version:

```text
v1.0
```

You developed:

```text
v2.0
```

You want to deploy v2.0 without risking all customers.

---

# 3. Traditional deployment

Normal rolling deployment might look like:

```text
10 Pods v1.0

Pod 1 → v2.0
Pod 2 → v2.0
Pod 3 → v2.0
...
```

Eventually:

```text
10 Pods v2.0
```

Traffic gradually moves because old Pods are replaced.

The problem is:

> You're not necessarily controlling **which percentage of user traffic** reaches v2.0.

---

# 4. Blue-Green Deployment

Create two environments.

```text
                  ALB
                   |
             Service / Router
                   |
          +--------+--------+
          |                 |
          v                 v
       BLUE              GREEN
       v1.0               v2.0
       10 Pods            10 Pods
```

Initially:

```text
ALB
 |
 +----100%----> BLUE v1.0

GREEN v2.0
 |
 +----0%----> traffic
```

You deploy v2.0 completely.

Then test it.

```text
GREEN
 |
 +-- health check
 +-- smoke test
 +-- integration test
 +-- performance test
```

If everything looks good:

```text
ALB
 |
 +----100%----> GREEN v2.0
```

---

# 5. Rollback in Blue-Green

This is the major advantage.

Suppose:

```text
BLUE  = v1.0
GREEN = v2.0
```

You switched:

```text
100% → GREEN
```

Then an error appears:

```text
5xx ↑
Latency ↑
Payment failures ↑
```

Rollback is:

```text
100% → BLUE
```

That's it.

```text
             ALB
              |
          100% traffic
              |
              v
           BLUE v1.0
```

You don't necessarily need to redeploy v1.0.

The old environment is already running.

---

# 6. Blue-Green Kubernetes example

You can implement a simple Blue-Green deployment using Services.

### Blue Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-blue
spec:
  replicas: 5

  selector:
    matchLabels:
      app: payment
      version: blue

  template:
    metadata:
      labels:
        app: payment
        version: blue

    spec:
      containers:
        - name: payment
          image: myrepo/payment:v1.0
          ports:
            - containerPort: 8080
```

---

### Green Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-green
spec:
  replicas: 5

  selector:
    matchLabels:
      app: payment
      version: green

  template:
    metadata:
      labels:
        app: payment
        version: green

    spec:
      containers:
        - name: payment
          image: myrepo/payment:v2.0
          ports:
            - containerPort: 8080
```

---

# 7. Service controls traffic

The Service initially selects:

```yaml
selector:
  app: payment
  version: blue
```

Therefore:

```text
Service
   |
   v
version=blue
   |
   v
v1.0 Pods
```

To switch to Green:

```yaml
selector:
  app: payment
  version: green
```

Now:

```text
Service
   |
   v
version=green
   |
   v
v2.0 Pods
```

That is a simple Kubernetes Blue-Green deployment.

---

# 8. Canary deployment

Now let's do the same deployment differently.

You have:

```text
v1.0 = 10 Pods
v2.0 = 1 Pod
```

Traffic:

```text
                 Service
                    |
             +------+------+
             |             |
             v             v
          v1.0           v2.0
         90%              10%
```

But there's an important point:

> **Kubernetes Service selectors alone don't give you reliable percentage-based traffic splitting.**

If you simply put:

```text
10 old Pods
1 new Pod
```

behind one Service, Kubernetes doesn't guarantee exactly:

```text
90% → old
10% → new
```

Traffic distribution depends on connection behavior, client behavior, kube-proxy implementation, etc.

For **real percentage-based canary**, use a traffic-management layer such as:

```text
Istio
NGINX Ingress
Gateway API implementations
AWS ALB weighted routing
Argo Rollouts
```

---

# 9. Canary with Argo Rollouts

Since you work with Argo CD, **Argo Rollouts** is especially useful to know.

You can define:

```yaml
strategy:
  canary:
    steps:
      - setWeight: 10
      - pause: {}
      - setWeight: 25
      - pause: {}
      - setWeight: 50
      - pause: {}
      - setWeight: 100
```

Conceptually:

```text
v1.0             v2.0
 90%              10%
                  ↑
               Canary
```

Then:

```text
90/10
  ↓
75/25
  ↓
50/50
  ↓
0/100
```

At every stage, you can analyze:

```text
Error rate
Latency
HTTP 5xx
CPU
Memory
Business metrics
```

---

# 10. Real production Canary

Suppose you have:

```text
100,000 requests/minute
```

You release:

```text
v2.0
```

Start:

```text
v1.0 = 99%
v2.0 = 1%
```

Observe:

```text
5xx = normal
latency = normal
CPU = normal
business KPI = normal
```

Then:

```text
v1.0 = 95%
v2.0 = 5%
```

Observe again.

Then:

```text
90 / 10
```

Then:

```text
75 / 25
```

Then:

```text
50 / 50
```

Eventually:

```text
0 / 100
```

---

# 11. What if Canary fails?

Suppose at:

```text
90% v1.0
10% v2.0
```

you observe:

```text
v2.0

5xx: 0.2% → 8%
latency: 100ms → 800ms
```

You stop the rollout.

```text
90% v1.0
10% v2.0
       X
       |
       v
     STOP
```

Then remove/reduce v2 traffic:

```text
100% → v1.0
```

This is a major benefit of Canary.

You discovered the problem with only a small portion of traffic exposed.

---

# 12. Blue-Green vs Canary

|                           | Blue-Green                    | Canary                   |
| ------------------------- | ----------------------------- | ------------------------ |
| Environments              | 2 complete versions           | Old + small new capacity |
| Initial new traffic       | 0%                            | Small %                  |
| Traffic switch            | Usually all at once           | Gradual                  |
| Risk                      | Higher during cutover         | Lower                    |
| Rollback                  | Very fast                     | Gradual/reverse traffic  |
| Infrastructure cost       | Higher                        | Usually lower            |
| Testing                   | Test full Green before switch | Test using real traffic  |
| Best for                  | Fast cutover/rollback         | Risk reduction           |
| Complexity                | Simpler                       | More complex             |
| Observability requirement | Moderate                      | High                     |

---

# 13. The visual difference

### Blue-Green

```text
             TRAFFIC
                |
                v
          +-----------+
          |   ALB     |
          +-----+-----+
                |
           100% traffic
                |
                v
         +-------------+
         | BLUE v1.0   |
         | 10 Pods     |
         +-------------+

         +-------------+
         | GREEN v2.0  |
         | 10 Pods     |
         +-------------+
              0%
```

Switch:

```text
             TRAFFIC
                |
                v
          +-----------+
          |   ALB     |
          +-----+-----+
                |
           100% traffic
                |
                v
         +-------------+
         | GREEN v2.0  |
         | 10 Pods     |
         +-------------+

         BLUE v1.0
         0%
```

---

### Canary

```text
             TRAFFIC
                |
                v
          +-----------+
          |   ALB     |
          +-----+-----+
                |
          +-----+------+
          |            |
         90%          10%
          |            |
          v            v
       v1.0           v2.0
      10 Pods         1 Pod
```

Then:

```text
       75%              25%
        |                |
        v                v
       v1.0             v2.0
```

Then:

```text
       50%              50%
        |                |
        v                v
       v1.0             v2.0
```

---

# 14. How this relates to Rolling Deployment

Don't confuse these three.

### Rolling

```text
10 v1
 ↓
9 v1 + 1 v2
 ↓
8 v1 + 2 v2
 ↓
7 v1 + 3 v2
...
 ↓
10 v2
```

You're gradually **replacing Pods**.

---

### Canary

```text
v1 → 90%
v2 → 10%

      ↓

v1 → 75%
v2 → 25%

      ↓

v1 → 50%
v2 → 50%
```

You're deliberately controlling **traffic exposure**.

---

### Blue-Green

```text
BLUE  = complete v1 environment
GREEN = complete v2 environment

        ↓

Traffic switches

BLUE  = 0%
GREEN = 100%
```

You're maintaining **two environments** and switching between them.

---

# 15. EKS example

Imagine:

```text
Internet
   |
   v
AWS ALB
   |
   v
Kubernetes Service
```

### Blue-Green

```text
ALB
 |
 v
Service
 |
 +----> BLUE Deployment
 |       v1.0
 |
 +----> GREEN Deployment
         v2.0
```

You switch the Service/Ingress routing from:

```text
BLUE
```

to:

```text
GREEN
```

---

### Canary

With a traffic-aware ingress/service mesh:

```text
ALB / Gateway
       |
       +------ 90% ------> v1.0
       |
       +------ 10% ------> v2.0
```

Then gradually:

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

# 16. When would I choose which?

### Choose Blue-Green when:

You need:

```text
Fast rollback
Simple cutover
Complete new environment testing
```

Example:

> Major application release where you want to test the entire new environment before sending production traffic.

---

### Choose Canary when:

You need:

```text
Low-risk release
Real production traffic validation
Gradual rollout
Automated metrics analysis
```

Example:

> Payment API where you want only 1% of users exposed initially and automatically stop if 5xx or latency increases.

---

# 17. Senior DevOps interview answer

If asked:

> **"What's the difference between Canary and Blue-Green deployment?"**

I'd answer:

> **"Blue-Green maintains two complete versions of the application. Initially all traffic goes to Blue, the old version, while Green is deployed and validated separately. Once Green is ready, traffic is switched to Green, giving us a very fast rollback by switching back to Blue.**
>
> **Canary instead exposes the new version to a small percentage of real production traffic. For example, I might start with 99% v1 and 1% v2, monitor error rate, latency and business metrics, then progressively move to 5%, 10%, 25%, 50% and finally 100%. If the metrics degrade, I stop or roll back the canary.**
>
> **So Blue-Green optimizes for fast cutover and rollback, while Canary optimizes for reducing release risk through gradual exposure."**

### One sentence to memorize:

> **Blue-Green = two complete environments + traffic switch. Canary = gradual traffic exposure + progressive rollout.**
