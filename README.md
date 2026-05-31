# yugabytedb_collection – Ansible Collection

[![CI](https://github.com/yugabyte/ansible-collection-yugabytedb/actions/workflows/ci.yml/badge.svg)](https://github.com/yugabyte/ansible-collection-yugabytedb/actions/workflows/ci.yml)
[![Ansible Galaxy](https://img.shields.io/badge/galaxy-yugabyte.db-blue.svg)](https://galaxy.ansible.com/yugabyte/db)
[![License](https://img.shields.io/badge/license-Apache--2.0-green.svg)](LICENSE)

An Ansible collection for deploying, configuring, and managing **YugabyteDB** clusters on enterprise Linux and Debian/Ubuntu distributions.

---

## Table of Contents

- [Supported Platforms](#supported-platforms)
- [Requirements](#requirements)
- [Collection Structure](#collection-structure)
- [Quick Start](#quick-start)
- [Inventory Layout](#inventory-layout)
- [Roles Reference](#roles-reference)
  - [yugabyte_preflight](#yugabyte_preflight)
  - [yugabyte_install](#yugabyte_install)
  - [yugabyte_configure](#yugabyte_configure)
  - [yugabyte_master](#yugabyte_master)
  - [yugabyte_tserver](#yugabyte_tserver)
  - [yugabyte_cluster](#yugabyte_cluster)
- [Playbooks Reference](#playbooks-reference)
- [Variables Reference](#variables-reference)
- [TLS Configuration](#tls-configuration)
- [Rolling Restart](#rolling-restart)
- [Upgrading the Cluster](#upgrading-the-cluster)
- [Testing with Molecule](#testing-with-molecule)
- [Contributing](#contributing)
- [License](#license)

---

## Supported Platforms

| Distribution       | Versions           |
|--------------------|--------------------|
| RHEL / Rocky / AlmaLinux | 8, 9, 10   |
| Debian             | 11, 12, 13         |
| Ubuntu             | 22.04, 24.04       |

---

## Requirements

**Control node**

| Dependency | Minimum version |
|---|---|
| Python | 3.10 |
| Ansible | 2.14 |
| ansible.posix | 1.5.0 |
| community.general | 7.0.0 |

**Managed nodes**

- SSH access with privilege escalation (`become: true`)
- Python 3 installed (or bootstrapped by Ansible)
- Internet access to `downloads.yugabyte.com` (or a local mirror configured via `yugabyte_download_base_url`)

### Install the collection

```bash
ansible-galaxy collection install yugabytedb_collection
```

Or pin a version in your `requirements.yml`:

```yaml
collections:
  - name: yugabytedb_collection
    version: ">=1.0.0"
```

Then install:

```bash
ansible-galaxy collection install -r requirements.yml
```

---

## Collection Structure

```
yugabytedb_collection/
├── galaxy.yml                  # Collection manifest
├── requirements.yml            # Collection dependencies
├── .ansible-lint               # Lint rules
├── .yamllint.yml               # YAML style rules
├── .github/
│   └── workflows/
│       ├── ci.yml              # CI: lint + molecule tests
│       └── release.yml         # Release: build + publish to Galaxy
├── inventory/
│   └── hosts.yml               # Sample inventory
├── molecule/
│   ├── default/                # Unit tests (preflight role, multi-OS)
│   │   ├── molecule.yml
│   │   ├── converge.yml
│   │   └── verify.yml
│   └── cluster/                # Integration tests (6-node stub cluster)
│       ├── molecule.yml
│       ├── prepare.yml
│       ├── converge.yml
│       └── verify.yml
├── playbooks/
│   ├── deploy.yml              # Full cluster deployment
│   ├── rolling_restart.yml     # Rolling restart
│   └── upgrade.yml             # Version upgrade
└── roles/
    ├── yugabyte_preflight/     # OS checks, packages, kernel tuning
    ├── yugabyte_install/       # Binary download & extraction
    ├── yugabyte_configure/     # Config files & systemd units
    ├── yugabyte_master/        # Start masters, init cluster
    ├── yugabyte_tserver/       # Start tservers, verify registration
    └── yugabyte_cluster/       # Health checks, deployment summary
```

---

## Quick Start

### 1. Create your inventory

```yaml
# inventory/hosts.yml
yugabyte_masters:
  hosts:
    node1: { ansible_host: 10.0.1.10 }
    node2: { ansible_host: 10.0.1.11 }
    node3: { ansible_host: 10.0.1.12 }

yugabyte_tservers:
  hosts:
    node1: { ansible_host: 10.0.1.10 }
    node2: { ansible_host: 10.0.1.11 }
    node3: { ansible_host: 10.0.1.12 }

yugabyte_nodes:
  children:
    yugabyte_masters: {}
    yugabyte_tservers: {}

all:
  vars:
    ansible_user: ansible
    yugabyte_version: "2.20.3.0"
    yugabyte_build: "b68"
    yugabyte_replication_factor: 3
```

### 2. Run the deploy playbook

```bash
ansible-playbook \
  -i inventory/hosts.yml \
  ~/.ansible/collections/ansible_collections/yugabyte/db/playbooks/deploy.yml
```

After a successful run you will see a summary like:

```
═══════════════════════════════════════════════════
 YugabyteDB cluster deployment SUCCESS
═══════════════════════════════════════════════════
 Replication factor : 3
 Masters            : node1, node2, node3
 TServers           : node1, node2, node3
 YSQL port          : 5433
 YCQL port          : 9042
 Master UI          : http://10.0.1.10:7000
 TServer UI         : http://10.0.1.10:9000
═══════════════════════════════════════════════════
```

---

## Inventory Layout

The collection uses three inventory groups:

| Group | Purpose |
|---|---|
| `yugabyte_masters` | Nodes that run `yb-master` |
| `yugabyte_tservers` | Nodes that run `yb-tserver` |
| `yugabyte_nodes` | Union group (children of both above) |

In a typical 3-node RF-3 cluster each node is in all three groups (combined topology). For larger clusters you can separate masters from tservers.

---

## Roles Reference

### yugabyte_preflight

Validates OS compatibility, hardware requirements, installs prerequisite packages, configures chrony (NTP), applies kernel tuning, and creates the `yugabyte` OS user.

**Key variables:**

| Variable | Default | Description |
|---|---|---|
| `yugabyte_min_ram_mb` | `2048` | Minimum RAM in MB |
| `yugabyte_min_cpu_cores` | `2` | Minimum vCPU count |
| `yugabyte_min_disk_gb` | `20` | Minimum free disk (GB) |
| `yugabyte_configure_ntp` | `true` | Manage chrony |
| `yugabyte_configure_kernel` | `true` | Apply sysctl + THP settings |
| `yugabyte_manage_firewall` | `true` | Open ports in firewalld/ufw |
| `yugabyte_transparent_hugepages` | `madvise` | THP setting (`never`/`madvise`/`always`) |

---

### yugabyte_install

Downloads the YugabyteDB tarball from the official CDN (or a custom mirror), extracts it, runs `post_install.sh`, and creates the `current` symlink.

**Key variables:**

| Variable | Default | Description |
|---|---|---|
| `yugabyte_version` | `2.20.3.0` | YugabyteDB version |
| `yugabyte_build` | `b68` | Build number |
| `yugabyte_install_dir` | `/opt/yugabyte` | Base installation directory |
| `yugabyte_data_dir` | `/mnt/data/yugabyte` | Data directory |
| `yugabyte_log_dir` | `/var/log/yugabyte` | Log directory |
| `yugabyte_download_base_url` | `https://downloads.yugabyte.com/releases` | Override for air-gapped environments |
| `yugabyte_keep_tarball` | `false` | Keep tarball after extraction |

---

### yugabyte_configure

Renders `yb-master.conf`, `yb-tserver.conf`, and systemd unit files from Jinja2 templates.

**Key variables:**

| Variable | Default | Description |
|---|---|---|
| `yugabyte_replication_factor` | `3` | Cluster replication factor |
| `yugabyte_enable_ysql` | `true` | Enable YSQL (PostgreSQL-compatible) |
| `yugabyte_enable_ycql` | `true` | Enable YCQL (Cassandra-compatible) |
| `yugabyte_enable_yedis` | `false` | Enable YEDIS (Redis-compatible) |
| `yugabyte_ysql_auth` | `false` | Enable YSQL authentication |
| `yugabyte_ycql_auth` | `false` | Enable YCQL authentication |
| `yugabyte_enable_tls` | `false` | Enable TLS encryption |
| `yugabyte_master_extra_flags` | `{}` | Additional `yb-master` flags (dict) |
| `yugabyte_tserver_extra_flags` | `{}` | Additional `yb-tserver` flags (dict) |

**Extra flags example:**

```yaml
yugabyte_master_extra_flags:
  log_min_duration_statement: "1000"
  ysql_pg_conf_csv: "log_connections=on,log_disconnections=on"
```

---

### yugabyte_master

Enables and starts the `yb-master` systemd service, waits for the RPC and HTTP ports, and (on the first master only) runs `yb-admin list_all_masters` to verify the cluster is operational.

---

### yugabyte_tserver

Enables and starts the `yb-tserver` systemd service, waits for RPC, YSQL, and YCQL ports, then verifies the tserver has registered with the masters.

---

### yugabyte_cluster

Runs a final health check (`list_all_masters`, `list_all_tablet_servers`), asserts the expected number of tservers are registered, optionally runs a YSQL `SELECT version()` health query, and prints a deployment summary.

---

## Playbooks Reference

| Playbook | Purpose |
|---|---|
| `playbooks/deploy.yml` | Full cluster deployment (end-to-end) |
| `playbooks/rolling_restart.yml` | Rolling restart of masters then tservers |
| `playbooks/upgrade.yml` | Install new version + rolling restart |

---

## Variables Reference

All variables have defaults defined in each role's `defaults/main.yml`. The most commonly overridden variables at the inventory level are:

```yaml
# Version
yugabyte_version: "2.20.3.0"
yugabyte_build: "b68"

# Cluster topology
yugabyte_replication_factor: 3

# Directories
yugabyte_install_dir: /opt/yugabyte
yugabyte_data_dir: /mnt/data/yugabyte
yugabyte_log_dir: /var/log/yugabyte

# Features
yugabyte_enable_ysql: true
yugabyte_enable_ycql: true
yugabyte_enable_yedis: false

# Security
yugabyte_enable_tls: false
yugabyte_ysql_auth: false
yugabyte_ycql_auth: false
```

---

## TLS Configuration

1. Generate CA and node certificates (e.g., using `openssl` or `cfssl`).
2. Place certs in a `files/certs/` directory relative to your playbook.
3. Set the following variables:

```yaml
yugabyte_enable_tls: true
yugabyte_cert_dir: /etc/yugabyte/certs
yugabyte_ca_cert: files/certs/ca.crt
yugabyte_node_cert: files/certs/node.crt
yugabyte_node_key: files/certs/node.key
```

Certificates are deployed by the `yugabyte_configure` role and referenced in both master and tserver configuration files.

---

## Rolling Restart

```bash
ansible-playbook \
  -i inventory/hosts.yml \
  ~/.ansible/collections/ansible_collections/yugabyte/db/playbooks/rolling_restart.yml
```

To restart only tservers:

```bash
ansible-playbook \
  -i inventory/hosts.yml \
  ~/.ansible/collections/ansible_collections/yugabyte/db/playbooks/rolling_restart.yml \
  -e "restart_masters=false restart_tservers=true"
```

---

## Upgrading the Cluster

```bash
ansible-playbook \
  -i inventory/hosts.yml \
  ~/.ansible/collections/ansible_collections/yugabyte/db/playbooks/upgrade.yml \
  -e "yugabyte_version=2.21.0.0 yugabyte_build=b12"
```

The upgrade playbook:
1. Installs the new binaries alongside the old ones.
2. Updates the `current` symlink.
3. Renders updated configuration files.
4. Performs a rolling restart of masters (one at a time) then tservers (one at a time).
5. Runs a final cluster health check.

---

## Testing with Molecule

### Prerequisites

```bash
pip install molecule molecule-plugins[docker] ansible-core docker
```

### Run the default scenario (preflight unit tests, all OS variants)

```bash
cd yugabytedb_collection
molecule test --scenario-name default
```

### Run the cluster integration scenario

```bash
cd yugabytedb_collection
molecule test --scenario-name cluster
```

### Run a single test step

```bash
molecule converge --scenario-name default   # provisioning only
molecule verify   --scenario-name default   # assertions only
molecule destroy  --scenario-name default   # cleanup
```

### CI matrix

The GitHub Actions workflow runs Molecule tests against all eight supported OS images in parallel:

| Image | OS |
|---|---|
| `geerlingguy/docker-rockylinux8-ansible` | RHEL 8 |
| `geerlingguy/docker-rockylinux9-ansible` | RHEL 9 |
| `borisskert/docker-almalinux10-ansible` | RHEL 10 |
| `geerlingguy/docker-debian11-ansible` | Debian 11 |
| `geerlingguy/docker-debian12-ansible` | Debian 12 |
| `geerlingguy/docker-debian13-ansible` | Debian 13 |
| `geerlingguy/docker-ubuntu2204-ansible` | Ubuntu 22.04 |
| `geerlingguy/docker-ubuntu2404-ansible` | Ubuntu 24.04 |

---

## Contributing

1. Fork the repository.
2. Create a feature branch (`git checkout -b feature/my-feature`).
3. Run linting locally: `yamllint yugabyte/db && ansible-lint yugabyte/db`.
4. Run at least the default Molecule scenario locally.
5. Open a pull request against `main`.

---

## License

Apache-2.0 – see [LICENSE](LICENSE) for details.
