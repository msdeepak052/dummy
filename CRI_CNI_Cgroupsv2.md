# CRI, CNI & Cgroups

These three are often discussed together because they sit at **different layers of the Kubernetes node architecture**:

```text
                    Kubernetes
                        |
                  kubelet
                        |
        +---------------+---------------+
        |               |               |
       CRI             CNI          cgroups v2
   Container       Networking       Resource
   runtime         for Pods         control
        |               |               |
        v               v               v
   containerd       Calico/Cilium    Linux kernel
   / CRI-O          / AWS VPC CNI
        |
        v
    Containers
```

The easiest way to remember:

> **CRI = How Kubernetes talks to the container runtime**
> **CNI = How Pods get networking**
> **cgroups v2 = How Linux controls and measures resources**

---

# 1. CRI — Container Runtime Interface

CRI is the interface between:

```text
kubelet
   |
   v
container runtime
```

Kubernetes does **not** directly manage containers itself.

Instead:

```text
Kubelet
   |
   | CRI
   v
containerd / CRI-O
   |
   v
runc
   |
   v
Linux containers
```

---

# 2. Why was CRI created?

Originally Kubernetes had tighter integration with Docker.

But Kubernetes wanted to support different container runtimes without changing kubelet.

So Kubernetes defined:

```text
CRI
```

as a standard interface.

Now kubelet can work with:

```text
containerd
CRI-O
```

and other CRI-compatible runtimes.

---

# 3. Example: Pod creation

Suppose you run:

```bash
kubectl run nginx --image=nginx
```

The actual flow is approximately:

```text
kubectl
   |
   v
API Server
   |
   v
Scheduler
   |
   v
Kubelet
   |
   | CRI
   v
containerd
   |
   v
runc
   |
   v
Container
```

The kubelet doesn't say:

```text
"containerd, execute this exact internal command."
```

Instead it uses the **CRI API**.

---

# 4. CRI has two major pieces

Conceptually, CRI exposes operations for:

```text
Pod lifecycle
Container lifecycle
```

Important concepts:

### PodSandbox

Kubernetes creates a **PodSandbox** first.

Then containers are created inside that Pod's network namespace.

Conceptually:

```text
Pod
 |
 +--- Pod Sandbox
 |       |
 |       +--- Network namespace
 |
 +--- Container 1
 |
 +--- Container 2
```

This is important because **all containers in a Pod share the same network namespace**.

---

# 5. CRI example

Suppose:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web
spec:
  containers:
    - name: nginx
      image: nginx
```

Kubelet roughly does:

```text
1. Create PodSandbox
2. Set up Pod networking
3. Pull nginx image
4. Create nginx container
5. Start nginx container
```

The networking part is where **CNI** comes in.

So:

```text
CRI
 |
 +-- Create PodSandbox
 |
 +-- Create container
 |
 +-- Start container
```

while:

```text
CNI
 |
 +-- Give Pod IP
 +-- Configure network interface
 +-- Configure routes
```

---

# 6. CRI vs containerd vs runc

This is another common interview confusion.

They are not the same thing.

```text
              kubelet
                 |
                CRI
                 |
             containerd
                 |
          containerd-shim
                 |
                runc
                 |
          Linux kernel
                 |
             container
```

### CRI

Interface/API.

### containerd

Container runtime / runtime manager.

Handles things such as:

```text
Image pulling
Container lifecycle
Snapshot/storage
Runtime management
```

### runc

Low-level OCI runtime.

It actually creates the Linux container using kernel features such as:

```text
Namespaces
cgroups
Capabilities
seccomp
```

---

# 7. CNI — Container Network Interface

Now let's move to networking.

CNI is responsible for configuring networking for containers/Pods.

Think:

> **CRI creates the Pod sandbox; CNI gives that sandbox networking.**

Architecture:

```text
                Kubelet
                   |
                   v
                 CRI
                   |
                   v
             Pod Sandbox
                   |
                   |
                  CNI
                   |
          +--------+--------+
          |        |        |
       IPAM      veth      routes
          |
          v
       Pod IP
```

---

# 8. What does CNI actually do?

When a Pod is created, CNI can perform operations such as:

```text
Create network interface
Assign IP address
Configure routes
Configure network namespace
Configure connectivity
Apply networking-related configuration
```

For example:

```text
Pod
 |
 | eth0
 v
10.244.1.20
```

---

# 9. Simple CNI example

Suppose Kubernetes creates:

```text
Pod:
nginx
```

The CNI plugin might configure:

```text
Pod network namespace
        |
        +--- eth0
              |
              IP = 10.244.1.20
```

Then connect the Pod to the node network.

A common Linux implementation uses a **veth pair**:

```text
Pod Network Namespace
        |
      eth0
        |
        | veth pair
        |
      vethXXX
        |
Node Network Namespace
```

Conceptually:

```text
+---------------- Pod ----------------+
|                                     |
| eth0: 10.244.1.20                   |
|                                     |
+-----------------+-------------------+
                  |
                veth
                  |
+-----------------+-------------------+
|                Node                 |
|                                     |
| vethXXX                            |
|                                     |
+-------------------------------------+
```

---

# 10. Different CNI implementations

CNI is an **interface/specification**, not one networking product.

Examples include:

```text
Calico
Cilium
Flannel
AWS VPC CNI
Canal
```

They implement networking differently.

For EKS, a common choice is:

```text
AWS VPC CNI
```

It integrates Pods with AWS VPC networking.

Conceptually:

```text
EKS
 |
 +-- kubelet
      |
      +-- CRI
      |
      +-- CNI
            |
            +-- AWS VPC CNI
                    |
                    v
                 AWS VPC
```

---

# 11. CNI and Kubernetes Service

One important distinction:

CNI primarily handles **Pod networking**.

Kubernetes Services involve additional networking mechanisms.

For example:

```text
Pod A
10.0.1.10
       \
        \
         Service
       10.0.100.20
         /
        /
Pod B
10.0.2.20
```

Service traffic may involve:

```text
kube-proxy
iptables
IPVS
eBPF
```

depending on the cluster/networking implementation.

CNI and Service networking are related but **not identical concepts**.

---

# 12. Cgroups

Now the third component:

```text
cgroups
```

Cgroups means:

> **Control Groups**

It's a Linux kernel mechanism used to control and account for resources used by processes.

Kubernetes uses cgroups to enforce things like:

```text
CPU
Memory
PIDs
```

among others.

---

# 13. Why does Kubernetes need cgroups?

Suppose you have:

```text
Node
8 CPU
32 GB RAM
```

And one Pod starts consuming:

```text
31 GB RAM
```

You don't want that application to consume everything and potentially destabilize the node.

Kubernetes can configure resource limits through cgroups.

For example:

```yaml
resources:
  requests:
    cpu: "500m"
    memory: "512Mi"

  limits:
    cpu: "1"
    memory: "1Gi"
```

Conceptually:

```text
Pod
 |
 +-- Container
       |
       +-- cgroup
             |
             +-- CPU limit = 1 CPU
             +-- Memory limit = 1Gi
```

---

# 14. cgroups v1 vs cgroups v2

This is where **cgroups v2** comes in.

Linux historically had:

```text
cgroups v1
```

which had separate hierarchies/controllers.

Newer Linux systems increasingly use:

```text
cgroups v2
```

which provides a unified hierarchy and updated resource-control model.

Conceptually:

### cgroups v1

```text
CPU hierarchy
    |
    +-- Pod

Memory hierarchy
    |
    +-- Pod
```

### cgroups v2

```text
Unified cgroup hierarchy
        |
        +-- Kubernetes
              |
              +-- Pod
                    |
                    +-- Container
```

---

# 15. cgroups v2 architecture

Think:

```text
Linux Kernel
     |
     v
cgroup v2
     |
     +-----------------------+
     |                       |
   Pod A                   Pod B
     |                       |
 Container                Container
```

The kernel uses cgroups to enforce resource policies.

---

# 16. Kubernetes resource example

Suppose:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: memory-demo
spec:
  containers:
    - name: app
      image: nginx
      resources:
        requests:
          memory: "256Mi"
          cpu: "250m"

        limits:
          memory: "512Mi"
          cpu: "500m"
```

Think:

```text
              Pod
               |
         +-----+-----+
         |           |
      Request      Limit
         |           |
         v           v
      256Mi        512Mi
      250m         500m
```

cgroups are involved in enforcing the container's resource boundaries.

---

# 17. Requests vs limits

This distinction is important.

### Request

```yaml
requests:
  cpu: 250m
  memory: 256Mi
```

Primarily tells the scheduler:

> "I need approximately this amount of resource for scheduling."

So:

```text
Scheduler
   |
   v
Which node has enough requested resources?
```

### Limit

```yaml
limits:
  cpu: 500m
  memory: 512Mi
```

This becomes a runtime resource boundary.

```text
Container
   |
   +--- CPU limit
   +--- Memory limit
```

cgroups participate in enforcing these limits.

---

# 18. What happens if memory exceeds the limit?

Suppose:

```yaml
limits:
  memory: 512Mi
```

Application tries to consume:

```text
800Mi
```

The container can hit an OOM condition and be terminated/restarted depending on the Pod's behavior and restart policy.

Conceptually:

```text
Container
    |
Memory = 512Mi limit
    |
Application tries 800Mi
    |
    v
Memory pressure / OOM
    |
    v
Container terminated
```

---

# 19. CPU is different

Suppose:

```yaml
limits:
  cpu: "500m"
```

The container tries to use:

```text
2 CPU
```

It generally isn't killed just because it exceeds the CPU limit.

Instead, CPU usage is **throttled** according to the configured limit.

So:

```text
CPU:
Too much → throttling

Memory:
Too much → can result in OOM kill
```

That's a useful interview distinction.

---

# 20. Now connect CRI + CNI + cgroups

This is the most important part.

Suppose you run:

```bash
kubectl create deployment nginx --image=nginx
```

The high-level flow:

```text
                    kubectl
                       |
                       v
                  API Server
                       |
                       v
                   Scheduler
                       |
                       v
                    Kubelet
                       |
             +---------+---------+
             |                   |
            CRI                 CNI
             |                   |
             v                   v
        containerd          Network setup
             |                   |
             v                   |
           runc                  |
             |                   |
             +---------+----------+
                       |
                       v
                  Linux Kernel
                       |
                  +----+----+
                  |         |
               cgroups   namespaces
                  |
                  v
               Container
```

---

# 21. Step-by-step Pod creation

Let's make the process explicit.

## Step 1 — Scheduler selects node

```text
Pod
 ↓
Scheduler
 ↓
worker-1
```

---

## Step 2 — Kubelet sees the Pod

Kubelet on `worker-1` sees:

```text
Pod nginx needs to run here.
```

---

## Step 3 — Kubelet talks to CRI

```text
Kubelet
   |
   | CRI
   v
containerd
```

---

## Step 4 — Runtime creates Pod sandbox

```text
containerd
    |
    v
Pod Sandbox
```

---

## Step 5 — CNI configures networking

```text
CNI
 |
 +-- network namespace
 +-- interface
 +-- IP
 +-- routes
```

For example:

```text
Pod
 |
 +-- eth0
       |
       +-- 10.244.1.20
```

---

## Step 6 — Runtime creates container

```text
containerd
    |
    v
runc
    |
    v
Linux container
```

---

## Step 7 — cgroups configured

Resource constraints are applied:

```text
cgroup
 |
 +-- CPU
 +-- Memory
 +-- PIDs
```

---

# 22. One complete architecture

```text
                         CONTROL PLANE
                              |
                         API Server
                              |
                         Scheduler
                              |
                              |
                         WORKER NODE
                              |
                           Kubelet
                              |
                +-------------+-------------+
                |                           |
               CRI                         CNI
                |                           |
                v                           v
           containerd                 CNI plugin
                |                           |
                v                           v
              runc                    Pod network
                |                           |
                +-------------+-------------+
                              |
                              v
                         Linux Kernel
                              |
               +--------------+--------------+
               |              |              |
           namespaces      cgroups        capabilities
                              |
                              v
                       Resource control
                              |
                              v
                           Container
```

---

# 23. CRI vs CNI vs cgroups v2

| Component      | Main job                    | Example                     |
| -------------- | --------------------------- | --------------------------- |
| **CRI**        | Kubelet ↔ container runtime | containerd, CRI-O           |
| **CNI**        | Pod networking              | AWS VPC CNI, Cilium, Calico |
| **cgroups v2** | Resource control/accounting | CPU, memory, PIDs           |
| **runc**       | Creates OCI containers      | runc                        |
| **kubelet**    | Manages Pods on node        | Node agent                  |

---

# 24. Very important interview distinction

Don't say:

> "containerd is CRI."

Better:

> **containerd is a container runtime that can expose a CRI implementation to kubelet.**

Similarly don't say:

> "CNI gives containers networking."

Better:

> **CNI is the plugin interface/specification used to configure networking for Pod sandboxes; implementations such as AWS VPC CNI, Cilium, and Calico provide the actual networking functionality.**

And:

> **cgroups v2 is a Linux kernel resource-control mechanism used by the container runtime and Kubernetes to manage resource consumption and accounting.**

---

# 25. How to check these on a Kubernetes node

### Check container runtime

```bash
kubectl get nodes -o wide
```

Then:

```bash
kubectl describe node <node-name>
```

Look for:

```text
Container Runtime Version:
containerd://...
```

You can also check:

```bash
kubectl get node <node> -o jsonpath='{.status.nodeInfo.containerRuntimeVersion}'
```

---

# 26. Check cgroups version

On a Linux node:

```bash
stat -fc %T /sys/fs/cgroup
```

If you get:

```text
cgroup2fs
```

you're using cgroups v2.

Another useful check:

```bash
mount | grep cgroup
```

You may see:

```text
cgroup2 on /sys/fs/cgroup
```

---

# 27. Check CNI

On many Kubernetes nodes:

```bash
ls /opt/cni/bin/
```

You might see plugins such as:

```text
bridge
host-local
loopback
```

and potentially your cluster-specific networking components.

CNI configuration is commonly found under:

```bash
ls /etc/cni/net.d/
```

For EKS, the AWS VPC CNI components are typically visible through the `aws-node` DaemonSet:

```bash
kubectl get daemonset -n kube-system
```

---

# 28. The easiest interview analogy

Think about a hotel.

```text
Kubernetes Pod = Hotel guest
```

### CRI

```text
CRI = Hotel management interface

Kubelet:
"Create a room for this guest."

Runtime:
"Room created."
```

### CNI

```text
CNI = Give the guest connectivity

"Here is your room's network connection,
IP and route to the rest of the hotel."
```

### cgroups v2

```text
cgroups = Resource control

"Your room can consume:
500 MB memory
0.5 CPU
etc."
```

So:

```text
CRI
 ↓
Create/manage the container

CNI
 ↓
Connect the Pod to the network

cgroups v2
 ↓
Control the Pod/container's resources
```

---

# 29. The one diagram I'd memorize

```text
                         KUBELET
                            |
                +-----------+-----------+
                |                       |
               CRI                     CNI
                |                       |
                v                       v
           containerd              Pod Network
                |                  + IP/interface
                v                  + routes
               runc
                |
                v
          Linux Container
                |
                v
          Linux Kernel
                |
                +---- namespaces
                |
                +---- cgroups v2
                |        |
                |        +-- CPU
                |        +-- Memory
                |        +-- PIDs
                |
                +---- capabilities
                |
                +---- seccomp
```

### Final interview answer

> **CRI is the interface through which kubelet communicates with the container runtime, such as containerd or CRI-O. CNI is the networking plugin interface used to configure Pod networking, including interfaces, IP addresses and routes. cgroups v2 is a Linux kernel mechanism providing a unified hierarchy for resource control and accounting, such as CPU and memory. During Pod creation, kubelet uses CRI to ask the runtime to create the Pod sandbox and containers, CNI configures the Pod's network namespace, and the runtime configures Linux isolation and cgroups for resource control.**
