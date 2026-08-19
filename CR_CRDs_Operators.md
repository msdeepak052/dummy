# CR, CRDs & Operators

This is a **very important Kubernetes extensibility topic**, and the three terms are easy to confuse:

```text
CRD
 ↓
Defines a new Kubernetes resource type

CR
 ↓
An actual object created from that resource type

Operator
 ↓
A controller that watches those objects and performs actions
```

The easiest way to remember:

> **CRD = blueprint**
> **CR = instance of the blueprint**
> **Operator = software that makes the instance actually do something**

Let's build this from zero.

---

# 1. Why do we need CRDs?

Kubernetes comes with built-in resources:

```text
Pod
Deployment
Service
ConfigMap
Secret
StatefulSet
Job
Ingress
```

For example:

```yaml
apiVersion: apps/v1
kind: Deployment
```

But imagine you want Kubernetes to understand a new resource called:

```text
Database
```

You want to be able to write:

```yaml
apiVersion: platform.example.com/v1
kind: Database

metadata:
  name: my-postgres

spec:
  version: "16"
  replicas: 3
  storage: 100Gi
```

Kubernetes doesn't understand `kind: Database` by default.

So you create a:

```text
CRD
CustomResourceDefinition
```

---

# 2. What is a CRD?

A **CustomResourceDefinition (CRD)** extends the Kubernetes API with your own resource type.

Before CRD:

```text
Kubernetes API
 |
 +-- Pods
 +-- Services
 +-- Deployments
 +-- ConfigMaps
```

After installing a CRD:

```text
Kubernetes API
 |
 +-- Pods
 +-- Services
 +-- Deployments
 +-- ConfigMaps
 |
 +-- Databases   ← Custom Resource Type
```

So a CRD essentially tells Kubernetes:

> "There is a new kind of API object called `Database`."

---

# 3. CRD vs CR

This distinction is extremely important.

Suppose:

```text
CRD = Database
```

That means:

> Kubernetes now understands the **Database resource type**.

Then you create:

```text
CR = my-postgres
```

which is one actual Database object.

Think about it like:

```text
CRD
 ↓
Class / schema / blueprint
 ↓
Database


CR
 ↓
Object / instance
 ↓
my-postgres
```

Or:

```text
CRD = "Pod-like resource called Database"

CR 1 = postgres-prod
CR 2 = postgres-dev
CR 3 = postgres-test
```

---

# 4. First CRD example

Let's create a simple `Database` CRD.

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition

metadata:
  name: databases.platform.example.com

spec:
  group: platform.example.com

  names:
    kind: Database
    plural: databases
    singular: database
    shortNames:
      - db

  scope: Namespaced

  versions:
    - name: v1
      served: true
      storage: true

      schema:
        openAPIV3Schema:
          type: object

          properties:
            spec:
              type: object

              properties:
                version:
                  type: string

                replicas:
                  type: integer

                storage:
                  type: string
```

Apply:

```bash
kubectl apply -f database-crd.yaml
```

---

# 5. What did we just create?

Check:

```bash
kubectl get crd
```

You might see:

```text
NAME
databases.platform.example.com
```

Now Kubernetes knows:

```text
Group:
platform.example.com

Version:
v1

Kind:
Database

Plural:
databases
```

You can check:

```bash
kubectl api-resources | grep -i database
```

You might get:

```text
databases   db   platform.example.com/v1
```

---

# 6. Group, Version and Kind

This is another thing interviewers ask.

Our CRD has:

```yaml
group: platform.example.com
```

and:

```yaml
version:
  name: v1
```

and:

```yaml
kind: Database
```

So:

```text
apiVersion:
platform.example.com/v1

kind:
Database
```

Together:

```yaml
apiVersion: platform.example.com/v1
kind: Database
```

Think:

```text
platform.example.com
       |
       +-- API Group

v1
       |
       +-- API Version

Database
       |
       +-- Kind
```

---

# 7. Create a CR

Now Kubernetes understands the new resource.

We can create an actual `Database`.

```yaml
apiVersion: platform.example.com/v1
kind: Database

metadata:
  name: postgres-prod

spec:
  version: "16"
  replicas: 3
  storage: 100Gi
```

Apply:

```bash
kubectl apply -f postgres.yaml
```

Now:

```bash
kubectl get databases
```

or:

```bash
kubectl get db
```

You might see:

```text
NAME            AGE
postgres-prod   10s
```

Congratulations:

> You just created a **Custom Resource (CR)**.

---

# 8. But something is missing

This is where people often misunderstand CRDs.

You created:

```text
Database CRD
```

and:

```text
Database CR
```

But does this automatically create PostgreSQL?

**No.**

This:

```yaml
kind: Database
```

doesn't magically install PostgreSQL.

At this point Kubernetes essentially knows:

```text
"Hey, there is an object called Database."
```

But who is going to act on it?

That's where the **Operator** comes in.

---

# 9. What is an Operator?

An Operator is essentially a **Kubernetes controller containing application/domain-specific operational logic**.

It watches Kubernetes resources and continuously tries to make the real cluster state match the desired state.

For our example:

```text
Database CR
     |
     v
Operator
     |
     +-- Create StatefulSet
     +-- Create Service
     +-- Create Secret
     +-- Create PVC
     +-- Configure PostgreSQL
     +-- Monitor health
     +-- Handle failover
     +-- Backup
     +-- Upgrade
```

This is the key difference:

```text
CRD
=
Defines the API

Operator
=
Implements the behavior
```

---

# 10. Controller pattern

The Operator follows the Kubernetes **control loop**.

```text
                 Desired State
                      |
                      v
                Database CR
                      |
                      v
                  Operator
                      |
                      v
                Observe cluster
                      |
                      v
                Compare state
                      |
              +-------+-------+
              |               |
            Match          Different
              |               |
              |               v
              |           Reconcile
              |               |
              |               v
              |        Create/update resources
              |               |
              +-------<--------+
```

This is called:

```text
Reconciliation
```

---

# 11. Desired vs actual state

Suppose your CR says:

```yaml
spec:
  replicas: 3
```

Operator checks:

```text
Desired:
3 PostgreSQL Pods

Actual:
2 PostgreSQL Pods
```

Operator says:

```text
Actual != Desired
```

Then:

```text
Create another Pod
```

Now:

```text
Desired = 3
Actual = 3
```

Reconciliation is complete.

---

# 12. Operator architecture

A simplified architecture:

```text
                   API Server
                       |
                       |
                 Database CR
                       |
                       v
              +----------------+
              |   Operator     |
              |                |
              | Reconciliation |
              |     Loop       |
              +-------+--------+
                      |
          +-----------+-----------+
          |           |           |
          v           v           v
     StatefulSet    Service      PVC
          |
          v
      PostgreSQL
```

---

# 13. Full example

Let's say the user creates:

```yaml
apiVersion: platform.example.com/v1
kind: Database

metadata:
  name: postgres-prod

spec:
  engine: postgres
  version: "16"
  replicas: 3
  storage: 100Gi
```

The Operator watches:

```text
Database
```

It sees:

```text
postgres-prod
```

Then it creates:

```text
StatefulSet
Service
PVC
Secret
ConfigMap
```

Architecture:

```text
Database CR
     |
     v
Postgres Operator
     |
     +----------+----------+----------+
     |          |          |          |
     v          v          v          v
StatefulSet  Service     PVC       Secret
     |
     +---- PostgreSQL-0
     |
     +---- PostgreSQL-1
     |
     +---- PostgreSQL-2
```

---

# 14. Operator isn't necessarily one specific technology

An Operator is a **pattern**, not a particular programming language.

You can build one using:

```text
Go
Python
Java
Rust
```

But Go + Kubernetes controller frameworks are very common.

Popular frameworks/tools include:

```text
Kubebuilder
Operator SDK
controller-runtime
```

---

# 15. Controller vs Operator

This is a subtle interview question.

A **controller** is a general Kubernetes control-loop concept.

Examples:

```text
Deployment controller
ReplicaSet controller
Job controller
```

An **Operator** is typically a controller that encodes operational knowledge for a particular application/system.

For example:

```text
PostgreSQL Operator
Redis Operator
Kafka Operator
```

So:

```text
Every Operator is controller-like,
but not every controller is normally called an Operator.
```

---

# 16. Example without an Operator

Suppose we only install:

```text
CRD
```

and create:

```text
Database CR
```

We have:

```text
Database CRD
      |
      v
Database CR
```

But:

```text
No Operator
```

Therefore:

```text
No PostgreSQL
No StatefulSet
No PVC
No failover
No backups
```

The CR is just stored as a Kubernetes API object.

---

# 17. Example with Operator

Now install:

```text
Database CRD
```

and:

```text
PostgreSQL Operator
```

Then:

```text
Database CR
      |
      v
Postgres Operator
      |
      +-- StatefulSet
      +-- Service
      +-- PVC
      +-- Secret
      +-- ConfigMap
```

Now the CR has behavior.

---

# 18. Operator watches resources

The Operator typically watches:

```text
Database CR
```

and possibly the resources it owns:

```text
StatefulSet
Pods
Services
PVCs
```

Example:

```text
API Server
    |
    +-- Database CR changed
    |
    v
Operator
    |
    v
Reconcile()
```

---

# 19. What happens if someone deletes a Pod?

Suppose:

```text
Database CR

replicas: 3
```

Current:

```text
postgres-0
postgres-1
postgres-2
```

Someone accidentally deletes:

```bash
kubectl delete pod postgres-1
```

The Operator/controller observes the resulting state.

Depending on the Operator design, the underlying StatefulSet may recreate the Pod, or the Operator may reconcile the state through its managed resources.

The general principle is:

```text
Desired:
3 replicas

Actual:
2 replicas

      ↓

Reconcile

      ↓

Restore desired state
```

This is the essence of Kubernetes operators.

---

# 20. `status` in a CR

A well-designed CR usually has:

```yaml
spec:
```

for:

> What the user wants.

and:

```yaml
status:
```

for:

> What the Operator observes/reports.

Example:

```yaml
apiVersion: platform.example.com/v1
kind: Database

metadata:
  name: postgres-prod

spec:
  version: "16"
  replicas: 3

status:
  phase: Ready
  readyReplicas: 3
```

Think:

```text
spec
 ↓
Desired state

status
 ↓
Observed state
```

This is an extremely important Kubernetes pattern.

---

# 21. Example: status changes

User:

```yaml
spec:
  replicas: 3
```

Operator starts provisioning.

Status:

```yaml
status:
  phase: Provisioning
  readyReplicas: 0
```

Then:

```yaml
status:
  phase: Provisioning
  readyReplicas: 2
```

Finally:

```yaml
status:
  phase: Ready
  readyReplicas: 3
```

This lets users run:

```bash
kubectl get db postgres-prod
```

and understand what's happening.

---

# 22. Conditions

Modern Kubernetes resources often use conditions.

For example:

```yaml
status:
  conditions:
    - type: Ready
      status: "True"
      reason: DatabaseReady
      message: "All database replicas are healthy"
```

Possible states:

```text
Ready=True
Ready=False
Ready=Unknown
```

This is much better than only having:

```text
phase: Ready
```

because conditions can represent multiple aspects of state.

---

# 23. CRD validation

A CRD can define a schema.

For example:

```yaml
spec:
  schema:
    openAPIV3Schema:
      type: object
      properties:
        spec:
          type: object
          properties:
            replicas:
              type: integer
              minimum: 1
              maximum: 5
```

Now this:

```yaml
spec:
  replicas: 10
```

can be rejected by CRD schema validation.

So:

```text
CR
 |
 v
CRD schema
 |
 +-- valid → accepted
 |
 +-- invalid → rejected
```

---

# 24. Why CRD validation is useful

Suppose you want:

```text
replicas = 1-5
```

Without validation:

```yaml
replicas: banana
```

could potentially get through the API schema if you haven't constrained it.

With CRD validation:

```yaml
replicas:
  type: integer
  minimum: 1
  maximum: 5
```

you get:

```text
replicas: banana
        |
        v
   Validation
        |
        v
      REJECT
```

---

# 25. `kubectl explain`

Once your CRD is installed, you can often inspect its schema through:

```bash
kubectl explain database
```

or:

```bash
kubectl explain database.spec
```

and:

```bash
kubectl explain database.spec.replicas
```

This is very useful when working with custom resources.

---

# 26. Namespaced vs cluster-scoped CRDs

When defining the CRD:

```yaml
scope: Namespaced
```

means:

```text
Database
   |
   +-- namespace: production
   +-- namespace: development
```

You can have:

```text
production/postgres
development/postgres
```

If:

```yaml
scope: Cluster
```

then the resource is cluster-scoped.

Example:

```text
Cluster-wide custom resource
```

---

# 27. Full hierarchy

Now put everything together:

```text
                Kubernetes API
                     |
                     v
                    CRD
                     |
        "I define a Database resource"
                     |
                     v
              Database CR
                     |
          "postgres-prod"
                     |
                     v
                 Operator
                     |
               Reconcile()
                     |
        +------------+------------+
        |            |            |
        v            v            v
   StatefulSet    Service        PVC
        |
        v
 PostgreSQL Pods
```

---

# 28. Real-world example: Prometheus Operator

A very useful real-world example is the Prometheus ecosystem.

Instead of manually creating all the Kubernetes objects needed for monitoring, you can define a custom resource such as:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: Prometheus

metadata:
  name: prometheus
```

The Prometheus Operator watches this resource and manages the underlying Kubernetes resources/configuration.

So:

```text
Prometheus CRD
       |
       v
Prometheus CR
       |
       v
Prometheus Operator
       |
       v
Prometheus deployment/configuration
```

This is the Operator pattern in the real world.

---

# 29. Real-world example: Kafka Operator

Suppose you use an Operator for Kafka.

You might create a custom resource conceptually like:

```yaml
apiVersion: kafka.example.com/v1
kind: KafkaCluster

metadata:
  name: production

spec:
  brokers: 3
  storage:
    size: 500Gi
```

The Operator can handle:

```text
Create Kafka brokers
Create Services
Create storage
Configure replication
Handle rolling upgrades
Monitor health
```

Instead of manually managing every object.

---

# 30. Real-world example: Redis Operator

Similarly:

```yaml
apiVersion: redis.example.com/v1
kind: Redis

metadata:
  name: redis-prod

spec:
  replicas: 3
```

Operator:

```text
Redis CR
   |
   v
Redis Operator
   |
   +-- StatefulSet
   +-- Services
   +-- Config
   +-- Secrets
   +-- Failover logic
```

The exact capabilities depend on the specific Redis Operator.

---

# 31. Operator = Kubernetes-native automation

This is the big idea.

Normally, an engineer might do:

```text
Install PostgreSQL
Configure replication
Create users
Create storage
Configure backups
Monitor health
Perform upgrades
Recover failures
```

An Operator tries to encode that operational knowledge into a controller:

```text
Human
  |
  v
Database CR
  |
  v
Operator
  |
  +-- install
  +-- configure
  +-- monitor
  +-- recover
  +-- upgrade
  +-- backup
```

That's why the Operator pattern is powerful.

---

# 32. CRD alone vs Operator

This is one of the most important comparisons.

|                                          | CRD            | Operator         |
| ---------------------------------------- | -------------- | ---------------- |
| Defines custom API                       | ✅              | Usually uses CRD |
| Creates custom resource type             | ✅              | No               |
| Stores CR                                | Kubernetes API | Kubernetes API   |
| Performs application-specific automation | ❌              | ✅                |
| Reconciliation loop                      | ❌              | ✅                |
| Creates child resources                  | ❌              | Usually          |
| Handles failures                         | ❌              | Usually          |
| Encodes operational knowledge            | ❌              | ✅                |

Remember:

```text
CRD = API extension

CR = desired configuration

Operator = automation/reconciliation
```

---

# 33. CRD vs ConfigMap

Another interview question.

You might ask:

> Why not just use a ConfigMap instead of a CR?

Because a ConfigMap is just generic key/value configuration.

For example:

```yaml
kind: ConfigMap
```

doesn't give you:

```text
custom API type
schema
kubectl get database
status
controller integration
```

A CRD gives Kubernetes a first-class API resource.

Instead of:

```text
ConfigMap:
database-config
```

you can have:

```text
Database:
postgres-prod
```

with a structured schema:

```yaml
spec:
  version:
  replicas:
  storage:
```

---

# 34. CRD vs Operator — a common misconception

Don't say:

> "An Operator is a CRD."

Incorrect.

Better:

> **An Operator commonly consists of a controller plus one or more Custom Resource Definitions and their controllers.**

The CRD defines the API.

The Operator implements behavior.

---

# 35. What happens when you delete the CR?

Suppose:

```text
Database CR
   |
   v
postgres-prod
```

You delete:

```bash
kubectl delete database postgres-prod
```

The Operator gets a deletion event and can clean up resources it owns.

Conceptually:

```text
Delete CR
   |
   v
Operator observes deletion
   |
   v
Cleanup
   |
   +-- StatefulSet
   +-- Service
   +-- PVC*
   +-- Secret*
```

The exact cleanup behavior depends on the Operator and Kubernetes ownership/deletion configuration.

The `*` is important because storage deletion behavior can be intentionally preserved depending on the design.

---

# 36. OwnerReferences

Operators commonly use Kubernetes ownership relationships.

For example:

```text
Database CR
     |
     | ownerReference
     v
StatefulSet
     |
     v
Pods
```

This tells Kubernetes which object owns which dependent resources.

Conceptually:

```text
Database
   |
   +-- owns StatefulSet
          |
          +-- owns Pods
```

This helps with garbage collection and controller reconciliation.

---

# 37. Reconciliation loop in detail

Suppose:

```yaml
spec:
  replicas: 3
```

Operator gets an event.

### Step 1 — Observe

```text
Read Database CR
```

### Step 2 — Calculate desired state

```text
Need 3 replicas
```

### Step 3 — Observe actual state

```text
Only 2 running
```

### Step 4 — Difference

```text
Desired = 3
Actual = 2
```

### Step 5 — Act

```text
Create/update resources
```

### Step 6 — Requeue/watch

```text
Check again
```

Eventually:

```text
Desired = 3
Actual = 3
```

Then:

```text
status.readyReplicas = 3
status.conditions[Ready] = True
```

---

# 38. What if someone manually changes the generated StatefulSet?

Suppose Operator wants:

```text
replicas = 3
```

but someone manually changes:

```bash
kubectl scale statefulset postgres --replicas=1
```

Operator detects:

```text
Desired:
3

Actual:
1
```

and reconciles it back toward:

```text
3
```

This is called **drift correction**.

This is exactly the same Kubernetes declarative model:

```text
Desired State
      |
      v
Controller
      |
      v
Actual State
      |
      v
Reconcile
```

---

# 39. Operator and GitOps

This becomes especially interesting with Argo CD.

You might have:

```text
Git
 |
 v
Database CR
 |
 v
Argo CD
 |
 v
Kubernetes API
 |
 v
Database Operator
 |
 v
StatefulSet/PVC/Service
```

Architecture:

```text
                 Git
                  |
                  v
                Argo CD
                  |
                  v
             Database CR
                  |
                  v
             API Server
                  |
                  v
          Database Operator
                  |
        +---------+---------+
        |         |         |
        v         v         v
    StatefulSet Service    PVC
```

Important:

> **Argo CD applies the desired CR; the Operator reconciles the application resources represented by that CR.**

This is a very common modern platform-engineering architecture.

---

# 40. CRD lifecycle

When you install an Operator, the installation often looks like:

```text
1. Install CRD
        ↓
2. Install Operator
        ↓
3. Create CR
        ↓
4. Operator watches CR
        ↓
5. Reconcile
        ↓
6. Create/manage resources
        ↓
7. Update CR status
```

Example:

```text
             CRD
              |
              v
       Kubernetes knows
        kind: Database
              |
              v
         Install Operator
              |
              v
       Create Database CR
              |
              v
          Reconcile
              |
      +-------+-------+
      |       |       |
      v       v       v
    PVC    StatefulSet Service
      |       |
      |       v
      |   PostgreSQL
      |
      v
   Storage
```

---

# 41. CRD versioning

For more advanced interviews, know that CRDs can support versions:

```yaml
versions:
  - name: v1
    served: true
    storage: true

  - name: v1beta1
    served: true
    storage: false
```

Important concepts:

```text
served
   ↓
Can clients use this version?

storage
   ↓
Is this the version used for storage internally?
```

In a mature platform, CRD version migrations and conversion webhooks may be used when schemas evolve between versions.

---

# 42. CRD vs built-in resource

From the API perspective, a CR becomes a real Kubernetes API object.

Built-in:

```text
apiVersion: apps/v1
kind: Deployment
```

Custom:

```text
apiVersion: platform.example.com/v1
kind: Database
```

Both can be interacted with using:

```bash
kubectl get
kubectl describe
kubectl apply
kubectl delete
```

For example:

```bash
kubectl get databases
```

and:

```bash
kubectl describe database postgres-prod
```

That's the power of extending Kubernetes.

---

# 43. The biggest interview trap

Question:

> "If I create a CRD and then create a CR, will Kubernetes automatically create the underlying resources?"

Answer:

**No.**

CRD:

```text
Makes Kubernetes understand the resource.
```

CR:

```text
Stores desired state as an API object.
```

Operator/controller:

```text
Actually watches the CR and performs reconciliation.
```

So:

```text
CRD alone:

Database CRD
     |
     v
Database CR
     |
     X
No automatic PostgreSQL
```

With Operator:

```text
Database CRD
     |
     v
Database CR
     |
     v
Operator
     |
     v
PostgreSQL infrastructure
```

---

# 44. CRD + CR + Operator — ultimate mental model

Memorize this:

```text
                     KUBERNETES API
                           |
                           |
                         CRD
                           |
              "Define a new resource"
                           |
                           v
                      Database
                     Resource Type
                           |
                           v
                      Database CR
                   "postgres-prod"
                           |
                           |
                    desired state
                           |
                           v
                      OPERATOR
                           |
                    Reconciliation
                           |
              +------------+------------+
              |            |            |
              v            v            v
         StatefulSet    Service        PVC
              |
              v
       PostgreSQL Pods
              |
              v
        Actual State

       Desired State
             |
             v
         Operator
             |
             v
       Actual State
             |
             +---- mismatch? ----+
                                  |
                                  v
                              Reconcile
```

---

# 45. The three definitions to memorize

### CRD

> **A CustomResourceDefinition extends the Kubernetes API by defining a new resource type, including its group, versions, scope, and schema.**

### CR

> **A Custom Resource is an actual instance of a custom resource type defined by a CRD and normally represents the user's desired state.**

### Operator

> **An Operator is a controller pattern that watches custom resources and reconciles the actual cluster state toward the desired state, embedding application-specific operational knowledge.**

---

# 46. Senior DevOps interview answer

If the interviewer asks:

> **"Explain CRD, CR and Operator with an example."**

I would answer:

> "Kubernetes has built-in resources such as Pods and Deployments, but we can extend its API using CustomResourceDefinitions. For example, I can define a `Database` CRD under `platform.example.com/v1`. Once that CRD exists, I can create a Custom Resource such as `postgres-prod` with a desired state like version 16, three replicas and 100Gi storage.
>
> The CRD itself doesn't provision PostgreSQL. An Operator provides the behavior. The Operator watches the Database CR and continuously reconciles the desired state with the actual cluster state. It might create a StatefulSet, Services, PVCs and Secrets, monitor the database, update the CR's status, and recover from drift or failures.
>
> So I remember it as **CRD defines the API, CR represents the desired state, and Operator implements the reconciliation logic.**"

That distinction—**API definition vs desired-state object vs reconciliation logic**—is the core of the whole topic.
