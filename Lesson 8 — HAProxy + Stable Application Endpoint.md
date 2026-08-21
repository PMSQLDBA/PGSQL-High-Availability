# Lesson 8 — HAProxy + Stable Application Endpoint

So far Patroni can fail over PostgreSQL, but your application still needs a reliable way to find the current primary.

Without a proxy:

```text
Application
   |
   v
pg01:5432
```

If `pg01` fails and `pg02` becomes primary, the application still points to `pg01`.

HAProxy solves that by giving the application one stable endpoint.

```text
Application
   |
   v
HAProxy:5000
   |
   +----> pg01:5432
   +----> pg02:5432
   +----> pg03:5432
```

HAProxy checks Patroni's REST API and sends new connections only to the node that is currently primary.

## 1. Add an HAProxy server

For the lab, add a fourth VM:

```text
haproxy01   192.168.10.20
```

Architecture:

```text
                  Application
                       |
                       v
             192.168.10.20:5000
                    HAProxy
                       |
           +-----------+-----------+
           |           |           |
           v           v           v
         pg01        pg02        pg03
        :5432       :5432       :5432
       Patroni     Patroni      Patroni
        :8008       :8008        :8008

           \           |           /
            +----------+----------+
                       |
                     etcd
```

The application no longer needs to know which PostgreSQL node is primary.

---

## 2. Install HAProxy

On `haproxy01`:

```bash
sudo apt update
sudo apt install -y haproxy
```

Check:

```bash
haproxy -v
```

and:

```bash
systemctl status haproxy
```

---

## 3. First understand the Patroni health check

Before configuring HAProxy, test Patroni directly.

Assume:

```text
pg01 = PRIMARY
pg02 = REPLICA
pg03 = REPLICA
```

From `haproxy01`, run:

```bash
curl -i http://192.168.10.11:8008/primary
```

For the current primary, Patroni should return a successful HTTP status.

Now:

```bash
curl -i http://192.168.10.12:8008/primary
```

Because `pg02` is a replica, it should not satisfy the primary health check.

Same for `pg03`.

This gives HAProxy a simple rule:

```text
HTTP check succeeds
        |
        v
This server is eligible for primary traffic
```

Patroni documents role-specific REST endpoints such as `/primary`, which are commonly used by load balancers for health checks. [Patroni REST API documentation](https://patroni.readthedocs.io/en/latest/rest_api.html?utm_source=chatgpt.com)

---

# 4. Configure HAProxy

Back up the existing config:

```bash
sudo cp /etc/haproxy/haproxy.cfg \
  /etc/haproxy/haproxy.cfg.backup
```

Edit:

```bash
sudo nano /etc/haproxy/haproxy.cfg
```

For the lab, add a PostgreSQL primary listener like this:

```text
listen postgres_primary
    bind *:5000
    mode tcp

    option httpchk GET /primary
    http-check expect status 200

    default-server inter 2s fall 3 rise 2 on-marked-down shutdown-sessions

    server pg01 192.168.10.11:5432 check port 8008
    server pg02 192.168.10.12:5432 check port 8008
    server pg03 192.168.10.13:5432 check port 8008
```

Let's understand every important part.

---

# 5. `bind *:5000`

```text
bind *:5000
```

HAProxy listens on port `5000`.

Your application connects to:

```text
192.168.10.20:5000
```

instead of:

```text
192.168.10.11:5432
```

Think:

```text
Application
    |
    | PostgreSQL protocol
    v
HAProxy:5000
    |
    v
Current PostgreSQL primary:5432
```

---

# 6. Why `mode tcp`?

PostgreSQL uses its own network protocol over TCP.

Therefore:

```text
mode tcp
```

tells HAProxy that this is a TCP connection rather than ordinary HTTP application traffic.

Notice something interesting:

```text
Client traffic
      |
      | TCP / PostgreSQL
      v
    :5432
```

while health checking uses:

```text
HAProxy
   |
   | HTTP
   v
Patroni :8008
```

So HAProxy uses two different ports for two different jobs.

```text
5432 = send database traffic

8008 = determine node role/health
```

This distinction is very important.

---

# 7. `option httpchk GET /primary`

This line tells HAProxy:

```text
Ask Patroni:

GET /primary
```

So for `pg01`:

```text
HAProxy
   |
   | GET /primary
   v
pg01:8008
```

If `pg01` is primary and healthy:

```text
HTTP 200
```

HAProxy considers that backend suitable.

For a replica:

```text
GET /primary
     |
     v
not successful for primary role
```

HAProxy marks that server unavailable for this particular backend.

It doesn't mean the PostgreSQL replica itself is broken.

It means:

> This node is not currently appropriate for **primary traffic**.

---

# 8. Understand this server line

Take:

```text
server pg01 192.168.10.11:5432 check port 8008
```

There are two ports here conceptually.

Traffic goes to:

```text
192.168.10.11:5432
```

Health check goes to:

```text
192.168.10.11:8008
```

That's exactly what we need.

```text
             pg01
        +-------------+
        | PostgreSQL  | :5432 <--- application traffic
        |             |
        | Patroni API | :8008 <--- HAProxy health check
        +-------------+
```

---

# 9. Validate the HAProxy configuration

Never restart HAProxy immediately after editing.

First:

```bash
sudo haproxy -c -f /etc/haproxy/haproxy.cfg
```

You want validation to succeed.

If there is a syntax error, fix it first.

Then:

```bash
sudo systemctl restart haproxy
```

Check:

```bash
sudo systemctl status haproxy
```

And:

```bash
sudo ss -lntp | grep 5000
```

You should see HAProxy listening on port `5000`.

---

# 10. Test PostgreSQL through HAProxy

From a client machine:

```bash
psql \
  -h 192.168.10.20 \
  -p 5000 \
  -U postgres \
  -d training
```

Notice:

```text
Host = HAProxy

Port = 5000
```

You are **not connecting directly to pg01**.

Inside PostgreSQL, run:

```sql
SELECT inet_server_addr(),
       inet_server_port(),
       pg_is_in_recovery();
```

Suppose `pg01` is primary.

You should see its server address and:

```text
pg_is_in_recovery = false
```

This proves:

```text
Application
     |
     v
HAProxy
     |
     v
pg01 PRIMARY
```

---

# 11. An even better test

Run:

```sql
SELECT
    inet_server_addr() AS server_ip,
    pg_is_in_recovery() AS is_replica;
```

Before failover:

```text
server_ip       is_replica
--------------- ----------
192.168.10.11   false
```

That's an excellent lab command because it tells you exactly where HAProxy sent the connection.

---

# 12. Now trigger another failover

Starting state:

```text
HAProxy
   |
   v
pg01 PRIMARY

pg02 REPLICA
pg03 REPLICA
```

Stop Patroni on `pg01`:

```bash
sudo systemctl stop patroni
```

Now watch:

```bash
patronictl -c /etc/patroni/patroni.yml list
```

Suppose Patroni promotes `pg02`.

New state:

```text
pg01 DOWN
pg02 PRIMARY
pg03 REPLICA
```

HAProxy's checks now change.

Before:

```text
pg01 /primary -> 200
pg02 /primary -> not primary
pg03 /primary -> not primary
```

After failover:

```text
pg01 -> unavailable

pg02 /primary -> 200

pg03 /primary -> not primary
```

Therefore:

```text
HAProxy
   |
   v
pg02:5432
```

No application configuration change is required.

---

# 13. Reconnect through HAProxy

Open a **new** connection:

```bash
psql \
  -h 192.168.10.20 \
  -p 5000 \
  -U postgres \
  -d training
```

Run:

```sql
SELECT
    inet_server_addr(),
    pg_is_in_recovery();
```

Now you should see:

```text
192.168.10.12 | false
```

Excellent.

You just demonstrated:

```text
BEFORE

Application
    |
 HAProxy
    |
   pg01


FAILURE

   pg01 X


AFTER

Application
    |
 HAProxy
    |
   pg02
```

The application's database hostname and port stayed the same.

That's the stable endpoint.

---

# 14. Important: existing connections

Here's something people often misunderstand.

Suppose the application already has an active PostgreSQL connection:

```text
App ---- TCP connection ----> pg01
```

Then `pg01` dies.

That TCP session is gone.

HAProxy cannot magically move an existing PostgreSQL session from `pg01` to `pg02`.

The application usually sees a connection failure.

It needs to reconnect.

Then:

```text
NEW connection
     |
     v
HAProxy
     |
     v
pg02
```

So HA requires cooperation from the application layer too.

You generally need sensible:

```text
connection retry
connection timeout
connection pool behavior
transaction retry logic
```

Think:

```text
HAProxy redirects NEW connections.

It does not migrate an existing PostgreSQL session.
```

That's an important production point.

---

# 15. Connection pools

Applications often use a connection pool.

For example:

```text
Application
     |
Connection Pool
     |
     v
HAProxy
     |
     v
PostgreSQL
```

After failover, some pooled connections may still refer to the dead primary and fail.

The pool should detect bad connections and create new ones through HAProxy.

This is why testing only:

```bash
psql
```

isn't enough for a production application.

Eventually you should test the real application's connection behavior during failover.

---

# 16. Add a replica endpoint

HAProxy can also provide a separate endpoint for read-only replica traffic.

For example:

```text
5000 = PRIMARY

5001 = REPLICAS
```

Conceptually:

```text
                   HAProxy

Application writes ----> :5000 ----> PRIMARY


Application reads  ----> :5001 ----> REPLICAS
```

A basic replica listener can use Patroni's replica health endpoint.

For example:

```text
listen postgres_replicas
    bind *:5001
    mode tcp

    option httpchk GET /replica
    http-check expect status 200

    balance roundrobin

    server pg01 192.168.10.11:5432 check port 8008
    server pg02 192.168.10.12:5432 check port 8008
    server pg03 192.168.10.13:5432 check port 8008
```

Then your architecture becomes:

```text
                      HAProxy
                    /         \
                   /           \
              :5000             :5001
             PRIMARY           REPLICAS
                |               /   \
                v              v     v
              pg02           pg01   pg03
```

This is useful for applications that separate reads from writes.

But be careful:

> A replica may be behind the primary.

So after writing:

```sql
INSERT ...
```

immediately reading through a replica endpoint may not always provide the read-after-write consistency your application expects.

That's an application architecture decision, not just an HAProxy setting.

---

# 17. HAProxy statistics

HAProxy can expose a statistics page, which is useful in a lab.

A simple configuration could include:

```text
listen stats
    bind *:7000
    mode http

    stats enable
    stats uri /stats
    stats refresh 5s
```

Then you can inspect the HAProxy backend state through:

```text
http://192.168.10.20:7000/stats
```

In production, protect administrative/statistics interfaces appropriately rather than exposing them openly.

---

# 18. What you should see

For the primary backend, HAProxy should effectively see:

```text
pg01       DOWN
pg02       UP
pg03       DOWN
```

Don't panic about `pg03 = DOWN` in that backend.

It may actually be:

```text
PostgreSQL = healthy
Patroni = healthy
Replica = healthy
```

It is simply not eligible for:

```text
/primary
```

Similarly, in the replica backend:

```text
pg01       UP
pg02       DOWN
pg03       UP
```

because `pg02` is the primary.

That's expected.

---

# 19. Full architecture now

You've now built nearly the entire stack:

```text
                   APPLICATION
                       |
                Stable Endpoint
                       |
                       v
               +---------------+
               |    HAProxy    |
               |               |
               | :5000 PRIMARY |
               | :5001 REPLICA |
               +-------+-------+
                       |
         +-------------+-------------+
         |             |             |
         v             v             v

       pg01          pg02          pg03
     PostgreSQL    PostgreSQL    PostgreSQL
      Patroni       Patroni       Patroni
      REPLICA       PRIMARY       REPLICA

        |             |             |
        +-------------+-------------+
                      |
                etcd cluster
```

Each layer now has a clear job.

| Component   | Responsibility           |
| ----------- | ------------------------ |
| PostgreSQL  | Database and replication |
| Patroni     | PostgreSQL HA management |
| etcd        | Distributed coordination |
| HAProxy     | Connection routing       |
| Application | Reconnect/retry behavior |

---

# 20. One new problem

We've just created another potential single point of failure:

```text
ONE HAProxy server
```

If:

```text
haproxy01
    X
  DOWN
```

then:

```text
PostgreSQL may be healthy
Patroni may be healthy
etcd may be healthy

BUT

Application can't reach the database
```

So a real production architecture normally shouldn't rely on only one HAProxy server.

You might see:

```text
HAProxy01
HAProxy02
    |
Keepalived / VIP
```

or an infrastructure/cloud load balancer.

For example:

```text
                Application
                     |
                     v
               Virtual IP
             192.168.10.25
                  /   \
                 /     \
                v       v

           HAProxy01   HAProxy02
                \       /
                 \     /
                  v   v

             Patroni cluster
```

We'll cover this later.

---

# 21. Production verification checklist

After a Patroni failover, check:

```text
1. patronictl list
2. New primary role
3. Replication health
4. HAProxy health status
5. Connection through HAProxy
6. Write test through primary endpoint
7. Application reconnect behavior
8. Old primary rejoin
```

For example:

```bash
patronictl -c /etc/patroni/patroni.yml list
```

then through HAProxy:

```bash
psql -h 192.168.10.20 -p 5000 -U postgres -d training
```

then:

```sql
SELECT
    inet_server_addr(),
    pg_is_in_recovery();
```

That single query is excellent for validating routing.

---

## Practice exercise

Run this several times:

```text
1. pg01 PRIMARY
2. Connect through HAProxy :5000
3. Confirm connection lands on pg01
4. Insert data
5. Stop Patroni on pg01
6. Watch Patroni promote another replica
7. Create a new connection through HAProxy
8. Confirm it lands on the new primary
9. Insert more data
10. Bring pg01 back
11. Confirm pg01 rejoins as replica
12. Run patronictl list
```

If you can explain why **Patroni performs the failover while HAProxy performs the routing**, you've understood one of the most important architectural boundaries in PostgreSQL HA.

## Next — Lesson 9: Production-Grade Architecture

Next we should move beyond the basic lab and look at what a real environment needs: **two HAProxy nodes + Keepalived/VIP, proper etcd placement, TLS/security, backups, PgBouncer, monitoring, WAL/archive considerations, failure domains, anti-affinity, and what should or should not be colocated on the same servers.**
