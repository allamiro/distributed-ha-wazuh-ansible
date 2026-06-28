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
the single `wazuh_indexer` cluster; each node is tagged with `node.attr.temp`
(hot/warm/cold). This sets up the **foundation** for tier-based allocation —
the node attributes ISM/allocation rules key on. Note: the project does **not
yet create an ISM policy or index templates**, so data is not automatically
aged hot → warm → cold out of the box. To enable aging you'd add an ISM policy
(with `allocation` actions on `temp`) and attach it to the Wazuh index
templates — a good follow-up task.

### Optional components (defined but commented out in the inventory)

| Component          | Suggested IP   | Inventory group | Purpose                              |
|--------------------|----------------|-----------------|--------------------------------------|
| MinIO S3 storage   | `10.0.100.80+` | `minio`         | Snapshots / backups / ML artifacts   |
| External ML server | `10.0.100.90`  | `ml_server`     | Anomaly detection / enrichment       |
| Wazuh agents       | `10.0.200.0/24`| `wazuh_agents`  | Endpoint enrollment                  |

## Security modes & inventories

Security and HTTP TLS are **two independent toggles** (see
[group_vars/all.yml](inventories/production/group_vars/all.yml)):

- `enable_ssl` — security plugin + transport certs + **authentication (login)**. Needs the CA.
- `http_tls` — HTTPS on the REST/dashboard layer (can be off).

The project ships three inventories covering the useful combinations:

| Inventory | `enable_ssl` | `http_tls` | CA host | Login? | Use |
|---|---|---|---|---|---|
| [`inventories/production`](inventories/production/) | true | true | `wazuh-ca` @ `10.0.100.100` | ✅ HTTPS | Real deployment — full PKI |
| [`inventories/test-httpauth`](inventories/test-httpauth/) | true | false | `wazuh-ca` @ `10.0.100.100` | ✅ plain HTTP | Authenticated, no HTTPS |
| [`inventories/test-nossl`](inventories/test-nossl/) | false | false | none | ❌ no auth | Fast lab smoke test, no certs |

(There is also [`inventories/staging`](inventories/staging/) — a no-SSL
single-node-per-tier footprint for quick playbook testing.)

```bash
# Secured HTTPS (default) — replace <USER> with your sudo-capable deploy user
ansible-playbook site.yml -i inventories/production/hosts.yml -u <USER> -k -K

# Authenticated over plain HTTP (login works, no HTTPS)
ansible-playbook site.yml -i inventories/test-httpauth/hosts.yml -u <USER> -k -K

# No-SSL / no-auth lab smoke test
ansible-playbook site.yml -i inventories/test-nossl/hosts.yml -u <USER> -k -K
```

When `enable_ssl` is **off**, the `certificates` role is skipped entirely, the
indexer runs with `plugins.security.disabled: true`, and the dashboard's
security plugin is removed (no login). Transport TLS (and thus the CA) is
mandatory whenever `enable_ssl` is **on**, even if `http_tls` is off.

> **Version: this is a frozen 4.9.0 deployment.** `wazuh_version: 4.9.0` is
> pinned and packages install `wazuh-*-4.9.0` (via `wazuh_pin_packages`), matched
> to the Filebeat module `0.4` and the `v4.9.0` alerts template. It is **not**
> tracking the current docs (4.14 / Filebeat `0.5`). To move to current, set
> `wazuh_version` (e.g. `4.14.x`) and `filebeat_module_version: "0.5"` in
> group_vars.

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
between `https`/`http` based on `http_tls` (auth credentials are sent whenever
`enable_ssl` is on). Filebeat also pulls the Wazuh alerts template + module and
includes the `rseq` seccomp allow needed on Rocky 9.

Firewalld ports are opened per role: indexer `9200/9300`, manager
`1514/1515/1516/55000`, dashboard `443`.

**Credentials.** The actual passwords/keys are plain values in
[`group_vars/all.yml`](inventories/production/group_vars/all.yml) — **no Ansible
Vault required** (a deliberate choice for this lab project).

> ⚠️ **Production:** the shipped values are well-known Wazuh defaults
> (`admin`/`kibanaserver`/`wazuh-wui`, a sample cluster key, etc.). Before any
> real/exposed deployment you **must** replace them with unique secrets — and
> ideally encrypt them with `ansible-vault` (every `vault_*` reference still has
> a default, so vault is opt-in, not required). On the targets they're still kept out of the on-disk service
configs: Filebeat reads them from the **Filebeat keystore**
(`${username}`/`${password}` in `filebeat.yml`) and the indexer connector from
the **Wazuh manager keystore** (`wazuh-keystore -f indexer`). After first deploy,
harden by rotating the default `admin` / `wazuh-wui` passwords with Wazuh's
`wazuh-passwords-tool.sh` (a future `secure.yml` could automate this). Encrypting
the group_vars values in a `vault.yml` is optional.

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
- **Stats**: `http://<lb>:8404/` (auth from group_vars), backed by the Runtime API socket.
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
├── playbooks/                  # bootstrap, certificates, indexer, manager, dashboard, lb, agents
└── roles/
    ├── common/                 # repo, firewall, OS prep (Rocky 9)
    ├── certificates/           # CA + node TLS certs
    ├── wazuh_indexer/          # OpenSearch cluster + ISM tiers
    ├── wazuh_manager/          # master/worker cluster + Filebeat
    ├── wazuh_dashboard/        # dashboard frontends
    └── load_balancer/          # HAProxy + keepalived
```

## Deployment guide

### Prerequisites

- **Targets**: Rocky Linux 9 VMs (today: created/cloned in **Proxmox**), each
  reachable on the **static IP** assigned in the inventory.
- **SSH + sudo**: a login user on every host that can **sudo** (the playbooks
  `become: true`). Either set it in the inventory (`ansible_user`) or pass it on
  the CLI with `-u <USER>`.
- **Control node**: Ansible 2.14+ and Python 3.

> **`<USER>` placeholder.** In the commands below, replace `<USER>` with your
> deploy account (the one with sudo). Auth flags:
> `-u <USER>` login user · `-k` prompt for SSH password (omit and use
> `--private-key ~/.ssh/key` if you use a key) · `-K` prompt for the sudo
> password (omit if that user has passwordless sudo).

> **Hostnames are set by Ansible, from the inventory.** You only need the IPs to
> be reachable — you do **not** have to pre-set hostnames on the VMs. The
> bootstrap step names each box `<inventory_hostname>.local.domain` (e.g.
> `indexer-hot-1.local.domain`) and writes `/etc/hosts` for all peers. See
> [Hostnames & networking](#hostnames--networking).

### 1. Control node setup

```bash
ansible-galaxy collection install -r requirements.yml
```

### 1b. (Recommended) prep the hosts — internet check, OS update, reboot

Run once before `site.yml` (works for any inventory). It verifies internet/DNS
(pings `8.8.8.8`, resolves `google.com` + `packages.wazuh.com`), fully updates
the OS, then reboots and waits:
```bash
ansible-playbook -i inventories/<inv>/hosts.yml playbooks/prep.yml -u <USER> -k -K
```
Toggles: `-e prep_require_internet=false` (airgap/mirror), `-e prep_update=false`,
`-e prep_reboot=false`. If the connectivity check fails, fix DNS/routing or point
`wazuh_repo_baseurl` at an internal mirror before deploying.

### 2. Point the inventory at your VMs

Edit [`inventories/production/hosts.yml`](inventories/production/hosts.yml) so
each `ansible_host` matches the IP you gave the VM in Proxmox. The hostnames
(the inventory names) are applied to the VMs for you in the next step.

### 3. Credentials (no vault required)

Credentials are plain values in
[`inventories/production/group_vars/all.yml`](inventories/production/group_vars/all.yml)
(cluster key, indexer/dashboard/API passwords). The project runs **without
Ansible Vault** — just edit those values if you want to change them. Encrypting
them in a `vault.yml` is optional and not required to deploy.

### 4. Check connectivity

```bash
ansible all -i inventories/production/hosts.yml -u <USER> -k -m ping
```

### 5. Bootstrap — set hostnames, /etc/hosts, DNS, repo

```bash
ansible-playbook -i inventories/production/hosts.yml playbooks/bootstrap.yml -u <USER> -k -K
```

This names every VM from the inventory and wires DNS/peers. (`site.yml` runs it
first automatically; you can also run it standalone any time.)

### 6. Deploy the stack

```bash
# Everything, in order: bootstrap -> CA -> indexer -> manager -> dashboard -> LB
ansible-playbook -i inventories/production/hosts.yml site.yml -u <USER> -k -K
```

Or one tier at a time:

```bash
ansible-playbook -i inventories/production/hosts.yml playbooks/certificates.yml -u <USER> -k -K
ansible-playbook -i inventories/production/hosts.yml playbooks/indexer.yml      -u <USER> -k -K
ansible-playbook -i inventories/production/hosts.yml playbooks/manager.yml      -u <USER> -k -K
ansible-playbook -i inventories/production/hosts.yml playbooks/dashboard.yml    -u <USER> -k -K
ansible-playbook -i inventories/production/hosts.yml playbooks/load_balancer.yml -u <USER> -k -K
```

### 7. Verify

```bash
# hostnames were applied from the inventory
ansible all -i inventories/production/hosts.yml -u <USER> -k -a hostname

# indexer cluster is green and all nodes joined
ansible indexer-hot-1 -i inventories/production/hosts.yml -u <USER> -k -K -b \
  -a "curl -sk -u admin:<pw> https://localhost:9200/_cluster/health?pretty"

# manager cluster nodes
ansible wazuh_manager_master -i inventories/production/hosts.yml -u <USER> -k -K -b \
  -a "/var/ossec/bin/cluster_control -l"
```

Then browse to **https://siem.local.domain** (the HAProxy VIP).

### No-SSL test build

No CA, no vault — `enable_ssl: false`, so the certificates role is skipped.

```bash
ansible-playbook -i inventories/test-nossl/hosts.yml site.yml -u <USER> -k -K
```

### Uninstall / start clean

To completely remove the stack (packages + all data/config) and redeploy fresh —
works for **both** the SSL and no-SSL builds, since it tears down whatever is
installed. It's **destructive** and requires explicit confirmation:

```bash
ansible-playbook -i <inventory>/hosts.yml playbooks/uninstall.yml \
  -u <USER> -k -K -e confirm=yes
```

It reverts, in reverse order (dashboards → HAProxy/keepalived → managers+Filebeat
→ indexers → CA → agents), **everything the deployment changed**:

- **Packages**: wazuh-dashboard, haproxy, keepalived, wazuh-manager, filebeat,
  wazuh-indexer, wazuh-agent
- **Data/config**: `/var/ossec`, `/etc/wazuh-*`, `/var/lib/wazuh-*`,
  `/usr/share/wazuh-*`, `/etc/filebeat`, cert dirs, systemd drop-ins, CA workdir
- **Firewall**: closes every port opened (9200/9300, 1514/1515/1516/55000,
  80/443, 8404) and the keepalived VRRP rich rule
- **SELinux**: turns off `haproxy_connect_any` and removes the 8404 `http_port_t`
  label
- **sysctl**: removes `vm.max_map_count`
- **common**: removes the managed `/etc/hosts` block and reverts the
  NetworkManager DNS/search overrides

Add `-e remove_repo=yes` to also drop the Wazuh yum repo + GPG key, and
`-e reboot=yes` to reboot every node afterwards for a guaranteed-clean slate
(off by default — disruptive). Without `-e confirm=yes` it refuses to run.

> Intentionally **left in place**: base utilities installed by `common`
> (chrony, firewalld, python3, …) and the host's hostname — reverting those has
> no safe target and they don't affect a redeploy.

Then redeploy clean:
```bash
ansible-playbook -i <inventory>/hosts.yml site.yml -u <USER> -k -K
```

### Hostnames & networking

The `common` role (run by `bootstrap.yml`) is the single source of truth for host
identity, all derived from the inventory:

| What | Value | Set by |
|---|---|---|
| Hostname | `<inventory_hostname>.local.domain` | `node_fqdn` in [common/tasks/hostname.yml](roles/common/tasks/hostname.yml) |
| `/etc/hosts` | every peer's IP + FQDN + short name | [common/tasks/hosts_file.yml](roles/common/tasks/hosts_file.yml) |
| DNS / gateway | `10.0.0.1` | [common/tasks/network.yml](roles/common/tasks/network.yml) |

> **Now vs. later.** Today the Proxmox VMs already have the inventory IPs, so
> Ansible just names them and wires DNS. Later, **cloud-init** will create/clone
> the VMs and assign the IPs; this project stays the same — bootstrap still
> applies the hostnames, `/etc/hosts`, and repo on top.
