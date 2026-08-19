# Helm in Kubernetes — from zero to interview level

If you work with Kubernetes, think of **Helm as a package manager + templating engine for Kubernetes**.

Instead of maintaining 20–30 YAML files with hardcoded values, you create a reusable **Helm chart** and supply environment-specific values.

```text
Without Helm

deployment-dev.yaml
deployment-prod.yaml
service-dev.yaml
service-prod.yaml
configmap-dev.yaml
configmap-prod.yaml
...

With Helm

              Helm Chart
                  |
          +-------+-------+
          |               |
      values-dev      values-prod
          |               |
          v               v
      Kubernetes       Kubernetes
```

---

# 1. What problem does Helm solve?

Suppose your application needs:

```text
Deployment
Service
ConfigMap
Secret
Ingress
HPA
ServiceAccount
```

Without Helm, you might have:

```text
k8s/
├── deployment.yaml
├── service.yaml
├── configmap.yaml
├── ingress.yaml
├── hpa.yaml
└── serviceaccount.yaml
```

Now you deploy to:

```text
dev
qa
staging
production
```

You could maintain separate YAMLs:

```text
dev/deployment.yaml
qa/deployment.yaml
prod/deployment.yaml
```

This creates duplication.

With Helm:

```text
myapp/
├── Chart.yaml
├── values.yaml
└── templates/
    ├── deployment.yaml
    ├── service.yaml
    ├── configmap.yaml
    └── ingress.yaml
```

Then:

```text
values-dev.yaml
values-prod.yaml
```

control the differences.

---

# 2. Helm architecture

The important thing is that **modern Helm 3 does not have the old Tiller architecture**.

Helm 3 is essentially:

```text
Developer
    |
    v
  Helm CLI
    |
    v
Kubernetes API Server
    |
    v
Kubernetes
```

For example:

```bash
helm install myapp ./mychart
```

Helm renders the templates and interacts with the Kubernetes API.

---

# 3. Important Helm terminology

You should know these terms.

### Chart

A package containing:

```text
Templates
Default values
Metadata
Optional dependencies
```

Example:

```text
myapp/
```

is a Helm chart.

---

### Release

A **release** is an installed instance of a chart.

For example:

```bash
helm install payment-api ./payment-chart
```

Here:

```text
Chart:
payment-chart

Release:
payment-api
```

You can install the same chart multiple times:

```text
payment-dev
payment-qa
payment-prod
```

---

### Repository

A Helm repository stores Helm charts.

Conceptually:

```text
Helm Repository
     |
     +-- nginx chart
     +-- redis chart
     +-- prometheus chart
```

---

# 4. Chart structure

Create a chart:

```bash
helm create myapp
```

You'll get:

```text
myapp/
├── Chart.yaml
├── values.yaml
├── charts/
├── templates/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── serviceaccount.yaml
│   ├── hpa.yaml
│   ├── _helpers.tpl
│   └── tests/
└── .helmignore
```

For learning, we'll create a smaller chart ourselves.

---

# 5. Create our demo chart

```bash
mkdir helm-demo
cd helm-demo
```

Create:

```text
helm-demo/
├── Chart.yaml
├── values.yaml
└── templates/
    ├── deployment.yaml
    ├── service.yaml
    └── configmap.yaml
```

---

# 6. `Chart.yaml`

This contains chart metadata.

```yaml
apiVersion: v2
name: helm-demo
description: A demo Helm chart
type: application
version: 0.1.0
appVersion: "1.0.0"
```

Important:

```text
version
```

is the **chart version**.

While:

```text
appVersion
```

is the version of the application being packaged.

Interview trap:

> `version` and `appVersion` are not the same thing.

---

# 7. `values.yaml`

This is where we define configurable values.

```yaml
replicaCount: 2

image:
  repository: nginx
  tag: "1.27"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80

app:
  name: helm-demo
  environment: dev

resources:
  requests:
    cpu: 100m
    memory: 128Mi

  limits:
    cpu: 500m
    memory: 512Mi
```

Think:

```text
values.yaml
      |
      v
Configuration
      |
      v
Templates
      |
      v
Kubernetes YAML
```

---

# 8. Deployment template

Create:

```text
templates/deployment.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: {{ .Release.Name }}-{{ .Values.app.name }}

spec:
  replicas: {{ .Values.replicaCount }}

  selector:
    matchLabels:
      app: {{ .Values.app.name }}

  template:
    metadata:
      labels:
        app: {{ .Values.app.name }}

    spec:
      containers:
        - name: {{ .Values.app.name }}

          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"

          imagePullPolicy: {{ .Values.image.pullPolicy }}

          ports:
            - containerPort: {{ .Values.service.port }}

          resources:
            {{- toYaml .Values.resources | nindent 12 }}
```

Now let's understand the template syntax.

---

# 9. `.Values`

This is one of the most important Helm concepts.

Given:

```yaml
replicaCount: 2
```

you access it using:

```text
{{ .Values.replicaCount }}
```

Given:

```yaml
image:
  repository: nginx
  tag: "1.27"
```

you use:

```text
{{ .Values.image.repository }}
```

and:

```text
{{ .Values.image.tag }}
```

So:

```text
values.yaml

image:
  repository: nginx
  tag: "1.27"

        ↓

.Values.image.repository
        ↓
nginx
```

---

# 10. `.Release`

`.Release` contains information about the current Helm release.

For example:

```text
{{ .Release.Name }}
```

If you install:

```bash
helm install payment-api ./helm-demo
```

then:

```text
.Release.Name
```

becomes:

```text
payment-api
```

So:

```yaml
name: {{ .Release.Name }}-{{ .Values.app.name }}
```

becomes:

```yaml
name: payment-api-helm-demo
```

---

# 11. Service template

Create:

```text
templates/service.yaml
```

```yaml
apiVersion: v1
kind: Service

metadata:
  name: {{ .Release.Name }}-{{ .Values.app.name }}

spec:
  type: {{ .Values.service.type }}

  selector:
    app: {{ .Values.app.name }}

  ports:
    - port: {{ .Values.service.port }}
      targetPort: {{ .Values.service.port }}
```

The Service selector must match:

```yaml
labels:
  app: {{ .Values.app.name }}
```

from the Deployment.

---

# 12. ConfigMap template

Create:

```text
templates/configmap.yaml
```

```yaml
apiVersion: v1
kind: ConfigMap

metadata:
  name: {{ .Release.Name }}-config

data:
  APP_NAME: {{ .Values.app.name | quote }}
  ENVIRONMENT: {{ .Values.app.environment | quote }}
```

Notice:

```text
| quote
```

This is a Helm template function.

So:

```yaml
APP_NAME: {{ .Values.app.name | quote }}
```

could render:

```yaml
APP_NAME: "helm-demo"
```

---

# 13. Render the chart without installing

This is one of the most important Helm commands:

```bash
helm template myrelease ./helm-demo
```

Helm renders:

```text
values.yaml
      +
templates/
      |
      v
Rendered Kubernetes YAML
```

Example output:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myrelease-helm-demo

spec:
  replicas: 2
```

This is extremely useful for debugging.

---

# 14. `helm lint`

Run:

```bash
helm lint ./helm-demo
```

It checks the chart for common issues.

Example:

```text
==> Linting ./helm-demo
[INFO] Chart.yaml: icon is recommended
1 chart(s) linted, 0 chart(s) failed
```

Think:

```text
helm lint
     ↓
Chart sanity check
```

---

# 15. Install the chart

```bash
helm install myrelease ./helm-demo
```

Now:

```bash
kubectl get pods
```

You should see something like:

```text
NAME                           READY   STATUS
myrelease-helm-demo-xxxxx      1/1     Running
myrelease-helm-demo-yyyyy      1/1     Running
```

because:

```yaml
replicaCount: 2
```

---

# 16. Check Helm releases

```bash
helm list
```

Example:

```text
NAME         NAMESPACE   REVISION   STATUS
myrelease    default     1          deployed
```

This tells you:

```text
Release name
Namespace
Revision
Status
```

---

# 17. Check release status

```bash
helm status myrelease
```

This gives information about the release.

Useful during troubleshooting.

---

# 18. Get release values

```bash
helm get values myrelease
```

This shows values explicitly supplied to the release.

For all values, including computed/default values:

```bash
helm get values myrelease --all
```

---

# 19. Get rendered manifests

Very useful:

```bash
helm get manifest myrelease
```

This shows the Kubernetes manifests Helm submitted/rendered for the release.

So:

```text
values.yaml
      |
      v
Helm template
      |
      v
Rendered manifests
      |
      v
Kubernetes
```

---

# 20. Override values from the command line

Suppose:

```yaml
replicaCount: 2
```

But you want:

```text
5 replicas
```

Run:

```bash
helm upgrade myrelease ./helm-demo \
  --set replicaCount=5
```

Now:

```bash
kubectl get deployment
```

should show:

```text
READY
5/5
```

---

# 21. Environment-specific values

This is where Helm becomes really useful.

Create:

```text
values-dev.yaml
```

```yaml
replicaCount: 1

image:
  repository: nginx
  tag: "1.27"

app:
  environment: dev

resources:
  requests:
    cpu: 100m
    memory: 128Mi

  limits:
    cpu: 250m
    memory: 256Mi
```

And:

```text
values-prod.yaml
```

```yaml
replicaCount: 5

image:
  repository: nginx
  tag: "1.27"

app:
  environment: production

resources:
  requests:
    cpu: 500m
    memory: 512Mi

  limits:
    cpu: "1"
    memory: 1Gi
```

Now:

```text
Same templates
       |
       +------ values-dev.yaml
       |
       +------ values-prod.yaml
```

---

# 22. Install dev

```bash
helm install myapp-dev ./helm-demo \
  -f values-dev.yaml
```

Production:

```bash
helm install myapp-prod ./helm-demo \
  -f values-prod.yaml
```

Now:

```text
myapp-dev
   |
   +-- 1 replica
   +-- dev configuration


myapp-prod
   |
   +-- 5 replicas
   +-- production configuration
```

Same chart.

Different configuration.

---

# 23. Multiple values files

You can provide multiple `-f` files.

For example:

```bash
helm upgrade myapp ./helm-demo \
  -f values.yaml \
  -f values-prod.yaml
```

The later file takes precedence for overlapping values.

Conceptually:

```text
values.yaml
    |
    v
values-prod.yaml
    |
    v
Final merged values
```

---

# 24. `--set` precedence

You can also use:

```bash
helm upgrade myapp ./helm-demo \
  -f values-prod.yaml \
  --set replicaCount=10
```

Conceptually:

```text
values.yaml
     ↓
values-prod.yaml
     ↓
--set
     ↓
Final values
```

Command-line overrides are useful for quick changes, but for GitOps I'd generally keep important environment configuration in version-controlled values files rather than relying on ad-hoc `--set`.

---

# 25. Helm upgrade

Suppose you change:

```yaml
replicaCount: 5
```

Run:

```bash
helm upgrade myapp-prod ./helm-demo \
  -f values-prod.yaml
```

Helm creates a new release revision.

Check:

```bash
helm list
```

or:

```bash
helm history myapp-prod
```

You might see:

```text
REVISION   STATUS
1          superseded
2          deployed
```

---

# 26. Helm rollback

Suppose revision 2 broke production.

Check:

```bash
helm history myapp-prod
```

Then:

```bash
helm rollback myapp-prod 1
```

Architecture:

```text
Revision 1
     |
     v
Revision 2 ← broken
     |
     v
Rollback
     |
     v
Revision 1
```

This is one of Helm's major operational benefits.

---

# 27. Helm uninstall

```bash
helm uninstall myapp-prod
```

This removes the release's managed Kubernetes resources according to Helm's release/resource behavior.

Check:

```bash
helm list
```

---

# 28. Helm repository commands

Add a repository:

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
```

Update repository indexes:

```bash
helm repo update
```

Search:

```bash
helm search repo nginx
```

List repositories:

```bash
helm repo list
```

Remove:

```bash
helm repo remove bitnami
```

---

# 29. Install a chart from a repository

For example:

```bash
helm install my-nginx bitnami/nginx
```

You can inspect its values:

```bash
helm show values bitnami/nginx
```

This is extremely useful when deploying third-party charts.

---

# 30. `helm show`

Useful commands:

```bash
helm show chart bitnami/nginx
```

Shows:

```text
Chart metadata
```

```bash
helm show values bitnami/nginx
```

Shows:

```text
Default values
```

```bash
helm show readme bitnami/nginx
```

Shows:

```text
README
```

And:

```bash
helm show all bitnami/nginx
```

shows all available chart information.

---

# 31. `helm pull`

Download a chart without installing it:

```bash
helm pull bitnami/nginx
```

To extract it:

```bash
helm pull bitnami/nginx --untar
```

Now you'll have:

```text
nginx/
├── Chart.yaml
├── values.yaml
└── templates/
```

---

# 32. `helm package`

Suppose you have:

```text
helm-demo/
```

Package it:

```bash
helm package helm-demo
```

You might get:

```text
helm-demo-0.1.0.tgz
```

That `.tgz` is a packaged Helm chart.

---

# 33. Helm dependency

Suppose your application needs Redis.

Your chart can declare dependencies in `Chart.yaml`:

```yaml
dependencies:
  - name: redis
    version: "..."
    repository: "..."
```

Then:

```bash
helm dependency update
```

or:

```bash
helm dependency build
```

This can populate:

```text
charts/
```

with dependent charts.

Architecture:

```text
myapp
 |
 +-- application templates
 |
 +-- Redis dependency
 |
 +-- PostgreSQL dependency
```

---

# 34. Helm template functions

Helm isn't just simple variable substitution.

It provides functions.

For example:

```yaml
name: {{ .Values.app.name | quote }}
```

Another:

```yaml
{{ .Values.labels | toYaml | nindent 4 }}
```

Useful functions you'll frequently encounter:

```text
quote
default
toYaml
nindent
indent
upper
lower
replace
trim
required
include
tpl
```

---

# 35. `default`

Suppose:

```yaml
service:
  type: ""
```

You can do:

```yaml
type: {{ .Values.service.type | default "ClusterIP" }}
```

If the value is empty:

```text
ClusterIP
```

is used.

---

# 36. `required`

You can force a value to exist:

```yaml
image:
  repository: ""
```

Template:

```yaml
image: {{ required "image.repository is required" .Values.image.repository }}
```

If missing:

```text
Helm rendering fails
```

This is useful for mandatory configuration.

---

# 37. `if`

You can conditionally render resources.

Values:

```yaml
ingress:
  enabled: false
```

Template:

```yaml
{{- if .Values.ingress.enabled }}

apiVersion: networking.k8s.io/v1
kind: Ingress

metadata:
  name: {{ .Release.Name }}

{{- end }}
```

If:

```yaml
enabled: false
```

the Ingress isn't rendered.

If:

```yaml
enabled: true
```

it is rendered.

---

# 38. Real example: optional HPA

Values:

```yaml
autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
```

Template:

```yaml
{{- if .Values.autoscaling.enabled }}

apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler

metadata:
  name: {{ .Release.Name }}

spec:
  minReplicas: {{ .Values.autoscaling.minReplicas }}
  maxReplicas: {{ .Values.autoscaling.maxReplicas }}

  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: {{ .Release.Name }}-{{ .Values.app.name }}

{{- end }}
```

Now:

```text
autoscaling.enabled=false
        |
        v
No HPA

autoscaling.enabled=true
        |
        v
HPA generated
```

---

# 39. `_helpers.tpl`

In real charts, you don't want to repeatedly write:

```text
{{ .Release.Name }}-{{ .Values.app.name }}
```

You can create reusable helpers.

```text
templates/_helpers.tpl
```

Example:

```gotemplate
{{- define "helm-demo.fullname" -}}
{{ .Release.Name }}-{{ .Values.app.name }}
{{- end }}
```

Then:

```yaml
metadata:
  name: {{ include "helm-demo.fullname" . }}
```

This improves consistency.

---

# 40. `include`

Suppose:

```gotemplate
{{- define "helm-demo.labels" -}}
app: {{ .Values.app.name }}
environment: {{ .Values.app.environment }}
{{- end }}
```

Use:

```yaml
labels:
  {{- include "helm-demo.labels" . | nindent 4 }}
```

The `.` passes the current template context.

This is very common in production charts.

---

# 41. `helm template` with environment values

This is a great way to debug.

```bash
helm template myapp ./helm-demo \
  -f values-prod.yaml
```

You can inspect exactly what Kubernetes will receive.

Then:

```bash
helm lint ./helm-demo
```

and:

```bash
helm upgrade --install myapp ./helm-demo \
  -f values-prod.yaml
```

This is a common workflow.

---

# 42. `helm upgrade --install`

Very useful in CI/CD:

```bash
helm upgrade --install myapp ./helm-demo \
  -n production \
  --create-namespace \
  -f values-prod.yaml
```

Meaning:

```text
Release exists?
    |
 +-- YES → upgrade
 |
 +-- NO → install
```

This makes pipelines idempotent.

---

# 43. Dry run

Before actually installing/upgrading:

```bash
helm upgrade --install myapp ./helm-demo \
  -f values-prod.yaml \
  --dry-run
```

You can also use:

```bash
helm install myapp ./helm-demo \
  --dry-run
```

This is useful for checking the rendered request without actually applying it.

For rendered YAML alone, I still recommend:

```bash
helm template
```

because it is straightforward and doesn't require an API-side dry-run.

---

# 44. Debugging Helm

A very useful command:

```bash
helm template myapp ./helm-demo \
  -f values-prod.yaml \
  --debug
```

If installation fails:

```bash
helm status myapp
```

Then:

```bash
helm get manifest myapp
```

And Kubernetes-level troubleshooting:

```bash
kubectl get events
kubectl describe pod <pod>
kubectl logs <pod>
```

Remember:

> Helm manages the release, but Kubernetes is still responsible for running the resulting resources.

---

# 45. Helm release revisions

Every upgrade creates a new revision.

Example:

```text
Release: payment-api

Revision 1
   ↓
Revision 2
   ↓
Revision 3
```

Check:

```bash
helm history payment-api
```

Rollback:

```bash
helm rollback payment-api 2
```

This is useful when a deployment introduces a bad configuration.

---

# 46. Helm and Kubernetes Secrets

Helm itself can template Secrets.

For example:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: {{ .Release.Name }}-secret

type: Opaque

stringData:
  username: {{ .Values.database.username | quote }}
```

But **don't put real production secrets directly into `values.yaml` in Git**.

Instead consider:

```text
External Secrets Operator
Sealed Secrets
Vault
Cloud secret managers
SOPS
```

This is a very important production consideration.

---

# 47. Helm + Argo CD

Since you work with Argo CD, this is especially important.

You can have:

```text
Git
 |
 +-- Helm chart
 |
 +-- values-prod.yaml
 |
 v
Argo CD
 |
 v
Helm rendering
 |
 v
Kubernetes API
 |
 v
Cluster
```

Argo CD can use Helm as a **templating/package source**.

Conceptually:

```text
Git
 |
 v
Argo CD
 |
 +-- Helm chart
 +-- values
 |
 v
Rendered manifests
 |
 v
Kubernetes
```

Important interview point:

> When Argo CD uses Helm, Helm is generally used to render the manifests; Argo CD remains responsible for GitOps reconciliation and synchronization.

---

# 48. Full project

A realistic small chart might look like:

```text
helm-demo/
│
├── Chart.yaml
│
├── values.yaml
│
├── values-dev.yaml
├── values-prod.yaml
│
├── charts/
│
└── templates/
    │
    ├── _helpers.tpl
    ├── deployment.yaml
    ├── service.yaml
    ├── configmap.yaml
    ├── ingress.yaml
    ├── hpa.yaml
    └── serviceaccount.yaml
```

Deployment flow:

```text
                     Git
                      |
                      v
                 Helm Chart
                      |
             +--------+--------+
             |                 |
        values-dev        values-prod
             |                 |
             v                 v
         Helm render       Helm render
             |                 |
             v                 v
           Dev              Production
```

---

# 49. Most important Helm commands

I'd memorize these:

### Create

```bash
helm create myapp
```

### Install

```bash
helm install myapp ./myapp
```

### List

```bash
helm list
```

### Status

```bash
helm status myapp
```

### Upgrade

```bash
helm upgrade myapp ./myapp
```

### Upgrade or install

```bash
helm upgrade --install myapp ./myapp
```

### Rollback

```bash
helm rollback myapp 1
```

### History

```bash
helm history myapp
```

### Uninstall

```bash
helm uninstall myapp
```

### Render

```bash
helm template myapp ./myapp
```

### Lint

```bash
helm lint ./myapp
```

### Get values

```bash
helm get values myapp
```

### Get all values

```bash
helm get values myapp --all
```

### Get manifests

```bash
helm get manifest myapp
```

### Add repository

```bash
helm repo add <name> <url>
```

### Update repositories

```bash
helm repo update
```

### Search

```bash
helm search repo nginx
```

### Show values

```bash
helm show values <repo>/<chart>
```

### Package

```bash
helm package ./myapp
```

### Dependencies

```bash
helm dependency update
```

---

# 50. Helm interview questions

For a Senior DevOps interview, be ready for:

### Basic

**What is Helm?**

> Kubernetes package manager and templating/release management tool.

**What is a chart?**

> A package containing metadata, templates, values and optional dependencies.

**What is a release?**

> An installed instance of a chart.

---

### Important

**Difference between `values.yaml` and `Chart.yaml`?**

```text
Chart.yaml
 ↓
Chart metadata

values.yaml
 ↓
Default configuration values
```

---

### Important

**How do you deploy different environments?**

```bash
helm upgrade --install myapp ./chart \
  -f values-prod.yaml
```

---

### Important

**How do you debug a Helm chart before deployment?**

```bash
helm lint ./chart

helm template myapp ./chart \
  -f values-prod.yaml
```

---

### Important

**How do you rollback?**

```bash
helm history myapp

helm rollback myapp 2
```

---

### Senior-level

**What happens when Helm upgrade fails?**

You inspect:

```bash
helm status myapp
helm history myapp
helm get manifest myapp
kubectl get events
kubectl describe ...
```

Then determine whether to:

```bash
helm rollback myapp <revision>
```

or fix the chart/values and upgrade again.

---

# 51. Helm vs Kustomize

You may also get this question.

### Helm

```text
Templates
+
Values
+
Packaging
+
Release management
+
Dependencies
```

### Kustomize

```text
Base manifests
+
Overlays
+
Patches
```

Conceptually:

```text
Helm:

Template
   +
Values
   ↓
Rendered YAML


Kustomize:

Base YAML
   +
Overlay/Patches
   ↓
Final YAML
```

Both are useful, and Argo CD supports both.

---

# 52. The Helm mental model

This is what I recommend memorizing:

```text
                         HELM
                          |
              +-----------+-----------+
              |                       |
             CHART                  RELEASE
              |                       |
       Package/templates       Installed instance
              |
       +------+------+
       |             |
 Chart.yaml      values.yaml
       |             |
       |             v
       |         Configuration
       |             |
       +-------> Templates
                     |
                     v
               Rendered YAML
                     |
                     v
              Kubernetes API
                     |
                     v
                  Cluster
```

And the most important production workflow:

```bash
# Validate
helm lint ./myapp

# Render
helm template myapp ./myapp -f values-prod.yaml

# Deploy
helm upgrade --install myapp ./myapp \
  -n production \
  --create-namespace \
  -f values-prod.yaml

# Check
helm status myapp

# History
helm history myapp

# Rollback if needed
helm rollback myapp <revision>
```

### The interview one-liner

> **Helm packages Kubernetes manifests into reusable charts, uses Go templating plus values to generate environment-specific manifests, and manages installed chart instances as releases with revision history and rollback capabilities.**

The three things I'd make sure you can explain **without hesitation** are **`.Values` vs `.Release`, chart vs release, and `helm template → upgrade --install → history → rollback`**.
