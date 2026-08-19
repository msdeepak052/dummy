# Kubernetes Pod Security

**Pod Security** is about controlling **what a Pod is allowed to do at the Linux/kernel/container level**.

For example, you may want to prevent a workload from:

* Running as `root`
* Running privileged containers
* Accessing host namespaces
* Mounting sensitive host paths
* Adding dangerous Linux capabilities
* Running with an overly permissive security context

The key Kubernetes feature to know is **Pod Security Admission (PSA)**.

---

# 1. The big picture

Think of Kubernetes security as multiple layers:

```text
                    Kubernetes Security
                           |
          +----------------+----------------+
          |                |                |
          v                v                v
        RBAC          Pod Security       NetworkPolicy
          |                |                |
     WHO can use     WHAT a Pod can     WHO can talk
     Kubernetes      do on the node     to whom
        API
```

So:

```text
RBAC
 ↓
Can this identity create/update this resource?

Pod Security
 ↓
Is this Pod configuration allowed?

NetworkPolicy
 ↓
Can this Pod communicate with another endpoint?
```

---

# 2. What is Pod Security Admission?

**Pod Security Admission (PSA)** is a built-in Kubernetes admission controller that enforces the **Pod Security Standards (PSS)**.

It evaluates Pods against security policies when Pods are created or updated.

Flow:

```text
kubectl apply
      |
      v
API Server
      |
      v
Authentication
      |
      v
Authorization / RBAC
      |
      v
Pod Security Admission
      |
   +--+--+
   |     |
 ALLOW  REJECT
   |
   v
 Pod
```

So PSA is an **admission-time security control**.

---

# 3. Pod Security Standards

Kubernetes defines three security levels:

```text
Privileged
Baseline
Restricted
```

Think of them as:

```text
More secure
    ↑
Restricted
    |
Baseline
    |
Privileged
    ↓
Less restrictive
```

---

# 4. Privileged

`privileged` basically means:

> Don't restrict the Pod using the Pod Security Standards.

It is intended for highly trusted workloads that need broad privileges.

Examples might include certain:

```text
node-level agents
system components
specialized infrastructure workloads
```

Example namespace:

```bash
kubectl label namespace dev \
  pod-security.kubernetes.io/enforce=privileged
```

This is the least restrictive profile.

**Important:** Privileged does not mean Kubernetes automatically gives every possible Linux capability in every configuration; it means the PSS policy isn't restricting the Pod.

---

# 5. Baseline

Baseline is designed to prevent **known privilege-escalation-style Pod configurations** while still allowing many normal workloads.

It blocks or restricts things such as:

```text
Privileged containers
HostNetwork
HostPID
HostIPC
Certain dangerous capabilities
HostPath usage in restricted contexts
```

The exact controls depend on the Kubernetes Pod Security Standards version.

Think:

```text
Normal application
       |
       v
   Baseline
       |
       +-- Common dangerous settings blocked
```

---

# 6. Restricted

Restricted is much more security-focused.

It expects workloads to follow stronger security practices such as:

```text
run as non-root
drop unnecessary capabilities
use seccomp
restrict privilege escalation
use safer volume types
```

This is the profile you generally want for workloads that can operate under stronger restrictions.

Think:

```text
Restricted
    |
    +-- Non-root
    +-- Minimal privileges
    +-- Seccomp
    +-- Restricted capabilities
    +-- Stronger securityContext
```

---

# 7. Quick comparison

| Profile        | Security | Restrictions                      |
| -------------- | -------- | --------------------------------- |
| **Privileged** | Lowest   | Minimal                           |
| **Baseline**   | Medium   | Blocks known risky configurations |
| **Restricted** | Highest  | Strong security requirements      |

Don't think:

```text
Privileged = bad
```

Some infrastructure workloads genuinely need privileges.

Instead:

> **Use the least permissive profile that allows the workload to function.**

---

# 8. Example: privileged container

Consider:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: dangerous
spec:
  containers:
    - name: app
      image: nginx
      securityContext:
        privileged: true
```

This container has elevated privileges.

Under a namespace enforcing Baseline or Restricted, this can be rejected.

```text
Pod
 |
 +-- privileged: true
 |
 v
Pod Security Admission
 |
 v
REJECT
```

---

# 9. Enable Pod Security on a namespace

For example:

```bash
kubectl label namespace production \
  pod-security.kubernetes.io/enforce=restricted
```

Now Kubernetes will enforce the Restricted profile for Pods in that namespace.

Check:

```bash
kubectl get namespace production --show-labels
```

You may see:

```text
pod-security.kubernetes.io/enforce=restricted
```

---

# 10. Three PSA modes

This is very important.

Pod Security Admission supports:

```text
enforce
audit
warn
```

These are **modes**, not security levels.

The security levels are:

```text
Privileged
Baseline
Restricted
```

The modes are:

```text
enforce
audit
warn
```

Don't mix them up.

---

# 11. `enforce`

```bash
kubectl label namespace production \
  pod-security.kubernetes.io/enforce=restricted
```

`enforce` means:

> Reject Pods that violate the specified policy.

Example:

```text
Pod
 |
 | violates Restricted
 v
PSA
 |
 v
REJECT
```

This is the strongest operational behavior.

---

# 12. `warn`

```bash
kubectl label namespace production \
  pod-security.kubernetes.io/warn=restricted
```

The Pod can still be created, but the user receives a warning.

Example:

```text
kubectl apply -f pod.yaml
```

could produce a warning indicating that the Pod violates the Restricted policy.

Think:

```text
warn
 ↓
Allow
 +
Warning
```

This is excellent when migrating existing workloads toward a stronger security posture.

---

# 13. `audit`

```bash
kubectl label namespace production \
  pod-security.kubernetes.io/audit=restricted
```

The Pod is allowed, but violations are recorded in the **API audit trail**.

Think:

```text
audit
 ↓
Allow
 +
Audit information
```

---

# 14. Use all three together

A very practical rollout strategy:

```text
warn=restricted
audit=restricted
enforce=baseline
```

Meaning:

```text
Enforce:
Baseline

Warn:
Restricted

Audit:
Restricted
```

This lets you discover which workloads will fail Restricted before actually enforcing Restricted.

This is a good production migration strategy.

---

# 15. Example namespace configuration

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: payments

  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

Now:

```text
payments namespace
       |
       v
Pod Security Admission
       |
       v
Restricted policy
```

---

# 16. `runAsNonRoot`

One of the most important Pod security settings is:

```yaml
securityContext:
  runAsNonRoot: true
```

Example:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-app
spec:
  securityContext:
    runAsNonRoot: true

  containers:
    - name: app
      image: nginx
```

This tells Kubernetes:

> The container should not run as UID 0.

---

# 17. Specify an explicit UID

You can be more explicit:

```yaml
securityContext:
  runAsUser: 10001
  runAsGroup: 10001
  runAsNonRoot: true
```

Now:

```text
Container
   |
   +-- UID = 10001
   +-- GID = 10001
   |
   X
 UID 0 / root
```

This is stronger than simply relying on the image's default user.

---

# 18. `allowPrivilegeEscalation`

You can prevent processes from gaining additional privileges:

```yaml
securityContext:
  allowPrivilegeEscalation: false
```

Example:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-app

spec:
  securityContext:
    runAsNonRoot: true

  containers:
    - name: app
      image: myapp:1.0

      securityContext:
        allowPrivilegeEscalation: false
```

Think:

```text
Application
    |
    X
Can't escalate privileges
```

---

# 19. Linux capabilities

Linux capabilities split traditional root privileges into smaller units.

Instead of giving an application broad root-like power, you can remove capabilities it doesn't need.

Example:

```yaml
securityContext:
  capabilities:
    drop:
      - ALL
```

This is a very common security-hardening pattern.

Architecture:

```text
Container
   |
   +-- capabilities
         |
         +-- DROP ALL
```

If the application genuinely needs one capability, you can add only that one.

For example:

```yaml
capabilities:
  drop:
    - ALL
  add:
    - NET_BIND_SERVICE
```

This is much better than making the entire container privileged.

---

# 20. Why `NET_BIND_SERVICE`?

Normally, Linux restricts binding to ports below 1024.

For example:

```text
80
443
```

An application may need the ability to bind to a privileged port.

Instead of:

```yaml
privileged: true
```

you could potentially grant:

```yaml
capabilities:
  add:
    - NET_BIND_SERVICE
```

This follows the principle:

> **Grant the smallest privilege required.**

---

# 21. Seccomp

Seccomp restricts the Linux system calls a process can make.

A common secure setting is:

```yaml
securityContext:
  seccompProfile:
    type: RuntimeDefault
```

Example:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-app

spec:
  securityContext:
    runAsNonRoot: true
    seccompProfile:
      type: RuntimeDefault

  containers:
    - name: app
      image: myapp:1.0

      securityContext:
        allowPrivilegeEscalation: false

        capabilities:
          drop:
            - ALL
```

This is a much stronger security posture than:

```yaml
privileged: true
```

---

# 22. Full secure Pod example

Here's a good interview/demo example:

```yaml
apiVersion: v1
kind: Pod

metadata:
  name: secure-app

spec:

  securityContext:
    runAsUser: 10001
    runAsGroup: 10001
    runAsNonRoot: true

    seccompProfile:
      type: RuntimeDefault

  containers:

    - name: app
      image: nginx:1.27

      securityContext:
        allowPrivilegeEscalation: false

        capabilities:
          drop:
            - ALL

      ports:
        - containerPort: 8080
```

The important security settings are:

```text
runAsNonRoot
runAsUser
runAsGroup
allowPrivilegeEscalation=false
capabilities.drop=ALL
seccompProfile=RuntimeDefault
```

---

# 23. Pod-level vs container-level securityContext

You can specify security settings at:

```text
Pod level
```

or:

```text
Container level
```

Example:

```yaml
spec:
  securityContext:
    runAsNonRoot: true

  containers:
    - name: app
      securityContext:
        allowPrivilegeEscalation: false
```

Think:

```text
Pod
 |
 +-- Pod securityContext
 |
 +-- Container A
 |      |
 |      +-- container securityContext
 |
 +-- Container B
```

Some fields are applicable at Pod level, some at container level, and some at both with specific semantics.

---

# 24. `hostNetwork`

This is another important security concern.

Normally:

```text
Pod
 |
 v
Pod network namespace
 |
 v
Pod IP
```

With:

```yaml
hostNetwork: true
```

the Pod uses the node's network namespace.

Conceptually:

```text
Normal:

Node
 |
 +-- Pod network namespace
       |
       +-- Pod IP


hostNetwork=true:

Node network namespace
       |
       +-- Pod
```

This gives the Pod much closer access to the node's networking environment and is restricted by Baseline/Restricted policies.

---

# 25. `hostPID`

Normally:

```text
Pod
 |
 +-- own process namespace
```

With:

```yaml
hostPID: true
```

the Pod can share the node's PID namespace.

Conceptually:

```text
Normal:
Pod sees its own processes

hostPID:
Pod can potentially see node processes
```

This is a significant security boundary and is restricted by Pod Security Standards.

---

# 26. `hostIPC`

Similarly:

```yaml
hostIPC: true
```

shares the host IPC namespace.

Again:

```text
hostIPC
   |
   v
Host IPC namespace
```

This can expose IPC mechanisms between the workload and host.

---

# 27. HostPath

Consider:

```yaml
volumes:
  - name: host
    hostPath:
      path: /etc
```

You're mounting a directory from the node into the Pod.

Conceptually:

```text
Node
 |
 +-- /etc
 |
 v
Pod
 |
 +-- /host/etc
```

This is dangerous because the workload can gain access to sensitive host files depending on the path and permissions.

That's why Pod Security Standards place restrictions on hostPath.

---

# 28. Privileged vs capabilities

Interview question:

> Why not just use `privileged: true`?

Because privileged containers grant extremely broad privileges.

Instead:

```text
Need one capability?
        |
        v
Add only that capability
```

Example:

```yaml
capabilities:
  drop:
    - ALL
  add:
    - NET_BIND_SERVICE
```

This follows:

```text
Least privilege
```

rather than:

```text
Give everything
```

---

# 29. Pod Security vs RBAC

You just learned RBAC, so compare them.

Suppose:

```text
developer
```

has:

```text
RBAC:
create Pods
```

So:

```text
Authorization
     |
     v
Can developer create Pod?
     |
     v
YES
```

But the Pod contains:

```yaml
privileged: true
```

Then:

```text
Pod Security Admission
       |
       v
Restricted policy
       |
       v
REJECT
```

So:

```text
RBAC:
"Are you allowed to create a Pod?"

Pod Security:
"Is this Pod configuration allowed?"
```

Both can block the overall operation, but at different layers.

---

# 30. Pod Security vs NetworkPolicy

Don't confuse these.

### Pod Security

Controls the **security properties of the workload**:

```text
root?
privileged?
hostNetwork?
capabilities?
seccomp?
```

### NetworkPolicy

Controls **network communication**:

```text
Pod A → Pod B?
Pod → database?
Pod → internet?
```

Diagram:

```text
             Pod
              |
       +------+------+
       |             |
       v             v
 Pod Security    NetworkPolicy
       |             |
       v             v
How powerful?    Who can it talk to?
```

---

# 31. Pod Security vs Admission Webhooks

Another important connection to your previous topic.

Pod Security Admission:

```text
Built into Kubernetes
```

Kyverno/Gatekeeper:

```text
Policy engines using admission mechanisms/webhooks
```

Architecture:

```text
                     API Server
                         |
                         v
                    Admission
                         |
             +-----------+-----------+
             |                       |
             v                       v
        Pod Security            Webhooks
        Admission             /           \
             |             Kyverno      Gatekeeper
             |
             v
       PSS enforcement
```

---

# 32. Example: enforce Restricted

Let's do a real demo.

Create namespace:

```bash
kubectl create namespace secure-apps
```

Enable Restricted enforcement:

```bash
kubectl label namespace secure-apps \
  pod-security.kubernetes.io/enforce=restricted
```

Now try:

```yaml
apiVersion: v1
kind: Pod

metadata:
  name: privileged-pod

spec:
  containers:
    - name: nginx
      image: nginx
      securityContext:
        privileged: true
```

Apply:

```bash
kubectl apply -f privileged.yaml
```

You should receive an admission error indicating that the Pod violates the Restricted policy.

The important flow is:

```text
kubectl apply
      |
      v
API Server
      |
      v
PSA
      |
      v
privileged=true
      |
      v
Restricted violation
      |
      v
REJECT
```

---

# 33. Test a compliant Pod

Now create:

```yaml
apiVersion: v1
kind: Pod

metadata:
  name: secure-pod

spec:
  securityContext:
    runAsNonRoot: true
    seccompProfile:
      type: RuntimeDefault

  containers:
    - name: nginx
      image: nginx:1.27

      securityContext:
        allowPrivilegeEscalation: false

        capabilities:
          drop:
            - ALL
```

This is much closer to what a Restricted-compatible workload should look like.

However, **exact compliance depends on the Kubernetes version and the specific Pod Security Standards revision in use**, so for production always test the actual manifest against the cluster's policy.

---

# 34. `warn` is extremely useful for migration

Imagine you have:

```text
500 existing workloads
```

and suddenly enforce:

```text
Restricted
```

You could break many applications.

Instead:

```bash
kubectl label namespace production \
  pod-security.kubernetes.io/warn=restricted
```

Developers start seeing warnings.

Then:

```text
Find violations
      |
      v
Fix applications
      |
      v
Test
      |
      v
Enable audit
      |
      v
Enable enforce
```

A good migration:

```text
warn
 ↓
audit
 ↓
fix workloads
 ↓
enforce
```

You can also use `baseline` as the initial enforced level while preparing for `restricted`.

---

# 35. Production security architecture

A mature Kubernetes platform might look like:

```text
                        Kubernetes
                            |
                     API Server
                            |
              +-------------+-------------+
              |                           |
             RBAC                  Pod Security
              |                       Admission
              |                           |
       WHO can access API?       What can Pod do?
              |                           |
              +-------------+-------------+
                            |
                            v
                       Kubernetes
                            |
                    +-------+-------+
                    |               |
                  Pods        NetworkPolicy
                                    |
                                    v
                            Network access
```

Then add external policy:

```text
API Server
    |
    +-- RBAC
    |
    +-- Pod Security Admission
    |
    +-- Kyverno/Gatekeeper
    |
    v
  Workload
```

---

# 36. What should a production Pod ideally have?

For a typical application that supports it:

```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 10001
  runAsGroup: 10001

  seccompProfile:
    type: RuntimeDefault
```

Container:

```yaml
securityContext:
  allowPrivilegeEscalation: false

  capabilities:
    drop:
      - ALL
```

Then add only specific capabilities if genuinely required.

This gives you:

```text
Non-root
+
No privilege escalation
+
Minimal capabilities
+
Seccomp
=
Stronger container security
```

---

# 37. Interview questions you should expect

### Q1. What is Pod Security Admission?

> A built-in Kubernetes admission controller that enforces Pod Security Standards when Pods are created or updated.

### Q2. What are the three Pod Security levels?

```text
Privileged
Baseline
Restricted
```

### Q3. What are the three PSA modes?

```text
enforce
audit
warn
```

### Q4. Difference between Baseline and Restricted?

> Baseline blocks common dangerous privilege configurations, while Restricted applies much stronger security requirements for workloads that can operate under a hardened security posture.

### Q5. How do you enforce Restricted in a namespace?

```bash
kubectl label namespace production \
  pod-security.kubernetes.io/enforce=restricted
```

### Q6. How do you prevent a container from running as root?

```yaml
securityContext:
  runAsNonRoot: true
```

Optionally specify:

```yaml
runAsUser: 10001
```

### Q7. How do you prevent privilege escalation?

```yaml
securityContext:
  allowPrivilegeEscalation: false
```

### Q8. How do you remove Linux capabilities?

```yaml
capabilities:
  drop:
    - ALL
```

### Q9. What is seccomp?

> A Linux kernel mechanism that restricts which system calls a process can invoke. Kubernetes can configure a container to use a seccomp profile such as `RuntimeDefault`.

---

# 38. Senior DevOps interview answer

If they ask:

> **"How would you secure Pods in a production Kubernetes cluster?"**

A strong answer would be:

> "I'd start with Pod Security Admission and enforce an appropriate Pod Security Standard at the namespace level, typically Baseline or Restricted depending on workload requirements. For hardened workloads I'd use `runAsNonRoot`, an explicit non-root UID where appropriate, `allowPrivilegeEscalation: false`, drop unnecessary Linux capabilities, and use the `RuntimeDefault` seccomp profile. I'd avoid privileged containers, host namespaces and unnecessary hostPath mounts. I'd use `warn` and `audit` first when migrating existing workloads so we can identify violations before enabling enforcement. For additional organization-specific policies, I'd use a policy engine such as Kyverno or Gatekeeper."

---

# 39. The mental model to memorize

```text
                       POD SECURITY
                            |
            +---------------+---------------+
            |               |               |
            v               v               v
       Pod Security     SecurityContext   Linux
        Admission          settings      controls
            |                 |              |
            v                 v              v
       PSS levels       runAsNonRoot     capabilities
       |                runAsUser       seccomp
       |                allowPrivilege   namespaces
       |
   +----+---------+---------+
   |              |         |
Privileged      Baseline  Restricted
```

And remember:

```text
RBAC
 ↓
WHO can use Kubernetes API?

PSA
 ↓
WHAT security posture must the Pod satisfy?

NetworkPolicy
 ↓
WHO can the Pod communicate with?
```

### The single most important interview statement:

> **Pod Security Admission is an admission-time control that enforces Pod Security Standards—Privileged, Baseline, or Restricted—using namespace-level `enforce`, `audit`, and `warn` modes. It complements RBAC rather than replacing it: RBAC controls API permissions, while Pod Security controls the security characteristics of Pods.**
