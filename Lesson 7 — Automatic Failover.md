# Lesson 7 — Automatic Failover

Now we simulate the real failure case.

Starting state:

```text
pg01    PRIMARY
pg02    REPLICA
pg03    REPLICA
```

Goal: stop `pg01` unexpectedly and watch Patroni promote another healthy replica.

## 1. Check the cluster before testing

Run:

```bash
patronictl -c /etc/patroni/patroni.yml list
```

You want all three members healthy.

Then on the current primary:

```sql
SELECT
    application_name,
    client_addr,
    state,
    sync_state
FROM pg_stat_replication;
```

Make sure `pg02` and `pg03` are streaming and replication lag is small.

Also insert one test row:

```sql
INSERT INTO employees
VALUES (6, 'Failover-Test');
```

Verify the replicas can see it before you continue.

---

## 2. Confirm pg01 is really the primary

On `pg01`:

```sql
SELECT pg_is_in_recovery();
```

Expected:

```text
f
```

On `pg02` and `pg03`:

```sql
SELECT pg_is_in_recovery();
```

Expected:

```text
t
```

So we have:

```text
pg01    f    PRIMARY
pg02    t    REPLICA
pg03    t    REPLICA
```

---

## 3. Simulate a primary failure

Because Patroni manages PostgreSQL, the cleanest lab test is to stop the **Patroni service/process** on the primary.

If you created a systemd service:

```bash
sudo systemctl stop patroni
```

If Patroni is still running interactively from our earlier lab, terminate that Patroni process on `pg01`.

The important point is that `pg01` can no longer maintain its Patroni leadership.

Your architecture becomes:

```text
Before:

             pg01
            PRIMARY
             /  \
            /    \
           v      v
        pg02      pg03
       REPLICA   REPLICA


Failure:

             pg01
              X
            DOWN

        pg02      pg03
       REPLICA   REPLICA
```

Do **not** immediately manually promote anything.

Let Patroni handle it.

---

## 4. Watch Patroni

From `pg02` or `pg03`, repeatedly check:

```bash
patronictl -c /etc/patroni/patroni.yml list
```

Initially, you may briefly see the old leader unavailable.

Then Patroni should select an eligible replica and promote it.

For example:

```text
+ Cluster: pg-ha-cluster -----------------------+
| Member | Host          | Role    | State     |
+--------+---------------+---------+-----------+
| pg01   | 192.168.10.11 | Replica | stopped   |
| pg02   | 192.168.10.12 | Leader  | running   |
| pg03   | 192.168.10.13 | Replica | streaming |
+--------+---------------+---------+-----------+
```

The exact candidate can vary depending on cluster state and configuration.

The key result is:

```text
OLD

pg01 PRIMARY
pg02 REPLICA
pg03 REPLICA


NEW

pg01 DOWN
pg02 PRIMARY
pg03 REPLICA
```

---

## 5. What happened internally?

The important sequence is roughly:

```text
pg01 stops
   |
   v
pg01 can no longer maintain leadership
   |
   v
Patroni members continue talking to DCS
   |
   v
Old leader lock expires / leadership is lost
   |
   v
Eligible replica wins leadership
   |
   v
PostgreSQL standby is promoted
   |
   v
New primary starts accepting writes
```

Patroni is not copying a new database during failover.

PostgreSQL already had replicas.

Patroni is changing the **role** of one of those replicas.

---

## 6. Why isn't failover instantaneous?

Remember these settings:

```yaml
ttl: 30
loop_wait: 10
retry_timeout: 10
```

Patroni has to safely determine that leadership can change.

It cannot assume:

> "I lost one network packet, therefore the primary is dead."

That would be dangerous.

Distributed HA systems intentionally balance:

```text
fast failure detection
        vs
safe failure detection
```

Making timeouts extremely aggressive can create instability during temporary network problems.

This is why production Patroni timeout tuning should be based on network behavior, infrastructure, RTO requirements, and failure testing—not guesswork.

---

## 7. Verify the new primary with PostgreSQL

Assume `pg02` was promoted.

Connect:

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

So PostgreSQL itself confirms:

```text
pg02 = PRIMARY
```

Check `pg03`:

```sql
SELECT pg_is_in_recovery();
```

Expected:

```text
t
```

---

## 8. Verify replication direction changed

Before the failure:

```text
pg01
 |
 +---- WAL ----> pg02
 |
 +---- WAL ----> pg03
```

After `pg02` becomes primary:

```text
pg02
 |
 +---- WAL ----> pg03
 |
 +---- WAL ----> pg01   once pg01 rejoins
```

On the new primary, check:

```sql
SELECT
    application_name,
    client_addr,
    state,
    sync_state
FROM pg_stat_replication;
```

Initially you may see only `pg03` because `pg01` is still down.

---

## 9. Verify application data

Connect to the new primary:

```bash
psql -h 192.168.10.12 -U postgres -d training
```

Run:

```sql
SELECT * FROM employees ORDER BY id;
```

You should still see your previous rows, including:

```text
6 | Failover-Test
```

Now insert a row **after failover**:

```sql
INSERT INTO employees
VALUES (7, 'New-Primary');
```

Verify:

```sql
SELECT * FROM employees ORDER BY id;
```

This proves the promoted replica is now accepting writes.

---

# 10. The important async replication risk

Remember Lesson 2.

If replication is asynchronous, it is possible for the old primary to acknowledge a transaction that has not yet reached the promoted replica.

Example:

```text
pg01 PRIMARY

Transaction 100  -> replica
Transaction 101  -> replica
Transaction 102  -> replica
Transaction 103  -> committed locally

                    X pg01 crashes

pg02 only has up to 102
```

If `pg02` is promoted:

```text
Transaction 103 may be missing
```

That is why:

```text
HA != automatically zero data loss
```

You must separately define:

```text
RTO
RPO
replication mode
synchronous requirements
failover policy
```

Patroni automates failover, but the durability guarantee still depends heavily on PostgreSQL replication configuration.

---

# 11. Bring the old primary back

Now start Patroni again on `pg01`.

For systemd:

```bash
sudo systemctl start patroni
```

Then:

```bash
patronictl -c /etc/patroni/patroni.yml list
```

The important thing is that `pg01` must **not** come back as primary.

It should discover:

```text
Current Patroni leader = pg02
```

and work toward becoming a replica.

Eventually:

```text
pg01    REPLICA
pg02    PRIMARY
pg03    REPLICA
```

---

# 12. Why can't pg01 simply resume where it stopped?

This is where PostgreSQL timelines matter.

Before failover:

```text
Timeline 1
--------------------------------->
                       pg01 PRIMARY
```

Then pg01 dies.

`pg02` is promoted:

```text
Timeline 1
----------------------+
                      \
                       \
Timeline 2              -------------------->
                         pg02 PRIMARY
```

Now pg02 has generated new transactions on the new timeline.

Meanwhile, pg01's old data directory belongs to the previous history.

So:

```text
pg01's history
       !=
pg02's current history
```

The old primary must be reconciled before becoming a replica.

---

# 13. Where `pg_rewind` helps

We configured:

```yaml
use_pg_rewind: true
```

`pg_rewind` can compare the old primary with the current primary and synchronize the diverged data directory where its requirements are satisfied.

Conceptually:

```text
OLD pg01
Timeline 1
    |
    | pg_rewind
    v
CURRENT pg02
Timeline 2
```

After alignment:

```text
pg01
REPLICA
   ^
   |
   | WAL
   |
pg02
PRIMARY
```

This can be much faster than rebuilding a large replica from scratch.

---

# 14. If pg01 does not rejoin

This is a realistic production-support situation.

Run:

```bash
patronictl -c /etc/patroni/patroni.yml list
```

If `pg01` remains failed, check its Patroni logs:

```bash
journalctl -u patroni -n 100
```

or:

```bash
journalctl -u patroni -f
```

Look for messages related to:

```text
DCS connection
PostgreSQL startup
replication
authentication
timeline divergence
pg_rewind
data directory
WAL
```

Don't immediately delete the PostgreSQL data directory.

First determine **why** it failed to rejoin.

---

# 15. `patronictl reinit`

Sometimes the old node needs to be rebuilt as a fresh replica.

Patroni provides:

```bash
patronictl -c /etc/patroni/patroni.yml reinit pg-ha-cluster pg01
```

Check your installed syntax first:

```bash
patronictl reinit --help
```

A reinit conceptually does this:

```text
Broken / unusable replica
          |
          v
Discard/re-create replica data
          |
          v
Copy from healthy cluster source
          |
          v
Start as replica
```

This is different from failover.

You're not changing the primary.

You're rebuilding a cluster member.

---

# 16. `pg_rewind` vs reinitialize

This distinction is very useful.

| Situation                                       | Typical approach     |
| ----------------------------------------------- | -------------------- |
| Old primary diverged but can be reconciled      | `pg_rewind`          |
| Replica data badly broken                       | Reinitialize         |
| Replica missing required WAL and cannot recover | Often reinitialize   |
| Healthy replica                                 | Do nothing           |
| Current primary                                 | Never blindly reinit |

Think:

```text
pg_rewind
   =
repair/reconcile existing data


reinit
   =
rebuild the replica
```

---

# 17. Never do this blindly

After a failover, don't SSH into the old primary and run:

```bash
sudo systemctl start postgresql
```

without understanding its Patroni state.

Suppose:

```text
pg02 = new PRIMARY
```

and you manually force the old `pg01` PostgreSQL instance into an unsafe writable state.

You could create:

```text
pg01 PRIMARY?       pg02 PRIMARY
     |                  |
   WRITE              WRITE
```

That is exactly the kind of situation HA software is designed to prevent.

When Patroni owns PostgreSQL, operate through the Patroni model.

---

# 18. What happens to application connections?

Suppose your application connects directly to:

```text
192.168.10.11:5432
```

When `pg01` fails:

```text
Application
     |
     v
192.168.10.11
     X
```

Patroni can successfully promote `pg02`, but the application's connection string still points to the dead server.

This is why **Patroni alone is not the entire client connectivity solution**.

You need something like:

```text
Application
     |
     v
HAProxy / Load Balancer
     |
     +----> current primary
```

or another suitable service-discovery/routing mechanism.

We'll cover that next.

---

# 19. What HAProxy will do

Patroni exposes role-aware REST endpoints.

Conceptually:

```text
HAProxy
   |
   +----> pg01:8008
   |
   +----> pg02:8008
   |
   +----> pg03:8008
```

HAProxy can ask:

```text
Are you the primary?
```

Suppose before failure:

```text
pg01 /primary -> healthy
pg02 /primary -> not primary
pg03 /primary -> not primary
```

HAProxy sends writes to:

```text
pg01:5432
```

After failover:

```text
pg01 -> down
pg02 /primary -> healthy
pg03 /primary -> not primary
```

HAProxy switches new connections toward:

```text
pg02:5432
```

Now the application can use a stable address instead of knowing the current Patroni leader.

---

# 20. Production troubleshooting mindset

If automatic failover does **not** happen, don't immediately force one.

Check these layers:

```text
1. Is the old primary really unavailable?

2. Is etcd healthy?

3. Does etcd have quorum?

4. Can replicas reach etcd?

5. Are replicas healthy?

6. Are replicas streaming?

7. Is replication lag acceptable?

8. Is a replica eligible for promotion?

9. What do Patroni logs say?

10. Is failover blocked by configuration?
```

That's much safer than immediately running:

```bash
patronictl failover
```

A manual failover command should be an informed decision, not the first troubleshooting step.

---

# 21. Useful commands during an incident

Patroni state:

```bash
patronictl -c /etc/patroni/patroni.yml list
```

Patroni logs:

```bash
journalctl -u patroni -f
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
    sync_state
FROM pg_stat_replication;
```

Patroni local REST state:

```bash
curl http://localhost:8008/
```

etcd health:

```bash
etcdctl \
  --endpoints=http://192.168.10.11:2379,http://192.168.10.12:2379,http://192.168.10.13:2379 \
  endpoint health
```

These commands give you different views of the same HA system.

---

# 22. Your failover exercise

Run this sequence several times until it feels natural:

```text
STEP 1
patronictl list

pg01 PRIMARY
pg02 REPLICA
pg03 REPLICA


STEP 2
Insert test data


STEP 3
Stop Patroni on pg01


STEP 4
Watch patronictl list


STEP 5
Identify new primary


STEP 6
SELECT pg_is_in_recovery();


STEP 7
Insert data on new primary


STEP 8
Start Patroni on pg01


STEP 9
Watch pg01 rejoin


STEP 10
Verify replication
```

The important part isn't just seeing failover happen.

You should be able to explain:

> `pg01` lost leadership, the remaining Patroni members coordinated through the DCS, an eligible standby was promoted, PostgreSQL started a new timeline, and the old primary later had to rejoin the cluster as a replica.

If you can explain that comfortably, you now understand the core Patroni failover mechanism.

## Next — Lesson 8: HAProxy + Stable Application Endpoint

Next we'll solve the biggest problem still remaining:

```text
How does the application automatically find
the new PostgreSQL primary after failover?
```

We'll configure **HAProxy**, connect it to Patroni's REST health checks, create a stable PostgreSQL endpoint, test application connections, trigger another failover, and verify that new database connections automatically move to the new primary.
