# Lesson 5 — `patroni.yml` and Bootstrap

Now we connect PostgreSQL + Patroni + etcd and create our first HA cluster.

We'll continue using:

```text
pg01  192.168.10.11
pg02  192.168.10.12
pg03  192.168.10.13
```

> These IPs and passwords are **lab examples only**. Don't use these credentials in production.

## 1. What is `patroni.yml`?

Every Patroni node needs a configuration file.

We'll use:

```text
/etc/patroni/patroni.yml
```

The three files will be almost identical. The main differences are the node's **name, IP address, and PostgreSQL connection address**.

The basic relationship is:

```text
/etc/patroni/patroni.yml
        |
        +-- cluster name
        +-- node name
        +-- Patroni REST API
        +-- etcd addresses
        +-- PostgreSQL settings
        +-- bootstrap settings
        +-- authentication
```

The official Patroni configuration reference is worth keeping open while doing the lab:

[Patroni YAML configuration documentation](https://patroni.readthedocs.io/en/latest/yaml_configuration.html?utm_source=chatgpt.com)

---

# 2. Create directories

Run on **all three nodes**:

```bash
sudo mkdir -p /etc/patroni
sudo mkdir -p /var/lib/patroni
```

The exact PostgreSQL data directory depends on your installation. For this lesson, we'll use:

```text
/var/lib/patroni/data
```

We'll also need the OS account running Patroni/PostgreSQL to own the required directories.

---

# 3. Our Patroni cluster name

We'll call the cluster:

```text
pg-ha-cluster
```

In Patroni terminology, this is the **scope**.

So:

```yaml
scope: pg-ha-cluster
```

All three members must use the same scope.

```text
                pg-ha-cluster

       pg01          pg02          pg03
        |              |             |
      Patroni        Patroni       Patroni
```

If you accidentally configure:

```text
pg01 -> scope: production

pg02 -> scope: test
```

Patroni sees them as different clusters.

---

# 4. Node name

Each member needs a unique name.

`pg01`:

```yaml
name: pg01
```

`pg02`:

```yaml
name: pg02
```

`pg03`:

```yaml
name: pg03
```

Think:

```text
scope = cluster identity
name  = server identity
```

---

# 5. Configure Patroni REST API

Patroni provides an HTTP REST API.

On `pg01`:

```yaml
restapi:
  listen: 0.0.0.0:8008
  connect_address: 192.168.10.11:8008
```

On `pg02`:

```yaml
restapi:
  listen: 0.0.0.0:8008
  connect_address: 192.168.10.12:8008
```

And `pg03`:

```yaml
restapi:
  listen: 0.0.0.0:8008
  connect_address: 192.168.10.13:8008
```

Remember:

```text
5432 = PostgreSQL

8008 = Patroni REST API
```

Later HAProxy can use Patroni's REST API to determine which node currently has a particular role.

[Patroni REST API documentation](https://patroni.readthedocs.io/en/latest/rest_api.html?utm_source=chatgpt.com)

---

# 6. Connect Patroni to etcd

All Patroni members need to know how to reach the DCS.

Because we're using etcd v3:

```yaml
etcd3:
  hosts:
    - 192.168.10.11:2379
    - 192.168.10.12:2379
    - 192.168.10.13:2379
```

Conceptually:

```text
           Patroni
              |
              v

     +--------+--------+
     |        |        |
     v        v        v

   etcd1    etcd2    etcd3
   :2379    :2379    :2379
```

All three Patroni servers use the same etcd cluster.

---

# 7. Bootstrap configuration

Now we need to tell Patroni:

> When this cluster doesn't exist yet, how should PostgreSQL be created?

That's the `bootstrap` section.

A simple lab configuration could include:

```yaml
bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10

    postgresql:
      use_pg_rewind: true
      use_slots: true

      parameters:
        wal_level: replica
        hot_standby: "on"
        max_wal_senders: 10
        max_replication_slots: 10
```

You've already seen most of these concepts.

---

# 8. Connect Lesson 2 to the configuration

Remember:

```yaml
wal_level: replica
```

This supports WAL information required for replication.

Then:

```yaml
max_wal_senders: 10
```

This allows WAL sender processes for replication connections.

Then:

```yaml
max_replication_slots: 10
```

Remember our discussion about replication slots?

```text
PRIMARY
   |
   +-------- WAL --------> REPLICA
```

Patroni can manage replication slots for members.

And:

```yaml
hot_standby: "on"
```

allows standby servers to accept read-only queries while they are in recovery.

---

# 9. `pg_rewind`

This one becomes very important after failover.

Suppose:

```text
pg01 = PRIMARY
pg02 = REPLICA
```

Then `pg01` fails:

```text
pg01 X
```

and Patroni promotes:

```text
pg02 = PRIMARY
```

Later `pg01` returns.

We don't want it coming back believing its old timeline is still authoritative.

The cluster may need to bring the old primary back into alignment with the new primary.

`pg_rewind` can help synchronize a diverged PostgreSQL data directory with the current primary under the right conditions.

So:

```yaml
use_pg_rewind: true
```

tells Patroni it can use that mechanism where applicable.

[PostgreSQL pg_rewind documentation](https://www.postgresql.org/docs/current/app-pgrewind.html?utm_source=chatgpt.com)

---

# 10. PostgreSQL authentication

We'll create two important PostgreSQL credentials.

### Superuser

For our lab:

```text
postgres / LabPostgres123
```

### Replication account

```text
replicator / LabReplication123
```

Again, these are deliberately obvious **training passwords**. Use proper secret management in production.

The replication user exists because replicas need to connect to the primary.

Conceptually:

```text
pg02
 |
 | user = replicator
 | password = ...
 |
 +---------------------> pg01
                          PRIMARY
```

---

# 11. Complete `pg01` configuration

Create:

```bash
sudo nano /etc/patroni/patroni.yml
```

For `pg01`, the core configuration looks like this:

```yaml
scope: pg-ha-cluster
namespace: /service/
name: pg01

restapi:
  listen: 0.0.0.0:8008
  connect_address: 192.168.10.11:8008

etcd3:
  hosts:
    - 192.168.10.11:2379
    - 192.168.10.12:2379
    - 192.168.10.13:2379

bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10

    postgresql:
      use_pg_rewind: true
      use_slots: true

      parameters:
        wal_level: replica
        hot_standby: "on"
        max_wal_senders: 10
        max_replication_slots: 10

  initdb:
    - encoding: UTF8
    - data-checksums

  pg_hba:
    - host replication replicator 192.168.10.0/24 scram-sha-256
    - host all all 192.168.10.0/24 scram-sha-256

postgresql:
  listen: 0.0.0.0:5432
  connect_address: 192.168.10.11:5432

  data_dir: /var/lib/patroni/data

  authentication:
    superuser:
      username: postgres
      password: LabPostgres123

    replication:
      username: replicator
      password: LabReplication123
```

We'll improve security and system integration later.

---

# 12. `pg_hba`

This part:

```yaml
pg_hba:
  - host replication replicator 192.168.10.0/24 scram-sha-256
  - host all all 192.168.10.0/24 scram-sha-256
```

eventually controls PostgreSQL client authentication rules.

The first line allows our replication account to connect from the lab network.

```text
pg02
 |
 | replicator
 |
 v
pg01
```

The second allows authenticated database connections from the lab network.

In production, don't casually allow an entire broad network. Limit access to the addresses and users that actually need it.

---

# 13. Configure pg02

Most of `pg02` is identical.

Change:

```yaml
name: pg02
```

REST API:

```yaml
restapi:
  listen: 0.0.0.0:8008
  connect_address: 192.168.10.12:8008
```

PostgreSQL:

```yaml
postgresql:
  listen: 0.0.0.0:5432
  connect_address: 192.168.10.12:5432
```

Everything representing cluster-wide settings stays consistent.

---

# 14. Configure pg03

Likewise:

```yaml
name: pg03
```

REST:

```yaml
connect_address: 192.168.10.13:8008
```

PostgreSQL:

```yaml
connect_address: 192.168.10.13:5432
```

So:

```text
               SAME CLUSTER

        scope = pg-ha-cluster

               |
       +-------+-------+
       |       |       |
       v       v       v

     pg01    pg02    pg03
```

but:

```text
name/address = unique
```

---

# 15. Very important bootstrap concept

Here's something new Patroni administrators need to understand.

The `bootstrap.dcs` configuration is primarily used when **initializing the cluster**.

After the cluster has been bootstrapped, dynamic configuration is stored in the DCS.

So don't assume:

> "I changed `bootstrap.dcs` in patroni.yml, therefore the running cluster configuration changed."

It may not.

For a running cluster, you'll later learn commands such as:

```bash
patronictl edit-config
```

This distinction matters a lot in production.

[Patroni dynamic configuration documentation](https://patroni.readthedocs.io/en/latest/dynamic_configuration.html?utm_source=chatgpt.com)

---

# 16. Check existing PostgreSQL

Before starting Patroni, make sure Ubuntu's automatically created PostgreSQL instance isn't already occupying port `5432`.

For example:

```bash
sudo ss -lntp | grep 5432
```

and:

```bash
pg_lsclusters
```

If an independently managed PostgreSQL instance is already running, don't have Patroni fight with it over the same port/data directory.

For a clean disposable lab, we want Patroni managing the PostgreSQL instance we're creating.

---

# 17. Permissions

The account running Patroni needs appropriate permissions for the PostgreSQL data directory.

For example, if we're running Patroni as the `postgres` OS user:

```bash
sudo mkdir -p /var/lib/patroni/data
sudo chown -R postgres:postgres /var/lib/patroni
sudo chmod 700 /var/lib/patroni/data
```

Also protect the configuration because it currently contains passwords:

```bash
sudo chown postgres:postgres /etc/patroni/patroni.yml
sudo chmod 600 /etc/patroni/patroni.yml
```

Do that on all three nodes.

---

# 18. Bootstrap pg01

Now comes the interesting part.

Start Patroni on **pg01 first**.

For our initial foreground test:

```bash
sudo -u postgres /opt/patroni/venv/bin/patroni /etc/patroni/patroni.yml
```

Watch the logs.

Conceptually, Patroni checks:

```text
Does pg-ha-cluster already exist in etcd?
             |
            NO
             |
             v
Can I acquire initialization/leadership?
             |
            YES
             |
             v
Run initdb
             |
             v
Create PostgreSQL cluster
             |
             v
Start PostgreSQL
             |
             v
pg01 becomes leader
```

Eventually we want:

```text
pg01
 |
 +-- Patroni running
 |
 +-- PostgreSQL running
 |
 +-- Role: Leader
```

---

# 19. Check Patroni REST API

From another terminal:

```bash
curl http://192.168.10.11:8008/
```

Patroni should return JSON describing the PostgreSQL/Patroni state.

You can also try the role-oriented endpoint:

```bash
curl -i http://192.168.10.11:8008/primary
```

When `pg01` is the current primary, the HTTP response should indicate that it satisfies the primary check.

This API becomes useful when we introduce HAProxy.

---

# 20. Use `patronictl`

Now try:

```bash
/opt/patroni/venv/bin/patronictl \
  -c /etc/patroni/patroni.yml \
  list
```

Initially you may see only:

```text
Cluster: pg-ha-cluster

Member    Role
------    ----
pg01      Leader
```

That's already a major milestone.

You have your first PostgreSQL database controlled by Patroni.

---

# 21. Start pg02

Now start Patroni on `pg02`:

```bash
sudo -u postgres /opt/patroni/venv/bin/patroni /etc/patroni/patroni.yml
```

Patroni checks the DCS and discovers:

```text
pg-ha-cluster already exists.

Leader = pg01
```

Therefore `pg02` shouldn't create a completely separate primary.

Instead, it needs to become a replica of the existing cluster.

Conceptually:

```text
               pg01
              PRIMARY
                 |
                 | base backup /
                 | replication
                 v
               pg02
              REPLICA
```

Patroni handles the replica creation process according to its configured replica creation methods.

---

# 22. Start pg03

Do the same on `pg03`:

```bash
sudo -u postgres /opt/patroni/venv/bin/patroni /etc/patroni/patroni.yml
```

Eventually:

```text
                  pg01
                 PRIMARY
                  /  \
                 /    \
              WAL      WAL
               /        \
              v          v

            pg02        pg03
           REPLICA     REPLICA
```

---

# 23. The moment we've been waiting for

Run:

```bash
/opt/patroni/venv/bin/patronictl \
  -c /etc/patroni/patroni.yml \
  list
```

You should see a table conceptually similar to:

```text
+ Cluster: pg-ha-cluster ---------------------+
| Member | Host          | Role    | State     |
+--------+---------------+---------+-----------+
| pg01   | 192.168.10.11 | Leader  | running   |
| pg02   | 192.168.10.12 | Replica | streaming |
| pg03   | 192.168.10.13 | Replica | streaming |
+--------+---------------+---------+-----------+
```

🎯 **This is the output you need to learn to read quickly.**

Ask:

```text
Who is leader?
      |
     pg01

Which nodes are replicas?
      |
   pg02 + pg03

Are they running?
      |
     yes

Is there replication lag?
      |
check Lag column
```

---

# 24. Verify PostgreSQL itself

Don't rely only on Patroni.

Connect to the primary:

```bash
psql -h 192.168.10.11 -U postgres -d postgres
```

Run:

```sql
SELECT pg_is_in_recovery();
```

On the primary:

```text
f
```

means:

> PostgreSQL is **not** in recovery.

On a replica:

```sql
SELECT pg_is_in_recovery();
```

should return:

```text
t
```

That means the PostgreSQL instance is operating as a standby/in recovery.

This is an excellent troubleshooting command.

---

# 25. Verify replication

On the primary, run:

```sql
SELECT
    application_name,
    client_addr,
    state,
    sync_state
FROM pg_stat_replication;
```

You should see replication connections from your replicas.

Conceptually:

```text
application_name    client_addr       state
------------------------------------------------
pg02                192.168.10.12    streaming
pg03                192.168.10.13    streaming
```

This is PostgreSQL telling you about replication directly.

Notice the troubleshooting distinction:

```text
patronictl list
      |
      +--> Patroni's cluster view


pg_stat_replication
      |
      +--> PostgreSQL's replication view
```

Knowing both is important.

---

# 26. Test actual data replication

On the primary:

```sql
CREATE DATABASE training;
```

Connect:

```bash
psql -h 192.168.10.11 -U postgres -d training
```

Create:

```sql
CREATE TABLE employees (
    id integer PRIMARY KEY,
    name text
);
```

Insert:

```sql
INSERT INTO employees VALUES
(1, 'Alice'),
(2, 'Bob'),
(3, 'Charlie');
```

Check:

```sql
SELECT * FROM employees;
```

Now connect to one of the replicas:

```bash
psql -h 192.168.10.12 -U postgres -d training
```

Run:

```sql
SELECT * FROM employees;
```

You should see the rows there as replication catches up.

---

# 27. Try writing to the replica

While connected to `pg02`:

```sql
INSERT INTO employees
VALUES (4, 'David');
```

You should get an error because the standby is read-only.

That's exactly what we want.

```text
                pg01
               PRIMARY
                  |
            READ + WRITE
                  |
            WAL replication
             /          \
            v            v

          pg02          pg03
         REPLICA       REPLICA

        READ ONLY      READ ONLY
```

---

# What you've built

You've moved from:

```text
PostgreSQL server
```

to:

```text
                Patroni HA Cluster

                      etcd
                       |
          +------------+------------+
          |            |            |
          v            v            v

        pg01          pg02         pg03
      PostgreSQL    PostgreSQL   PostgreSQL
       Patroni       Patroni      Patroni

       LEADER        REPLICA      REPLICA
          |
          +------------ WAL ----------->
```

That's a significant milestone.

## Commands worth remembering

You don't need to memorize the whole YAML yet. Start remembering these:

```bash
patronictl -c /etc/patroni/patroni.yml list
```

**What's the Patroni cluster state?**

```sql
SELECT pg_is_in_recovery();
```

**Am I primary or standby?**

```sql
SELECT * FROM pg_stat_replication;
```

**What replicas does this primary currently see?**

```bash
curl http://localhost:8008/
```

**What does Patroni's REST API say about this node?**

And:

```bash
journalctl -u patroni
```

will become one of your best friends once Patroni runs under systemd.

---

## Next — Lesson 6: Switchover

Before intentionally breaking servers, we'll learn the safe operation first.

We'll take:

```text
BEFORE

pg01  PRIMARY
pg02  REPLICA
pg03  REPLICA
```

and deliberately move leadership:

```text
AFTER

pg01  REPLICA
pg02  PRIMARY
pg03  REPLICA
```

using **`patronictl switchover`**.

Then we'll verify the new primary using PostgreSQL itself, verify the old primary became a replica, check the timeline, confirm our test data survived, and understand exactly what Patroni did behind the scenes.

After that comes the really useful exercise: **Lesson 7 — kill the primary and watch Patroni perform an automatic failover.**
