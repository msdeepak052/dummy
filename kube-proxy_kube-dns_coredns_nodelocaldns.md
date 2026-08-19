# kube-proxy_kube-dns_coredns_nodelocaldns

These four are easy to mix up because **kube-proxy is mainly about Service traffic**, while **CoreDNS/kube-dns/NodeLocal DNS are about DNS resolution**.

The first thing to memorize:

```text
kube-proxy
    ↓
Service networking / traffic forwarding

CoreDNS
    ↓
DNS resolution inside Kubernetes

kube-dns
    ↓
Old/name for the Kubernetes DNS service

NodeLocal DNSCache
    ↓
Local DNS caching on every node
```

---

# 1. Big picture

Suppose you have:

```text
frontend Pod
     |
     | calls
     v
backend.default.svc.cluster.local
     |
     | DNS lookup
     v
CoreDNS / NodeLocal DNS
     |
     | returns Service ClusterIP
     v
10.96.20.50
     |
     | Service traffic
     v
kube-proxy / eBPF dataplane
     |
     +----------+----------+
     |                     |
     v                     v
backend Pod 1          backend Pod 2
10.244.1.10            10.244.2.15
```

So there are really **two different problems**:

### Problem 1 — "What IP belongs to this Service name?"

```text
backend.default.svc.cluster.local
              |
              v
             DNS
              |
              v
         10.96.20.50
```

That's:

```text
CoreDNS / NodeLocal DNS
```

### Problem 2 — "How does traffic to 10.96.20.50 reach a backend Pod?"

```text
10.96.20.50
      |
      v
Service dataplane
      |
      v
backend Pod
```

Traditionally:

```text
kube-proxy
```

---

# 2. kube-proxy

## What is kube-proxy?

`kube-proxy` is a Kubernetes node component responsible for implementing **Service networking rules** on nodes.

It watches Kubernetes Services and EndpointSlices and configures the node's networking dataplane so that traffic sent to a Service can reach one of the backend Pods.

Important:

> **kube-proxy does not normally act as a user-space proxy for every packet.**

This is a common interview trap.

Historically, the name "proxy" makes people think:

```text
Pod → kube-proxy process → Pod
```

But with common modes such as iptables or IPVS, kube-proxy primarily **programs kernel networking rules**.

---

# 3. Example Service

Suppose:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend
spec:
  selector:
    app: backend
  ports:
    - port: 80
      targetPort: 8080
```

And we have:

```text
backend-1 → 10.244.1.10:8080
backend-2 → 10.244.2.20:8080
backend-3 → 10.244.3.30:8080
```

Kubernetes gives the Service a ClusterIP:

```text
backend Service
ClusterIP = 10.96.50.100
```

Architecture:

```text
                 Service
             10.96.50.100:80
                     |
          +----------+----------+
          |          |          |
          v          v          v
       Pod 1       Pod 2       Pod 3
     10.244.1.10 10.244.2.20 10.244.3.30
       :8080        :8080       :8080
```

---

# 4. What kube-proxy does

kube-proxy watches:

```text
Service
EndpointSlice
```

and programs the node's networking rules.

Conceptually:

```text
Destination:
10.96.50.100:80

        ↓

Choose backend

        ↓

10.244.1.10:8080
```

Or another request:

```text
10.96.50.100:80
        ↓
10.244.2.20:8080
```

---

# 5. kube-proxy modes

You should know:

```text
iptables
IPVS
nftables
```

The exact modes/features depend on your Kubernetes version and deployment.

Historically:

```text
iptables
IPVS
```

were commonly discussed.

Modern Kubernetes also has:

```text
nftables
```

as a kube-proxy backend.

---

# 6. kube-proxy with iptables

Conceptually:

```text
Pod
 |
 | destination = Service ClusterIP
 v
Linux kernel
 |
 | iptables rules
 v
Backend Pod
```

kube-proxy creates/maintains rules that effectively perform:

```text
10.96.50.100:80
       |
       +----> 10.244.1.10:8080
       |
       +----> 10.244.2.20:8080
       |
       +----> 10.244.3.30:8080
```

The actual packet processing is performed by the Linux networking stack.

---

# 7. Important: kube-proxy isn't DNS

Suppose your application does:

```bash
curl http://backend:80
```

Two things happen:

### DNS

```text
backend
  |
  v
DNS
  |
  v
10.96.50.100
```

### Service routing

```text
10.96.50.100
  |
  v
Service dataplane
  |
  v
10.244.x.x
```

So:

```text
DNS → CoreDNS

Service forwarding → kube-proxy/dataplane
```

---

# 8. CoreDNS

Now let's move to DNS.

Suppose your Pod runs:

```bash
curl http://backend
```

How does `backend` become an IP?

The Pod's `/etc/resolv.conf` typically points to the cluster DNS service.

For example:

```text
nameserver 10.96.0.10
```

Then:

```text
Pod
 |
 | DNS query:
 | backend.default.svc.cluster.local
 v
Cluster DNS
 |
 v
CoreDNS
 |
 v
10.96.50.100
```

---

# 9. What is CoreDNS?

**CoreDNS is the DNS server commonly used by Kubernetes clusters.**

It provides DNS records for Kubernetes Services and Pods according to the Kubernetes DNS conventions.

For example:

```text
backend.default.svc.cluster.local
```

can resolve to:

```text
10.96.50.100
```

---

# 10. Kubernetes DNS name

A Service named:

```text
backend
```

in namespace:

```text
default
```

normally gets the DNS name:

```text
backend.default.svc.cluster.local
```

Break it down:

```text
backend
   |
   +-- Service name

default
   |
   +-- Namespace

svc
   |
   +-- Service

cluster.local
   |
   +-- Cluster DNS domain
```

So:

```text
backend.default.svc.cluster.local
```

means:

> Service `backend` in namespace `default`.

---

# 11. CoreDNS architecture

Usually you have:

```text
             CoreDNS
                |
        +-------+-------+
        |               |
     CoreDNS          CoreDNS
      Pod 1             Pod 2
        |               |
        +-------+-------+
                |
        Kubernetes API
```

CoreDNS watches Kubernetes resources and uses its Kubernetes plugin to serve DNS records.

---

# 12. CoreDNS example

Suppose:

```text
Service:
backend

Namespace:
production

ClusterIP:
10.96.100.50
```

A Pod can query:

```bash
nslookup backend.production.svc.cluster.local
```

Conceptually:

```text
Pod
 |
 | DNS query
 v
CoreDNS
 |
 | Kubernetes DNS data
 v
10.96.100.50
```

Then the application connects to:

```text
10.96.100.50
```

and the Service dataplane sends the traffic to a backend Pod.

---

# 13. kube-dns

This causes a lot of confusion.

Historically, Kubernetes used a DNS implementation called:

```text
kube-dns
```

Later, **CoreDNS became the common/default DNS implementation**.

So when you hear:

> "Check kube-dns"

don't automatically assume there is a separate DNS implementation running.

In many clusters you'll see:

```bash
kubectl get pods -n kube-system
```

and find:

```text
coredns-xxxxx
coredns-yyyyy
```

rather than Pods literally called `kube-dns`.

---

# 14. Why do people still say kube-dns?

Because Kubernetes has traditionally exposed the cluster DNS through a Service often named:

```text
kube-dns
```

You may see:

```bash
kubectl get svc -n kube-system
```

and get something like:

```text
NAME       TYPE       CLUSTER-IP
kube-dns   ClusterIP  10.96.0.10
```

while the actual DNS Pods are:

```text
coredns-xxxxx
coredns-yyyyy
```

This is an important distinction:

```text
kube-dns
   ↓
Often the Kubernetes DNS Service name

CoreDNS
   ↓
DNS server implementation
```

---

# 15. Typical CoreDNS setup

You might see:

```text
kube-system namespace

Service:
kube-dns
    |
    v
10.96.0.10
    |
    +---------+---------+
    |                   |
    v                   v
CoreDNS Pod 1       CoreDNS Pod 2
```

Pods use:

```text
nameserver 10.96.0.10
```

So:

```text
Pod
 |
 | DNS query
 v
kube-dns Service
 |
 +--------+--------+
 |                 |
 v                 v
CoreDNS-1       CoreDNS-2
```

---

# 16. NodeLocal DNSCache

Now we come to:

```text
NodeLocal DNSCache
```

This is designed to improve DNS performance and reduce DNS traffic to the central CoreDNS Service.

Instead of:

```text
Every Pod
    |
    v
kube-dns Service
    |
    v
CoreDNS
```

you can have:

```text
Every Pod
    |
    v
NodeLocal DNSCache
    |
    +---- cache hit → answer immediately
    |
    +---- cache miss
             |
             v
          CoreDNS
```

---

# 17. Why NodeLocal DNSCache?

Imagine:

```text
1000 Pods
```

All continuously making DNS queries.

Without local caching:

```text
1000 Pods
    |
    v
kube-dns Service
    |
    v
CoreDNS
```

This creates lots of DNS traffic.

With NodeLocal DNSCache:

```text
                 Node 1
          +------------------+
          |                  |
Pods ---->| NodeLocal DNS    |
          |     Cache        |
          +--------+---------+
                   |
                   | cache miss
                   v
                CoreDNS


                 Node 2
          +------------------+
          |                  |
Pods ---->| NodeLocal DNS    |
          |     Cache        |
          +--------+---------+
                   |
                   v
                CoreDNS
```

Each node has a local DNS cache.

---

# 18. NodeLocal DNSCache is a DaemonSet

Typically:

```text
One DNS cache Pod per node
```

So:

```text
Node 1
  |
  +-- NodeLocal DNSCache

Node 2
  |
  +-- NodeLocal DNSCache

Node 3
  |
  +-- NodeLocal DNSCache
```

This is commonly implemented as a DaemonSet.

---

# 19. DNS request with NodeLocal DNSCache

Suppose:

```bash
curl http://backend
```

Application first performs:

```text
DNS:
backend.default.svc.cluster.local
```

Flow:

```text
Application
     |
     v
Pod DNS configuration
     |
     v
NodeLocal DNSCache
     |
     +------ Cache HIT ------> IP returned
     |
     |
     +------ Cache MISS -----> CoreDNS
                                  |
                                  v
                              DNS response
                                  |
                                  v
                         NodeLocal DNSCache
                                  |
                                  v
                              Application
```

Next query can be served from local cache.

---

# 20. CoreDNS vs NodeLocal DNSCache

| Feature              | CoreDNS                | NodeLocal DNSCache            |
| -------------------- | ---------------------- | ----------------------------- |
| Main purpose         | DNS server             | DNS caching                   |
| Runs                 | Usually multiple Pods  | Usually one per node          |
| Watches K8s DNS data | Yes                    | No, primarily forwards/caches |
| Provides Service DNS | Yes                    | Caches/forwards it            |
| Cache                | Yes, CoreDNS can cache | Local cache                   |
| Reduces DNS traffic  | Base DNS service       | Yes, significantly            |
| Typical deployment   | Deployment             | DaemonSet                     |

---

# 21. Full DNS architecture without NodeLocal DNS

```text
                 Kubernetes Cluster
                       |
                       |
               +-------+-------+
               |               |
              Pod A           Pod B
               |               |
               | DNS           | DNS
               |               |
               +-------+-------+
                       |
                       v
                kube-dns Service
                  10.96.0.10
                       |
                +------+------+
                |             |
                v             v
             CoreDNS       CoreDNS
                |             |
                +------+------+
                       |
                 Kubernetes API
```

---

# 22. Full architecture with NodeLocal DNS

```text
                    Kubernetes
                         |
          +--------------+--------------+
          |                             |
       Node 1                          Node 2
          |                             |
     +----+-----+                  +----+-----+
     |          |                  |          |
   Pod A      Pod B              Pod C      Pod D
     |          |                  |          |
     +----+-----+                  +----+-----+
          |                             |
          v                             v
   NodeLocal DNS                  NodeLocal DNS
       Cache                          Cache
          |                             |
          +-------------+---------------+
                        |
                        v
                    CoreDNS
                  +---------+
                  |         |
               CoreDNS   CoreDNS
```

---

# 23. Now connect everything

Let's take this real request:

```bash
curl http://backend.default.svc.cluster.local
```

The complete flow is:

```text
                 Frontend Pod
                      |
                      |
             DNS lookup
                      |
                      v
             NodeLocal DNS
                  Cache
                      |
                      | cache miss
                      v
                  CoreDNS
                      |
                      |
                returns:
              10.96.50.100
                      |
                      v
             Frontend Pod
                      |
             connects to
          10.96.50.100:80
                      |
                      v
          Service dataplane
       kube-proxy / equivalent
                      |
             +--------+--------+
             |                 |
             v                 v
         Backend Pod 1      Backend Pod 2
        10.244.1.10       10.244.2.20
```

Notice the two distinct stages:

### Stage 1 — DNS

```text
backend.default.svc.cluster.local
                  ↓
             10.96.50.100
```

Handled by:

```text
NodeLocal DNS
CoreDNS
```

### Stage 2 — Service routing

```text
10.96.50.100
      ↓
Backend Pod
```

Traditionally handled by:

```text
kube-proxy
```

or potentially another service dataplane such as an eBPF-based implementation.

---

# 24. Important: DNS does NOT return Pod IPs for a normal Service

Suppose:

```text
backend Service
ClusterIP = 10.96.50.100

Backend Pods:
10.244.1.10
10.244.2.20
10.244.3.30
```

Normal Service DNS:

```text
backend.default.svc.cluster.local
```

resolves to:

```text
10.96.50.100
```

not:

```text
10.244.1.10
10.244.2.20
10.244.3.30
```

Then the Service dataplane selects a backend.

---

# 25. Headless Service is different

If you create:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend
spec:
  clusterIP: None
  selector:
    app: backend
  ports:
    - port: 80
```

This is a **headless Service**.

Now DNS can return the Pod IPs rather than a normal ClusterIP.

Conceptually:

```text
backend.default.svc.cluster.local
             |
             v
          CoreDNS
             |
       +-----+-----+
       |     |     |
       v     v     v
    Pod IP Pod IP Pod IP
```

Example:

```text
10.244.1.10
10.244.2.20
10.244.3.30
```

This is heavily used with:

```text
StatefulSets
databases
distributed systems
```

---

# 26. How to troubleshoot DNS

If an application says:

```text
Could not resolve backend
```

you should think:

```text
Is the Pod's DNS configuration correct?
             |
             v
Is NodeLocal DNS working?
             |
             v
Is kube-dns Service working?
             |
             v
Are CoreDNS Pods healthy?
             |
             v
Can CoreDNS reach the Kubernetes API?
```

Useful commands:

```bash
kubectl get pods -n kube-system
```

Look for:

```text
coredns-xxxxx
```

Then:

```bash
kubectl get svc -n kube-system
```

You may see:

```text
kube-dns
```

Check DNS from a Pod:

```bash
kubectl exec -it <pod> -- cat /etc/resolv.conf
```

You may see something like:

```text
nameserver 10.96.0.10
search default.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```

Then:

```bash
kubectl exec -it <pod> -- nslookup kubernetes.default.svc.cluster.local
```

---

# 27. How to troubleshoot Service networking

If DNS works:

```text
backend → 10.96.50.100
```

but:

```bash
curl http://backend
```

fails, then don't immediately blame CoreDNS.

Test the Service:

```bash
kubectl get svc backend
```

Then:

```bash
kubectl get endpointslice
```

or:

```bash
kubectl get endpoints backend
```

Check:

```text
Service
  |
  +-- Selector
  |
  +-- EndpointSlice
       |
       +-- Pod IPs
```

Then investigate:

```text
kube-proxy
CNI
NetworkPolicy
Pod health
Application listening port
```

---

# 28. kube-proxy vs CNI

Another important interview distinction:

```text
CNI
 ↓
Pod networking
```

Examples:

```text
Pod gets IP
Pod connects to another Pod
Pod connects to node/network
```

While:

```text
kube-proxy
 ↓
Service networking
```

Examples:

```text
ClusterIP
NodePort
Service → backend Pod
```

Conceptually:

```text
             Pod
              |
       +------+------+
       |             |
       v             v
      CNI        Service traffic
       |             |
       v             v
 Pod networking   kube-proxy
```

There is overlap in the broader dataplane, especially with eBPF implementations, but this distinction is excellent for interviews.

---

# 29. One important modern caveat

Don't say:

> "Every Kubernetes cluster must have kube-proxy."

That's no longer universally true.

Some networking implementations can replace kube-proxy functionality with an **eBPF-based service dataplane**.

For example, certain Cilium configurations can provide Kubernetes Service handling without relying on kube-proxy.

So a better interview statement is:

> **kube-proxy is the traditional Kubernetes component responsible for implementing Service networking rules, but some modern networking solutions can provide equivalent Service dataplane functionality without kube-proxy.**

---

# 30. Final cheat sheet

```text
┌──────────────────────────────────────────────┐
│                  KUBERNETES                  │
│                                              │
│  Service networking                         │
│       ↓                                      │
│  kube-proxy / service dataplane              │
│       ↓                                      │
│  ClusterIP → Backend Pod                     │
│                                              │
│  DNS                                         │
│       ↓                                      │
│  CoreDNS                                     │
│       ↓                                      │
│  Service DNS records                         │
│                                              │
│  DNS caching                                 │
│       ↓                                      │
│  NodeLocal DNSCache                          │
│       ↓                                      │
│  Local cache → CoreDNS on miss               │
└──────────────────────────────────────────────┘
```

### Memorize these four sentences:

**kube-proxy**

> Implements Kubernetes Service networking on nodes by programming a networking dataplane such as iptables, IPVS, or nftables, depending on configuration.

**CoreDNS**

> Provides DNS resolution for Kubernetes Services and other cluster DNS records.

**kube-dns**

> Historically the Kubernetes DNS implementation, and also commonly the name of the Kubernetes DNS Service even when CoreDNS is the actual DNS server.

**NodeLocal DNSCache**

> Runs a DNS cache locally on each node to reduce DNS latency and traffic to the central CoreDNS service.

And the complete request:

```text
curl http://backend
       |
       v
DNS lookup
       |
       v
NodeLocal DNSCache
       |
       | cache miss
       v
CoreDNS
       |
       v
Service ClusterIP
       |
       v
kube-proxy / service dataplane
       |
       v
Backend Pod
```

**That DNS → Service → Pod flow is the one I would definitely be able to draw on a whiteboard in a DevOps/Kubernetes interview.**
