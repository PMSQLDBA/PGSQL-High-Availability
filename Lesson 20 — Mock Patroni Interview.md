# Lesson 20 — Mock Patroni Interview — Detailed Practice

This final lesson combines everything from Lessons 1–19 into a realistic PostgreSQL/Patroni interview.

The goal is not to memorize commands. You should be able to explain:

```text
What happened?
Why did it happen?
How would you verify it?
How would you fix it safely?
How would you prevent it?
```

A strong Patroni engineer thinks in this order:

```text
Application
    |
PgBouncer / HAProxy
    |
PostgreSQL
    |
Patroni
    |
DCS / etcd
    |
Infrastructure
```

---

# Round 1 — Fundamentals

## Question 1: What is Patroni?

### Strong interview answer

> Patroni is a high-availability management framework for PostgreSQL. PostgreSQL itself performs the actual streaming replication, while Patroni manages cluster membership, leader election, automatic failover, planned switchover, PostgreSQL configuration and recovery of failed members. Patroni uses a distributed configuration store such as etcd for coordination.

### Important distinction

Don't say:

> Patroni replicates PostgreSQL data.

Instead:

```text
PostgreSQL
    |
    +-- WAL
    |
    +-- Streaming replication


Patroni
    |
    +-- Leader election
    +-- Failover
    +-- Switchover
    +-- Cluster management


etcd
    |
    +-- Distributed consensus
    +-- Leader lock
    +-- Cluster state
```

---

# Question 2: Why do we need Patroni if PostgreSQL already supports replication?

PostgreSQL provides replication, but it doesn't by itself provide the complete HA orchestration layer.

Suppose:

```text
pg01 PRIMARY
pg02 REPLICA
pg03 REPLICA
```

and `pg01` crashes.

PostgreSQL does not independently coordinate:

```text
Which replica should become primary?

Who is allowed to promote?

How do other members discover the new primary?

How do we avoid conflicting promotions?

How does the old primary safely rejoin?
```

Patroni provides that coordination.

A good answer:

> PostgreSQL provides the replication technology. Patroni adds automated HA management around it, including leader election, promotion, failover and cluster state coordination.

---

# Question 3: What is etcd's role?

A strong answer:

> etcd acts as Patroni's distributed configuration store. Patroni members use it to coordinate cluster state and determine which node currently owns leadership. Because etcd uses consensus, Patroni can make safer leadership decisions during failures and network partitions.

Architecture:

```text
             etcd cluster
        etcd01 etcd02 etcd03
               |
          leader state
               |
       +-------+-------+
       |       |       |
      pg01    pg02    pg03
```

Important:

```text
etcd does NOT contain PostgreSQL table data.
```

It contains coordination information.

---

# Question 4: What is quorum?

Suppose etcd has three members:

```text
etcd01
etcd02
etcd03
```

Quorum is:

```text
floor(N / 2) + 1
```

Therefore:

```text
3 nodes -> 2
5 nodes -> 3
7 nodes -> 4
```

If one member of a 3-member cluster fails:

```text
2 / 3 available
```

Quorum remains.

If two fail:

```text
1 / 3
```

No quorum.

### Interview trap

> Why not use two etcd servers?

Because:

```text
2-member etcd
quorum = 2
```

Lose one:

```text
1 / 2
```

No quorum.

Three members tolerate one member failure while retaining quorum.

---

# Round 2 — Failover

## Question 5: Explain automatic Patroni failover

Start:

```text
pg01 PRIMARY
pg02 REPLICA
pg03 REPLICA
```

Application:

```text
Application
    |
 HAProxy
    |
  pg01
```

Now `pg01` crashes.

Conceptually:

```text
pg01
  X
PostgreSQL unavailable
  |
Patroni can no longer maintain leadership
  |
DCS leader state changes/expires
  |
remaining Patroni members evaluate state
  |
eligible replica obtains leadership
  |
PostgreSQL promoted
```

Suppose `pg02` wins:

```text
pg01 DOWN

pg02 PRIMARY
pg03 REPLICA
```

HAProxy detects the new primary:

```text
Application
    |
 HAProxy
    |
  pg02
```

That is the complete failover path.

---

# Question 6: What happens to existing application connections?

They normally break.

Suppose:

```text
Application
     |
TCP connection
     |
   pg01
```

When `pg01` dies:

```text
TCP connection
      X
```

HAProxy cannot magically move that TCP session to `pg02`.

Instead:

```text
old connection fails
      |
application/pool retries
      |
new connection
      |
HAProxy
      |
pg02
```

This is why application retry behavior matters.

A database can fail over in 15 seconds while the application remains unavailable for 60 seconds because its connection pool retries badly.

---

# Question 7: What is the difference between switchover and failover?

### Switchover

Planned:

```text
pg01 PRIMARY
      |
planned maintenance
      |
      v
pg02 PRIMARY
```

Example:

```bash
patronictl switchover
```

Use for:

```text
OS maintenance
hardware maintenance
planned restart
role movement
```

### Failover

Failure-driven:

```text
pg01 PRIMARY
      |
      X
unexpected failure
      |
      v
pg02 promoted
```

Simple interview answer:

> Switchover is planned movement of the primary role between healthy nodes. Failover is promotion in response to primary failure or an emergency condition.

---

# Round 3 — Troubleshooting

## Question 8: Primary failed, but Patroni didn't fail over. What do you do?

This is a very important senior interview question.

Don't immediately say:

```bash
patronictl failover
```

Start with:

```bash
patronictl list
```

Determine:

```text
Is there currently a leader?

Are replicas running?

How far behind are they?

Is the DCS healthy?

Are they eligible for promotion?
```

Then check etcd.

For example:

```bash
etcdctl endpoint health --cluster
```

Then Patroni logs:

```bash
journalctl -u patroni
```

Look for:

```text
DCS errors
leader-lock errors
replication problems
timeline issues
promotion rejection
member tags
synchronous-mode restrictions
```

Then PostgreSQL state:

```sql
SELECT pg_is_in_recovery();
```

On replicas check WAL:

```sql
SELECT
    pg_last_wal_receive_lsn(),
    pg_last_wal_replay_lsn();
```

### Strong interview answer

> I wouldn't immediately force promotion. I'd first check `patronictl list`, verify DCS quorum, inspect Patroni logs and confirm that at least one replica is healthy, sufficiently current and eligible for promotion. I would determine why automatic failover didn't occur before forcing a manual failover, because forcing a new primary without understanding the old primary or DCS state can create a split-brain risk.

Excellent interview point:

```text
Diagnose first.
Force promotion second.
```

---

# Question 9: What is split brain?

Split brain means more than one database node believes it is primary and can accept writes.

```text
        NETWORK PARTITION

pg01                         pg02
PRIMARY                      PRIMARY

writes A                     writes B
```

Now PostgreSQL has two histories.

That's extremely dangerous.

Patroni uses DCS coordination and leader locking to reduce this risk.

Additional protection can include:

```text
watchdog
fencing
STONITH
network isolation
load-balancer controls
```

---

# Question 10: Why is fencing important?

Suppose:

```text
pg01 = old primary
pg02 = new primary
```

You must prevent:

```text
Application A -> pg01
Application B -> pg02
```

Fencing makes the old primary unable to continue serving writes.

A strong answer:

> Promotion alone isn't enough. During ambiguous failures I also need confidence that the old primary cannot continue accepting writes. Fencing, watchdog mechanisms and correct client routing help provide that protection.

---

# Round 4 — Old Primary Recovery

## Question 11: pg01 failed, pg02 became primary, then pg01 returned. What happens?

Before:

```text
pg01 PRIMARY
pg02 REPLICA
```

Failover:

```text
pg01 DOWN
pg02 PRIMARY
```

New writes happen on `pg02`.

Now `pg01` returns.

It cannot simply say:

```text
I'm back.
I'm primary again.
```

The histories may have diverged.

Patroni needs to make `pg01` follow the current primary.

Possible recovery:

```text
pg01
 |
pg_rewind
 |
 v
synchronize with pg02
 |
 v
pg01 REPLICA
```

If rewind cannot safely work:

```text
reinitialize pg01
```

---

# Question 12: Explain `pg_rewind`

Imagine:

```text
Timeline 1:

A -- B -- C -- D
               |
             failover

pg01: A-B-C-D-X-Y

pg02: A-B-C-D-E-F-G
```

The two nodes diverged after `D`.

`pg_rewind` can bring the old primary back toward the new authoritative history without requiring a complete base backup in suitable conditions.

After:

```text
pg02 PRIMARY
     |
     +------> pg01 REPLICA
```

A good interview answer:

> `pg_rewind` is useful when an old primary has diverged from the new primary after failover. It can synchronize the old data directory with the new timeline much faster than rebuilding the entire database, provided its prerequisites and required WAL/history are available.

---

# Question 13: What if `pg_rewind` fails?

Don't repeatedly force it.

Investigate:

```text
required WAL available?
permissions correct?
authentication correct?
timeline valid?
data checksums/wal_log_hints prerequisites?
source reachable?
```

If the node cannot safely be rewound:

```bash
patronictl reinit <cluster> <member>
```

may rebuild it.

Remember:

> `reinit` is destructive to that member's existing local PostgreSQL data.

---

# Round 5 — Replication

## Question 14: Explain asynchronous replication

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
    v
Replica receives WAL later
```

Advantage:

```text
lower commit latency
```

Risk:

If the primary dies after acknowledging a commit but before the replica receives the WAL:

```text
acknowledged transaction
        |
        X
possible loss
```

---

# Question 15: Explain synchronous replication

Conceptually:

```text
Application
    |
 COMMIT
    |
    v
Primary
    |
WAL
    |
    v
Synchronous standby
    |
acknowledgement
    |
    v
Primary completes commit
```

Benefit:

```text
stronger durability
```

Cost:

```text
network latency affects writes
standby health may affect writes
```

Strong interview answer:

> I would choose synchronous replication only when the business RPO justifies the additional latency and availability dependency.

---

# Question 16: How do you investigate replication lag?

On primary:

```sql
SELECT
    application_name,
    client_addr,
    state,
    sync_state,
    sent_lsn,
    write_lsn,
    flush_lsn,
    replay_lsn
FROM pg_stat_replication;
```

On replica:

```sql
SELECT
    pg_last_wal_receive_lsn(),
    pg_last_wal_replay_lsn();
```

Then determine where the bottleneck is.

### Case A

```text
WAL generated
   |
   X
not reaching replica
```

Check:

```text
network
wal sender/receiver
authentication
firewall
```

### Case B

```text
WAL reaches replica
   |
   v
replay slow
```

Check:

```text
disk latency
CPU
standby queries
I/O saturation
recovery conflicts
```

---

# Round 6 — WAL Problem

## Question 17: `pg_wal` is filling the disk. What do you do?

Never start with:

```bash
rm -rf $PGDATA/pg_wal/*
```

That can destroy the database.

Check:

```sql
SELECT
    slot_name,
    active,
    restart_lsn
FROM pg_replication_slots;
```

Then:

```text
WAL archiving working?
replica disconnected?
replication slot inactive?
backup process holding WAL?
WAL generation unusually high?
```

A classic situation:

```text
Replica offline
      |
replication slot remains
      |
PostgreSQL retains WAL
      |
pg_wal grows
      |
disk fills
```

Fix the root cause.

---

# Round 7 — HAProxy and PgBouncer

## Question 18: Why use HAProxy?

Patroni knows:

```text
pg02 is primary
```

but applications shouldn't have to know server names.

Use:

```text
Application
    |
    v
HAProxy :5000
    |
    v
Current Patroni primary
```

HAProxy checks Patroni's REST API and routes new connections to the appropriate PostgreSQL member.

---

# Question 19: Why use PgBouncer?

Suppose:

```text
5,000 application connections
```

Direct PostgreSQL:

```text
Application
    |
5000 sessions
    |
PostgreSQL
```

Instead:

```text
5000 application connections
       |
       v
   PgBouncer
       |
   300 DB connections
       |
       v
  PostgreSQL
```

PgBouncer solves:

```text
connection pooling
```

HAProxy solves:

```text
server routing
```

Very common interview question:

> PgBouncer and HAProxy are complementary; they solve different problems.

---

# Round 8 — DCS Failure

## Question 20: What happens if one etcd member fails?

Three-member cluster:

```text
etcd01 UP
etcd02 UP
etcd03 DOWN
```

You still have:

```text
2 / 3
```

Quorum exists.

The cluster is degraded but operational.

Repair `etcd03`.

---

# Question 21: What if two etcd members fail?

```text
etcd01 UP
etcd02 DOWN
etcd03 DOWN
```

Only:

```text
1 / 3
```

No quorum.

This is not the time to randomly start changing PostgreSQL roles.

Strong answer:

> DCS quorum loss is a serious HA-control-plane incident. I would determine whether the problem is actual node loss or network partition, restore quorum safely and avoid manual promotion until I understand the existing primary state.

---

# Question 22: PostgreSQL primary is healthy but etcd quorum is lost. What do you do?

This tests distributed-systems understanding.

Don't say:

> Restart Patroni.

Ask:

```text
Why did etcd lose quorum?
```

Check:

```text
etcd processes
network connectivity
disk/fsync latency
certificates
CPU/memory
firewall
Raft health
```

The priority is restoring the **DCS control plane** safely.

---

# Round 9 — Backup & DR

## Question 23: Why isn't a replica a backup?

Suppose someone runs:

```sql
DROP TABLE customers;
```

Streaming replication does:

```text
Primary
DROP TABLE customers
      |
      v
Replica
DROP TABLE customers
```

Replication faithfully reproduces the mistake.

Backup/PITR gives:

```text
Restore backup
      |
replay WAL
      |
stop before DROP
```

So:

```text
Replication = availability

Backup = recoverability
```

You need both.

---

# Question 24: Explain PITR

Suppose:

```text
10:00 backup

10:00-14:00 WAL archived

13:47 accidental DELETE
```

You can:

```text
Restore base backup
      |
replay archived WAL
      |
stop at 13:46:59
```

That's Point-in-Time Recovery.

---

# Question 25: Entire Patroni cluster is destroyed. How do you recover?

A strong recovery model:

```text
Backup repository
       |
restore
       |
       v
one PostgreSQL node
       |
PITR / WAL recovery
       |
validate
       |
       v
authoritative primary
       |
       +------> rebuild replica 1
       |
       +------> rebuild replica 2
```

Don't independently restore three nodes and promote all of them.

Establish one authoritative history first.

---

# Round 10 — Security

## Question 26: How would you secure Patroni?

A strong answer covers layers:

```text
PostgreSQL
    |
SCRAM
TLS
pg_hba least privilege

Patroni
    |
REST authentication
TLS
network restrictions

etcd
    |
TLS/mTLS
authentication/RBAC
private network

Linux
    |
service accounts
file permissions
SSH controls

Backup
    |
encryption
separate credentials
separate failure domain
```

Mention secrets too:

```text
No passwords in Git.
```

---

# Round 11 — Major Upgrade

## Question 27: Can I upgrade PostgreSQL 17 to 18 one replica at a time?

Not as a normal physical-replication rolling upgrade.

Major PostgreSQL versions cross a storage/protocol compatibility boundary.

Typical approaches:

```text
pg_upgrade
```

or:

```text
logical replication / blue-green
```

For an in-place Patroni upgrade:

```text
maintenance
   |
upgrade primary
   |
establish PG18 cluster
   |
rebuild PG18 replicas
```

For very low downtime:

```text
PG17 Patroni cluster
       |
logical replication
       |
PG18 Patroni cluster
       |
cutover
```

---

# Question 28: When would you choose blue-green?

Requirements such as:

```text
Database = large
Downtime < 1 minute
```

may favor:

```text
OLD PG17 cluster
      |
logical replication
      |
NEW PG18 cluster
```

Advantages:

```text
old production remains intact during build
long testing window
shorter final cutover
```

But you must handle:

```text
sequences
DDL
roles
extensions
permissions
cutover
rollback after new writes
```

---

# Round 12 — Performance

## Question 29: Patroni cluster is slow. What do you check?

Don't immediately tune Patroni.

Check:

```text
queries
CPU
memory
storage latency
connections
locks
WAL
checkpoints
autovacuum
network
replication
```

Patroni is usually not the query-performance bottleneck.

Use:

```text
pg_stat_statements
```

for expensive SQL.

Use PostgreSQL statistics and OS monitoring to identify the actual resource bottleneck.

---

# Question 30: Why not set `max_connections=5000`?

Because PostgreSQL uses a process-based backend model and large numbers of active connections consume significant resources.

Better architecture for many workloads:

```text
5000 application connections
        |
    PgBouncer
        |
200-500 PostgreSQL connections
```

The exact number must come from workload testing.

---

# Round 13 — Architecture Challenge

## Question 31: Design a production Patroni architecture

Requirements:

```text
Database:       5 TB
Peak TPS:       20k
RPO:            near zero locally
RTO:            < 60 seconds
Locations:      3 AZ
DR:             another region
Application:    24x7
```

I would propose something like:

```text
                     APPLICATIONS
                          |
                          v
                   Redundant LB/VIP
                          |
             +------------+------------+
             |                         |
        PgBouncer01                PgBouncer02
             |                         |
        HAProxy01                  HAProxy02
             \                         /
              +-----------+-----------+
                          |
        +-----------------+-----------------+
        |                 |                 |
        v                 v                 v

      AZ-A              AZ-B              AZ-C

      pg01              pg02              pg03
    Patroni           Patroni           Patroni
 PostgreSQL         PostgreSQL        PostgreSQL

   PRIMARY          SYNC standby      ASYNC standby


      etcd01            etcd02            etcd03
        \                 |                 /
         +----------------+----------------+
                          |
                       quorum


                   Backup repository
                          |
                   independent storage
                          |
                  cross-region copy


                    DR REGION
                        |
                 DR PostgreSQL
```

---

# Question 32: Why three AZs?

Because it maps well to majority-based consensus.

```text
AZ-A = etcd01
AZ-B = etcd02
AZ-C = etcd03
```

Lose AZ-A:

```text
etcd02 + etcd03
```

still provide:

```text
2 / 3 quorum
```

You also still have two PostgreSQL nodes if database members are distributed similarly.

---

# Question 33: Why not synchronous replication to the DR region?

Because WAN latency could become part of transaction commit latency.

Example:

```text
Primary region
      |
      | 60 ms RTT
      |
DR region
```

If every commit waits on that remote standby, application latency and availability may suffer.

A common design is:

```text
Local synchronous replication
        +
Remote asynchronous DR
```

subject to the actual business RPO.

---

# Round 14 — Production Incident

## Question 34: Here's your incident

At 2:00 AM:

```text
Application alerts:
DATABASE CONNECTION FAILED

patronictl list:

+ Cluster: prod --------+
| pg01 | Replica | stopped |
| pg02 | Leader  | running |
| pg03 | Replica | running |
```

What do you do?

Notice:

```text
Patroni already failed over.
```

So don't promote anything.

`pg02` is leader.

Check:

```sql
SELECT pg_is_in_recovery();
```

on `pg02`.

Expected:

```text
false
```

Then test the application path:

```text
Application
    |
DNS/LB
    |
PgBouncer
    |
HAProxy
    |
pg02
```

Check HAProxy's Patroni health check.

Then application retry/connection pool behavior.

### Strong answer

> Patroni has already restored database leadership, so I would not perform another failover. I would verify pg02 is writable and then troubleshoot the client path—HAProxy health checks, PgBouncer, DNS/load balancer and application reconnection. At the same time I'd investigate pg01 separately and restore it as a replica after service is stable.

That's a very good production answer.

---

# Question 35: pg01 comes back but cannot rejoin

Logs show:

```text
requested WAL segment has already been removed
```

What do you do?

Determine whether:

```text
pg_rewind
```

can recover it using available WAL/history.

If not:

```text
reinitialize pg01
```

from the current cluster.

For a very large database, also consider whether your backup tooling provides a faster supported replica creation method.

---

# Round 15 — Dangerous Scenario

## Question 36: Two nodes say they are primary

```text
pg01:
pg_is_in_recovery() = false

pg02:
pg_is_in_recovery() = false
```

What do you do?

Treat this as potential:

```text
SPLIT BRAIN
```

Priority:

```text
STOP WRITE DIVERGENCE
```

Then establish:

```text
Which node owns valid Patroni leadership?

What does DCS say?

What are the PostgreSQL timelines?

Which node received writes?

Is there a network partition?

Where is application traffic going?
```

Do not randomly run:

```text
promote
reinit
pg_rewind
```

until you establish the authoritative history.

A senior-level answer:

> I would first isolate the non-authoritative writer to stop further divergence. Then I would use DCS leadership, Patroni state, PostgreSQL timelines and transaction/WAL evidence to determine the authoritative history. Only after that would I rewind or rebuild the losing node.

---

# Round 16 — Maintenance

## Question 37: You need to reboot the current primary

Current:

```text
pg01 PRIMARY
pg02 REPLICA
pg03 REPLICA
```

First verify:

```bash
patronictl list
```

Ensure replicas are healthy.

Perform planned switchover:

```text
pg01 PRIMARY
     |
switchover
     |
     v
pg02 PRIMARY
```

Verify application.

Then:

```text
stop/reboot/patch pg01
```

Bring it back:

```text
pg01 REPLICA
pg02 PRIMARY
pg03 REPLICA
```

That is a normal HA maintenance workflow.

---

# Round 17 — Capacity Question

## Question 38: Why should replicas have similar capacity to primary?

Because tomorrow:

```text
Replica
   |
failover
   |
   v
Primary
```

If:

```text
pg01 = 32 CPU / 128 GB
pg02 = 4 CPU / 16 GB
```

promotion may technically succeed while the application collapses under load.

HA means:

```text
surviving the workload after failure
```

not simply:

```text
successfully promoting PostgreSQL
```

---

# Round 18 — Monitoring

## Question 39: What would you alert on?

At minimum:

```text
Patroni
-------
No leader
Unexpected leader change
Member missing
Pending restart

Replication
-----------
Replica disconnected
High lag
WAL receiver stopped

PostgreSQL
----------
Connection saturation
Long transactions
Lock contention
Autovacuum problems

Infrastructure
--------------
CPU
Memory
Disk capacity
Disk latency
Network

etcd
----
Quorum
Leader changes
Request latency
Disk/fsync latency

HAProxy/PgBouncer
-----------------
No primary backend
Connection failures
Pool saturation

Backup
------
Backup failure
WAL archive failure
Restore-test failure

Security
--------
Certificate expiry
Authentication failures
```

---

# Round 19 — RPO/RTO

## Question 40: Business asks for RPO=0 and RTO=30 seconds. What do you say?

Don't immediately promise it.

A strong architecture answer:

> I would first define exactly which failures the RPO=0 requirement applies to. For local infrastructure failures, I would evaluate synchronous replication to a standby in another low-latency failure domain. For RTO 30 seconds, I would test Patroni failure detection, promotion, routing and application reconnect end-to-end. Cross-region RPO=0 is a separate architecture decision because synchronous WAN replication may introduce significant latency and availability trade-offs.

That's much stronger than:

> “Patroni supports automatic failover, so yes.”

---

# Final Interview Challenge

Now imagine the interviewer gives you this:

> “We have a 10 TB PostgreSQL database running on one server. It supports a critical payment application. We want high availability with Patroni. We can tolerate less than one minute of downtime, almost no transaction loss, and we have three availability zones. Design it.”

Your answer can be:

> I would first confirm the exact RPO and RTO and measure the current workload, particularly TPS, WAL generation, storage latency and connection count. For HA, I would deploy three PostgreSQL/Patroni members across the three availability zones, with enough CPU, memory and storage capacity for any eligible replica to become primary.
>
> I would use a three-member etcd cluster distributed across the same independent failure domains so the DCS retains quorum after losing one zone. Because the requirement is almost no transaction loss, I would evaluate a synchronous standby in another low-latency AZ, while keeping another replica available for additional HA capacity.
>
> Applications would not connect directly to a PostgreSQL hostname. I would provide a redundant connection layer using HAProxy or an equivalent service, with PgBouncer where connection pooling is required. The application must also have tested reconnect and retry behavior because existing database sessions will break during failover.
>
> Patroni would manage leader election and promotion, while PostgreSQL streaming replication would move the WAL. I would use fencing or watchdog protection appropriate to the platform to reduce split-brain risk.
>
> HA alone isn't enough, so I would also implement physical backups, continuous WAL archiving and PITR to an independent repository, plus a separate cross-region DR strategy if regional failure is in scope.
>
> Finally, before production I would test primary failure, VM failure, AZ failure, DCS member failure, network partition, HAProxy failure, old-primary rejoin, `pg_rewind`, replica rebuild, backup restore and DR recovery. I would measure the actual end-to-end RPO and RTO rather than assuming the design meets the SLA.

That is the type of answer expected from someone who understands the **architecture**, not just Patroni commands.

# Final Cheat Sheet

Before an interview, remember this flow:

```text
               APPLICATION
                    |
                    v
              PgBouncer
          Connection pooling
                    |
                    v
                HAProxy
               Routing
                    |
                    v
             PostgreSQL
          Data + WAL + SQL
                    |
                    v
               Patroni
           HA orchestration
                    |
                    v
                 etcd
        Consensus + leader state
```

And remember these distinctions:

| Question                    | Answer                |
| --------------------------- | --------------------- |
| Who replicates data?        | PostgreSQL            |
| Who manages HA?             | Patroni               |
| Who coordinates leadership? | Patroni + DCS         |
| What provides consensus?    | etcd                  |
| Who routes applications?    | HAProxy/LB            |
| Who pools connections?      | PgBouncer             |
| Planned role change?        | Switchover            |
| Unplanned role change?      | Failover              |
| Old primary recovery?       | `pg_rewind` / rebuild |
| Data-loss protection?       | Replication design    |
| Historical recovery?        | Backup + WAL/PITR     |
| Cross-region protection?    | DR architecture       |
| Two active writers?         | Potential split brain |

And for troubleshooting, memorize the **order**, not just commands:

```text
1. Observe
      |
      v
2. Identify current leader
      |
      v
3. Check DCS/quorum
      |
      v
4. Check PostgreSQL roles
      |
      v
5. Check replication/timelines
      |
      v
6. Check Patroni logs
      |
      v
7. Check routing/application
      |
      v
8. Decide safest recovery action
```

The most important production rule from the entire course is:

> **Never force a failover, promote a node, reinitialize a member, delete WAL, or modify DCS state until you understand the current cluster state and know which PostgreSQL history is authoritative.**

That completes **Lesson 20 and the Patroni training series.**
