# Lesson 16 — PostgreSQL Major Upgrade with Patroni

A PostgreSQL major upgrade is different from a normal patch or restart.

For example:

```text
PostgreSQL 17 -> PostgreSQL 18
```

is a **major upgrade**.

You cannot safely treat it like:

```text
install new package
restart Patroni
done
```

because major versions can change PostgreSQL's internal storage format. PostgreSQL officially supports several upgrade paths, including `pg_upgrade`, dump/restore, and replication-based migration. ([PostgreSQL][1])

## 1. Minor vs major upgrade

Minor upgrade:

```text
18.4 -> 18.6
```

Usually:

```text
replace binaries
restart PostgreSQL
same PGDATA
```

Major upgrade:

```text
17 -> 18
```

requires data migration or conversion.

Think:

```text
MINOR

PG17.10
   |
new binaries
   |
PG17.11

Same data format
```

versus:

```text
MAJOR

PG17 data
   |
   X cannot simply attach
   |
migration/conversion
   |
PG18 data
```

PostgreSQL explicitly notes that major releases may change the internal data format and can also introduce application-visible incompatibilities. ([PostgreSQL][1])

---

# 2. Patroni adds another layer

Without Patroni:

```text
PostgreSQL
   |
pg_upgrade
   |
PostgreSQL new version
```

With Patroni:

```text
               etcd
                |
             Patroni
                |
             PostgreSQL
```

You must coordinate:

```text
PostgreSQL binaries
PGDATA
Patroni
DCS state
replicas
HAProxy
applications
extensions
backups
```

That's why this needs a proper runbook.

---

# 3. Three main upgrade strategies

For Patroni environments, think about these:

| Method              |       Downtime |         Complexity | Typical use                         |
| ------------------- | -------------: | -----------------: | ----------------------------------- |
| `pg_upgrade`        | Moderate/short |             Medium | Same infrastructure, planned outage |
| Logical replication |       Very low |             Higher | Low-downtime migration              |
| Dump/restore        |           High | Lower conceptually | Small databases / restructuring     |

For large production Patroni systems, the decision is usually between:

```text
pg_upgrade
```

and:

```text
logical replication / blue-green
```

---

# 4. Strategy A — `pg_upgrade`

This is the classic in-place approach.

Suppose:

```text
pg01  PG17 PRIMARY
pg02  PG17 REPLICA
pg03  PG17 REPLICA
```

You cannot simply upgrade `pg02` to PostgreSQL 18 and keep it physically streaming from a PostgreSQL 17 primary.

Physical replication requires compatible binary/storage formats.

So a major upgrade is not a normal rolling Patroni upgrade.

Patroni's current documentation gives the supported broad process as:

1. Stop Patroni.
2. Upgrade PostgreSQL and run `pg_upgrade` on the **primary**.
3. Update `patroni.yml`.
4. Reset the relevant Patroni/DCS initialization state because `pg_upgrade` creates a new PostgreSQL system identifier.
5. Start Patroni on the upgraded primary.
6. Upgrade standby binaries.
7. Rebuild the standbys from the upgraded primary. ([Patroni][2])

That last point is important:

> **Running `pg_upgrade` independently on Patroni replicas is not the normal supported approach.**

Patroni recommends rebuilding replicas after upgrading the primary. ([Patroni][2])

---

# 5. Why the PostgreSQL system identifier matters

Every PostgreSQL cluster has a system identifier.

Conceptually:

```text
Old PG17 cluster

system_identifier =
123456789
```

After `pg_upgrade`, the new cluster has a new identity.

```text
New PG18 cluster

system_identifier =
987654321
```

Patroni/DCS still remembers information about the previous cluster.

So Patroni must not blindly treat:

```text
new PG18 cluster
```

as if it were the old PG17 physical cluster.

This is why the Patroni major-upgrade procedure requires handling the DCS initialization state. ([Patroni][2])

---

# 6. High-level `pg_upgrade` architecture

Before:

```text
             Patroni Cluster

pg01             pg02             pg03
PG17             PG17             PG17
PRIMARY           REPLICA          REPLICA
```

Maintenance begins:

```text
Applications stopped / writes blocked
             |
             v
Patroni stopped
             |
             v
PG17 primary stopped
```

Upgrade:

```text
pg01

PG17 data
    |
pg_upgrade
    |
    v
PG18 data
```

Then:

```text
pg01 PG18 PRIMARY
        |
        +------> rebuild pg02 PG18
        |
        +------> rebuild pg03 PG18
```

Final:

```text
pg01             pg02             pg03
PG18             PG18             PG18
PRIMARY           REPLICA          REPLICA
```

---

# 7. Pre-upgrade checks

Never start with `pg_upgrade`.

First collect:

```text
Current Patroni state
PostgreSQL versions
Patroni version
Extension versions
Database sizes
Replication health
Backup status
Disk capacity
Application compatibility
Rollback requirements
```

Useful commands:

```bash
patronictl -c /etc/patroni/patroni.yml list
```

```sql
SELECT version();
```

```sql
SELECT extname, extversion
FROM pg_extension
ORDER BY extname;
```

```sql
SELECT datname,
       pg_size_pretty(pg_database_size(datname))
FROM pg_database
ORDER BY pg_database_size(datname) DESC;
```

Also verify backups before touching the cluster.

---

# 8. Extensions are a major upgrade risk

Suppose PostgreSQL uses:

```text
PostGIS
pg_stat_statements
TimescaleDB
pg_cron
pg_partman
```

You need compatible PostgreSQL 18 versions of those extensions installed before completing the upgrade.

A common failure is:

```text
PostgreSQL 18 installed
       |
       v
extension shared library missing
       |
       v
database won't start correctly
```

So build an extension compatibility matrix before the maintenance window.

For example:

| Extension            | PG17 version | PG18 compatible package available? |
| -------------------- | -----------: | ---------------------------------- |
| `pg_stat_statements` |      bundled | Yes                                |
| PostGIS              |          x.x | Verify                             |
| TimescaleDB          |          x.x | Verify                             |
| Custom extension     |     internal | Test                               |

---

# 9. Run `pg_upgrade --check`

Before actually upgrading, use the new version's `pg_upgrade` binary to perform compatibility checks.

Conceptually:

```bash
/path/to/pg18/bin/pg_upgrade \
  --check \
  --old-bindir=/path/to/pg17/bin \
  --new-bindir=/path/to/pg18/bin \
  --old-datadir=/old/pgdata \
  --new-datadir=/new/pgdata
```

This catches problems such as:

```text
missing libraries
unsupported data types
extension incompatibilities
configuration problems
```

before the destructive/real migration begins.

Do this in your pre-production rehearsal too.

---

# 10. `pg_upgrade` modes

`pg_upgrade` can move data in different ways.

Traditional copy mode:

```text
Old PGDATA
     |
copy
     |
New PGDATA
```

Safest conceptually, but needs extra storage and can take longer.

Link mode:

```text
old files <---- hard links ----> new cluster
```

Very fast, but rollback becomes more complicated because once the new cluster modifies shared files, the old cluster isn't safely reusable.

PostgreSQL 18 also adds a `--swap` mode, and PostgreSQL documents other fast approaches including link/clone depending on filesystem support. ([PostgreSQL][3])

For your first production major upgrade, prioritize **rollback clarity** over shaving a few minutes off the procedure.

---

# 11. A safer Patroni `pg_upgrade` workflow

Conceptually:

```text
PRE-CHECK
   |
   v
Full backup + restore confidence
   |
   v
Confirm Patroni healthy
   |
   v
Stop application writes
   |
   v
Stop Patroni on all nodes
   |
   v
Upgrade primary using pg_upgrade
   |
   v
Update Patroni configuration/binary paths
   |
   v
Reset required DCS initialization state
   |
   v
Start Patroni on upgraded primary
   |
   v
Validate PG18 primary
   |
   v
Upgrade software on replicas
   |
   v
Reinitialize/rebuild replicas
   |
   v
Wait for streaming
   |
   v
Restore routing/application access
   |
   v
Post-upgrade validation
```

This is a maintenance-window operation.

---

# 12. Why replicas are normally rebuilt

Old replicas contain PostgreSQL 17 physical data.

After primary upgrade:

```text
pg01 = PG18
```

But:

```text
pg02 = PG17 physical replica
```

A PG17 data directory cannot simply continue physical streaming from a PG18 primary.

So the safest Patroni approach is:

```text
Upgrade PG18 binaries on pg02
       |
       v
remove/prepare old replica PGDATA
       |
       v
Patroni reinitializes
       |
       v
copy PG18 state from pg01
       |
       v
pg02 PG18 replica
```

Then repeat for `pg03`.

Patroni's major-upgrade guidance specifically recommends letting Patroni recreate replicas rather than running `pg_upgrade` separately on each standby. ([Patroni][2])

---

# 13. Large database problem

Suppose:

```text
Database size = 15 TB
```

Rebuilding two replicas could mean transferring a huge amount of data.

```text
pg01
 |
 +---- 15 TB ----> pg02
 |
 +---- 15 TB ----> pg03
```

This affects:

```text
network
primary I/O
backup infrastructure
recovery time
RTO
```

For large clusters, plan replica recreation carefully.

You might use:

```text
pgBackRest restore
storage snapshots
rsync procedure
```

depending on platform and supportability.

Patroni notes PostgreSQL's documented `rsync` procedure as an alternative for experienced operators, although rebuilding the standby through Patroni is described as the safest route. ([Patroni][2])

---

# 14. Strategy B — Logical replication

For systems needing much less downtime, build a **new PostgreSQL 18 cluster alongside the PostgreSQL 17 cluster**.

```text
OLD CLUSTER                       NEW CLUSTER

Patroni PG17                      Patroni PG18

pg17-01 PRIMARY  ----logical----> pg18-01
pg17-02 REPLICA                   pg18-02 REPLICA
pg17-03 REPLICA                   pg18-03 REPLICA
```

PostgreSQL logical replication supports replication between different major versions specifically as an upgrade technique. ([PostgreSQL][4])

This is often called:

```text
Blue / Green
```

or:

```text
Side-by-side migration
```

---

# 15. Logical replication migration flow

Old cluster:

```text
BLUE

PG17 Patroni
```

New:

```text
GREEN

PG18 Patroni
```

Process:

```text
Build PG18 cluster
       |
       v
Create schema/extensions
       |
       v
Configure logical replication
       |
       v
Initial data copy
       |
       v
Continuously replicate changes
       |
       v
Lag approaches zero
       |
       v
Maintenance/cutover
       |
       v
Stop writes to PG17
       |
       v
Wait until PG18 fully caught up
       |
       v
Switch application
       |
       v
PG18 becomes production
```

PostgreSQL documents that replication-based upgrades can reduce cutover downtime to only several seconds in suitable cases. ([PostgreSQL][5])

---

# 16. Logical replication setup requirements

Publisher needs:

```conf
wal_level = logical
```

and enough:

```text
max_replication_slots
max_wal_senders
```

PostgreSQL 18's current documentation requires `wal_level=logical` on the publisher and enough replication slots/senders for the subscriptions and table synchronization workers. ([PostgreSQL][6])

The basic model:

```sql
CREATE PUBLICATION app_pub
FOR ALL TABLES;
```

On the new cluster:

```sql
CREATE SUBSCRIPTION app_sub
CONNECTION '...old cluster...'
PUBLICATION app_pub;
```

Do not copy these into production blindly. Authentication, publication scope, sequence handling, DDL and cutover all need planning.

---

# 17. Logical replication does NOT replicate everything

This is critical.

Logical replication is mainly about table data changes.

You must separately handle things like:

```text
Roles/users
DDL changes
Extensions
Sequences
Large objects in some designs
Configuration
Privileges
Tables not included in publication
Schema changes during migration
```

So:

```text
data synchronized
```

does not automatically mean:

```text
database migration complete
```

---

# 18. Sequences are a common cutover problem

Suppose old cluster has:

```text
orders_id_seq = 1,000,000
```

New cluster copied table rows but its sequence is:

```text
orders_id_seq = 500,000
```

After cutover:

```sql
INSERT INTO orders ...
```

could generate an already-used ID.

So your cutover runbook must explicitly synchronize sequences.

Example conceptually:

```text
Stop writes
    |
wait for logical replication
    |
sync sequences
    |
validate
    |
switch application
```

---

# 19. DDL is another problem

If developers run:

```sql
ALTER TABLE orders ADD COLUMN status text;
```

on PG17 during migration, that DDL is not automatically treated like normal row-level logical replication.

Therefore during the migration period you need a process:

```text
DDL change
   |
   +--> apply to old cluster
   |
   +--> apply compatible change to new cluster
```

or freeze schema changes during migration.

This is one reason logical replication upgrades require more operational discipline.

---

# 20. Blue-green Patroni architecture

A strong low-downtime architecture might be:

```text
                       Applications
                            |
                            v
                         HAProxy
                            |
              +-------------+-------------+
              |                           |
            BLUE                        GREEN
          PG17 cluster                PG18 cluster

       pg17-01 PRIMARY  ======>     pg18-01 PRIMARY
       pg17-02 REPLICA  logical     pg18-02 REPLICA
       pg17-03 REPLICA   repl       pg18-03 REPLICA
```

During normal migration:

```text
Application -> BLUE
```

At cutover:

```text
Application -> GREEN
```

This has a big advantage:

> You are not transforming your only production cluster in place.

---

# 21. Rollback with blue-green

Before cutover:

```text
BLUE = production
GREEN = synchronized
```

Cut over:

```text
Application -> GREEN
```

If something immediately fails before significant new writes occur, rollback may be easier:

```text
Application -> BLUE
```

But be careful.

Once applications write to GREEN:

```text
GREEN contains new transactions
```

BLUE may no longer contain those changes.

Rollback is no longer simply:

```text
switch DNS back
```

You need a **reverse replication or data reconciliation strategy** if rollback after new writes is required.

This needs to be designed before migration.

---

# 22. Which method should you choose?

### `pg_upgrade`

Good when:

```text
downtime window available
same infrastructure
large database
simple rollback carefully planned
fewer moving parts preferred
```

### Logical replication

Good when:

```text
very low downtime required
new infrastructure available
application can handle migration restrictions
extra complexity acceptable
```

### Dump/restore

Good when:

```text
small database
long downtime acceptable
want clean logical rebuild
major restructuring required
```

---

# 23. Practical example

Suppose:

```text
Current:
PostgreSQL 17
3 Patroni nodes
2 TB database

Requirement:
< 30 minutes downtime
```

`pg_upgrade` might be realistic if you've rehearsed and know:

```text
pg_upgrade conversion = 5 min
application checks = 10 min
Patroni bootstrap = 5 min
routing/cutover = 5 min
```

But remember replicas may take much longer to rebuild afterward.

The service can still potentially return with:

```text
one PG18 primary
```

while replicas are rebuilding, but your HA redundancy is temporarily reduced.

That risk must be explicitly accepted.

---

# 24. Suppose downtime requirement is 30 seconds

Then an in-place `pg_upgrade` likely isn't the architecture I'd start with.

I'd investigate:

```text
PG17 Patroni BLUE
        |
logical replication
        |
PG18 Patroni GREEN
```

and perform a short cutover.

PostgreSQL's documentation specifically describes major-version upgrades via logical replication and notes that switchover can result in only seconds of downtime. ([PostgreSQL][5])

---

# 25. Pre-upgrade validation checklist

Before maintenance, verify:

```text
[ ] Patroni cluster healthy
[ ] All expected replicas healthy
[ ] etcd quorum healthy
[ ] HAProxy healthy
[ ] Full backup successful
[ ] Restore test known-good
[ ] WAL archive current
[ ] PG18 packages available
[ ] All extensions PG18-compatible
[ ] Application tested against PG18
[ ] pg_upgrade --check successful
[ ] Disk space sufficient
[ ] Rollback plan written
[ ] Maintenance window approved
[ ] Monitoring suppression/change planned
```

---

# 26. Application testing is mandatory

Don't focus only on PostgreSQL.

An application can break because PostgreSQL 18 behaves differently around:

```text
SQL behavior
removed/deprecated features
drivers
query plans
extensions
authentication
data types
reserved words
```

PostgreSQL itself warns that major releases can introduce user-visible incompatibilities requiring application changes. ([PostgreSQL][1])

So have a staging environment:

```text
Production copy
      |
      v
PG18 test cluster
      |
      v
Application regression tests
```

---

# 27. Query plan changes

Even if SQL works:

```text
PG17 query = 50 ms

PG18 query = 4 sec
```

is still a failed upgrade.

Major upgrades can change:

```text
planner behavior
statistics
optimizer features
execution paths
```

PostgreSQL 18 improves `pg_upgrade` by preserving optimizer statistics, though extended statistics are not preserved. ([PostgreSQL][3])

Still test your critical workloads.

After upgrade check:

```sql
EXPLAIN (ANALYZE, BUFFERS)
...
```

for important queries.

---

# 28. Post-upgrade checks

After PostgreSQL 18 is online:

```bash
patronictl list
```

Then:

```sql
SELECT version();
```

Expected:

```text
PostgreSQL 18.x
```

Check roles:

```sql
SELECT pg_is_in_recovery();
```

Check replicas:

```sql
SELECT
    application_name,
    state,
    sync_state
FROM pg_stat_replication;
```

Check extensions:

```sql
SELECT extname, extversion
FROM pg_extension;
```

Also check:

```text
application connectivity
authentication
HAProxy health
replication lag
backup jobs
WAL archiving
monitoring
scheduled jobs
performance
```

---

# 29. Backup immediately after upgrade

After validating the new PostgreSQL major version, establish a fresh backup baseline.

Why?

Your old backup belongs to:

```text
PostgreSQL 17
```

while production is now:

```text
PostgreSQL 18
```

You want a known-good PostgreSQL 18 recovery point.

So:

```text
Upgrade
   |
validate
   |
take PG18 backup
   |
verify WAL archiving
```

---

# 30. Don't delete PG17 immediately

Suppose upgrade completed at:

```text
22:00
```

Don't automatically delete all PG17 binaries/data at:

```text
22:05
```

Keep rollback artifacts for the approved retention period, assuming your chosen `pg_upgrade` mode allows safe rollback.

The exact rollback ability depends on whether you used:

```text
copy
link
clone
swap
```

and what has happened since cutover.

Document this before the upgrade.

---

# 31. Example in-place Patroni runbook

A simplified production runbook could be:

```text
T-7 DAYS
------
Test PG17 -> PG18 upgrade in staging
Validate extensions
Run application tests
Measure duration


T-1 DAY
------
Confirm backups
Confirm WAL archive
Run pg_upgrade --check
Check disk


MAINTENANCE
-----------
Disable/stop application writes

Verify:
patronictl list

Stop Patroni on all nodes

Stop PostgreSQL old cluster

Run pg_upgrade on primary

Update Patroni configuration

Handle Patroni/DCS initialization state

Start Patroni on upgraded primary

Validate PG18

Upgrade binaries on pg02/pg03

Reinitialize replicas

Verify streaming

Enable HAProxy/application

Run application tests

Create new PG18 backup
```

Exact commands should be written and tested for your OS packaging, PostgreSQL layout, Patroni version and backup tool before the window.

---

# 32. Example blue-green runbook

```text
WEEK 1
------
Build new PG18 Patroni cluster

Install extensions/config

Create logical publication/subscription

Initial sync


MIGRATION PERIOD
----------------
Monitor logical replication

Apply DDL changes to both systems

Run comparison tests


CUTOVER
-------
Stop application writes on PG17

Wait until replication lag = 0

Synchronize sequences

Final data validation

Switch HAProxy/DNS/service endpoint

Enable writes on PG18

Validate application


AFTER
-----
Monitor PG18

Keep PG17 isolated/read-only temporarily

Take PG18 backup

Decommission PG17 after rollback window
```

---

# 33. Patroni's responsibility during the upgrade

Patroni is not actually transforming PostgreSQL 17 files into PostgreSQL 18.

`pg_upgrade` or logical migration does that work.

Patroni's responsibilities remain:

```text
Cluster membership
Primary/replica roles
Replication management
DCS coordination
Post-upgrade failover
```

Think:

```text
PostgreSQL upgrade tool
        |
        +--> changes database version/data


Patroni
        |
        +--> restores/manages HA around the new cluster
```

---

# 34. What I would choose

For a normal business system where a maintenance window of tens of minutes is acceptable:

```text
pg_upgrade
+
rebuild Patroni replicas
```

is usually the simpler architecture.

For a highly critical system requiring seconds of write downtime:

```text
new Patroni cluster
+
logical replication
+
controlled blue-green cutover
```

is generally the stronger approach, despite greater complexity.

## What you should remember from Lesson 16

A PostgreSQL major upgrade **is not a rolling Patroni restart**.

With an in-place Patroni upgrade, upgrade the primary using `pg_upgrade` and normally rebuild the standbys afterward; Patroni's current documentation explicitly recommends that model. ([Patroni][2])

For very low downtime, logical replication lets different PostgreSQL major versions run side-by-side and is officially supported as an upgrade technique. ([PostgreSQL][4])

Always validate **extensions, application compatibility, backup/restore, `pg_upgrade --check`, rollback, replication rebuild time and post-upgrade performance** before touching production.

### Next — Lesson 17: Patroni Performance Tuning & Capacity Planning

The next useful lesson would cover **CPU/RAM/storage sizing, `shared_buffers`, `work_mem`, connection sizing, PgBouncer, WAL generation, checkpoint tuning, replication bandwidth, replica lag, etcd latency, Patroni timing settings, HAProxy sizing, and how to capacity-plan a Patroni cluster without creating failover problems.**

[1]: https://www.postgresql.org/docs/current/upgrading.html "PostgreSQL: Documentation: 18: 18.6. Upgrading a PostgreSQL Cluster"
[2]: https://patroni.readthedocs.io/_/downloads/en/latest/pdf/ "Patroni Documentation"
[3]: https://www.postgresql.org/docs/18/release-18.html "PostgreSQL: Documentation: 18: E.5. Release 18"
[4]: https://www.postgresql.org/docs/current/logical-replication.html "PostgreSQL: Documentation: 18: Chapter 29. Logical Replication"
[5]: https://www.postgresql.org/docs/17/upgrading.html "PostgreSQL: Documentation: 17: 18.6. Upgrading a PostgreSQL Cluster"
[6]: https://www.postgresql.org/docs/current/logical-replication-config.html "PostgreSQL: Documentation: 18: 29.12. Configuration Settings"
