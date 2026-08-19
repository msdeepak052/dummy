# **Admission Controllers** 

## 1. What is an Admission Controller?

An **Admission Controller** is a component in the Kubernetes API request path that can **inspect and potentially modify or reject an API request after authentication and authorization, but before the object is persisted in etcd**.

Think:

```text
User
  |
  | kubectl apply
  v
API Server
  |
  +--> Authentication
  |
  +--> Authorization
  |
  +--> Admission Control
  |
  +--> Validation
  |
  v
etcd
```

The key point:

> **Admission happens before the object is persisted.**

---

# 2. Why do we need Admission Controllers?

Suppose a developer creates:

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

The Kubernetes API server could accept it.

But your company might have policies like:

```text
Every Pod must:
✓ Have resource limits
✓ Not use privileged containers
✓ Use approved registries
✓ Have required labels
✓ Run as non-root
✓ Have securityContext
```

You don't want every developer to remember all of these.

Admission controllers can enforce them centrally.

---

# 3. Real-world example

Developer submits:

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

Your policy says:

```text
Must have:

resources.limits.memory
resources.limits.cpu
```

Admission controller checks:

```text
Pod request
    |
    v
Admission Controller
    |
    +---- limits present? YES → allow
    |
    +---- limits present? NO → reject
```

The Pod never gets persisted if rejected.

---

# 4. Authentication vs Authorization vs Admission

This is extremely important in interviews.

Suppose:

```bash
kubectl create pod nginx
```

The API server roughly goes through:

```text
                 API Server
                     |
                     v
              Authentication
                     |
              "Who are you?"
                     |
                     v
              Authorization
                     |
             "Can you do this?"
                     |
                     v
              Admission
                     |
             "Should we allow
              this request?"
                     |
                     v
                 Persist
```

### Authentication

Answers:

> **Who are you?**

Examples:

```text
Certificate
OIDC
ServiceAccount token
Cloud IAM integration
```

---

### Authorization

Answers:

> **Are you allowed to perform this action?**

RBAC is the common mechanism.

Example:

```text
developer
   |
   +-- create Pods? YES
```

---

### Admission

Answers:

> **Should this request be allowed according to cluster policy, and should it be modified first?**

Example:

```text
Pod
 |
 +-- privileged=true
 |
 v
Admission
 |
 v
REJECT
```

---

# 5. Admission Controller workflow

A simplified flow:

```text
                  kubectl
                     |
                     v
                API Server
                     |
                     v
              Authentication
                     |
                     v
              Authorization
                     |
                     v
           Mutating Admission
                     |
                     v
          Object may be modified
                     |
                     v
          Validating Admission
                     |
              +------+------+
              |             |
            Allow          Reject
              |             |
              v             v
             etcd          Error
              |
              v
           Kubernetes
```

Modern Kubernetes admission processing has additional internal validation and admission stages, so don't treat the diagram as a literal implementation sequence for every internal check. But for interviews, the key relationship is:

> **Authentication → Authorization → Admission → persistence**, with mutating admission occurring before the final validating phase.

---

# 6. Types of Admission Controllers

There are two major concepts you need to understand:

```text
Mutating
    ↓
Can modify the object

Validating
    ↓
Can accept or reject the object
```

And there are two ways these can be implemented:

```text
Built-in admission plugins

Webhooks
```

So:

```text
Admission
 |
 +----------------------+
 |                      |
Built-in              Webhook
 |                      |
 +----------+-----------+
            |
      +-----+-----+
      |           |
   Mutating    Validating
```

---

# 7. Mutating Admission

A **mutating admission controller can modify the incoming object** before it is persisted.

Example:

Developer sends:

```yaml
containers:
  - name: nginx
    image: nginx
```

Your organization requires:

```yaml
resources:
  requests:
    cpu: 100m
    memory: 128Mi
```

A mutating admission controller can automatically add those values.

So:

```text
Developer request
      |
      v
Mutating Controller
      |
      | adds defaults
      v
Modified object
      |
      v
Validation
      |
      v
Persist
```

---

# 8. Real example: automatic sidecar injection

This is one of the best examples.

Suppose the developer creates:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
  labels:
    app: myapp

spec:
  containers:
    - name: app
      image: myapp:1.0
```

But your service mesh requires a sidecar:

```text
envoy
```

The developer doesn't explicitly add it.

Mutating webhook receives the request:

```text
Pod
 |
 +-- app container
 |
 v
Mutating Webhook
 |
 +-- inject envoy
 |
 v
Pod
 |
 +-- app container
 +-- envoy sidecar
```

The resulting Pod might conceptually become:

```yaml
containers:
  - name: app
    image: myapp:1.0

  - name: envoy
    image: envoy:...
```

This is a classic example of mutation.

---

# 9. Another mutating example — default labels

Developer creates:

```yaml
metadata:
  name: nginx
```

Webhook adds:

```yaml
metadata:
  labels:
    environment: production
    team: payments
```

Result:

```text
Before:

Pod
  |
  +-- name=nginx


After mutation:

Pod
  |
  +-- name=nginx
  +-- environment=production
  +-- team=payments
```

---

# 10. MutatingWebhookConfiguration

To register a mutating webhook, Kubernetes uses:

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingWebhookConfiguration
```

Conceptual example:

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingWebhookConfiguration
metadata:
  name: my-mutating-webhook

webhooks:
  - name: mutate.example.com

    clientConfig:
      service:
        name: mutation-service
        namespace: webhook-system
        path: /mutate

    rules:
      - apiGroups:
          - ""
        apiVersions:
          - v1
        operations:
          - CREATE
        resources:
          - pods

    admissionReviewVersions:
      - v1

    sideEffects: None
```

The important parts are:

```text
MutatingWebhookConfiguration
        |
        +-- Which resources?
        +-- Which operations?
        +-- Which webhook Service?
        +-- Which API versions?
```

---

# 11. How does the webhook actually work?

Suppose:

```text
Pod CREATE request
```

API server matches:

```text
Resource = pods
Operation = CREATE
```

against the webhook rules.

Then:

```text
API Server
    |
    | AdmissionReview request
    v
Webhook Service
    |
    v
Webhook application
```

The webhook responds with an admission decision.

For mutation, it can return a patch.

Conceptually:

```text
Request:

Pod
 |
 +-- container: nginx

Webhook response:

ALLOW
+
PATCH:
add securityContext
add label
add sidecar
```

API server applies the mutation.

---

# 12. AdmissionReview

The communication between the API server and webhook uses an **AdmissionReview** object.

Conceptually:

```text
API Server
    |
    | AdmissionReview
    v
Webhook
    |
    | AdmissionResponse
    v
API Server
```

The request contains information such as:

```text
Resource
Namespace
Operation
User
Object
OldObject
```

depending on the operation/context.

The webhook returns:

```text
allowed: true/false
```

and for mutation may also return a patch.

---

# 13. Validating Admission

Now:

> **Validating admission can approve or reject the request, but does not modify the object.**

Example policy:

```text
Image must come from:
registry.company.com
```

Developer:

```yaml
image: docker.io/nginx
```

Webhook:

```text
Is image from approved registry?
       |
       v
      NO
       |
       v
    REJECT
```

The Pod isn't created.

---

# 14. Validating webhook example

Conceptually:

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: image-policy

webhooks:
  - name: validate.example.com

    clientConfig:
      service:
        name: validation-service
        namespace: policy-system
        path: /validate

    rules:
      - apiGroups:
          - ""
        apiVersions:
          - v1
        operations:
          - CREATE
          - UPDATE
        resources:
          - pods

    admissionReviewVersions:
      - v1

    sideEffects: None
```

The webhook checks:

```text
image = docker.io/nginx
```

Policy:

```text
Only registry.company.com/*
```

Result:

```text
allowed: false
```

API server returns an error to the user.

---

# 15. Mutating vs Validating

This is the most important comparison.

|                     | Mutating             | Validating         |
| ------------------- | -------------------- | ------------------ |
| Can modify object?  | ✅                    | ❌                  |
| Can reject?         | ✅                    | ✅                  |
| Can add defaults?   | ✅                    | ❌                  |
| Can inject sidecar? | ✅                    | ❌                  |
| Can enforce policy? | ✅                    | ✅                  |
| Typical use         | Injection/defaulting | Policy enforcement |

Remember:

```text
Mutating
   ↓
CHANGE IT

Validating
   ↓
ALLOW or REJECT
```

---

# 16. Why is mutation before validation?

Suppose:

```text
Developer creates Pod
```

without:

```yaml
resources:
  limits:
```

Mutating webhook adds:

```yaml
resources:
  limits:
    cpu: 500m
    memory: 512Mi
```

Then validating webhook checks:

```text
Does Pod have resource limits?
       |
       v
      YES
       |
       v
Allow
```

So conceptually:

```text
Original object
      |
      v
Mutating
      |
      v
Modified object
      |
      v
Validating
      |
      v
Persist
```

This ordering is important.

---

# 17. Real-world example: Security policy

Imagine your organization says:

```text
No privileged containers.
```

Developer creates:

```yaml
containers:
  - name: app
    image: myapp:1.0
    securityContext:
      privileged: true
```

Validating webhook:

```text
privileged=true?
       |
       v
      YES
       |
       v
     REJECT
```

User sees something like:

```text
admission webhook "validate.example.com" denied the request
```

The Pod never reaches the normal running state.

---

# 18. Real-world example: Image registry policy

Policy:

```text
Only approved registry:
registry.company.com
```

Developer:

```yaml
image: nginx:latest
```

Validation:

```text
registry.company.com?
       |
       v
      NO
       |
       v
    REJECT
```

Developer must use:

```yaml
image: registry.company.com/platform/nginx:1.2.3
```

---

# 19. Real-world example: Automatic sidecar injection

Service mesh scenario:

```text
Developer
   |
   v
Pod
   |
   v
Mutating Webhook
   |
   +-- Inject sidecar
   |
   v
Pod
 +-------------+
 | app         |
 | sidecar     |
 +-------------+
```

Examples of systems that can use admission webhooks for mutation include service meshes and policy engines.

---

# 20. Kubernetes built-in admission controllers

Not every admission controller is an external webhook.

Kubernetes has built-in admission plugins.

Examples include:

```text
NamespaceLifecycle
LimitRanger
ServiceAccount
ResourceQuota
PodSecurity
```

The exact enabled/default set depends on Kubernetes version and configuration.

For example:

### ResourceQuota

Suppose:

```text
Namespace quota:
CPU = 10
Memory = 20Gi
```

Developer tries to create Pods that exceed the quota.

Admission can reject the request.

```text
Pod request
    |
    v
ResourceQuota admission
    |
    +-- quota exceeded
    |
    v
REJECT
```

---

# 21. LimitRanger

Suppose you want every container to have default resources.

You can configure a LimitRange:

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-resources
  namespace: development

spec:
  limits:
    - type: Container

      default:
        cpu: "500m"
        memory: "512Mi"

      defaultRequest:
        cpu: "100m"
        memory: "128Mi"
```

Now a Pod that doesn't specify resources can receive defaults through the relevant admission processing.

Conceptually:

```text
Pod
 |
 | no resources
 v
LimitRanger
 |
 | inject defaults
 v
Pod
 |
 +-- requests
 +-- limits
```

This is a great example of **built-in admission behavior**.

---

# 22. Pod Security Admission

Modern Kubernetes has **Pod Security Admission (PSA)** as a built-in admission controller.

It can enforce Pod Security Standards such as:

```text
Privileged
Baseline
Restricted
```

For example, a namespace can be labeled to enforce the Restricted profile:

```bash
kubectl label namespace production \
  pod-security.kubernetes.io/enforce=restricted
```

Then a Pod violating the Restricted policy can be rejected.

Conceptually:

```text
Pod
 |
 v
Pod Security Admission
 |
 +-- compliant → ALLOW
 |
 +-- violation → REJECT
```

This is a very useful modern Kubernetes interview topic.

---

# 23. Where do Kyverno and Gatekeeper fit?

This is also important for DevOps interviews.

Tools such as:

```text
Kyverno
OPA Gatekeeper
```

can provide policy enforcement using Kubernetes admission webhooks.

Conceptually:

```text
                 API Server
                     |
                     v
               Admission
                     |
          +----------+----------+
          |                     |
       Kyverno             Gatekeeper
          |                     |
       Policies              Policies
          |                     |
          v                     v
       Allow/Reject          Allow/Reject
```

Kyverno can also perform mutations depending on the policy.

---

# 24. Complete production example

Suppose your company has these requirements:

```text
1. Every Pod needs labels
2. Images must come from company registry
3. Pods cannot run privileged
4. Applications need sidecar injection
5. Resource requests/limits should exist
```

Possible architecture:

```text
                        API Server
                            |
                            v
                    Admission Control
                            |
        +-------------------+-------------------+
        |                   |                   |
        v                   v                   v
    Mutating             Validating        Built-in
    Webhooks             Webhooks          Admission
        |                   |                   |
        v                   v                   v
Sidecar injection     Image policy       Pod Security
Default labels        Security policy    ResourceQuota
        |                   |                   |
        +-------------------+-------------------+
                            |
                            v
                           etcd
```

---

# 25. Full request example

Developer runs:

```bash
kubectl apply -f app.yaml
```

The Pod contains:

```yaml
containers:
  - name: app
    image: registry.company.com/payments/api:1.0
```

But developer didn't specify:

```text
resources
sidecar
labels
```

The flow:

```text
kubectl
  |
  v
API Server
  |
  v
Authentication
  |
  v
Authorization
  |
  v
Mutating Admission
  |
  +-- add labels
  +-- add resource defaults
  +-- inject sidecar
  |
  v
Validating Admission
  |
  +-- registry allowed? YES
  +-- privileged? NO
  +-- policy compliant? YES
  |
  v
Persist
  |
  v
etcd
```

The final stored object is not necessarily identical to what the developer originally submitted because mutation may have changed it.

---

# 26. What happens if a webhook is down?

This is a **very good senior-level interview question**.

Suppose:

```text
API Server
    |
    v
Validating Webhook
    |
    X
   DOWN
```

What happens depends on:

```yaml
failurePolicy:
```

---

# 27. `failurePolicy: Fail`

```yaml
failurePolicy: Fail
```

Means:

> If the webhook can't be reached or produces an error, reject/fail the admission request.

```text
API Server
    |
    v
Webhook
    |
    X
Unavailable
    |
    v
REQUEST FAILS
```

This is appropriate for **critical security policies**, but you must understand the availability impact.

---

# 28. `failurePolicy: Ignore`

```yaml
failurePolicy: Ignore
```

Means:

> If the webhook can't be reached, ignore the webhook failure and continue admission.

```text
API Server
    |
    v
Webhook
    |
    X
Unavailable
    |
    v
Ignore
    |
    v
Continue
```

This improves availability but creates a potential policy bypass.

---

# 29. Security vs availability

This gives you a classic production trade-off:

```text
failurePolicy: Fail
        |
        +-- Stronger enforcement
        +-- Webhook outage can block workloads


failurePolicy: Ignore
        |
        +-- Better availability
        +-- Policy may be bypassed during outage
```

For a critical security control, blindly choosing `Ignore` can be dangerous.

---

# 30. `namespaceSelector`

You can tell a webhook:

> Only apply this webhook to Pods in namespaces matching certain labels.

For example:

```yaml
namespaceSelector:
  matchLabels:
    environment: production
```

Then:

```text
development namespace
        |
        X
Webhook doesn't apply


production namespace
        |
        v
Webhook applies
```

This is very useful for gradually rolling out policies.

---

# 31. `objectSelector`

You can also match objects using labels.

Conceptually:

```yaml
objectSelector:
  matchLabels:
    policy: enforce
```

Then only matching objects invoke the webhook.

---

# 32. Important webhook failure scenario

Suppose you install a webhook that intercepts:

```text
CREATE Pods
```

and the webhook Service itself runs inside Kubernetes.

If you accidentally make the webhook depend on Pods that are themselves blocked by the webhook, you can create a **deadlock**.

Example:

```text
API Server
   |
   v
Webhook
   |
   v
Webhook Pod
   |
   X
Cannot start because webhook blocks it
```

This is why production webhook design requires:

```text
High availability
Correct namespace exclusions
Careful failurePolicy
Resource selectors
TLS
Monitoring
```

---

# 33. Webhook TLS

The API server communicates securely with admission webhooks.

A webhook configuration contains information such as:

```yaml
clientConfig:
  service:
    name: my-webhook
    namespace: webhook-system
    path: /validate
```

and the API server needs to trust the webhook's serving certificate.

In practice you'll commonly see:

```yaml
caBundle: ...
```

or certificate management handled by your deployment tooling.

So:

```text
API Server
   |
   | HTTPS
   v
Webhook Service
   |
   v
Webhook Pod
```

---

# 34. Mutating vs Validating webhook flow

This is worth memorizing:

```text
                  API Server
                      |
                      v
             Mutating Webhooks
                      |
              +-------+-------+
              |               |
           mutate           reject
              |               |
              v               v
       modified object      STOP
              |
              v
         Validation
              |
      +-------+-------+
      |               |
    allow            reject
      |               |
      v               v
     etcd            STOP
```

The key:

```text
Mutating → can CHANGE
Validating → can ACCEPT/REJECT
```

---

# 35. Admission vs RBAC

You asked about RBAC just before this, so this connection is important.

Suppose:

```text
developer
```

has RBAC permission:

```text
create pods
```

So authorization says:

```text
RBAC
 |
 +-- create Pod?
       |
       +-- YES
```

But admission can still say:

```text
Image from approved registry?
       |
       +-- NO
       |
       v
     REJECT
```

Therefore:

> **RBAC permission does not guarantee the request will be admitted.**

Architecture:

```text
Developer
   |
   v
Authentication
   |
   v
Authorization / RBAC
   |
   | "Can you create?"
   | YES
   v
Admission
   |
   | "Should this Pod be created?"
   | NO
   v
Rejected
```

This distinction is excellent for interviews.

---

# 36. Admission vs NetworkPolicy

Don't confuse these either.

### Admission

Controls:

```text
Can this Kubernetes object be created/updated?
```

### NetworkPolicy

Controls:

```text
Can this Pod communicate with another endpoint?
```

So:

```text
Admission
   ↓
API object policy

NetworkPolicy
   ↓
Network traffic policy
```

---

# 37. Demo: validating webhook concept

Imagine you write a webhook application in Python/Go.

It receives:

```text
Pod CREATE
```

It checks:

```text
for container in pod.spec.containers:
    if not image.startswith("registry.company.com/"):
        reject
```

Response:

```text
allowed = false
status.message = "Only approved registry is allowed"
```

Then:

```bash
kubectl apply -f pod.yaml
```

returns something conceptually like:

```text
Error from server:
admission webhook "validate.example.com"
denied the request:
Only approved registry is allowed
```

The Pod doesn't get created.

---

# 38. Demo: mutating webhook concept

Developer:

```yaml
metadata:
  name: app
```

Webhook receives it.

It generates a patch:

```text
ADD:
metadata.labels.team = payments
```

API server applies the patch.

Final object:

```yaml
metadata:
  name: app
  labels:
    team: payments
```

So:

```text
Request
  |
  v
Mutating Webhook
  |
  +-- JSONPatch
  |
  v
API Server
  |
  v
Modified object
```

---

# 39. One important modern security point

Don't use admission webhooks when a built-in Kubernetes mechanism already solves the requirement unless you have a good reason.

For example:

```text
Pod security
    ↓
Pod Security Admission
```

may be preferable to creating custom policy logic.

Similarly:

```text
Resource limits/defaults
    ↓
LimitRange
```

can solve many simple namespace-level defaults.

Use webhooks/policy engines when you need more sophisticated organizational policies.

---

# 40. Interview cheat sheet

```text
Authentication
     ↓
WHO ARE YOU?

Authorization
     ↓
CAN YOU DO THIS?
     ↓
RBAC

Admission
     ↓
SHOULD THIS REQUEST BE ACCEPTED/MODIFIED?
```

Then:

```text
Admission
    |
    +-- Built-in plugins
    |
    +-- Webhooks
          |
          +-- Mutating
          |      |
          |      +-- Modify object
          |
          +-- Validating
                 |
                 +-- Allow / Reject
```

### Mutating example

```text
Pod
 |
 v
Mutating Webhook
 |
 +-- inject sidecar
 +-- add labels
 +-- add defaults
 |
 v
Modified Pod
```

### Validating example

```text
Pod
 |
 v
Validating Webhook
 |
 +-- privileged? YES
 |
 v
REJECT
```

### Webhook failure

```text
failurePolicy: Fail
    ↓
Webhook unavailable → request fails

failurePolicy: Ignore
    ↓
Webhook unavailable → continue
```

---

# 41. The answer I'd give in a Senior DevOps interview

> **Admission controllers are part of the Kubernetes API request pipeline and operate after authentication and authorization but before the object is persisted. They provide a policy and object-processing layer. Built-in admission plugins can enforce things like Pod Security, resource quotas, and defaults. Admission webhooks allow external services to participate in admission. Mutating webhooks can modify an object, for example injecting a service-mesh sidecar or adding defaults, while validating webhooks can only approve or reject the request, such as enforcing an approved container registry.**
>
> **The typical flow is authentication → authorization/RBAC → mutating admission → validation → persistence. For webhooks, `failurePolicy: Fail` prioritizes enforcement but can block workloads if the webhook is unavailable, while `Ignore` prioritizes availability but can allow requests through when the webhook is down.**

And the diagram to remember:

```text
                     kubectl
                        |
                        v
                   API Server
                        |
                        v
                 Authentication
                   "WHO ARE YOU?"
                        |
                        v
                 Authorization
                  "CAN YOU?"
                        |
                        v
               ┌─────────────────┐
               │    MUTATING     │
               │    WEBHOOKS     │
               │                 │
               │ modify object   │
               │ inject sidecar  │
               │ add defaults    │
               └────────┬────────┘
                        |
                        v
               ┌─────────────────┐
               │   VALIDATING    │
               │    WEBHOOKS     │
               │                 │
               │  ALLOW/REJECT   │
               │  policy checks  │
               └────────┬────────┘
                        |
                 +------+------+
                 |             |
               ALLOW         REJECT
                 |             |
                 v             v
                etcd         Error
                 |
                 v
              Kubernetes
```

**One sentence to never forget:**

> **RBAC answers "Can you do it?", while Admission answers "Should this particular object be allowed, and does it need to be modified before it is stored?"**
