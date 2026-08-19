# Pod-Terminations_Restart-Policies_ Image-Pull-Policies_Pod-Lifecycle

These four are closely related, but they answer **different questions**:

```text
Pod Lifecycle
    ↓
What states can a Pod go through?

Pod Termination
    ↓
What happens when a Pod is being stopped?

RestartPolicy
    ↓
What should kubelet do when a container exits?

ImagePullPolicy
    ↓
When should kubelet pull the container image?
```

Let's build the complete picture.

---

# 1. Pod Lifecycle

A Kubernetes Pod has a lifecycle.

At a high level:

```text
                 Pod Created
                      |
                      v
                  Pending
                      |
                      v
                 Running
                  /     \
                 /       \
                v         v
            Succeeded   Failed
```

There is also:

```text
Unknown
```

as a Pod phase.

The five Pod phases are:

```text
Pending
Running
Succeeded
Failed
Unknown
```

---

# 2. `Pending`

When you create:

```bash
kubectl apply -f pod.yaml
```

the Pod initially may be:

```text
STATUS: Pending
```

This means Kubernetes has accepted the Pod, but it isn't yet running successfully.

For example:

```text
Pod created
     |
     v
Pending
     |
     +-- Waiting for scheduler
     +-- Waiting for image pull
     +-- Waiting for volume
     +-- Waiting for node resources
```

Example:

```bash
kubectl get pods
```

```text
NAME    READY   STATUS    RESTARTS
nginx   0/1     Pending   0
```

---

# 3. Why can a Pod remain Pending?

Common reasons:

### Insufficient resources

```text
Pod requests:
4 CPU

Node available:
2 CPU
```

Scheduler can't place it.

### Node selector doesn't match

```yaml
nodeSelector:
  environment: gpu
```

but no node has:

```text
environment=gpu
```

### Required affinity isn't satisfied

```text
requiredDuringScheduling...
```

and no suitable node exists.

### PVC isn't available

```text
Pod
 |
 v
PVC
 |
 v
Pending
```

### Taints aren't tolerated

```text
Node:
dedicated=gpu:NoSchedule

Pod:
no toleration
```

---

# 4. `Running`

A Pod is `Running` when it has been assigned to a node and at least one container is running or starting.

Example:

```text
Node
 |
 +-- Pod
      |
      +-- nginx container
             |
             +-- Running
```

Check:

```bash
kubectl get pod nginx
```

```text
NAME    READY   STATUS    RESTARTS
nginx   1/1     Running   0
```

---

# 5. `Succeeded`

Suppose you have:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: job
spec:
  restartPolicy: Never

  containers:
    - name: task
      image: busybox
      command:
        - sh
        - -c
        - echo "Hello"; exit 0
```

The container exits successfully:

```text
exit code = 0
```

Pod becomes:

```text
Succeeded
```

```bash
kubectl get pod job
```

```text
NAME   READY   STATUS      RESTARTS
job    0/1     Succeeded   0
```

This is common for one-time tasks.

---

# 6. `Failed`

If the container terminates unsuccessfully and isn't going to be restarted, the Pod can reach:

```text
Failed
```

For example:

```text
exit code = 1
```

with:

```yaml
restartPolicy: Never
```

Then:

```text
Container
    |
    | exit 1
    v
Pod
    |
    v
Failed
```

---

# 7. `Unknown`

`Unknown` is less common.

It means Kubernetes cannot determine the state of the Pod, often because it can't communicate with the node/kubelet.

For example:

```text
API Server
    |
    X
  Kubelet
```

The control plane may not know the actual Pod state.

---

# 8. Important distinction: Pod phase vs container state

This is a common interview question.

Pod:

```text
Running
```

doesn't necessarily mean every container is healthy.

A Pod can be:

```text
STATUS: Running
READY: 0/1
```

because the container may be:

```text
Waiting
```

or:

```text
Running but failing readiness probe
```

So distinguish:

```text
Pod Phase
    ↓
Pending / Running / Succeeded / Failed / Unknown

Container State
    ↓
Waiting / Running / Terminated
```

---

# 9. Container states

Each container can have:

```text
Waiting
Running
Terminated
```

### Waiting

Container hasn't started yet.

Examples:

```text
ImagePullBackOff
ErrImagePull
ContainerCreating
CrashLoopBackOff
```

### Running

Container process is running.

### Terminated

Container process has exited.

It includes:

```text
exitCode
reason
startedAt
finishedAt
```

Example:

```text
exitCode: 1
reason: Error
```

---

# 10. Pod termination

Now let's say you have:

```text
Running Pod
```

and execute:

```bash
kubectl delete pod nginx
```

Kubernetes doesn't normally immediately kill the process.

It performs **graceful termination**.

Typical flow:

```text
Running
   |
   | delete request
   v
Terminating
   |
   v
SIGTERM
   |
   | grace period
   v
Application shuts down
   |
   v
SIGKILL if still running
   |
   v
Pod removed
```

---

# 11. SIGTERM vs SIGKILL

This is very important.

When a container needs to stop, Kubernetes/container runtime normally gives the main process:

```text
SIGTERM
```

Meaning:

> "Please shut down gracefully."

Your application can catch this and:

```text
Close connections
Finish current request
Flush buffers
Close files
Shutdown gracefully
```

If it doesn't exit within the grace period:

```text
SIGKILL
```

is used.

Meaning:

> "Stop immediately."

---

# 12. Default termination grace period

Pods have:

```yaml
terminationGracePeriodSeconds: 30
```

by default if you don't specify otherwise.

Example:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: graceful-app
spec:
  terminationGracePeriodSeconds: 60

  containers:
    - name: app
      image: nginx
```

Now Kubernetes allows up to roughly:

```text
60 seconds
```

for graceful shutdown before forceful termination.

---

# 13. Complete termination flow

Suppose:

```bash
kubectl delete pod myapp
```

Flow:

```text
           kubectl delete
                 |
                 v
            API Server
                 |
                 v
          Pod marked for
            deletion
                 |
                 v
           kubelet notices
                 |
                 v
        Graceful termination
                 |
                 v
              SIGTERM
                 |
          +------+------+
          |             |
      app exits      app hangs
          |             |
          v             v
       Success       grace period
                        |
                        v
                     SIGKILL
                        |
                        v
                   Container gone
```

---

# 14. `preStop` hook

You can run a command before the container is terminated.

Example:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: graceful-app
spec:
  terminationGracePeriodSeconds: 30

  containers:
    - name: app
      image: nginx

      lifecycle:
        preStop:
          exec:
            command:
              - /bin/sh
              - -c
              - echo "Graceful shutdown started"
```

Flow:

```text
Delete Pod
    |
    v
preStop hook
    |
    v
SIGTERM
    |
    v
Application shutdown
```

`preStop` is commonly used for application-specific shutdown preparation.

---

# 15. Pod termination and Services

This is very important for production.

Suppose:

```text
Service
  |
  +-- Pod A
  +-- Pod B
  +-- Pod C
```

Pod B is being terminated.

Kubernetes removes it from the set of ready Service endpoints as appropriate, so new traffic should stop being sent to it once endpoint updates propagate.

Conceptually:

```text
Before:

Service
  |
  +-- A
  +-- B
  +-- C


Pod B terminating:

Service
  |
  +-- A
  +-- C

Pod B
  |
  +-- graceful shutdown
```

This is why graceful termination matters for avoiding dropped requests.

---

# 16. RestartPolicy

Now let's move to:

```text
restartPolicy
```

This answers:

> **What should kubelet do when a container in the Pod terminates?**

There are three values:

```text
Always
OnFailure
Never
```

---

# 17. `Always`

```yaml
restartPolicy: Always
```

If a container exits:

```text
Container exits
     |
     v
Restart container
```

Example:

```text
nginx container
     |
     | exits
     v
kubelet restarts it
```

`Always` is the default for normal Pods.

It is also the behavior used by workloads such as Deployments.

---

# 18. `OnFailure`

```yaml
restartPolicy: OnFailure
```

Restart only if the container exits with a failure.

Example:

```text
exit 0
   |
   v
Don't restart

exit 1
   |
   v
Restart
```

Useful for tasks where success should end the container.

---

# 19. `Never`

```yaml
restartPolicy: Never
```

Container exits:

```text
Container
   |
   v
Terminated
   |
   v
No restart
```

Useful for one-time workloads.

---

# 20. RestartPolicy example

### Example 1

```yaml
restartPolicy: Never
```

Command:

```bash
exit 0
```

Result:

```text
Succeeded
```

---

### Example 2

```yaml
restartPolicy: Never
```

Command:

```bash
exit 1
```

Result:

```text
Failed
```

---

### Example 3

```yaml
restartPolicy: OnFailure
```

Command:

```bash
exit 1
```

Result:

```text
Container restarted
```

---

### Example 4

```yaml
restartPolicy: OnFailure
```

Command:

```bash
exit 0
```

Result:

```text
Succeeded
```

---

# 21. Important: RestartPolicy is not the same as Deployment self-healing

This distinction is very important.

Suppose:

```text
Pod
 |
 +-- container crashes
```

kubelet can restart the container according to:

```text
restartPolicy
```

But if:

```text
Entire Pod is deleted
```

then:

```text
restartPolicy
```

doesn't recreate the Pod.

A controller such as a:

```text
Deployment
ReplicaSet
StatefulSet
Job
```

handles Pod-level replacement according to its own semantics.

---

# 22. Example: Deployment

You create:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 3

  selector:
    matchLabels:
      app: nginx

  template:
    metadata:
      labels:
        app: nginx

    spec:
      containers:
        - name: nginx
          image: nginx
```

Architecture:

```text
Deployment
    |
    v
ReplicaSet
    |
 +--+--+--+
 |  |  |
 v  v  v
Pod Pod Pod
```

If one Pod is deleted:

```text
Pod A ❌
```

ReplicaSet notices:

```text
Desired = 3
Actual = 2
```

and creates another:

```text
Pod D
```

That's different from container restart.

---

# 23. ImagePullPolicy

Now the third concept.

`imagePullPolicy` tells kubelet:

> **When should the image be pulled from the container registry?**

Values:

```text
Always
IfNotPresent
Never
```

---

# 24. `Always`

```yaml
imagePullPolicy: Always
```

Kubelet attempts to resolve/pull the image whenever a container is launched.

Important nuance:

> With a registry-backed image, "Always" does not necessarily mean the entire image is downloaded every time. Container runtimes can use image layers that are already present.

For example:

```yaml
image: nginx:latest
imagePullPolicy: Always
```

When container starts:

```text
Kubelet
  |
  v
Check registry / image reference
  |
  v
Pull/update image if needed
```

---

# 25. `IfNotPresent`

```yaml
imagePullPolicy: IfNotPresent
```

Meaning:

> Use the locally available image if present; otherwise pull it.

Example:

```text
Node already has nginx:1.27
        |
        v
Use local image
```

If not:

```text
Image missing
    |
    v
Pull from registry
```

This can reduce unnecessary registry traffic.

---

# 26. `Never`

```yaml
imagePullPolicy: Never
```

Kubelet will not pull the image.

It must already exist locally.

```text
Image exists?
    |
 +--+--+
 |     |
YES    NO
 |      |
 v      v
Run   Error
```

Useful in certain offline/testing scenarios, but not typical for normal production clusters.

---

# 27. `imagePullPolicy` defaults

This is an interview favorite.

If you specify:

```yaml
image: nginx:latest
```

the default is generally:

```text
Always
```

If you specify a normal non-`latest` tag such as:

```yaml
image: nginx:1.27
```

the default is generally:

```text
IfNotPresent
```

If you specify an image by digest:

```text
nginx@sha256:...
```

the default is generally:

```text
IfNotPresent
```

So don't blindly say:

> "Default is IfNotPresent."

It depends on the image reference and how the field is specified.

---

# 28. Why `latest` is discouraged

Consider:

```yaml
image: myapp:latest
```

Today:

```text
latest → version 1
```

Tomorrow:

```text
latest → version 2
```

The same Deployment manifest can result in different image content.

Better:

```yaml
image: myapp:1.5.2
```

or, for stronger immutability:

```text
myapp@sha256:<digest>
```

This makes deployments more predictable.

---

# 29. ImagePullBackOff

Suppose you specify:

```yaml
image: myregistry.com/myapp:999
```

but the tag doesn't exist.

Kubelet tries:

```text
Pull image
   |
   v
Registry
   |
   X
Image not found
```

You may see:

```text
ErrImagePull
```

followed by:

```text
ImagePullBackOff
```

The `BackOff` means Kubernetes is backing off before retrying.

This is a **container waiting state**, not a Pod phase.

---

# 30. RestartPolicy vs ImagePullPolicy

Don't confuse them.

### RestartPolicy

Answers:

> What should happen when the container exits?

```text
Container exits
      |
      v
restartPolicy
      |
 +----+-----+
 |    |     |
Always OnFailure Never
```

### ImagePullPolicy

Answers:

> When starting the container, should kubelet pull/check the image?

```text
Container needs to start
          |
          v
imagePullPolicy
          |
     +----+--------+
     |             |
   Pull?         Local?
```

---

# 31. Complete example combining all three

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: demo
spec:
  restartPolicy: OnFailure

  terminationGracePeriodSeconds: 30

  containers:
    - name: app
      image: busybox:1.36
      imagePullPolicy: IfNotPresent

      command:
        - sh
        - -c
        - |
          echo "Application started"
          sleep 10
          exit 1
```

What happens?

### Container starts

```text
imagePullPolicy = IfNotPresent
```

If image isn't on the node:

```text
Pull image
```

Then application starts:

```text
Running
```

After 10 seconds:

```text
exit 1
```

Because:

```text
restartPolicy = OnFailure
```

kubelet restarts the container.

So:

```text
Running
   |
   v
exit 1
   |
   v
restart
   |
   v
Running
   |
   v
exit 1
   |
   v
restart
```

Eventually Kubernetes applies exponential backoff to repeated restarts, and you may see:

```text
CrashLoopBackOff
```

---

# 32. CrashLoopBackOff

This is another interview favorite.

Suppose:

```text
Application
   |
   v
starts
   |
   v
crashes
   |
   v
restart
   |
   v
crashes
   |
   v
restart
```

This happens repeatedly.

Kubernetes doesn't continuously restart it immediately.

It applies increasing delays:

```text
Restart
   ↓
wait
   ↓
Restart
   ↓
longer wait
   ↓
Restart
```

You may see:

```text
CrashLoopBackOff
```

Important:

> `CrashLoopBackOff` is **not a restart policy**.

It's a status/reason indicating Kubernetes is backing off repeated restart attempts.

---

# 33. Pod lifecycle + termination + restart

Now combine everything.

```text
                  Pod Created
                       |
                       v
                    Pending
                       |
                       v
                    Running
                       |
             +---------+---------+
             |                   |
       Container exits      Pod deleted
             |                   |
             v                   v
       restartPolicy          Termination
             |                   |
       +-----+-----+        SIGTERM
       |           |           |
     restart     no restart    v
       |           |       preStop/grace
       |           |           |
       v           v           v
    Running     Succeeded/   SIGKILL
                Failed          |
                                v
                          Pod removed
```

---

# 34. Graceful termination with a Deployment

This is a very common production scenario.

Suppose:

```text
Deployment
replicas: 3

Pod A
Pod B
Pod C
```

You perform:

```bash
kubectl rollout restart deployment myapp
```

Kubernetes gradually terminates old Pods and creates new ones according to the Deployment's rollout strategy.

For each terminating Pod:

```text
Pod
 |
 v
Terminating
 |
 v
Endpoint removed / traffic drains as updates propagate
 |
 v
preStop
 |
 v
SIGTERM
 |
 v
Application exits
 |
 v
Pod deleted
```

This is why applications should handle `SIGTERM` correctly.

---

# 35. `terminationGracePeriodSeconds` vs `preStop`

They are related but different.

### `terminationGracePeriodSeconds`

Controls the grace period available for termination.

```yaml
terminationGracePeriodSeconds: 60
```

### `preStop`

Runs a lifecycle hook before termination.

```yaml
lifecycle:
  preStop:
    exec:
      command:
        - /bin/sh
        - -c
        - "..."
```

Think:

```text
terminationGracePeriod
        |
        | overall shutdown window
        v
     preStop
        |
        v
     SIGTERM
        |
        v
    application
```

---

# 36. Important Kubernetes interview traps

### Trap 1

> Is `CrashLoopBackOff` a restart policy?

**No.**

It's a status/reason associated with repeated container failures and restart backoff.

---

### Trap 2

> Does `restartPolicy` recreate a deleted Pod?

**No.**

It controls container restarts inside a Pod.

Controllers recreate Pods.

---

### Trap 3

> Does `imagePullPolicy: Always` download all layers every time?

**Not necessarily.**

The runtime can reuse cached layers.

---

### Trap 4

> Does `Running` mean all containers are healthy?

**No.**

Pod phase and container readiness are different concepts.

---

### Trap 5

> Does deleting a Pod immediately send SIGKILL?

**Normally no.**

Kubernetes normally attempts graceful termination first.

---

# 37. One final diagram to memorize

```text
                         POD LIFECYCLE
                              |
                              v
                           Pending
                              |
                              v
                           Running
                              |
              +---------------+---------------+
              |                               |
       Container exits                  Pod deletion
              |                               |
              v                               v
       RestartPolicy                    Termination
              |                               |
       +------+------+                  preStop
       |             |                     |
     restart      don't                   SIGTERM
       |          restart                   |
       v             |                 Grace period
    Running          v                     |
               Succeeded/Failed            v
                                      SIGKILL if needed
                                            |
                                            v
                                       Pod removed


                    CONTAINER START
                         |
                         v
                  ImagePullPolicy
                         |
             +-----------+-----------+
             |           |           |
           Always    IfNotPresent   Never
             |           |           |
             v           v           v
           Pull     Local/Pull    Local only
```

### Interview-ready summary

| Concept                           | Question it answers                              |
| --------------------------------- | ------------------------------------------------ |
| **Pod Phase**                     | What overall lifecycle state is the Pod in?      |
| **Container State**               | What state is an individual container in?        |
| **Termination**                   | How does Kubernetes gracefully stop a Pod?       |
| **RestartPolicy**                 | Should kubelet restart an exited container?      |
| **ImagePullPolicy**               | When should the image be pulled/resolved?        |
| **CrashLoopBackOff**              | Why is Kubernetes backing off repeated restarts? |
| **preStop**                       | What should run before container termination?    |
| **terminationGracePeriodSeconds** | How long should graceful termination be allowed? |

The **Senior DevOps-level connection** to remember is:

```text
Deployment
    |
    v
Pod lifecycle
    |
    +---- container crashes
    |          |
    |          v
    |     restartPolicy
    |
    +---- Pod replaced/rolled out
               |
               v
        graceful termination
               |
          preStop + SIGTERM
               |
        terminationGracePeriod
               |
               v
             SIGKILL

Container startup
    |
    v
imagePullPolicy
    |
    v
image available
    |
    v
container starts
```

This is the flow I'd use in an interview rather than memorizing the individual definitions.
