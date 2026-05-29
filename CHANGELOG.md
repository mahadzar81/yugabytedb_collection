# Changelog

All notable changes to the `yugabyte.db` Ansible collection will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.0.0] - 2026-05-30

### Added

- `yugabyte_preflight` role: OS/hardware validation, package installation, chrony, sysctl tuning, THP configuration, firewall management, and `yugabyte` OS user creation. Supports RHEL 8/9/10, Debian 11/12/13, Ubuntu 22.04/24.04.
- `yugabyte_install` role: downloads and extracts YugabyteDB binaries from the official CDN (or a custom mirror), runs `post_install.sh`, and manages the `current` symlink.
- `yugabyte_configure` role: renders `yb-master.conf`, `yb-tserver.conf`, and systemd unit files from Jinja2 templates. Supports TLS, YSQL/YCQL/YEDIS feature flags, and arbitrary extra flags.
- `yugabyte_master` role: starts `yb-master` via systemd, waits for RPC and HTTP ports, and performs one-time cluster initialisation on the primary master.
- `yugabyte_tserver` role: starts `yb-tserver` via systemd, waits for all service ports, and verifies tserver registration with the masters.
- `yugabyte_cluster` role: post-deploy health validation (`list_all_masters`, `list_all_tablet_servers`), optional YSQL connectivity check, and deployment summary.
- `playbooks/deploy.yml`: full end-to-end cluster deployment playbook.
- `playbooks/rolling_restart.yml`: serial rolling restart of masters then tservers.
- `playbooks/upgrade.yml`: installs new YugabyteDB version and performs a rolling restart.
- Molecule `default` scenario: unit tests the preflight role across all 8 supported OS images.
- Molecule `cluster` scenario: 6-container integration test using binary stubs.
- GitHub Actions CI workflow: lint + multi-OS Molecule matrix + Galaxy publish on merge to `main`.
- GitHub Actions release workflow: tag-triggered build, GitHub Release creation, and Galaxy publish.