# Raspberry Pi + k3s Setup

This guide walks through the complete setup of the Raspberry Pi, k3s, and the monitoring stack (Prometheus + Grafana).

## Prerequisites

- Raspberry Pi 4B (Ubuntu Server 64-bit)
- MicroSD card (min. 16GB, Class 10 recommended)
- Router with OpenWrt
- `kubectl` and `helm` installed on the local machine

---

## 1. Flash Raspberry Pi OS

1. Open Raspberry Pi Imager
2. **Choose OS:** Ubuntu Server (64-bit) — no desktop, no unnecessary overhead
3. **Choose SD card**
4. Enable SSH and create a user
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

## 4. Prepare the System

On the Pi:

```bash
sudo apt update && sudo apt upgrade -y
```

Enable cgroups for k3s:
```bash
sudo sed -i '$ s/$/ cgroup_enable=memory cgroup_memory=1/' /boot/firmware/cmdline.txt
```

Verify:
```bash
cat /boot/firmware/cmdline.txt
# Must contain at the end: cgroup_enable=memory cgroup_memory=1
```

Reboot — required, otherwise k3s won't start:
```bash
sudo reboot
```

---

## 5. Install k3s

```bash
curl -fL https://get.k3s.io | sudo sh -
```

Verify installation:
```bash
sudo systemctl status k3s
sudo k3s kubectl get nodes
```

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

## 11. Install InfluxDB v2

InfluxDB is a time-series database — optimized for timestamped data (e.g. metrics, sensor values). Unlike Prometheus, which actively scrapes metrics from apps (pull model), apps write directly into InfluxDB (push model). For Spring Boot apps with Micrometer this is the more direct approach.

**Add Helm repo:**
```powershell
helm repo add influxdata https://helm.influxdata.com/
helm repo update
```

**Install InfluxDB:**
```powershell
helm install influxdb influxdata/influxdb2 `
  --namespace monitoring `
  --values infra/influxdb/values.yaml `
  --set adminUser.password="YOUR-SECURE-PASSWORD" `
  --set adminUser.token="YOUR-SECURE-TOKEN-AT-LEAST-32-CHARS"
```

Note: Password and token are not stored in `values.yaml`. The file is safe to commit.

**Check status:**
```powershell
kubectl get pods -n monitoring
kubectl get svc -n monitoring
```

**Open InfluxDB UI:** http://<PI-IP>:32086  
Login: `admin` / your password

---

## 12. Create API Token in InfluxDB

1. Log in to InfluxDB (http://<PI-IP>:32086)
2. **Load Data** → **API Tokens** → **+ Generate API Token** → **All Access API Token**
3. Description: `grafana-springboot`
4. Copy the token and store it securely — it is only shown once

---

## 13. Connect Grafana to InfluxDB

1. In Grafana: **Connections** → **Data Sources** → **+ Add new data source** → **InfluxDB**
2. Configuration:

| Field | Value |
|---|---|
| Query Language | Flux |
| URL | `http://influxdb-influxdb2.monitoring.svc.cluster.local:8086` |
| Organization | `homelab` |
| Token | API token from step above |
| Default Bucket | `springboot` |

3. **Save & Test** → expect a green banner

---

## 14. Connect Spring Boot App to InfluxDB (Micrometer)

Add the following dependency to `pom.xml` in the Spring Boot app:

```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-influx</artifactId>
</dependency>
```

In `application.properties`:

```properties
management.influx.metrics.export.uri=http://<PI-IP>:32086
management.influx.metrics.export.org=homelab
management.influx.metrics.export.bucket=springboot
management.influx.metrics.export.token=YOUR-API-TOKEN
management.influx.metrics.export.enabled=true
```

After starting the app, metrics appear automatically in the `springboot` bucket. In Grafana → Explore → InfluxDB data source → query with Flux:

```flux
from(bucket: "springboot")
  |> range(start: -1h)
  |> filter(fn: (r) => r._measurement == "jvm_memory_used_bytes")
```

---

## 15. Run Ansible Playbook (automated cluster setup)

The Ansible playbook automates steps 2–6 of this guide. Prerequisites: Pi is flashed, SSH is reachable, static IP is set.

**Install Ansible (WSL or Ubuntu laptop):**
```bash
sudo apt update
sudo apt install ansible -y
ansible --version
```

**Copy SSH key to Pi (one-time):**
```bash
ssh-copy-id <your-user>@<PI-IP>
```
After this, no password prompt on SSH login.

**Run playbook:**
```bash
ansible-playbook -i ansible/inventory.ini ansible/playbook.yml --ask-become-pass
```
`--ask-become-pass` asks once for the Pi user's sudo password.

**Syntax check (without executing):**
```bash
ansible-playbook -i ansible/inventory.ini ansible/playbook.yml --syntax-check
```

**Dry run (shows what would happen without making changes):**
```bash
ansible-playbook -i ansible/inventory.ini ansible/playbook.yml --check --ask-become-pass
```

**Add a worker node:**
1. Add a line under `[k3s_agent]` in `ansible/inventory.ini`
2. Run the playbook again — the `common` and `k3s-agent` roles run on the new node
3. Verify: `kubectl get nodes`
