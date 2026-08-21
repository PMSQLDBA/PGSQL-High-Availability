# Lesson 13 — Patroni Monitoring & Alerting

A production Patroni cluster is only useful if you know when it is becoming unhealthy **before** users report an outage.

The monitoring goal is simple:

```text
Detect degradation early
        |
        v
Understand which layer is failing
        |
        v
Alert the right team
        |
        v
Act before HA capacity is lost
```

## 1. Monitor every layer

Your monitoring should cover:

```text
Application connectivity
        |
HAProxy / Load Balancer
        |
Patroni
        |
PostgreSQL
        |
Replication
        |
OS / Disk / Network

And separately:

Patroni ---> etcd / DCS
```

If you monitor only PostgreSQL, you can miss a DCS or HAProxy failure.

---

## 2. Patroni metrics to watch

At minimum monitor:

```text
Current leader
Number of cluster members
Replica state
Replication lag
Timeline
Pending restart
Patroni REST API availability
Failover/switchover events
DCS connectivity
```

A healthy cluster should normally have:

```text
Exactly one leader
Expected number of replicas
Replicas streaming
No unexpected pending restart
```

If your expected topology is:

```text
1 primary
2 replicas
```

and suddenly:

```text
1 primary
1 replica
1 stopped
```

the application may still work, but HA protection has already been reduced.

That should alert.

---

# 3. Leader monitoring

This is one of the most important checks.

You want:

```text
Leader count = 1
```

Dangerous conditions:

```text
Leader count = 0
```

means:

> No writable Patroni leader.

And:

```text
Leader count > 1
```

could indicate a serious split-brain or monitoring inconsistency that needs immediate investigation.

For Patroni's REST API, a role-aware check can query:

```bash
curl -i http://pg01:8008/primary
```

You can monitor all nodes and expect only the current primary to satisfy that role check.

---

# 4. PostgreSQL replication monitoring

On the primary:

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

The most important state is usually:

```text
state = streaming
```

If a replica disappears from `pg_stat_replication`, investigate immediately.

Possible reasons:

```text
Replica down
Network issue
Authentication failure
WAL receiver failure
PostgreSQL restart
Disk problem
```

---

# 5. Replication lag

You should alert on lag, but don't use one universal threshold for every system.

For example:

```text
Warning  : > 100 MB
Critical : > 1 GB
```

might be reasonable in one workload and useless in another.

A high-write system may generate 1 GB of WAL very quickly.

A better question is:

> How far behind can a replica be before it becomes unsafe as a failover candidate?

You can inspect receive and replay positions:

```sql
SELECT
    pg_last_wal_receive_lsn(),
    pg_last_wal_replay_lsn();
```

And replay delay:

```sql
SELECT
    now() - pg_last_xact_replay_timestamp();
```

Useful monitoring dimensions are:

```text
Bytes behind
Time behind
Receive lag
Replay lag
```

---

# 6. Why time lag and byte lag are different

Imagine:

```text
Replica A: 2 GB behind
Replica B: 50 MB behind
```

You might assume B is healthier.

But if WAL generation is:

```text
10 GB/minute
```

then 2 GB could represent only seconds.

If the system generates:

```text
10 MB/minute
```

then 50 MB could represent several minutes.

So monitor both:

```text
WAL distance
and
replay time
```

rather than relying on one number.

---

# 7. Replication slot monitoring

This is critical.

Run:

```sql
SELECT
    slot_name,
    slot_type,
    active,
    restart_lsn
FROM pg_replication_slots;
```

Alert on:

```text
Inactive replication slot
        +
WAL retention increasing
```

Because:

```text
Replica down
    |
Slot retained
    |
WAL retained
    |
pg_wal grows
    |
Disk fills
```

A slot itself isn't bad.

An inactive slot retaining a large amount of WAL is the danger.

---

# 8. Monitor `pg_wal`

Watch filesystem usage:

```bash
df -h
```

and:

```bash
du -sh $PGDATA/pg_wal
```

Example alerting approach:

```text
Disk > 70%   = informational/watch
Disk > 80%   = warning
Disk > 90%   = critical
Disk > 95%   = emergency
```

Those numbers are examples, not universal production values.

More important is **rate of growth**.

If disk usage increases:

```text
70%
75%
82%
90%
```

within 30 minutes, that is much more dangerous than a filesystem sitting steadily at 80%.

---

# 9. Disk latency

Capacity isn't enough.

PostgreSQL and etcd both care about storage latency.

Monitor:

```text
Read latency
Write latency
fsync latency
I/O queue depth
Disk utilization
```

A disk may be:

```text
40% full
```

but overloaded enough to make PostgreSQL replication or etcd unstable.

This is especially important for:

```text
etcd
WAL filesystem
database data filesystem
```

---

# 10. etcd monitoring

Your 3-member etcd cluster needs:

```text
2 healthy members for quorum
```

Monitor:

```text
Member availability
Quorum
Raft leader
Leader changes
Peer communication
Disk latency
fsync duration
Database size
Network latency
```

Basic health test:

```bash
etcdctl endpoint health \
  --endpoints=http://etcd01:2379,http://etcd02:2379,http://etcd03:2379
```

And:

```bash
etcdctl endpoint status \
  --endpoints=http://etcd01:2379,http://etcd02:2379,http://etcd03:2379 \
  --write-out=table
```

If one member fails:

```text
2 / 3 healthy
```

the cluster still works.

But alert immediately because you have lost redundancy.

If another member fails:

```text
1 / 3
```

you lose quorum.

---

# 11. HAProxy monitoring

Monitor:

```text
HAProxy process
Frontend availability
Primary backend
Replica backend
Connection count
Health-check failures
Backend transitions
Connection errors
```

For your primary listener, you normally expect:

```text
Exactly one primary backend UP
```

Example:

```text
pg01 UP
pg02 DOWN
pg03 DOWN
```

That's healthy for a `/primary` health check.

After failover:

```text
pg01 DOWN
pg02 UP
pg03 DOWN
```

also healthy.

But:

```text
pg01 DOWN
pg02 DOWN
pg03 DOWN
```

should trigger a critical alert.

---

# 12. Don't misinterpret HAProxy DOWN

Remember:

```text
pg02 DOWN
```

inside a **primary backend** can simply mean:

```text
pg02 is a healthy replica
```

because it doesn't satisfy:

```text
/primary
```

So monitoring should distinguish:

```text
Backend role eligibility
```

from:

```text
Actual server health
```

For replicas, use a separate replica-aware backend/check.

---

# 13. PostgreSQL connection monitoring

Monitor:

```text
Current connections
Maximum allowed connections
Connection utilization
Failed authentication
Connection wait
Idle sessions
Idle in transaction
```

Example:

```text
max_connections = 500
current = 480
```

That's:

```text
96%
```

and deserves attention.

You don't want the first sign of connection exhaustion to be:

```text
FATAL: remaining connection slots are reserved
```

If PgBouncer is used, monitor both:

```text
Application -> PgBouncer connections
PgBouncer -> PostgreSQL connections
```

---

# 14. Long-running transactions

Long transactions can cause serious PostgreSQL problems.

Monitor:

```sql
SELECT
    pid,
    usename,
    datname,
    state,
    xact_start,
    now() - xact_start AS transaction_age,
    query
FROM pg_stat_activity
WHERE xact_start IS NOT NULL
ORDER BY xact_start;
```

Why care?

Long transactions can contribute to:

```text
vacuum problems
table bloat
WAL retention
recovery conflicts
replica lag
```

A Patroni cluster can be perfectly healthy while PostgreSQL itself is slowly developing a serious operational problem.

---

# 15. Locks and blocking

Monitor blocking sessions.

A simplified PostgreSQL investigation starts with:

```sql
SELECT
    pid,
    wait_event_type,
    wait_event,
    state,
    query
FROM pg_stat_activity
WHERE wait_event IS NOT NULL;
```

A database can be:

```text
PostgreSQL running
Patroni healthy
Replica healthy
```

while the application is effectively down because sessions are blocked.

That is why HA monitoring and database-performance monitoring are both needed.

---

# 16. Checkpoint and WAL generation

Monitor:

```text
WAL generation rate
Checkpoint frequency
Checkpoint duration
Buffers written
Archive failures
```

Why?

If WAL generation suddenly jumps:

```text
500 MB/min
      ->
10 GB/min
```

you may see:

```text
replication lag
archive backlog
disk growth
network saturation
```

That one change can affect several layers.

---

# 17. Backup monitoring

A Patroni cluster without working backups is not production-safe.

Alert on:

```text
Last successful full backup
Last successful incremental/differential backup
WAL archive freshness
Backup duration
Backup size
Repository space
Restore verification status
```

A backup system saying:

```text
Last successful backup: 8 days ago
```

should be a critical issue even if Patroni is completely healthy.

---

# 18. WAL archiving

If you use archive-based recovery, monitor:

```text
Last archived WAL
Archive failure count
Archive queue/backlog
Repository availability
```

PostgreSQL exposes archiver information through:

```sql
SELECT *
FROM pg_stat_archiver;
```

Important fields include:

```text
archived_count
failed_count
last_archived_wal
last_archived_time
last_failed_wal
last_failed_time
```

If:

```text
failed_count keeps increasing
```

you may lose your expected point-in-time recovery coverage.

---

# 19. Patroni failover alerts

You should alert whenever leadership changes unexpectedly.

For example:

```text
08:00 pg01 Leader

08:34 pg02 Leader
```

If there was no planned switchover, investigate.

Possible reasons:

```text
Primary crash
Network issue
DCS problem
Storage stall
Patroni restart
OS failure
Manual intervention
```

A successful automatic failover is still an incident.

HA prevented a bigger outage, but something still failed.

---

# 20. Timeline changes

After promotion, PostgreSQL creates a new timeline.

So timeline changes can be useful operational signals.

Example:

```text
Timeline 12
    |
failover
    |
Timeline 13
```

An unexpected timeline increase should correlate with:

```text
failover
switchover
promotion
recovery event
```

If it doesn't, investigate.

---

# 21. Pending restart alert

If Patroni shows:

```text
Pending restart
```

you don't necessarily need a critical alert.

But you should track it.

Example:

```text
Warning if pending restart > 24 hours
```

because it often means:

```text
Configuration has changed
but is not yet active
```

That can cause confusion during troubleshooting.

---

# 22. Prometheus architecture

A common monitoring design is:

```text
PostgreSQL exporter ----+
Patroni metrics --------+
node exporter ----------+
etcd metrics -----------+----> Prometheus
HAProxy exporter -------+         |
                                  v
                               Grafana
                                  |
                                  v
                            Alertmanager
                                  |
                    +-------------+-------------+
                    |                           |
                  Email                       Slack/etc.
```

The exact tooling can vary.

The important architecture is:

```text
Collect
   |
Store
   |
Visualize
   |
Alert
```

---

# 23. PostgreSQL exporter

A PostgreSQL exporter can collect metrics such as:

```text
connections
database size
transactions
locks
replication
WAL
checkpoint activity
dead tuples
cache activity
```

You can also expose custom SQL metrics for things like:

```text
replication lag
slot retention
oldest transaction
```

---

# 24. Node exporter

For the Linux layer monitor:

```text
CPU
memory
disk
filesystem
load
network
I/O
```

This is important because many Patroni incidents start outside PostgreSQL.

Example:

```text
PostgreSQL replication lag
```

could actually be caused by:

```text
replica disk 100% utilized
```

Node metrics help prove that.

---

# 25. Alert severity levels

I recommend using at least:

```text
INFO
WARNING
CRITICAL
```

Example:

### INFO

```text
Planned Patroni switchover completed
```

### WARNING

```text
One replica unavailable
```

Application is still healthy, but redundancy is reduced.

### CRITICAL

```text
No Patroni leader
```

or:

```text
etcd quorum lost
```

or:

```text
PostgreSQL disk > 95%
```

This prevents every alert from feeling equally urgent.

---

# 26. Suggested Patroni alerts

A practical starting set:

| Alert                        | Severity                                         |
| ---------------------------- | ------------------------------------------------ |
| No Patroni leader            | Critical                                         |
| Multiple leaders detected    | Critical                                         |
| Patroni REST API unavailable | Warning/Critical                                 |
| Replica missing              | Warning                                          |
| Replica lag beyond threshold | Warning                                          |
| Replica lag extremely high   | Critical                                         |
| Unexpected failover          | Critical                                         |
| Pending restart              | Info/Warning                                     |
| Patroni process down         | Critical on primary, Warning/Critical on replica |

Severity should depend on your environment and redundancy.

---

# 27. Suggested etcd alerts

| Alert                                  | Severity         |
| -------------------------------------- | ---------------- |
| One etcd member down in 3-node cluster | Warning          |
| etcd quorum lost                       | Critical         |
| Frequent Raft leader changes           | Warning          |
| High fsync latency                     | Warning          |
| Peer communication failures            | Warning/Critical |
| etcd disk almost full                  | Critical         |

Remember:

```text
1 etcd node down
```

is not a database outage.

But it means:

```text
you now have zero additional etcd failure tolerance
```

---

# 28. Suggested PostgreSQL alerts

Monitor at least:

```text
PostgreSQL down
connections high
disk high
replication disconnected
replication lag
long transactions
blocking
deadlocks
WAL archive failures
slot WAL retention
checkpoint pressure
database growth
```

Examples:

```text
Connections > 80% max
    -> Warning

Connections > 95%
    -> Critical
```

Again, use workload-specific thresholds.

---

# 29. Avoid noisy alerts

Bad monitoring:

```text
Replica lag 1 MB
ALERT

Replica lag 0 MB
RESOLVED

Replica lag 2 MB
ALERT
```

every few seconds.

That creates alert fatigue.

Use:

```text
threshold
+
duration
```

Example:

```text
Lag > threshold
for 5 minutes
```

before warning.

For critical issues like:

```text
No leader
```

you may want much faster alerting.

Different problems need different delay strategies.

---

# 30. Alert on redundancy loss

This is a very important architecture concept.

Healthy:

```text
pg01 PRIMARY
pg02 REPLICA
pg03 REPLICA
```

One replica dies:

```text
pg01 PRIMARY
pg02 REPLICA
pg03 DOWN
```

Application still works.

But HA capacity has degraded.

So alert on:

```text
Expected member count != actual healthy member count
```

Don't wait for the primary to fail too.

---

# 31. Monitor failover capability, not just availability

Imagine:

```text
pg01 PRIMARY
pg02 REPLICA nofailover=true
pg03 REPLICA nofailover=true
```

All PostgreSQL servers are healthy.

Monitoring might say:

```text
GREEN
```

But from an HA perspective:

```text
0 eligible failover candidates
```

That is risky.

A mature monitoring platform should understand:

```text
Can the cluster actually fail over?
```

not just:

```text
Are three postgres processes running?
```

---

# 32. Useful dashboard layout

A good Patroni dashboard could have:

```text
TOP ROW
Current leader
Cluster member count
Healthy replicas
DCS quorum
HAProxy primary backend

SECOND ROW
Replication lag
WAL generation
Connection utilization
Transaction rate

THIRD ROW
CPU
Memory
Disk usage
Disk latency

FOURTH ROW
Replication slots
WAL archive status
Backup status
Long transactions
```

That lets you answer quickly:

> Is this database, Patroni, DCS, routing, or infrastructure?

---

# 33. Example production incident

Alert:

```text
CRITICAL
Replica pg03 lag > 10 GB
```

You open Grafana.

See:

```text
WAL receive rate = normal
Replay rate = falling
Disk latency = very high
CPU = normal
Network = normal
```

Now your diagnosis becomes:

```text
Replication network OK
        |
        v
WAL arrives
        |
        v
Replica disk slow
        |
        v
Replay cannot keep up
```

That's far better than blindly restarting Patroni.

---

# 34. Another incident

Alert:

```text
No primary backend available in HAProxy
```

Dashboard shows:

```text
PostgreSQL nodes = UP

Patroni leader = pg02

HAProxy backend = all DOWN

Patroni API connectivity from HAProxy = failed
```

Diagnosis:

```text
Database healthy

Routing health-check path broken
```

Possible network/firewall issue between HAProxy and Patroni REST API.

Again, multi-layer monitoring gives you the answer faster.

---

# 35. Daily monitoring review

A DBA or platform engineer should be able to answer:

```text
Who is primary?

Are all replicas healthy?

How much lag exists?

Is etcd healthy?

Do we still have quorum?

Are replication slots healthy?

Is WAL growing unusually?

Are disks healthy?

Do we have a recent backup?

Is WAL archiving working?

Did leadership change recently?

Are any restarts pending?
```

If you can't answer these quickly, monitoring needs improvement.

---

# Monitoring cheat sheet

Think in these groups:

```text
PATRONI
Leader
Members
Failover
Pending restart


POSTGRESQL
Connections
Transactions
Locks
WAL
Replication
Slots


ETCD
Quorum
Members
Raft
Disk latency


HAPROXY
Primary backend
Replica backend
Connections
Health checks


OS
CPU
Memory
Disk
Network


BACKUPS
Last backup
WAL archive
Restore test
```

## The most important alerts

If I had to start with only ten alerts, I'd choose:

1. **No Patroni leader**
2. **Unexpected Patroni leadership change**
3. **Replica unavailable**
4. **Replication lag high**
5. **etcd quorum lost**
6. **One etcd member lost**
7. **PostgreSQL filesystem critically full**
8. **Replication slot causing excessive WAL retention**
9. **HAProxy has no healthy primary backend**
10. **Backup/WAL archive not current**

That gives you strong coverage across the entire HA architecture.

### Next — Lesson 14: Backup, Restore & Disaster Recovery

The logical next lesson is the part HA cannot solve: **pgBackRest-style backups, WAL archiving, Point-in-Time Recovery (PITR), restoring a complete Patroni cluster after catastrophic failure, and the difference between HA and DR.**
