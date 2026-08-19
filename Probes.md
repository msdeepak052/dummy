# Kubernetes Probes

Kubernetes probes are how **kubelet checks the health/state of a container**.

There are **three probes** you need to know:

```text
                    Kubernetes Probes
                           |
            +--------------+--------------+
            |              |              |
            v              v              v
        Liveness        Readiness       Startup
        "Are you        "Can you        "Have you
         alive?"         receive         started?"
                         traffic?"
```

The most important interview distinction:

> **Liveness = restart the container if it's unhealthy**
> **Readiness = remove the Pod from Service traffic if it's not ready**
> **Startup = give slow-starting applications time to initialize before liveness/readiness checks take over**

---

# 1. Why do we need probes?

Imagine your container process is still running:

```text
Pod
 |
 +-- application process
       |
       +-- PID exists
       |
       +-- but application is DEADLOCKED
```

From Kubernetes' perspective:

```text
Container process = running
```

But your application may not actually be usable.

That's where probes come in.

```text
Container running ≠ Application healthy
```

---

# 2. Liveness Probe

Liveness answers:

> **"Is this application still alive, or should Kubernetes restart the container?"**

Example:

```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 10
```

If the probe repeatedly fails:

```text
Liveness failure
       |
       v
kubelet considers container unhealthy
       |
       v
Container restarted
```

---

# 3. Simple liveness example

Suppose your application exposes:

```text
GET /health
```

and returns:

```http
HTTP/1.1 200 OK
```

Pod:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
    - name: app
      image: myapp:1.0

      ports:
        - containerPort: 8080

      livenessProbe:
        httpGet:
          path: /health
          port: 8080

        initialDelaySeconds: 10
        periodSeconds: 10
        timeoutSeconds: 2
        failureThreshold: 3
```

Kubelet effectively does:

```text
Every 10 seconds
       |
       v
GET /health
       |
   +---+---+
   |       |
  200    failure
   |       |
   v       v
Healthy  count failure
           |
           v
      3 consecutive
         failures
           |
           v
       Restart
```

---

# 4. Readiness Probe

Readiness answers:

> **"Can this Pod receive traffic right now?"**

This is different from liveness.

Suppose your application is alive:

```text
Application process = running
```

but:

```text
Database connection = unavailable
```

You might not want Kubernetes to restart the application.

You simply don't want it receiving traffic.

That's what readiness is for.

```text
Readiness failure
       |
       v
Pod removed from ready Service endpoints
       |
       v
No new Service traffic
```

The container itself continues running.

---

# 5. Readiness example

```yaml
readinessProbe:
  httpGet:
    path: /ready
    port: 8080

  initialDelaySeconds: 5
  periodSeconds: 5
  timeoutSeconds: 2
  failureThreshold: 3
```

Suppose:

```text
Pod A → Ready
Pod B → Ready
Pod C → Not Ready
```

Service:

```text
              Service
                 |
          +------+------+
          |             |
          v             v
        Pod A          Pod B
        Ready          Ready

Pod C
Not Ready
   |
   X
Not included in ready endpoints
```

The important point:

> **Readiness failure does NOT restart the container.**

---

# 6. Liveness vs Readiness

This is probably the most frequently asked probe question.

|                           | Liveness              | Readiness                   |
| ------------------------- | --------------------- | --------------------------- |
| Question                  | Is application alive? | Can it receive traffic?     |
| Failure action            | Restart container     | Remove from ready endpoints |
| Container restarted?      | Usually yes           | No                          |
| Used for traffic routing? | No                    | Yes                         |
| Example                   | Deadlock              | DB temporarily unavailable  |

Think:

```text
Liveness:
"Should I restart you?"

Readiness:
"Should I send traffic to you?"
```

---

# 7. Startup Probe

Now imagine your application takes:

```text
2 minutes
```

to start.

If you configure:

```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 8080
  periodSeconds: 10
  failureThreshold: 3
```

Kubernetes could conclude:

```text
Application starting...
   |
   | 10 sec
   v
health check fails
   |
   | 10 sec
   v
fails
   |
   | 10 sec
   v
fails
   |
   v
Restart!
```

But your application wasn't broken.

It was simply slow to start.

That's what **startupProbe** solves.

---

# 8. Startup probe example

```yaml
startupProbe:
  httpGet:
    path: /health
    port: 8080

  periodSeconds: 10
  failureThreshold: 18
```

This gives:

```text
10 seconds × 18 failures
≈ 180 seconds
```

for the application to successfully start.

While startup is still failing:

```text
startupProbe
     |
     v
Application still starting
```

Once startup succeeds:

```text
startupProbe = SUCCESS
        |
        v
Liveness/readiness become active
```

This is extremely useful for:

```text
Java applications
Large ML models
Applications with migrations
Slow initialization
Legacy applications
```

---

# 9. How the three probes work together

This is the diagram I'd memorize:

```text
                  Container starts
                         |
                         v
                  Startup Probe
                         |
                  +------+------+
                  |             |
                FAIL          SUCCESS
                  |             |
                  |             v
                  |       Liveness Probe
                  |             |
                  |       Readiness Probe
                  |             |
                  |       +-----+-----+
                  |       |           |
                  |     Ready       Not Ready
                  |       |           |
                  |       v           v
                  |    Traffic     No traffic
                  |
                  v
             Failure threshold
                  |
                  v
             Container restart
```

Important:

> **When a startup probe is configured, Kubernetes does not run the liveness/readiness probes until the startup probe succeeds.**

---

# 10. Probe types

Each probe can use several mechanisms.

The main ones are:

```text
HTTP
TCP
Exec
gRPC
```

---

# 11. HTTP probe

Most common.

```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 8080
```

Kubelet sends:

```text
GET /health
```

If it gets an acceptable HTTP response:

```text
200-399
```

the probe succeeds.

Example:

```text
Kubelet
   |
   | HTTP GET /health
   v
Container :8080
   |
   v
HTTP 200
```

---

# 12. TCP probe

Useful when your application doesn't have an HTTP health endpoint.

```yaml
livenessProbe:
  tcpSocket:
    port: 8080

  periodSeconds: 10
```

Kubelet tries to establish a TCP connection:

```text
Kubelet
   |
   | TCP connect
   v
Pod :8080
```

If the port accepts the connection:

```text
SUCCESS
```

Otherwise:

```text
FAIL
```

Example use cases:

```text
Redis
MySQL
TCP-based services
Custom protocols
```

But note: a successful TCP connection only proves the port is accepting connections. It doesn't prove the application is logically healthy.

---

# 13. Exec probe

Kubelet runs a command inside the container.

```yaml
livenessProbe:
  exec:
    command:
      - cat
      - /tmp/healthy
```

If the command returns:

```text
exit code 0
```

→ success.

Anything else:

```text
exit code != 0
```

→ failure.

Example:

```text
Container
 |
 +-- /tmp/healthy
       |
       +-- exists → success
       |
       +-- missing → failure
```

---

# 14. gRPC probe

For applications exposing gRPC health checking:

```yaml
livenessProbe:
  grpc:
    port: 50051
```

Useful for:

```text
gRPC services
```

You can also specify a service name when using gRPC health checking.

---

# 15. Probe timing parameters

These are very important.

Example:

```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 8080

  initialDelaySeconds: 10
  periodSeconds: 10
  timeoutSeconds: 2
  successThreshold: 1
  failureThreshold: 3
```

Let's understand each.

---

# 16. `initialDelaySeconds`

How long to wait before the first probe.

```yaml
initialDelaySeconds: 30
```

means:

```text
Container starts
      |
      | wait 30 sec
      v
First probe
```

However, for applications with unpredictable/long startup, **startupProbe is generally a better pattern** than trying to make a huge `initialDelaySeconds` for liveness.

---

# 17. `periodSeconds`

How frequently to run the probe.

```yaml
periodSeconds: 10
```

means approximately:

```text
Probe
 |
10 sec
 |
Probe
 |
10 sec
 |
Probe
```

---

# 18. `timeoutSeconds`

How long to wait for a probe attempt.

```yaml
timeoutSeconds: 2
```

If the application doesn't respond within the timeout:

```text
Probe attempt = failure
```

---

# 19. `failureThreshold`

How many consecutive failures are required before Kubernetes takes the probe's failure action.

```yaml
failureThreshold: 3
```

means:

```text
Failure 1
   ↓
Failure 2
   ↓
Failure 3
   ↓
Action
```

For liveness:

```text
restart container
```

For readiness:

```text
mark Pod NotReady
```

---

# 20. `successThreshold`

How many consecutive successful probes are required to consider the probe successful.

For example:

```yaml
successThreshold: 2
```

means:

```text
Success
   |
   v
Success
   |
   v
Healthy
```

For most probes, `successThreshold` is normally `1`.

Also remember that Kubernetes has restrictions around this field; for example, for liveness and startup probes it must be `1`.

---

# 21. Full production-style example

Let's say we have a Java application:

```text
Startup time = up to 90 seconds
Health endpoint = /actuator/health
Readiness = /actuator/health/readiness
```

We could use:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-api

spec:
  replicas: 3

  selector:
    matchLabels:
      app: payment-api

  template:
    metadata:
      labels:
        app: payment-api

    spec:
      containers:
        - name: payment-api
          image: payment-api:1.5.0

          ports:
            - containerPort: 8080

          startupProbe:
            httpGet:
              path: /actuator/health
              port: 8080
            periodSeconds: 5
            failureThreshold: 24

          livenessProbe:
            httpGet:
              path: /actuator/health
              port: 8080
            periodSeconds: 10
            timeoutSeconds: 2
            failureThreshold: 3

          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8080
            periodSeconds: 5
            timeoutSeconds: 2
            failureThreshold: 3
```

Now:

```text
Application starts
       |
       v
Startup probe
       |
       | up to ~120 sec
       |
       v
Startup succeeds
       |
       +------------------+
       |                  |
       v                  v
 Liveness             Readiness
       |                  |
       |                  |
   "Are you alive?"   "Can you take traffic?"
       |                  |
       v                  v
 Restart if           Remove from
 unhealthy            Service endpoints
```

---

# 22. Real scenario: database temporarily unavailable

Suppose:

```text
Payment API
    |
    v
Database ❌
```

Application itself is alive:

```text
Process = running
HTTP server = running
```

You might want:

```text
Liveness → SUCCESS
Readiness → FAILURE
```

Result:

```text
Pod
 |
 +-- Container stays running
 |
 +-- Readiness = false
 |
 +-- Service sends no new traffic
```

You **don't** want:

```text
DB temporarily unavailable
        |
        v
Liveness fails
        |
        v
Restart application
        |
        v
Application reconnects
        |
        v
Maybe makes the situation worse
```

This is why probe design matters.

---

# 23. Real scenario: application deadlock

Now:

```text
Application process = running
HTTP server = stuck
```

Readiness might fail:

```text
Readiness
   |
   v
Not Ready
```

But suppose the application never recovers.

Liveness should eventually fail:

```text
Liveness
   |
   v
Failure × 3
   |
   v
Container restart
```

So:

```text
Readiness = traffic control

Liveness = recovery
```

---

# 24. Real scenario: slow startup

Suppose:

```text
Application needs 2 minutes
```

Without startupProbe:

```text
Liveness
   |
   +-- fails
   +-- fails
   +-- fails
   |
   v
Restart
```

With startupProbe:

```text
             Startup Probe
                  |
          application starts
                  |
          +-------+-------+
          |               |
       still starting    success
          |               |
          v               v
       keep waiting    Liveness
                       Readiness
```

This prevents false restarts.

---

# 25. Probes and Service traffic

Suppose:

```text
Service
   |
   +--- Pod A Ready
   +--- Pod B Ready
   +--- Pod C NotReady
```

Traffic goes:

```text
Service
   |
   +---- Pod A
   |
   +---- Pod B
```

Not:

```text
Pod C
```

So readiness is critical during:

```text
Rolling deployments
Startup
Shutdown
Temporary dependency failures
Maintenance
```

---

# 26. Probes and rolling deployment

Suppose:

```text
Deployment
replicas: 3
```

Current:

```text
Pod A Ready
Pod B Ready
Pod C Ready
```

New version starts:

```text
Pod D
```

If Pod D isn't ready:

```text
Pod D = NotReady
```

The Service doesn't send normal traffic to it yet.

Once:

```text
Readiness = success
```

then:

```text
Pod D = Ready
```

and traffic can start flowing to it.

This is one reason **readiness probes are critical for zero/minimal-downtime deployments**.

---

# 27. Common mistakes

### Mistake 1

Using liveness to check dependencies:

```text
/health
   |
   +-- DB
   +-- Redis
   +-- Kafka
```

If Redis temporarily fails, liveness fails and the application gets restarted.

Often a better design is:

```text
Liveness
   ↓
Is application process/functionality alive?

Readiness
   ↓
Can application handle traffic?
```

The exact health design depends on the application.

---

### Mistake 2

Using only readiness.

Then a deadlocked application can remain running forever:

```text
Container = Running
Readiness = false
```

but never recover.

That's why liveness can be useful.

---

### Mistake 3

Using aggressive liveness settings.

For example:

```yaml
periodSeconds: 1
failureThreshold: 1
```

A temporary 1-second network/application hiccup could cause a restart.

Probe configuration should reflect the application's behavior.

---

# 28. Probe vs restartPolicy

Another important distinction:

```text
Probe
 |
 +-- determines health
```

while:

```text
restartPolicy
 |
 +-- determines whether an exited container is restarted
```

For liveness:

```text
Liveness fails
     |
     v
Kubernetes kills/restarts container
```

The restart behavior is then governed by the Pod/workload semantics.

Don't say:

> "Liveness probe itself is the restart policy."

It isn't.

---

# 29. Interview cheat sheet

```text
                 PROBES
                   |
       +-----------+-----------+
       |           |           |
       v           v           v
   Startup      Readiness   Liveness
       |           |           |
       v           v           v
 "Have you      "Can you     "Are you
  started?"      receive       alive?"
                  traffic?"
       |           |           |
       |           v           v
       |       Remove from   Restart
       |       endpoints     container
       |
       v
 Protect slow startup
```

### Probe mechanisms:

```text
HTTP
TCP
Exec
gRPC
```

### Important timing fields:

```text
initialDelaySeconds
periodSeconds
timeoutSeconds
failureThreshold
successThreshold
```

### The three sentences to memorize:

> **Startup probe protects slow-starting applications and prevents liveness/readiness probes from acting before startup succeeds.**

> **Readiness controls whether the Pod should receive Service traffic; a readiness failure does not restart the container.**

> **Liveness determines whether Kubernetes should restart an unhealthy container.**

And the production flow:

```text
                Container starts
                       |
                       v
                Startup Probe
                       |
                 SUCCESS
                       |
              +--------+--------+
              |                 |
              v                 v
          Liveness          Readiness
              |                 |
        unhealthy?          healthy?
              |                 |
              v                 v
           Restart         Service traffic
                            allowed
```

That **startup → readiness → liveness** relationship is one of the most useful ways to explain probes in a Kubernetes interview.
