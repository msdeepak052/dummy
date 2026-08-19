# Pod Scheduling Constraints

The easiest mental model:

```text
Node Affinity
    ↓
"I WANT to run on these nodes"

Node Anti-Affinity
    ↓
"I DON'T WANT to run near these Pods"
```

There are actually **two major types** you should know:

```text
                    Affinity
                       |
          +------------+------------+
          |                         |
     Node Affinity            Pod Affinity
          |                         |
  Based on node labels       Based on other Pods


                 Anti-Affinity
                       |
          +------------+------------+
          |                         |
   Node Anti-Affinity        Pod Anti-Affinity
```

Let's go through each.

---

# 1. Node Affinity

Node affinity tells the scheduler:

> **"Schedule my Pod on nodes having certain labels."**

Suppose your nodes are:

```text
node-1
  env=prod

node-2
  env=dev

node-3
  env=prod
```

You can label them:

```bash
kubectl label node node-1 env=prod
kubectl label node node-2 env=dev
kubectl label node node-3 env=prod
```

Now your Pod can say:

```text
I want env=prod nodes.
```

---

# 2. Basic Node Affinity example

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: prod-app
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: env
                operator: In
                values:
                  - prod

  containers:
    - name: nginx
      image: nginx
```

The important part:

```yaml
requiredDuringSchedulingIgnoredDuringExecution:
```

means:

> This requirement must be satisfied when scheduling the Pod.

So:

```text
              Pod
               |
               | Node Affinity
               |
               v
          env=prod ?
           /     \
         YES      NO
          |        |
          v        X
       Schedule   Reject
```

---

# 3. `required` vs `preferred`

This is very important for interviews.

## Required

```yaml
requiredDuringSchedulingIgnoredDuringExecution:
```

Means:

> **Hard requirement**

If no matching node exists:

```text
Pod
 |
 v
No matching node
 |
 v
Pending
```

---

## Preferred

```yaml
preferredDuringSchedulingIgnoredDuringExecution:
```

Means:

> **Soft preference**

Kubernetes tries to satisfy it, but can schedule elsewhere if necessary.

Example:

```yaml
affinity:
  nodeAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        preference:
          matchExpressions:
            - key: env
              operator: In
              values:
                - prod
```

Think:

```text
required  = MUST
preferred = SHOULD
```

---

# 4. `IgnoredDuringExecution`

You will notice:

```text
requiredDuringSchedulingIgnoredDuringExecution
```

The second part means:

```text
IgnoredDuringExecution
```

Suppose the Pod is already running on:

```text
node-1
env=prod
```

Later someone changes the label:

```text
env=dev
```

The existing Pod isn't automatically evicted because its affinity requirement is ignored **during execution**.

So:

```text
Scheduling:
   affinity matters ✅

After Pod is running:
   label changes
       |
       v
   Pod remains
```

This is a common interview detail.

---

# 5. Node Anti-Affinity

Strictly speaking, Kubernetes doesn't have a separate `nodeAntiAffinity` field.

You express it using **nodeAffinity with `NotIn` / `DoesNotExist`**.

Example:

> Don't run this Pod on nodes labeled `env=dev`.

```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
        - matchExpressions:
            - key: env
              operator: NotIn
              values:
                - dev
```

So:

```text
env=prod  → ✅
env=stage → ✅
env=dev   → ❌
```

---

# 6. Pod Affinity

Now things become more interesting.

**Pod affinity** means:

> "I want to run close to another Pod."

For example:

```text
Frontend Pod
      |
      | wants to be near
      v
Backend Pod
```

Why?

Maybe your frontend communicates heavily with backend and you want them in the same topology zone.

Suppose backend Pods have:

```yaml
labels:
  app: backend
```

Frontend can say:

```text
Place me near Pods with app=backend.
```

---

# 7. Pod Affinity example

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: frontend
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchExpressions:
              - key: app
                operator: In
                values:
                  - backend

          topologyKey: kubernetes.io/hostname

  containers:
    - name: frontend
      image: nginx
```

The important part:

```yaml
topologyKey: kubernetes.io/hostname
```

means:

> Put the frontend on the same node as the matching backend Pod.

Architecture:

```text
Node 1
+----------------------+
| Backend Pod          |
| app=backend          |
|                      |
| Frontend Pod         |
+----------------------+

Node 2
+----------------------+
| Backend Pod          |
+----------------------+
```

The scheduler prefers/requires the frontend to be on the same node as the backend depending on the affinity rule.

---

# 8. What is `topologyKey`?

This is very important.

`topologyKey` tells Kubernetes **what "close" means**.

For example:

```yaml
topologyKey: kubernetes.io/hostname
```

means:

> Same node.

But:

```yaml
topologyKey: topology.kubernetes.io/zone
```

means:

> Same availability zone.

And:

```yaml
topologyKey: kubernetes.io/hostname
```

is roughly:

```text
Node-level relationship
```

while:

```yaml
topology.kubernetes.io/zone
```

is:

```text
AZ-level relationship
```

---

# 9. Pod Anti-Affinity

This is probably one of the most useful production patterns.

Pod anti-affinity means:

> **"Don't place me near another matching Pod."**

Suppose you have:

```text
myapp
myapp
myapp
```

You don't want all replicas on the same node.

Why?

Because if the node dies:

```text
Node 1
+----------------+
| myapp-1        |
| myapp-2        |
| myapp-3        |
+----------------+

Node 1 dies
    |
    v
ALL replicas gone
```

Instead:

```text
Node 1             Node 2             Node 3
+---------+        +---------+        +---------+
| myapp-1 |        | myapp-2 |        | myapp-3 |
+---------+        +---------+        +---------+
```

That's **Pod anti-affinity**.

---

# 10. Pod Anti-Affinity example

Suppose all Pods have:

```yaml
labels:
  app: myapp
```

Deployment:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3

  selector:
    matchLabels:
      app: myapp

  template:
    metadata:
      labels:
        app: myapp

    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchExpressions:
                  - key: app
                    operator: In
                    values:
                      - myapp

              topologyKey: kubernetes.io/hostname

      containers:
        - name: nginx
          image: nginx
```

The scheduler now tries to ensure:

```text
Same hostname
+
same app=myapp
```

doesn't happen.

So:

```text
Node 1
  myapp-1

Node 2
  myapp-2

Node 3
  myapp-3
```

instead of:

```text
Node 1
  myapp-1
  myapp-2
  myapp-3
```

---

# 11. Why is this important in production?

Imagine:

```text
3 replicas
3 worker nodes
```

Without anti-affinity:

```text
Node A
  app-1
  app-2
  app-3

Node B
  nothing

Node C
  nothing
```

Node A fails:

```text
Node A ❌

app-1 ❌
app-2 ❌
app-3 ❌
```

With anti-affinity:

```text
Node A        Node B        Node C
app-1         app-2         app-3
```

Node A fails:

```text
Node A ❌

app-1 ❌

app-2 ✅
app-3 ✅
```

The Deployment controller recreates `app-1` elsewhere.

This improves **availability and failure isolation**.

---

# 12. Affinity vs Anti-Affinity

Think about the question you're asking.

### Node Affinity

> Which **nodes** do I want?

```text
Pod
 ↓
Node labels
```

Example:

```text
Run GPU workloads on GPU nodes.
```

---

### Pod Affinity

> Which **Pods** do I want to be close to?

```text
Pod
 ↓
Other Pod labels
```

Example:

```text
Run frontend close to backend.
```

---

### Pod Anti-Affinity

> Which **Pods** do I want to stay away from?

```text
Pod
 ↓
Other Pod labels
```

Example:

```text
Don't put two replicas on the same node.
```

---

# 13. Taints vs Affinity

This is another interview favorite.

### Taint/Toleration

The **node controls access**.

```text
Node:
"Don't come here unless you tolerate my taint."
```

### Node Affinity

The **Pod controls preference/requirement**.

```text
Pod:
"I want to run on nodes with this label."
```

So:

```text
Taint
  ↓
Node says NO

Toleration
  ↓
Pod says "I'm allowed"

Node Affinity
  ↓
Pod says "I WANT this node"
```

---

# 14. Real production example

Suppose your EKS cluster has:

```text
Node Group 1
General workloads

Node Group 2
GPU workloads

Node Group 3
Critical workloads
```

You can use:

### GPU node label

```bash
kubectl label node gpu-node-1 workload=gpu
```

### GPU node taint

```bash
kubectl taint node gpu-node-1 workload=gpu:NoSchedule
```

Then GPU Pod uses:

```yaml
tolerations:
  - key: workload
    operator: Equal
    value: gpu
    effect: NoSchedule

affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
        - matchExpressions:
            - key: workload
              operator: In
              values:
                - gpu
```

Now you have:

```text
                 GPU Pod
                    |
          +---------+---------+
          |                   |
     Toleration          Node Affinity
          |                   |
   "I'm allowed"        "I want GPU node"
          |                   |
          +---------+---------+
                    |
                    v
                GPU Node
```

This is a **very strong real-world scheduling pattern**.

---

# 15. Affinity strength cheat sheet

Remember these four:

```text
requiredDuringSchedulingIgnoredDuringExecution
        ↓
       MUST

preferredDuringSchedulingIgnoredDuringExecution
        ↓
      SHOULD
```

For Pod affinity:

```text
podAffinity
        ↓
"I want to be close"

podAntiAffinity
        ↓
"I want to be separated"
```

---
Absolutely — here it is in the usual chat format, clean and interview-friendly.

# 16. Topology Spread Constraints

**Topology Spread Constraints** are used to control **how Pods are distributed across failure domains**, such as:

* Nodes
* Availability Zones
* Regions

The main goal is:

> **Keep replicas evenly distributed instead of allowing them to become concentrated in one topology.**

---

## 1. Why do we need Topology Spread Constraints?

Suppose you have 3 replicas:

```text
Pod 1
Pod 2
Pod 3
```

And your cluster has 3 Availability Zones:

```text
Zone A
Zone B
Zone C
```

Without any constraint, Kubernetes might schedule:

```text
Zone A → Pod 1
Zone A → Pod 2
Zone A → Pod 3

Zone B → Nothing
Zone C → Nothing
```

That's bad for availability.

If **Zone A goes down**, all replicas disappear.

With topology spread constraints, you can tell Kubernetes:

> "Spread my Pods across the zones as evenly as possible."

Ideally:

```text
Zone A → Pod 1
Zone B → Pod 2
Zone C → Pod 3
```

---

# 2. Basic Example

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3

  selector:
    matchLabels:
      app: myapp

  template:
    metadata:
      labels:
        app: myapp

    spec:
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule

          labelSelector:
            matchLabels:
              app: myapp

      containers:
        - name: nginx
          image: nginx
```

The important part is:

```yaml
topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app: myapp
```

---

# 3. Understanding `topologyKey`

This tells Kubernetes **what topology domain to spread across**.

For Availability Zones:

```yaml
topologyKey: topology.kubernetes.io/zone
```

For nodes:

```yaml
topologyKey: kubernetes.io/hostname
```

So:

### Spread across AZs

```yaml
topologyKey: topology.kubernetes.io/zone
```

Example:

```text
AZ-A → 2 Pods
AZ-B → 2 Pods
AZ-C → 1 Pod
```

### Spread across nodes

```yaml
topologyKey: kubernetes.io/hostname
```

Example:

```text
Node-1 → 2 Pods
Node-2 → 1 Pod
Node-3 → 1 Pod
```

---

# 4. What is `maxSkew`?

This is the **maximum allowed difference** between the most-loaded topology domain and the least-loaded topology domain.

Formula:

```text
skew = maximum Pods - minimum Pods
```

Suppose:

```text
Zone A → 2 Pods
Zone B → 1 Pod
Zone C → 1 Pod
```

Then:

```text
maximum = 2
minimum = 1

skew = 2 - 1
     = 1
```

Therefore:

```yaml
maxSkew: 1
```

is satisfied.

---

## Another example

Suppose:

```text
Zone A → 3 Pods
Zone B → 1 Pod
Zone C → 1 Pod
```

Then:

```text
3 - 1 = 2
```

If:

```yaml
maxSkew: 1
```

then the distribution violates the desired skew.

Kubernetes will try to avoid making the distribution that uneven.

---

# 5. `whenUnsatisfiable`

This tells Kubernetes what to do if placing a Pod would violate the topology constraint.

There are two important values.

## `DoNotSchedule`

```yaml
whenUnsatisfiable: DoNotSchedule
```

Meaning:

> Don't schedule the Pod if doing so would violate the constraint.

The Pod remains:

```text
Pending
```

This is a **hard scheduling constraint**.

---

## `ScheduleAnyway`

```yaml
whenUnsatisfiable: ScheduleAnyway
```

Meaning:

> Try to spread the Pods, but schedule the Pod anyway if perfect spreading isn't possible.

This is more like a **soft preference**.

---

# 6. `labelSelector`

This tells Kubernetes:

> "Which Pods should be considered when calculating the distribution?"

Example:

```yaml
labelSelector:
  matchLabels:
    app: myapp
```

Kubernetes looks at Pods having:

```yaml
app: myapp
```

and calculates their distribution across the topology domains.

For example:

```text
Zone A → myapp Pod
Zone B → myapp Pod
Zone C → myapp Pod
```

---

# 7. Complete Example

Imagine an EKS cluster:

```text
                 EKS Cluster
                     |
       +-------------+-------------+
       |             |             |
     AZ-A           AZ-B          AZ-C
       |             |             |
    Node-A1       Node-B1       Node-C1
       |             |             |
     Pod-1         Pod-2         Pod-3
```

Deployment:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 6

  selector:
    matchLabels:
      app: web

  template:
    metadata:
      labels:
        app: web

    spec:
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: web

      containers:
        - name: nginx
          image: nginx:1.27
```

Kubernetes tries to achieve something like:

```text
AZ-A → 2 Pods
AZ-B → 2 Pods
AZ-C → 2 Pods
```

Distribution:

```text
2 / 2 / 2
```

Skew:

```text
2 - 2 = 0
```

Perfectly balanced.

---

# 8. What happens when a zone already has more Pods?

Suppose the current distribution is:

```text
AZ-A → 3
AZ-B → 2
AZ-C → 2
```

Skew:

```text
3 - 2 = 1
```

Still satisfies:

```yaml
maxSkew: 1
```

But suppose Kubernetes wants to place another Pod.

If it places it in AZ-A:

```text
AZ-A → 4
AZ-B → 2
AZ-C → 2
```

Now:

```text
4 - 2 = 2
```

That violates:

```yaml
maxSkew: 1
```

With:

```yaml
whenUnsatisfiable: DoNotSchedule
```

Kubernetes won't place that Pod in AZ-A.

It will try another eligible topology domain.

---

# 9. Topology Spread vs Pod Anti-Affinity

This is a **very important interview comparison**.

### Pod Anti-Affinity

You basically say:

> "Don't put these Pods together."

Example:

```yaml
podAntiAffinity:
  requiredDuringSchedulingIgnoredDuringExecution:
    - labelSelector:
        matchLabels:
          app: myapp
      topologyKey: kubernetes.io/hostname
```

This can enforce:

```text
Node 1 → Pod 1
Node 2 → Pod 2
Node 3 → Pod 3
```

---

### Topology Spread

You say:

> **"Keep these Pods evenly distributed."**

Example:

```yaml
topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app: myapp
```

This gives you much more control over **how balanced** the distribution should be.

---

# 10. Simple way to remember

### Pod Affinity

> **Put Pods together.**

```text
A → Pod 1
A → Pod 2
```

### Pod Anti-Affinity

> **Keep Pods apart.**

```text
Node 1 → Pod 1
Node 2 → Pod 2
```

### Topology Spread Constraints

> **Keep Pods evenly distributed.**

```text
Zone A → 2
Zone B → 2
Zone C → 2
```

---

# Interview Answer

If the interviewer asks:

**"What are topology spread constraints?"**

A good answer is:

> **Topology spread constraints are Kubernetes scheduling constraints used to control the distribution of Pods across topology domains such as nodes or Availability Zones. We use `topologyKey` to define the topology, `maxSkew` to define how much imbalance is acceptable, and `whenUnsatisfiable` to determine whether Kubernetes should reject or tolerate an uneven placement. They are useful for improving high availability and preventing replicas from becoming concentrated in a single node or AZ.**

The key mental model is:

```text
Affinity
   ↓
Where SHOULD my Pod go?

Anti-Affinity
   ↓
Where SHOULD my Pod NOT go?

Topology Spread
   ↓
How EVENLY should my Pods be distributed?
```

---

# 16. Final mental model

```text
                         Kubernetes Scheduler
                                  |
                 +----------------+----------------+
                 |                                 |
             Node rules                         Pod rules
                 |                                 |
        +--------+--------+                +-------+-------+
        |                 |                |               |
     Affinity        Taints           Pod Affinity    Pod Anti-Affinity
        |                 |                |               |
   "I want these      "Don't come      "Stay close"    "Stay apart"
      nodes"             here"          to these Pods"   from these Pods
```

And in a real production deployment:

```text
                     Application Pod
                           |
             +-------------+-------------+
             |             |             |
             v             v             v
       Node Affinity   Toleration   Pod Anti-Affinity
             |             |             |
             |             |             |
             v             v             v
        Select node     Allow node    Spread replicas
             |             |             |
             +-------------+-------------+
                           |
                           v
                     Desired Node
```

### The interview answer to memorize

> **Node affinity controls which nodes a Pod should or must run on based on node labels. Pod affinity/anti-affinity controls placement relative to other Pods using their labels and a topology domain. `required` is a hard requirement, while `preferred` is a soft preference. Affinity is commonly combined with taints/tolerations for dedicated node groups and pod anti-affinity for spreading replicas across nodes or zones.**
