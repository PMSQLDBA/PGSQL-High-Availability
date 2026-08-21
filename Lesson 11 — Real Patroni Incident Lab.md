# Lesson 11 — Real Patroni Incident Lab

This lesson is different. I’ll give you realistic incidents with command output. Your job is to diagnose them before reading the answer.

For each case, think in this order:

```text
1. What is the symptom?
2. Which layer is failing?
3. What evidence proves it?
4. What is the safest next action?
5. What should I NOT do?
```

## Incident 1 — Primary died, no failover

You receive:

> Application cannot connect to PostgreSQL.

You run:

```bash
patronictl -c /etc/patroni/patroni.yml list
```

Output:

```text
+ Cluster: prod-db -----------------------------+
| Member | Host     | Role    | State           |
+--------+----------+---------+-----------------+
| pg01   | 10.0.1.11| Replica | stopped         |
| pg02   | 10.0.1.12| Replica | running         |
| pg03   | 10.0.1.13| Replica | running         |
+--------+----------+---------+-----------------+

Leader: None
```

You check etcd:

```bash
etcdctl endpoint health \
  --endpoints=http://etcd01:2379,http://etcd02:2379,http://etcd03:2379
```

Output:

```text
etcd01: unhealthy
etcd02: unhealthy
etcd03: healthy
```

### Diagnose before reading

What is the main problem?

The key clue is:

```text
3-member etcd cluster
only 1 member healthy
```

Quorum requires:

```text
2 / 3
```

So:

```text
No etcd quorum
      |
      v
Patroni cannot safely coordinate new leadership
      |
      v
No automatic failover
```

### Correct response

Investigate and restore **etcd quorum** first.

Check:

```bash
etcdctl endpoint status \
  --endpoints=http://etcd01:2379,http://etcd02:2379,http://etcd03:2379 \
  --write-out=table
```

Then investigate the unavailable members:

```text
network connectivity
etcd service
disk
CPU/memory
TLS/certificates
peer communication
```

### Dangerous action

Don't immediately force PostgreSQL promotion manually.

You first need to understand why the DCS cannot coordinate cluster leadership.

---

# Incident 2 — Replica is 25 GB behind

Cluster:

```text
pg01   Leader     running
pg02   Replica    streaming    Lag 0 MB
pg03   Replica    streaming    Lag 25600 MB
```

On `pg01`:

```sql
SELECT
    application_name,
    state,
    sent_lsn,
    write_lsn,
    flush_lsn,
    replay_lsn
FROM pg_stat_replication;
```

You see:

```text
pg02 | streaming | 0/90000000 | 0/90000000 | 0/90000000 | 0/90000000

pg03 | streaming | 0/90000000 | 0/90000000 | 0/90000000 | 0/70000000
```

Notice:

```text
pg03 write_lsn  = current
pg03 flush_lsn  = current
pg03 replay_lsn = far behind
```

### What's happening?

The network isn't the main problem.

`pg03` has already **received and flushed** WAL.

The problem is:

```text
WAL received
     |
     v
WAL written
     |
     v
WAL flushed
     |
     X
Replay cannot keep up
```

So investigate **WAL replay performance on pg03**.

Possible causes:

```text
slow storage
CPU pressure
long standby query
recovery conflicts
heavy workload
I/O saturation
```

Check:

```sql
SELECT
    pg_last_wal_receive_lsn(),
    pg_last_wal_replay_lsn(),
    now() - pg_last_xact_replay_timestamp();
```

And OS performance.

### Important distinction

If:

```text
sent_lsn >> write_lsn
```

you'd suspect network/receiver problems.

Here:

```text
write_lsn >> replay_lsn
```

points toward replay problems.

---

# Incident 3 — Replica disappeared

You see:

```text
pg01   Leader     running
pg02   Replica    streaming
pg03   Replica    stopped
```

On `pg03`:

```bash
journalctl -u patroni -n 50
```

shows:

```text
No space left on device
could not write to file
PANIC: could not write WAL file
```

Then:

```bash
df -h
```

shows:

```text
Filesystem      Size  Used Avail Use%
/dev/sdb1       200G  197G   3G   99%
```

### Root cause

This isn't primarily a Patroni problem.

The operating system/storage layer is failing:

```text
Disk full
  |
  v
PostgreSQL unable to write
  |
  v
PostgreSQL stopped
  |
  v
Patroni reports replica stopped
```

### Next checks

Find what consumed space:

```bash
du -sh /var/lib/postgresql/*
```

or for our lab:

```bash
du -sh /var/lib/patroni/data/*
```

Check:

```bash
du -sh /var/lib/patroni/data/pg_wal
```

Also check:

```text
PostgreSQL logs
system logs
backup files
temporary files
WAL
```

Don't simply delete random files inside `$PGDATA`.

---

# Incident 4 — WAL directory is enormous

On the primary:

```bash
du -sh /var/lib/patroni/data/pg_wal
```

Output:

```text
184G    /var/lib/patroni/data/pg_wal
```

Replication slots:

```sql
SELECT
    slot_name,
    active,
    restart_lsn
FROM pg_replication_slots;
```

Output:

```text
slot_name | active | restart_lsn
----------+--------+-------------
pg02      | t      | 0/A1000000
pg03      | f      | 0/21000000
```

`pg03` has been offline for four days.

### Diagnosis

The inactive `pg03` slot may be preventing old WAL from being recycled.

```text
pg03 offline
    |
replication slot retained
    |
required WAL retained
    |
pg_wal grows
```

### What do you do?

First determine whether:

```text
pg03 will return soon
pg03 can still catch up
pg03 should be rebuilt
```

If you're going to rebuild `pg03`, the old slot may eventually be removed/recreated through the proper Patroni process.

Don't immediately run:

```sql
SELECT pg_drop_replication_slot('pg03');
```

without understanding the consequences.

---

# Incident 5 — Old primary cannot rejoin

Before:

```text
pg01 PRIMARY
pg02 REPLICA
pg03 REPLICA
```

`pg01` fails.

Patroni promotes `pg02`.

Now:

```text
pg02 PRIMARY
pg03 REPLICA
```

You restart Patroni on `pg01`.

Logs show:

```text
requested timeline 8 is not a child of this server's history
pg_rewind execution failed
```

### What's likely happening?

The old primary has diverged from the current PostgreSQL history.

```text
Timeline 7
----------------------- pg01

                  FAILOVER
                     |
                     v

Timeline 8
----------------------- pg02
```

`pg01` can't simply start following timeline 8 without reconciliation.

### Check

Verify Patroni configuration:

```yaml
use_pg_rewind: true
```

Then determine why rewind failed.

Potential causes:

```text
required WAL unavailable
data checksums unavailable and wal_log_hints not enabled
authentication problem
connection issue
permissions
```

If rewind isn't possible, the clean approach may be:

```bash
patronictl reinit prod-db pg01
```

after confirming `pg01` is definitely the failed replica and `pg02` is the authoritative current primary.

---

# Incident 6 — Failover candidate refuses promotion

State:

```text
pg01    stopped
pg02    Replica    streaming
pg03    Replica    streaming

Leader: None
```

etcd is healthy.

Patroni logs on `pg02` show it doesn't participate in elections.

Configuration:

```yaml
tags:
  nofailover: true
```

`pg03` also has:

```yaml
tags:
  nofailover: true
```

### Diagnosis

Your HA software is working according to configuration.

Both surviving replicas have been explicitly marked:

```text
Do not promote me
```

So there is no eligible failover candidate.

### Lesson

During production review, don't look only at:

```text
number of replicas
```

Also understand:

```text
which replicas are eligible for failover
```

A cluster with two replicas isn't necessarily a cluster with two failover candidates.

---

# Incident 7 — HAProxy reports all PostgreSQL primary nodes DOWN

HAProxy status:

```text
postgres_primary

pg01 DOWN
pg02 DOWN
pg03 DOWN
```

But:

```bash
patronictl list
```

shows:

```text
pg01 Leader  running
pg02 Replica streaming
pg03 Replica streaming
```

From the HAProxy host:

```bash
curl http://pg01:8008/primary
```

returns:

```text
Connection timed out
```

On `pg01` itself:

```bash
curl http://localhost:8008/primary
```

returns HTTP 200.

### Diagnosis

PostgreSQL HA is fine.

Patroni is fine.

HAProxy cannot reach the Patroni REST API.

Likely:

```text
firewall
network ACL
routing
Patroni listen address
port 8008 blocked
```

Architecture:

```text
HAProxy
   |
   X port 8008
   |
Patroni
```

So HAProxy interprets every backend as unhealthy even though PostgreSQL itself is working.

### Correct troubleshooting

From HAProxy:

```bash
nc -vz pg01 8008
nc -vz pg02 8008
nc -vz pg03 8008
```

Then inspect firewall/network configuration.

Don't restart PostgreSQL.

---

# Incident 8 — Application fails after successful failover

Before:

```text
pg01 PRIMARY
```

Failover succeeds:

```text
pg02 PRIMARY
```

`patronictl list` is healthy.

HAProxy also shows:

```text
pg02 UP
```

But the application reports:

```text
connection reset
connection terminated unexpectedly
```

A few seconds later, new application connections work.

### Is Patroni broken?

No.

Existing TCP/PostgreSQL sessions connected to `pg01` were destroyed when `pg01` failed.

HAProxy cannot move this existing session:

```text
Application ---- pg01
```

into:

```text
Application ---- pg02
```

It can only route **new connections**.

So the application needs:

```text
connection retry
pool validation
reconnect behavior
transaction retry where appropriate
```

This is why application failover testing is required.

---

# Incident 9 — PostgreSQL process is running, but node isn't healthy in Patroni

On `pg03`:

```bash
ps -ef | grep postgres
```

shows PostgreSQL running.

But:

```bash
patronictl list
```

shows:

```text
pg03 | Replica | unknown
```

Patroni logs:

```text
etcdserver: request timed out
failed to get cluster from DCS
```

### Diagnosis

The PostgreSQL process is alive.

But the Patroni control plane can't reliably communicate with the DCS.

Think:

```text
PostgreSQL process health = OK

Patroni/DCS health = NOT OK
```

Don't confuse:

```text
postgres running
```

with:

```text
healthy HA cluster member
```

Check:

```bash
etcdctl endpoint health
```

and network connectivity from `pg03` specifically.

---

# Incident 10 — Two PostgreSQL nodes say they are primary

This is the most serious scenario.

You run on `pg01`:

```sql
SELECT pg_is_in_recovery();
```

Result:

```text
f
```

On `pg02`:

```sql
SELECT pg_is_in_recovery();
```

Result:

```text
f
```

Application A is writing to `pg01`.

Application B is writing to `pg02`.

### Stop and think

You potentially have:

```text
        NETWORK / CONTROL FAILURE

pg01                          pg02
PRIMARY                       PRIMARY
  |                             |
Writes A                     Writes B
```

This can create diverging data histories.

### Immediate priority

The priority is not:

```text
make Patroni green
```

The priority is:

> **Stop further data divergence.**

Determine which node Patroni/DCS considers authoritative:

```bash
patronictl list
```

Check Patroni APIs:

```bash
curl http://pg01:8008/
curl http://pg02:8008/
```

Check:

```text
DCS quorum
network partition
timelines
Patroni logs
whether somebody manually started PostgreSQL
```

One very common operational cause is bypassing Patroni and manually starting an old primary.

### Do not

Do not start merging databases manually or arbitrarily choose whichever node "looks newest."

You need to establish:

```text
authoritative timeline
latest valid transactions
current DCS leader
application write history
```

This is an escalation-worthy incident.

---

# Bonus Incident 11 — Replica keeps restarting

Patroni:

```text
pg01 Leader  running
pg02 Replica streaming
pg03 Replica start failed
```

Logs:

```text
FATAL: could not connect to primary
FATAL: password authentication failed for user "replicator"
```

### Diagnosis

This is an authentication issue.

Check:

```text
Patroni replication username/password
pg_hba.conf rules
SCRAM credentials
password consistency
```

The replication path is:

```text
pg03
 |
 | replicator credentials
 |
 v
pg01
```

Patroni cannot fix incorrect PostgreSQL credentials by itself.

---

# Bonus Incident 12 — Replica appears healthy but cannot be used for failover

Cluster:

```text
pg01 Leader   running
pg02 Replica  streaming  Lag 0
pg03 Replica  streaming  Lag 0
```

Looks perfect.

But `pg02` config contains:

```yaml
tags:
  nofailover: true
```

and `pg03` is used for reporting and contains:

```yaml
tags:
  nofailover: true
```

So operationally:

```text
3 PostgreSQL nodes
```

but effectively:

```text
1 primary-capable node
```

If `pg01` fails:

```text
no eligible candidate
```

That's why architecture reviews must include **Patroni tags and policy**, not just server counts.

---

# Your production triage card

When you get a Patroni incident, use this order:

```text
               INCIDENT
                   |
                   v
       patronictl list
                   |
                   v
        Who is leader?
                   |
                   v
        DCS quorum healthy?
                   |
                   v
      Patroni API healthy?
                   |
                   v
    PostgreSQL role correct?
                   |
                   v
    Replication healthy?
                   |
                   v
     OS/disk/network OK?
                   |
                   v
           Read logs
                   |
                   v
      Identify root cause
                   |
                   v
Take smallest safe corrective action
```

## Commands I want you to remember

Patroni:

```bash
patronictl -c /etc/patroni/patroni.yml list
```

Patroni logs:

```bash
journalctl -u patroni -n 200
```

PostgreSQL role:

```sql
SELECT pg_is_in_recovery();
```

Replication:

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

Replica receiver:

```sql
SELECT * FROM pg_stat_wal_receiver;
```

Slots:

```sql
SELECT
    slot_name,
    active,
    restart_lsn
FROM pg_replication_slots;
```

DCS:

```bash
etcdctl endpoint health
```

Disk:

```bash
df -h
```

Network ports:

```bash
ss -lntp
```

Patroni API:

```bash
curl http://localhost:8008/
```

The real skill isn't memorizing these commands. It's knowing **which one answers which question**.

## Mini exam

Try these without looking back:

**A.** Primary died, no failover, only 1/3 etcd members available.
→ **DCS quorum problem.**

**B.** Replica `write_lsn` is current but `replay_lsn` is far behind.
→ **WAL replay problem.**

**C.** `pg_wal` is 180 GB and a replication slot has been inactive for days.
→ **Slot/WAL retention investigation.**

**D.** Patroni elected a new primary but clients connected to the old primary received errors.
→ **Expected for existing connections; application must reconnect.**

**E.** Old primary won't rejoin after failover and timelines diverged.
→ **Investigate `pg_rewind`; reinitialize if necessary.**

**F.** HAProxy says all nodes DOWN, but Patroni has a healthy leader.
→ **Check HAProxy → Patroni REST API connectivity and health-check configuration.**

**G.** Two servers return `pg_is_in_recovery() = false`.
→ **Potential split-brain emergency; stop additional divergence and establish authoritative leadership/history.**

At this point, you have enough foundation to start thinking like someone supporting Patroni rather than just someone who installed it.

### Next — Lesson 12: Patroni Commands & Daily DBA Operations

The next useful step is a practical **daily operations lesson**: `patronictl list`, `show-config`, `edit-config`, `reload`, `restart`, `switchover`, `failover`, `reinit`, `pause`, `resume`, member tags, pending restart, maintenance workflows, and which commands are safe versus dangerous in production.
