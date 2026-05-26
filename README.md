# k3s Homelab

Single-node Kubernetes cluster on a Raspberry Pi 4B. Goal: self-host my own apps.

## Hardware

| Component | Details |
|---|---|
| Board | Raspberry Pi 4B |
| OS | Ubuntu Server 64-bit |
| Kubernetes | k3s v1.35.4 |
| Router | OpenWrt with AdGuard Home |

## Running Services

| Service    | Available at            |
|------------|-------------------------|
| Grafana    | http://< PI-IP >:32000  |
| Prometheus | internal (cluster-only) |
| InfluxDB   | http://< PI-IP >:32086  |

## Structure

```
k3s-homelab/
├── ansible/      # Automated setup via Ansible (Pi + k3s)
│   ├── inventory.ini
│   ├── playbook.yml
│   └── roles/    # common, k3s-server, k3s-agent
├── infra/        # Kubernetes manifests for infrastructure components
├── docs/         # Documentation and setup guides
│   └── setup.md  # Complete setup guide
└── README.md
```

## Ansible: Set Up Pi & k3s Automatically

The Ansible playbooks fully configure the Pi: system updates, SSH hardening, and k3s installation.

### Prerequisites

- Ansible installed on your local machine (`pip install ansible`)
- SSH access to the Pi (SSH key recommended over password)
- Pi reachable on the network

### Configuration Before First Run

**1. Edit `ansible/inventory.ini`:**

```ini
[k3s_server]
raspberry4b ansible_host=192.168.x.x ansible_user=<your-user>
```

- `ansible_host` → IP address of your Pi
- `ansible_user` → username on the Pi (e.g. `ubuntu` on a fresh Ubuntu Server image)

**2. Set up SSH key auth (recommended):**

Without an SSH key you'll be prompted for a password on every playbook run. With a key it works automatically:

```bash
ssh-copy-id <your-user>@192.168.x.x
```

### Running the Playbook

```bash
cd ansible

# With SSH key (recommended):
ansible-playbook -i inventory.ini playbook.yml --ask-become-pass

# With password auth:
ansible-playbook -i inventory.ini playbook.yml --ask-pass --ask-become-pass
```

- `--ask-become-pass` → sudo password on the Pi (required because playbooks use `become: true`)
- `--ask-pass` → SSH password (only needed if no SSH key is set up)

### What the Playbook Does

| Role | What happens |
|---|---|
| `common` | System updates, installs required packages, enables cgroups |
| `k3s-server` | Installs and starts k3s as the server node |
| `k3s-agent` | Sets up worker nodes (none configured yet) |

## Quick Start (after Ansible setup)

Prerequisites: `kubectl` and `helm` installed locally, kubeconfig configured.

```powershell
kubectl get nodes
kubectl get pods -n monitoring
```

Full setup guide: [docs/setup.md](docs/setup.md)
