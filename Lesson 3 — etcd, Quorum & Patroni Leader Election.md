## Lesson 3 — etcd, Quorum & Patroni Leader Election

This is one of the most important Patroni concepts. Once you understand **etcd + quorum + leader lock**, Patroni failover becomes much easier to understand.

### 1. Why does Patroni need etcd?

Suppose we have three PostgreSQL servers:

```text
pg01
PostgreSQL + Patroni

pg02
PostgreSQL + Patroni

pg03
PostgreSQL + Patroni
```

Patroni instances need a shared place to coordinate information such as:

> Which PostgreSQL node currently owns the leader role?

For our lab, we'll use **etcd** as that Distributed Configuration Store (DCS).

Patroni supports etcd and other DCS options. [Patroni documentation — DCS options](https://patroni.readthedocs.io/en/latest/?utm_source=chatgpt.com)

Think of the responsibilities like this:

```text
PostgreSQL = Database

Patroni    = HA manager

etcd       = Distributed coordination

HAProxy    = Connection routing
```

---

# 2. Our architecture

Eventually we'll build something similar to:

```text
                 Application
                      |
                      v
                   HAProxy
                      |
                      v
       +--------------+--------------+
       |              |              |
       v              v              v
     pg01           pg02           pg03
   PostgreSQL     PostgreSQL     PostgreSQL
    Patroni        Patroni        Patroni
   PRIMARY         REPLICA        REPLICA

       |              |              |
       v              v              v

     etcd1 <-------> etcd2 <-------> etcd3
```

The etcd members communicate with each other and maintain consistent distributed state.

---

# 3. What is quorum?

This is critical.

**Quorum means a majority of members.**

For a 3-member etcd cluster:

```text
Members = 3

Majority = 2
```

So etcd needs **2 of the 3 members** to form a majority for operations requiring consensus.

For five members:

```text
Members = 5

Majority = 3
```

A useful formula is:

```text
Quorum = floor(N / 2) + 1
```

Examples:

| Members | Quorum | Member failures tolerated |
| ------: | -----: | ------------------------: |
|       1 |      1 |                         0 |
|       2 |      2 |                         0 |
|       3 |      2 |                         1 |
|       4 |      3 |                         1 |
|       5 |      3 |                         2 |

Notice something interesting:

**Going from 3 to 4 members doesn't increase failure tolerance.**

That's one reason odd-sized consensus clusters such as **3 or 5** are common.

[etcd FAQ — cluster size and fault tolerance](https://etcd.io/docs/v3.5/faq/?utm_source=chatgpt.com)

---

# 4. What does Patroni store there?

Simplifying a little, Patroni uses the DCS to coordinate cluster information including who owns the **leader lock**.

Imagine:

```text
etcd

Patroni cluster: production-db

Leader:
    pg01
```

All Patroni members can see that coordination state.

```text
             etcd
        Leader = pg01
          /    |    \
         /     |     \
        v      v      v
      pg01    pg02   pg03

     PRIMARY  replica replica
```

The important thing is that nodes aren't independently deciding:

> "I feel like becoming primary."

They coordinate through the DCS.

---

# 5. The leader lock

Let's simplify the concept.

Assume `pg01` currently owns the Patroni leader lock:

```text
                etcd
                  |
          Leader lock = pg01
                  |
                  v
                pg01
              PRIMARY
```

Patroni doesn't acquire this lock permanently.

The lock has a **TTL — Time To Live**.

The Patroni leader must continue refreshing its leadership information.

Conceptually:

```text
pg01 Patroni
     |
     | "I'm alive"
     v
   etcd

leader = pg01
TTL refreshed
```

This happens continuously while things are healthy.

---

# 6. What if the primary crashes?

Suppose:

```text
pg01 = PRIMARY
pg02 = REPLICA
pg03 = REPLICA
```

Then:

```text
pg01
  X
CRASH
```

`pg01` can no longer maintain its leadership lock.

Conceptually:

```text
                etcd
                  |
          pg01 leader lock
                  |
              expires
```

The remaining Patroni members can then coordinate to determine whether an eligible replica can become leader.

For example:

```text
BEFORE

pg01            pg02            pg03
PRIMARY         REPLICA         REPLICA
  |
  X


FAILOVER

                 etcd
                  |
           leader election
                  |
                  v
                pg02


AFTER

pg01            pg02            pg03
DOWN            PRIMARY         REPLICA
```

PostgreSQL `pg02` gets promoted.

---

# 7. Why not promote immediately?

This is where distributed systems get interesting.

Suppose `pg01` disappears from `pg02`'s network.

Does that definitely mean `pg01` is dead?

No.

Maybe:

```text
pg01 is alive
```

but:

```text
network between pg01 and pg02 is broken
```

That's very different.

Consider:

```text
           Network problem

     pg01       X       pg02
   PRIMARY             REPLICA
```

`pg02` shouldn't simply say:

> "I can't see pg01, therefore I'll become primary."

If `pg01` is still serving writes, you could potentially end up with two writable databases.

That's a dangerous situation.

---

# 8. Split brain

The general problem is called **split brain**.

Conceptually:

```text
         Network partition
              X
              X
              X

pg01                     pg02
PRIMARY                  PRIMARY

App A                    App B
  |                        |
WRITE                    WRITE
  |                        |
  v                        v
pg01                     pg02
```

Now both databases can diverge.

For example:

```text
pg01:

customer balance = $500


pg02:

customer balance = $700
```

Which database is correct?

That's a serious problem.

Patroni's DCS-based coordination is designed to avoid independent promotions like this.

---

# 9. Network partition example

Let's make this more realistic.

We have:

```text
              etcd1
                |
              etcd2
                |
              etcd3


pg01           pg02           pg03
PRIMARY        REPLICA        REPLICA
```

Suppose `pg01` becomes isolated:

```text
                 NETWORK PARTITION

 pg01       X        pg02 + pg03 + etcd quorum
```

If Patroni on `pg01` can no longer safely maintain its leader lock through the DCS, it should not simply continue behaving indefinitely as the authoritative primary.

Meanwhile, the side that can reach a healthy DCS quorum can coordinate leadership.

This is why understanding the **DCS** is so important.

---

# 10. What happens if one etcd server fails?

Suppose:

```text
etcd1 = UP
etcd2 = UP
etcd3 = DOWN
```

We still have:

```text
2 / 3
```

That's quorum.

The etcd cluster can continue operating.

```text
etcd1 <----> etcd2      etcd3
  UP           UP         X
                        DOWN

       QUORUM EXISTS
```

Good.

---

# 11. What if two etcd servers fail?

Now:

```text
etcd1 = UP
etcd2 = DOWN
etcd3 = DOWN
```

We only have:

```text
1 / 3
```

No majority.

```text
etcd1       etcd2       etcd3
  UP          X           X

   1 member available

   NO QUORUM
```

This is very different from a normal PostgreSQL replica failure.

The DCS can't safely accept the consensus operations Patroni depends on.

The important production lesson is:

> **Don't treat an etcd outage as "only a monitoring problem."**

Your HA control plane depends on it.

---

# 12. etcd leader vs PostgreSQL leader

This confuses almost everyone initially.

There can be an **etcd/Raft leader** and a **Patroni/PostgreSQL leader**.

They are not the same thing.

For example:

```text
ETCD CLUSTER

etcd1
  |
  +-- Raft leader

etcd2
etcd3
```

Meanwhile:

```text
PATRONI CLUSTER

pg01
  |
  +-- PostgreSQL PRIMARY
      Patroni leader

pg02
  +-- replica

pg03
  +-- replica
```

You generally don't care which etcd member is the Raft leader from an application perspective.

What you care about is that **etcd has quorum and is healthy**.

---

# 13. The most important Patroni timing settings

Later you'll see settings like:

```yaml
loop_wait: 10
ttl: 30
retry_timeout: 10
```

Don't copy these blindly into production. They are just useful example values for understanding the concepts.

### `loop_wait`

How frequently Patroni performs its regular HA control loop.

Conceptually:

```text
Patroni
  |
wait
  |
check cluster
  |
take required action
  |
wait
  |
repeat
```

### `ttl`

The lifetime associated with the leader lock.

The leader needs to keep refreshing its ownership.

### `retry_timeout`

Controls retry behavior around DCS operations.

We'll discuss how these settings interact when we configure our cluster rather than tuning them now.

[Patroni configuration settings](https://patroni.readthedocs.io/en/latest/patroni_configuration.html?utm_source=chatgpt.com)

---

# 14. Production scenario

Imagine you're on call.

You receive:

```text
ALERT

PostgreSQL pg01 unavailable
```

You log in and run:

```bash
patronictl list
```

You might see something conceptually like:

```text
+ Cluster: production-db --------+
| Member | Host | Role    | State     |
+--------+------+---------+-----------+
| pg01   | ...  | Replica | stopped   |
| pg02   | ...  | Leader  | running   |
| pg03   | ...  | Replica | streaming |
+--------+------+---------+-----------+
```

Your first reaction shouldn't be:

> "pg01 is down! Start PostgreSQL immediately!"

You first want to understand the cluster state.

If `pg02` has already been promoted:

```text
pg02 = NEW PRIMARY
```

blindly forcing the old database into a writable state could make the incident much worse.

**Always understand the cluster state before manually changing PostgreSQL during a Patroni incident.**

---

# 15. Another important rule

When PostgreSQL is controlled by Patroni, normally you don't want to manage PostgreSQL independently with commands such as:

```bash
systemctl start postgresql
systemctl stop postgresql
```

without understanding what Patroni is doing.

The relationship is more like:

```text
              Patroni
                 |
        manages PostgreSQL
                 |
                 v
             postgres
```

You'll typically interact through Patroni-aware operations where appropriate:

```bash
patronictl list
patronictl switchover
patronictl failover
patronictl reinit
```

We'll practice all of these.

---

# The mental model to remember

If you remember only one picture from this lesson, remember this:

```text
                  APPLICATION
                       |
                       v
                    HAProxy
                       |
                       v
        +--------------+--------------+
        |              |              |
        v              v              v

      pg01           pg02           pg03
    PostgreSQL     PostgreSQL     PostgreSQL
     Patroni        Patroni        Patroni

    PRIMARY         REPLICA        REPLICA

        \              |              /
         \             |             /
          +------------+------------+
                       |
                       v
                DCS / etcd cluster

              etcd1--etcd2--etcd3
                    quorum
```

PostgreSQL answers:

> **Where is my data?**

PostgreSQL replication answers:

> **How does the standby receive database changes?**

Patroni answers:

> **Which PostgreSQL node should be primary?**

etcd answers:

> **How can the Patroni nodes coordinate that decision safely?**

HAProxy answers:

> **Where should the application connect?**

## Quick test

Make sure these answers make sense before moving on:

| Question                                            | Answer                   |
| --------------------------------------------------- | ------------------------ |
| 3 etcd nodes need how many for quorum?              | **2**                    |
| 5 etcd nodes need how many?                         | **3**                    |
| Can a 3-node etcd cluster lose one member?          | **Yes**                  |
| Can it lose two and retain quorum?                  | **No**                   |
| Does etcd contain your PostgreSQL application data? | **No**                   |
| Does Patroni perform PostgreSQL WAL replication?    | **No**                   |
| Who performs replication?                           | **PostgreSQL**           |
| What coordinates Patroni leadership?                | **DCS, etcd in our lab** |
| Planned primary move?                               | **Switchover**           |
| Primary unexpectedly dies?                          | **Failover**             |

### Next — Lesson 4: Build the actual Patroni lab

This is where we move from theory to **hands-on work**.

We'll create a 3-node environment and go step-by-step through **Linux preparation → PostgreSQL → etcd → Patroni installation → `patroni.yml` → bootstrap → `patronictl list` → replication verification → switchover → actual primary failure → automatic failover**.

We'll also stop at each major command and explain **what it does and why we're running it**, rather than just copying commands.
