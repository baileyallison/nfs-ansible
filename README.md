# NFS High Availability Cluster — Ansible

Ansible playbooks and roles to deploy a highly available NFS server cluster using **Pacemaker**, **Corosync**, and **PCS** on RHEL/Rocky Linux (9) or Ubuntu 22 hosts.

## Architecture

The playbooks configure an **active/passive** two-node (or multi-node) HA cluster:

```
                  ┌──────────────┐
                  │  Virtual IP  │
                  │  (Cluster)   │
                  └──────┬───────┘
                         │
              ┌──────────┴──────────┐
              │                     │
        ┌─────┴─────┐        ┌─────┴─────┐
        │  Node 1   │        │  Node 2   │
        │ (active)  │        │ (standby) │
        │           │        │           │
        │ Pacemaker │        │ Pacemaker │
        │ Corosync  │        │ Corosync  │
        │ NFS       │        │ NFS       │
        └─────┬─────┘        └─────┬─────┘
              │                     │
              └──────────┬──────────┘
                         │
                  ┌──────┴───────┐
                  │ Shared       │
                  │ Storage      │
                  └──────────────┘
```

Pacemaker manages a **virtual IP**, the **NFS server** service, and an **NFS export filesystem** resource. If the active node fails, all resources are migrated to the standby node automatically.

## Prerequisites

- **Ansible** >= 2.12
- **Python** >= 3.8 on the control node
- **Collections**: `ansible.posix` (see [Installing dependencies](#installing-dependencies))
- **Target OS**: RHEL/Rocky Linux 9 *or* Ubuntu (22.04+)
- **Shared storage** accessible from all cluster nodes (cephfs)
- **Network**: All cluster nodes must be able to reach each other on the corosync port (UDP 5405) and the PCS daemon port (TCP 2224)
- SSH access with `become` (sudo) privileges to all target hosts

## Installing dependencies

Install the required Ansible collections:

```bash
ansible-galaxy collection install -r requirements.yml
```

## Configuration

### Inventory

Edit the inventory file at `inventory/hosts` (or `inventory/hosts.yml`) to define your NFS cluster nodes:

```ini
[nfs]
nfs-node-1 ansible_host=192.168.1.10
nfs-node-2 ansible_host=192.168.1.11
```

### Variables

Copy the example variable file and edit it with your environment-specific values:

```bash
cp group_vars/nfs.yml.example group_vars/nfs.yml
```

Key variables to configure:

| Variable | Description | Example |
|---|---|---|
| `cluster_name` | Name of the PCS cluster | `nfs-ha-cluster` |
| `hacluster_password` | Password for the `hacluster` user | *(use ansible-vault)* |
| `virtual_ip` | Floating IP for the NFS service | `192.168.1.100` |
| `nfs_shared_device` | Path to the shared storage device | `/dev/sdb1` |
| `nfs_mount_point` | Where the shared filesystem is mounted | `/srv/nfs` |
| `nfs_export_path` | Path exported to NFS clients | `/srv/nfs/share` |
| `nfs_export_options` | NFS export options string | `*(rw,sync,no_root_squash)` |
| `firewall_services` | List of firewalld services to allow | `[high-availability, nfs, mountd, rpc-bind]` |

### Sensitive values

Encrypt sensitive variables (such as `hacluster_password`) with Ansible Vault:

```bash
ansible-vault encrypt_string 'your_password_here' --name 'hacluster_password'
```

Paste the output into `group_vars/nfs.yml`, then run playbooks with `--ask-vault-pass` or `--vault-password-file`.

## Usage

### Deploy the cluster

```bash
ansible-playbook setup_nfs.yml
```

Run only specific stages with tags:

```bash
ansible-playbook setup_nfs.yml --tags os          # OS prerequisites only
ansible-playbook setup_nfs.yml --tags cluster      # HA cluster setup only
ansible-playbook setup_nfs.yml --tags nfs          # NFS configuration only
```

### Tear down the cluster

> **Warning**: This is destructive — it stops the cluster, removes all HA packages, and deletes configuration files.

```bash
ansible-playbook purge_nfs.yml
```

You will be prompted for confirmation before any changes are made.

## Roles

| Role | Description |
|---|---|
| `os_configs` | Installs prerequisite packages, configures firewall rules, sets up hostnames/hosts file |
| `ha_cluster` | Installs and configures Pacemaker, Corosync, and PCS; authenticates nodes; creates the cluster |
| `nfs_configs` | Configures NFS exports, creates Pacemaker resources for the VIP, filesystem, and NFS service |

## Testing

This project includes a basic [Molecule](https://molecule.readthedocs.io/) test scenario. To run it:

```bash
pip install molecule molecule-docker ansible-lint yamllint
molecule test
```

## Linting

```bash
yamllint .
ansible-lint
```

## License

GPL-3.0 — see [LICENSE](LICENSE) for details.
