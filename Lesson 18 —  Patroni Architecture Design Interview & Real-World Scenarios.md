# Lesson 18 — Patroni Architecture Design Interview & Real-World Scenarios

This lesson is about explaining **why** you chose an architecture, not just drawing three PostgreSQL servers and saying “HA”.

A strong architecture answer usually follows this order:

```text
1. Understand requirements
2. Define RPO and RTO
3. Choose PostgreSQL topology
4. Choose Patroni/DCS design
5. Design client connectivity
6. Design backups and DR
7. Handle failure domains
8. Cover security and monitoring
9. Explain failover behavior
10. Explain trade-offs
```

## Scenario 1 — 5 TB database, 20k TPS, 60-second RTO

Requirements:

```text
Database size       5 TB
Peak workload       20,000 TPS
RTO                 < 60 seconds
RPO                 very low
Availability        production critical
Maintenance         minimal downtime
```

A reasonable starting architecture:

```text
                     Applications
                          |
                          v
                    Load Balancer
                          |
                 +--------+--------+
                 |                 |
             HAProxy01         HAProxy02
                 \                 /
                  \               /
                   +------+------+
                          |
        +-----------------+-----------------+
        |                 |                 |
        v                 v                 v

      pg01              pg02              pg03
   PostgreSQL         PostgreSQL         PostgreSQL
    Patroni            Patroni            Patroni

    PRIMARY           REPLICA            REPLICA

        \                 |                 /
         \                |                /
          +---------------+---------------+
                          |
                     etcd cluster

                 etcd01 etcd02 etcd03
```

For 20k TPS, I would focus heavily on:

```text
low-latency storage
WAL throughput
network bandwidth
connection pooling
replica replay capacity
```

Patroni itself is unlikely to be the bottleneck.

---

## How I would explain this design

I would say:

> I would use three PostgreSQL/Patroni members so that after losing the primary, I still have a promoted node and another standby available. I would place the members across independent failure domains where possible. Patroni would coordinate HA through a three-member etcd cluster, and applications would connect through redundant HAProxy or an equivalent load-balancing service.

That is already stronger than:

> “Three nodes because Patroni needs three nodes.”

Patroni doesn't require three PostgreSQL servers. **The architecture requirement does.**

---

# Scenario 2 — Near-zero RPO

Now the business says:

```text
We cannot lose committed financial transactions.
```

That changes the architecture.

Asynchronous replication may allow:

```text
Application
    |
 COMMIT
    |
    v
Primary
    |
success returned
    |
    X Primary fails before WAL reaches replica
```

Potential result:

```text
acknowledged transaction lost
```

So I would investigate synchronous replication.

Conceptually:

```text
Application
    |
  COMMIT
    |
    v
  pg01
 PRIMARY
    |
    | WAL
    v
  pg02
SYNC REPLICA
    |
 acknowledgement
    |
    v
  pg01
    |
    v
COMMIT completes
```

But don't stop there.

Near-zero RPO introduces a trade-off:

```text
Stronger durability
       |
       v
More dependency on standby/network health
       |
       v
Potentially higher write latency
```

So the correct interview answer is not:

> “Enable synchronous replication.”

It is:

> “I would validate whether the business really requires zero acknowledged transaction loss, then evaluate Patroni synchronous mode with at least one suitable synchronous standby. I would test the impact of network latency and standby loss on write availability before choosing the final policy.”

That demonstrates architecture thinking.

---

# Scenario 3 — Two availability zones

Suppose infrastructure gives you only:

```text
AZ-A
AZ-B
```

You have three PostgreSQL nodes.

Possible layout:

```text
AZ-A                    AZ-B

pg01                    pg02
pg03
```

Problem:

If AZ-A dies:

```text
pg01 X
pg03 X

pg02 survives
```

You still have one PostgreSQL node, but the failure took two members.

If etcd is arranged similarly:

```text
AZ-A                    AZ-B

etcd01                  etcd03
etcd02
```

losing AZ-A means:

```text
1 / 3 etcd members remain
```

No quorum.

This is why **two-zone consensus architecture is awkward**.

A third failure domain is much cleaner:

```text
AZ-A       AZ-B       AZ-C

pg01       pg02       pg03

etcd01     etcd02     etcd03
```

Then losing any one AZ leaves:

```text
2 / 3 etcd
```

and two PostgreSQL nodes.

---

# Scenario 4 — Only two data centers available

This is a good interview question.

Suppose:

```text
DC1
DC2
```

and someone proposes:

```text
DC1             DC2

etcd01          etcd02
```

That's bad because 2-node etcd quorum is:

```text
2 / 2
```

Losing either data center loses quorum.

A better option could involve a third small quorum site:

```text
DC1             DC2             Witness/3rd site

etcd01          etcd02          etcd03
```

The third site doesn't necessarily need to host application PostgreSQL data.

Its purpose is to create an independent failure domain for consensus.

That is a key distributed-systems design principle.

---

# Scenario 5 — Cross-region Disaster Recovery

Requirements:

```text
Primary Region: Chicago
DR Region: Virginia

RPO: < 5 minutes
RTO: < 30 minutes
```

I would **not automatically stretch the primary Patroni cluster across regions**.

Instead, a common architecture is:

```text
             PRIMARY REGION

        Patroni PostgreSQL cluster
       pg01 / pg02 / pg03
              |
              |
      asynchronous replication
              |
              v
          DR REGION

        DR PostgreSQL replica
              +
        backup repository copy
```

Why?

Because cross-region latency can make consensus and synchronous replication more complex.

I would prefer:

```text
Local Patroni HA
       +
Separate DR strategy
```

over:

```text
One giant Patroni cluster
stretched across long-distance regions
```

unless there is a very strong reason and it has been carefully tested.

---

# Scenario 6 — Region completely lost

Suppose Chicago disappears.

You have:

```text
Chicago Patroni cluster = unavailable
Virginia DR replica     = available
```

DR runbook:

```text
1. Confirm primary region really unavailable

2. Stop any chance of split writes

3. Validate DR replica state

4. Determine RPO / last replayed transaction

5. Promote DR PostgreSQL

6. Establish new Patroni/HA control plane if required

7. Redirect applications

8. Validate business services

9. Rebuild lost HA capacity
```

The key question is:

> How do you prevent the old region from coming back and accepting writes?

That is a **fencing** problem.

Good DR design always thinks about:

```text
promotion
+
fencing
```

not promotion alone.

---

# Scenario 7 — Maintenance without downtime

Requirement:

```text
Patch pg01 OS
No application outage
```

Current state:

```text
pg01 PRIMARY
pg02 REPLICA
pg03 REPLICA
```

Workflow:

```text
Check cluster health
      |
      v
Switchover pg01 -> pg02
      |
      v
Verify pg02 primary
      |
      v
Patch/reboot pg01
      |
      v
Verify pg01 rejoins replica
```

Then:

```text
patch pg03
```

if needed.

The key design feature allowing this is:

```text
roles can move independently of server identity
```

That is one of the main operational benefits of Patroni.

---

# Scenario 8 — One replica is used for reporting

Architecture:

```text
pg01 PRIMARY

pg02 REPLICA
HA failover candidate

pg03 REPLICA
heavy reporting workload
```

Would you automatically allow `pg03` to become primary?

Maybe not.

Reporting may consume:

```text
CPU
memory
I/O
long queries
```

You could mark it appropriately:

```yaml
tags:
  nofailover: true
```

Then:

```text
pg02 = preferred HA candidate
pg03 = reporting-only replica
```

But now consider:

```text
pg02 fails
```

Before the primary fails.

You effectively have:

```text
pg01 primary
pg03 nofailover
```

and no automatic candidate.

So tagging decisions need monitoring and operational awareness.

---

# Scenario 9 — 10 TB database, replica rebuild takes 12 hours

Requirements:

```text
Database: 10 TB
Replica rebuild: 12 hours
```

If your topology is:

```text
pg01 PRIMARY
pg02 REPLICA
pg03 REPLICA
```

and `pg03` dies:

```text
pg01 PRIMARY
pg02 REPLICA
pg03 DOWN
```

you are degraded for 12 hours.

If `pg01` then fails during that period:

```text
pg02 PRIMARY
```

Now you have one database node.

That may be unacceptable.

Possible architecture improvements:

```text
additional replica
faster replica provisioning
pgBackRest-based restore
storage snapshots
higher network bandwidth
```

This is why **replica rebuild time is an availability metric**.

---

# Scenario 10 — Application has 8,000 connections

Don't simply configure:

```text
max_connections = 10000
```

Instead think:

```text
8,000 application connections
          |
          v
      PgBouncer
          |
          v
300–500 PostgreSQL connections
```

Exact pool sizes need workload testing.

Architecture:

```text
Applications
     |
     v
PgBouncer
     |
     v
HAProxy
     |
     v
Patroni primary
```

Now ask another question:

> Where does PgBouncer run?

Possible approaches include:

```text
application hosts
dedicated connection layer
same routing tier
```

Each has operational trade-offs.

---

# Scenario 11 — HAProxy itself must not fail

If you deploy:

```text
Application
    |
HAProxy01
    |
Patroni
```

you have introduced:

```text
single point of failure
```

A production solution may be:

```text
                 Application
                      |
                 Virtual IP
                      |
          +-----------+-----------+
          |                       |
      HAProxy01               HAProxy02
          \                       /
           +----------+----------+
                      |
                 Patroni cluster
```

Or use your cloud/provider load-balancing layer.

The architecture requirement is:

```text
stable redundant connection endpoint
```

not necessarily Keepalived specifically.

---

# Scenario 12 — `etcd` on PostgreSQL servers?

Interview question:

> Would you put etcd on the same three servers as PostgreSQL?

Good answer:

> For a small environment, colocating etcd and PostgreSQL can simplify operations and may be acceptable. For critical or resource-intensive production systems, I would evaluate separate etcd nodes because PostgreSQL CPU, storage, or network failures could otherwise affect the DCS at the same time.

Example problem:

```text
pg01 storage overloaded
       |
       +--> PostgreSQL affected
       |
       +--> etcd on same disk affected
```

You've coupled two important failure domains.

---

# Scenario 13 — Which replica should Patroni promote?

Current:

```text
pg01 PRIMARY

pg02 REPLICA
lag = 10 MB

pg03 REPLICA
lag = 5 GB
```

Primary dies.

Prefer:

```text
pg02
```

because it has more current WAL.

But lag is not the only factor.

Potential candidate considerations include:

```text
timeline compatibility
replication health
tags
member health
synchronous state
failover policy
```

You don't want to blindly say:

> Lowest server number gets promoted.

Patroni makes candidate decisions from cluster state and configured policies.

---

# Scenario 14 — Database is healthy but business is down

Cluster:

```text
pg01 Leader
pg02 Replica
pg03 Replica
```

Everything green.

But application cannot connect.

Architecture troubleshooting:

```text
Application
   |
DNS?
   |
Load Balancer?
   |
HAProxy?
   |
PgBouncer?
   |
PostgreSQL?
```

The answer isn't always Patroni.

Good architecture reviews explicitly include the **client path**.

---

# Scenario 15 — Near-zero downtime PostgreSQL upgrade

Requirements:

```text
PG17 -> PG18

Downtime:
< 1 minute
```

I would likely prefer:

```text
BLUE
PG17 Patroni cluster
      |
logical replication
      |
GREEN
PG18 Patroni cluster
```

Then:

```text
Stop writes to BLUE
      |
wait for lag zero
      |
sync sequences/final state
      |
switch endpoint to GREEN
      |
enable writes
```

This is more complex than `pg_upgrade`, but it fits the downtime requirement better.

Architecture should follow the SLA.

---

# Scenario 16 — What happens if etcd goes down but PostgreSQL stays up?

This is a classic interview question.

You should not simply say:

> PostgreSQL stops immediately.

The exact behavior depends on Patroni state, leader-lock timing, watchdog configuration, DCS availability and network conditions.

The architecture goal is:

> A node should not continue indefinitely as authoritative primary if it cannot safely maintain leadership in the DCS.

That's why:

```text
ttl
loop_wait
retry_timeout
watchdog
```

matter.

And it's why you need DCS failure testing before production.

---

# Scenario 17 — Primary and DCS become partitioned

Imagine:

```text
          NETWORK PARTITION

pg01 PRIMARY             pg02 + pg03
    |                         |
    X                         |
    |                         v
  etcd?                   DCS quorum
```

This is dangerous because `pg01` may still be alive.

Your architecture must prevent:

```text
pg01 continues writes
       +
pg02 promoted
```

from becoming two active histories.

This is where:

```text
leader lock
DCS quorum
TTL
watchdog/fencing
routing
```

all work together.

---

# Scenario 18 — Backup repository is in same storage array

Suppose:

```text
PostgreSQL cluster
        |
        +
Backup repository
        |
Same SAN
```

Then SAN fails.

Result:

```text
production lost
backup lost
```

A strong architecture answer would say:

> I would place backups in a separate failure domain and, depending on business requirements, maintain offsite or cross-region copies and immutable retention.

HA and backup independence matter.

---

# Scenario 19 — How many PostgreSQL replicas?

There is no magic answer.

Ask:

```text
How many failures must we tolerate?

Do we need read scaling?

Do we need reporting replicas?

How long does rebuild take?

Do we need local + DR copies?
```

Example:

```text
PRIMARY
REPLICA 1 = HA candidate
REPLICA 2 = HA candidate
REPLICA 3 = reporting
REPLICA 4 = DR
```

That's very different from:

```text
PRIMARY + one replica
```

Even though both use Patroni.

---

# Scenario 20 — Design for a critical financial system

Let's put everything together.

Requirements:

```text
5 TB PostgreSQL

20k TPS

RPO:
near zero for local infrastructure failure

RTO:
< 60 seconds

3 availability zones

Cross-region DR:
RPO < 5 minutes
RTO < 30 minutes

24x7 application

No single network proxy
```

A possible architecture:

```text
                      APPLICATIONS
                           |
                           v
                    Redundant LB/VIP
                           |
                 +---------+---------+
                 |                   |
             PgBouncer01         PgBouncer02
                 |                   |
             HAProxy01           HAProxy02
                 \                   /
                  +--------+--------+
                           |
       +-------------------+-------------------+
       |                   |                   |
       v                   v                   v

 AZ-A pg01             AZ-B pg02            AZ-C pg03
 PostgreSQL            PostgreSQL           PostgreSQL
 Patroni               Patroni              Patroni

 PRIMARY               SYNC REPLICA         ASYNC REPLICA

       \                   |                   /
        \                  |                  /
         +-----------------+-----------------+
                           |
                      etcd cluster

             AZ-A etcd01
             AZ-B etcd02
             AZ-C etcd03


                       Backup
                          |
                          v
                Independent repository
                          |
                          v
                 Cross-region copy


             Async DR replication
                       |
                       v
                  DR Region
               PostgreSQL replica
```

Why:

**Three AZs** protect against one AZ loss while preserving DCS quorum.

**Three PostgreSQL nodes** leave another standby after one promotion.

**One synchronous replica** can improve local durability guarantees.

**One asynchronous replica** provides flexibility if synchronous infrastructure becomes constrained.

**Three-member etcd** gives quorum with one member lost.

**Two HAProxy/PgBouncer nodes** avoid a connection-layer SPOF.

**Independent backups** protect against logical corruption and full-cluster loss.

**Separate cross-region DR** avoids forcing long-distance latency into the local HA path.

---

# Interview question — “Why not synchronous replication to DR?”

A strong response:

> Because cross-region round-trip latency would become part of the commit path and could severely increase transaction latency or reduce write availability during network problems. I would normally satisfy local near-zero-RPO requirements using a low-latency synchronous standby and use asynchronous replication plus continuous WAL archiving for cross-region DR, unless the business explicitly accepts the latency/availability trade-off.

That's architectural reasoning.

---

# Interview question — “What is the biggest Patroni risk?”

There's no single answer, but a good response is:

> The biggest risk is treating Patroni as the entire HA solution. Patroni only manages PostgreSQL leadership. DCS quorum, networking, client routing, application reconnect behavior, storage, backups, monitoring and operational procedures all need their own HA design.

That's an important principle.

---

# Interview question — “How would you validate this before production?”

I would say:

```text
Test primary process failure

Test primary VM failure

Test replica failure

Test one etcd member failure

Test DCS quorum loss

Test network partition

Test HAProxy failure

Test application reconnect

Test old-primary rejoin

Test pg_rewind

Test replica rebuild

Test disk pressure

Test backup restore

Test PITR

Test complete DR failover

Measure actual RPO/RTO
```

Not just:

```text
patronictl list looks good
```

---

# Architecture interview framework

When someone gives you a requirement, use this simple framework.

### Availability

Ask:

```text
How many failures must survive?
```

### Durability

Ask:

```text
How much committed data can be lost?
```

### Recovery

Ask:

```text
How quickly must service return?
```

### Scale

Ask:

```text
DB size?
TPS?
connections?
WAL rate?
growth?
```

### Geography

Ask:

```text
One AZ?
Multiple AZ?
Multiple regions?
```

### Operations

Ask:

```text
Maintenance downtime?
Upgrade downtime?
Replica rebuild time?
```

### Recovery

Ask:

```text
Backup?
PITR?
DR?
Restore testing?
```

Then design Patroni around those answers.

## A good architecture answer sounds like this

If someone asks:

> “Design PostgreSQL HA using Patroni.”

A strong concise answer is:

> I would first establish the required RPO, RTO, workload and failure domains. For a critical system I would typically use three PostgreSQL/Patroni members across three independent zones, backed by a three-member etcd cluster also distributed across those zones. Applications would connect through a redundant load-balancing/connection-pooling layer rather than directly to individual PostgreSQL servers. I would choose asynchronous or synchronous replication based on the required RPO and measured network latency. Backups and archived WAL would live in a separate failure domain, with cross-region DR if required. I would monitor Patroni leadership, replication lag, etcd quorum, routing, WAL, storage and backups, and validate the entire architecture with actual failure and recovery testing.

That demonstrates that you understand **the full system**, not only Patroni commands.

### Next — Lesson 19: Patroni Interview Questions & Answers

The next lesson can be a focused interview pack with around **40 Patroni/PostgreSQL HA questions**, ranging from beginner to senior level, including architecture, failover, etcd quorum, replication, `pg_rewind`, HAProxy, troubleshooting, backup/DR and scenario questions—with short answers you can practice verbally.
