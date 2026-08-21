## Lesson 1 — What problem does Patroni solve?

Imagine you have one PostgreSQL server:

```text
Application
     |
     v
 PostgreSQL
   pg01
```

If `pg01` crashes, your application loses its database.

A common solution is to maintain another PostgreSQL server containing a continuously updated copy:

```text
                  WAL / Streaming
                   Replication
                       |
                       v
Application ---> pg01 ----------> pg02
                 PRIMARY          STANDBY
```

PostgreSQL itself provides the replication.

But now there is another problem:

> What happens when `pg01` suddenly dies?

Someone needs to determine that the primary is really unavailable, choose an appropriate standby, promote it, avoid having two primaries, and make sure applications reach the new primary.

That's where **Patroni** comes in.

## 1. PostgreSQL vs Patroni

This distinction is very important.

**PostgreSQL manages the database.**

It handles data, SQL, transactions, WAL, replication, users, indexes, backups, etc.

**Patroni manages PostgreSQL HA.**

Patroni controls PostgreSQL processes and coordinates which PostgreSQL instance should be the **leader/primary**.

So Patroni does **not replace PostgreSQL replication**. It automates and coordinates the HA environment around PostgreSQL.

The official Patroni documentation describes Patroni as a template for building highly available PostgreSQL systems.

[Official Patroni Documentation](https://patroni.readthedocs.io/en/latest/?utm_source=chatgpt.com)

## 2. What is etcd?

This is usually one of the confusing parts for beginners.

Suppose we have:

```text
pg01
pg02
pg03
```

Every Patroni node needs a reliable way to coordinate:

> "Who currently owns the leader lock?"

Patroni uses a **Distributed Configuration Store (DCS)** for this coordination.

One common choice is **etcd**.

Think of etcd as the coordination system rather than the database containing your application data.

```text
              etcd cluster
          "pg01 holds leader lock"
                 / | \
                /  |  \
               /   |   \
              v    v    v

           Patroni Patroni Patroni
              |      |      |
             pg01   pg02   pg03

           PRIMARY REPLICA REPLICA
```

Patroni supports several distributed configuration systems, including etcd and Consul, as well as Kubernetes-based coordination.

## 3. What happens when the primary fails?

Suppose:

```text
pg01 = PRIMARY
pg02 = REPLICA
pg03 = REPLICA
```

Everything is healthy:

```text
              PostgreSQL replication

           +--------> pg02
           |
pg01 ------+
PRIMARY    |
           +--------> pg03
```

Now `pg01` crashes.

```text
pg01
 XXX
DOWN
```

Patroni detects that the existing leader is no longer healthy.

The remaining Patroni members coordinate through the DCS, and an eligible replica can be promoted.

For example:

```text
BEFORE

pg01       pg02       pg03
PRIMARY    REPLICA    REPLICA


pg01 FAILS
   X


AFTER

pg01       pg02       pg03
DOWN       PRIMARY    REPLICA
```

That automatic promotion is one of the main reasons companies deploy Patroni.

## 4. Where does HAProxy fit?

There's another problem.

Your application might originally connect to:

```text
pg01:5432
```

After failover, the primary could be:

```text
pg02:5432
```

We don't want every application to manually figure this out.

A proxy/load-balancing layer can provide a stable endpoint.

For example:

```text
              Application
                   |
                   v
                HAProxy
                   |
          "Where is PRIMARY?"
                   |
        +----------+----------+
        |          |          |
        v          v          v
      pg01       pg02       pg03
     replica    PRIMARY     replica
```

Patroni exposes REST API health endpoints that tools such as HAProxy can use to determine node roles and health.

## 5. Put everything together

A typical learning architecture looks something like:

```text
                   APPLICATION
                        |
                        v
                    HAProxy
                        |
              PostgreSQL :5432
                        |
          +-------------+-------------+
          |             |             |
          v             v             v

        pg01           pg02          pg03
      PostgreSQL     PostgreSQL    PostgreSQL
       Patroni        Patroni       Patroni
      PRIMARY        REPLICA       REPLICA

          |             |             |
          +-------------+-------------+
                        |
                        v
                  etcd cluster
```

Remember these four components:

| Component                  | Main job                                                             |
| -------------------------- | -------------------------------------------------------------------- |
| **PostgreSQL**             | Stores and processes your data                                       |
| **PostgreSQL replication** | Copies changes from primary to replicas                              |
| **Patroni**                | Manages PostgreSQL HA and failover                                   |
| **etcd**                   | Provides distributed coordination                                    |
| **HAProxy**                | Provides a stable connection path to the appropriate PostgreSQL node |

## 6. Five terms you need to know

**Primary** — PostgreSQL instance currently accepting normal read/write activity.

**Replica / Standby** — PostgreSQL instance receiving changes from the primary.

**WAL** — Write-Ahead Log. PostgreSQL records database changes in WAL, and streaming replication sends WAL information to standby servers.

**Switchover** — You intentionally move the primary role from one healthy node to another, often for maintenance.

```text
pg01 PRIMARY       pg02 REPLICA
       |
       | planned switchover
       v
pg01 REPLICA       pg02 PRIMARY
```

**Failover** — The primary unexpectedly fails and another node takes over.

```text
pg01 PRIMARY       pg02 REPLICA
     X
   FAILURE
       |
       v
pg01 DOWN          pg02 PRIMARY
```

Knowing the difference between **switchover and failover** is especially important in production support.

## Your first exercise

Before installing anything, make sure you can explain this scenario:

```text
Initial state:

pg01 = Primary
pg02 = Replica
pg03 = Replica


             pg01
            PRIMARY
             /  \
            /    \
           v      v
        pg02      pg03
       REPLICA   REPLICA


pg01 crashes.


New state:

pg01 = Down
pg02 = Primary
pg03 = Replica
```

Ask yourself:

**Who stores the actual application data?**
PostgreSQL.

**Who replicates the database?**
PostgreSQL streaming replication.

**Who manages the HA behavior?**
Patroni.

**Who provides distributed coordination?**
etcd in our example.

**How can applications get a stable database endpoint?**
HAProxy or another suitable connection-routing layer.

Once those concepts are clear, the next lesson should be **PostgreSQL WAL and streaming replication**. That's the foundation you need before we install Patroni.

After that, we'll build a **3-node lab**, configure `etcd`, install Patroni, create `patroni.yml`, use `patronictl`, perform a switchover, kill the primary, watch automatic failover happen, and troubleshoot common failures.
