# Lesson 6 — Patroni Switchover

Now we'll perform our first controlled role change.

A **switchover** is planned. All nodes are healthy, but we intentionally move the PostgreSQL primary role from one server to another.

[Official Patroni documentation](https://patroni.readthedocs.io/en/latest/)

Our starting state:

```text
pg01    PRIMARY
pg02    REPLICA
pg03    REPLICA
```

Our goal:

```text
pg01    REPLICA
pg02    PRIMARY
pg03    REPLICA
```

No server is failing. We're deliberately changing roles.

---

## 1. Why perform a switchover?

Imagine `pg01` needs OS patching.

Currently:

```text
Application
    |
    v
  pg01
 PRIMARY
   / \
  v   v
pg02 pg03
```

You don't want to shut down the active primary.

Instead:

```text
1. Move primary to pg02
2. Verify pg02
3. Perform maintenance on pg01
4. Bring pg01 back
5. Verify it is a replica
```

That's a typical use case for switchover.

---

# 2. First rule: check cluster health

Before changing anything:

```bash
patronictl -c /etc/patroni/patroni.yml list
```

You want something similar to:

```text
+ Cluster: pg-ha-cluster ------------------------+
| Member | Host          | Role    | State     |
+--------+---------------+---------+-----------+
| pg01   | 192.168.10.11 | Leader  | running   |
| pg02   | 192.168.10.12 | Replica | streaming |
| pg03   | 192.168.10.13 | Replica | streaming |
+--------+---------------+---------+-----------+
```

Before a production switchover, don't look only at `State`.

Pay attention to:

```text
Current leader
Replica health
Replication lag
Pending restarts
Node reachability
Application impact
```

For our lab, we mainly want all three nodes healthy and the replicas caught up.

---

# 3. Verify PostgreSQL replication

Connect to `pg01`:

```bash
psql -h 192.168.10.11 -U postgres
```

Run:

```sql
SELECT
    application_name,
    client_addr,
    state,
    sync_state
FROM pg_stat_replication;
```

You should see `pg02` and `pg03` streaming.

For example:

```text
application_name | client_addr    | state     | sync_state
-----------------+----------------+-----------+-----------
pg02             | 192.168.10.12 | streaming | async
pg03             | 192.168.10.13 | streaming | async
```

Remember from Lesson 2:

```text
async
```

means asynchronous replication.

---

# 4. Create data before the switchover

Let's prove that data survives our role change.

Connect to our training database:

```bash
psql -h 192.168.10.11 -U postgres -d training
```

Insert:

```sql
INSERT INTO employees
VALUES (4, 'David');
```

Check:

```sql
SELECT * FROM employees ORDER BY id;
```

We expect:

```text
1  Alice
2  Bob
3  Charlie
4  David
```

Now verify `pg02` can see it:

```bash
psql -h 192.168.10.12 -U postgres -d training
```

```sql
SELECT * FROM employees ORDER BY id;
```

Once the replica is caught up, you should see the same data.

---

# 5. Check whether pg02 is currently a standby

On `pg02`:

```sql
SELECT pg_is_in_recovery();
```

Expected:

```text
t
```

Remember:

```text
t = standby / replica

f = primary
```

That's a very useful PostgreSQL check independent of Patroni.

---

# 6. Perform the switchover

Now run:

```bash
patronictl -c /etc/patroni/patroni.yml switchover
```

Patroni will ask you to select the current leader and candidate depending on the version and command options.

We want:

```text
Current leader = pg01
Candidate       = pg02
```

Review the requested operation carefully and confirm it.

You can see the current CLI options for your installed Patroni version with:

```bash
patronictl switchover --help
```

[Patroni patronictl documentation](https://patroni.readthedocs.io/en/latest/patronictl.html?utm_source=chatgpt.com)

---

# 7. What happens behind the scenes?

This is more important than memorizing the command.

Initially:

```text
                  etcd
                   |
           leader = pg01
                   |
                   v

pg01             pg02             pg03
PRIMARY          REPLICA          REPLICA
   |
   +------ WAL ---->|
   |
   +------------------------>|
```

We request:

```text
pg01 -> pg02
```

Patroni coordinates a controlled transition.

Conceptually:

```text
             SWITCHOOVER

pg01
PRIMARY
  |
  | Step down
  v
REPLICA


pg02
REPLICA
  |
  | Promote
  v
PRIMARY
```

The DCS state is updated to reflect the new leadership.

Afterward:

```text
                  etcd
                   |
           leader = pg02
                   |
                   v

pg01             pg02             pg03
REPLICA          PRIMARY          REPLICA
   ^                                ^
   |                                |
   +----------- WAL ----------------+
                 |
                pg02
```

Notice something important:

### The replication direction changed.

Before:

```text
pg01
 |
 +----> pg02
 |
 +----> pg03
```

After:

```text
pg02
 |
 +----> pg01
 |
 +----> pg03
```

That's a key HA concept.

---

# 8. Check Patroni

Immediately run:

```bash
patronictl -c /etc/patroni/patroni.yml list
```

We want:

```text
pg01    Replica
pg02    Leader
pg03    Replica
```

This should become your first check after the operation.

---

# 9. Verify using PostgreSQL

Don't stop at `patronictl`.

Connect to `pg02`:

```bash
psql -h 192.168.10.12 -U postgres
```

Run:

```sql
SELECT pg_is_in_recovery();
```

Expected:

```text
f
```

Therefore:

```text
pg02 = PRIMARY
```

Now connect to `pg01`:

```bash
psql -h 192.168.10.11 -U postgres
```

Run:

```sql
SELECT pg_is_in_recovery();
```

Expected:

```text
t
```

Therefore:

```text
pg01 = REPLICA
```

Excellent.

We've confirmed the role change from **both Patroni and PostgreSQL**.

---

# 10. Test writing to the new primary

Connect to `pg02`:

```bash
psql -h 192.168.10.12 -U postgres -d training
```

Insert another row:

```sql
INSERT INTO employees
VALUES (5, 'Emma');
```

Check:

```sql
SELECT * FROM employees ORDER BY id;
```

Now:

```text
1 Alice
2 Bob
3 Charlie
4 David
5 Emma
```

This proves `pg02` is accepting writes.

---

# 11. Verify the old primary receives the new data

Now connect to `pg01`:

```bash
psql -h 192.168.10.11 -U postgres -d training
```

Run:

```sql
SELECT * FROM employees ORDER BY id;
```

You should eventually see:

```text
5 Emma
```

Think about what just happened.

Before:

```text
pg01 --WAL--> pg02
```

After switchover:

```text
pg02 --WAL--> pg01
```

That's a very important practical Patroni concept.

---

# 12. Try writing to old primary

While connected to `pg01`:

```sql
INSERT INTO employees
VALUES (6, 'Frank');
```

It should fail because `pg01` is now a standby.

That's good.

We want:

```text
pg01
REPLICA
READ ONLY
```

and:

```text
pg02
PRIMARY
READ + WRITE
```

---

# 13. Check replication from the new primary

On `pg02`:

```sql
SELECT
    application_name,
    client_addr,
    state,
    sync_state
FROM pg_stat_replication;
```

Now `pg02` should see replication connections associated with the other replicas.

Conceptually:

```text
NEW PRIMARY: pg02

       +---- pg01
       |
pg02 --+
       |
       +---- pg03
```

---

# 14. WAL timeline

There's another important concept hiding underneath this operation: **PostgreSQL timelines**.

When PostgreSQL promotes a standby, it begins a new timeline.

Simplified:

```text
Timeline 1

pg01 PRIMARY
     |
     +---- WAL ---->
```

Then:

```text
Switchover
     |
     v
pg02 promoted
```

PostgreSQL creates a new timeline for the new history.

Conceptually:

```text
Timeline 1
========================
                       \
                        \
Timeline 2               ====================>
                         pg02 PRIMARY
```

Why does PostgreSQL need this?

Because after promotion, database history has branched.

This becomes particularly important when dealing with:

```text
failover
old primary recovery
pg_rewind
PITR
WAL archives
```

You don't need to master timelines yet. Just remember:

> **Promoting a standby creates a new PostgreSQL timeline.**

[PostgreSQL standby and failover documentation](https://www.postgresql.org/docs/current/warm-standby-failover.html?utm_source=chatgpt.com)

---

# 15. Why `pg_rewind` matters

Remember this setting?

```yaml
use_pg_rewind: true
```

Consider an unplanned failure:

```text
Timeline 1

pg01 PRIMARY
   |
   X
 failure


pg02 promoted

Timeline 2
   |
   v
pg02 PRIMARY
```

Later `pg01` returns.

Its data directory represents the old history.

It cannot simply become primary again.

We need:

```text
OLD pg01
   |
   | synchronize
   v
CURRENT pg02
```

`pg_rewind` can help Patroni bring the old primary back in line with the new primary when its requirements are met.

This becomes much more obvious in our next lesson.

---

# 16. Switchover vs failover

You should now clearly understand the difference.

|                            | Switchover                         | Failover                                          |
| -------------------------- | ---------------------------------- | ------------------------------------------------- |
| Planned?                   | Yes                                | Usually no                                        |
| Existing primary healthy?  | Usually yes                        | Typically failed/unavailable                      |
| Used for maintenance?      | Yes                                | Not primarily                                     |
| Candidate promoted?        | Yes                                | Yes                                               |
| Application impact?        | Should be controlled/minimized     | Depends on failure and architecture               |
| Possible data-loss concern | Lower with healthy/caught-up nodes | More important, especially with async replication |

Think:

```text
SWITCHOVER
"We choose to move primary."


FAILOVER
"We need another primary because the old one failed."
```

---

# 17. A production DBA workflow

When someone says:

> "We need to patch pg01."

A good thought process is:

```text
Check Patroni cluster
       |
       v
Check replication health
       |
       v
Check replication lag
       |
       v
Check candidate
       |
       v
Coordinate application/maintenance
       |
       v
Perform switchover
       |
       v
Verify new primary
       |
       v
Verify old primary became replica
       |
       v
Verify replication
       |
       v
Perform maintenance
```

Don't think:

```text
systemctl stop postgresql
```

first.

In an HA environment, always understand the cluster state before manipulating an individual database process.

---

# 18. Commands to remember

These four are worth practicing repeatedly.

### Patroni cluster

```bash
patronictl -c /etc/patroni/patroni.yml list
```

### PostgreSQL role

```sql
SELECT pg_is_in_recovery();
```

### Replication connections

```sql
SELECT
    application_name,
    client_addr,
    state,
    sync_state
FROM pg_stat_replication;
```

### Controlled primary move

```bash
patronictl -c /etc/patroni/patroni.yml switchover
```

You should know what each tells you rather than just memorizing commands.

---

# Practice exercise

Do the operation again in reverse.

Current:

```text
pg01    REPLICA
pg02    PRIMARY
pg03    REPLICA
```

Perform:

```text
pg02 -> pg01
```

Then verify:

```text
pg01    PRIMARY
pg02    REPLICA
pg03    REPLICA
```

Check all three layers:

```text
1. patronictl list

2. SELECT pg_is_in_recovery();

3. SELECT ... FROM pg_stat_replication;
```

If you can perform that comfortably and explain **why the replication direction changes**, you've understood switchover.

## Next — Lesson 7: Automatic Failover 🚨

This is the Patroni exercise you've been building toward.

We'll start with:

```text
pg01    PRIMARY
pg02    REPLICA
pg03    REPLICA
```

Then we'll **simulate an actual primary failure** rather than requesting a switchover.

We'll watch Patroni detect the failure, observe the leader lock/DCS behavior, see either `pg02` or `pg03` become primary, measure what happens to connections, verify data, bring `pg01` back, understand `pg_rewind`/rejoin behavior, and troubleshoot the case where the old primary **doesn't automatically rejoin**.

That's the exercise that turns the architecture from theory into something you can support.
