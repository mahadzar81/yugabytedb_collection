# Molecule Testing for YugabyteDB Ansible Collection

This directory contains Molecule scenarios for testing the YugabyteDB Ansible collection across multiple operating systems.

## Supported Platforms

The following OS versions are tested:

### RHEL Family
- **RHEL 8** - `molecule test -s rhel8`
- **RHEL 9** - `molecule test -s rhel9`
- **RHEL 10** - `molecule test -s rhel10`

### Debian Family
- **Debian 11 (Bullseye)** - `molecule test -s debian11`
- **Debian 12 (Bookworm)** - `molecule test -s debian12`
- **Debian 13 (Trixie)** - `molecule test -s debian13`

### Ubuntu Family
- **Ubuntu 22.04 (Jammy Jellyfish)** - `molecule test -s ubuntu2204`
- **Ubuntu 24.04 (Noble Numbat)** - `molecule test -s ubuntu2404`

## Usage

### Test a Specific Scenario

```bash
# Test on RHEL 8
molecule test -s rhel8

# Test on Debian 12
molecule test -s debian12

# Test on Ubuntu 24.04
molecule test -s ubuntu2404
```

### Test All Scenarios

```bash
# Run all scenarios sequentially
for scenario in rhel8 rhel9 rhel10 debian11 debian12 debian13 ubuntu2204 ubuntu2404; do
    molecule test -s $scenario
done
```

### Individual Molecule Commands

```bash
# Create instances
molecule create -s rhel8

# Run converge (apply the roles)
molecule converge -s rhel8

# Verify the state
molecule verify -s rhel8

# Clean up
molecule destroy -s rhel8
```

## Configuration

Each scenario uses the **delegated driver**, which means:

1. You need to provide your own target hosts (via SSH or other connection methods)
2. The scenarios define inventory variables like `target_os` and `target_os_version` for role logic
3. Connection details should be configured via environment variables or inventory files

### Environment Variables

You can customize connections using environment variables:

```bash
export ANSIBLE_HOST_KEY_CHECKING=False
export ANSIBLE_USER=your_ssh_user
export ANSIBLE_PASSWORD=your_ssh_password  # if using password auth
export ANSIBLE_PRIVATE_KEY_FILE=/path/to/key  # if using key auth
```

### Customizing Inventory

To test against actual remote hosts, modify the inventory section in each scenario's `molecule.yml`:

```yaml
provisioner:
  inventory:
    hosts:
      all:
        hosts:
          rhel8:
            ansible_host: your-rhel8-host.example.com
            ansible_user: root
            ansible_python_interpreter: /usr/bin/python3
            target_os: rhel
            target_os_version: "8"
```

## Playbooks

- **converge.yml** - Applies all three roles (install, configure, manage)
- **verify.yml** - Verifies the system state after convergence

## Adding New Scenarios

To add a new OS version:

1. Copy an existing scenario directory:
   ```bash
   cp -r molecule/rhel9 molecule/rhel10
   ```

2. Update the `molecule.yml` in the new directory:
   - Change the scenario name
   - Update platform name
   - Update `target_os` and `target_os_version` variables

3. Test the new scenario:
   ```bash
   molecule test -s rhel10
   ```

## Troubleshooting

### Connection Issues

If you encounter SSH connection errors:
- Verify SSH access to target hosts
- Check that Python 3 is installed on target hosts
- Ensure proper authentication (key-based or password)

### Role-Specific Issues

For role-specific failures, check:
- Role defaults in `roles/<role_name>/defaults/main.yml`
- Task definitions in `roles/<role_name>/tasks/main.yml`
- Handler definitions in `roles/<role_name>/handlers/main.yml`
