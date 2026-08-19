# Argo CD concepts 

```text
Sync Policy  → HOW Argo CD reconciles
Self-Heal    → Fix manual drift
Prune        → Remove resources deleted from Git
Sync Waves   → Control deployment ORDER
```

Let's use one realistic application throughout.

---

# 1. Start with the GitOps model

Suppose Git contains:

```text
Git
 |
 +-- Deployment
 +-- Service
 +-- ConfigMap
```

Argo CD compares:

```text
             Git
              |
              | Desired State
              v
          Argo CD
              |
              | compare
              v
          Kubernetes
              |
              | Actual State
```

If:

```text
Desired == Actual
```

then:

```text
Synced
```

If:

```text
Desired != Actual
```

then:

```text
OutOfSync
```

The four concepts determine what happens next.

---

# 2. Sync Policy

First, **Sync Policy**.

It controls **how Argo CD synchronizes the application**.

Example:

```yaml
spec:
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

This says:

```text
automatically sync
+
automatically prune
+
automatically self-heal
```

Without automated sync:

```text
Git change
   |
   v
Argo CD detects change
   |
   v
OutOfSync
   |
   X
Doesn't automatically apply
```

You need:

```bash
argocd app sync payment-api
```

or:

```bash
argocd app sync payment-api --prune
```

---

# 3. Automated Sync

Example:

```yaml
syncPolicy:
  automated: {}
```

Now:

```text
Git
 |
 | commit
 v
Argo CD
 |
 | detects OutOfSync
 v
Automatically Sync
 |
 v
Kubernetes
```

Suppose Git changes:

```yaml
replicas: 3
```

from:

```yaml
replicas: 2
```

Argo CD automatically applies it.

```text
Git = 3
Cluster = 2

        ↓

Argo CD

        ↓

Cluster = 3
```

---

# 4. Self-Heal

Now suppose Git says:

```yaml
replicas: 3
```

Cluster currently:

```text
replicas: 3
```

Everything is healthy:

```text
Git       Cluster
  3   ==    3
```

But someone manually runs:

```bash
kubectl scale deployment payment-api --replicas=1
```

Now:

```text
Git        Cluster
  3    !=     1
```

Argo CD detects:

```text
OutOfSync
```

If you have:

```yaml
syncPolicy:
  automated:
    selfHeal: true
```

Argo CD automatically changes:

```text
1
↓
3
```

So:

```text
              Git
               |
               | desired = 3
               v
            Argo CD
               |
               | detects drift
               v
        Kubernetes = 1
               |
               v
          SELF HEAL
               |
               v
        Kubernetes = 3
```

---

# 5. What does Self-Heal actually mean?

The important phrase is:

> **Self-heal means Argo CD automatically reconciles out-of-band changes in the live cluster back to the Git-defined desired state.**

Examples:

Someone changes:

```bash
kubectl edit deployment payment-api
```

or:

```bash
kubectl scale deployment payment-api --replicas=1
```

or:

```bash
kubectl patch deployment ...
```

Argo CD sees the difference and restores Git's desired state.

---

# 6. Self-Heal requires automated sync

Typically:

```yaml
syncPolicy:
  automated:
    selfHeal: true
```

The important distinction:

```text
automated
   |
   +-- automatically sync Git changes

selfHeal
   |
   +-- automatically correct live-cluster drift
```

So I remember it as:

```text
Git change
   ↓
automated sync

Manual cluster change
   ↓
selfHeal
```

---

# 7. Prune

Now suppose Git initially contains:

```text
deployment.yaml
service.yaml
configmap.yaml
```

Argo CD manages:

```text
Deployment
Service
ConfigMap
```

Now you delete:

```text
configmap.yaml
```

from Git.

Desired state becomes:

```text
Deployment
Service
```

But Kubernetes still has:

```text
Deployment
Service
ConfigMap
```

So:

```text
Git                 Cluster

Deployment     =    Deployment
Service        =    Service

(no ConfigMap)      ConfigMap
                     ↑
                   extra
```

The ConfigMap is now an **orphaned/extra managed resource** relative to the desired state.

This is where **Prune** comes in.

---

# 8. Enable Prune

```yaml
syncPolicy:
  automated:
    prune: true
```

Now:

```text
Delete manifest from Git
          |
          v
Argo CD detects resource
          |
          v
Resource no longer desired
          |
          v
PRUNE
          |
          v
Delete resource from cluster
```

So:

```text
Git
 |
 X ConfigMap removed
 |
 v
Argo CD
 |
 v
ConfigMap deleted
```

---

# 9. Self-Heal vs Prune

This is one of the most common interview questions.

### Self-Heal

Resource **still exists in Git**, but someone changed it in the cluster.

```text
Git:
replicas = 3

Cluster:
replicas = 1

       ↓

Self-Heal

       ↓

Cluster:
replicas = 3
```

---

### Prune

Resource **no longer exists in Git**.

```text
Git:
Deployment
Service

Cluster:
Deployment
Service
ConfigMap

       ↓

Prune

       ↓

ConfigMap deleted
```

Memorize:

> **Self-heal fixes changed resources. Prune removes resources that Git no longer wants.**

---

# 10. Sync Policy with both

A common production configuration:

```yaml
spec:
  syncPolicy:

    automated:

      prune: true

      selfHeal: true

    syncOptions:
      - CreateNamespace=true
```

This gives:

```text
Git change
   ↓
Automatic Sync

Manual cluster change
   ↓
Self-Heal

Resource removed from Git
   ↓
Prune
```

---

# 11. Complete Application example

Let's create a realistic Argo CD Application.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application

metadata:
  name: payment-api
  namespace: argocd

spec:

  project: default

  source:
    repoURL: https://github.com/company/gitops.git
    targetRevision: main
    path: apps/payment

  destination:
    server: https://kubernetes.default.svc
    namespace: payment

  syncPolicy:

    automated:
      prune: true
      selfHeal: true

    syncOptions:
      - CreateNamespace=true
```

Now:

```text
                    Git
                     |
                     v
               payment-api
                     |
                     v
                  Argo CD
                     |
          +----------+----------+
          |          |          |
          v          v          v
       Sync       Self-Heal   Prune
```

---

# 12. What does `syncOptions` mean?

You may see:

```yaml
syncOptions:
  - CreateNamespace=true
```

This is **not the same thing as `syncPolicy.automated`**.

Think:

```text
syncPolicy
   |
   +-- automated
   |     |
   |     +-- automatic sync
   |     +-- selfHeal
   |     +-- prune
   |
   +-- syncOptions
         |
         +-- behavior/configuration of sync
```

Examples of sync options include:

```yaml
syncOptions:
  - CreateNamespace=true
  - ServerSideApply=true
```

depending on your use case.

---

# 13. Now Sync Waves

This is a different problem.

Suppose you have:

```text
Namespace
   ↓
ConfigMap
   ↓
Deployment
   ↓
Ingress
```

You don't necessarily want Argo CD to create everything in an arbitrary order.

You can use:

```text
Sync Waves
```

to explicitly control ordering.

---

# 14. Example

Suppose:

```text
Namespace
```

should come first.

```yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "-2"
```

Then:

```text
ConfigMap
```

wave:

```yaml
argocd.argoproj.io/sync-wave: "-1"
```

Deployment:

```yaml
argocd.argoproj.io/sync-wave: "0"
```

Ingress:

```yaml
argocd.argoproj.io/sync-wave: "1"
```

Result:

```text
Wave -2
   |
   v
Namespace
   |
   v
Wave -1
   |
   v
ConfigMap
   |
   v
Wave 0
   |
   v
Deployment
   |
   v
Wave 1
   |
   v
Ingress
```

---

# 15. Full Sync Wave example

### Namespace

```yaml
apiVersion: v1
kind: Namespace

metadata:
  name: payment

  annotations:
    argocd.argoproj.io/sync-wave: "-2"
```

---

### ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap

metadata:
  name: payment-config

  annotations:
    argocd.argoproj.io/sync-wave: "-1"

data:
  ENV: production
```

---

### Deployment

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: payment-api

  annotations:
    argocd.argoproj.io/sync-wave: "0"

spec:
  replicas: 3

  selector:
    matchLabels:
      app: payment

  template:

    metadata:
      labels:
        app: payment

    spec:
      containers:
        - name: payment
          image: company/payment:v2
```

---

### Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress

metadata:
  name: payment

  annotations:
    argocd.argoproj.io/sync-wave: "1"

spec:
  ...
```

---

# 16. How Sync Waves work

Argo CD conceptually does:

```text
Find resources
       |
       v
Sort by sync wave
       |
       v
Wave -2
       |
       v
Wait until healthy/ready according to sync behavior
       |
       v
Wave -1
       |
       v
Wait
       |
       v
Wave 0
       |
       v
Wait
       |
       v
Wave 1
```

So waves are basically:

> **"Deploy these resources in this order."**

---

# 17. Negative waves are very useful

For infrastructure:

```text
-3 → CRDs
-2 → controllers
-1 → configuration
 0 → applications
 1 → ingress
```

Example:

```text
             Argo CD Sync
                  |
                  v
             Wave -3
               CRDs
                  |
                  v
             Wave -2
       AWS Load Balancer Controller
                  |
                  v
             Wave -1
              Config
                  |
                  v
             Wave 0
            Backend
                  |
                  v
             Wave 1
             Frontend
```

This is particularly useful in platform GitOps.

---

# 18. Sync Waves + App of Apps

This connects directly to what we discussed earlier.

You can have:

```text
                    Root App
                 App of Apps
                      |
       +--------------+--------------+
       |              |              |
       v              v              v
Infrastructure     Platform       Applications
       |              |              |
       v              v              v
     Wave -2        Wave -1         Wave 0
```

For example:

```text
Wave -3
  |
  +-- CRDs

Wave -2
  |
  +-- cert-manager
  +-- ALB controller

Wave -1
  |
  +-- ExternalDNS

Wave 0
  |
  +-- Redis
  +-- Backend

Wave 1
  |
  +-- Frontend
```

---

# 19. What happens when one wave fails?

Suppose:

```text
Wave -2
CRDs
   |
   v
SUCCESS

Wave -1
ALB Controller
   |
   X
FAILED
```

Argo CD should not blindly proceed with dependent resources as if everything were healthy.

You effectively have:

```text
Wave -2 → completed
Wave -1 → failed
Wave 0  → blocked/not progressed
```

This gives you controlled deployment ordering.

---

# 20. Complete production example

Let's combine everything.

You have:

```text
payment-api
```

Git:

```text
apps/payment/
│
├── namespace.yaml
├── configmap.yaml
├── deployment.yaml
├── service.yaml
└── ingress.yaml
```

Argo Application:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application

metadata:
  name: payment-api
  namespace: argocd

spec:

  source:
    repoURL: https://github.com/company/gitops.git
    path: apps/payment
    targetRevision: main

  destination:
    server: https://kubernetes.default.svc
    namespace: payment

  syncPolicy:

    automated:
      prune: true
      selfHeal: true

    syncOptions:
      - CreateNamespace=true
```

Resources:

```text
Namespace       wave -2
ConfigMap       wave -1
Deployment      wave  0
Service         wave  0
Ingress         wave  1
```

Architecture:

```text
                         Git
                          |
                          v
                    +-----------+
                    |  Argo CD  |
                    +-----+-----+
                          |
                    Sync Policy
                          |
          +---------------+---------------+
          |               |               |
          v               v               v
       Auto Sync       Self-Heal        Prune
          |               |               |
          |               |               |
          +---------------+---------------+
                          |
                          v
                     Sync Waves
                          |
          +---------------+---------------+
          |               |               |
          v               v               v
       Wave -2         Wave -1          Wave 0
      Namespace        ConfigMap       Deployment
                                          |
                                          v
                                       Service
                                          |
                                          v
                                       Wave 1
                                        Ingress
```

---

# 21. Real-world scenario: Developer changes replicas

Git:

```yaml
replicas: 5
```

Cluster:

```text
replicas: 3
```

Developer commits:

```text
3 → 5
```

With:

```yaml
automated: {}
```

Argo CD:

```text
OutOfSync
   ↓
Automatic Sync
   ↓
5 replicas
```

---

# 22. Real-world scenario: Someone manually changes replicas

Git:

```text
replicas = 5
```

Cluster:

```text
replicas = 2
```

With:

```yaml
selfHeal: true
```

Argo CD:

```text
Detect drift
    ↓
Self-heal
    ↓
replicas = 5
```

---

# 23. Real-world scenario: Developer removes Redis

Initially Git:

```text
backend
redis
```

Cluster:

```text
backend
redis
```

Developer deletes:

```text
redis/
```

from Git.

Now:

```text
Git:

backend

Cluster:

backend
redis
```

With:

```yaml
prune: true
```

Argo CD:

```text
Redis no longer desired
        ↓
Prune
        ↓
Redis resources deleted
```

---

# 24. Very important production warning about Prune

Be careful with:

```yaml
prune: true
```

because:

> **Deleting a resource from Git can result in the actual Kubernetes resource being deleted.**

For example, if you accidentally delete:

```text
database.yaml
```

from Git and pruning is enabled, you could potentially delete the database resource.

That's why production teams often use:

```text
Git PR
 ↓
Code review
 ↓
Argo CD diff
 ↓
Approval
 ↓
Sync
```

rather than allowing unrestricted destructive changes.

There are also Argo CD sync options such as resource-level deletion protection patterns depending on the resource.

---

# 25. `prune: true` vs `--prune`

Manual sync:

```bash
argocd app sync payment-api
```

doesn't necessarily mean:

> Delete everything no longer in Git.

You can explicitly use:

```bash
argocd app sync payment-api --prune
```

With automated sync:

```yaml
automated:
  prune: true
```

Argo CD can prune automatically during automated synchronization.

---

# 26. Sync Policy cheat sheet

Think of this:

```text
                 syncPolicy
                     |
          +----------+----------+
          |                     |
      automated              options
          |
    +-----+-----+
    |           |
   selfHeal    prune
```

### `automated`

```yaml
automated: {}
```

> Automatically apply Git changes.

### `selfHeal`

```yaml
selfHeal: true
```

> Automatically correct manual live-cluster changes.

### `prune`

```yaml
prune: true
```

> Delete resources that are no longer defined in Git.

### `syncOptions`

```yaml
syncOptions:
  - CreateNamespace=true
```

> Change how synchronization behaves.

### `sync-wave`

```yaml
argocd.argoproj.io/sync-wave: "1"
```

> Control synchronization order.

---

# 27. The interview distinction

If they ask:

### "What is Self-Healing?"

Say:

> "Self-healing detects drift between the desired state in Git and the live Kubernetes state and automatically reconciles the live state back to Git."

### "What is Pruning?"

> "Pruning removes resources from the cluster that are still managed by Argo CD but are no longer present in the desired Git state."

### "What are Sync Waves?"

> "Sync waves control the order in which Argo CD synchronizes resources. I use negative waves for prerequisites such as CRDs and controllers, and later waves for dependent applications."

### "What is Sync Policy?"

> "Sync policy defines how Argo CD synchronizes an Application, including automated synchronization, self-healing, pruning and sync options."

---

# 28. One diagram to memorize

```text
                         GIT
                          |
                          v
                    +-----------+
                    |  Argo CD  |
                    +-----+-----+
                          |
                    Sync Policy
                          |
          +---------------+---------------+
          |               |               |
          v               v               v
      Auto Sync        Self-Heal        Prune
          |               |               |
          |          Fix manual       Remove deleted
          |             drift           resources
          |               |               |
          +---------------+---------------+
                          |
                          v
                    Sync Waves
                          |
              +-----------+-----------+
              |           |           |
              v           v           v
           Wave -2     Wave -1      Wave 0
             CRDs       Config     Applications
                                      |
                                      v
                                  Wave 1
                                   Ingress
```

### The easiest way to remember all four:

**Sync Policy = HOW**

**Self-Heal = FIX**

**Prune = DELETE**

**Sync Waves = ORDER**

That's the mental model I'd use in your Argo CD interview.
