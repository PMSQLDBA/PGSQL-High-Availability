# Lesson 4 — Build the Actual Patroni Lab

Now we move into hands-on work. Since you're starting from zero, we'll build this in stages rather than dropping a huge configuration on you.

For this training lab, we'll use **3 Ubuntu Linux VMs**, PostgreSQL, Patroni, and a 3-member etcd cluster. Current Patroni documentation is at release **4.1.5**, and it supports installation with etcd v3 dependencies. ([Patroni][1])

> **Important:** This is a learning environment. We'll initially use simple unencrypted internal connections. A real production environment should add TLS, authentication, firewall rules, backups, monitoring, and proper secret management.

## Step 1 — Create three VMs

You can use VMware, VirtualBox, cloud VMs, or anything similar.

I'd give each VM roughly:

```text
2 CPU
4 GB RAM
20–30 GB disk
Ubuntu 24.04 LTS
```

Our example network will be:

```text
pg01    192.168.10.11
pg02    192.168.10.12
pg03    192.168.10.13
```

Each machine will initially contain:

```text
+-----------------------+
| pg01                  |
|                       |
| PostgreSQL            |
| Patroni               |
| etcd                  |
+-----------------------+

+-----------------------+
| pg02                  |
|                       |
| PostgreSQL            |
| Patroni               |
| etcd                  |
+-----------------------+

+-----------------------+
| pg03                  |
|                       |
| PostgreSQL            |
| Patroni               |
| etcd                  |
+-----------------------+
```

For learning, colocating them makes the lab easier. Patroni itself does **not** require PostgreSQL nodes and DCS nodes to be the same machines; those layers are decoupled. ([Patroni][2])

---

## Step 2 — Configure hostnames

On `pg01`:

```bash
sudo hostnamectl set-hostname pg01
```

On `pg02`:

```bash
sudo hostnamectl set-hostname pg02
```

On `pg03`:

```bash
sudo hostnamectl set-hostname pg03
```

Then, on **all three servers**, edit:

```bash
sudo nano /etc/hosts
```

Add:

```text
192.168.10.11    pg01
192.168.10.12    pg02
192.168.10.13    pg03
```

Test from `pg01`:

```bash
ping -c 2 pg02
ping -c 2 pg03
```

Do the equivalent testing from the other nodes.

We want:

```text
pg01 <----> pg02
  ^           ^
  |           |
  +---> pg03 <+
```

Don't continue until all three servers can communicate reliably.

---

# Step 3 — Understand our ports

You'll encounter these ports:

|   Port | Service    | Purpose                 |
| -----: | ---------- | ----------------------- |
| `5432` | PostgreSQL | Database connections    |
| `8008` | Patroni    | REST API                |
| `2379` | etcd       | Client communication    |
| `2380` | etcd       | etcd peer communication |

Think:

```text
5432 = Database
8008 = Patroni API
2379 = talk TO etcd
2380 = etcd talks to etcd
```

That distinction becomes useful when troubleshooting.

---

# Step 4 — Update the OS

Run on **all three servers**:

```bash
sudo apt update
sudo apt upgrade -y
```

Then install some basic utilities:

```bash
sudo apt install -y curl wget vim python3 python3-pip python3-venv
```

---

# Step 5 — Install PostgreSQL

For our first lab, keeping installation straightforward matters more than choosing the newest PostgreSQL release.

Install PostgreSQL packages on all three nodes:

```bash
sudo apt install -y postgresql postgresql-contrib
```

Check:

```bash
psql --version
```

And:

```bash
pg_config --version
```

You'll get output showing the PostgreSQL version installed by your repository.

### One important point

Ubuntu's PostgreSQL package may automatically create/start a PostgreSQL cluster.

But eventually we want:

```text
Patroni
   |
   v
PostgreSQL
```

rather than independently managing PostgreSQL.

So **don't start manually configuring replication yet**.

Patroni will bootstrap and manage our PostgreSQL HA cluster.

---

# Step 6 — Install etcd

Current etcd documentation recommends using current prebuilt binaries on Linux; it specifically warns that Linux distribution packages can be significantly outdated. ([etcd][3])

[Official etcd installation guide](https://etcd.io/docs/v3.7/install/?utm_source=chatgpt.com)

For the lab, use the current stable etcd release from the official installation instructions rather than blindly installing whatever old version happens to be in an Ubuntu repository.

After installation, verify:

```bash
etcd --version
```

and:

```bash
etcdctl version
```

You should have both:

```text
etcd
etcdctl
```

`etcd` is the server.

`etcdctl` is the command-line client we use to interact with etcd.

---

# Step 7 — Configure our etcd cluster

This is where Lesson 3 becomes real.

We'll build:

```text
             ETCD CLUSTER

     etcd1       etcd2       etcd3
       |           |           |
     pg01        pg02        pg03
       \           |           /
        +----------+----------+
                   |
                 quorum
                  2/3
```

Conceptually, `pg01` will start etcd with settings equivalent to:

```text
name = etcd1

client:
192.168.10.11:2379

peer:
192.168.10.11:2380
```

`pg02`:

```text
name = etcd2

client:
192.168.10.12:2379

peer:
192.168.10.12:2380
```

`pg03`:

```text
name = etcd3

client:
192.168.10.13:2379

peer:
192.168.10.13:2380
```

And all three need the same initial cluster membership:

```text
etcd1=http://192.168.10.11:2380
etcd2=http://192.168.10.12:2380
etcd3=http://192.168.10.13:2380
```

The exact service/config files depend on how you installed etcd, so don't blindly paste an old `/etc/default/etcd` tutorial here. We'll use the configuration appropriate for the current etcd binary/service installation.

---

# Step 8 — Verify etcd BEFORE installing Patroni

This is extremely important.

Don't configure Patroni until etcd is healthy.

From one node, eventually we'll run something like:

```bash
ETCDCTL_API=3 etcdctl \
  --endpoints=http://192.168.10.11:2379,http://192.168.10.12:2379,http://192.168.10.13:2379 \
  endpoint health
```

We want all three endpoints reporting healthy.

Then:

```bash
ETCDCTL_API=3 etcdctl \
  --endpoints=http://192.168.10.11:2379,http://192.168.10.12:2379,http://192.168.10.13:2379 \
  endpoint status --write-out=table
```

This gives you a view of the etcd members.

Current etcd documentation also uses `etcdctl endpoint health` as an installation/health check. ([etcd][3])

---

# Step 9 — Perform your first etcd test

Remember Lesson 3?

etcd stores distributed key/value information.

Let's prove it.

On `pg01`:

```bash
etcdctl \
  --endpoints=http://192.168.10.11:2379 \
  put training "hello-patroni"
```

Then from `pg02`:

```bash
etcdctl \
  --endpoints=http://192.168.10.12:2379 \
  get training
```

You should get something like:

```text
training
hello-patroni
```

That's a small but important test.

You wrote through:

```text
etcd1
```

and read through:

```text
etcd2
```

because they belong to the same distributed etcd cluster.

The official etcd quickstart uses the same `put`/`get` pattern to verify the cluster is working. ([etcd][4])

---

# Step 10 — Test quorum

Now we're going to intentionally break something.

Suppose:

```text
etcd1 = UP
etcd2 = UP
etcd3 = UP

3/3 healthy
```

Stop etcd on `pg03`.

Then check health again.

Conceptually:

```text
etcd1       etcd2       etcd3

  UP <-----> UP           DOWN

       2 / 3

       QUORUM
```

Your etcd cluster should still be capable of consensus operations.

That's Lesson 3 becoming real.

Start `pg03` again and make sure it rejoins before continuing.

---

# Step 11 — Install Patroni

Patroni's current documentation supports pip installation with PostgreSQL and etcd dependencies. For example, the documented dependency pattern includes `psycopg3` and `etcd3`. ([Patroni][1])

A clean training approach is to create a Python virtual environment on each node:

```bash
sudo mkdir -p /opt/patroni
sudo python3 -m venv /opt/patroni/venv
```

Then:

```bash
sudo /opt/patroni/venv/bin/pip install --upgrade pip
```

Install Patroni with PostgreSQL and etcd3 support:

```bash
sudo /opt/patroni/venv/bin/pip install "patroni[psycopg3,etcd3]"
```

Verify:

```bash
/opt/patroni/venv/bin/patroni --version
```

And:

```bash
/opt/patroni/venv/bin/patronictl version
```

At this point you have two important commands:

```text
patroni
```

Runs the Patroni service.

And:

```text
patronictl
```

Administers/inspects the Patroni cluster.

---

# Step 12 — Where we are now

At this point our environment should look like:

```text
                 Patroni Lab

       pg01          pg02          pg03
        |             |             |
   PostgreSQL    PostgreSQL    PostgreSQL
        |             |             |
     Patroni       Patroni       Patroni
        |             |             |
        v             v             v

      etcd1 <------> etcd2 <------> etcd3

        <-------- QUORUM -------->
```

But we **do not have a Patroni PostgreSQL cluster yet**.

That's intentional.

We first built the infrastructure Patroni needs.

---

# Step 13 — Understand what comes next

The next major file is:

```text
patroni.yml
```

This is where everything we've learned starts connecting.

It will contain sections conceptually like:

```yaml
scope:
name:

restapi:

etcd3:

bootstrap:

postgresql:
```

For example:

```text
scope
   |
   +--> Which Patroni cluster?


name
   |
   +--> Which member am I?


restapi
   |
   +--> Patroni API


etcd3
   |
   +--> Where is our DCS?


bootstrap
   |
   +--> How should the first PostgreSQL
        cluster be initialized?


postgresql
   |
   +--> Where is PostgreSQL?
        Which port?
        Which data directory?
        Authentication?
```

This configuration is the heart of Patroni.

---

## Your checkpoint

Before moving on, verify these on **all three nodes**:

```bash
hostname
psql --version
etcd --version
etcdctl version
/opt/patroni/venv/bin/patroni --version
/opt/patroni/venv/bin/patronictl version
```

And verify etcd:

```bash
etcdctl \
  --endpoints=http://192.168.10.11:2379,http://192.168.10.12:2379,http://192.168.10.13:2379 \
  endpoint health
```

Don't worry if you don't have three VMs available right now—you can still follow the concepts and build them later.

### Lesson 5 — `patroni.yml` and Bootstrap

This is the fun part. We'll create the **actual configuration for `pg01`, `pg02`, and `pg03`**, explain every important parameter, bootstrap the first PostgreSQL node, watch the other two automatically become replicas, and finally run:

```bash
patronictl list
```

to see our first working Patroni cluster.

If you want to continue now, just say **Next**.

[1]: https://patroni.readthedocs.io/_/downloads/en/latest/pdf/?utm_source=chatgpt.com "Patroni Documentation"
[2]: https://patroni.readthedocs.io/_/downloads/en/rel_3_1/pdf/?utm_source=chatgpt.com "Patroni Documentation"
[3]: https://etcd.io/docs/v3.6/install/?utm_source=chatgpt.com "Install | etcd"
[4]: https://etcd.io/docs/v3.8/quickstart/?utm_source=chatgpt.com "Quickstart | etcd"
