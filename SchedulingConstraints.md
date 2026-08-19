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
