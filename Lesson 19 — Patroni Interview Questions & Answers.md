# Lesson 19 — Patroni Interview Questions & Answers

Below is a practical interview pack you can use for PostgreSQL + Patroni roles. I’ll keep the answers short enough to say verbally, but strong enough for a technical interview.

## 1. What is Patroni?

**Answer:** Patroni is a high-availability framework for PostgreSQL. It manages PostgreSQL roles, leader election, failover, switchover, and cluster coordination using a distributed configuration store such as etcd.

---

## 2. Does Patroni replicate PostgreSQL data?

**Answer:** No. PostgreSQL streaming replication handles the actual data replication. Patroni manages which PostgreSQL node is primary or replica and coordinates failover.

---

## 3. What is the role of etcd in Patroni?

**Answer:** etcd acts as the Distributed Configuration Store, or DCS. Patroni uses it for cluster coordination, leader locking, and shared HA state.

---

## 4. What is quorum?

**Answer:** Quorum is the majority of members required for consensus.

For 3 etcd members:

```text
Quorum = 2
```

For 5:

```text
Quorum = 3
```

---

## 5. Why use three etcd nodes instead of two?

**Answer:** A two-member etcd cluster requires both nodes for quorum, so losing one member loses quorum. A three-member cluster requires two and can tolerate one failure.

---

## 6. What is a Patroni leader lock?

**Answer:** It is a lease stored in the DCS that identifies the current Patroni leader. The leader continuously renews it. If it can no longer renew the lock, another eligible member may acquire leadership.

---

## 7. What is the difference between primary and leader?

In a normal Patroni cluster they refer to the same active PostgreSQL node from different viewpoints.

**Primary** is the PostgreSQL database role.

**Leader** is the Patroni cluster role.

---

## 8. What is a Patroni switchover?

**Answer:** A switchover is a planned change of PostgreSQL leadership from a healthy primary to a healthy replica, usually for maintenance.

```text
pg01 Primary
      |
 switchover
      |
      v
pg02 Primary
```

---

## 9. What is failover?

**Answer:** Failover occurs when the current primary becomes unavailable and an eligible replica is promoted to primary.

The key difference is:

```text
Switchover = planned
Failover   = failure-driven
```

---

## 10. What happens when the Patroni primary crashes?

**Answer:** The old leader stops renewing its DCS leadership. Remaining Patroni members evaluate cluster state and an eligible replica can acquire leadership and promote PostgreSQL.

---

## 11. Is Patroni failover immediate?

**Answer:** No. Patroni uses timing controls such as `loop_wait`, `ttl`, and retry behavior. It needs enough time to distinguish a real failure from a temporary connectivity issue.

---

## 12. What does `loop_wait` mean?

**Answer:** It controls roughly how often Patroni runs its HA control loop and checks cluster state.

---

## 13. What does `ttl` mean?

**Answer:** It is the time-to-live of the leader lock in the DCS. The current leader must keep renewing it.

---

## 14. Why shouldn't `ttl` be extremely low?

**Answer:** An aggressive TTL can make the cluster too sensitive to short network or DCS delays and may cause unnecessary leadership changes.

---

## 15. What command do you use first during a Patroni incident?

```bash
patronictl list
```

**Answer:** It quickly shows the leader, members, roles, state, timeline, and replication lag.

---

## 16. How do you confirm whether PostgreSQL itself is primary?

```sql
SELECT pg_is_in_recovery();
```

Interpretation:

```text
false = primary
true  = standby
```

---

## 17. How do you check streaming replication from the primary?

```sql
SELECT
    application_name,
    client_addr,
    state,
    sync_state
FROM pg_stat_replication;
```

You normally want:

```text
state = streaming
```

---

## 18. What is WAL?

**Answer:** WAL stands for Write-Ahead Log. PostgreSQL records database changes into WAL before the corresponding data-file changes are persisted. WAL is also the foundation of physical streaming replication and crash recovery.

---

## 19. What is replication lag?

**Answer:** It is the difference between the primary's WAL progress and the replica's receive/replay progress.

Large lag means the replica is behind and may be a poor failover candidate.

---

## 20. How do you differentiate receive lag from replay lag?

On a replica:

```sql
SELECT
    pg_last_wal_receive_lsn(),
    pg_last_wal_replay_lsn();
```

If receive is current but replay is far behind, WAL is arriving but PostgreSQL cannot replay it quickly enough.

---

## 21. What is asynchronous replication?

**Answer:** The primary can acknowledge a commit without waiting for the standby to confirm it.

Advantage:

```text
lower latency
```

Risk:

```text
possible acknowledged transaction loss
during certain failures
```

---

## 22. What is synchronous replication?

**Answer:** PostgreSQL waits for confirmation from a configured synchronous standby before completing commits according to the configured synchronous level.

It provides stronger durability but can increase write latency and make standby availability affect writes.

---

## 23. What are RPO and RTO?

**RPO:** How much data loss is acceptable.

**RTO:** How long service can remain unavailable.

For example:

```text
RPO = 0
RTO = 60 seconds
```

means very strong durability and fast recovery requirements.

---

## 24. Can Patroni guarantee zero data loss?

**Answer:** Patroni alone cannot. Data-loss guarantees depend heavily on PostgreSQL replication mode, synchronous configuration, failure scenarios, and infrastructure.

---

## 25. What is split brain?

**Answer:** Split brain happens when more than one PostgreSQL node independently acts as primary and accepts writes.

Example:

```text
pg01 PRIMARY    pg02 PRIMARY
    |               |
  writes           writes
```

The histories can diverge.

---

## 26. How does Patroni help avoid split brain?

**Answer:** Patroni uses DCS-based leader coordination and leader-lock ownership. Only a node that safely holds leadership should remain authoritative.

Watchdog or other fencing mechanisms can add extra protection.

---

## 27. What is fencing?

**Answer:** Fencing ensures that an old or isolated primary cannot continue serving writes after leadership has moved elsewhere.

Examples include:

```text
watchdog reset
STONITH
VM shutdown
storage fencing
network isolation
```

---

## 28. What is Patroni watchdog?

**Answer:** Patroni can use the Linux watchdog as an additional safety mechanism. If the leader cannot safely maintain DCS coordination, watchdog behavior can help ensure the server is reset rather than remaining an unsafe primary.

---

## 29. What is `pg_rewind`?

**Answer:** `pg_rewind` can synchronize an old primary's diverged data directory with the new primary after failover.

Example:

```text
pg01 old primary
      |
      | pg_rewind
      v
pg02 new primary
```

Then `pg01` can rejoin as a replica.

---

## 30. When would `pg_rewind` not work?

Possible reasons include:

```text
required WAL missing
data checksums/wal_log_hints requirements not met
authentication failure
filesystem/permission issue
timeline problems
```

If rewind cannot safely recover the node, the replica may need to be rebuilt.

---

## 31. What is `patronictl reinit`?

**Answer:** It rebuilds a replica from a healthy cluster source.

Use it when a replica cannot be repaired or rewound.

It is destructive to the target member's local database copy.

---

## 32. What command is dangerous if used on the wrong member?

A few:

```bash
patronictl reinit
patronictl failover
patronictl remove
```

Always confirm:

```text
cluster
member
role
current leader
```

before using them.

---

## 33. What is `nofailover`?

A Patroni tag such as:

```yaml
tags:
  nofailover: true
```

means the member should not be automatically selected as a failover candidate.

It can be useful for reporting or special-purpose replicas.

---

## 34. What is `patronictl show-config`?

**Answer:** It displays the dynamic Patroni cluster configuration stored in the DCS.

```bash
patronictl show-config
```

---

## 35. Why doesn't changing `bootstrap.dcs` always affect a running cluster?

**Answer:** `bootstrap.dcs` is mainly used when the cluster is initially created. After bootstrap, dynamic settings are stored in the DCS.

Running-cluster changes are generally made through Patroni's dynamic configuration mechanism, such as:

```bash
patronictl edit-config
```

---

## 36. What does `Pending restart` mean?

**Answer:** A configuration change exists that PostgreSQL cannot apply through reload and requires that member to restart.

It doesn't automatically mean the node is unhealthy.

---

## 37. What does `patronictl reload` do?

**Answer:** It asks Patroni/PostgreSQL to reload configuration that can be applied without restarting PostgreSQL.

---

## 38. What does `patronictl restart` do?

**Answer:** It restarts PostgreSQL under Patroni control for the selected member.

Before using it, always confirm whether that member is currently primary or replica.

---

## 39. What does `patronictl pause` do?

**Answer:** Pause mode changes normal Patroni automation behavior for maintenance.

It should be used only when you understand why normal HA automation needs to be temporarily reduced.

After maintenance:

```bash
patronictl resume
```

---

# 40. Why is HAProxy used with Patroni?

**Answer:** Patroni can promote PostgreSQL, but applications still need a stable endpoint that finds the new primary.

HAProxy can health-check Patroni's REST API and route new database connections to the correct node.

```text
Application
    |
 HAProxy
    |
current primary
```

---

# 41. Which ports are commonly involved?

Typical:

```text
5432 = PostgreSQL
8008 = Patroni REST API
2379 = etcd client
2380 = etcd peer
```

Port assignments can be changed, but these are common defaults/examples.

---

# 42. Can HAProxy move an existing PostgreSQL session after failover?

**Answer:** No.

An existing TCP session to the failed primary is lost.

HAProxy can route **new connections** to the new primary.

The application or connection pool must reconnect.

---

# 43. Why use PgBouncer?

**Answer:** PgBouncer provides connection pooling.

Instead of thousands of direct PostgreSQL server processes, applications can maintain many client connections while PgBouncer uses a smaller controlled pool of PostgreSQL connections.

---

# 44. PgBouncer vs HAProxy?

Good interview question.

**HAProxy:**

> Which PostgreSQL server should I connect to?

**PgBouncer:**

> How should database connections be pooled and reused?

They solve different problems.

---

# 45. What happens if HAProxy itself fails?

If there is only one HAProxy:

```text
PostgreSQL healthy
Patroni healthy
etcd healthy

but

Application cannot connect
```

So the routing layer should also be redundant.

For example:

```text
HAProxy01
HAProxy02
+
VIP / cloud load balancer
```

---

# 46. What happens if one etcd node fails in a 3-node cluster?

You still have:

```text
2 / 3
```

so quorum remains.

The cluster is operational but degraded.

Repair the failed etcd member because there is no additional member-failure tolerance.

---

# 47. What happens if two etcd nodes fail?

Only:

```text
1 / 3
```

remain.

No quorum.

Patroni can no longer safely use the DCS for normal leadership coordination.

This is a critical incident.

---

# 48. Why shouldn't you immediately force failover if etcd has lost quorum?

**Answer:** You need to understand whether you're dealing with actual server loss or a network partition. Forcing an independent primary while another primary may still exist can create split brain.

---

# 49. What would you check if automatic failover didn't happen?

Good answer:

```text
patronictl list
DCS quorum
Patroni logs
replica health
replication lag
timeline
member tags
synchronous requirements
network connectivity
```

Then determine why no replica was eligible or able to acquire leadership.

---

# 50. What would you check if a replica is lagging?

I would check:

```text
pg_stat_replication
pg_stat_wal_receiver
receive_lsn vs replay_lsn
CPU
disk latency
network throughput
long-running standby queries
WAL generation rate
```

The key is determining whether WAL is:

```text
not arriving
```

or:

```text
arriving but replaying slowly
```

---

# 51. What can cause `pg_wal` to grow unexpectedly?

Common causes:

```text
inactive replication slot
broken WAL archive
replica offline
high WAL generation
checkpoint/archive problems
```

Replication slots are especially important because PostgreSQL may retain required WAL for an offline consumer.

---

# 52. How do you check replication slots?

```sql
SELECT
    slot_name,
    active,
    restart_lsn
FROM pg_replication_slots;
```

Investigate inactive slots that are retaining significant WAL.

---

# 53. Is a Patroni replica a backup?

**Answer:** No.

Replication protects mainly against server failure.

If someone runs:

```sql
DROP TABLE customer;
```

that operation is replicated too.

You still need physical backups, WAL archiving, PITR, and DR.

---

# 54. What is PITR?

**Answer:** Point-in-Time Recovery restores a base backup and replays archived WAL until a chosen time or restore point.

Example:

```text
10:35:20 bad DELETE

Recover to:
10:35:19
```

---

# 55. Why use pgBackRest?

**Answer:** pgBackRest is a PostgreSQL backup and restore tool that supports physical backups, WAL archiving, retention, repository management, PITR, and other production backup capabilities.

---

# 56. What would you do after complete Patroni cluster loss?

Strong answer:

> I would restore one authoritative PostgreSQL instance from a known-good backup and archived WAL, validate it, establish it as the recovered primary, and then rebuild the remaining replicas from that source rather than independently restoring multiple potential primaries.

---

# 57. Where should backups live?

**Answer:** Outside the PostgreSQL cluster's main failure domain.

Preferably:

```text
separate storage
offsite copy
cross-region copy
immutable storage where needed
```

depending on business requirements.

---

# 58. How would you monitor Patroni?

Monitor:

```text
leader present
member count
replica state
replication lag
unexpected failover
Patroni REST
DCS connectivity
pending restart
timeline changes
```

But also monitor PostgreSQL, etcd, HAProxy, OS, backups, and application connectivity.

---

# 59. What are the most important alerts?

Good answer:

```text
No Patroni leader
Unexpected failover
Replica missing
High replication lag
etcd quorum lost
Disk critically full
WAL retention excessive
HAProxy no primary backend
Backup failure
WAL archive failure
```

---

# 60. How do you secure a Patroni environment?

Mention multiple layers:

```text
PostgreSQL SCRAM
TLS
restrictive pg_hba.conf
least-privilege DB roles
Patroni REST authentication/TLS
secure etcd with TLS/mTLS
network segmentation
firewalls
protected patroni.yml
secret management
backup encryption
certificate monitoring
```

That is a strong architecture answer.

---

# 61. Why use SCRAM instead of MD5?

**Answer:** SCRAM-SHA-256 provides stronger password authentication. PostgreSQL has deprecated MD5 password authentication and recommends moving toward SCRAM.

---

# 62. Why secure Patroni REST port 8008?

**Answer:** Patroni REST is part of the HA control interface and can be involved in administrative operations, not only health checks.

It should be restricted by network policy and protected with authentication/TLS as appropriate.

---

# 63. Why secure etcd strongly?

**Answer:** etcd stores the distributed coordination state Patroni depends on. Unauthorized access to the DCS could interfere with cluster leadership or configuration.

---

# 64. How would you perform OS maintenance on the current primary?

Good workflow:

```text
Check cluster
Check replicas
Switchover primary
Verify new primary
Patch old primary
Restart
Verify it rejoins as replica
```

That avoids intentionally restarting the active primary.

---

# 65. Can PostgreSQL major upgrades be done like normal Patroni rolling restarts?

**Answer:** No.

A major version such as:

```text
PostgreSQL 17 -> PostgreSQL 18
```

changes the PostgreSQL data-version boundary.

Use a proper migration method such as:

```text
pg_upgrade
logical replication
dump/restore
```

---

# 66. How would you perform an in-place Patroni major upgrade?

High-level answer:

```text
Back up
Test pg_upgrade
Stop writes
Stop Patroni cluster
Upgrade primary with pg_upgrade
Update configuration/DCS state
Start upgraded primary
Upgrade replica binaries
Rebuild replicas
Validate
```

---

# 67. How would you achieve very low downtime during a major upgrade?

Use a blue-green approach:

```text
Old PG17 Patroni cluster
       |
logical replication
       |
New PG18 Patroni cluster
```

Sync data, stop writes briefly, catch up, synchronize sequences, and switch the application endpoint.

---

# 68. What is the biggest risk with logical replication upgrades?

Good points:

```text
DDL isn't automatically handled like row changes
sequences need synchronization
extensions must be recreated
roles/privileges need migration
cutover/rollback needs careful planning
```

---

# 69. How do you capacity-plan a Patroni cluster?

I would size for:

```text
CPU
RAM
storage IOPS/latency
WAL throughput
network bandwidth
connections
backup workload
replica replay capacity
growth
failover capacity
replica rebuild time
```

The key is sizing for failures, not just the healthy state.

---

# 70. Why should replicas be sized like the primary?

Because after failover:

```text
replica -> primary
```

It needs to handle the production workload.

A tiny standby that cannot carry the workload is not a useful HA candidate.

---

# 71. Why isn't `max_connections=5000` necessarily a good solution?

Because each PostgreSQL connection consumes resources and too many active backends can reduce performance.

Connection pooling with PgBouncer is often a better architecture.

---

# 72. Why does WAL generation rate matter?

It affects:

```text
replication bandwidth
replica lag
WAL storage
backup/archive bandwidth
recovery time
```

If replicas cannot process WAL as fast as the primary generates it, lag keeps growing.

---

# 73. What is a good production failure test?

Examples:

```text
kill PostgreSQL primary
kill primary VM
kill one etcd node
lose etcd quorum in controlled test
network partition
HAProxy failure
old primary rejoin
pg_rewind
replica rebuild
backup restore
PITR
DR failover
```

The important part is measuring actual RPO and RTO.

---

# 74. What is a good answer to “Design PostgreSQL HA with Patroni”?

You can say:

> I would start with the business RPO, RTO, workload, failure domains, and geographic requirements. For a critical workload, I would typically deploy three PostgreSQL/Patroni nodes across independent failure domains and use a three-member DCS such as etcd with quorum distributed across those domains. Applications would connect through a redundant routing and connection-pooling layer rather than directly to a database node. I would choose synchronous or asynchronous replication based on the durability requirement and measured network latency. I would also include off-cluster backups, WAL archiving, monitoring, fencing, security controls, and tested disaster-recovery procedures.

That is a strong senior-level answer.

---

# 75. Scenario — primary dies but no failover

Interviewer:

> Primary crashed but no replica was promoted. What do you do?

Strong answer:

> I first check `patronictl list` to understand cluster state. Then I verify etcd quorum because Patroni needs a healthy DCS to coordinate leadership. I check Patroni logs on the replicas, replication state and lag, failover tags such as `nofailover`, timeline compatibility, and synchronous-replication constraints. I would not force a failover until I understand why automatic failover did not occur and which replica contains the safest data.

---

# 76. Scenario — old primary comes back

Interviewer:

> pg01 failed, pg02 became primary, and pg01 comes back. What happens?

Answer:

> pg01 should not simply resume as primary. Patroni sees pg02 as the current leader. Because PostgreSQL history may have diverged, Patroni can use `pg_rewind` when possible to reconcile pg01 with the new timeline. If it cannot be rewound, pg01 may need to be reinitialized as a replica.

---

# 77. Scenario — two nodes appear primary

Interviewer:

> Two PostgreSQL nodes return `pg_is_in_recovery() = false`. What do you do?

Strong answer:

> I treat it as a potential split-brain incident. My priority is to stop additional write divergence, determine which Patroni/DCS leadership is authoritative, inspect network partition and timelines, and identify where writes occurred. I would not randomly restart or promote nodes. If the authoritative data history is unclear, I would escalate before taking destructive action.

---

# 78. Scenario — disk 98% full

Interviewer:

> The primary disk is 98% full. What do you check?

Answer:

```text
pg_wal size
replication slots
WAL archiving
logs
temporary files
database growth
filesystem usage
```

Do not randomly delete files from `$PGDATA`.

---

# 79. Scenario — HAProxy all primary backends DOWN

Good answer:

> I first check whether Patroni actually has a leader. If yes, I manually test the same Patroni REST health endpoint HAProxy uses, such as `/primary`, from the HAProxy host. If it fails, I investigate network/firewall, Patroni REST listen address, TLS or health-check configuration. I don't immediately restart PostgreSQL because the database may be healthy.

---

# 80. Scenario — synchronous replica fails

Interviewer:

> What can happen if the required synchronous replica becomes unavailable?

Answer:

> Depending on PostgreSQL and Patroni synchronous configuration, commits may block or the synchronous standby assignment may change. The trade-off is stronger durability versus write availability, so the exact behavior must be intentionally designed and tested.

---

# Quick 10-question practice round

Try answering these verbally without looking above:

1. What does Patroni do?
2. Who performs PostgreSQL replication?
3. Why does Patroni need etcd?
4. What is quorum for three etcd nodes?
5. Switchover vs failover?
6. How do you determine PostgreSQL primary/replica role?
7. Why can async replication lose data?
8. What does `pg_rewind` solve?
9. Why isn't a replica a backup?
10. Why does HAProxy need Patroni's REST API?

If you can answer those smoothly, your foundation is strong.

## Senior-level practice

These are the questions that separate basic knowledge from architecture knowledge:

```text
How do you prevent split brain?

What happens during DCS quorum loss?

How does synchronous mode affect availability?

How do you safely recover an old primary?

How would you design Patroni across availability zones?

How would you design cross-region DR?

How would you perform a PostgreSQL major upgrade?

How do you verify actual RPO/RTO?

How do you handle routing-layer HA?

How do you capacity-plan for failover rather than normal operation?
```

At this point, you have covered most of the knowledge expected for a solid **PostgreSQL + Patroni production support / engineering interview**.

### Next — Lesson 20: Mock Patroni Interview

The best next step is to make this interactive: I can act as the interviewer and give you **one Patroni question at a time**, you answer in your own words, and I’ll score your answer, correct anything missing, and show you a stronger interview response before moving to the next question.
