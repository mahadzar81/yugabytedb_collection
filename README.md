# YugabyteDB Ansible Collection

[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)
[![Ansible Galaxy](https://img.shields.io/badge/Galaxy-yugabytedb.yugabytedb-blue.svg)](https://galaxy.ansible.com/yugabytedb/yugabytedb)

Automate the deployment and management of **YugabyteDB** multi-node distributed SQL database clusters across RHEL, Debian, and Ubuntu distributions.

## ✨ Features

- 🚀 **Full Automation**: Complete cluster bootstrap from bare OS to running cluster
- 🔧 **System Tuning**: Automatic THP disable, ulimits, kernel params, and swap configuration
- 🔒 **Security**: Firewall configuration (firewalld/ufw) with required ports
- 📦 **Multi-OS Support**: RHEL 8/9/10, Debian 11/12/13, Ubuntu 22.04/24.04
- 🎯 **Flexible Configuration**: Customizable replication factor, ports, data directories
- 🔄 **Lifecycle Management**: Start, stop, restart, and status operations
- 🗄️ **Dual API Support**: YSQL (PostgreSQL-compatible) and YCQL (Cassandra-compatible)

## 📋 Supported Platforms

| Distribution | Versions | Package Manager | Firewall |
|-------------|----------|----------------|----------|
| **RHEL/Rocky/AlmaLinux** | 8, 9, 10 | dnf/yum | firewalld |
| **Debian** | 11, 12, 13 | apt | ufw |
| **Ubuntu** | 22.04, 24.04 | apt | ufw |

## 📦 Installation

### From Ansible Galaxy (Recommended)

```bash
ansible-galaxy collection install yugabytedb.yugabytedb
```

### From Source

```bash
# Clone the repository
git clone https://github.com/yugabytedb/ansible-collection.git
cd ansible-collection

# Build and install
ansible-galaxy collection build
ansible-galaxy collection install yugabytedb-yugabytedb-*.tar.gz
```

### Verify Installation

```bash
ansible-galaxy collection list | grep yugabytedb
```

## 🏗️ Architecture

This collection provides three specialized roles:

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│    INSTALL      │────▶│   CONFIGURE     │────▶│     MANAGE      │
│                 │     │                 │     │                 │
│ • OS Validation │     │ • Config Files  │     │ • Start/Stop    │
│ • Prerequisites │     │ • Systemd Units │     │ • Restart       │
│ • Download DB   │     │ • Firewall      │     │ • Status Check  │
│ • Extract       │     │ • Cluster Init  │     │                 │
│ • System Tuning │     │ • Bootstrap     │     │                 │
└─────────────────┘     └─────────────────┘     └─────────────────┘
```

## 🚀 Quick Start

### 1. Create Inventory

Create an inventory file (`inventory.ini`):

```ini
[masters]
master1 ansible_host=192.168.1.10
master2 ansible_host=192.168.1.11
master3 ansible_host=192.168.1.12

[tservers]
tserver1 ansible_host=192.168.1.20
tserver2 ansible_host=192.168.1.21
tserver3 ansible_host=192.168.1.22

[yugabytedb_nodes:children]
masters
tservers

[yugabytedb_nodes:vars]
ansible_user=centos
ansible_become=true
ansible_python_interpreter=/usr/bin/python3
```

### 2. Deploy Cluster

Run the deployment playbook:

```bash
ansible-playbook -i inventory.ini playbooks/deploy_cluster.yml
```

### 3. Verify Cluster

```bash
# Check cluster status
ansible-playbook -i inventory.ini playbooks/manage_cluster.yml \
  -e "yugabytedb_action=status"

# Connect to YSQL
ysqlsh -h master1 -p 5433

# Connect to YCQL
ycqlsh tserver1 9042
```

## 📖 Roles Reference

### `yugabytedb.install`

Installs YugabyteDB binaries and performs system preparation.

**Key Tasks:**
- Validates OS compatibility
- Installs prerequisites (libedit, numactl, etc.)
- Downloads and extracts YugabyteDB
- Creates `yugabyte` user and groups
- Configures ulimits and kernel parameters
- Disables Transparent Huge Pages (THP)
- Sets up directory structure

**Example:**
```yaml
- hosts: yugabytedb_nodes
  become: true
  roles:
    - yugabytedb.yugabytedb.install
```

### `yugabytedb.configure`

Configures cluster settings and initializes the database.

**Key Tasks:**
- Generates gflags configuration files
- Creates systemd service units
- Configures firewall rules
- Bootstraps master nodes
- Joins tserver nodes to cluster
- Enables services on boot

**Example:**
```yaml
- hosts: yugabytedb_nodes
  become: true
  vars:
    yugabytedb_master_nodes: "{{ groups['masters'] }}"
    yugabytedb_tserver_nodes: "{{ groups['tservers'] }}"
  roles:
    - yugabytedb.yugabytedb.configure
```

### `yugabytedb.manage`

Manages cluster lifecycle operations.

**Supported Actions:**
- `start` - Start all services
- `stop` - Stop all services
- `restart` - Restart all services
- `status` - Check service status
- `enable` - Enable on boot
- `disable` - Disable on boot

**Example:**
```yaml
- hosts: yugabytedb_nodes
  become: true
  vars:
    yugabytedb_action: restart
  roles:
    - yugabytedb.yugabytedb.manage
```

## ⚙️ Variables Reference

### Core Variables

| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `yugabytedb_version` | YugabyteDB version to install | `2.40.2.0` | No |
| `yugabytedb_download_url` | Custom download URL | Auto-generated | No |
| `yugabytedb_install_dir` | Installation directory | `/opt/yugabyte` | No |
| `yugabytedb_data_dir` | Data directory | `/var/lib/yugabyte` | No |
| `yugabytedb_log_dir` | Log directory | `/var/log/yugabyte` | No |
| `yugabytedb_user` | System user | `yugabyte` | No |
| `yugabytedb_group` | System group | `yugabyte` | No |

### Network Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `yugabytedb_bind_address` | Bind address | `0.0.0.0` |
| `yugabytedb_master_ports` | Master RPC port | `7100` |
| `yugabytedb_master_ui_port` | Master UI port | `7000` |
| `yugabytedb_tserver_ports` | TServer RPC port | `9100` |
| `yugabytedb_tserver_ui_port` | TServer UI port | `9000` |
| `yugabytedb_ysql_port` | YSQL port | `5433` |
| `yugabytedb_ycql_port` | YCQL port | `9042` |
| `yugabytedb_redis_port` | Redis port | `6379` |

### Cluster Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `yugabytedb_master_nodes` | List of master hostnames/IPs | Auto-detected |
| `yugabytedb_tserver_nodes` | List of tserver hostnames/IPs | Auto-detected |
| `yugabytedb_replication_factor` | Replication factor | `3` |
| `yugabytedb_enable_ysql` | Enable YSQL API | `true` |
| `yugabytedb_enable_ycql` | Enable YCQL API | `true` |
| `yugabytedb_enable_redis` | Enable Redis API | `false` |

### System Tuning Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `yugabytedb_disable_thp` | Disable THP | `true` |
| `yugabytedb_configure_ulimits` | Configure ulimits | `true` |
| `yugabytedb_configure_firewall` | Configure firewall | `true` |
| `yugabytedb_ulimit_nofile` | Open files limit | `65536` |
| `yugabytedb_ulimit_nproc` | Max processes | `131072` |

## 📁 Directory Structure

```
yugabytedb_collection/
├── galaxy.yml                    # Collection metadata
├── README.md                     # This file
├── meta/
│   └── runtime.yml               # Runtime requirements
├── roles/
│   ├── install/
│   │   ├── tasks/
│   │   │   └── main.yml          # Installation tasks
│   │   └── defaults/
│   │       └── main.yml          # Default variables
│   ├── configure/
│   │   ├── tasks/
│   │   │   └── main.yml          # Configuration tasks
│   │   ├── templates/
│   │   │   ├── yugabyte-master.service.j2
│   │   │   ├── yugabyte-tserver.service.j2
│   │   │   ├── master_gflags.conf.j2
│   │   │   └── tserver_gflags.conf.j2
│   │   └── defaults/
│   │       └── main.yml          # Default variables
│   └── manage/
│       ├── tasks/
│       │   └── main.yml          # Management tasks
│       └── defaults/
│           └── main.yml          # Default variables
├── playbooks/
│   ├── deploy_cluster.yml        # Full deployment playbook
│   ├── manage_cluster.yml        # Management playbook
│   └── inventory.example         # Sample inventory
└── tests/
    └── test.yml                  # Test playbook
```

## 🔧 Advanced Usage

### Custom Configuration

```yaml
- hosts: yugabytedb_nodes
  become: true
  vars:
    yugabytedb_version: "2.40.2.0"
    yugabytedb_replication_factor: 5
    yugabytedb_data_dir: "/mnt/data/yugabyte"
    yugabytedb_log_dir: "/mnt/logs/yugabyte"
    yugabytedb_enable_redis: true
    yugabytedb_redis_port: 6380
    yugabytedb_ulimit_nofile: 131072
    
    # Custom gflags
    yugabytedb_custom_master_flags:
      - "--logtostderr=true"
      - "--v=2"
    yugabytedb_custom_tserver_flags:
      - "--logtostderr=true"
      - "--cache_size_mb=4096"
      
  roles:
    - yugabytedb.yugabytedb.install
    - yugabytedb.yugabytedb.configure
```

### Rolling Restart

```bash
# Perform rolling restart (one node at a time)
ansible-playbook -i inventory.ini playbooks/manage_cluster.yml \
  -e "yugabytedb_action=restart" \
  --limit "tserver1"
  
# Repeat for each node
```

### Upgrade Workflow

```bash
# 1. Stop cluster
ansible-playbook -i inventory.ini playbooks/manage_cluster.yml \
  -e "yugabytedb_action=stop"

# 2. Update version variable and run install
ansible-playbook -i inventory.ini playbooks/deploy_cluster.yml \
  -e "yugabytedb_version=2.41.0.0"

# 3. Start cluster
ansible-playbook -i inventory.ini playbooks/manage_cluster.yml \
  -e "yugabytedb_action=start"
```

## 🔍 Troubleshooting

### Check Service Status

```bash
# Via Ansible
ansible-playbook -i inventory.ini playbooks/manage_cluster.yml \
  -e "yugabytedb_action=status"

# Direct SSH
ssh <node> systemctl status yugabyte-master
ssh <node> systemctl status yugabyte-tserver
```

### View Logs

```bash
# Master logs
ssh <node> journalctl -u yugabyte-master -f

# TServer logs
ssh <node> journalctl -u yugabyte-tserver -f

# Application logs
ssh <node> tail -f /var/log/yugabyte/master/*.INFO
```

### Common Issues

**Issue: THP not disabled**
```bash
# Verify THP status
cat /sys/kernel/mm/transparent_hugepage/enabled
# Should show: [never] always madvise
```

**Issue: Firewall blocking connections**
```bash
# RHEL
firewall-cmd --list-all

# Debian/Ubuntu
ufw status verbose
```

**Issue: Port conflicts**
```bash
# Check if ports are in use
ss -tlnp | grep -E '7100|9100|5433|9042'
```

## 🤝 Contributing

Contributions are welcome! Please follow these steps:

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/amazing-feature`)
3. Commit your changes (`git commit -m 'Add amazing feature'`)
4. Push to the branch (`git push origin feature/amazing-feature`)
5. Open a Pull Request

### Development Setup

```bash
# Install collection in development mode
ansible-galaxy collection build --force
ansible-galaxy collection install --force yugabytedb-yugabytedb-*.tar.gz

# Run tests
ansible-playbook tests/test.yml -i tests/inventory
```

## 📄 License

This collection is licensed under the [Apache License 2.0](LICENSE).

## 🙏 Acknowledgments

- [YugabyteDB](https://www.yugabyte.com/) - The distributed SQL database
- [Ansible](https://www.ansible.com/) - Automation engine
- Community contributors and maintainers

## 📞 Support

- **Documentation**: [YugabyteDB Docs](https://docs.yugabyte.com/)
- **Issues**: [GitHub Issues](https://github.com/yugabytedb/ansible-collection/issues)
- **Discussions**: [GitHub Discussions](https://github.com/yugabytedb/ansible-collection/discussions)
- **Slack**: [YugabyteDB Community Slack](https://www.yugabyte.com/slack/)
