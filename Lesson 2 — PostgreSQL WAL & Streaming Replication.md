## Lesson 2 — PostgreSQL WAL & Streaming Replication

Before Patroni, you need to understand **how PostgreSQL keeps a standby synchronized with the primary**. Patroni relies heavily on this.

### 1. What is WAL?

**WAL = Write-Ahead Log.**

When an application changes data, PostgreSQL doesn't simply change the database files and call it done. It first records information about the change in WAL.

A simplified flow:

```text
Application
    |
    | UPDATE employee SET salary=...
    v
PostgreSQL
    |
    +----> WAL record
    |
    +----> Database data files
```

The important idea is:

> **Changes are recorded in WAL before the corresponding data-file changes are written to disk.**

This is fundamental to PostgreSQL crash recovery and replication.

[PostgreSQL WAL documentation](https://www.postgresql.org/docs/current/wal-intro.html?utm_source=chatgpt.com)

---

## 2. Why is WAL important for Patroni?

Consider two PostgreSQL servers:

```text
pg01                         pg02
PRIMARY                      STANDBY

Application
    |
    v
  INSERT
    |
    v
PostgreSQL
    |
    |      WAL stream
    +------------------------>
                             PostgreSQL
```

The primary generates WAL.

The standby receives WAL and replays it.

That's how `pg02` stays close to `pg01`.

This is called **streaming replication**.

Patroni doesn't replace this mechanism.

```text
             PostgreSQL
          Streaming Replication
                  |
       +----------+----------+
       |                     |
       v                     v

     pg01                  pg02
   PostgreSQL            PostgreSQL
    PRIMARY               STANDBY
       ^                     ^
       |                     |
    Patroni                Patroni
```

PostgreSQL handles replication. **Patroni manages the HA roles around it.**

[PostgreSQL Streaming Replication documentation](https://www.postgresql.org/docs/current/warm-standby.html?utm_source=chatgpt.com#STREAMING-REPLICATION)

---

## 3. Primary and standby

Let's say:

```text
pg01 = Primary
pg02 = Standby
```

Your application writes:

```sql
CREATE TABLE employee (
    id integer,
    name varchar(100)
);

INSERT INTO employee
VALUES (1, 'John');
```

Conceptually:

```text
             INSERT John
                  |
                  v
             +----------+
             |   pg01   |
             | PRIMARY  |
             +----------+
                  |
                  | WAL
                  |
                  v
             +----------+
             |   pg02   |
             | STANDBY  |
             +----------+
                  |
               Replay WAL
```

Now both PostgreSQL instances can represent the same committed database changes, subject to replication timing and configuration.

---

# 4. Asynchronous replication

This concept becomes very important when discussing Patroni failover.

PostgreSQL streaming replication is commonly configured asynchronously.

Imagine:

```text
Application
    |
    | INSERT
    v
pg01 PRIMARY
    |
    | COMMIT
    |
Application receives SUCCESS
    |
    |
    +------ WAL ------> pg02
```

The primary may confirm the transaction **before the standby has received/replayed it**.

This gives good performance, but introduces a risk.

Suppose the primary commits:

```text
Transaction #101
Transaction #102
Transaction #103
```

But the replica currently has only:

```text
#101
#102
```

Before `#103` reaches the replica:

```text
pg01
  X
CRASH
```

If `pg02` gets promoted, transaction `#103` could be missing.

This is one reason you will hear about **RPO** in HA discussions.

---

# 5. RPO and RTO

These are two terms you should know for production architecture.

### RPO — Recovery Point Objective

Think:

> **How much data can we afford to lose?**

Example:

```text
Primary:

100
101
102
103
104
105
      X crash


Replica:

100
101
102
103
```

If the replica is promoted, transactions `104` and `105` aren't present there.

For a system requiring extremely low or zero transaction loss under defined failure conditions, you need to think carefully about replication mode, acknowledgements, failure scenarios, and architecture.

---

### RTO — Recovery Time Objective

Think:

> **How long can the service remain unavailable?**

Without HA:

```text
Primary crashes
      |
DBA receives alert
      |
DBA investigates
      |
DBA chooses replica
      |
DBA promotes replica
      |
Applications reconnect

Maybe significant downtime
```

With a correctly designed Patroni environment:

```text
Primary crashes
      |
Patroni detects failure
      |
Cluster coordinates
      |
Eligible replica promoted
      |
Routing directs traffic
      |
Application reconnects
```

The goal is to reduce recovery time through automation.

---

# 6. Synchronous replication

PostgreSQL can also use synchronous replication.

Conceptually:

```text
Application
     |
    INSERT
     |
     v
  PRIMARY
     |
     | WAL
     v
  STANDBY
     |
 acknowledgement
     |
     v
  PRIMARY
     |
   COMMIT completes
     |
     v
Application
```

This can improve durability guarantees, but there are tradeoffs.

If the required synchronous standby becomes unavailable, writes can potentially wait depending on the configuration.

So:

```text
Asynchronous
     |
     +--> usually lower commit latency
     +--> replica can lag
     +--> potential data loss during some failures


Synchronous
     |
     +--> stronger durability possibilities
     +--> higher latency
     +--> standby availability affects writes
```

We'll return to this later because Patroni has important configuration around synchronous operation.

---

# 7. Replication lag

Another term you'll see constantly as a Patroni administrator is **replication lag**.

Imagine:

```text
Primary WAL position
=============================> 100 GB


Replica WAL position
========================>      99.5 GB
```

The replica is behind.

Conceptually:

```text
PRIMARY
WAL: 100

       |
       | replication
       v

REPLICA
WAL: 95

Lag = difference
```

A replica that's significantly behind may be a poor failover candidate.

Patroni therefore needs to consider whether a replica is healthy enough to participate in failover.

---

# 8. Replication slots

You'll also encounter **replication slots**.

Consider:

```text
PRIMARY
   |
   +------> Replica
```

Suppose the replica goes offline.

Meanwhile, the primary continues generating WAL:

```text
WAL 100
WAL 101
WAL 102
...
WAL 500
```

Eventually PostgreSQL may recycle WAL files that are no longer considered necessary.

A replication slot can tell PostgreSQL, conceptually:

> Keep the WAL needed by this replication consumer.

That's useful, but there's an operational danger.

If a replica remains offline for a long time while its slot remains active, retained WAL can consume significant disk space.

So this situation matters:

```text
Replica DOWN
    |
    v

Replication slot retains WAL
    |
    v

WAL keeps accumulating
    |
    v

Disk usage increases
    |
    v

DISK FULL   <-- serious production incident
```

That's something you'll need to monitor.

---

# 9. Important PostgreSQL configuration

Later, when we build the lab, you'll see parameters such as:

```text
wal_level
max_wal_senders
max_replication_slots
hot_standby
wal_keep_size
```

Don't memorize them yet.

For now understand their purpose.

For example:

```text
wal_level
   |
   +--> controls information written to WAL


max_wal_senders
   |
   +--> limits concurrent WAL sender processes


max_replication_slots
   |
   +--> controls available replication slots


hot_standby
   |
   +--> allows queries while server is in standby mode
```

---

# 10. How Patroni fits into all of this

Now connect Lesson 1 and Lesson 2.

```text
                   APPLICATION
                        |
                        v
                     HAProxy
                        |
              +---------+---------+
              |                   |
              v                   v

            pg01                pg02
          PostgreSQL          PostgreSQL
           PRIMARY             REPLICA
              |                   ^
              |                   |
              +------ WAL --------+
              |
          Patroni               Patroni
              \                   /
               \                 /
                +------DCS------+
                       |
                      etcd
```

There are different responsibilities:

**PostgreSQL**

```text
SQL
Transactions
WAL
Streaming replication
Data storage
```

**Patroni**

```text
HA management
Primary/replica role management
Health checks
Promotion/failover coordination
Cluster operations
```

**etcd**

```text
Distributed coordination
Leader information/lock
Cluster state
```

**HAProxy**

```text
Connection routing
```

Keeping those responsibilities separate will make troubleshooting much easier later.

---

# Production scenario

Imagine you receive an alert:

> **PostgreSQL primary pg01 is down.**

Don't immediately think:

> "Patroni problem."

Instead think through the layers:

```text
Application
     |
     v
Connection routing
     |
     v
Patroni
     |
     v
PostgreSQL
     |
     v
Replication
     |
     v
Operating System / Network / Storage
```

The actual problem could be PostgreSQL, Patroni, etcd/DCS, networking, disk space, replication, the operating system, or the connection-routing layer.

That's how we'll eventually approach Patroni troubleshooting.

---

## Quick knowledge check

Before Lesson 3, make sure you can answer these:

1. **What is WAL?**
   Write-Ahead Log.

2. **Who performs streaming replication?**
   PostgreSQL, not Patroni.

3. **What does replication lag mean?**
   A standby is behind the primary's WAL progress.

4. **What is RPO?**
   The acceptable data-loss objective.

5. **What is RTO?**
   The acceptable recovery-time objective.

6. **Why can replication slots become dangerous?**
   A disconnected consumer can cause required WAL to be retained and potentially consume disk space.

7. **Does Patroni copy database records between servers?**
   No. PostgreSQL replication handles that.

### Next: Lesson 3 — etcd, quorum and leader election

This is the **most important Patroni architecture lesson** before we start installing software. We'll cover why you normally want an odd number of DCS members, what quorum actually means, what happens when etcd nodes fail, how Patroni's leader lock works, and how the architecture helps prevent two PostgreSQL nodes from independently behaving as primary.
