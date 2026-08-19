# SPI_SLO_SLA_SLI_RPO_RTO

First, one correction:

> **SPI** is not as universally standardized as SLI/SLO/SLA. In DevOps/SRE discussions, people may use **SPI = Service Performance Indicator** (or sometimes Security/Service Performance Indicator depending on the organization). I'll explain the common **Service Performance Indicator** meaning.

The easiest way to understand the whole set is:

```text
                 CUSTOMER / BUSINESS
                        |
                        v
                  +-----------+
                  |    SLA    |  ← What we promise
                  +-----------+
                        |
                        v
                  +-----------+
                  |    SLO    |  ← Our internal target
                  +-----------+
                        |
                        v
                  +-----------+
                  |    SLI    |  ← What we measure
                  +-----------+
                        |
                        v
                  +-----------+
                  |    SPI    |  ← Service health/performance
                  +-----------+

      Disaster / Data Recovery
              |
        +-----+-----+
        |           |
        v           v
       RPO         RTO
    How much     How fast
    data can     we recover
    we lose
```

---

# 1. SLI — Service Level Indicator

### SLI = What you measure

An **SLI is a quantitative measurement of service behavior**.

For example:

```text
Availability
Latency
Error rate
Throughput
Durability
```

Suppose you run:

```text
payment-api
```

You want to measure availability.

You could define:

```text
SLI = successful requests / total valid requests
```

Suppose:

```text
Total requests = 1,000,000

Successful = 999,500
```

Then:

```text
SLI = 999,500 / 1,000,000

    = 99.95%
```

That **99.95% is your SLI measurement**.

---

# 2. SLI example — latency

Suppose your API has:

```text
1 million requests
```

You define:

> Percentage of requests completing within 300 ms.

Suppose:

```text
950,000 requests
```

completed within 300 ms.

Then:

```text
SLI = 950,000 / 1,000,000

    = 95%
```

So:

```text
Latency SLI = 95%
```

---

# 3. SLI example — error rate

Suppose:

```text
Total requests = 100,000
5xx requests   = 200
```

Error rate:

```text
200 / 100,000
=
0.2%
```

You can define your SLI as:

```text
Successful request rate = 99.8%
```

or:

```text
Error rate = 0.2%
```

Both can represent the same underlying measurement, depending on how you've defined the indicator.

---

# 4. SLO — Service Level Objective

Now we say:

> **What level of performance do we want?**

That's the SLO.

Suppose your SLI is:

```text
Availability = 99.95%
```

You define:

```text
SLO = 99.9%
```

So:

```text
SLI = actual measurement

SLO = desired target
```

---

# 5. Simple example

Suppose:

```text
SLI = 99.97%
SLO = 99.90%
```

Then:

```text
99.97 > 99.90
```

You're meeting the objective.

If:

```text
SLI = 99.70%
SLO = 99.90%
```

then:

```text
99.70 < 99.90
```

You've violated the SLO.

---

# 6. SLI vs SLO

This is an extremely common interview question.

### SLI

> **What actually happened?**

Example:

```text
Availability = 99.93%
```

### SLO

> **What level did we want?**

Example:

```text
Availability target = 99.90%
```

So:

```text
SLI = measurement
SLO = target
```

---

# 7. SLA — Service Level Agreement

Now we go outside the engineering team.

SLA is:

> **A formal commitment/agreement with customers or another party.**

For example, your company provides:

```text
Payment API
```

Contract says:

> Monthly availability will be at least 99.9%.

That's an SLA.

```text
Customer
    |
    | contract
    v
Company
    |
    v
SLA = 99.9% availability
```

---

# 8. SLA vs SLO

This is another important distinction.

Suppose:

```text
SLA = 99.9%
SLO = 99.95%
```

Why make the SLO stricter?

Because you don't want to operate exactly at the contractual limit.

```text
                    100%
                     |
                     |
                  SLO 99.95%
                     |
                     |
                  SLA 99.90%
                     |
                     |
                     0%
```

You keep an internal buffer.

---

# 9. Why SLO is usually stricter than SLA

Suppose:

```text
SLA = 99.9%
```

If your service is running at:

```text
99.91%
```

technically you met the SLA.

But that's dangerously close to the boundary.

So engineering might define:

```text
SLO = 99.95%
```

Now you get an early warning.

```text
SLI
 |
 | 99.99%  ← excellent
 |
 | 99.96%
 |
 | 99.95%  ← SLO
 |
 | 99.93%  ← warning
 |
 | 99.90%  ← SLA
 |
 +----------------
```

---

# 10. SLA can have business consequences

Suppose:

```text
SLA = 99.9%
```

Contract says:

> If availability falls below 99.9%, customer receives a service credit.

Then:

```text
99.95%
   ↓
SLA satisfied
   ↓
No credit
```

But:

```text
99.5%
   ↓
SLA violated
   ↓
Potential service credit / contractual consequence
```

The exact consequences depend on the contract.

---

# 11. Availability example

Let's calculate monthly availability.

Suppose:

```text
30-day month
```

Total minutes:

```text
30 × 24 × 60

= 43,200 minutes
```

For:

```text
99.9% availability
```

allowed downtime:

```text
43,200 × 0.1%

= 43.2 minutes
```

So approximately:

```text
99.9%
=
43.2 minutes downtime/month
```

---

# 12. 99.99% is very different

```text
43,200 × 0.01%

= 4.32 minutes
```

So:

| Availability | Approx. monthly downtime |
| ------------ | -----------------------: |
| 99%          |                   7h 12m |
| 99.9%        |                  43m 12s |
| 99.95%       |                  21m 36s |
| 99.99%       |                   4m 19s |
| 99.999%      |                      26s |

This is why every additional "9" becomes expensive.

---

# 13. SPI — Service Performance Indicator

Now SPI.

Unlike SLI/SLO/SLA, **SPI isn't as consistently standardized across SRE organizations**.

A common usage is:

> **SPI = Service Performance Indicator — a metric representing how the service is performing operationally.**

Examples:

```text
CPU utilization
Memory utilization
Request rate
Latency
Error rate
Queue depth
Pod restart count
Database connections
Kafka consumer lag
```

For example:

```text
payment-api

SPI:
p95 latency = 180 ms
error rate = 0.1%
throughput = 2,000 req/sec
```

These tell you about the health/performance of the service.

---

# 14. SPI vs SLI

The distinction depends on how your organization defines SPI.

A practical way to explain it:

```text
SPI
 ↓
Broad service-performance metrics

SLI
 ↓
Specifically selected metric used to evaluate
an SLO
```

Example:

```text
Service metrics:

CPU
Memory
Requests/sec
p50 latency
p95 latency
p99 latency
5xx
DB connections
Queue depth
```

You might choose:

```text
SLI:
percentage of requests < 300ms
```

and then:

```text
SLO:
99% of requests < 300ms
```

So:

```text
SPI → operational/performance measurement
SLI → reliability measurement used against an SLO
```

If an interviewer uses SPI differently in their company, ask what definition they use.

---

# 15. Now RPO

RPO is completely different from SLI/SLO/SLA.

RPO = **Recovery Point Objective**

It answers:

> **"How much data can we afford to lose?"**

Imagine:

```text
Database
   |
   v
Backups every 1 hour
```

If the database crashes at:

```text
2:47 PM
```

and the latest backup is:

```text
2:00 PM
```

you could potentially lose:

```text
47 minutes of data
```

Therefore your effective RPO is around:

```text
1 hour
```

---

# 16. RPO example

Suppose a bank says:

```text
RPO = 5 minutes
```

Meaning:

> In a disaster, we cannot afford to lose more than approximately 5 minutes of committed data.

Architecture might be:

```text
Primary DB
    |
    | replication
    v
Standby DB

Replication lag
< 5 minutes
```

Or frequent backups/log shipping.

---

# 17. RPO = data loss tolerance

Remember:

```text
RPO
=
How much data can I lose?
```

Examples:

```text
RPO = 24 hours
```

Potentially acceptable for:

```text
Internal reporting
```

But:

```text
RPO = 0 / near-zero
```

might be required for:

```text
Financial transactions
```

depending on business requirements.

---

# 18. RTO

RTO = **Recovery Time Objective**

It answers:

> **"How quickly must the service be restored?"**

Suppose:

```text
RTO = 30 minutes
```

A disaster occurs:

```text
10:00 AM
```

The service should be restored by approximately:

```text
10:30 AM
```

---

# 19. RTO vs RPO

This is one of the easiest interview questions if you remember:

```text
RPO → DATA
RTO → TIME
```

### RPO

```text
How much data can I lose?
```

### RTO

```text
How long can the service be unavailable?
```

---

# 20. Disaster scenario

Suppose your production database crashes at:

```text
2:00 PM
```

Business requirements:

```text
RPO = 5 minutes
RTO = 30 minutes
```

Meaning:

```text
                 Disaster
                    |
                  2:00
                    |
       +------------+------------+
       |                         |
       v                         v
      RPO                       RTO
    5 minutes                  30 minutes
       |                         |
       v                         v
Data loss <= 5m           Service restored <= 30m
```

---

# 21. Real AWS example

Imagine:

```text
AWS
 |
 +-- EKS
 |
 +-- RDS PostgreSQL
 |
 +-- S3
```

Business requirements:

```text
RPO = 5 minutes
RTO = 30 minutes
```

You might design:

```text
             Production
                 |
                 v
             RDS Primary
                 |
                 | replication
                 v
            Standby / DR
```

For Kubernetes:

```text
Cluster A
   |
   | GitOps
   |
   v
Cluster B / DR
```

And backups:

```text
RDS
 |
 v
Automated backups
 |
 v
S3 / backup storage
```

The architecture depends on the exact recovery requirements.

---

# 22. Scenario: EKS cluster disaster

Suppose:

```text
Production EKS
```

becomes unavailable.

Requirements:

```text
RTO = 1 hour
RPO = 15 minutes
```

Your recovery architecture might be:

```text
                Git
                 |
          +------+------+
          |             |
          v             v
       EKS-A          EKS-B
      Primary           DR
          |             |
          +------+------+
                 |
              Database
                 |
            replication
```

If EKS-A fails:

```text
EKS-A
  X
  |
  v
EKS-B
  |
  v
Restore workloads
```

You need to ensure the entire recovery process fits within the RTO.

---

# 23. What determines RTO?

RTO isn't just:

> "How fast can Kubernetes start?"

It includes the whole recovery process.

For example:

```text
Detect failure             5 min
Provision/activate DR      10 min
Restore DB                  5 min
Deploy workloads           10 min
DNS/traffic switch          5 min
Validation                  5 min
--------------------------------
Total                      40 min
```

If:

```text
RTO = 30 min
```

your architecture fails the requirement.

---

# 24. What determines RPO?

RPO depends heavily on:

```text
Backup frequency
Replication
Replication lag
Transaction log shipping
Cross-region replication
Database architecture
```

Example:

```text
Backup every 24h
```

gives poor RPO.

While:

```text
Synchronous replication
```

can provide near-zero data loss, although the exact achievable RPO depends on architecture and failure mode.

---

# 25. Now combine all six

Let's build a real **payment platform**.

```text
                         PAYMENT PLATFORM
                                |
              +-----------------+----------------+
              |                                  |
              v                                  v
         Reliability                       Disaster Recovery
              |                                  |
       +------+------+                      +----+----+
       |      |      |                      |         |
      SLI    SLO    SLA                    RPO       RTO
       |      |      |                      |         |
       v      v      v                      v         v
    Measure  Target Contract             Data      Recovery
```

---

# 26. Example requirements

### SLI

```text
99.96% successful requests
```

### SLO

```text
99.95% successful requests
```

### SLA

```text
99.90% availability
```

### SPI

```text
p95 latency = 180 ms
p99 latency = 450 ms
5xx = 0.1%
throughput = 2,500 req/sec
```

### RPO

```text
5 minutes
```

### RTO

```text
30 minutes
```

---

# 27. Now imagine an incident

At:

```text
10:00 AM
```

a database issue causes API failures.

Metrics:

```text
SLI:
99.95% → 98.2%
```

SLO:

```text
99.95%
```

Therefore:

```text
SLI < SLO
```

SLO breached.

---

# 28. SLA question

Suppose the monthly SLA is:

```text
99.9%
```

If this incident causes enough downtime that monthly availability becomes:

```text
99.7%
```

then:

```text
99.7% < 99.9%
```

SLA breached.

That could have contractual consequences.

---

# 29. SPI during the incident

Your operational metrics may show:

```text
p95 latency
200ms → 2,000ms

5xx
0.1% → 15%

DB connections
500 → 2,000

CPU
60% → 95%
```

These are useful **service-performance indicators** that help diagnose what is happening.

---

# 30. RPO during the incident

Suppose:

```text
Last replicated transaction:
9:57 AM

Failure:
10:00 AM
```

Potential data loss:

```text
3 minutes
```

If:

```text
RPO = 5 minutes
```

then:

```text
3 < 5
```

RPO requirement is satisfied.

---

# 31. RTO during the incident

Suppose:

```text
Failure:
10:00

Service restored:
10:22
```

Then:

```text
Recovery time = 22 minutes
```

If:

```text
RTO = 30 minutes
```

then:

```text
22 < 30
```

RTO satisfied.

---

# 32. The complete incident picture

```text
                    INCIDENT
                        |
          +-------------+-------------+
          |             |             |
          v             v             v
         SLI           SPI           DR
          |             |             |
          v             v        +----+----+
       Compare        Diagnose    |         |
       with SLO                    RPO       RTO
          |                         |         |
          v                         v         v
      SLO breach                Data loss   Recovery
                                    |        time
                                    |         |
                                    v         v
                                  <=5m      <=30m
```

---

# 33. Error budget — very important related concept

Once you understand SLO, you should know **Error Budget**.

If:

```text
SLO = 99.9%
```

then allowed failure is:

```text
100 - 99.9
=
0.1%
```

That:

```text
0.1%
```

is your error budget.

For a 30-day month:

```text
43.2 minutes
```

of unavailability.

---

# 34. Why error budget matters to DevOps

Suppose your team has:

```text
SLO = 99.9%
```

and you've already consumed:

```text
40 minutes
```

of your:

```text
43.2-minute
```

budget.

Then someone proposes:

> "Let's deploy this risky infrastructure change tonight."

You might say:

> "We're almost out of error budget, so let's stabilize production before taking additional release risk."

This is where SRE principles connect reliability to engineering velocity.

---

# 35. Error budget diagram

```text
SLO = 99.9%
       |
       v
Error Budget = 0.1%
       |
       v
43.2 min/month
       |
       +------ incidents
       |
       +------ failed deployments
       |
       +------ outages
       |
       v
Budget consumed
```

---

# 36. SLI → SLO → SLA relationship

Think:

```text
                    CUSTOMER
                       |
                       v
                     SLA
                "We promise 99.9%"
                       |
                       v
                     SLO
              "We target 99.95%"
                       |
                       v
                     SLI
              "We're currently 99.97%"
```

So:

```text
SLI = measurement
SLO = target
SLA = commitment
```

---

# 37. RPO/RTO relationship

Think:

```text
             DISASTER
                 |
       +---------+---------+
       |                   |
       v                   v
      RPO                 RTO
       |                   |
       v                   v
How much data?       How much time?
can we lose?         can we be down?
```

---

# 38. Very common interview trap

### Question:

> "Our application has 99.99% SLO. Does that mean we have an RTO of 4 minutes?"

**No.**

They're completely different concepts.

```text
99.99% SLO
    ↓
Service reliability target

RTO = 30 minutes
    ↓
Disaster recovery target
```

A service could have:

```text
99.99% availability
```

and:

```text
RTO = 2 hours
```

if its normal availability is excellent but disaster recovery takes two hours.

---

# 39. Another interview trap

> "If our RPO is 1 hour, our RTO is also 1 hour."

No.

You could have:

```text
RPO = 1 hour
RTO = 15 minutes
```

Meaning:

```text
We can lose up to 1 hour of data,
but we must restore the service within 15 minutes.
```

---

# 40. Senior DevOps scenario

Interviewer:

> **"Your payment API has an SLO of 99.95%. Yesterday it had 99.8% availability. What do you do?"**

A good answer:

> "First I'd confirm the SLI calculation and the measurement window, then identify the source of the availability degradation using metrics, logs and traces. Since 99.8% is below the 99.95% SLO, we've consumed more error budget than planned. I'd determine whether this was an application, infrastructure, dependency or deployment issue, mitigate the incident, and then review whether our SLO, alerting and error-budget policy need adjustment."

---

# 41. Another scenario

> **"The business says RPO must be 5 minutes and RTO must be 30 minutes. How do you design the platform?"**

Answer:

> "For a 5-minute RPO, I'd need a backup or replication mechanism that limits potential data loss to five minutes or less. For the 30-minute RTO, I'd design and test the recovery path—including infrastructure, database recovery, application deployment, secrets, DNS or traffic switching, and validation—to complete within 30 minutes. I wouldn't claim the requirement is satisfied until we've tested the actual recovery process."

That last sentence is important.

**RTO is a tested capability, not just a Terraform variable.**

---

# 42. Final cheat sheet

| Term    | Meaning                       | Question                       |
| ------- | ----------------------------- | ------------------------------ |
| **SLI** | Service Level Indicator       | What are we measuring?         |
| **SLO** | Service Level Objective       | What target do we want?        |
| **SLA** | Service Level Agreement       | What did we promise?           |
| **SPI** | Service Performance Indicator | How is the service performing? |
| **RPO** | Recovery Point Objective      | How much data can we lose?     |
| **RTO** | Recovery Time Objective       | How quickly must we recover?   |

### Memorize this:

```text
SLI → MEASURE
SLO → TARGET
SLA → PROMISE

SPI → PERFORMANCE

RPO → DATA LOSS
RTO → RECOVERY TIME
```

And the relationship:

```text
                 RELIABILITY
                     |
          +----------+----------+
          |                     |
          v                     v
       SLI/SLO/SLA             DR
          |                     |
          |                +----+----+
          |                |         |
          v                v         v
      Reliability         RPO       RTO
      measurement         Data      Time
```

For a **Senior DevOps/Platform interview**, also know **error budget, burn rate, availability calculations, multi-region DR, backup vs replication, and how you actually measure SLOs with Prometheus/Grafana/Dynatrace**—those are the natural follow-up questions.
