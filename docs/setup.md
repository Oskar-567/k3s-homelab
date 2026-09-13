# Raspberry Pi + k3s Setup

This guide walks through the complete setup of the Raspberry Pi, k3s, and the monitoring stack (Prometheus + Grafana).

## Prerequisites

- Raspberry Pi 4B (Raspberry Pi OS Lite 64-bit)
- MicroSD card (min. 16GB; A2 / "Endurance" cards cope best with k3s' small synced writes)
- Router with OpenWrt
- `kubectl` and `helm` installed on the local machine

---

## 1. Flash Raspberry Pi OS

1. Open Raspberry Pi Imager
2. **Choose OS:** Raspberry Pi OS (other) → **Raspberry Pi OS Lite (64-bit)** — no desktop
3. **Choose SD card**
4. **Edit settings:** hostname, username + password, time zone; **Services → SSH → public-key authentication only** with your `id_ed25519.pub`; no Wi-Fi if the Pi uses Ethernet
5. Start flashing

---

## 2. First Boot & SSH Connection

```bash
ssh <your-user>@<PI-IP>
```

Check internet connectivity:
```bash
ping -c 3 8.8.8.8   # IP level
ping -c 3 google.com # DNS level
```

---

## 3. Set Static IP in Router (OpenWrt)

To make the Pi always reachable at the same address, set a DHCP reservation in the router.

**In OpenWrt:** Network → Interfaces → LAN → DHCP Server → Static Leases  
Assign a fixed IP to the Pi's MAC address

---

## 4. Prepare the System and Install k3s (Ansible)

System preparation (updates, cgroup kernel parameters, SD-card write reduction) and the k3s installation are automated — see [section 15](#15-run-ansible-playbook-automated-cluster-setup). Do not install k3s by hand: the playbook writes `/etc/rancher/k3s/config.yaml` **before** installing, and k3s only reads parts of it on first start.

---

## 7. Configure kubectl Locally (Windows)

**Install kubectl:**
```powershell
winget install Kubernetes.kubectl
```

**Fetch kubeconfig from Pi:**
```bash
# On the Pi:
sudo cat /etc/rancher/k3s/k3s.yaml
```

Copy the content, then on Windows:
```powershell
mkdir $HOME\.kube
notepad $HOME\.kube\config
```

Paste the content and replace the server address with your Pi's IP:
```yaml
# From:
https://127.0.0.1:6443
# To:
https://YOUR-PI-IP:6443
```

Test:
```powershell
kubectl get nodes
# NAME          STATUS   ROLES           AGE   VERSION
# raspberry4b   Ready    control-plane   ...   v1.35.4+k3s1
```

---

## 8. Install Helm (Windows)

```powershell
winget install Helm.Helm
helm version
```

Helm is the package manager for Kubernetes — like winget, but for cluster apps.

---

## 9. Install Monitoring Stack (Prometheus + Grafana)

**Add repositories:**
```powershell
helm repo add grafana https://grafana.github.io/helm-charts
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

**Install Grafana:**
```powershell
helm install grafana grafana/grafana `
  --namespace monitoring `
  --create-namespace `
  --set service.type=NodePort `
  --set service.nodePort=32000
```

**Install Prometheus:**
```powershell
helm install prometheus prometheus-community/prometheus --namespace monitoring
```

**Check status:**
```powershell
kubectl get pods -n monitoring
kubectl get svc -n monitoring
```

**Open Grafana:** http://<PI-IP>:32000

**Get admin password:**
```powershell
kubectl get secret -n monitoring grafana -o jsonpath="{.data.admin-password}" | ForEach-Object { [System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($_)) }
```

---

## 10. Connect Prometheus to Grafana

1. In Grafana: **Connections → Data Sources → Add data source**
2. Select **Prometheus**
3. Enter URL:
   ```
   http://prometheus-server.monitoring.svc.cluster.local
   ```
4. **Save & Test**

**Import Kubernetes dashboard:**
- Dashboards → Import → ID `315` → Load → Import

**Import Node Exporter Full dashboard (installed):**
- Dashboards → Import → ID `1860` → Load → Import
- Shows detailed Pi hardware metrics: CPU, RAM, disk, network

---

## 9a. Prometheus Optimizations for Raspberry Pi (2GB RAM)

The default Prometheus installation is not optimized for low-memory hardware. Apply these settings via `infra/prometheus/values.yaml` before or after installation.

**What gets optimized and why:**

| Setting | Value | Reason |
|---|---|---|
| `retention` | `12h` | Default is 15 days — drastically reduces RAM usage |
| `memory limit` | `200Mi` | Prevents Prometheus from consuming all available RAM |
| `scrape_interval` | `5m` | Default is 15s — reduces CPU and I/O load on the Pi |
| `livenessProbeInitialDelay` | `60s` | Pi is slow to start Prometheus — without this it gets killed in a restart loop |
| `alertmanager` | disabled | Not configured, no reason to run it |
| `pushgateway` | disabled | Only needed for batch jobs, not used here |

**Apply the optimizations:**

```powershell
helm upgrade prometheus prometheus-community/prometheus `
  --namespace monitoring `
  --values infra/prometheus/values.yaml
```

**Verify memory usage:**

```powershell
kubectl top pods -n monitoring
# prometheus-server should stay under 200Mi
```

On the Pi:
```bash
free -h
# "available" should be 350Mi+ and Swap should be 0
```

> **Note:** If you have two separate `server:` blocks in your `values.yaml`, the second one silently overrides the first. Always keep all `server:` settings in a single block.

---

## Troubleshooting

**k3s won't start:**
```bash
sudo systemctl status k3s
sudo journalctl -u k3s -f
```

**DNS not working (Pi not receiving DNS via DHCP):**
```bash
resolvectl status
# If no DNS is listed under eth0 → set OpenWrt DHCP option 6 (see step 4)
```

**kubectl can't connect:**
- Check IP in `~/.kube/config` (`<PI-IP>:6443`)
- Is k3s running on the Pi? `sudo systemctl status k3s`

**cgroups issue:**
```bash
cat /boot/firmware/cmdline.txt
# Must contain: cgroup_enable=memory cgroup_memory=1
```

---

## 15. Run Ansible Playbook (automated cluster setup)

The playbook runs the roles `common` → `os-tuning` → `k3s-server` (→ `k3s-agent` for workers). Prerequisites: Pi flashed (section 1), reachable via SSH with your key, DHCP reservation set (section 3).

Ansible does not run natively on Windows — use WSL. Installation (no sudo needed), inventory setup and the run command are described in the root [README](../README.md#-provisioning-with-ansible).

**Syntax check:**
```bash
~/.venvs/ansible/bin/ansible-playbook -i inventory.local.ini playbook.yml --syntax-check
```

**Verify after the run (on the Pi):**
```bash
swapon --show                      # only /dev/zram0
findmnt -no OPTIONS /              # contains noatime
cat /etc/rancher/k3s/config.yaml   # trimmed k3s config
sudo k3s kubectl get nodes         # Ready
```

**Add a worker node:**
1. Add a line under `[k3s_agent]` in `ansible/inventory.local.ini`
2. Run the playbook again — `common`, `os-tuning` and `k3s-agent` run on the new node
3. Verify: `kubectl get nodes`
