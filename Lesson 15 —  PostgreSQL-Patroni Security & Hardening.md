# Lesson 15 — PostgreSQL / Patroni Security & Hardening

Now we secure the HA stack we built.

A Patroni environment has several attack surfaces:

```text
Application
    |
    v
HAProxy / PgBouncer
    |
    v
PostgreSQL :5432
    |
    v
Patroni REST :8008
    |
    v
etcd / DCS :2379 / :2380
    |
    v
Linux / Storage / Backups
```

The main goal is:

> Only the right systems and users should be able to connect, authenticate, read data, or change cluster state.

---

## 1. Start with network segmentation

Do not expose every port everywhere.

A better design is:

```text
Application subnet
       |
       | PostgreSQL only
       v
HAProxy / PgBouncer
       |
       v
PostgreSQL nodes


Management subnet
       |
       +----> Patroni REST
       +----> SSH
       +----> monitoring


Patroni nodes
       |
       +----> etcd client port


etcd members
       |
       +----> etcd peer communication
```

For our stack:

| Port | Service      | Who should normally reach it?               |
| ---: | ------------ | ------------------------------------------- |
| 5432 | PostgreSQL   | HAProxy/PgBouncer/application as required   |
| 8008 | Patroni REST | Patroni, HAProxy health checks, DBA tooling |
| 2379 | etcd client  | Patroni and approved management clients     |
| 2380 | etcd peer    | etcd members only                           |
|   22 | SSH          | Management network/bastion only             |

Avoid:

```text
0.0.0.0/0 -> 2379
```

or:

```text
Internet -> Patroni REST API
```

---

# 2. PostgreSQL authentication — use SCRAM

For password authentication, use:

```text
scram-sha-256
```

rather than legacy MD5.

PostgreSQL 18 explicitly describes SCRAM-SHA-256 as the strongest currently supported built-in password method and notes that MD5-encrypted passwords are deprecated and will be removed in a future release. ([PostgreSQL][1])

For example:

```conf
password_encryption = 'scram-sha-256'
```

Then:

```sql
ALTER ROLE app_user
PASSWORD 'a-strong-secret';
```

The stored password verifier will use SCRAM.

---

# 3. Avoid `trust`

A dangerous `pg_hba.conf` rule is:

```text
host all all 0.0.0.0/0 trust
```

`trust` effectively means:

> If you can connect, PostgreSQL trusts the username you claim.

PostgreSQL documentation confirms that `trust` allows a connecting client to log in without password verification. ([PostgreSQL][2])

That's almost never appropriate for production remote access.

Use authentication such as:

```text
scram-sha-256
```

and narrow network ranges.

---

# 4. Harden `pg_hba.conf`

Our lab used:

```text
host all all 192.168.10.0/24 scram-sha-256
```

That's intentionally broad.

Production should be closer to:

```text
# Replication
hostssl replication replicator 10.20.10.11/32 scram-sha-256
hostssl replication replicator 10.20.10.12/32 scram-sha-256
hostssl replication replicator 10.20.10.13/32 scram-sha-256

# Application
hostssl appdb app_user 10.30.20.0/24 scram-sha-256

# DBA
hostssl all dba_admin 10.40.5.0/24 scram-sha-256
```

Notice the differences:

```text
specific database
specific user
specific network
TLS required
```

rather than:

```text
all databases
all users
entire corporate network
```

PostgreSQL's `hostssl` records match only encrypted TLS connections. ([PostgreSQL][2])

---

# 5. Enable PostgreSQL TLS

Without TLS:

```text
Application
     |
     | SQL / results
     v
PostgreSQL
```

traffic may traverse the network without encryption.

With TLS:

```text
Application
     |
     | encrypted
     v
PostgreSQL
```

PostgreSQL natively supports TLS for client/server traffic. ([PostgreSQL][3])

Conceptually configure:

```conf
ssl = on
ssl_cert_file = '/path/to/server.crt'
ssl_key_file  = '/path/to/server.key'
ssl_ca_file   = '/path/to/ca.crt'
```

Then use:

```text
hostssl
```

in `pg_hba.conf`.

---

# 6. Clients should verify PostgreSQL certificates

Simply enabling TLS encryption isn't the strongest configuration.

A client can use:

```text
sslmode=require
```

which requires encryption.

Stronger identity verification is:

```text
sslmode=verify-full
```

Conceptually:

```text
Client
   |
   | Does certificate come from trusted CA?
   |
   | Does hostname match certificate?
   |
   v
PostgreSQL
```

This helps prevent connecting to an impersonated server.

PostgreSQL also supports SCRAM channel binding with TLS, which strengthens protection against man-in-the-middle attacks for compatible clients. ([PostgreSQL][4])

---

# 7. Client certificates

For stronger environments you can require client certificates.

For example:

```text
hostssl appdb app_user 10.30.20.0/24 scram-sha-256 clientcert=verify-full
```

Then the client must provide:

```text
trusted certificate
+
valid PostgreSQL credentials
```

PostgreSQL supports certificate verification in `hostssl` rules using `clientcert=verify-ca` or `verify-full`. ([PostgreSQL][2])

This is useful for highly controlled service-to-service communication.

---

# 8. PostgreSQL role design

Don't let applications connect as:

```text
postgres
```

Your application should have its own role:

```sql
CREATE ROLE app_user
LOGIN
PASSWORD '...';
```

Then grant only what is required.

Think:

```text
postgres
   |
   +--> DBA / emergency administration


app_user
   |
   +--> application-required permissions


replicator
   |
   +--> replication only


monitor_user
   |
   +--> monitoring only


backup_user
   |
   +--> backup permissions
```

This is **least privilege**.

---

# 9. Avoid application superusers

Bad:

```sql
ALTER ROLE app_user SUPERUSER;
```

If the application account is compromised, an attacker now has database-administrator capability.

Applications usually need things such as:

```text
SELECT
INSERT
UPDATE
DELETE
EXECUTE
```

on specific schemas/objects.

They normally do not need:

```text
SUPERUSER
CREATEROLE
REPLICATION
BYPASSRLS
```

---

# 10. Protect the Patroni REST API

Remember:

```text
Patroni REST API = :8008
```

This API isn't just informational.

It supports functions involved in:

```text
failover
switchover
restart
reload
reinitialize
```

Patroni specifically identifies the **DCS and REST API** as interfaces requiring protection. ([Patroni][5])

Therefore:

```text
Application network
      |
      X
      |
Patroni REST
```

Access should be limited.

---

# 11. REST API authentication

Patroni supports authentication for state-changing REST operations.

Conceptually:

```yaml
restapi:
  listen: 0.0.0.0:8008

  authentication:
    username: patroni_admin
    password: <secret>
```

Patroni documentation explains that unsafe `PUT`, `POST`, `PATCH`, and `DELETE` operations can be protected with REST API authentication. ([Patroni][5])

Do not put an easy password like:

```text
admin/admin
```

into production.

---

# 12. Enable TLS for Patroni REST

Without TLS:

```text
patronictl
     |
     | HTTP
     v
Patroni
```

Better:

```text
patronictl
     |
     | HTTPS
     v
Patroni
```

Patroni supports TLS for its REST API and can also use client-certificate verification. ([Patroni][5])

Conceptually:

```yaml
restapi:
  certfile: /etc/patroni/tls/server.crt
  keyfile: /etc/patroni/tls/server.key
  cafile: /etc/patroni/tls/ca.crt
```

Exact options should always be checked against the Patroni version you're running.

---

# 13. Patroni mutual TLS

A stronger configuration is:

```text
Patroni Node
    |
client certificate
    |
    v
Patroni REST API
```

Patroni supports REST client-certificate verification modes such as required/optional/none. ([Patroni][5])

This can protect even more of the API surface than basic authentication alone.

---

# 14. HAProxy health checks with secured Patroni

This requires planning.

Previously HAProxy simply did:

```text
GET http://pg01:8008/primary
```

If REST TLS is enabled:

```text
HAProxy
    |
    | HTTPS
    v
Patroni
```

HAProxy may also need:

```text
CA trust
client certificate if required
hostname/certificate validation
```

So don't enable strict Patroni TLS at 3 PM in production and discover that HAProxy can no longer perform health checks.

Test the complete path first.

---

# 15. Secure etcd aggressively

etcd is particularly sensitive because it coordinates Patroni.

Remember:

```text
Patroni
   |
   v
etcd
```

If an unauthorized actor can modify the Patroni DCS, they may be able to interfere with cluster configuration or HA state.

Patroni's security guidance recommends TLS for DCS communication. ([Patroni][5])

Current etcd documentation supports TLS and client certificates for both client/server and peer communications. ([etcd][6])

---

# 16. etcd client TLS

Instead of:

```text
Patroni
   |
HTTP :2379
   |
etcd
```

use:

```text
Patroni
   |
HTTPS / mTLS
   |
etcd
```

Conceptually:

```text
etcd client:
  CA
  client certificate
  private key
```

Each Patroni node can authenticate itself when connecting to etcd.

---

# 17. Secure etcd peer communication

Remember:

```text
2379 = clients talk to etcd

2380 = etcd members talk to each other
```

Protect both.

Architecture:

```text
            TLS / mTLS

etcd01 <--------------> etcd02
   ^                       ^
   |                       |
   +---------> etcd03 <----+
```

Current etcd documentation supports certificate authentication for both client-to-server and peer-to-peer communication. ([etcd][6])

---

# 18. Don't assume etcd auth is automatically enabled

This is important.

etcd does not enable all authentication/RBAC security features by default. Current etcd documentation explicitly warns that an unsecured cluster can expose its data to clients. ([etcd][6])

So:

```text
etcd installed
```

does **not** automatically mean:

```text
etcd secured
```

Validate:

```text
TLS
authentication
RBAC
network firewall
certificate validation
```

explicitly.

---

# 19. Don't expose etcd publicly

Bad architecture:

```text
Internet
   |
   v
etcd :2379
```

Better:

```text
Private DB / management network
        |
        v
      etcd
```

Ideally only:

```text
Patroni nodes
approved DBA tooling
etcd cluster members
monitoring
```

can communicate with etcd.

---

# 20. Protect `patroni.yml`

Your Patroni configuration may contain:

```text
PostgreSQL superuser password
replication password
REST API credentials
DCS credentials
TLS paths
```

So:

```bash
ls -l /etc/patroni/patroni.yml
```

should not show something like:

```text
-rw-r--r--  root root
```

where every local user can read secrets.

A stronger simple model:

```bash
sudo chown postgres:postgres /etc/patroni/patroni.yml
sudo chmod 600 /etc/patroni/patroni.yml
```

depending on the account running Patroni.

---

# 21. Use secret management

For larger environments, storing every secret directly inside YAML is difficult to manage.

Consider technologies such as:

```text
HashiCorp Vault
AWS Secrets Manager
Azure Key Vault
Google Secret Manager
Kubernetes Secrets
Ansible Vault
```

depending on infrastructure.

The goal is:

```text
No credentials in Git
No passwords in tickets
No passwords in documentation
Controlled rotation
Auditable access
```

---

# 22. Linux service account

Run Patroni and PostgreSQL using dedicated non-root accounts.

For example:

```text
postgres
```

should own:

```text
PGDATA
PostgreSQL configuration
required Patroni files
```

but it should not have unrestricted root access.

And Patroni itself should normally not need:

```text
root login
```

for routine operation.

---

# 23. Protect `$PGDATA`

Check:

```bash
ls -ld /var/lib/patroni/data
```

A common secure permission is:

```text
700
```

Conceptually:

```text
postgres = full access
everyone else = none
```

The data directory contains:

```text
database files
WAL
configuration
internal metadata
```

It should not be readable by arbitrary OS users.

---

# 24. Protect private keys

TLS private keys deserve stricter handling.

Example:

```text
/etc/postgresql/tls/server.key
/etc/patroni/tls/server.key
/etc/etcd/tls/peer.key
```

Permissions should restrict access to the owning service.

Never:

```text
chmod 777 server.key
```

Private keys should generally be accessible only to the service account requiring them.

---

# 25. SSH hardening

Database nodes should not be general-purpose login servers.

Typical controls include:

```text
Bastion/jump host
SSH key authentication
MFA where infrastructure supports it
Restricted admin groups
No shared admin account
Central logging
Disable unused accounts
```

Avoid:

```text
DBAs all share password for root
```

because there is no meaningful individual accountability.

---

# 26. Patch the operating system

Security isn't only PostgreSQL configuration.

Keep current:

```text
Linux packages
OpenSSL
Python
Patroni
PostgreSQL
etcd
HAProxy
PgBouncer
backup software
```

But patching an HA environment needs coordination.

Use Lesson 12's workflow:

```text
Patch replica
     |
verify
     |
patch another replica
     |
verify
     |
switchover
     |
patch old primary
```

rather than rebooting everything together.

---

# 27. Reduce unnecessary software

Database nodes should ideally not contain:

```text
random developer tools
web servers
desktop environments
unnecessary agents
unused network services
```

Each additional service increases:

```text
attack surface
patching requirements
resource contention
operational complexity
```

---

# 28. Firewall design

A simplified firewall policy might be:

```text
APP SUBNET
    |
    +----> HAProxy :5000


HAPROXY
    |
    +----> PostgreSQL :5432
    +----> Patroni :8008


PATRONI NODES
    |
    +----> etcd :2379


ETCD MEMBERS
    |
    +----> each other :2380


DBA NETWORK
    |
    +----> SSH
    +----> management APIs
```

Everything else:

```text
DENY
```

That's much safer than:

```text
allow entire internal network everywhere
```

---

# 29. Secure HAProxy

HAProxy may expose:

```text
PostgreSQL proxy port
Statistics page
Management socket
```

Don't expose statistics anonymously.

Bad:

```text
http://haproxy:7000/stats
```

available to everyone.

Restrict it by:

```text
management network
authentication
TLS
firewall
```

depending on your environment.

---

# 30. PgBouncer security

If PgBouncer is introduced, secure it as another authentication boundary.

Protect:

```text
auth_file
auth_query
TLS keys
administrative database
PgBouncer console
```

Do not allow unrestricted access to administrative commands.

Think:

```text
Application
     |
     v
PgBouncer
     |
     v
HAProxy
     |
     v
PostgreSQL
```

Every new layer creates another security boundary.

---

# 31. Protect backups

Backups may be even more attractive to attackers than the running database.

Why?

Because they contain:

```text
entire database history
customer data
credentials/data rows
years of historical information
```

Protect the repository with:

```text
encryption
least privilege
separate credentials
restricted network access
immutable copies where appropriate
audit logging
```

And preferably:

```text
production DB account
        !=
backup deletion account
```

so compromising PostgreSQL does not automatically allow destroying every backup.

---

# 32. Encryption at rest

TLS protects:

```text
data in transit
```

It doesn't protect disks stolen from the server.

Depending on infrastructure, use:

```text
encrypted VM disks
LUKS
encrypted SAN
cloud disk encryption
encrypted backup repository
```

PostgreSQL documentation distinguishes network encryption from storage-level encryption; disk encryption protects offline storage but not data once the filesystem is mounted and accessible to the operating system. ([PostgreSQL][7])

---

# 33. Logging and auditing

You want to know:

```text
Who logged in?
Who changed roles?
Who performed failover?
Who changed Patroni configuration?
Who accessed etcd?
Who changed firewall rules?
```

For PostgreSQL consider appropriate logging for:

```text
connections
disconnections
authentication failures
DDL
privileged operations
slow statements
```

but balance this against volume and sensitive-data exposure.

Do not blindly log:

```text
all SQL including passwords or sensitive values
```

without understanding what will end up in logs.

---

# 34. Credential rotation

You should have a procedure for rotating:

```text
Application password
Replication password
PostgreSQL admin password
Patroni REST credentials
etcd certificates
TLS server certificates
Backup credentials
```

Rotation itself is an HA operation.

For example, changing the replication password improperly could cause:

```text
pg02 disconnects
pg03 disconnects
        |
        v
no healthy replicas
```

So test rotations and stage changes carefully.

---

# 35. Certificate expiry monitoring

This gets missed surprisingly often.

Suppose your etcd certificates expire at midnight.

You could suddenly get:

```text
Patroni
   X
etcd
```

even though PostgreSQL itself is healthy.

Monitor:

```text
PostgreSQL certificate expiry
Patroni REST certificate expiry
etcd client certificate expiry
etcd peer certificate expiry
HAProxy certificate expiry
```

Alert well before expiration.

For example:

```text
Warning  = 30 days
Critical = 7 days
```

as an illustrative policy.

---

# 36. A hardened conceptual architecture

Your production environment might look like:

```text
                 APPLICATION NETWORK
                         |
                         | TLS
                         v
                 +---------------+
                 | HAProxy pair  |
                 +-------+-------+
                         |
                 TLS PostgreSQL
                         |
          +--------------+--------------+
          |              |              |
          v              v              v
        pg01           pg02           pg03
      PostgreSQL     PostgreSQL     PostgreSQL
       Patroni        Patroni        Patroni

          \              |              /
           \        TLS / mTLS         /
            +------------+------------+
                         |
                    etcd cluster
                 etcd01/02/03
                  TLS + Auth


                MANAGEMENT NETWORK
                         |
            DBA / monitoring / backups


                 BACKUP NETWORK
                         |
                         v
              Secured repository
                  encrypted
                  offsite copy
```

---

# 37. Production hardening checklist

Before production, I would review at least:

| Area                | Requirement                     |
| ------------------- | ------------------------------- |
| PostgreSQL auth     | SCRAM                           |
| Application role    | Not superuser                   |
| Replication user    | Dedicated role                  |
| `pg_hba.conf`       | Least privilege                 |
| PostgreSQL network  | TLS                             |
| Patroni REST        | Restricted network              |
| Patroni REST        | Authentication                  |
| Patroni REST        | TLS where required              |
| etcd client traffic | TLS/mTLS                        |
| etcd peer traffic   | TLS/mTLS                        |
| etcd network        | Private/restricted              |
| `patroni.yml`       | Restricted permissions          |
| TLS keys            | Restricted permissions          |
| Secrets             | Not in source control           |
| Linux accounts      | Least privilege                 |
| SSH                 | Restricted/admin-controlled     |
| HAProxy             | Protected management interfaces |
| Backups             | Encrypted/restricted            |
| Certificates        | Expiry monitoring               |
| OS/software         | Regular patching                |
| Audit/logging       | Enabled appropriately           |

---

# 38. Security troubleshooting example

Suppose after security hardening:

```text
pg01 PRIMARY
pg02 replica disconnected
pg03 replica disconnected
```

Logs:

```text
certificate verify failed
```

Don't immediately disable TLS.

Troubleshoot:

```text
1. Is CA certificate correct?

2. Has certificate expired?

3. Does certificate SAN contain correct hostname/IP?

4. Is Patroni using correct certificate path?

5. Can pg02 reach pg01?

6. Is pg_hba requiring hostssl?

7. Does PostgreSQL trust the client certificate?

8. Do filesystem permissions allow Patroni/Postgres to read keys?
```

A common bad response is:

```text
TLS broke replication
so disable TLS
```

The correct response is:

```text
TLS exposed a certificate/configuration problem
fix the certificate/config
```

---

# 39. Another security incident

Suppose:

```text
HAProxy reports every primary backend DOWN
```

after you enable Patroni REST TLS.

Patroni itself is healthy.

Try:

```bash
curl https://pg01:8008/primary
```

If you receive:

```text
certificate verify failed
```

HAProxy may have the same trust problem.

Potential fix:

```text
Install correct CA
Configure certificate verification
Provide client certificate if Patroni requires one
```

Not:

```text
turn off security everywhere
```

---

# 40. The security mental model

Think about each connection:

```text
WHO
 |
 | connects to
 v
WHAT
 |
 | using
 v
WHICH IDENTITY
 |
 | over
 v
ENCRYPTED CHANNEL?
 |
 | allowed by
 v
WHICH NETWORK RULE?
```

For example:

```text
pg02 Patroni
     |
     v
etcd01
     |
client certificate
     |
TLS
     |
2379
     |
DB management subnet only
```

If you can answer those five questions for every connection in the architecture, your security design becomes much easier to review.

## What to remember from Lesson 15

The highest-value points are:

**Use SCRAM instead of legacy MD5 authentication.** PostgreSQL now explicitly deprecates MD5-encrypted passwords. ([PostgreSQL][1])

**Require TLS for sensitive PostgreSQL network connections and use narrow `pg_hba.conf` rules.** ([PostgreSQL][2])

**Protect both Patroni's REST API and its DCS.** Patroni identifies these as its two major control interfaces and recommends TLS for DCS traffic. ([Patroni][5])

**Secure etcd with private networking, TLS/mTLS, authentication and restricted peer/client access.** etcd does not automatically enable all of these protections. ([etcd][6])

**Protect secrets, TLS private keys and backups as carefully as the database itself.**

### Next — Lesson 16: PostgreSQL Major Upgrade with Patroni

The next useful production topic is **how to upgrade a Patroni-managed PostgreSQL cluster from one major version to another**—for example PostgreSQL 17 → 18—including why a normal rolling restart doesn't work, `pg_upgrade` vs logical replication, blue/green approaches, downtime considerations, rollback strategy and how Patroni fits into the migration.

[1]: https://www.postgresql.org/docs/18/auth-password.html "PostgreSQL: Documentation: 18: 20.5. Password Authentication"
[2]: https://www.postgresql.org/docs/18/auth-pg-hba-conf.htmlm "PostgreSQL: Documentation: 18: 20.1. The pg_hba.conf File"
[3]: https://www.postgresql.org/docs/current/ssl-tcp.htmlm "PostgreSQL: Documentation: 18: 18.9. Secure TCP/IP Connections with SSL"
[4]: https://www.postgresql.org/docs/18/sasl-authentication.html "PostgreSQL: Documentation: 18: 54.3. SASL Authentication"
[5]: https://patroni.readthedocs.io/en/rel_3_3/security.html "Security Considerations — Patroni 3.3.11 documentation"
[6]: https://etcd.io/docs/v3.6/op-guide/security/?utm_source=chatgpt.com "Transport security model | etcd"
[7]: https://www.postgresql.org/docs/18/encryption-options.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 18: 18.8. Encryption Options"
