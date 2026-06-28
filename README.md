# Distributed HA Wazuh — Ansible

Ansible automation to deploy an **Enterprise Distributed Wazuh SIEM** on
**Rocky Linux 9**: HA dashboards, a tiered (hot/warm/cold) indexer cluster, a
clustered Wazuh server (master + workers), and an HAProxy access layer — with
optional MinIO object storage and an external ML server.

```
Users/Analysts ──> HAProxy (https://siem.local.domain) ──> Dashboard-1 / Dashboard-2
                                                              │
                                  ┌───────────────────────────┘
                                  ▼
                  Wazuh Indexer Cluster (ISM tiers)
                  Hot ×3  ──>  Warm ×1  ──>  Cold ×1  ──> (MinIO S3 snapshots)
                                  ▲                         (External ML server)
                                  │ Filebeat
                  Wazuh Server Cluster: Master + Worker ×2
                                  ▲
                  Data sources: Linux / Windows agents, Syslog
```

## Host inventory

| Role / Tier            | Hostname / FQDN                  | IP address (/16) | Inventory group        |
|------------------------|----------------------------------|------------------|------------------------|
| Certificate Authority  | `wazuh-ca.local.domain`          | `10.0.100.100`   | `wazuh_ca`             |
| HAProxy load balancer  | `haproxy-1.local.domain`         | `10.0.100.10`    | `load_balancer`        |
| Wazuh Dashboard 1      | `dashboard-1.local.domain`       | `10.0.100.20`    | `wazuh_dashboard`      |
| Wazuh Dashboard 2      | `dashboard-2.local.domain`       | `10.0.100.30`    | `wazuh_dashboard`      |
| Indexer — hot 1        | `indexer-hot-1.local.domain`     | `10.0.100.40`    | `indexer_hot`          |
| Indexer — hot 2        | `indexer-hot-2.local.domain`     | `10.0.100.41`    | `indexer_hot`          |
| Indexer — hot 3        | `indexer-hot-3.local.domain`     | `10.0.100.42`    | `indexer_hot`          |
| Indexer — warm 1       | `indexer-warm-1.local.domain`    | `10.0.100.50`    | `indexer_warm`         |
| Indexer — cold 1       | `indexer-cold-1.local.domain`    | `10.0.100.60`    | `indexer_cold`         |
| Wazuh server — master  | `manager-master-1.local.domain`  | `10.0.100.70`    | `wazuh_manager_master` |
| Wazuh server — worker 1| `manager-worker-1.local.domain`  | `10.0.100.71`    | `wazuh_manager_worker` |
| Wazuh server — worker 2| `manager-worker-2.local.domain`  | `10.0.100.72`    | `wazuh_manager_worker` |

> **Network:** `10.0.0.0/16` — every host uses a `/16` prefix, gateway
> `10.0.0.1`, DNS `10.0.0.1`, domain `local.domain`.
> **Public entry point:** HAProxy at `10.0.100.10`, reached as
> **`https://siem.local.domain`** (the dashboard's main URL).
> **Hostnames:** each box is named `<inventory_hostname>.local.domain` and is
> set on the server by the `common` role during the playbook run.

The `indexer_hot` / `indexer_warm` / `indexer_cold` subgroups all roll up into
the single `wazuh_indexer` cluster; each sets `node.attr.temp` so Index State
Management (ISM) can age data hot → warm → cold.

### Optional components (defined but commented out in the inventory)

| Component          | Suggested IP   | Inventory group | Purpose                              |
|--------------------|----------------|-----------------|--------------------------------------|
| MinIO S3 storage   | `10.0.100.80+` | `minio`         | Snapshots / backups / ML artifacts   |
| External ML server | `10.0.100.90`  | `ml_server`     | Anomaly detection / enrichment       |
| Wazuh agents       | `10.0.200.0/24`| `wazuh_agents`  | Endpoint enrollment                  |

## Two inventories: SSL and no-SSL

The project ships two inventories so you can stand up a quick functional test
without TLS, then deploy the real, secured cluster:

| Inventory | TLS | CA host | Use |
|---|---|---|---|
| [`inventories/production`](inventories/production/) | **on** (`enable_ssl: true`) | `wazuh-ca` @ `10.0.100.100` | Real deployment — full PKI |
| [`inventories/test-nossl`](inventories/test-nossl/) | **off** (`enable_ssl: false`) | none | Fast functional test, no certs |

```bash
# Secured (default)
ansible-playbook site.yml -i inventories/production/hosts.yml --ask-vault-pass

# No-SSL test build
ansible-playbook site.yml -i inventories/test-nossl/hosts.yml
```

When `enable_ssl` is **off**, the `certificates` role is skipped entirely and
the indexer runs with `plugins.security.disabled: true`.

### How the CA works (`enable_ssl: true`)

1. `wazuh-ca` downloads the Wazuh certs tool and generates the **root CA**.
2. It **signs** a certificate for every indexer / server / dashboard node
   (plus the indexer **admin** cert) from one `config.yml` built off the inventory.
3. The `certificates` role **copies each node's certs back** to that node
   (`root-ca.pem`, `<node>.pem`, `<node>-key.pem`) into the component's cert dir.
4. Each component role then consumes them (the indexer fails fast if its cert is missing).

## Indexer integration (manager → indexer)

The `wazuh_manager` role wires up **both** documented forwarders that move data
from the server cluster to the indexer cluster:

| Forwarder | Configured in | Carries | SSL source |
|---|---|---|---|
| **Filebeat** | `/etc/filebeat/filebeat.yml` | alerts / events (real time) | `/etc/filebeat/certs` |
| **Indexer connector** | `<indexer>` block in `ossec.conf` | vulnerability data (ECS) | `/etc/filebeat/certs` |

Both point at **every** indexer node (derived from the inventory) and flip
between `https`/`http` based on `enable_ssl`. Filebeat also pulls the Wazuh
alerts template + module and includes the `rseq` seccomp allow needed on Rocky 9.

Firewalld ports are opened per role: indexer `9200/9300`, manager
`1514/1515/1516/55000`, dashboard `443`.

## Load balancing (HAProxy)

The `load_balancer` role fronts the dashboards as the HA access layer on
`https://siem.local.domain`:

- **Default — SSL passthrough** (`mode tcp`): dashboards keep their own certs,
  TLS stays end-to-end. `balance source` keeps a user pinned to one dashboard
  (session stickiness). Health checks drop unresponsive backends.
- **Optional — TLS termination** (`lb_terminate_tls: true`): HAProxy terminates
  TLS with a certificate **issued by the local CA (`wazuh-ca`)** — no Let's
  Encrypt. The cert is signed by the same root CA the certificates role builds
  (SANs: `siem.local.domain`, the VIP, and each LB IP) and re-encrypts to the
  dashboards. If the CA hasn't been built yet, the role stops and tells you to
  run `playbooks/certificates.yml` first.
- **Stats**: `http://<lb>:8404/` (auth from vault), backed by the Runtime API socket.
- **Optional agent balancing** (`lb_balance_agents: true`): TCP balance agent
  events (1514) across all managers and enrollment (1515) to the master.
- **VIP HA**: with a second LB host, `keepalived` floats the VIP via VRRP
  (elect-by-priority, no split-brain on startup).

SELinux: the role sets `haproxy_connect_any` so HAProxy can reach backend ports
on Rocky 9. The config is validated (`haproxy -c -f`) before it is written.

## Scaling the deployment

Every tier has commented **expansion slots** in
[`inventories/production/hosts.yml`](inventories/production/hosts.yml). To grow
the deployment, uncomment a slot, assign the next free IP, and re-run the
relevant playbook:

| To add…              | Uncomment under group | Then run                                  |
|----------------------|-----------------------|-------------------------------------------|
| Another hot node     | `indexer_hot`         | `ansible-playbook playbooks/indexer.yml`  |
| Another warm node    | `indexer_warm`        | `ansible-playbook playbooks/indexer.yml`  |
| Another cold node    | `indexer_cold`        | `ansible-playbook playbooks/indexer.yml`  |
| Another server worker| `wazuh_manager_worker`| `ansible-playbook playbooks/manager.yml`  |
| Another dashboard    | `wazuh_dashboard`     | `ansible-playbook playbooks/dashboard.yml`|
| A second HAProxy (HA)| `load_balancer`       | `ansible-playbook playbooks/load_balancer.yml` |

> Keep the indexer **master-eligible** node count **odd** (3, 5, …) to preserve
> quorum when adding hot nodes.

## Repository layout

```
.
├── ansible.cfg                 # project defaults (inventory, become, ssh)
├── requirements.yml            # Galaxy collections
├── site.yml                    # master playbook (runs every tier in order)
├── inventories/
│   ├── production/             # real topology above
│   │   ├── hosts.yml
│   │   └── group_vars/         # all / per-tier / vault vars
│   └── staging/                # single-node-per-tier test topology
├── playbooks/                  # certificates, indexer, manager, dashboard, lb, agents
└── roles/
    ├── common/                 # repo, firewall, OS prep (Rocky 9)
    ├── certificates/           # CA + node TLS certs
    ├── wazuh_indexer/          # OpenSearch cluster + ISM tiers
    ├── wazuh_manager/          # master/worker cluster + Filebeat
    ├── wazuh_dashboard/        # dashboard frontends
    └── load_balancer/          # HAProxy + keepalived
```

## Quick start

```bash
# 1. Install collections
ansible-galaxy collection install -r requirements.yml

# 2. Create and encrypt secrets
cp inventories/production/group_vars/vault.yml.example \
   inventories/production/group_vars/vault.yml
ansible-vault encrypt inventories/production/group_vars/vault.yml

# 3. Check connectivity
ansible all -i inventories/production/hosts.yml -m ping

# 4. Deploy the full stack
ansible-playbook site.yml --ask-vault-pass
```

Deploy a single tier with its playbook, e.g.
`ansible-playbook playbooks/indexer.yml`.
