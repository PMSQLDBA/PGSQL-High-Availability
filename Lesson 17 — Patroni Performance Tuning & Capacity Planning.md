# Lesson 17 — Patroni Performance Tuning & Capacity Planning

Patroni itself usually is not the main performance bottleneck. Most tuning is really about **PostgreSQL, storage, connections, WAL/replication, and the infrastructure around Patroni**.

A good mental model is:

```text
Application workload
      |
      v
Connections / PgBouncer
      |
      v
PostgreSQL CPU + Memory + Storage
      |
      v
WAL generation
      |
      v
Replication network
      |
      v
Replica replay
```

Patroni sits around this and manages HA.

## 1. Start with workload, not parameters

Before tuning, understand:

```text
Transactions per second
Read/write ratio
Peak connections
Database size
Growth rate
WAL generation rate
Largest queries
Latency requirement
RPO
RTO
```

For example:

```text
DB size: 2 TB
Peak connections: 800
Write rate: 15,000 TPS
WAL generation: 200 GB/day
RPO: < 1 minute
RTO: < 2 minutes
```

Those numbers drive architecture decisions.

---

## 2. CPU sizing

PostgreSQL benefits from CPU for:

```text
query execution
sorting
hashing
index operations
WAL processing
vacuum
parallel queries
```

But don't assume:

```text
more CPU = automatically faster
```

If your bottleneck is storage latency, adding CPU may do little.

Watch:

```text
CPU utilization
load average
run queue
context switching
query latency
```

A useful rule is to avoid running the primary constantly at 95–100% CPU.

Why?

Because after failover, a replica may need to handle the same workload immediately.

Capacity-plan for:

```text
normal workload
+
failure headroom
```

---

# 3. Failover capacity is more important than normal capacity

Suppose:

```text
pg01 primary = 70% CPU
pg02 replica = 20% CPU
pg03 replica = 20% CPU
```

Looks fine.

But if `pg01` fails, `pg02` may suddenly receive:

```text
application workload
+
replication duties
+
background processes
```

If pg02 hardware is much smaller than pg01, failover may succeed technically but application performance may collapse.

Production replicas intended for promotion should generally have enough capacity to become primary.

Think:

```text
HA replica
!= cheap backup server
```

---

# 4. Memory sizing

Important PostgreSQL memory parameters include:

```text
shared_buffers
work_mem
maintenance_work_mem
effective_cache_size
```

But they do different things.

### `shared_buffers`

This is PostgreSQL's own buffer cache.

A common starting point on dedicated database servers is around:

```text
25% of RAM
```

but that's only a starting point, not a universal rule.

Example:

```text
RAM = 64 GB

shared_buffers ≈ 16 GB
```

Then benchmark and monitor.

---

# 5. `work_mem` can be dangerous

This parameter is often misunderstood.

Suppose:

```text
work_mem = 64 MB
```

People sometimes think:

> PostgreSQL can use only 64 MB for all query work.

No.

`work_mem` can be used **per sort/hash operation, per query, per session**.

For example:

```text
500 active sessions
x
multiple sort/hash operations
x
64 MB
```

can consume huge amounts of memory.

So don't calculate:

```text
RAM / max_connections
```

and blindly assign the result to `work_mem`.

Monitor real workloads.

---

# 6. `maintenance_work_mem`

This is used for operations such as:

```text
VACUUM
CREATE INDEX
ALTER TABLE ADD FOREIGN KEY
```

It can usually be larger than `work_mem` because maintenance operations are less concurrent.

But remember:

```text
autovacuum workers
```

may also consume maintenance memory depending on configuration.

---

# 7. `effective_cache_size`

This is not memory PostgreSQL allocates.

It is a planner estimate:

> How much OS + PostgreSQL cache is likely available?

For example:

```text
Server RAM = 64 GB
shared_buffers = 16 GB
OS cache available = significant
```

You might use a higher `effective_cache_size` estimate.

It influences query planning, especially whether index scans appear attractive.

---

# 8. Connection sizing

A major PostgreSQL capacity problem is excessive direct connections.

Suppose:

```text
max_connections = 2000
```

That does not mean PostgreSQL should actually run 2000 active heavy queries at once.

Each backend consumes:

```text
memory
process resources
locks
kernel resources
```

and heavy concurrency can reduce throughput.

Sometimes:

```text
200 efficient active sessions
```

perform much better than:

```text
2000 competing active sessions
```

---

# 9. PgBouncer helps

Architecture:

```text
Applications
    |
  5000 client connections
    |
    v
PgBouncer
    |
  200 PostgreSQL connections
    |
    v
PostgreSQL
```

PgBouncer reduces connection overhead and smooths spikes.

This can be especially valuable during Patroni failover because applications may reconnect aggressively.

Without pooling:

```text
primary fails
    |
5000 clients reconnect
    |
new primary hit with connection storm
```

With pooling:

```text
applications
    |
PgBouncer absorbs/reuses connections
    |
new PostgreSQL primary
```

You still need to design failover and pool behavior carefully.

---

# 10. Storage is often the real bottleneck

PostgreSQL is highly dependent on storage performance.

Especially:

```text
WAL writes
random reads
random writes
checkpoints
vacuum
temporary files
```

Monitor:

```text
IOPS
throughput
read latency
write latency
fsync latency
queue depth
disk utilization
```

A storage system reporting:

```text
3 ms writes
```

is very different from:

```text
80 ms writes
```

even if CPU and RAM look healthy.

---

# 11. WAL storage matters

Transactions depend heavily on WAL durability.

Conceptually:

```text
COMMIT
   |
   v
WAL written
   |
   v
fsync
   |
   v
transaction confirmed
```

Slow WAL storage can directly increase transaction latency.

For write-heavy systems, monitor:

```text
WAL write latency
WAL sync latency
WAL generation rate
```

PostgreSQL exposes WAL statistics through views such as `pg_stat_wal`.

---

# 12. WAL generation rate

You need to know how much WAL your workload produces.

Example:

```text
Normal:
50 GB/hour

Peak:
150 GB/hour
```

That affects:

```text
replication network
replica disk
WAL archive bandwidth
backup repository
replication slots
recovery time
```

If the primary can generate WAL faster than replicas can receive/replay it:

```text
WAL generation > replica capacity
        |
        v
replication lag grows forever
```

No Patroni parameter fixes that.

You need more network/storage/CPU capacity or workload changes.

---

# 13. Replication network sizing

Suppose peak WAL generation is:

```text
300 MB/s
```

Your network path to a replica must sustain at least that rate, preferably with margin.

Bad design:

```text
Primary WAL = 300 MB/s
Network capacity = 125 MB/s

Result:
lag continuously grows
```

Design for:

```text
peak WAL
+
headroom
```

not average WAL.

---

# 14. Cross-region replicas

Latency becomes important.

Imagine:

```text
Primary: Chicago
Replica: London
```

For asynchronous replication, latency mostly affects:

```text
replication lag
catch-up speed
```

For synchronous replication, network RTT can directly impact transaction commits.

Example:

```text
local sync replica RTT = 1 ms
```

versus:

```text
remote sync replica RTT = 90 ms
```

A remote synchronous standby can significantly increase commit latency.

So don't enable synchronous replication across long-distance links without testing.

---

# 15. Replication lag capacity

Your replica should be able to:

```text
receive WAL
and
replay WAL
```

faster than average production generation.

Ideally:

```text
Replica replay capacity > peak sustainable WAL rate
```

because after temporary disruption, it needs to catch up.

For example:

```text
Normal WAL generation = 100 MB/s

Replica replay capability = 110 MB/s
```

That leaves little catch-up headroom.

Better:

```text
Replica replay capability = 200 MB/s
```

so lag can shrink after temporary outages.

---

# 16. Checkpoint tuning

PostgreSQL periodically writes dirty buffers during checkpoints.

Poor checkpoint behavior can create I/O spikes.

Think:

```text
normal workload
normal workload
normal workload

CHECKPOINT
|||||||||||||||||
huge disk spike
```

Useful parameters include:

```text
checkpoint_timeout
checkpoint_completion_target
max_wal_size
```

You usually want checkpoint writes spread across time rather than causing violent bursts.

Monitor:

```text
checkpoint frequency
checkpoint write time
checkpoint sync time
WAL volume
```

---

# 17. `max_wal_size`

If this is too small for a write-heavy workload, PostgreSQL may checkpoint frequently.

Example:

```text
high WAL generation
+
small max_wal_size
=
frequent checkpoints
```

Frequent checkpoints can increase:

```text
disk I/O
WAL volume
latency
```

But making it enormous without understanding recovery/storage implications is not ideal either.

Tune from workload data.

---

# 18. Autovacuum is part of performance planning

A Patroni cluster can be completely healthy while tables become bloated because autovacuum cannot keep up.

Monitor:

```text
dead tuples
vacuum frequency
autovacuum workers
transaction ID age
table growth
```

If a large table changes heavily:

```text
UPDATE/DELETE
      |
dead tuples
      |
autovacuum falls behind
      |
table/index bloat
      |
queries slow
```

This affects primary performance and replication because more WAL may be generated.

---

# 19. Capacity for vacuum

Vacuum needs:

```text
CPU
disk I/O
memory
```

Don't size production hardware only for application queries.

You also need room for:

```text
autovacuum
checkpoints
backups
replication
monitoring
maintenance
```

Think:

```text
Total capacity
=
application workload
+
database maintenance
+
HA overhead
+
failure headroom
```

---

# 20. Backups compete for resources

A backup can consume:

```text
disk reads
network
CPU for compression/encryption
repository bandwidth
```

Suppose primary storage normally runs at:

```text
70% utilization
```

Then backup starts:

```text
+ 30% I/O
```

Now storage is saturated.

That's why backup timing and replica-based backup strategies matter.

---

# 21. Patroni timing settings

Now to Patroni itself.

Typical configuration includes:

```yaml
loop_wait: 10
ttl: 30
retry_timeout: 10
```

These are not performance-tuning parameters in the normal PostgreSQL sense.

They influence:

```text
failure detection
DCS retry behavior
leader lock timing
```

Reducing them aggressively can make failover faster, but can increase sensitivity to transient network or DCS problems.

For example:

```text
Network jitter
   |
Patroni misses DCS
   |
very aggressive TTL
   |
leadership lost
```

You don't want normal network hiccups causing unnecessary failovers.

---

# 22. Don't tune Patroni timeouts for bragging rights

Bad requirement:

> We want failover in 2 seconds because faster is better.

Ask:

```text
What is actual RTO?
How reliable is network?
How fast can HAProxy detect role changes?
How fast can application reconnect?
How fast can PostgreSQL promote?
```

Overall recovery time is:

```text
Failure detection
+
Patroni election/promotion
+
routing update
+
application reconnect
```

Changing Patroni TTL is only one piece.

---

# 23. HAProxy health-check tuning

Suppose:

```text
inter 2s
fall 3
rise 2
```

Conceptually:

```text
check every 2 seconds

3 failures
-> mark DOWN

2 successes
-> mark UP
```

That means routing failover also takes time.

If Patroni promotes in 10 seconds but HAProxy waits another 10 seconds, application recovery isn't 10 seconds.

Measure the **end-to-end** RTO.

---

# 24. Application reconnect time matters

Suppose:

```text
Patroni failover = 15 sec
HAProxy update   = 3 sec
```

but application's connection pool retries every:

```text
60 sec
```

Users may experience close to a minute of disruption.

Again:

```text
Database HA
!=
application HA
```

Capacity and failover testing must include the real application.

---

# 25. etcd latency

Patroni depends on etcd.

etcd itself is sensitive to:

```text
disk fsync
network latency
CPU saturation
```

Monitor:

```text
etcd request latency
fsync duration
Raft leader changes
peer RTT
```

If etcd nodes are overloaded:

```text
Patroni
   |
slow DCS response
   |
timeouts
   |
HA instability
```

Keep etcd small, stable and predictable.

---

# 26. Don't overload etcd nodes

Running etcd on the same host as PostgreSQL can work in a lab.

But imagine production:

```text
PostgreSQL hits 100% disk
        |
same disk hosts etcd
        |
etcd fsync latency spikes
        |
Patroni loses reliable DCS
```

This is why separate failure/resource domains may be preferable for critical systems.

---

# 27. HAProxy sizing

HAProxy usually handles PostgreSQL TCP proxying efficiently, but still monitor:

```text
connections per second
concurrent connections
CPU
memory
network bandwidth
connection errors
```

If HAProxy becomes saturated:

```text
PostgreSQL healthy
Patroni healthy
etcd healthy

Application still slow
```

The connection layer needs capacity too.

---

# 28. Capacity planning example

Suppose current workload:

```text
Primary CPU           55%
Primary RAM           60%
Storage utilization   45%
Peak WAL generation   80 MB/s
Network capacity      1 GB/s
Connections           300
```

Expected annual growth:

```text
40%
```

Projected:

```text
CPU ~77%
WAL ~112 MB/s
Connections ~420
```

That might still fit, but you now have less failover and maintenance headroom.

Capacity planning should ask:

```text
What does this look like
12 months from now?
```

not only:

```text
Is it healthy today?
```

---

# 29. Growth planning

Track:

```text
Database size
Index size
WAL rate
TPS
QPS
Connections
CPU
Disk
Backup duration
Restore duration
Replica rebuild duration
```

Then trend them.

Example:

```text
January database = 1 TB
April            = 1.3 TB
July             = 1.7 TB
October          = 2.2 TB
```

That's much more useful than waiting for:

```text
disk 95% full
```

---

# 30. Capacity for replica rebuild

This gets overlooked.

Suppose:

```text
Database = 10 TB
```

A failed replica must be rebuilt.

Network throughput:

```text
200 MB/s
```

Very roughly:

```text
10 TB / 200 MB/s
```

means many hours even before accounting for protocol, compression, disk and concurrent workload.

During that time:

```text
HA redundancy is reduced
```

So replica rebuild time is itself an availability metric.

For very large systems, consider:

```text
backup-based replica creation
storage snapshots
higher bandwidth
additional replicas
```

---

# 31. Capacity for failover load

If read traffic is offloaded to replicas:

```text
Primary:
writes + some reads

Replica1:
reporting reads

Replica2:
reporting reads
```

after failover:

```text
Replica1 becomes primary
```

What happens to its reporting workload?

Now it may handle:

```text
production writes
+
application reads
+
reporting queries
```

This can overload the new primary.

So promotion candidates may need workload controls.

Some replicas may intentionally use:

```text
nofailover: true
```

if they serve special-purpose workloads.

---

# 32. Read replica capacity

If HAProxy load-balances read queries across two replicas:

```text
Read traffic = 1000 QPS

pg02 = 500
pg03 = 500
```

If pg03 fails:

```text
pg02 = 1000 QPS
```

Can pg02 handle it?

Capacity planning should assume:

```text
N-1
```

where reasonable.

Design so losing one member doesn't automatically overload the remaining members.

---

# 33. Monitoring performance baselines

Before tuning, establish normal values:

```text
CPU
disk latency
TPS
WAL MB/s
connections
replication lag
checkpoint frequency
query latency
```

Example:

```text
Normal p95 query latency = 40 ms
```

After a configuration change:

```text
p95 = 90 ms
```

You have evidence that something got worse.

Without baseline data, tuning becomes guesswork.

---

# 34. Don't tune everything at once

Bad:

```text
shared_buffers changed
work_mem changed
max_wal_size changed
checkpoint_timeout changed
autovacuum changed
kernel changed
```

then:

```text
performance improved 8%
```

You don't know why.

Better:

```text
Measure
   |
Change one logical area
   |
Measure
   |
Compare
```

Production tuning should be evidence-driven.

---

# 35. Top slow queries matter more than many server settings

Before spending hours tuning Patroni or PostgreSQL memory, check whether one query is doing:

```text
full scan of 2 billion rows
```

A missing index can cost more than any server parameter.

Use:

```text
pg_stat_statements
```

to identify:

```text
highest total execution time
highest average execution time
most calls
highest I/O
```

Typical priorities:

```text
bad query
missing index
bad plan
unnecessary workload
```

before:

```text
tiny parameter optimization
```

---

# 36. Example performance incident

Alert:

```text
Replica lag = 20 GB
```

Metrics:

```text
Primary WAL generation = normal
Network = normal
Replica CPU = 40%
Replica disk latency = 120 ms
Replay rate = low
```

Diagnosis:

```text
slow replica storage
       |
WAL received
       |
replay slow
       |
lag increases
```

Correct response is storage investigation.

Not:

```text
change Patroni ttl
```

---

# 37. Another example

Application latency increases.

Metrics:

```text
CPU 25%
Memory 50%
Disk normal
PostgreSQL connections = 950/1000
```

Investigation shows:

```text
many idle application sessions
```

Possible solution:

```text
PgBouncer
+
better application pool limits
```

rather than buying larger database servers.

---

# 38. Practical capacity headroom

There is no universal percentage, but production HA systems should avoid operating constantly at hard capacity limits.

I generally want clear headroom for:

```text
traffic spikes
failover
backup
vacuum
checkpoint
replica rebuild
maintenance
```

If normal primary usage is:

```text
CPU 95%
Disk 90%
Connections 98%
```

you don't really have a highly available system.

You have a system that happens to have replicas.

---

# 39. Capacity planning worksheet

For each Patroni cluster, maintain something like:

| Metric          | Current |     Peak |             Limit | 12-month estimate |
| --------------- | ------: | -------: | ----------------: | ----------------: |
| DB size         |    2 TB |        — |  5 TB disk budget |            3.2 TB |
| CPU             |     40% |      70% |              100% |               85% |
| Connections     |     250 |      600 |               800 |               750 |
| WAL rate        | 30 MB/s | 100 MB/s | network dependent |          140 MB/s |
| Replica lag     |       0 |   500 MB |     SLA threshold |                 — |
| Backup duration |      2h |     2.5h |         6h window |                4h |
| Replica rebuild |      3h |        — |         8h target |                5h |

This makes capacity discussions concrete.

---

# 40. The architecture mindset

When someone says:

> Patroni performance is slow.

Ask:

```text
Is PostgreSQL slow?
Is storage slow?
Are queries slow?
Are there too many connections?
Is replication lagging?
Is etcd slow?
Is HAProxy saturated?
Is the application retrying badly?
```

Patroni is usually the orchestration layer, not the query-performance engine.

## What to remember from Lesson 17

The most important principles are:

**Size replicas so they can actually become primary.**

**Plan for N-1 capacity, not only the normal state.**

**Track WAL generation because it drives replication, backup and recovery requirements.**

**PgBouncer is often more effective than simply raising `max_connections`.**

**Storage latency is frequently more important than raw CPU count.**

**Don't aggressively reduce Patroni TTL/loop settings just to make failover look faster. Measure end-to-end RTO.**

**Capacity-plan for backups, vacuum, failover and replica rebuilds—not just application queries.**

**Trend growth months ahead instead of reacting when disk reaches 95%.**

### Next — Lesson 18: Patroni Architecture Design Interview & Real-World Scenarios

A very useful next step would be scenario-based architecture questions: **design a Patroni platform for 5 TB, 20k TPS, two availability zones, near-zero RPO, 60-second RTO, cross-region DR, maintenance without downtime, and explain every design decision like you would in a technical interview or architecture review.**
