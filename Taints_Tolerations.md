# Taints & Tolerations in Kubernetes

The easiest way to remember:

> **Taint = Node says "Don't schedule Pods here."**
> **Toleration = Pod says "I'm allowed to be scheduled on a node with this taint."**

But one important point:

> **A toleration does NOT force a Pod onto that node.** It only makes the Pod *eligible* to run there.

---

# 1. Why do we need taints?

Suppose you have:

```text
Kubernetes Cluster

Node 1 → normal workloads
Node 2 → GPU node
Node 3 → production-only node
```

You don't want random workloads landing on Node 2:

```text
             GPU Node
                |
       +--------+--------+
       |        |        |
     GPU Pod  nginx    random Pod
```

You want:

```text
GPU Node
   |
   +--- GPU workloads only
```

That's where **taints and tolerations** help.

---

# 2. Taint a node

Suppose:

```text
node-2
```

is dedicated to GPU workloads.

Add a taint:

```bash
kubectl taint nodes node-2 gpu=true:NoSchedule
```

Now:

```text
node-2
   |
   +-- taint:
       gpu=true:NoSchedule
```

The meaning is:

> "Don't schedule Pods here unless they tolerate `gpu=true`."

---

# 3. What does this syntax mean?

```bash
kubectl taint nodes node-2 gpu=true:NoSchedule
```

Break it down:

```text
node-2
  |
  +--- gpu=true
  |       |
  |       +-- key = value
  |
  +--- NoSchedule
```

So:

```text
gpu=true
```

is the taint.

And:

```text
NoSchedule
```

is the **effect**.

---

# 4. Three taint effects

There are three important effects.

## `NoSchedule`

New Pods that don't tolerate the taint **will not be scheduled**.

```text
Node
 |
 +-- gpu=true:NoSchedule
```

Existing Pods are generally not evicted just because the taint is added.

---

## `PreferNoSchedule`

This is a **soft restriction**.

Kubernetes tries to avoid scheduling the Pod there, but it isn't an absolute prohibition.

```text
PreferNoSchedule
        |
        v
"Try not to put Pods here"
```

---

## `NoExecute`

This is stronger.

It affects **existing Pods as well**.

```text
Node gets:
gpu=true:NoExecute

Existing non-tolerating Pod
          |
          v
       Evicted
```

And new non-tolerating Pods won't be scheduled there.

---

# 5. Basic example

Let's say:

```text
node-1
node-2
```

Taint node-2:

```bash
kubectl taint nodes node-2 dedicated=production:NoSchedule
```

Architecture:

```text
                 Cluster
                    |
          +---------+---------+
          |                   |
        node-1              node-2
                              |
                         Taint:
                    dedicated=production
                         :NoSchedule
```

Now create a normal Pod:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
    - name: nginx
      image: nginx
```

There is no toleration.

Kubernetes can schedule it on:

```text
node-1
```

but not:

```text
node-2
```

because:

```text
node-2
   |
   +-- dedicated=production:NoSchedule
                         ^
                         |
                    Pod doesn't
                    tolerate it
```

---

# 6. Add a toleration

Now suppose we actually want a production Pod to run on node-2.

Add:

```yaml
tolerations:
  - key: "dedicated"
    operator: "Equal"
    value: "production"
    effect: "NoSchedule"
```

Complete example:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: production-app
spec:
  tolerations:
    - key: "dedicated"
      operator: "Equal"
      value: "production"
      effect: "NoSchedule"

  containers:
    - name: nginx
      image: nginx
```

Now:

```text
Node:
dedicated=production:NoSchedule

            ↑
            |
       Toleration
            |
            v
        production-app
```

The Pod **can** be scheduled there.

---

# 7. Very important: toleration does NOT mean "schedule me here"

This is probably the most important interview point.

Suppose:

```text
node-1
node-2
```

Node-2:

```text
dedicated=production:NoSchedule
```

Pod:

```text
tolerates dedicated=production
```

The Pod can run on:

```text
node-1
```

or:

```text
node-2
```

because toleration only removes the taint-based restriction.

It does **not** say:

> "I must run on node-2."

---

# 8. Taint + Toleration + NodeSelector

If you want:

> "Only production Pods should run on this node, and production Pods should actually go there."

Use:

```text
Taint
+
Toleration
+
NodeSelector / NodeAffinity
```

Example:

```text
                 node-2
              +-----------+
              | production|
              |           |
              | Taint     |
              | dedicated |
              | =production
              +-----------+
                    ^
                    |
              Toleration
                    +
              NodeSelector
```

Node:

```bash
kubectl label node node-2 workload=production
```

Pod:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: production-app
spec:

  nodeSelector:
    workload: production

  tolerations:
    - key: dedicated
      operator: Equal
      value: production
      effect: NoSchedule

  containers:
    - name: nginx
      image: nginx
```

Now two conditions must be satisfied:

```text
                     Pod
                      |
          +-----------+-----------+
          |                       |
          v                       v
     Toleration               NodeSelector
          |                       |
          v                       v
  "I'm allowed here"       "I want this node"
          |                       |
          +-----------+-----------+
                      |
                      v
                   node-2
```

This is a very common production pattern.

---

# 9. Real-world example — dedicated monitoring node

Suppose you have:

```text
Node 1 → Application workloads
Node 2 → Application workloads
Node 3 → Monitoring
```

You don't want application Pods accidentally using Node 3.

Taint it:

```bash
kubectl taint nodes node-3 workload=monitoring:NoSchedule
```

Then Prometheus/Grafana workloads get:

```yaml
tolerations:
  - key: workload
    operator: Equal
    value: monitoring
    effect: NoSchedule
```

And optionally:

```yaml
nodeSelector:
  workload: monitoring
```

Architecture:

```text
                 Kubernetes
                     |
       +-------------+-------------+
       |             |             |
       v             v             v
    Node 1         Node 2        Node 3
    Apps           Apps          Monitoring
                                  |
                              Taint:
                         workload=monitoring
                              :NoSchedule
                                  ^
                                  |
                            Toleration
                                  |
                         Prometheus/Grafana
```

---

# 10. `NoExecute` example

This one is especially important.

Suppose:

```text
node-1
```

has:

```text
maintenance=true:NoExecute
```

Command:

```bash
kubectl taint nodes node-1 maintenance=true:NoExecute
```

What happens?

```text
Existing Pod without toleration
            |
            v
          Evicted
```

New Pod without toleration:

```text
          Pod
           |
           X
        node-1
```

New Pod with toleration:

```yaml
tolerations:
  - key: maintenance
    operator: Equal
    value: true
    effect: NoExecute
```

can remain.

---

# 11. `tolerationSeconds`

You can also tolerate a `NoExecute` taint for a limited amount of time.

Example:

```yaml
tolerations:
  - key: maintenance
    operator: Equal
    value: true
    effect: NoExecute
    tolerationSeconds: 300
```

Meaning:

```text
NoExecute taint appears
        |
        v
Pod remains
        |
        | 300 seconds
        v
Pod gets evicted
```

Useful for temporary node problems.

---

# 12. Toleration operators

Two common operators:

### `Equal`

```yaml
tolerations:
  - key: dedicated
    operator: Equal
    value: production
    effect: NoSchedule
```

Matches:

```text
dedicated=production
```

---

### `Exists`

```yaml
tolerations:
  - key: dedicated
    operator: Exists
    effect: NoSchedule
```

This says:

> I tolerate the `dedicated` taint regardless of its value.

For example, it can tolerate:

```text
dedicated=production
dedicated=testing
dedicated=monitoring
```

---

# 13. Common Kubernetes example — control-plane nodes

Kubernetes control-plane nodes commonly have a taint such as:

```text
node-role.kubernetes.io/control-plane:NoSchedule
```

This prevents normal workloads from being scheduled onto control-plane nodes.

Conceptually:

```text
Control Plane
      |
      +-- API Server
      +-- Scheduler
      +-- Controller Manager
      +-- etcd
      |
      +-- NoSchedule taint
```

Therefore normal application Pods go to worker nodes.

---

# 14. Check taints

Use:

```bash
kubectl describe node node-2
```

Look for:

```text
Taints:
dedicated=production:NoSchedule
```

Or:

```bash
kubectl get node node-2 -o jsonpath='{.spec.taints}'
```

---

# 15. Remove a taint

If you have:

```text
dedicated=production:NoSchedule
```

remove it with:

```bash
kubectl taint nodes node-2 dedicated=production:NoSchedule-
```

Notice the final:

```text
-
```

That means remove it.

---

# 16. Complete demo

Let's make a simple scenario.

### Step 1 — See nodes

```bash
kubectl get nodes
```

Suppose:

```text
NAME     STATUS
node-1   Ready
node-2   Ready
```

### Step 2 — Taint node-2

```bash
kubectl taint node node-2 dedicated=backend:NoSchedule
```

### Step 3 — Create normal Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: normal-pod
spec:
  containers:
    - name: nginx
      image: nginx
```

Apply:

```bash
kubectl apply -f normal-pod.yaml
```

Check:

```bash
kubectl get pod normal-pod -o wide
```

It should not be scheduled onto the tainted node if another suitable node is available.

---

### Step 4 — Create Pod with toleration

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: backend-pod
spec:

  tolerations:
    - key: dedicated
      operator: Equal
      value: backend
      effect: NoSchedule

  containers:
    - name: nginx
      image: nginx
```

Now:

```bash
kubectl apply -f backend-pod.yaml
```

Check:

```bash
kubectl get pod backend-pod -o wide
```

It **can** run on `node-2`.

---

# 17. The interview trap

### Question:

> "If a node has a taint and the Pod has a matching toleration, will the Pod definitely run on that node?"

**Answer: No.**

Toleration only means:

```text
"Pod is allowed to run there."
```

It does not mean:

```text
"Pod must run there."
```

To target a specific node, use:

```text
Toleration
+
NodeSelector
```

or preferably for more flexible scheduling:

```text
Toleration
+
NodeAffinity
```

---

# 18. Taints vs Node Affinity

This distinction is very useful:

### Node Affinity

Pod says:

> **"I want to run on these nodes."**

```text
Pod
 |
 +----> Node Affinity
          |
          +-- choose these nodes
```

### Taint

Node says:

> **"I don't want these Pods."**

```text
Node
 |
 +----> Taint
          |
          +-- reject Pods without toleration
```

### Together

```text
             Pod
              |
       +------+------+
       |             |
       v             v
 Toleration      Node Affinity
       |             |
       |             |
       v             v
"I'm allowed"   "I want this node"
       |             |
       +------+------+
              |
              v
          Target Node
```

---

# 19. Easy memory trick

Remember:

```text
TAINT = NODE says NO ❌

TOLERATION = POD says
             "I can handle that" ✅

NODE AFFINITY = POD says
                "I WANT that node" 🎯
```

So for a dedicated node:

```text
Node:
dedicated=database:NoSchedule

        +
        
Pod:
tolerates database

        +
        
Pod:
nodeAffinity → database nodes
```

Result:

```text
              Database Node
             +-------------+
             |             |
             | Taint       |
             | database    |
             |             |
             +-------------+
                    ^
                    |
              Toleration
                    +
              Node Affinity
                    |
                    v
              Database Pod
```

**This combination is a very common interview scenario for dedicated nodes, GPU nodes, infrastructure nodes, and workload isolation.**
