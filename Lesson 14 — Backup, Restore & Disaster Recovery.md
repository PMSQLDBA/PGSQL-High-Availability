# Lesson 14 — Backup, Restore & Disaster Recovery

This lesson covers something very important:

> **Patroni gives you High Availability. It does not give you Disaster Recovery.**

A healthy 3-node Patroni cluster can still lose data if the wrong SQL is executed, corruption is replicated, or the whole site is lost.

## 1. HA vs Backup vs DR

Keep these three concepts separate:

```text
HIGH AVAILABILITY
-----------------
Problem:
Server/process fails

Solution:
Patroni + PostgreSQL replication

Example:
pg01 dies
   |
pg02 promoted


BACKUP / RECOVERY
-----------------
Problem:
Data is deleted/corrupted

Solution:
Backup + archived WAL

Example:
DROP TABLE customers
       |
       v
Restore previous state


DISASTER RECOVERY
-----------------
Problem:
Entire cluster/site is lost

Solution:
Offsite backup / DR environment

Example:
Data center unavailable
       |
       v
Recover somewhere else
```

Replication alone cannot protect you from logical mistakes.

For example:

```sql
DROP TABLE customers;
```

PostgreSQL creates WAL for that operation.

Then:

```text
pg01 PRIMARY
DROP TABLE customers
       |
       | WAL
       v
pg02 REPLICA
table deleted
       |
       v
pg03 REPLICA
table deleted
```

All replicas can faithfully reproduce the mistake.

That is why:

```text
Replication != Backup
```

---

# 2. What a PostgreSQL backup strategy needs

For a serious Patroni environment, you normally want both:

```text
Base Backup
    +
Archived WAL
```

Together they allow:

```text
Full restore
PITR
Recovery after entire cluster loss
Replica reconstruction
```

PostgreSQL's continuous archiving model works by restoring a base backup and then replaying archived WAL. It can stop WAL replay at a chosen recovery point, which is what makes Point-in-Time Recovery possible. ([PostgreSQL][1])

---

# 3. Think of the backup as a starting point

Suppose Sunday:

```text
Sunday 01:00
FULL BACKUP
```

Then PostgreSQL produces WAL continuously:

```text
Sunday backup
     |
     v
WAL 001
WAL 002
WAL 003
WAL 004
...
WAL 950
     |
     v
Wednesday
```

If the database fails Wednesday, you don't necessarily restore only Sunday's state.

You can restore:

```text
Sunday base backup
        +
WAL 001 ... WAL 950
        |
        v
Wednesday state
```

That's the power of continuous archiving.

---

# 4. What is Point-in-Time Recovery?

Suppose someone accidentally runs:

```sql
DROP TABLE orders;
```

at:

```text
10:35:20
```

You discover the problem at:

```text
11:15
```

You don't want:

```text
11:15 state
```

because the table is already gone.

You want something like:

```text
10:35:19
```

Architecture:

```text
Base backup
     |
     v
Replay WAL
     |
     v
10:30
     |
     v
10:34
     |
     v
10:35:19
     |
    STOP
     |
     X
DROP TABLE happened at 10:35:20
```

That is **PITR — Point-in-Time Recovery**.

PostgreSQL supports recovery targets based on items such as time and named restore points. ([PostgreSQL][2])

---

# 5. Where pgBackRest fits

A very common PostgreSQL backup solution is **pgBackRest**.

Its current user guide includes commands for:

```text
backup
restore
archive-push
archive-get
check
info
expire
stanza-create
```

and supports using a separate backup repository. ([pgBackRest][3])

Conceptually:

```text
                  Patroni cluster

             pg01 PRIMARY
                  |
                  | backup
                  | WAL archive
                  v
          +----------------+
          |   pgBackRest   |
          +--------+-------+
                   |
                   v
          Backup Repository
           /             \
       Backups        Archived WAL
```

---

# 6. Use a separate backup repository

Don't store your only backups on the PostgreSQL primary.

Bad:

```text
pg01
 |
 +-- PostgreSQL
 |
 +-- Backup
```

If `pg01` storage dies:

```text
database = gone
backup   = gone
```

Better:

```text
PostgreSQL cluster
       |
       v
Separate pgBackRest repository
```

Better still for critical systems:

```text
Primary site
     |
     v
Backup repository
     |
     +----> Offsite/object storage copy
```

The backup needs a failure domain different from the database.

---

# 7. pgBackRest terminology — stanza

You'll hear:

```text
stanza
```

Think of a stanza as pgBackRest's configuration definition for a PostgreSQL cluster.

For example:

```ini
[prod-db]
pg1-path=/var/lib/patroni/data
```

Then operations reference:

```bash
pgbackrest --stanza=prod-db ...
```

For example:

```bash
pgbackrest --stanza=prod-db stanza-create
```

Current pgBackRest documentation uses `stanza-create` to initialize the required repository metadata for a configured PostgreSQL cluster. ([pgBackRest][3])

---

# 8. Check pgBackRest configuration

Before taking backups:

```bash
sudo -u postgres pgbackrest \
  --stanza=prod-db \
  check
```

Think:

```text
Configuration
      |
      v
Repository reachable?
      |
      v
WAL archiving working?
      |
      v
PostgreSQL accessible?
```

Don't schedule backups before this works.

---

# 9. WAL archiving

PostgreSQL generates WAL locally:

```text
$PGDATA/pg_wal
```

But local WAL is not your durable archive.

You want WAL copied somewhere else.

Conceptually:

```text
PostgreSQL
    |
creates WAL
    |
    v
pg_wal
    |
    | archive_command
    v
pgBackRest archive-push
    |
    v
Backup Repository
```

A pgBackRest configuration commonly integrates an `archive-push` command with PostgreSQL, while recovery uses `archive-get`. The current pgBackRest guide documents both flows. ([pgBackRest][3])

Conceptually:

```text
archive-push
PostgreSQL ---> Repository

archive-get
Repository ---> PostgreSQL
```

---

# 10. Why WAL archiving must be monitored

Suppose:

```text
01:00 backup successful
```

but WAL archiving breaks at:

```text
02:00
```

Then at 15:00 the database crashes.

You might think:

> We have today's backup.

But your recovery chain could only cover:

```text
01:00 backup
   +
WAL until 02:00
```

You may not be able to recover to 14:59.

PostgreSQL requires an unbroken WAL sequence from the relevant backup point for continuous-archive recovery. ([PostgreSQL][1])

So monitor:

```text
Last archived WAL
Archive failures
Archive backlog
Repository availability
```

---

# 11. Backup types

With pgBackRest you'll commonly encounter:

```text
Full
Differential
Incremental
```

Think:

### Full

```text
Complete database backup
```

Largest and typically slowest, but independent foundation.

### Differential

```text
Changes since latest FULL
```

### Incremental

```text
Changes since previous backup
```

Example schedule:

```text
Sunday     FULL
Monday     DIFFERENTIAL
Tuesday    INCREMENTAL
Wednesday  INCREMENTAL
Thursday   DIFFERENTIAL
Friday     INCREMENTAL
Saturday   INCREMENTAL
```

The right schedule depends on:

```text
database size
change rate
backup window
restore time
repository capacity
RPO/RTO
```

---

# 12. Taking a backup

A basic pgBackRest command might look like:

```bash
sudo -u postgres pgbackrest \
  --stanza=prod-db \
  --type=full \
  backup
```

Then inspect backups:

```bash
sudo -u postgres pgbackrest \
  --stanza=prod-db \
  info
```

Current pgBackRest exposes `backup` and `info` specifically for these purposes. ([pgBackRest][3])

You should know:

```text
Latest backup
Backup type
Start/end time
Database size
Repository size
WAL coverage
```

---

# 13. Backup from a replica

An advantage in larger environments is taking backups from a standby where appropriate.

Architecture:

```text
             pg01
            PRIMARY
              |
              | WAL
              v
             pg02
            REPLICA
              |
              | backup
              v
       pgBackRest repository
```

This can reduce backup workload on the primary.

The current pgBackRest guide explicitly documents **Backup from a Standby**. ([pgBackRest][3])

But make sure the replica used for backups is healthy and appropriately configured.

---

# 14. Normal full restore

Suppose the whole PostgreSQL data directory on one recovery server is unusable.

Conceptually:

```text
Broken PostgreSQL
      |
      v
Stop PostgreSQL/Patroni
      |
      v
Prepare empty PGDATA
      |
      v
pgBackRest restore
      |
      v
Retrieve archived WAL
      |
      v
PostgreSQL recovery
      |
      v
Consistent database
```

PostgreSQL's native continuous recovery model similarly consists of restoring the database files and then replaying archived WAL through recovery. ([PostgreSQL][2])

---

# 15. Basic pgBackRest restore

Conceptually:

```bash
sudo -u postgres pgbackrest \
  --stanza=prod-db \
  restore
```

But don't run that against a live production primary.

A restore normally requires carefully controlling:

```text
Patroni
PostgreSQL process
PGDATA
repository
recovery target
ownership
permissions
```

Restores are inherently destructive to the target data directory.

---

# 16. PITR example

Let's make this practical.

Suppose:

```text
10:00   everything healthy

10:28   important customer orders entered

10:35   DBA accidentally runs:
        DELETE FROM orders;

10:45   mistake discovered
```

Goal:

```text
Restore database to approximately:
10:34:59
```

A PITR workflow might conceptually be:

```text
Choose backup before 10:34:59
       |
       v
Restore base backup
       |
       v
Retrieve WAL from archive
       |
       v
Replay WAL
       |
       v
Stop at recovery target time
10:34:59
       |
       v
Promote restored PostgreSQL
       |
       v
Validate application data
```

Do **not** immediately overwrite your current production cluster if you're uncertain.

A much safer strategy is often:

```text
Production cluster
       |
       | untouched
       |
       +----> Separate recovery server
                    |
                    v
             Restore to 10:34:59
                    |
                    v
             Validate data
```

Then decide how to recover the required data/service.

---

# 17. Named restore points

Instead of guessing timestamps around planned risky work, PostgreSQL can create named restore points.

For example:

```sql
SELECT pg_create_restore_point('before_release_2026_08');
```

Conceptually:

```text
WAL
 |
 +---- RESTORE POINT
 |     before_release
 |
 +---- deployment begins
 |
 +---- application changes
```

If necessary, recovery can target that named point.

This can be useful before controlled major application/database changes, but it still depends on the corresponding WAL being archived.

---

# 18. Patroni makes whole-cluster recovery different

This part matters.

Suppose:

```text
pg01 destroyed
pg02 destroyed
pg03 destroyed
```

but backup repository survives.

You don't necessarily restore all three members separately from backup.

A common conceptual strategy is:

```text
Backup repository
      |
      v
Restore ONE PostgreSQL instance
      |
      v
Establish it as recovered primary
      |
      v
Bring Patroni cluster back safely
      |
      v
Clone/reinitialize pg02
      |
      v
Clone/reinitialize pg03
```

So:

```text
Restore authoritative database once

then

Rebuild replicas from authoritative primary
```

This avoids independently restoring several databases that might accidentally become conflicting primaries.

---

# 19. Patroni replica creation can integrate with backup tools

Patroni isn't limited to `pg_basebackup`.

Its replica creation mechanism supports custom methods, and Patroni documentation includes examples of using **pgBackRest** as a replica creation method. ([Patroni][4])

Conceptually:

```yaml
postgresql:

  create_replica_methods:
    - pgbackrest
    - basebackup
```

Then:

```text
Need new pg03
      |
      v
Patroni
      |
      v
Try pgBackRest restore
      |
      v
Build replica
```

This can be particularly useful for very large databases where copying everything from the current primary could place significant load on it.

---

# 20. Why DR restore should usually be isolated first

Suppose the production cluster is partially alive but corrupted.

You should be careful not to:

```text
Restore old backup
      |
      v
accidentally join it
      |
      v
live Patroni cluster
```

A recovery environment should often initially have:

```text
No application writes
No HAProxy production routing
Controlled DCS configuration
Network isolation as needed
```

Think:

```text
RESTORE
   |
   v
VALIDATE
   |
   v
DECIDE
   |
   v
CUTOVER
```

not:

```text
RESTORE
   |
   v
HOPE
```

---

# 21. What if only one table was deleted?

This is an interesting point.

Physical PITR restores the **whole PostgreSQL cluster state** to an earlier time.

Suppose:

```text
customers = fine
products  = fine
orders    = accidentally deleted
```

You may not want to rewind the entire production database to yesterday.

Instead:

```text
Backup/WAL
    |
    v
Restore separate PostgreSQL instance
to before DELETE
    |
    v
Extract needed rows/table
    |
    v
Import into production carefully
```

This is often much safer than rolling the whole production environment backward.

---

# 22. Logical backups still have value

Tools such as:

```text
pg_dump
pg_dumpall
```

are logical backup tools.

They are useful for:

```text
individual databases
schema backup
individual table recovery
migration
portable logical copies
```

But PostgreSQL's documentation makes clear that `pg_dump`/`pg_dumpall` are not file-system base backups and cannot provide the base for continuous WAL/PITR recovery. ([PostgreSQL][1])

So mature environments may use both:

```text
Physical backup + WAL
        |
        +--> disaster/PITR recovery

Logical dump
        |
        +--> object-level portability/recovery
```

---

# 23. RPO and backup design

Remember:

**RPO = how much data can we afford to lose?**

Suppose:

```text
RPO = 5 minutes
```

Then taking:

```text
one backup every 24 hours
```

doesn't meet the requirement by itself.

Continuous WAL archiving may bring your recoverable state much closer to the failure point:

```text
Base backup
      +
Continuous WAL archive
```

But your real RPO also depends on how quickly/reliably WAL gets into durable storage.

---

# 24. RTO and restore design

**RTO = how quickly must service return?**

Imagine:

```text
Database = 20 TB
```

A restore may take hours depending on:

```text
repository throughput
network throughput
compression
storage performance
parallelism
WAL volume to replay
```

So don't promise:

```text
RTO = 30 minutes
```

without actually testing restore performance.

The correct process is:

```text
Business says:
RTO 60 minutes
      |
      v
Architecture designed
      |
      v
Recovery tested
      |
      v
Measured restore = 42 minutes
```

Then you have evidence.

---

# 25. Backup retention

Don't keep everything forever without a reason.

Example retention policy:

```text
Daily backups
  7 days

Weekly backups
  4 weeks

Monthly backups
  12 months
```

But actual retention depends on:

```text
business requirements
compliance
storage costs
PITR window
legal requirements
```

pgBackRest supports expiring backups according to configured retention policies. Its current command set includes `expire`. ([pgBackRest][3])

---

# 26. Protect the backup repository

Your backup repository may contain your entire business database.

Secure it like production data.

Protect:

```text
repository credentials
encryption keys
network access
filesystem permissions
object-store credentials
backup administration rights
```

And consider:

```text
immutability
versioning
separate credentials
offline/offsite copies
```

where appropriate.

Because ransomware that can delete:

```text
production database
+
backup repository
```

has defeated your recovery plan.

---

# 27. Backups must be monitored

Monitor at least:

```text
Last successful full backup
Last successful diff/incremental
Latest archived WAL
Archive failures
Backup duration
Backup size
Repository capacity
Retention/expiration
```

A dashboard saying:

```text
PostgreSQL: HEALTHY
Patroni: HEALTHY
etcd: HEALTHY
HAProxy: HEALTHY

Backups: FAILED FOR 12 DAYS
```

means you do **not** have a healthy production database platform.

---

# 28. The most important rule: restore testing

This statement:

```text
Backup completed successfully
```

isn't enough.

You need:

```text
Backup restored successfully
```

A useful DR exercise is:

```text
1. Select recent backup

2. Provision separate recovery VM

3. Restore backup

4. Replay WAL

5. Start PostgreSQL

6. Validate database

7. Run application queries

8. Measure restore duration

9. Document problems

10. Repeat periodically
```

This gives you a tested RTO instead of an assumed one.

---

# 29. Full Patroni disaster scenario

Let's simulate the nightmare.

At 09:00:

```text
pg01 PRIMARY
pg02 REPLICA
pg03 REPLICA

All healthy
```

At 09:15 the storage platform fails:

```text
pg01 X
pg02 X
pg03 X
```

But:

```text
pgBackRest repository = healthy
```

Recovery architecture:

```text
Backup Repository
      |
      v
Choose latest valid backup
      |
      v
Restore pg01/recovery node
      |
      v
Replay archived WAL
      |
      v
Reach latest recoverable point
      |
      v
Validate PostgreSQL
      |
      v
Establish authoritative primary
      |
      +----------+
      |          |
      v          v
Build pg02    Build pg03
replica       replica
      |
      v
Restore connection routing
      |
      v
Application validation
```

That's **Disaster Recovery**.

Patroni helps rebuild the HA environment after the authoritative database has been restored, but the backup system is what saves the data.

---

# 30. Another disaster — accidental DELETE

At:

```text
14:00 database healthy
14:20 DELETE FROM payments;
14:25 problem discovered
```

Patroni sees:

```text
pg01 PRIMARY healthy
pg02 REPLICA healthy
pg03 REPLICA healthy
```

Everything is GREEN.

But from a business perspective:

```text
DATA IS BROKEN
```

Recovery:

```text
Backup repository
       |
       v
Restore separate server
       |
       v
PITR to 14:19:59
       |
       v
Validate payments
       |
       v
Extract/recover required data
```

That's why:

```text
Infrastructure healthy
```

doesn't necessarily mean:

```text
Data healthy
```

---

# 31. Commands worth knowing

Check pgBackRest:

```bash
pgbackrest --stanza=prod-db check
```

View backups:

```bash
pgbackrest --stanza=prod-db info
```

Full backup:

```bash
pgbackrest \
  --stanza=prod-db \
  --type=full \
  backup
```

Restore:

```bash
pgbackrest \
  --stanza=prod-db \
  restore
```

And PostgreSQL archive monitoring:

```sql
SELECT
    archived_count,
    last_archived_wal,
    last_archived_time,
    failed_count,
    last_failed_wal,
    last_failed_time
FROM pg_stat_archiver;
```

Don't memorize restore flags yet. **Restore commands depend heavily on whether you're doing full recovery, PITR, a standby rebuild, or a DR cutover.**

---

# 32. Production recovery decision tree

This is worth remembering:

```text
                     DATA PROBLEM
                          |
          +---------------+---------------+
          |                               |
      Server failure                  Logical/data loss
          |                               |
          v                               v
Can Patroni fail over?             Is good data on replica?
          |                               |
       YES                               Usually NO
          |                               |
          v                               v
Automatic failover                 Backup / PITR
                                          |
                                          v
                                Restore separate instance
                                          |
                                          v
                                     Validate data
```

For complete cluster loss:

```text
NO surviving PostgreSQL
        |
        v
Backup repository
        |
        v
Restore authoritative primary
        |
        v
Rebuild replicas
        |
        v
Rebuild HA/routing
```

## What you should remember from Lesson 14

If you remember only six things, remember these:

**Patroni is HA, not backup.**

**Replication replicates mistakes too.**

**Physical base backup + continuous WAL gives you PITR.** PostgreSQL's continuous archiving mechanism is specifically designed around that model. ([PostgreSQL][1])

**Keep backups outside the PostgreSQL failure domain.**

**After complete cluster loss, restore one authoritative database first, then rebuild replicas.**

**A backup is trustworthy only after you've successfully restored it.**

### Next — Lesson 15: PostgreSQL/Patroni Security & Hardening

The next useful lesson is production security: **SCRAM authentication, `pg_hba.conf`, TLS for PostgreSQL, TLS/authentication for Patroni REST, securing etcd, firewall/port design, service accounts, Linux permissions, protecting `patroni.yml`, secret management, HAProxy security, and a production hardening checklist.**

[1]: https://www.postgresql.org/docs/14/continuous-archiving.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 14: 26.3. Continuous Archiving and Point-in-Time Recovery (PITR)"
[2]: https://www.postgresql.org/docs/17/continuous-archiving.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 17: 25.3. Continuous Archiving and Point-in-Time Recovery (PITR)"
[3]: https://pgbackrest.org/user-guide.html?utm_source=chatgpt.com "pgBackRest User Guide - Debian & Ubuntu"
[4]: https://patroni.readthedocs.io/_/downloads/en/rel_3_1/pdf/?utm_source=chatgpt.com "Patroni Documentation"
