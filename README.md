<div align="center">

# 🏠 k3s Homelab

**Single-node Kubernetes on a Raspberry Pi 4B — self-hosting my own apps with GitOps.**

![k3s](https://img.shields.io/badge/k3s-v1.35.4-FFC61C?logo=k3s&logoColor=black)
![Raspberry Pi](https://img.shields.io/badge/Raspberry%20Pi-4B%20·%202GB-A22846?logo=raspberrypi&logoColor=white)
![Flux CD](https://img.shields.io/badge/GitOps-Flux%20CD-5468FF?logo=flux&logoColor=white)
![Ansible](https://img.shields.io/badge/Provisioning-Ansible-EE0000?logo=ansible&logoColor=white)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-16-4169E1?logo=postgresql&logoColor=white)

</div>

---

## ✨ Overview

| | |
|---|---|
| **Board** | Raspberry Pi 4B (2 GB RAM) |
| **OS** | Ubuntu Server 64-bit |
| **Kubernetes** | k3s v1.35.4 (single node) |
| **Network** | OpenWrt router with AdGuard Home |
| **Delivery** | GitHub Actions → GHCR → Flux CD |

## 🗺️ Architecture

```mermaid
flowchart LR
    dev([git push]) --> gha[GitHub Actions<br/>build arm64 image]
    gha --> ghcr[(GHCR)]
    gha -- bump image tag --> repo[k3s-homelab repo]

    subgraph pi[Raspberry Pi 4B · k3s]
        flux[Flux CD] --> server[Plant Watering<br/>Server]
        server --> pg[(PostgreSQL)]
        server <--> mqtt[Mosquitto]
    end

    repo -. watched by .-> flux
    ghcr -. image pull .-> server
    esp[ESP32] <-- MQTT --> mqtt
    app[Mobile App] -- REST --> server
```

**How a deploy works:** a push to the app repo builds an arm64 image, pushes it to GHCR and commits the new image tag into this repo. Flux notices the commit and rolls out the new version — no manual `kubectl apply`.

## 🚀 Running Services

| Service | Namespace | Access |
|---|---|---|
| 🌱 Plant Watering System Server | `apps` | `http://<PI-IP>:30080` |
| 📡 Mosquitto (MQTT broker) | `apps` | `mqtt://<PI-IP>:31883` |
| 🐘 PostgreSQL | `database` | cluster-internal · NodePort `32432` for local dev |
| 🔄 Flux CD | `flux-system` | internal GitOps controller |

<details>
<summary><b>💤 Not currently running</b> (uninstalled to save RAM)</summary>

| Service | Namespace | Values |
|---|---|---|
| Grafana | `monitoring` | [`infra/monitoring/grafana/values.yaml`](infra/monitoring/grafana/values.yaml) |
| Prometheus | `monitoring` | [`infra/monitoring/prometheus/values.yaml`](infra/monitoring/prometheus/values.yaml) |

</details>

## 📁 Repository Layout

```
k3s-homelab/
├── ansible/                 # Automated Pi + k3s provisioning
│   ├── inventory.ini
│   ├── playbook.yml
│   └── roles/               # common, k3s-server, k3s-agent
├── infra/                   # Kubernetes manifests (see infra/README.md)
│   ├── apps/                # mosquitto, plant-watering-system-server
│   ├── database/            # PostgreSQL
│   ├── flux/                # GitRepository + Kustomization
│   └── monitoring/          # Grafana, Prometheus (values saved, not running)
└── docs/
    └── setup.md             # Complete manual setup guide
```

## ⚡ Quick Start

> **Prerequisites:** `kubectl` and `helm` installed locally, kubeconfig pointing at `https://<PI-IP>:6443`.

```powershell
kubectl get nodes          # node should be Ready
kubectl get pods -A        # all workloads across namespaces
kubectl top pods -A        # memory/CPU usage per pod
```

📖 Full manual walkthrough: [`docs/setup.md`](docs/setup.md) · Manifest details: [`infra/README.md`](infra/README.md)

## 🤖 Provisioning with Ansible

The playbook prepares the Pi and installs k3s in one run.

| Role | What it does |
|---|---|
| `common` | Updates and upgrades system packages, enables memory cgroups, reboots if needed |
| `k3s-server` | Installs k3s (pinned version) and makes the kubeconfig readable |
| `k3s-agent` | Joins worker nodes (none configured yet) |

<details>
<summary><b>1. Prerequisites</b></summary>

- Ansible on your local machine (`pip install ansible`)
- Pi reachable on the network with SSH enabled
- SSH key auth recommended (otherwise you're asked for a password on every run):

```bash
ssh-copy-id <your-user>@<PI-IP>
```

</details>

<details>
<summary><b>2. Configure the inventory</b></summary>

Edit `ansible/inventory.ini`:

```ini
[k3s_server]
raspberry4b ansible_host=<PI-IP> ansible_user=<your-user>
```

| Key | Meaning |
|---|---|
| `ansible_host` | IP address of the Pi |
| `ansible_user` | Username on the Pi (e.g. `ubuntu` on a fresh Ubuntu Server image) |

</details>

<details>
<summary><b>3. Run the playbook</b></summary>

```bash
cd ansible

# With SSH key (recommended)
ansible-playbook -i inventory.ini playbook.yml --ask-become-pass

# With password auth
ansible-playbook -i inventory.ini playbook.yml --ask-pass --ask-become-pass
```

| Flag | Why |
|---|---|
| `--ask-become-pass` | sudo password on the Pi (the playbook uses `become: true`) |
| `--ask-pass` | SSH password — only needed without an SSH key |

</details>
