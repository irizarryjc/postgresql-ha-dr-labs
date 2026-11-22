# PostgreSQL HA/DR Companion Lab Configurations

Companion configuration templates for *PostgreSQL High Availability and Disaster Recovery by Objective* by Juan C. Irizarry, OCP.

Repository URL: `https://github.com/irizarryjc/postgresql-ha-dr-labs`

All templates live in the `lab-configs/` folder.

> **Lab safety rule:** Use these files only in an isolated lab network. Never inject crashes, network partitions, or destructive SQL against production or shared staging databases. Replace every `<placeholder>` value with lab-only hostnames, IP addresses, and credentials before starting services.

## Baseline and Compatibility

- **Baseline engine:** PostgreSQL 17 with `--data-checksums` enabled during `initdb` (data checksums become the `initdb` default in PostgreSQL 18).
- **Patroni support:** Official Patroni documentation lists support for PostgreSQL 9.3 through 18. Confirm current Patroni release notes before testing against PostgreSQL 19.
- **Authentication:** All templates use `scram-sha-256` (`md5` password authentication is deprecated starting in PostgreSQL 18). Never commit real passwords to version control.

## Client Routing Topology

These templates follow the production-safe client routing path taught in Chapters 2 and 8:

```text
Application -> keepalived VIP (<vip_address>)
            -> PgBouncer (:6432, pooling, RECONNECT on cutover)
            -> HAProxy (127.0.0.1:5000 primary / 127.0.0.1:5001 replica)
               probing Patroni REST (:8008/primary and :8008/replica)
            -> PostgreSQL Primary / Standby (<primary_host>, <standby_host>, <replica2_host>)
```

An alternative topology places a per-node PgBouncer co-located on each database host behind HAProxy; Chapter 8 compares both layouts.

## File Inventory

All paths below are relative to `lab-configs/`.

| File | Purpose | Chapters & Labs |
|---|---|---|
| `docker-compose.yml` | Lightweight local container topology (3-node etcd + 3 PG 17 containers + repository placeholder) | Ch. 2, Ch. 5–9 (quick start) |
| `patroni-node1.yml` | Patroni configuration template for `<primary_host>` (`pg-node1`) | Ch. 6 (L3–L6), Ch. 10 (L9–L10) |
| `patroni-node2.yml` | Patroni configuration template for `<standby_host>` (`pg-node2`) | Ch. 6 (L3–L6), Ch. 10 (L9–L10) |
| `patroni-node3.yml` | Patroni configuration template for `<replica2_host>` (`pg-node3`), the optional third local member | Ch. 6 (optional), Ch. 8 (L7) |
| `haproxy.cfg` | HAProxy 2.8+ TCP router probing Patroni REST `:8008/primary` and `/replica` | Ch. 2, Ch. 8 (L7), Ch. 10 (L9–L10) |
| `pgbouncer.ini` | PgBouncer transaction-pooling config targeting local HAProxy ports | Ch. 2, Ch. 8 (L7), Ch. 10 (L9–L10) |
| `keepalived.conf` | Unicast VRRP VIP configuration tracking both PgBouncer and HAProxy | Ch. 2, Ch. 8 (L7), Ch. 10 (L9–L10) |
| `pgbackrest.conf` | Dedicated repository host (`<repo_host>`) and database-node stanza templates | Ch. 9 (L8), Ch. 10 (L9–L10) |

## Port Map

| Component | Port | Binding / Role |
|---|---|---|
| PostgreSQL | `5432` | Database engine on `<primary_host>`, `<standby_host>`, `<replica2_host>` |
| Patroni REST API | `8008` | Health probes (`/primary`, `/replica`) and `patronictl` management |
| etcd client / peer | `2379` / `2380` | 3-node DCS (`<dcs1_host>`, `<dcs2_host>`, `<dcs3_host>`) |
| PgBouncer | `6432` | Client-facing pooler bound behind `<vip_address>` |
| HAProxy primary | `5000` | Local TCP backend (`127.0.0.1:5000`) for read-write traffic |
| HAProxy replica | `5001` | Local TCP backend (`127.0.0.1:5001`) for read-only traffic |
| HAProxy stats | `7000` | Local monitoring dashboard (`127.0.0.1:7000/stats`) |

## Quick Start

Clone the repository onto your lab workstation or jump host:

```bash
git clone https://github.com/irizarryjc/postgresql-ha-dr-labs.git
cd postgresql-ha-dr-labs
```

Validate YAML syntax locally before substituting your lab values:

```bash
python3 -c '
import glob, yaml
for p in sorted(glob.glob("lab-configs/*.yml")):
    yaml.safe_load(open(p))
    print("OK:", p)
'
```

## Notes on the Docker Compose Lab

`lab-configs/docker-compose.yml` starts a three-node etcd cluster, three PostgreSQL 17 containers initialized with `--data-checksums`, and a placeholder repository container on a private bridge network. Only `pg-primary` starts with replication settings. The `pg-standby` and `pg-replica2` containers start as independent, empty primaries: they are not standbys until you clone them from `pg-primary` with `pg_basebackup` as shown in Chapter 5. Patroni and pgBackRest are not preinstalled in the container images; use virtual machines for the Patroni, routing, and full-site drills.

## Using `patroni-node3.yml`

`pg-node3` on `<replica2_host>` joins the same Patroni scope as `pg-node1` and `pg-node2` (PostgreSQL on `5432`, Patroni REST on `8008`). It is optional in Chapter 6 and expected by the HAProxy backends and Lab L7 in Chapter 8. For the Chapter 10 remote-site design, reconfigure the Site B node as described in that chapter instead of joining it to the Site A scope.

## Placeholder Reference

Replace all angle-bracket placeholders consistently across files:

- Hosts & network: `<primary_host>`, `<standby_host>`, `<replica2_host>`, `<dcs1_host>`, `<dcs2_host>`, `<dcs3_host>`, `<repo_host>`, `<router1_host>`, `<router2_host>`, `<vip_address>`, `<network_interface>`
- Identifiers: `<cluster_name>`, `<scope_name>`, `<app_name>`, `<slot_name>`, `<stanza_name>`
- Role-separated secrets (never reuse one password across roles; store in `.pgpass`, environment variables, or protected config files with `0600` permissions): `<superuser_password>`, `<repl_password>`, `<rewind_password>`, `<patroni_api_password>`, `<app_password>`, `<pgbouncer_password>`, `<backup_password>`, `<vrrp_auth_password>`

## Disclaimer

PostgreSQL and Postgres are trademarks or registered trademarks of the PostgreSQL Community Association of Canada. This repository and the companion book are not affiliated with or endorsed by the PostgreSQL Community Association of Canada, the PostgreSQL Global Development Group, or the PostgreSQL Project.

## License

Released under the MIT License. See `LICENSE`.
