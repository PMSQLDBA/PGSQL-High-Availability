# Lesson 12 — Patroni Commands & Daily DBA Operations

This lesson is about the commands you’ll use most often as a DBA or production support engineer.

The goal is not to memorize every option. It is to know **what each command changes, what it does not change, and when it is safe to use**.

## 1. `patronictl list`

This is your first command in almost every incident or maintenance window.

```bash
patronictl -c /etc/patroni/patroni.yml list
```

Typical output:

```text
+ Cluster: prod-db ------------------------------+
| Member | Host     | Role    | State     | Lag |
+--------+----------+---------+-----------+-----+
| pg01   | 10.0.1.11| Leader  | running   |     |
| pg02   | 10.0.1.12| Replica | streaming |   0 |
| pg03   | 10.0.1.13| Replica | streaming |   0 |
+--------+----------+---------+-----------+-----+
```

Use it to answer:

```text
Who is primary?
Which replicas are healthy?
Is there lag?
Is any node stopped?
Did leadership change?
```

This is usually read-only and safe.

---

# 2. `patronictl show-config`

This shows the **dynamic cluster configuration** stored in the DCS.

```bash
patronictl -c /etc/patroni/patroni.yml show-config
```

You may see settings like:

```yaml
loop_wait: 10
retry_timeout: 10
ttl: 30

postgresql:
  parameters:
    max_connections: 500
    max_wal_senders: 10

  use_pg_rewind: true
  use_slots: true
```

This is important because there are two configuration sources:

```text
Local patroni.yml
       |
       +--> node-specific configuration

DCS dynamic config
       |
       +--> cluster-wide settings
```

A common beginner mistake is editing:

```text
bootstrap.dcs
```

in `patroni.yml` on an already-running cluster and expecting the live cluster configuration to change.

Usually, dynamic cluster settings should be inspected and changed through Patroni's cluster configuration mechanisms.

---

# 3. `patronictl edit-config`

This changes the dynamic configuration.

```bash
patronictl -c /etc/patroni/patroni.yml edit-config
```

Patroni opens an editor.

Suppose:

```yaml
postgresql:
  parameters:
    max_connections: 500
```

You change it to:

```yaml
postgresql:
  parameters:
    max_connections: 700
```

Patroni stores the new configuration in the DCS.

But this does **not necessarily mean PostgreSQL immediately starts using 700 connections**.

Why?

Because some PostgreSQL parameters require a restart.

So think:

```text
edit-config
     |
     v
configuration changes
     |
     +--> some reload immediately
     |
     +--> some require restart
```

Always know whether a PostgreSQL parameter is reloadable or restart-required before changing it.

---

# 4. Pending restart

After changing a restart-required parameter, `patronictl list` may show something equivalent to:

```text
Pending restart
```

Conceptually:

```text
Current PostgreSQL
max_connections = 500

Desired config
max_connections = 700

Status:
Pending restart
```

That does not necessarily mean something is broken.

It means:

> PostgreSQL needs a restart before the new setting becomes active.

This is very common during maintenance.

---

# 5. `patronictl reload`

A reload tells Patroni/PostgreSQL to reload applicable configuration without restarting the database.

```bash
patronictl -c /etc/patroni/patroni.yml reload prod-db
```

Depending on your Patroni version, you'll select cluster/member options accordingly.

Use reload for settings that PostgreSQL can reload dynamically.

Conceptually:

```text
Config changed
     |
     v
reload
     |
     v
PostgreSQL re-reads configuration
     |
     v
No database restart
```

This is preferable to restarting when a restart isn't needed.

---

# 6. `patronictl restart`

This is much more disruptive.

```bash
patronictl -c /etc/patroni/patroni.yml restart prod-db pg03
```

Conceptually:

```text
pg03 PostgreSQL
      |
      v
     STOP
      |
      v
     START
```

This will interrupt connections to that PostgreSQL member.

For a replica, impact may be limited.

For a primary, impact can be much more serious.

Before restarting anything:

```text
1. patronictl list
2. Confirm member
3. Confirm role
4. Check replication
5. Understand application impact
```

Never assume:

```text
pg03 = replica
```

just because it was a replica yesterday.

Roles can change.

---

# 7. Safe replica restart workflow

Suppose:

```text
pg01 PRIMARY
pg02 REPLICA
pg03 REPLICA
```

and `pg03` needs a restart.

First:

```bash
patronictl -c /etc/patroni/patroni.yml list
```

Confirm:

```text
pg03 = Replica
```

Then:

```bash
patronictl -c /etc/patroni/patroni.yml restart prod-db pg03
```

Afterward:

```bash
patronictl -c /etc/patroni/patroni.yml list
```

Verify:

```text
pg03
Replica
streaming
lag acceptable
```

Then check PostgreSQL replication from the leader if needed:

```sql
SELECT
    application_name,
    state,
    sync_state
FROM pg_stat_replication;
```

---

# 8. Restarting the primary

Suppose `pg01` is primary and needs restart-required PostgreSQL changes.

You have two choices conceptually.

### Option A — Restart current primary

This creates a service interruption.

```text
pg01 PRIMARY
    |
 restart
    |
 temporary outage
```

### Option B — Switchover first

Usually more controlled:

```text
BEFORE

pg01 PRIMARY
pg02 REPLICA
pg03 REPLICA
```

Perform:

```text
pg01 -> pg02
```

Then:

```text
pg01 REPLICA
pg02 PRIMARY
pg03 REPLICA
```

Now restart `pg01`.

This is often a better maintenance pattern.

---

# 9. `patronictl switchover`

We've already practiced this.

Use when the current leader is healthy and you're intentionally moving leadership.

```bash
patronictl -c /etc/patroni/patroni.yml switchover
```

Typical use cases:

```text
OS patching
database maintenance
hardware maintenance
planned restart
data-center work
testing
```

Think:

```text
Healthy primary
      |
planned transition
      |
      v
Healthy replica promoted
```

This is normally safer than intentionally crashing the primary.

---

# 10. `patronictl failover`

This is different.

```bash
patronictl -c /etc/patroni/patroni.yml failover
```

A failover is generally associated with an unhealthy/missing leader or a deliberate forced promotion.

It is more dangerous because you may be making a decision under failure conditions.

Before manually invoking failover, understand:

```text
Is old primary really unavailable?
Is DCS healthy?
Which replica has latest WAL?
Is candidate eligible?
What is replication lag?
Could old primary still accept writes?
```

Don't treat:

```bash
patronictl failover
```

as a normal troubleshooting command.

---

# 11. Switchover vs failover again

This distinction should now be automatic in your head.

```text
SWITCHOVER
-----------
planned
healthy leader
controlled transition


FAILOVER
--------
leader unavailable/problem
emergency or failure condition
potentially higher risk
```

If someone says:

> We need to patch the primary tonight.

Think:

```text
switchover
```

If someone says:

> Primary server crashed unexpectedly.

Think:

```text
automatic failover / failover investigation
```

---

# 12. `patronictl reinit`

Use this to rebuild a broken replica.

Example:

```bash
patronictl -c /etc/patroni/patroni.yml reinit prod-db pg03
```

Conceptually:

```text
pg03 broken
   |
   v
discard/recreate replica
   |
   v
copy from healthy cluster source
   |
   v
streaming replica
```

This is destructive to the target member's local PostgreSQL data directory.

So before running it, verify:

```text
Cluster name
Member name
Member role
Current leader
Backup status
Healthy source exists
```

Never casually reinitialize the leader.

---

# 13. `patronictl remove`

This is much more serious.

It removes Patroni cluster information from the DCS.

You might see:

```bash
patronictl remove prod-db
```

Do not use this as a normal node-removal command.

Think:

```text
remove cluster metadata from DCS
```

not:

```text
remove one broken replica
```

This command can be destructive to the Patroni cluster state.

Treat it as an administrative recovery/decommissioning operation, not daily maintenance.

---

# 14. `patronictl pause`

Pause mode is useful during certain maintenance operations.

```bash
patronictl -c /etc/patroni/patroni.yml pause
```

Conceptually:

```text
Normal mode:

Patroni actively manages HA
and may take automated actions


Paused mode:

Patroni reduces/suspends
certain automatic HA actions
```

This can be useful when performing maintenance where Patroni's automatic intervention could interfere.

But pause mode is dangerous if forgotten.

If the cluster is paused and the primary fails, normal HA behavior may not occur as expected.

Always verify:

```bash
patronictl list
```

and cluster state after maintenance.

---

# 15. `patronictl resume`

After maintenance:

```bash
patronictl -c /etc/patroni/patroni.yml resume
```

This returns Patroni to normal active HA management.

A safe maintenance pattern is:

```text
Understand why pause is needed
      |
      v
pause
      |
      v
perform controlled maintenance
      |
      v
verify PostgreSQL
      |
      v
resume
      |
      v
verify Patroni cluster
```

Do not use pause just because it feels safer.

Often normal Patroni-aware operations such as switchover are better.

---

# 16. Member tags

Patroni tags change how a node participates in HA.

Examples include:

```yaml
tags:
  nofailover: true
```

Meaning:

```text
Do not automatically promote this node
```

Another common concept is excluding nodes from load-balanced replica traffic depending on environment/configuration.

Tags are useful for:

```text
reporting replicas
DR replicas
special-purpose replicas
maintenance
delayed replicas
```

But they can also create surprises.

Imagine:

```text
pg01 PRIMARY

pg02 REPLICA
nofailover: true

pg03 REPLICA
nofailover: true
```

Then `pg01` dies.

You technically have:

```text
2 replicas
```

but:

```text
0 eligible failover candidates
```

Always include tags when reviewing HA readiness.

---

# 17. Change configuration safely

A production configuration workflow should look like:

```text
Requirement
    |
    v
Determine setting
    |
    v
Check whether reload/restart required
    |
    v
Review Patroni implications
    |
    v
edit-config
    |
    v
Verify config
    |
    v
reload OR planned restart
    |
    v
Verify cluster
```

Not:

```text
Edit YAML everywhere
restart all servers
hope
```

---

# 18. Node-specific vs cluster-wide settings

This distinction is important.

Some configuration belongs to the individual member:

```text
name
connect_address
data_dir
REST API address
DCS endpoint information
```

These are generally in local `patroni.yml`.

Other settings are cluster-wide dynamic configuration:

```text
ttl
loop_wait
retry_timeout
PostgreSQL parameters
synchronous mode settings
```

Think:

```text
Local YAML
   |
   +--> Who am I?


Dynamic DCS config
   |
   +--> How should the cluster behave?
```

This mental model helps avoid inconsistent configurations across nodes.

---

# 19. Check differences between nodes

If one node behaves strangely, compare its local Patroni configuration with the others.

Look at:

```text
scope
namespace
name
etcd endpoints
REST API
PostgreSQL address
data_dir
authentication
tags
```

A simple typo like:

```yaml
scope: prod-db2
```

instead of:

```yaml
scope: prod-db
```

can cause a server to effectively belong to another Patroni cluster.

---

# 20. Daily DBA health checks

A reasonable daily Patroni review could include:

```bash
patronictl -c /etc/patroni/patroni.yml list
```

Then review:

```text
Leader present
All expected replicas present
Replication state
Lag
Pending restart
Timeline consistency
```

On the PostgreSQL primary:

```sql
SELECT
    application_name,
    client_addr,
    state,
    sync_state
FROM pg_stat_replication;
```

Check replication slots:

```sql
SELECT
    slot_name,
    active,
    restart_lsn
FROM pg_replication_slots;
```

Check disk:

```bash
df -h
```

Especially:

```text
PGDATA
pg_wal
backup filesystem
log filesystem
```

You want to discover problems while you're still in:

```text
degraded but available
```

rather than:

```text
full outage
```

---

# 21. OS patching workflow

Suppose you need to patch all three nodes.

Starting:

```text
pg01 PRIMARY
pg02 REPLICA
pg03 REPLICA
```

A controlled workflow might be:

```text
1. Check cluster health

2. Patch/reboot pg03

3. Verify pg03 rejoins

4. Patch/reboot pg02

5. Verify pg02 rejoins

6. Switchover pg01 -> pg02

7. Verify pg02 primary

8. Patch/reboot pg01

9. Verify pg01 rejoins

10. Final cluster validation
```

That maintains a healthy replica wherever possible.

---

# 22. PostgreSQL minor upgrade workflow

The same basic thinking applies.

Before touching the primary:

```text
Verify replicas
       |
       v
upgrade one replica
       |
       v
verify
       |
       v
upgrade another replica
       |
       v
switchover
       |
       v
upgrade old primary
```

Exact PostgreSQL package/upgrade procedures depend on version and operating system, but Patroni's role is to help coordinate which server is primary while you perform maintenance.

---

# 23. PostgreSQL major upgrades are different

Do not treat a PostgreSQL major upgrade like:

```text
restart required
```

For example:

```text
PostgreSQL 16 -> 17
```

is not the same type of operation as changing:

```text
max_connections
```

Major upgrades require a proper upgrade strategy such as:

```text
pg_upgrade
logical replication
blue/green cluster
dump/restore
```

depending on architecture and downtime requirements.

Patroni doesn't magically remove the need for a PostgreSQL major upgrade plan.

---

# 24. `patronictl` command safety levels

Here's a useful mental ranking:

| Command           | Risk level  | Why                                |
| ----------------- | ----------- | ---------------------------------- |
| `list`            | Very low    | Read-only                          |
| `show-config`     | Very low    | Read-only                          |
| `reload`          | Low         | No database restart                |
| `edit-config`     | Medium      | Changes cluster behavior           |
| `restart` replica | Medium      | Member interruption                |
| `switchover`      | Medium      | Controlled primary move            |
| `pause/resume`    | Medium-high | Changes HA automation              |
| `restart` primary | High        | Application impact                 |
| `failover`        | High        | Primary role change during failure |
| `reinit`          | High        | Rebuilds target data               |
| `remove`          | Very high   | Removes Patroni cluster metadata   |

The exact risk depends on the situation, but this is a good operational mindset.

---

# 25. Never trust the server name alone

Imagine your monitoring ticket says:

```text
Restart pg03
```

Before you do anything:

```bash
patronictl list
```

Because Patroni could currently show:

```text
pg03 Leader
```

Roles move.

Server identity is static.

Database role is dynamic.

That distinction is fundamental in HA environments.

---

# 26. Maintenance example

Change request:

> Increase PostgreSQL `max_connections` from 300 to 500.

Think through it:

```text
1. Does parameter require restart?
        |
       YES

2. Change dynamic configuration.

3. patronictl list
   shows Pending restart.

4. Restart replicas one at a time.

5. Verify each returns.

6. Switchover primary.

7. Restart old primary.

8. Validate configuration everywhere.
```

That's much safer than restarting all three nodes at once.

---

# 27. Validate configuration from PostgreSQL

Don't assume Patroni configuration means PostgreSQL actually applied it.

For example:

```sql
SHOW max_connections;
```

You can also inspect whether settings are pending at PostgreSQL level using appropriate system views depending on what you're checking.

The key rule is:

```text
Config intended
      !=
Config actually active
```

Always verify.

---

# 28. Change management mindset

For production Patroni changes, document:

```text
Current leader

Expected candidate

Current cluster health

Change being made

Expected Patroni behavior

Application impact

Rollback plan

Validation commands
```

For example:

```text
Before:
pg01 Leader
pg02 Replica
pg03 Replica

Action:
Switchover to pg02

Expected:
pg02 becomes Leader
pg01 becomes Replica

Validation:
patronictl list
pg_is_in_recovery()
pg_stat_replication
application connectivity
```

This makes operations predictable.

---

# 29. Daily command cheat sheet

Cluster state:

```bash
patronictl -c /etc/patroni/patroni.yml list
```

Dynamic configuration:

```bash
patronictl -c /etc/patroni/patroni.yml show-config
```

Edit cluster config:

```bash
patronictl -c /etc/patroni/patroni.yml edit-config
```

Reload:

```bash
patronictl -c /etc/patroni/patroni.yml reload <cluster>
```

Restart:

```bash
patronictl -c /etc/patroni/patroni.yml restart <cluster> <member>
```

Planned primary move:

```bash
patronictl -c /etc/patroni/patroni.yml switchover
```

Emergency/manual promotion:

```bash
patronictl -c /etc/patroni/patroni.yml failover
```

Rebuild replica:

```bash
patronictl -c /etc/patroni/patroni.yml reinit <cluster> <member>
```

Maintenance pause:

```bash
patronictl -c /etc/patroni/patroni.yml pause
```

Resume:

```bash
patronictl -c /etc/patroni/patroni.yml resume
```

Always check your installed version's syntax with:

```bash
patronictl --help
```

and:

```bash
patronictl <command> --help
```

because available flags can vary by Patroni release.

---

# 30. What I want you to remember

For day-to-day Patroni support:

```text
Observe first
     |
     v
Understand role
     |
     v
Understand replication
     |
     v
Understand DCS
     |
     v
Make smallest necessary change
     |
     v
Verify afterward
```

The three commands you'll probably use most are:

```bash
patronictl list
```

```bash
patronictl show-config
```

```bash
patronictl switchover
```

And the three commands you should treat with extra care are:

```bash
patronictl failover
patronictl reinit
patronictl remove
```

### Next — Lesson 13: Patroni Monitoring & Alerting

The next useful lesson is how to monitor a Patroni environment properly: **which PostgreSQL, Patroni, etcd, HAProxy, OS, disk, WAL, replication-slot, and failover metrics matter; what should trigger warning vs critical alerts; and how Prometheus/Grafana-style monitoring fits into the architecture.**
