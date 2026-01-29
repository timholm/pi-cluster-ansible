# Pi Cluster Ansible

Ansible playbooks for deploying a highly available Kubernetes cluster on Raspberry Pi nodes using kubeadm.

## Cluster Architecture

- **3 Control Plane Nodes**: rpi-1, cm3588-plus, rpi-3
- **6 Worker Nodes**: rpi-5, rpi-6, rpi-7, rpi-9, rpi-10, rpi-12

## Prerequisites

- Ansible installed on your local machine
- SSH access to all nodes
- Debian/Raspbian OS on all nodes

## Quick Start

1. **Set up vault password** for SSH credentials:
   ```bash
   ansible-vault create inventory/group_vars/all/vault.yml
   ```
   Add: `vault_ssh_password: "your_password"`

2. **Deploy the full cluster**:
   ```bash
   ansible-playbook site.yml --ask-vault-pass
   ```

## Playbooks

| Playbook | Description |
|----------|-------------|
| `site.yml` | Full cluster deployment (runs all playbooks) |
| `playbooks/00-prerequisites.yml` | System preparation (kernel modules, sysctl, swap) |
| `playbooks/01-containerd.yml` | Install and configure containerd runtime |
| `playbooks/02-kubeadm-install.yml` | Install kubeadm, kubelet, kubectl |
| `playbooks/03-init-control-plane.yml` | Initialize first control plane node |
| `playbooks/04-install-cni.yml` | Install Flannel CNI |
| `playbooks/05-join-control-plane.yml` | Join additional control plane nodes |
| `playbooks/06-join-workers.yml` | Join worker nodes |
| `playbooks/reset-cluster.yml` | Teardown and reset cluster |

## Individual Playbook Execution

Run playbooks step by step:

```bash
# Prepare all nodes
ansible-playbook playbooks/00-prerequisites.yml --ask-vault-pass

# Install containerd
ansible-playbook playbooks/01-containerd.yml --ask-vault-pass

# Install kubernetes tools
ansible-playbook playbooks/02-kubeadm-install.yml --ask-vault-pass

# Initialize cluster
ansible-playbook playbooks/03-init-control-plane.yml --ask-vault-pass

# Install CNI
ansible-playbook playbooks/04-install-cni.yml --ask-vault-pass

# Join other control plane nodes
ansible-playbook playbooks/05-join-control-plane.yml --ask-vault-pass

# Join workers
ansible-playbook playbooks/06-join-workers.yml --ask-vault-pass
```

## Reset Cluster

To completely tear down the cluster:

```bash
ansible-playbook playbooks/reset-cluster.yml --ask-vault-pass
```

## Inventory

Edit `inventory/hosts.yml` to match your node IPs and usernames.

## Configuration

- **Kubernetes version**: 1.32 (configurable in `playbooks/02-kubeadm-install.yml`)
- **CNI**: Flannel (configurable in `playbooks/04-install-cni.yml`)
- **Pod CIDR**: 10.244.0.0/16

## Node IPs

| Node | IP | Role |
|------|-----|------|
| rpi-1 | 192.168.8.197 | Control Plane |
| cm3588-plus | 192.168.8.152 | Control Plane |
| rpi-3 | 192.168.8.195 | Control Plane |
| rpi-5 | 192.168.8.108 | Worker |
| rpi-6 | 192.168.8.235 | Worker |
| rpi-7 | 192.168.8.209 | Worker |
| rpi-9 | 192.168.8.187 | Worker |
| rpi-10 | 192.168.8.210 | Worker |
| rpi-12 | 192.168.8.105 | Worker |
