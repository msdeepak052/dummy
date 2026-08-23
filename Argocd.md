# 1. What is App of Apps?

The basic idea is:

> **One Argo CD Application manages other Argo CD Applications.**

Normally:

```text
Argo CD
   |
   +---- Application A
   |
   +---- Application B
   |
   +---- Application C
```

With App of Apps:

```text
                  Argo CD
                     |
                     v
              Root Application
              "app-of-apps"
                     |
        +------------+------------+
        |            |            |
        v            v            v
     App A        App B         App C
   frontend      backend        redis
        |            |            |
        v            v            v
    Deployment    Deployment    StatefulSet
```

The **root application doesn't deploy your application's Pods directly**.

Instead, it creates/manages the child Argo CD `Application` objects.

---

# 2. Why do we need it?

Imagine you have 30 applications:

```text
frontend
backend
payment
order
catalog
redis
kafka
monitoring
...
```

Without App of Apps, you have many independent Argo CD Applications.

You need to create/manage them individually.

With App of Apps:

```text
Git
 |
 +-- app-of-apps
       |
       +-- frontend Application
       +-- backend Application
       +-- payment Application
       +-- order Application
       +-- redis Application
       +-- kafka Application
```

Now **Git becomes the source of truth for the entire application landscape**.

---

# 3. Repository structure

A simple production-style repository:

```text
gitops-repo/
│
├── root/
│   └── app-of-apps.yaml
│
└── applications/
    │
    ├── frontend.yaml
    ├── backend.yaml
    ├── payment.yaml
    └── redis.yaml
```

Then your actual application manifests/charts can be elsewhere:

```text
gitops-repo/
│
├── root/
│   └── app-of-apps.yaml
│
├── applications/
│   ├── frontend.yaml
│   ├── backend.yaml
│   ├── payment.yaml
│   └── redis.yaml
│
└── charts/
    ├── frontend/
    ├── backend/
    └── payment/
```

---

# 4. Root Application

This is the most important object.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application

metadata:
  name: app-of-apps
  namespace: argocd

spec:

  project: default

  source:
    repoURL: https://github.com/company/gitops.git
    targetRevision: main
    path: applications

  destination:
    server: https://kubernetes.default.svc
    namespace: argocd

  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

The important part is:

```yaml
source:
  path: applications
```

Argo CD looks inside:

```text
applications/
```

and finds:

```text
frontend.yaml
backend.yaml
payment.yaml
redis.yaml
```

Those files are themselves **Argo CD Application resources**.

---

# 5. Child Application

For example:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application

metadata:
  name: frontend
  namespace: argocd

spec:

  project: default

  source:
    repoURL: https://github.com/company/gitops.git
    targetRevision: main
    path: charts/frontend

  destination:
    server: https://kubernetes.default.svc
    namespace: frontend

  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

The hierarchy becomes:

```text
                app-of-apps
                     |
                     v
              +------+------+
              |             |
              v             v
          frontend       backend
          Application    Application
              |             |
              v             v
          Helm chart     Helm chart
              |             |
              v             v
           K8s Pods      K8s Pods
```

---

# 6. Complete example

Let's say you have:

```text
frontend
backend
payment
```

Your repository:

```text
gitops/
│
├── root/
│   └── app-of-apps.yaml
│
├── applications/
│   ├── frontend.yaml
│   ├── backend.yaml
│   └── payment.yaml
│
└── apps/
    ├── frontend/
    │   └── deployment.yaml
    │
    ├── backend/
    │   └── deployment.yaml
    │
    └── payment/
        └── deployment.yaml
```

---

# 7. Root Application

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application

metadata:
  name: production-apps
  namespace: argocd

spec:

  project: production

  source:
    repoURL: https://github.com/company/gitops.git
    targetRevision: main
    path: applications

  destination:
    server: https://kubernetes.default.svc
    namespace: argocd

  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

---

# 8. Frontend child

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application

metadata:
  name: frontend
  namespace: argocd

spec:

  project: production

  source:
    repoURL: https://github.com/company/gitops.git
    targetRevision: main
    path: apps/frontend

  destination:
    server: https://kubernetes.default.svc
    namespace: frontend

  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

---

# 9. Backend child

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application

metadata:
  name: backend
  namespace: argocd

spec:

  project: production

  source:
    repoURL: https://github.com/company/gitops.git
    targetRevision: main
    path: apps/backend

  destination:
    server: https://kubernetes.default.svc
    namespace: backend

  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

---

# 10. Payment child

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application

metadata:
  name: payment
  namespace: argocd

spec:

  project: production

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
```

---

# 11. What happens when you deploy?

You initially apply only:

```bash
kubectl apply -f root/app-of-apps.yaml
```

Argo CD sees:

```text
production-apps
       |
       v
applications/
       |
       +-- frontend.yaml
       +-- backend.yaml
       +-- payment.yaml
```

It creates:

```text
Argo CD
   |
   +-- production-apps
          |
          +-- frontend
          +-- backend
          +-- payment
```

Then each child Application manages its actual workloads:

```text
frontend
   |
   +-- Deployment
   +-- Service
   +-- ConfigMap

backend
   |
   +-- Deployment
   +-- Service
   +-- ConfigMap

payment
   |
   +-- Deployment
   +-- Service
   +-- Secret
```

---

# 12. Why is this powerful?

Suppose tomorrow you add:

```text
notification-service
```

You only create:

```text
applications/notification.yaml
```

Commit it:

```bash
git add .
git commit -m "Add notification service"
git push
```

Argo CD sees the change.

```text
Git
 |
 v
App of Apps
 |
 v
notification Application
 |
 v
notification workloads
```

You don't manually create another Argo CD Application.

---

# 13. App of Apps + Helm
he **App of Apps** pattern uses a single parent ArgoCD Application to discover and deploy multiple child Applications. When managing Helm-based child apps with baseline and environment-specific overrides, ArgoCD allows you to supply multiple values files (`values.yaml` and `override.yaml`) directly within the child `Application` spec.

---

### Directory Structure

```text
git-ops-repo/
├── bootstrap/                      # Parent App (Plain YAML or Helm)
│   └── root-app.yaml               # Root Application pointing to parent-app/
├── parent-app/                     # Parent Helm Chart
│   ├── Chart.yaml
│   ├── values.yaml
│   └── templates/
│       ├── guestbook.yaml          # Child App 1 (Application CRD)
│       └── backend.yaml            # Child App 2 (Application CRD)
└── charts/                         # Child Helm Charts
    ├── guestbook/
    │   ├── Chart.yaml
    │   ├── values.yaml             # Default values
    │   ├── override.yaml           # Environment/Instance overrides
    │   └── templates/
    │       └── deployment.yaml
    └── backend/
        ├── Chart.yaml
        ├── values.yaml
        ├── override.yaml
        └── templates/
            └── deployment.yaml

```

---

### 1. Root Bootstrap Application (`bootstrap/root-app.yaml`)

This is applied manually once (or via CI/CD) to initiate the parent app.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: root-application
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/<your-org>/git-ops-repo.git
    targetRevision: HEAD
    path: parent-app
    helm:
      valueFiles:
        - values.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true

```

---

### 2. Parent Chart (`parent-app/Chart.yaml`)

```yaml
apiVersion: v2
name: parent-app
description: Umbrella chart deploying ArgoCD Child Applications
version: 0.1.0
appVersion: "1.0.0"

```

---

### 3. Child Application Templates (`parent-app/templates/`)

Each child application is defined as an ArgoCD `Application` resource. Using `valueFiles`, ArgoCD merges `values.yaml` and `override.yaml` in sequential order (keys in `override.yaml` take precedence).

#### `parent-app/templates/guestbook.yaml`

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: guestbook
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/<your-org>/git-ops-repo.git
    targetRevision: HEAD
    path: charts/guestbook
    helm:
      valueFiles:
        - values.yaml
        - override.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: guestbook-prod
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true

```

#### `parent-app/templates/backend.yaml`

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: backend
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/<your-org>/git-ops-repo.git
    targetRevision: HEAD
    path: charts/backend
    helm:
      valueFiles:
        - values.yaml
        - override.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: backend-prod
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true

```

---

### 4. Child Helm App Values Example

#### `charts/guestbook/values.yaml` (Default)

```yaml
replicaCount: 1
image:
  repository: gcr.io/heptio-images/ks-guestbook-demo
  tag: v0.1
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80

resources:
  limits:
    cpu: 100m
    memory: 128Mi
  requests:
    cpu: 50m
    memory: 64Mi

```

#### `charts/guestbook/override.yaml` (Overrides)

```yaml
replicaCount: 3

service:
  type: NodePort

resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 200m
    memory: 256Mi

```

---

### 5. Deployment Step

Apply the root application to your cluster:

```bash
kubectl apply -f bootstrap/root-app.yaml

```

Once synced, ArgoCD will:

1. Render the `parent-app` templates into the individual `Application` manifests.
2. Create `guestbook` and `backend` Applications inside the `argocd` namespace.
3. Automatically trigger Helm runs for both child charts, applying `values.yaml` first and merging `override.yaml` on top.
---
### Memorize this:

**App of Apps = Application that manages Applications.**

**ApplicationSet = tool that generates Applications.**

**Sync Waves = control the order in which Argo CD syncs resources/applications.**

That distinction will cover a large portion of the Argo CD architecture questions you'll likely get in a Senior DevOps/Platform interview.


---

# 24. Interview answer

If they ask:

> **"Explain the App of Apps pattern in Argo CD."**

A strong Senior-level answer:

> "App of Apps is a bootstrap and hierarchical GitOps pattern where one parent Argo CD Application manages a set of child Argo CD Applications. The parent points to a Git directory containing Application manifests. Each child Application then manages the actual Kubernetes resources or Helm charts for an individual service.
>
> For example, I might have a root Application managing frontend, backend, payment, Redis and platform components. This gives us a Git-controlled application hierarchy and allows us to bootstrap an entire environment from a single parent Application. For ordering, I can use sync waves, and for large multi-cluster or multi-environment fleets I'd generally consider ApplicationSet because it can generate Applications dynamically."

---

# 25. The diagram to remember

```text
                         GIT
                          |
                          v
                 ┌─────────────────┐
                 │   App of Apps   │
                 │  Root App       │
                 └────────┬────────┘
                          |
            ┌─────────────┼─────────────┐
            |             |             |
            v             v             v
       ┌─────────┐   ┌─────────┐   ┌─────────┐
       │Frontend │   │ Backend │   │ Payment │
       │  App    │   │   App   │   │   App   │
       └────┬────┘   └────┬────┘   └────┬────┘
            |             |             |
            v             v             v
        Deployment    Deployment    Deployment
        Service       Service       Service
            |             |             |
            v             v             v
          Pods           Pods           Pods
```

---
T
