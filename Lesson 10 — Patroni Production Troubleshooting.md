# Lesson 10 — Patroni Production Troubleshooting

The most important troubleshooting rule with Patroni is:

> **Don't restart things randomly. First identify which layer is actually broken.**

A Patroni environment has several layers:

```text
Application
    |
    v
HAProxy / Load Balancer
    |
    v
Patroni
    |
    v
PostgreSQL
    |
    v
Streaming Replication
    |
    v
OS / Network / Storage

Patroni also depends on:

Patroni ---> etcd / DCS
```

A database outage can originate from any of them.

---

# 1. Your first five checks

During an incident, I would normally start here.

### Check 1 — Patroni cluster state

```bash
patronictl -c /etc/patroni/patroni.yml list
```

Look for:

```text
Leader
Replica
State
TL / Timeline
Lag
Pending restart
```

Example healthy cluster:

```text
+ Cluster: pg-ha-cluster -----------------------------+
| Member | Host          | Role    | State     | Lag |
+--------+---------------+---------+-----------+-----+
| pg01   | 192.168.10.11 | Leader  | running   |     |
| pg02   | 192.168.10.12 | Replica | streaming |   0 |
| pg03   | 192.168.10.13 | Replica | streaming |   0 |
+--------+---------------+---------+-----------+-----+
```

---

### Check 2 — Patroni API

On each node:

```bash
curl http://localhost:8008/
```

Or:

```bash
curl -i http://localhost:8008/primary
```

This tells you what **Patroni thinks** about the node.

---

### Check 3 — PostgreSQL role

```sql
SELECT pg_is_in_recovery();
```

Interpret it:

```text
false = primary
true  = replica
```

This tells you what **PostgreSQL itself thinks**.

PostgreSQL documents `pg_is_in_recovery()` specifically as the check for whether recovery is still in progress. ([PostgreSQL][1])

---

### Check 4 — Replication

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

You normally want replicas showing:

```text
state = streaming
```

---

### Check 5 — DCS / etcd

```bash
etcdctl \
  --endpoints=http://192.168.10.11:2379,http://192.168.10.12:2379,http://192.168.10.13:2379 \
  endpoint health
```

These five checks already answer a large percentage of Patroni incidents.

---

# Scenario 1 — `patronictl list` shows no leader

Imagine:

```text
pg01    Replica
pg02    Replica
pg03    Replica

Leader = none
```

This is serious.

Don't immediately run:

```bash
patronictl failover
```

First determine why no leader was elected.

Check etcd:

```bash
etcdctl \
  --endpoints=http://192.168.10.11:2379,http://192.168.10.12:2379,http://192.168.10.13:2379 \
  endpoint health
```

Then check Patroni logs:

```bash
journalctl -u patroni -n 200
```

Look for:

```text
DCS connection errors
leader lock
cannot acquire lock
PostgreSQL startup failure
replication problems
candidate not healthy
timeline issues
```

Also:

```bash
curl http://localhost:8008/
```

on each surviving node.

The cause might be:

```text
etcd lost quorum
network partition
all replicas too unhealthy
Patroni communication failure
PostgreSQL failure
manual configuration problem
```

A forced/manual failover is an option only after you've identified a suitable target. Current Patroni supports `patronictl failover` when there is no leader, but requires explicitly choosing the new leader in that situation. ([Patroni][2])

---

# Scenario 2 — Replica lag is growing

Suppose:

```text
pg01     Leader
pg02     Replica    Lag 0 MB
pg03     Replica    Lag 20 GB
```

First question:

> Is pg03 receiving WAL slowly or receiving it but replaying slowly?

On `pg03`:

```sql
SELECT
    pg_last_wal_receive_lsn(),
    pg_last_wal_replay_lsn();
```

PostgreSQL exposes these separately: one represents WAL received through streaming replication and the other represents WAL already replayed. ([PostgreSQL][1])

You can also check:

```sql
SELECT now() - pg_last_xact_replay_timestamp();
```

Then check replication processes:

```sql
SELECT *
FROM pg_stat_wal_receiver;
```

Possible causes include:

```text
network throughput
high WAL generation
slow replica disk
CPU pressure
long-running standby queries
replay conflicts
WAL receiver problems
storage latency
```

Don't assume every lag problem is a network problem.

---

# Scenario 3 — PostgreSQL is running but Patroni says something is wrong

You check:

```bash
ps aux | grep postgres
```

and PostgreSQL is clearly alive.

But:

```bash
patronictl list
```

shows the node unavailable or unhealthy.

That means you need to separate:

```text
PostgreSQL process health

from

Patroni membership/HA health
```

Check Patroni:

```bash
systemctl status patroni
```

and:

```bash
journalctl -u patroni -n 200
```

Then:

```bash
curl http://localhost:8008/
```

Also verify the node can reach the DCS:

```bash
curl http://192.168.10.11:2379/health
```

or use `etcdctl`.

Potential problem:

```text
PostgreSQL running
       +
Patroni unable to access etcd
```

That is not a healthy HA state.

---

# Scenario 4 — etcd loses one member

Our DCS:

```text
etcd01
etcd02
etcd03
```

Suppose:

```text
etcd01 UP
etcd02 UP
etcd03 DOWN
```

You still have:

```text
2 / 3
```

so quorum exists.

Patroni operations should generally remain possible.

But treat this as degraded HA:

```text
Current failure tolerance = reduced
```

Repair the failed etcd node rather than leaving it broken indefinitely.

---

# Scenario 5 — etcd loses quorum

Now:

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

This is much more serious.

Do **not** begin changing etcd membership or deleting DCS directories randomly.

First determine:

```text
Are nodes actually dead?

Or is this a network partition?
```

Check individual endpoints:

```bash
etcdctl endpoint status \
  --endpoints=http://etcd01:2379,http://etcd02:2379,http://etcd03:2379 \
  --write-out=table
```

and health:

```bash
etcdctl endpoint health \
  --endpoints=http://etcd01:2379,http://etcd02:2379,http://etcd03:2379
```

Think:

```text
DCS unavailable
      |
      v
Patroni cannot safely coordinate leadership
```

This is exactly why etcd health is part of database HA monitoring.

---

# Scenario 6 — Primary failed, but automatic failover didn't happen

Suppose:

```text
pg01 DOWN
pg02 REPLICA
pg03 REPLICA

no new leader
```

Don't immediately conclude:

> Patroni is broken.

Check:

```text
1. DCS quorum
2. Patroni processes
3. network connectivity
4. replication state
5. lag
6. candidate eligibility
7. node tags/configuration
8. timeline
```

Start with:

```bash
patronictl list
```

Then:

```bash
journalctl -u patroni -n 200
```

On replicas:

```sql
SELECT pg_is_in_recovery();
```

And:

```sql
SELECT *
FROM pg_stat_wal_receiver;
```

Check Patroni configuration for things such as:

```yaml
tags:
  nofailover: true
```

A node intentionally marked unsuitable for failover shouldn't be expected to become primary.

---

# Scenario 7 — Old primary won't rejoin

This is very common.

Starting:

```text
pg01 PRIMARY
pg02 REPLICA
```

Failure:

```text
pg01 X

pg02 promoted
```

Now:

```text
pg02 PRIMARY
```

Later `pg01` comes back.

But:

```text
pg01 stopped
```

or:

```text
pg01 start failed
```

First:

```bash
journalctl -u patroni -n 200
```

Search for:

```text
timeline
rewind
WAL
replication
authentication
restore_command
checkpoint
```

The old primary may have diverged.

Conceptually:

```text
Timeline 1
----------------------+
                      \
                       \
Timeline 2              ---------------->
                         pg02
```

`pg01` belongs to the old history.

---

# Scenario 8 — `pg_rewind` fails

Remember:

```yaml
use_pg_rewind: true
```

`pg_rewind` is designed exactly for synchronizing a PostgreSQL data directory whose timeline diverged after failover. ([PostgreSQL][3])

But it has requirements.

For example, the target cluster needs either:

```text
data checksums enabled

OR

wal_log_hints = on
```

and `full_page_writes` must be enabled. ([PostgreSQL][3])

Also, required WAL around the divergence point must be available either locally or through WAL archive recovery. ([PostgreSQL][3])

If rewind fails, don't repeatedly force it.

The PostgreSQL documentation warns that after a failed `pg_rewind`, the target data directory may no longer be recoverable and recommends taking a new backup/rebuilding it. ([PostgreSQL][3])

This is where:

```bash
patronictl reinit
```

may become appropriate.

---

# Scenario 9 — Reinitialize a bad replica

Suppose:

```text
pg01 PRIMARY
pg02 REPLICA healthy
pg03 REPLICA broken
```

If `pg03` can't be repaired or rewound, rebuilding it may be the cleanest option.

First verify:

```bash
patronictl list
```

Then:

```bash
patronictl reinit pg-ha-cluster pg03
```

Check exact syntax for your version:

```bash
patronictl reinit --help
```

Conceptually:

```text
pg03 bad copy
      |
      v
remove/rebuild replica data
      |
      v
copy from healthy source
      |
      v
start PostgreSQL
      |
      v
streaming replica
```

### Critical warning

Never casually reinitialize the primary.

Before running `reinit`, verify:

```text
cluster
member
current role
healthy source
backup status
```

twice.

---

# Scenario 10 — Replication slot is filling the disk

Suppose `pg03` goes offline.

But the primary retains WAL for its replication slot:

```text
pg03 DOWN
    |
slot still exists
    |
WAL retained
    |
pg_wal grows
```

Check replication slots:

```sql
SELECT
    slot_name,
    slot_type,
    active,
    restart_lsn
FROM pg_replication_slots;
```

Check WAL directory:

```bash
du -sh $PGDATA/pg_wal
```

And disk:

```bash
df -h
```

A critical problem might look like:

```text
Filesystem 95%
```

This can become more serious than the original replica failure because a full PostgreSQL filesystem can affect the primary.

Don't blindly drop the slot.

First determine:

```text
Which node owns it?

Is the replica coming back?

Can it be rebuilt?

How much WAL is being retained?
```

---

# Scenario 11 — HAProxy shows all primary backends DOWN

HAProxy shows:

```text
PRIMARY backend

pg01 DOWN
pg02 DOWN
pg03 DOWN
```

Don't immediately restart HAProxy.

Remember the health check:

```text
HAProxy
   |
   | GET /primary
   v
Patroni :8008
```

Test directly:

```bash
curl -i http://pg01:8008/primary
curl -i http://pg02:8008/primary
curl -i http://pg03:8008/primary
```

Then:

```bash
patronictl list
```

Possible findings:

### No Patroni leader

```text
pg01 Replica
pg02 Replica
pg03 Replica
```

Then HAProxy is behaving correctly.

There is genuinely no primary.

### Patroni leader exists but HAProxy can't reach port 8008

Then investigate:

```text
firewall
routing
Patroni REST listen address
HAProxy configuration
network ACL
```

### Patroni API works but health check is wrong

Check:

```text
GET /primary
```

versus whatever path HAProxy actually uses.

Always test the same endpoint manually.

---

# Scenario 12 — Application can't connect but Patroni is healthy

You see:

```text
pg01 Leader running
pg02 Replica streaming
pg03 Replica streaming
```

Looks perfect.

Application still says:

```text
connection refused
```

Now move up a layer.

Check HAProxy:

```bash
systemctl status haproxy
```

Then:

```bash
ss -lntp | grep 5000
```

Test:

```bash
nc -vz haproxy01 5000
```

Then try:

```bash
psql -h haproxy01 -p 5000 -U postgres
```

Potential issue could be:

```text
HAProxy down
firewall
DNS
VIP problem
PgBouncer
application password
pg_hba.conf
TLS certificate
```

Patroni isn't responsible for every PostgreSQL connection failure.

---

# Scenario 13 — Two servers appear to be primary

This is an emergency-level condition.

Suppose:

```text
pg01
SELECT pg_is_in_recovery();
false
```

and:

```text
pg02
SELECT pg_is_in_recovery();
false
```

Do not start issuing writes or arbitrarily restarting nodes.

First determine which node owns Patroni leadership:

```bash
patronictl list
```

Check DCS state.

Then Patroni APIs:

```bash
curl http://pg01:8008/
curl http://pg02:8008/
```

Check timelines and WAL state.

You may be dealing with:

```text
network partition
manually started old primary
Patroni bypass
DCS failure
incorrect recovery state
```

The biggest mistake is allowing applications to write to both sides.

Think:

```text
STOP additional divergence
        |
        v
identify authoritative primary
        |
        v
understand timelines
        |
        v
repair/reinitialize old branch
```

This is a case where escalation to an experienced PostgreSQL/Patroni administrator is appropriate if ownership of the latest consistent data isn't obvious.

---

# 14. Useful PostgreSQL troubleshooting queries

### Who am I?

```sql
SELECT
    inet_server_addr(),
    inet_server_port(),
    pg_is_in_recovery();
```

---

### Replicas connected to primary

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

---

### WAL receiver on replica

```sql
SELECT *
FROM pg_stat_wal_receiver;
```

---

### Receive vs replay position

```sql
SELECT
    pg_last_wal_receive_lsn(),
    pg_last_wal_replay_lsn();
```

---

### Replay delay

```sql
SELECT
    now() - pg_last_xact_replay_timestamp();
```

---

### Replication slots

```sql
SELECT
    slot_name,
    slot_type,
    active,
    restart_lsn
FROM pg_replication_slots;
```

These queries tell you more than simply:

```text
replication = broken
```

They help identify **where** replication stopped.

---

# 15. Useful Linux commands

Disk:

```bash
df -h
```

PostgreSQL WAL size:

```bash
du -sh /var/lib/patroni/data/pg_wal
```

Memory:

```bash
free -m
```

Processes:

```bash
ps -ef | grep -E 'postgres|patroni'
```

Ports:

```bash
ss -lntp
```

Patroni:

```bash
systemctl status patroni
```

Logs:

```bash
journalctl -u patroni -n 200
```

Live logs:

```bash
journalctl -u patroni -f
```

These may sound basic, but many "Patroni problems" eventually turn out to be:

```text
disk full
DNS failure
filesystem permissions
port blocked
OOM
service killed
network failure
```

---

# 16. Things I would avoid during an incident

These commands aren't inherently wrong, but they're dangerous when used without understanding cluster state:

```bash
systemctl start postgresql
```

```bash
systemctl stop postgresql
```

```bash
rm -rf $PGDATA
```

```bash
patronictl failover
```

```bash
patronictl reinit
```

```bash
etcdctl member remove ...
```

or manually changing PostgreSQL recovery state.

Before performing any destructive operation, know:

```text
Current leader
Current timeline
Replication state
DCS health
Backup status
Which node contains newest data
```

---

# 17. A practical incident workflow

This is the workflow I'd like you to remember.

```text
ALERT
  |
  v
1. patronictl list
  |
  v
2. Identify leader
  |
  v
3. Check Patroni API
  |
  v
4. Check PostgreSQL role
  |
  v
5. Check replication
  |
  v
6. Check DCS quorum
  |
  v
7. Check OS / network / storage
  |
  v
8. Read logs
  |
  v
9. Identify root cause
  |
  v
10. Choose safest corrective action
```

Not:

```text
Alert
 |
 v
restart everything
```

---

# 18. Troubleshooting cheat sheet

| Symptom                  | First checks                                  |
| ------------------------ | --------------------------------------------- |
| No leader                | `patronictl list`, etcd health, Patroni logs  |
| Replica lag              | `pg_stat_replication`, WAL receive/replay LSN |
| Replica stopped          | Patroni logs, PostgreSQL logs                 |
| Failover didn't occur    | DCS quorum, candidate health/tags             |
| Old primary won't rejoin | timeline, Patroni logs, `pg_rewind`           |
| `pg_rewind` fails        | checksums/wal_log_hints, WAL availability     |
| Replica badly broken     | Consider `patronictl reinit`                  |
| `pg_wal` huge            | replication slots, inactive replicas          |
| Disk almost full         | WAL, logs, filesystem usage                   |
| HAProxy all DOWN         | Patroni `/primary`, leader status             |
| App can't connect        | HAProxy/PgBouncer/network/authentication      |
| Two apparent primaries   | Stop divergence and investigate DCS/timelines |

---

# 19. Production interview-style scenario

Someone tells you:

> "The primary died but Patroni didn't fail over. What do you do?"

A strong answer would be:

```text
First I check patronictl list to understand the cluster state.

Then I verify DCS/etcd quorum because Patroni requires a healthy
coordination layer to safely elect a new leader.

I check the Patroni logs on the replicas to see why they're not
eligible or unable to obtain leadership.

I verify PostgreSQL replication state and lag, and confirm whether
the replicas are healthy.

I also check tags such as nofailover and any synchronous replication
requirements.

I would not immediately force a failover until I know why automatic
failover didn't happen and which replica contains the safest data.
```

That answer shows much more Patroni understanding than:

> "I run `patronictl failover`."

---

# 20. Another production scenario

Someone asks:

> "pg01 was primary, failed, pg02 became primary, and now pg01 won't start. What do you check?"

Think:

```text
Failover happened
       |
       v
Timeline changed
       |
       v
Old pg01 may have diverged
       |
       v
Can Patroni use pg_rewind?
       |
       +--> YES -> rewind/rejoin
       |
       +--> NO
              |
              v
           reinitialize replica
```

`pg_rewind` is specifically designed for bringing an old primary back as a standby after timeline divergence, and PostgreSQL notes that it can be much faster than recreating the entire server when only a fraction of blocks changed. ([PostgreSQL][3])

---

## What you should know after Lesson 10

At this point you should be comfortable with this distinction:

```text
patronictl
   |
   +--> What does Patroni believe?


PostgreSQL SQL queries
   |
   +--> What is PostgreSQL actually doing?


etcdctl
   |
   +--> Is distributed coordination healthy?


HAProxy
   |
   +--> Can clients reach the correct role?


Linux commands
   |
   +--> Is the underlying server healthy?
```

That's the foundation of supporting Patroni in production.

## Next — Lesson 11: Real Patroni Incident Lab

The next lesson should be completely hands-on. I'll give you **10 broken-cluster scenarios** like an actual production support ticket. You diagnose each one from command output before seeing the solution—for example **replica lagging 25 GB, etcd quorum lost, no Patroni leader, old primary unable to rewind, disk at 98%, HAProxy has all backends DOWN, or a failed replica requiring reinitialization**.

That will help you move from knowing Patroni concepts to actually troubleshooting them.

[1]: https://www.postgresql.org/docs/17/functions-admin.html "PostgreSQL: Documentation: 17: 9.28. System Administration Functions"
[2]: https://patroni.readthedocs.io/_/downloads/en/master/pdf/ "Patroni Documentation"
[3]: https://www.postgresql.org/docs/current/app-pgrewind.html "PostgreSQL: Documentation: 18: pg_rewind"
