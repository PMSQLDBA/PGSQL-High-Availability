# Lesson 9 — Production-Grade Patroni Architecture

Our lab works, but production HA needs more than just three PostgreSQL nodes and one HAProxy server.

A practical architecture might look like this:

```text
                         APPLICATIONS
                              |
                         DNS / VIP / LB
                              |
                  +-----------+-----------+
                  |                       |
                  v                       v
              HAProxy01               HAProxy02
                :5000                   :5000
                  \                       /
                   \                     /
                    +---------+---------+
                              |
             +----------------+----------------+
             |                |                |
             v                v                v
           pg01             pg02             pg03
        PostgreSQL       PostgreSQL       PostgreSQL
         Patroni          Patroni          Patroni

         PRIMARY          REPLICA          REPLICA
             \                |                /
              \               |               /
               +--------------+--------------+
                              |
                        DCS / etcd

                 etcd01     etcd02     etcd03
```

The important idea is that **every critical layer needs its own HA strategy**.

---

## 1. Separate the architecture into layers

I would think about Patroni production architecture as six layers:

| Layer                 | Technology/example     | Responsibility                |
| --------------------- | ---------------------- | ----------------------------- |
| Database              | PostgreSQL             | Data and replication          |
| HA controller         | Patroni                | Promotion, demotion, failover |
| Consensus             | etcd                   | Cluster coordination          |
| Connection routing    | HAProxy/LB             | Find current primary          |
| Connection management | PgBouncer              | Pool database connections     |
| DR                    | pgBackRest/WAL archive | Restore after serious failure |

A common mistake is thinking:

> "We installed Patroni, therefore PostgreSQL is highly available."

Not quite.

Patroni solves an important part of HA, but if your only HAProxy dies, applications still lose connectivity.

---

# 2. PostgreSQL topology

For many environments, start with:

```text
pg01   PRIMARY
pg02   REPLICA
pg03   REPLICA
```

Three PostgreSQL nodes provide useful flexibility.

If `pg01` fails:

```text
pg02 -> can become PRIMARY
pg03 -> remains another REPLICA
```

Compare that with only two database nodes:

```text
pg01 PRIMARY
pg02 REPLICA
```

After failover:

```text
pg01 DOWN
pg02 PRIMARY
```

Now you have:

```text
ONE database node remaining
```

Your service may still be available, but you have **no healthy standby** until you repair or rebuild `pg01`.

Three nodes give you more operational breathing room.

---

# 3. Put nodes in different failure domains

This is much more important than simply having three servers.

Bad:

```text
Physical Host A
 |
 +-- pg01
 +-- pg02
 +-- pg03
```

If Host A dies:

```text
ALL THREE DATABASE NODES DIE
```

That's not meaningful HA.

Better:

```text
Host A        Host B        Host C
  |             |             |
pg01          pg02          pg03
```

Cloud example:

```text
Availability Zone A
      pg01

Availability Zone B
      pg02

Availability Zone C
      pg03
```

Try to avoid sharing the same:

```text
hypervisor
rack
power supply
network switch
storage system
availability zone
```

when the infrastructure allows it.

The real principle is:

> **Don't let one infrastructure failure remove several HA members at once.**

---

# 4. etcd production architecture

Our lab ran:

```text
pg01 + etcd1
pg02 + etcd2
pg03 + etcd3
```

That's fine for learning.

For more critical production environments, I prefer thinking about etcd as its own infrastructure component:

```text
PostgreSQL / Patroni

pg01
pg02
pg03


DCS

etcd01
etcd02
etcd03
```

A three-member etcd cluster tolerates one member failure because quorum is 2/3.

A five-member etcd cluster requires quorum of 3 and can tolerate two member failures. etcd recommends odd-sized clusters such as 3, 5 or 7; larger clusters increase failure tolerance but also increase consensus overhead. ([etcd][1])

For most normal deployments:

```text
3 etcd nodes
```

is a sensible starting point.

---

# 5. Don't create a two-node etcd cluster

This is an architectural trap.

```text
etcd01
etcd02
```

Quorum is:

```text
2 / 2
```

Therefore:

```text
one fails
    |
    v
1 / 2
    |
NO QUORUM
```

You've paid for two machines but gained effectively no member-failure tolerance.

etcd explicitly recommends production clusters larger than two members and requires a majority to process cluster membership changes. ([etcd][2])

Use:

```text
3
5
7
```

rather than:

```text
2
4
6
```

for normal consensus-cluster design.

---

# 6. etcd disk performance matters

This is easy to overlook.

etcd isn't simply a small configuration file stored somewhere.

Raft consensus requires durable writes.

Slow storage can cause:

```text
slow etcd writes
      |
      v
slow heartbeats
      |
      v
leader instability
      |
      v
Patroni DCS problems
```

etcd recommends SSD storage for production and notes that disk latency can affect cluster stability. ([etcd][3])

So don't deploy etcd on unreliable or heavily overloaded storage just because its database is small.

---

# 7. Back up etcd too

PostgreSQL backups are more important because they contain your business data.

But you should also understand how to snapshot and recover the DCS.

etcd supports:

```text
snapshot
restore
cluster recreation
```

for disaster recovery. ([etcd][4])

This becomes important if you suffer:

```text
multiple etcd failures
corrupt DCS
bad membership changes
major infrastructure loss
```

But remember:

> An etcd backup is **not a substitute for PostgreSQL backups**.

---

# 8. Make HAProxy highly available

Our previous architecture had:

```text
Application
    |
    v
HAProxy01
    |
    v
Patroni cluster
```

Problem:

```text
HAProxy01
    X

Application cannot connect
```

even if PostgreSQL is perfectly healthy.

Use at least two routing nodes:

```text
                    APPLICATION
                         |
                         v
                  Stable endpoint
                         |
                 +-------+-------+
                 |               |
                 v               v
             HAProxy01       HAProxy02
                 \               /
                  \             /
                   +-----------+
                         |
                    PostgreSQL
```

---

# 9. Add a VIP when appropriate

In a traditional VM/on-prem environment you may see:

```text
HAProxy01          HAProxy02
     \                /
      \              /
        Keepalived
            |
            v
       Virtual IP
       10.10.10.50
```

Application connects to:

```text
10.10.10.50:5000
```

The VIP initially lives on:

```text
HAProxy01
```

If `HAProxy01` dies:

```text
HAProxy01
    X

VIP
 |
 v
HAProxy02
```

So the application connection string doesn't change.

---

# 10. Cloud environments may work differently

Don't automatically introduce Keepalived everywhere.

In AWS, Azure, GCP, Kubernetes or managed networking environments, you may use:

```text
Cloud Load Balancer
Network Load Balancer
internal service endpoint
DNS-based discovery
Kubernetes Service
```

instead.

Architecture should fit the infrastructure.

The goal isn't:

> "Use Keepalived."

The goal is:

> **Don't create a single routing failure point.**

---

# 11. Where PgBouncer fits

HAProxy and PgBouncer solve different problems.

HAProxy:

```text
Which PostgreSQL server
should this connection reach?
```

PgBouncer:

```text
How efficiently should
PostgreSQL connections be pooled?
```

Example:

```text
Application
     |
     v
PgBouncer
     |
     v
HAProxy
     |
     v
Current Primary
```

or sometimes architectures place components differently depending on pooling and failover requirements.

PgBouncer is especially useful when applications create large numbers of connections.

Instead of:

```text
5,000 application connections

        |
        v

5,000 PostgreSQL backends
```

you might have:

```text
5,000 client connections
        |
        v
    PgBouncer
        |
      200
        |
        v
   PostgreSQL
```

The exact numbers depend on workload; don't copy those as sizing recommendations.

---

# 12. Patroni REST API security

Remember port:

```text
8008
```

We've been using:

```bash
curl http://pg01:8008/
```

In production, don't assume that endpoint should be accessible from everywhere.

Patroni's REST interface isn't just useful for health checks. It can also participate in administrative operations through Patroni tooling.

Patroni's security documentation explicitly calls out both the **DCS and REST API as interfaces that need protection**. ([Patroni][5])

Use:

```text
network segmentation
firewall rules
TLS where appropriate
authentication for management operations
restricted management networks
```

---

# 13. Protect etcd

Port `2379` should definitely not be casually exposed.

Think:

```text
Application network
     |
     X
     |
   etcd
```

Only systems that actually need DCS access should reach it.

For example:

```text
Patroni ------+
              |
patronictl ---+----> etcd
              |
etcd peers ---+
```

Patroni recommends protecting the DCS and generally recommends enabling TLS for DCS communication. ([Patroni][5])

Production might therefore use:

```text
Patroni
  |
 TLS
  |
  v
etcd:2379
```

with certificate-based authentication.

---

# 14. PostgreSQL connections should also be secured

Our lab used:

```text
host all all 192.168.10.0/24 scram-sha-256
```

That's intentionally simple.

Production should be much more restrictive.

For example:

```text
Application subnet
       |
       v
specific database
specific application role
```

Avoid rules equivalent to:

```text
everyone
from everywhere
to every database
```

unless there is a very specific reason.

And generally use:

```text
SCRAM authentication
TLS
least-privilege roles
network firewall controls
```

---

# 15. Don't store cleartext passwords carelessly

Our lab YAML contained:

```yaml
password: LabReplication123
```

Fine for learning.

Not good enough for sensitive production systems.

Treat:

```text
PostgreSQL superuser password
replication password
REST API credentials
DCS credentials
TLS private keys
backup repository credentials
```

as secrets.

How you manage them depends on the environment:

```text
Vault
cloud secret managers
configuration management secrets
protected environment/config files
Kubernetes Secrets
```

The key principle is:

> Keep secrets out of source control and restrict filesystem access.

---

# 16. Patroni watchdog

This is an advanced but important protection.

Imagine the current leader becomes isolated from the DCS.

You don't want an isolated PostgreSQL primary indefinitely continuing to accept writes while another side of the network legitimately elects a new leader.

Patroni supports watchdog integration as an extra split-brain safety mechanism.

If the leader fails to renew access to the DCS within the required timeframe, the watchdog can force the machine to reset.

Patroni's current documentation describes this as an additional fail-safe and notes that when watchdog mode is required, Patroni won't promote a node if it cannot activate the watchdog. ([Patroni][6])

Conceptually:

```text
Primary
   |
   X
Cannot reach DCS
   |
   v
Cannot safely prove leadership
   |
   v
Watchdog timeout
   |
   v
Server reset
```

This deserves careful testing before enabling it in production.

A watchdog that is incorrectly configured can reboot servers—which is exactly what it's designed to do.

---

# 17. Synchronous vs asynchronous replication

This becomes a business requirement, not simply a PostgreSQL setting.

### Asynchronous

```text
Application
    |
  COMMIT
    |
    v
PRIMARY
    |
SUCCESS sent to app
    |
    +---- WAL ----> REPLICA
```

Advantage:

```text
lower write latency
```

Risk:

```text
possible acknowledged transaction loss
during certain primary failures
```

### Synchronous

```text
Application
    |
  COMMIT
    |
    v
PRIMARY
    |
    +------ WAL ------> SYNC STANDBY
                           |
                           v
                      acknowledgement
                           |
                           v
                        PRIMARY
                           |
                           v
                   transaction completes
```

Advantage:

```text
stronger durability guarantees
```

Tradeoff:

```text
higher latency
standby health can affect writes
```

Patroni supports synchronous operation and maintains synchronous-standby state through the DCS. ([Patroni][7])

---

# 18. Don't automatically choose synchronous replication

Ask the business first:

```text
How much data can we lose?
```

That's RPO.

For example:

```text
RPO = 5 minutes

vs

RPO = near zero
```

Those are very different requirements.

Also ask:

```text
How much downtime is acceptable?
```

That's RTO.

Architecture should flow from:

```text
business requirements
       |
       +--> RPO
       |
       +--> RTO
       |
       v
technical design
```

not the other way around.

---

# 19. Patroni is not a backup solution

This is probably the most important production lesson.

Suppose somebody runs:

```sql
DROP TABLE customers;
```

PostgreSQL happily writes that to WAL.

Replication then does this:

```text
PRIMARY
DROP TABLE customers
        |
        | WAL
        v
REPLICA 1
DROP TABLE customers
        |
        v
REPLICA 2
DROP TABLE customers
```

Congratulations:

```text
all three copies are broken
```

Replication protects against:

```text
server failure
```

It doesn't automatically protect against:

```text
user mistakes
bad application changes
logical corruption
malicious deletes
ransomware
site-wide disaster
```

You still need backups.

---

# 20. Production backup architecture

A strong setup may use something like **pgBackRest** or another supported PostgreSQL backup system.

Conceptually:

```text
                    PostgreSQL
                        |
                WAL archiving
                        |
                        v
              Backup Repository
               /              \
              /                \
        Full backups        WAL archive
```

You want the ability to do:

```text
Full restore

Point-in-Time Recovery

Restore to another server

Restore after entire cluster loss
```

A backup isn't proven because:

```text
backup job = successful
```

It's proven because:

```text
restore test = successful
```

---

# 21. Keep backups outside the cluster failure domain

Bad:

```text
pg01
pg02
pg03
backups

ALL ON SAME STORAGE
```

Storage dies:

```text
everything gone
```

Better:

```text
PostgreSQL cluster
      |
      v
Separate backup storage
      |
      +--> preferably another failure domain
```

Depending on requirements, also consider:

```text
object storage
offsite copy
immutable backup
cross-region copy
```

---

# 22. Monitor PostgreSQL

At minimum monitor:

```text
database availability
CPU
memory
disk
filesystem usage
connections
long transactions
locks
deadlocks
WAL generation
checkpoint activity
replication lag
replication slots
database growth
```

Especially:

```text
disk usage
```

Remember Lesson 2:

```text
dead replica
    +
replication slot
    |
    v
WAL retained
    |
    v
disk fills
```

That can become a major incident surprisingly quickly.

---

# 23. Monitor Patroni

Monitor:

```text
Current leader

Replica count

Node state

Patroni REST API availability

Failover events

Timeline changes

Pending restart

Replication lag
```

Don't discover:

```text
pg02 died three weeks ago
```

only when:

```text
pg01 dies today
```

HA environments often fail because the first failure went unnoticed and the second failure caused the outage.

---

# 24. Monitor etcd

Monitor:

```text
member health
quorum
leader changes
disk latency
database size
network latency
peer communication
fsync duration
```

etcd's responsiveness matters because Patroni's control plane relies on the DCS.

Remember:

```text
PostgreSQL healthy
        +
Patroni healthy
        +
DCS unhealthy
        =
HA behavior potentially compromised
```

---

# 25. Monitor HAProxy

Monitor:

```text
HAProxy process availability

Primary backend status

Replica backend status

connection count

connection errors

health-check failures
```

The most alarming scenario is:

```text
PRIMARY backend:

pg01 DOWN
pg02 DOWN
pg03 DOWN
```

That means HAProxy can't identify an eligible primary.

Don't immediately blame HAProxy.

Possible causes include:

```text
no Patroni leader
network problem
Patroni REST failure
incorrect health check
PostgreSQL outage
DCS problem
```

---

# 26. Production architecture I would start with

For a normal important system:

```text
                    Applications
                         |
                  Load Balancer/VIP
                         |
               +---------+---------+
               |                   |
            HAProxy01           HAProxy02
               |                   |
               +---------+---------+
                         |
              PostgreSQL Endpoint
                         |
       +-----------------+-----------------+
       |                 |                 |
       v                 v                 v

     pg01              pg02              pg03
  PostgreSQL        PostgreSQL        PostgreSQL
   Patroni           Patroni           Patroni

  Primary           Replica           Replica

       \                 |                 /
        \                |                /
         +---------------+---------------+
                         |
                     etcd/DCS

       +-----------------+-----------------+
       |                 |                 |
     etcd01            etcd02            etcd03


                Backup Repository
                      ^
                      |
                 pgBackRest
                      |
                WAL + Backups
```

Then place those components intelligently across failure domains.

---

# 27. What I would avoid

I wouldn't consider this production-grade:

```text
Server01
 |
 +-- PostgreSQL primary
 +-- Patroni
 +-- etcd
 +-- HAProxy
 +-- PgBouncer
 +-- monitoring
 +-- backup repository
```

It's technically many software components.

Architecturally it's:

```text
ONE BIG SINGLE POINT OF FAILURE
```

Likewise:

```text
VM1 pg01
VM2 pg02
VM3 pg03
```

isn't enough information.

I'd ask:

```text
Which hypervisors?

Which racks?

Which storage?

Which network?

Which data center/AZ?

Where is etcd?

Where is HAProxy?

Where are backups?
```

That's architecture thinking.

---

# 28. Failure scenarios you must test

Before calling a Patroni platform production-ready, I would test at least:

| Failure                          | Expected result                                            |
| -------------------------------- | ---------------------------------------------------------- |
| PostgreSQL process dies          | Patroni handles/restarts or failover occurs as appropriate |
| Primary VM dies                  | Replica promoted                                           |
| Replica dies                     | Primary continues                                          |
| One etcd member dies             | DCS continues with quorum                                  |
| Primary loses DCS/network access | Safe leadership behavior                                   |
| HAProxy01 dies                   | HAProxy02 continues                                        |
| Old primary returns              | Rejoins as replica                                         |
| Replica cannot rewind            | Reinitialize/recover                                       |
| Disk fills                       | Alert before service loss                                  |
| Backup restore                   | Database recoverable                                       |
| Application connection dies      | App reconnects                                             |
| Entire PostgreSQL cluster lost   | Restore procedure works                                    |

The architecture diagram isn't complete until these scenarios have been demonstrated.

---

# 29. The full mental model

At this stage, you should think:

```text
                 APPLICATION
                      |
                      v
              CONNECTION LAYER
          HAProxy / PgBouncer / LB
                      |
                      v
                POSTGRESQL
                      |
              WAL REPLICATION
                      |
             +--------+--------+
             |                 |
          Replica           Replica

                      ^
                      |
                   PATRONI
                      |
                      v
                  DCS / etcd
                      |
                     quorum


                 BACKUPS
                    |
                    v
             Disaster Recovery


                MONITORING
                    |
       watches every layer above
```

And remember the responsibility boundaries:

**PostgreSQL** stores and replicates data.

**Patroni** manages database HA.

**etcd** provides safe distributed coordination.

**HAProxy** finds the appropriate PostgreSQL node.

**PgBouncer** manages connection pooling.

**Backup tooling** protects against data loss and cluster-wide disasters.

**Monitoring** tells you when any of those assumptions stop being true.

## Next — Lesson 10: Patroni Production Troubleshooting

This is probably the most useful lesson for actual support work.

We'll take real incident-style scenarios such as **`patronictl list` shows no leader, replica lag is growing, PostgreSQL is running but Patroni says stopped, etcd has lost quorum, failover didn't happen, old primary won't rejoin, `pg_rewind` fails, replication slot fills the disk, HAProxy shows every backend down, and the cluster has two nodes that appear to be primary**.

For each one, we'll go through **what to check first, commands to run, how to interpret the output, and what actions are dangerous**.

[1]: https://etcd.io/docs/v2.3/admin_guide/ "Administration | etcd"
[2]: https://etcd.io/docs/v3.4/op-guide/runtime-configuration/ "Runtime reconfiguration | etcd"
[3]: https://etcd.io/docs/v3.2/faq/ "Frequently Asked Questions (FAQ) | etcd"
[4]: https://etcd.io/docs/v3.7/op-guide/recovery/ "Disaster recovery | etcd"
[5]: https://patroni.readthedocs.io/en/rel_3_3/security.html "Security Considerations — Patroni 3.3.11 documentation"
[6]: https://patroni.readthedocs.io/_/downloads/en/latest/pdf/?utm_source=chatgpt.com "Patroni Documentation"
[7]: https://patroni.readthedocs.io/_/downloads/en/master/pdf/?utm_source=chatgpt.com "Patroni Documentation"
