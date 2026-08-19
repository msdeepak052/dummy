# RBAC in k8s

**RBAC (Role-Based Access Control)** is one of the most important Kubernetes security topics for a DevOps/Senior DevOps interview.

The easiest mental model is:

```text
WHO
 |
 | ServiceAccount / User / Group
 v
ROLE
 |
 | What can they do?
 v
PERMISSIONS
 |
 v
Kubernetes Resources
```

For example:

```text
Developer
   |
   v
Role
   |
   +-- get Pods
   +-- list Pods
   +-- get Deployments
   |
   v
Namespace: development
```

---

# 1. What is Kubernetes RBAC?

RBAC controls:

> **Who can perform which actions on which Kubernetes resources.**

For example:

```text
Alice
  |
  +-- can get Pods
  +-- can list Pods
  +-- cannot delete Pods
```

Another user:

```text
Admin
  |
  +-- get
  +-- list
  +-- create
  +-- update
  +-- delete
```

So RBAC answers three questions:

```text
WHO?
WHAT?
WHERE?
```

For example:

```text
WHO   → developer
WHAT  → get/list Pods
WHERE → namespace dev
```

---

# 2. Kubernetes RBAC has four main objects

You should know these very well:

```text
Role
ClusterRole
RoleBinding
ClusterRoleBinding
```

Think:

```text
             RBAC
              |
      +-------+-------+
      |               |
    Role         ClusterRole
      |               |
   Namespace       Cluster-wide
      |               |
      +-------+-------+
              |
       +------+------+
       |             |
 RoleBinding   ClusterRoleBinding
```

---

# 3. Role

A **Role defines permissions inside one namespace**.

Example:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: development

rules:
  - apiGroups: [""]
    resources:
      - pods
    verbs:
      - get
      - list
      - watch
```

This means:

> Whoever receives this Role can get/list/watch Pods in the `development` namespace.

Notice:

```yaml
namespace: development
```

That's important.

A Role is **namespace-scoped**.

---

# 4. What are `apiGroups`, `resources`, and `verbs`?

This is the part you need to understand properly.

### `apiGroups`

Which Kubernetes API group?

For core resources:

```yaml
apiGroups: [""]
```

Examples:

```text
Pods
Services
ConfigMaps
Secrets
Nodes
```

For Deployments:

```yaml
apiGroups:
  - apps
```

because Deployments belong to:

```text
apps
```

---

# 5. Resources

Examples:

```yaml
resources:
  - pods
```

or:

```yaml
resources:
  - deployments
  - replicasets
```

You can also specify subresources.

For example:

```text
pods/log
pods/exec
deployments/scale
```

---

# 6. Verbs

Verbs represent actions.

Common ones:

```text
get
list
watch
create
update
patch
delete
deletecollection
```

Think:

```text
get     → read one object
list    → list objects
watch   → watch changes
create  → create
update  → replace/update
patch   → partial update
delete  → delete
```

---

# 7. Complete Role example

Suppose you want a developer to:

```text
Read Pods
Read Deployments
Read Services
```

But not modify anything.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer-readonly
  namespace: development

rules:

  - apiGroups: [""]
    resources:
      - pods
      - services
    verbs:
      - get
      - list
      - watch

  - apiGroups:
      - apps
    resources:
      - deployments
    verbs:
      - get
      - list
      - watch
```

So:

```text
developer-readonly
        |
        +-- Pods
        |    +-- get
        |    +-- list
        |    +-- watch
        |
        +-- Services
        |    +-- get
        |    +-- list
        |    +-- watch
        |
        +-- Deployments
             +-- get
             +-- list
             +-- watch
```

---

# 8. Role alone doesn't give anybody permission

This is **very important**.

Creating:

```text
Role
```

doesn't automatically assign it to someone.

You need:

```text
RoleBinding
```

So:

```text
Role
  |
  | defines permissions
  |
RoleBinding
  |
  | assigns permissions
  v
User / Group / ServiceAccount
```

---

# 9. RoleBinding

Suppose we have:

```text
User: deepak
```

and:

```text
Role: developer-readonly
```

We bind them:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: developer-readonly-binding
  namespace: development

subjects:
  - kind: User
    name: deepak

roleRef:
  kind: Role
  name: developer-readonly
  apiGroup: rbac.authorization.k8s.io
```

Architecture:

```text
                 deepak
                    |
                    v
             RoleBinding
                    |
                    v
          developer-readonly
                    |
          +---------+---------+
          |         |         |
         Pods   Services   Deployments
```

---

# 10. What can Deepak do?

Assuming this is the only permission:

```bash
kubectl get pods -n development
```

✅ Allowed

```bash
kubectl get deployments -n development
```

✅ Allowed

```bash
kubectl get svc -n development
```

✅ Allowed

But:

```bash
kubectl delete pod nginx -n development
```

❌ Forbidden

Because the Role doesn't contain:

```yaml
verbs:
  - delete
```

---

# 11. Test RBAC permissions

This is extremely useful in interviews and real troubleshooting.

```bash
kubectl auth can-i get pods -n development --as=deepak
```

Output:

```text
yes
```

Check delete:

```bash
kubectl auth can-i delete pods -n development --as=deepak
```

Output:

```text
no
```

This command is worth remembering.

---

# 12. ClusterRole

Now let's move to:

```text
ClusterRole
```

A ClusterRole is **not restricted to one namespace by itself**.

It can define permissions for cluster-scoped resources such as:

```text
nodes
persistentvolumes
namespaces
```

and it can also define permissions for namespaced resources.

Example:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-reader

rules:
  - apiGroups: [""]
    resources:
      - nodes
    verbs:
      - get
      - list
      - watch
```

Now this permission is cluster-wide because Nodes are cluster-scoped.

---

# 13. ClusterRoleBinding

To give that ClusterRole to a user:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: node-reader-binding

subjects:
  - kind: User
    name: deepak

roleRef:
  kind: ClusterRole
  name: node-reader
  apiGroup: rbac.authorization.k8s.io
```

Architecture:

```text
                  deepak
                     |
                     v
            ClusterRoleBinding
                     |
                     v
                node-reader
                     |
                     v
                   Nodes
              +------+------+
              |      |      |
            node1  node2  node3
```

Deepak can now:

```bash
kubectl get nodes
```

---

# 14. Role vs ClusterRole

This is one of the most common interview questions.

| Role                                 | ClusterRole                               |
| ------------------------------------ | ----------------------------------------- |
| Namespace-scoped object              | Cluster-scoped object                     |
| Defines permissions                  | Defines permissions                       |
| Usually used for namespace resources | Can be used for cluster resources         |
| Needs RoleBinding                    | Can use RoleBinding or ClusterRoleBinding |
| Example: Pods in `dev`               | Example: Nodes                            |

But there's an important nuance:

> A **ClusterRole can also be bound into a specific namespace using a RoleBinding**.

This is extremely useful.

---

# 15. ClusterRole + RoleBinding

Suppose you create:

```yaml
kind: ClusterRole
metadata:
  name: pod-reader
```

with:

```yaml
resources:
  - pods
verbs:
  - get
  - list
```

Then:

```yaml
kind: RoleBinding
metadata:
  name: pod-reader-binding
  namespace: development

roleRef:
  kind: ClusterRole
  name: pod-reader
```

Now the user gets that ClusterRole's permissions **only within `development`** through that RoleBinding.

So:

```text
ClusterRole
     |
     | RoleBinding
     v
development namespace
```

Not:

```text
all namespaces
```

This distinction is commonly tested.

---

# 16. ServiceAccount

In real Kubernetes workloads, you very often give permissions to a **ServiceAccount**, not a human user.

For example:

```text
Deployment
    |
    v
Pod
    |
    v
ServiceAccount
    |
    v
RBAC
```

Create:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-sa
  namespace: development
```

Then create a Role:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: development

rules:
  - apiGroups: [""]
    resources:
      - pods
    verbs:
      - get
      - list
      - watch
```

Then bind it:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: app-sa-binding
  namespace: development

subjects:
  - kind: ServiceAccount
    name: app-sa
    namespace: development

roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

---

# 17. Use ServiceAccount in Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
  namespace: development

spec:
  serviceAccountName: app-sa

  containers:
    - name: app
      image: nginx
```

Now:

```text
Pod
 |
 +-- ServiceAccount: app-sa
          |
          v
     RoleBinding
          |
          v
        Role
          |
          v
      Pod permissions
```

The application running inside the Pod can authenticate to the Kubernetes API using that identity, subject to the configured ServiceAccount token mechanism.

---

# 18. Real-world example — monitoring application

Suppose you install a monitoring agent.

It needs to:

```text
get Pods
list Pods
watch Pods
get Nodes
```

You shouldn't give it:

```text
create
delete
update
```

unless it genuinely needs those permissions.

You create a ServiceAccount:

```text
monitoring-sa
```

Then permissions:

```text
ClusterRole
    |
    +-- get/list/watch Pods
    +-- get/list/watch Nodes
```

Then:

```text
ClusterRoleBinding
        |
        v
monitoring-sa
```

Architecture:

```text
Monitoring Pod
      |
      v
monitoring-sa
      |
      v
ClusterRoleBinding
      |
      v
ClusterRole
      |
 +----+---------+
 |              |
Pods           Nodes
```

This is a very common RBAC pattern.

---

# 19. Least privilege

This is the most important security principle.

Don't do this:

```yaml
verbs:
  - "*"
resources:
  - "*"
```

unless you have a very specific reason.

Instead:

```yaml
resources:
  - pods

verbs:
  - get
  - list
  - watch
```

Give only what the application actually needs.

Think:

```text
BAD:

Application
    |
    v
Can do EVERYTHING
```

versus:

```text
GOOD:

Application
    |
    +-- get Pods
    +-- list Pods
    +-- watch Pods

Nothing else
```

---

# 20. Wildcards

You can technically do:

```yaml
apiGroups:
  - "*"

resources:
  - "*"

verbs:
  - "*"
```

That effectively gives very broad permissions.

This is dangerous because it can allow:

```text
create
delete
modify
secrets
deployments
roles
rolebindings
etc.
```

Use explicit permissions wherever possible.

---

# 21. RBAC API groups example

Suppose you want to allow reading:

```text
Pods
Deployments
Ingress
```

You need different API groups.

```yaml
rules:

  # Pods
  - apiGroups: [""]
    resources:
      - pods
    verbs:
      - get
      - list

  # Deployments
  - apiGroups:
      - apps
    resources:
      - deployments
    verbs:
      - get
      - list

  # Ingress
  - apiGroups:
      - networking.k8s.io
    resources:
      - ingresses
    verbs:
      - get
      - list
```

Remember:

```text
Pods
 ↓
core API group
""

Deployments
 ↓
apps

Ingress
 ↓
networking.k8s.io
```

---

# 22. RBAC and Secrets

This is especially important.

Suppose:

```yaml
resources:
  - secrets
verbs:
  - get
```

This means the identity can potentially read Secret contents.

So giving:

```text
get secrets
```

is a **highly sensitive permission**.

For example, don't give a monitoring application:

```yaml
resources:
  - "*"
verbs:
  - "*"
```

because it could potentially access credentials stored in Secrets.

---

# 23. RBAC with namespaces

Suppose:

```text
development
production
```

You give a developer:

```text
RoleBinding → development
```

Then:

```text
kubectl get pods -n development
```

✅

But:

```text
kubectl get pods -n production
```

❌

unless they have another binding granting access to production.

Architecture:

```text
              Developer
                  |
                  v
             RoleBinding
                  |
             development
                  |
            +-----+-----+
            |           |
           Pods      Deployments

production
    |
    X
Developer doesn't have permission
```

This is one of the main benefits of namespace-scoped RBAC.

---

# 24. RoleBinding vs ClusterRoleBinding

Memorize this:

### RoleBinding

```text
RoleBinding
    |
    +--> Role
    |
    +--> User/Group/ServiceAccount
```

Permissions apply within the RoleBinding's namespace.

It can also reference a ClusterRole.

---

### ClusterRoleBinding

```text
ClusterRoleBinding
       |
       +--> ClusterRole
       |
       +--> User/Group/ServiceAccount
```

Permissions are granted cluster-wide according to the ClusterRole's rules.

---

# 25. Full practical demo

Let's build:

```text
developer
    |
    v
RoleBinding
    |
    v
Role
    |
    +-- get/list/watch Pods
    +-- get/list/watch Deployments
    |
    v
development namespace
```

### Step 1 — Namespace

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: development
```

---

### Step 2 — Role

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer-readonly
  namespace: development

rules:
  - apiGroups: [""]
    resources:
      - pods
      - services
    verbs:
      - get
      - list
      - watch

  - apiGroups:
      - apps
    resources:
      - deployments
    verbs:
      - get
      - list
      - watch
```

---

### Step 3 — RoleBinding

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: developer-readonly-binding
  namespace: development

subjects:
  - kind: User
    name: developer

roleRef:
  kind: Role
  name: developer-readonly
  apiGroup: rbac.authorization.k8s.io
```

---

### Step 4 — Test

```bash
kubectl auth can-i get pods \
  --namespace development \
  --as developer
```

```text
yes
```

Test delete:

```bash
kubectl auth can-i delete pods \
  --namespace development \
  --as developer
```

```text
no
```

Test production:

```bash
kubectl auth can-i get pods \
  --namespace production \
  --as developer
```

```text
no
```

---

# 26. The four RBAC objects — final mental model

```text
                     RBAC
                      |
          +-----------+-----------+
          |                       |
        Role                 ClusterRole
          |                       |
   Namespace scoped        Cluster-scoped
          |                       |
          |                 Can define rules
          |                 for cluster resources
          |                       |
          +-----------+-----------+
                      |
               Binding objects
                      |
            +---------+---------+
            |                   |
       RoleBinding      ClusterRoleBinding
            |                   |
            v                   v
      User/Group/SA       User/Group/SA
```

---

# 27. Interview questions you should expect

### Q1. What's the difference between Role and ClusterRole?

> Role is namespace-scoped. ClusterRole is cluster-scoped and can define permissions for cluster-scoped resources such as Nodes, as well as namespaced resources.

### Q2. Does Role give permissions to a user?

> No. Role only defines permissions. RoleBinding assigns those permissions to a User, Group, or ServiceAccount.

### Q3. Can a RoleBinding reference a ClusterRole?

> Yes. A RoleBinding can reference a ClusterRole and restrict the resulting permissions to the RoleBinding's namespace.

### Q4. Difference between RoleBinding and ClusterRoleBinding?

> RoleBinding grants permissions within a namespace, while ClusterRoleBinding grants the referenced ClusterRole's permissions cluster-wide.

### Q5. How do you check whether a user can perform an action?

```bash
kubectl auth can-i get pods -n development --as=developer
```

### Q6. How do you follow least privilege?

> Grant only the required API groups, resources, verbs, and namespaces rather than using wildcard permissions.

---

# 28. One diagram to remember for your interview

```text
                     WHO
                      |
       +--------------+--------------+
       |              |              |
      User          Group       ServiceAccount
       |              |              |
       +--------------+--------------+
                      |
                      v
                   Binding
                /            \
               /              \
       RoleBinding       ClusterRoleBinding
            |                    |
            v                    v
          Role              ClusterRole
            |                    |
            v                    v
       Namespace          Cluster-wide
            |              permissions
            |
      +-----+------+
      |            |
    Pods      Deployments
```

### The one-line answer:

> **Kubernetes RBAC separates identity, permissions, and assignment: Roles/ClusterRoles define what actions are allowed, while RoleBindings/ClusterRoleBindings attach those permissions to Users, Groups, or ServiceAccounts. RoleBindings are namespace-scoped, while ClusterRoleBindings grant cluster-wide permissions.**

For a **Senior DevOps interview**, after basic RBAC, make sure you also know **ServiceAccount → projected token → RBAC → Kubernetes API**, **Role vs ClusterRole with RoleBinding**, **`resourceNames`**, **subresources like `pods/log` and `pods/exec`**, and **how RBAC interacts with Secrets**.
