Below are some solid online options for learning **Patroni with PostgreSQL**, including hands-on HA/failover training.

### Best options

1. **CYBERTEC – PostgreSQL High Availability Architecture with Patroni**
   This is probably the closest match if you want a structured instructor-led course specifically around Patroni. It is offered **online for 3 days, 5 hours/day** and covers PostgreSQL streaming replication, Patroni cluster management, **etcd/quorum**, and automatic failover. ([CYBERTEC PostgreSQL | Services & Support][1])
   [CYBERTEC Online PostgreSQL Training](https://www.cybertec-postgresql.com/en/online-training/)

2. **Percona – Patroni / PostgreSQL HA material**
   Percona has very useful practical material around PostgreSQL HA. Their PostgreSQL training material includes an **Automation / Patroni** section covering installation and configuration, along with replication, HA, backup/recovery, and operations. ([Percona][2])
   [Percona PostgreSQL Training Material](https://www.percona.com/sites/default/files/PostgreSQL_Training_72622.pdf)

They also have a good hands-on Patroni video:

[Setting Up and Using Patroni to Automate Failover in PostgreSQL](https://www.youtube.com/watch?v=dkp-3S3l8tY)

And this one is particularly useful once you understand the basics:

[PostgreSQL HA With Patroni – Failure Scenarios and Cluster Recovery](https://www.youtube.com/watch?v=ALGC1v9wfW4)

3. **Official Patroni documentation – free**
   I strongly recommend using this alongside any course. The current documentation covers installation, configuration, replication modes, DCS, failover behavior, `patronictl`, Kubernetes, multi-datacenter HA, and other operational topics. Patroni currently supports PostgreSQL versions **9.3 through 18**. ([Patroni][3])
   [Official Patroni Documentation](https://patroni.readthedocs.io/en/rel_4_0/)
   [Official Patroni GitHub Repository](https://github.com/patroni/patroni)

4. **Percona HA deployment guide – free and hands-on**
   This is useful for building your own lab. Their current guide walks through a PostgreSQL HA architecture with Patroni, including deployment and testing replication/failover. ([Percona Documentation][4])
   [Percona – High Availability PostgreSQL with Patroni](https://docs.percona.com/postgresql/18/solutions/high-availability.html)

### How we would learn it

If our goal is to become comfortable enough to **support Patroni in production**, we wouldn't focus only on installing it. we'd follow this order:

**PostgreSQL replication → etcd/quorum → Patroni architecture → Patroni YAML → `patronictl` → HAProxy/load balancing → switchover → failover → node rebuild/rejoin → DCS failure → network partition/split-brain scenarios → monitoring → backup/restore → production troubleshooting.**

For example, build a small lab like:

```text
                  Application
                      |
                   HAProxy
                      |
          +-----------+-----------+
          |                       |
     PostgreSQL-1              PostgreSQL-2
       Patroni                   Patroni
      PRIMARY                   REPLICA
          |                       |
          +----------+------------+
                     |
               etcd cluster
              etcd1/etcd2/etcd3
```

Then deliberately break things: 

stop PostgreSQL, stop Patroni, kill the primary server, break replication, stop an etcd member, simulate network problems, perform manual switchovers, and rebuild a failed replica. That's where Patroni starts making sense.

For a **paid structured course**, we'd start with the CYBERTEC Patroni course. 

For **self-paced/free learning**, we'd combine the official Patroni docs + Percona deployment guide + the Percona videos above.

[1]: https://www.cybertec-postgresql.com/en/online-training/ "Online Training"
[2]: https://www.percona.com/sites/default/files/PostgreSQL_Training_72622.pd "PERCONA TRAINING"
[3]: https://patroni.readthedocs.io/en/rel_4_0/ "Patroni: A Template for PostgreSQL HA with ZooKeeper, etcd or Consul — Patroni 4.0.10 documentation"
[4]: https://docs.percona.com/postgresql/18/solutions/high-availability.html "Overview - Percona Distribution for PostgreSQL"


============================================================================

With **no Patroni experience**, we would not jump directly into Patroni configuration. 

Patroni becomes much easier once you understand a few PostgreSQL HA concepts.

### Beginner learning path

We'd spend about **2 weeks**, 1–2 hours per day:

| Days | Focus                | What you should understand                                          |
| ---- | -------------------- | ------------------------------------------------------------------- |
| 1–2  | PostgreSQL basics    | instance, database, roles, `psql`, config files, service management |
| 3–4  | PostgreSQL internals | WAL, checkpoints, `pg_hba.conf`, `postgresql.conf`                  |
| 5–6  | Replication          | primary/standby, streaming replication, replication slots, lag      |
| 7    | HA concepts          | HA, RPO/RTO, quorum, split-brain, switchover vs failover            |
| 8    | etcd                 | DCS, leader election, quorum, why 3 nodes                           |
| 9–10 | Patroni              | architecture, `patroni.yml`, REST API, `patronictl`                 |
| 11   | Build cluster        | 3 PostgreSQL + Patroni nodes                                        |
| 12   | HAProxy              | route applications to the current primary                           |
| 13   | Failure testing      | kill primary, observe automatic failover                            |
| 14   | Operations           | switchover, reinitialize replica, troubleshooting                   |

PostgreSQL's own documentation is the right place to understand replication first. 

PostgreSQL streaming replication uses a **primary and standby** model, which is the foundation Patroni builds on. ([PostgreSQL][1])

[PostgreSQL Replication Documentation](https://www.postgresql.org/docs/current/runtime-config-replication.html)

### Then learn Patroni architecture

Keep this simple mental model:

```text
                     Application
                         |
                      HAProxy
                         |
              "Who is the primary?"
                         |
          +--------------+--------------+
          |              |              |
       NODE 1          NODE 2         NODE 3
      PostgreSQL      PostgreSQL     PostgreSQL
      Patroni         Patroni        Patroni
       Primary         Replica        Replica
          |              |              |
          +--------------+--------------+
                         |
                    etcd cluster
```

**PostgreSQL** stores the data and performs replication.

**Patroni** manages PostgreSQL and decides what should happen when servers fail.

**etcd** provides the distributed coordination/consensus information Patroni needs.

**HAProxy** gives applications a stable endpoint and can route connections to the appropriate PostgreSQL node.

This architecture is also close to Percona's reference architecture. 

Their current material recommends a three-instance PostgreSQL cluster for HA and explains how Patroni, etcd and a connection proxy fit together. ([Percona Documentation][2])

### Best free hands-on course for you

We would actually start with **Percona's step-by-step documentation** rather than paying for a Patroni course immediately.

[Percona PostgreSQL HA with Patroni](https://docs.percona.com/postgresql/18/solutions/high-availability.html)

They provide an initial lab setup with three nodes and walk through building a highly available PostgreSQL environment. ([Percona Documentation][3])

[Percona Patroni Initial Setup](https://docs.percona.com/postgresql/18/solutions/ha-init-setup.html)

There's also a detailed Patroni setup guide covering installation, `patroni.yml`, systemd, starting the cluster and checking Patroni. ([Percona Documentation][4])

[Percona Patroni Setup Guide](https://docs.percona.com/postgresql/16/solutions/ha-patroni.html)

### The most important part: build a lab

Reading Patroni documentation alone won't be enough. 

Build **3 Linux VMs** and practice.

For example:

```text
pg01   192.168.10.11   PostgreSQL + Patroni + etcd
pg02   192.168.10.12   PostgreSQL + Patroni + etcd
pg03   192.168.10.13   PostgreSQL + Patroni + etcd

haproxy 192.168.10.20  HAProxy
```

Once it's working, deliberately break it.

```bash
patronictl list

patronictl switchover

systemctl stop patroni

systemctl start patroni

systemctl status patroni

journalctl -u patroni
```

We want to reach the point where you can explain: **"pg01 died → Patroni detected the failure → etcd coordination allowed a new leader to be selected → pg02 was promoted → HAProxy sent new connections to pg02 → pg01 later returned as a replica."** That's the core Patroni concept. ([Percona Documentation][5])

