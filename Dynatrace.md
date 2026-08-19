# DT ActiveGate vs OneAgent
This is an important Dynatrace topic for a **Senior DevOps / Platform Engineer**, especially because you need to understand where **OneAgent, ActiveGate, Dynatrace Operator, Fluent Bit and Dynatrace itself** fit together.

The biggest thing to remember is:

> **OneAgent = collects/observes.**
> **ActiveGate = gateway/proxy and, depending on enabled capabilities, performs monitoring/remote collection.**

And **ActiveGate is not mandatory in every architecture**. With Dynatrace SaaS, OneAgent can communicate directly with the Dynatrace cluster if network connectivity allows it. ActiveGate becomes particularly valuable for private/restricted networks, centralized routing, Kubernetes/API monitoring, remote technologies, and other capabilities. ([Dynatrace Documentation][1])

---

# 1. First understand the Dynatrace architecture

For your EKS environment, think of:

```text
                         Dynatrace SaaS
                              |
                              |
                    +---------+---------+
                    |                   |
                    | HTTPS            |
                    |                   |
              +-----+------+       +----+-----+
              | ActiveGate |       | OneAgent |
              |            |       |          |
              +-----+------+       +----------+
                    |
                    |
             Kubernetes / Network
                    |
       +------------+------------+
       |            |            |
       v            v            v
     Node 1       Node 2       Node 3
       |            |            |
   OneAgent     OneAgent     OneAgent
       |            |            |
       v            v            v
   Applications / Processes
```

But there are actually **two separate responsibilities**:

```text
ONEAGENT
    ↓
Observe the workload/host/application

ACTIVEGATE
    ↓
Provide local gateway/proxy
+
Monitor remote technologies / APIs
+
Route traffic
```

---

# 2. What is OneAgent?

**Dynatrace OneAgent is the monitoring agent installed on a host/workload that collects infrastructure, process and application telemetry.**

Dynatrace describes OneAgent as the component responsible for collecting monitoring data in the monitored environment. It discovers processes and can activate technology-specific instrumentation, providing deeper application/code-level visibility. ([Dynatrace Documentation][2])

For example:

```text
Linux server
     |
     +-- Java application
     +-- Nginx
     +-- Redis
     +-- OS
```

OneAgent can observe:

```text
CPU
Memory
Disk
Network
Processes
Java
.NET
Node.js
PHP
etc.
```

And for supported technologies it can perform deeper instrumentation.

---

# 3. OneAgent is not just a metrics exporter

This is an important distinction.

Don't describe OneAgent as:

> "Dynatrace's Prometheus exporter."

That's incorrect.

OneAgent can provide:

```text
Infrastructure monitoring
        +
Process monitoring
        +
Application monitoring
        +
Distributed tracing
        +
Code-level visibility
        +
Network/process relationships
        +
Log monitoring
```

Dynatrace documents OneAgent as discovering processes and automatically activating instrumentation for supported application stacks. ([Dynatrace Documentation][3])

---

# 4. OneAgent example

Suppose you have:

```text
EKS
 |
 +-- payment-api
 |      |
 |      +-- Java
 |
 +-- order-api
        |
        +-- Node.js
```

With OneAgent:

```text
              EKS NODE
                 |
        +--------+--------+
        |                 |
        v                 v
   payment-api        order-api
      Java               Node.js
        |                  |
        +--------+---------+
                 |
             OneAgent
                 |
                 v
             Dynatrace
```

Dynatrace can understand:

```text
Payment API
    |
    v
PostgreSQL
    |
    v
Redis
```

and:

```text
Frontend
    |
    v
Payment API
    |
    v
Database
```

That relationship/topology is a major value of full-stack monitoring.

---

# 5. What does OneAgent actually monitor?

Think in layers:

```text
                 ONEAGENT
                    |
       +------------+-------------+
       |            |             |
       v            v             v
    HOST         PROCESS       APPLICATION
       |            |             |
       |            |             |
     CPU          JVM           Requests
     RAM          Node          Errors
     Disk         .NET          Latency
     Network      etc.          Traces
```

Depending on technology and configuration, OneAgent can inject supported code modules into application processes for deeper monitoring. ([Dynatrace Documentation][3])

---

# 6. What is ActiveGate?

ActiveGate is a **Dynatrace component that provides a local gateway/proxy between OneAgents and Dynatrace**, and it can also perform monitoring tasks itself.

Dynatrace explicitly describes ActiveGate as a secure proxy and says it can reduce network traffic and simplify connectivity, particularly in sealed/restricted networks. It can also monitor cloud and remote technologies using its functional modules. ([Dynatrace Documentation][1])

Think:

```text
OneAgent
    |
    | local/private network
    v
ActiveGate
    |
    | outbound
    v
Dynatrace
```

---

# 7. Why was ActiveGate introduced?

Imagine a large enterprise:

```text
                 Internet
                    |
             Firewall / Proxy
                    |
             Dynatrace SaaS
```

Inside:

```text
                     Enterprise
                         |
        +----------------+----------------+
        |                |                |
      DC-1             DC-2             AWS
        |                |                |
    500 servers       300 servers      EKS
```

You don't necessarily want every OneAgent individually establishing external connectivity.

Instead:

```text
DC-1
 |
 +-- 500 OneAgents
        |
        v
    ActiveGate
        |
        v
 Dynatrace SaaS
```

So ActiveGate acts as a local Dynatrace presence.

---

# 8. ActiveGate has TWO major concepts you need to remember

### A. Routing

```text
OneAgent
    |
    v
ActiveGate
    |
    v
Dynatrace
```

### B. Monitoring

ActiveGate itself can query/monitor technologies where installing OneAgent isn't the right approach.

For example:

```text
ActiveGate
    |
    +-- AWS
    +-- Kubernetes API
    +-- VMware
    +-- databases
    +-- network technologies
    +-- Prometheus endpoints
```

Dynatrace documents both OneAgent routing and remote/cloud monitoring as ActiveGate capabilities. ([Dynatrace Documentation][1])

---

# 9. The biggest difference

Memorize this:

```text
ONEAGENT
========

Lives with / on the monitored workload
        |
        v
Collects telemetry from that environment


ACTIVEGATE
==========

Lives between your environment and Dynatrace
        |
        v
Routes traffic
+
Can perform remote monitoring
```

---

# 10. OneAgent → ActiveGate → Dynatrace

Let's take a simple VM.

```text
              AWS VPC
+-----------------------------------------+
|                                         |
|   EC2                                   |
|   +------------------+                  |
|   | Java application |                  |
|   +--------+---------+                  |
|            |                            |
|            v                            |
|        OneAgent                         |
|            |                            |
|            | HTTPS                      |
|            v                            |
|       ActiveGate                       |
|            |                            |
+------------|----------------------------+
             |
             | HTTPS outbound
             v
       Dynatrace SaaS
```

OneAgent collects:

```text
Java metrics
HTTP requests
traces
errors
CPU
memory
process information
```

ActiveGate:

```text
receives OneAgent traffic
routes it
compresses/buffers/authenticates as applicable
```

Dynatrace:

```text
stores/processes/visualizes
```

ActiveGate's routing module handles routing, buffering, compression and authentication functions for OneAgent traffic. ([Dynatrace Documentation][4])

---

# 11. Does OneAgent REQUIRE ActiveGate?

**No.**

This is one of the most important answers to your question.

For Dynatrace SaaS:

```text
OneAgent
    |
    | HTTPS
    v
Dynatrace SaaS
```

can work.

Dynatrace explicitly documents that OneAgent can connect directly to the Dynatrace cluster, or through one or more ActiveGates. ([Dynatrace Documentation][3])

So:

```text
             ActiveGate present?
                    |
          +---------+---------+
          |                   |
         YES                  NO
          |                   |
          v                   v
      OneAgent            OneAgent
          |                   |
          v                   v
     ActiveGate          Dynatrace
          |
          v
      Dynatrace
```

---

# 12. What happens if ActiveGate is NOT present?

### Scenario 1 — Network allows direct Dynatrace connectivity

Then:

```text
OneAgent
    |
    | HTTPS
    v
Dynatrace SaaS
```

Everything can still work for the capabilities supported by OneAgent.

Dynatrace's current connectivity documentation explicitly describes direct SaaS connectivity as an alternative path when an Environment ActiveGate isn't reachable. ([Dynatrace Documentation][5])

---

# 13. Scenario 2 — ActiveGate is mandatory because of network restrictions

Now imagine:

```text
EKS
 |
 | NO INTERNET ACCESS
 |
 X
 |
Dynatrace SaaS
```

You deploy:

```text
EKS
 |
 +-- OneAgent
 |
 +-- ActiveGate
```

Firewall:

```text
EKS
 |
 | allowed
 v
ActiveGate
 |
 | allowed
 v
Dynatrace
```

Architecture:

```text
                 INTERNET
                    |
              Dynatrace SaaS
                    ^
                    |
                 HTTPS
                    |
              +-----+------+
              | ActiveGate |
              +-----+------+
                    ^
                    |
              Private network
                    |
              +-----+------+
              | EKS        |
              |            |
              | OneAgents  |
              +------------+
```

Here ActiveGate is effectively the **controlled egress gateway**.

---

# 14. Why this matters in enterprise

Suppose security says:

> "Only these two servers can communicate with Dynatrace."

You don't want:

```text
1000 servers
   |
   +-- outbound to Dynatrace
```

Instead:

```text
1000 OneAgents
      |
      v
2 ActiveGates
      |
      v
Dynatrace
```

Now your firewall policy becomes:

```text
Only ActiveGate → Dynatrace
```

This dramatically simplifies network architecture.

---

# 15. ActiveGate HA

You generally don't want:

```text
500 OneAgents
      |
      v
   ActiveGate
      X
```

Instead:

```text
                 Dynatrace
                    ^
                    |
             +------+------+
             |             |
        ActiveGate-1   ActiveGate-2
             ^             ^
             |             |
             +------+------+
                    |
               OneAgents
```

If one ActiveGate becomes unavailable, OneAgents can use another available ActiveGate according to Dynatrace's connectivity and network-zone rules. ([Dynatrace Documentation][6])

---

# 16. Network zones

For a large enterprise, you'll often see:

```text
              Dynatrace
                  |
       +----------+----------+
       |                     |
     Zone-A                Zone-B
       |                     |
   ActiveGate             ActiveGate
       |                     |
   EKS Mumbai           EKS Bangalore
```

Network zones influence which ActiveGates OneAgents prefer and provide fallback behavior. ([Dynatrace Documentation][7])

This prevents something like:

```text
EKS India
   |
   v
ActiveGate Europe
   |
   v
Dynatrace
```

when you could use a local ActiveGate.

---

# 17. ActiveGate is NOT a replacement for OneAgent

This is a common misconception.

Suppose:

```text
ActiveGate
```

is installed.

Does it automatically give you:

```text
Java code-level monitoring?
Distributed tracing?
Process-level application instrumentation?
```

**No.**

ActiveGate and OneAgent have different responsibilities.

```text
             Dynatrace
                 ^
                 |
           ActiveGate
                 ^
                 |
              Network
                 ^
                 |
             OneAgent
                 ^
                 |
            Application
```

---

# 18. Can ActiveGate monitor things without OneAgent?

**Yes.**

This is another key distinction.

Suppose:

```text
VMware vCenter
```

You may not install OneAgent on vCenter itself.

Instead:

```text
ActiveGate
     |
     | API
     v
VMware vCenter
```

ActiveGate queries it.

Similarly, depending on enabled capabilities:

```text
ActiveGate
   |
   +---- AWS APIs
   |
   +---- Kubernetes API
   |
   +---- VMware
   |
   +---- remote technologies
```

Dynatrace documents ActiveGate extensions for remote technologies where OneAgent installation isn't an option. ([Dynatrace Documentation][8])

---

# 19. Now bring Kubernetes into the picture

This is especially relevant for you.

Modern Dynatrace Kubernetes architecture can look like:

```text
                    Dynatrace
                        ^
                        |
                    ActiveGate
                        |
                Kubernetes API
                        |
            +-----------+-----------+
            |                       |
          Node 1                  Node 2
            |                       |
        OneAgent                 OneAgent
            |                       |
       +----+----+             +----+----+
       |         |             |         |
      Pod       Pod           Pod       Pod
```

The ActiveGate can provide Kubernetes monitoring capabilities, while OneAgent provides node/process/application monitoring. Dynatrace's current full-stack Kubernetes documentation describes ActiveGate as routing observability data and monitoring the Kubernetes API, while OneAgent collects host metrics and code modules provide deep application monitoring. ([Dynatrace Documentation][9])

---

# 20. Where Dynatrace Operator fits

This is another component you need to understand.

You don't necessarily manually install:

```text
OneAgent
ActiveGate
```

everywhere.

In Kubernetes, you can use:

```text
Dynatrace Operator
```

which manages Dynatrace components using Kubernetes-native resources.

Architecture:

```text
                   Dynatrace
                       ^
                       |
              +--------+--------+
              |                 |
         ActiveGate          OneAgent
              ^                 ^
              |                 |
              +-------+---------+
                      |
               Dynatrace Operator
                      |
                      v
                  DynaKube
                      |
                      v
                  Kubernetes
```

---

# 21. DynaKube

You define desired Dynatrace configuration through a `DynaKube` custom resource.

Conceptually:

```yaml
apiVersion: dynatrace.com/v1beta5
kind: DynaKube

metadata:
  name: dynakube

spec:

  oneAgent:
    cloudNativeFullStack: {}

  activeGate:
    capabilities:
      - routing
      - kubernetes-monitoring
```

The exact fields depend on the Dynatrace Operator/API version you're using.

The important concept is:

```text
DynaKube
   |
   v
Dynatrace Operator
   |
   +---- OneAgent
   |
   +---- ActiveGate
   |
   +---- injection/configuration
```

---

# 22. Full-stack monitoring

Suppose:

```text
EKS
 |
 +-- payment-api
 |      |
 |      +-- Java
 |
 +-- order-api
        |
        +-- Node.js
```

With full-stack monitoring:

```text
                  Dynatrace
                      ^
                      |
                  ActiveGate
                      ^
                      |
              Kubernetes API
                      ^
                      |
                OneAgent
                      |
          +-----------+-----------+
          |                       |
      payment-api             order-api
          |                       |
        Java                    Node.js
          |                       |
       code module            code module
```

Dynatrace's Kubernetes full-stack model uses OneAgent for node monitoring and injected code modules for deep application monitoring. ([Dynatrace Documentation][9])

---

# 23. How OneAgent gets into application Pods

This is very important for Kubernetes interviews.

Dynatrace Operator can use **mutating webhook-based injection** for OneAgent code modules.

Conceptually:

```text
kubectl apply Deployment
             |
             v
     Kubernetes API
             |
             v
       Admission webhook
             |
             v
     Pod specification modified
             |
             v
      OneAgent code module
             |
             v
        Application
```

Dynatrace documents full-stack observability as using mutating webhooks to inject code modules into application Pods. ([Dynatrace Documentation][9])

---

# 24. So where does Fluent Bit fit?

This connects directly to your previous question.

You now have:

```text
                   EKS
                    |
       +------------+-------------+
       |                          |
       v                          v
   OneAgent                   Fluent Bit
       |                          |
       | metrics/traces           | logs
       |                          |
       +-------------+------------+
                     |
                     v
                ActiveGate
                     |
                     v
                Dynatrace
```

But **don't assume Fluent Bit must send through ActiveGate in every architecture**.

The exact ingestion path depends on the Dynatrace integration and configuration.

Conceptually:

```text
OneAgent
    |
    v
ActiveGate
    |
    v
Dynatrace
```

while:

```text
Fluent Bit
    |
    v
Dynatrace log ingestion
```

may use its configured ingestion endpoint, potentially through an ActiveGate/OTLP gateway depending on your architecture.

---

# 25. Important distinction: ActiveGate vs Fluent Bit

Don't confuse these.

### Fluent Bit

```text
Purpose:
LOG COLLECTION

Reads:
stdout/stderr
files

Processes:
logs

Sends:
logs
```

### ActiveGate

```text
Purpose:
GATEWAY + REMOTE MONITORING

Handles:
OneAgent routing
remote monitoring
Kubernetes/API monitoring
extensions
```

So:

```text
Fluent Bit ≠ ActiveGate
```

---

# 26. Demo 1 — OneAgent WITHOUT ActiveGate

Let's start with the simplest architecture.

```text
                    Dynatrace SaaS
                         ^
                         |
                         | HTTPS
                         |
                    OneAgent
                         ^
                         |
                    EC2 / EKS
                         ^
                         |
                    Application
```

Requirements:

```text
Outbound connectivity
DNS resolution
TLS connectivity
valid Dynatrace credentials/configuration
```

If those are satisfied:

```text
OneAgent
    |
    +---- CPU
    +---- Memory
    +---- Process
    +---- Application
    +---- Traces
    |
    v
Dynatrace
```

No ActiveGate required.

Dynatrace explicitly states that in the absence of an ActiveGate, OneAgent can send collected data directly to the Dynatrace cluster. ([Dynatrace Documentation][10])

---

# 27. Demo 2 — Add ActiveGate

Now suppose security says:

> "EKS cannot directly access Dynatrace SaaS."

Deploy:

```text
EKS
 |
 +-- OneAgent
 |
 +-- ActiveGate
```

Network:

```text
OneAgent
    |
    | internal/private
    v
ActiveGate
    |
    | HTTPS outbound
    v
Dynatrace
```

Now OneAgent doesn't need direct Dynatrace connectivity.

---

# 28. What changed?

Before:

```text
OneAgent
   |
   +---- Internet
   |
   v
Dynatrace
```

After:

```text
OneAgent
   |
   v
ActiveGate
   |
   v
Dynatrace
```

The **monitoring data still originates from OneAgent**.

ActiveGate is primarily providing the communication path.

---

# 29. Demo 3 — ActiveGate monitoring Kubernetes

Now suppose:

```text
EKS
```

and you want Kubernetes-specific monitoring.

Architecture:

```text
                 Dynatrace
                     ^
                     |
                 ActiveGate
                     |
                     | Kubernetes API
                     v
              Kubernetes API Server
                     |
          +----------+----------+
          |          |          |
         Pods       Nodes    Deployments
```

ActiveGate can use its Kubernetes monitoring capability to query Kubernetes APIs and collect cluster-level information. Dynatrace's full-stack documentation describes ActiveGate as monitoring the Kubernetes API in this architecture. ([Dynatrace Documentation][9])

---

# 30. Demo 4 — Remote database

Suppose:

```text
RDS PostgreSQL
```

You don't install OneAgent into RDS.

Instead, depending on the Dynatrace-supported monitoring capability:

```text
ActiveGate
     |
     | database connection/API
     v
RDS PostgreSQL
```

ActiveGate collects remote database information.

So:

```text
OneAgent
   ↓
Good for processes/applications/hosts

ActiveGate
   ↓
Good for remote technologies where
OneAgent isn't installed
```

---

# 31. What if OneAgent is missing?

This is important.

Suppose:

```text
ActiveGate
    |
    v
Dynatrace
```

but:

```text
NO OneAgent
```

Can Dynatrace still monitor some things?

**Yes.**

For example, ActiveGate can perform remote/cloud/Kubernetes/API monitoring using its enabled capabilities.

But you lose the telemetry that OneAgent would have provided.

For an application:

```text
Java application
```

without OneAgent, you generally won't get the same OneAgent-based deep process/code-level visibility.

So:

```text
ActiveGate only
       |
       v
Remote/API monitoring
```

is possible.

But:

```text
ActiveGate only
       X
Deep OneAgent application instrumentation
```

---

# 32. What if ActiveGate is missing?

Opposite scenario:

```text
OneAgent
    |
    v
Dynatrace SaaS
```

If direct connectivity is allowed:

**works.**

But you lose the ActiveGate-specific capabilities/path.

For example:

```text
No ActiveGate
     |
     +-- Direct OneAgent → SaaS
     |
     +-- No local proxy/gateway
     |
     +-- Remote monitoring capabilities requiring ActiveGate
          aren't available through that missing component
```

---

# 33. What if BOTH are missing?

```text
Application
     |
     X
No OneAgent
     |
     X
No ActiveGate
```

Dynatrace doesn't magically get deep application telemetry.

You might still send telemetry using other integrations:

```text
Prometheus
Fluent Bit
OpenTelemetry
Cloud integrations
```

depending on what you're monitoring.

For example:

```text
Fluent Bit
     |
     v
Dynatrace
```

can still provide logs even if OneAgent isn't installed.

This is why you should think of Dynatrace as an **observability platform with multiple ingestion/collection mechanisms**, not "everything must go through OneAgent."

---

# 34. Very important architecture comparison

| Component          | Main job                           | Required?                                 |
| ------------------ | ---------------------------------- | ----------------------------------------- |
| OneAgent           | Host/process/application telemetry | Depends on monitoring goal                |
| ActiveGate         | Routing + remote monitoring        | Optional in direct SaaS connectivity      |
| Dynatrace Operator | Manage Dynatrace on Kubernetes     | Used for Kubernetes deployment/management |
| Fluent Bit         | Log collection/forwarding          | Optional; depends on logging architecture |
| Prometheus         | Metrics collection                 | Optional; depends on metrics architecture |

---

# 35. Think of it like this

```text
                    DYNATRACE
                       |
        +--------------+--------------+
        |              |              |
        |              |              |
        v              v              v
    OneAgent       Fluent Bit      Prometheus
        |              |              |
        |              |              |
    metrics        logs           metrics
    traces
    processes
        |
        |
        +--------+
                 |
                 v
             ActiveGate
                 |
                 v
             Dynatrace
```

Again, **not every arrow necessarily has to pass through ActiveGate**; it depends on your deployment and ingestion configuration.

---

# 36. The interview question: "Explain OneAgent vs ActiveGate"

A strong answer:

> **"OneAgent is the monitoring agent that runs with the monitored environment and collects host, process and application telemetry. For supported technologies it can inject code modules to provide deeper application and code-level visibility. ActiveGate is a gateway component that can route OneAgent traffic to Dynatrace and can also perform monitoring tasks itself, such as Kubernetes, cloud and remote-technology monitoring through its enabled capabilities. ActiveGate is not inherently mandatory for OneAgent. In a Dynatrace SaaS environment, OneAgent can communicate directly with the Dynatrace cluster if network connectivity allows it. I would deploy ActiveGate when I need centralized routing, restricted-egress architecture, network-zone control, or ActiveGate-based remote monitoring."** ([Dynatrace Documentation][1])

That's the answer I'd give in a senior interview.

---

# 37. The EKS architecture I'd draw

For your specific platform-engineering context:

```text
                              Internet
                                 |
                                 v
                         +---------------+
                         | Dynatrace SaaS |
                         +-------+-------+
                                 ^
                                 |
                              HTTPS
                                 |
                    +------------+------------+
                    |       ActiveGate        |
                    |                         |
                    | routing                 |
                    | kubernetes-monitoring   |
                    +------------+------------+
                                 ^
                                 |
                         Kubernetes API
                                 |
              +------------------+------------------+
              |                  |                  |
           Worker 1           Worker 2           Worker 3
              |                  |                  |
          OneAgent           OneAgent           OneAgent
              |                  |                  |
       +------+------+    +------+------+    +------+------+
       |             |    |             |    |             |
     Pod A         Pod B Pod C         Pod D Pod E        Pod F
       |             |    |             |    |             |
     Java          Node  Java         Python Go            Java
       |             |    |             |    |             |
       +-------------+----+-------------+----+-------------+
                             |
                        Fluent Bit
                       (DaemonSet)
                             |
                             |
                       Logs → Dynatrace
```

For **full-stack observability**, OneAgent and its injected code modules provide application/deep monitoring, ActiveGate provides routing and Kubernetes/API monitoring, and Fluent Bit can independently provide a log-shipping path. ([Dynatrace Documentation][9])

---

# 38. One more important thing: ActiveGate can be containerized

For EKS, you don't necessarily need:

```text
EC2 VM
  |
  +-- ActiveGate
```

You can run:

```text
EKS
 |
 +-- ActiveGate Pod
```

Dynatrace supports containerized ActiveGate deployments, including Kubernetes. ([Dynatrace Documentation][1])

Architecture:

```text
EKS
 |
 +-- ActiveGate Pod
 |
 +-- OneAgent
 |
 +-- Application Pods
```

This is very natural for a Kubernetes platform.

---

# 39. ActiveGate capabilities

An ActiveGate isn't just one universal feature.

Dynatrace has different purposes/modules, including:

```text
Routing
Kubernetes monitoring
Cloud monitoring
Remote technology monitoring
Synthetic
API
```

The exact available modules depend on the ActiveGate deployment/purpose. Dynatrace explicitly distinguishes ActiveGate purposes and functional modules. ([Dynatrace Documentation][11])

So when designing:

```yaml
activeGate:
  capabilities:
    - routing
    - kubernetes-monitoring
```

you're effectively saying:

> "I want this ActiveGate to perform these capabilities."

---

# 40. The complete mental model

Memorize this:

```text
                         Dynatrace SaaS
                               ^
                               |
                      +--------+--------+
                      |                 |
                 ActiveGate          Direct
                      ^                 ^
                      |                 |
                  OneAgent -------------+
                      ^
                      |
                Host / Process
                      ^
                      |
                 Application
```

And:

```text
ActiveGate
    |
    +---- routes OneAgent traffic
    |
    +---- monitors Kubernetes
    |
    +---- monitors cloud APIs
    |
    +---- monitors remote technologies
    |
    +---- provides local gateway
```

while:

```text
OneAgent
    |
    +---- Host metrics
    +---- Process metrics
    +---- Application metrics
    +---- Traces
    +---- Code-level monitoring
    +---- Network/process relationships
    +---- Log monitoring capabilities
```

---

# 41. The three scenarios you should remember

### Scenario A — No ActiveGate

```text
EKS
 |
OneAgent
 |
 | HTTPS
 v
Dynatrace SaaS
```

**Works if direct connectivity is allowed.**

---

### Scenario B — ActiveGate + OneAgent

```text
EKS
 |
OneAgent
 |
 v
ActiveGate
 |
 v
Dynatrace
```

**Best when you want centralized routing/restricted egress and/or ActiveGate capabilities.**

---

### Scenario C — ActiveGate without OneAgent

```text
EKS / Remote system
       |
       v
  ActiveGate
       |
       v
   Dynatrace
```

**Can work for ActiveGate-supported remote/API monitoring, but does not replace OneAgent's deep application/process instrumentation.**

---

# 42. And your Fluent Bit architecture

Since your previous question was specifically about Fluent Bit → Dynatrace, put everything together like this:

```text
                              DYNATRACE
                                  ^
                                  |
                     +------------+-------------+
                     |                          |
                  ActiveGate              Log ingestion
                     ^                          ^
                     |                          |
                  OneAgent                 Fluent Bit
                     ^                          ^
                     |                          |
              Application/Host            Pod stdout
                     ^                          ^
                     |                          |
                     +------------+-------------+
                                  |
                                EKS
```

**OneAgent ≠ Fluent Bit ≠ ActiveGate.**

```text
OneAgent
→ deep host/process/application observability

Fluent Bit
→ log collection and forwarding

ActiveGate
→ gateway/routing + remote/API monitoring
```

That's the clean mental model an interviewer is looking for. ([Dynatrace Documentation][1])

[1]: https://docs.dynatrace.com/docs/ingest-from/dynatrace-activegate?utm_source=chatgpt.com "Dynatrace ActiveGate — Dynatrace Docs"
[2]: https://docs.dynatrace.com/docs/ingest-from/dynatrace-oneagent?utm_source=chatgpt.com "Dynatrace OneAgent — Dynatrace Docs"
[3]: https://docs.dynatrace.com/docs/platform/oneagent/how-one-agent-works?utm_source=chatgpt.com "How OneAgent works — Dynatrace Docs"
[4]: https://docs.dynatrace.com/docs/ingest-from/dynatrace-activegate/configuration/configure-activegate?utm_source=chatgpt.com "Configuration properties and parameters of ActiveGate — Dynatrace Docs"
[5]: https://docs.dynatrace.com/docs/ingest-from/dynatrace-activegate/supported-connectivity-schemes-for-activegates?utm_source=chatgpt.com "Supported connectivity schemes for ActiveGates — Dynatrace Docs"
[6]: https://docs.dynatrace.com/docs/manage/network-zones/oneagent-connectivity?utm_source=chatgpt.com "OneAgent connectivity in network zones — Dynatrace Docs"
[7]: https://docs.dynatrace.com/docs/manage/network-zones/network-zones-basic-info?utm_source=chatgpt.com "Get started with network zones — Dynatrace Docs"
[8]: https://docs.dynatrace.com/docs/ingest-from/dynatrace-activegate/capabilities/routing-monitoring-purpose?utm_source=chatgpt.com "Route OneAgent traffic to Dynatrace, monitor cloud environments and remote technologies using extensions — Dynatrace Docs"
[9]: https://docs.dynatrace.com/docs/ingest-from/setup-on-k8s/how-it-works/cloud-native-fullstack?utm_source=chatgpt.com "Full-stack observability — Dynatrace Docs"
[10]: https://docs.dynatrace.com/docs/manage/data-privacy-and-security/data-security/data-security-controls?utm_source=chatgpt.com "Data security controls — Dynatrace Docs"
[11]: https://docs.dynatrace.com/docs/ingest-from/dynatrace-activegate/capabilities?utm_source=chatgpt.com "ActiveGate purposes and functionality — Dynatrace Docs"
