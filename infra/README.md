# 🧱 infra/ — Deployment Runbook

> **Purpose:** how to get everything in this directory onto a cluster — by hand, from scratch, in the right order.
> What runs and why is described in the [root README](../README.md); this file is the *operations* side: who manages each manifest, which secrets must exist first, and how to rebuild the cluster.

---

## 🧭 Who Manages What

Not every manifest is applied the same way. This matters: **never `kubectl apply` or `kubectl edit` a Flux-managed resource by hand** — Flux owns it, and manual changes either get overwritten or block Flux's next apply.

| Component | Path | Managed by | Namespace | Port |
|---|---|---|---|---|
| 🌱 Plant Watering Server | [`apps/plant-watering-system-server/`](apps/plant-watering-system-server/) | **Flux** (auto-deployed from Git) | `apps` | NodePort `30080` |
| 📡 Mosquitto | [`apps/mosquitto/mosquitto.yaml`](apps/mosquitto/mosquitto.yaml) | `kubectl apply` | `apps` | NodePort `31883` |
| 🐘 PostgreSQL 16 | [`database/postgres/postgres.yaml`](database/postgres/postgres.yaml) | `kubectl apply` | `database` | NodePort `32432` |
| 🔄 Flux controllers | [`flux/flux-system/`](flux/flux-system/) | `kubectl apply -k` once (Flux v2.8.8, source + kustomize only, no leader election) | `flux-system` | — |
| 🔄 Flux source + sync | [`flux/k3s-homelab-source.yaml`](flux/k3s-homelab-source.yaml), [`flux/plant-watering-server-kustomization.yaml`](flux/plant-watering-server-kustomization.yaml) | `kubectl apply` once, then Flux | `flux-system` | — |
| 📊 Grafana *(not running)* | [`monitoring/grafana/values.yaml`](monitoring/grafana/values.yaml) | Helm | `monitoring` | — |
| 📈 Prometheus *(not running)* | [`monitoring/prometheus/values.yaml`](monitoring/prometheus/values.yaml) | Helm | `monitoring` | — |

## 🔐 Required Secrets

Secrets are **not stored in Git**. They must exist in the cluster before the workloads that use them start.

| Secret | Namespace | Keys | Used by |
|---|---|---|---|
| `postgres-secret` | `database` | `password` | PostgreSQL |
| `mosquitto-passwd` | `apps` | `mosquitto_passwd` (hashed password file) | Mosquitto |
| `server-secret` | `apps` | `DB_USERNAME`, `DB_PASSWORD`, `JWT_SECRET`, `APP_PASSWORD`, `MQTT_USERNAME`, `MQTT_PASSWORD` | Plant Watering Server |
| `ghcr-pull-secret` | `apps` | Docker registry credentials | Pulling the server image from GHCR |

> [!IMPORTANT]
> Values must match across secrets: `server-secret.DB_PASSWORD` = `postgres-secret.password`, and `MQTT_USERNAME`/`MQTT_PASSWORD` must be a user in the Mosquitto password file. The PostgreSQL user is `plant` (set in `postgres.yaml`).

---

## 🛠️ Rebuild From Scratch

Run from the repo root once k3s is up and `kubectl get nodes` shows `Ready`.

### 1. Namespaces

```powershell
kubectl create namespace database
kubectl create namespace apps
```

### 2. Secrets

```powershell
# PostgreSQL
kubectl create secret generic postgres-secret -n database `
  --from-literal=password=<db-password>

# Mosquitto password file (generated with the broker image)
docker run --rm -v ${PWD}:/work eclipse-mosquitto:2 mosquitto_passwd -b -c /work/mosquitto_passwd plant <mqtt-password>
kubectl create secret generic mosquitto-passwd -n apps --from-file=mosquitto_passwd
Remove-Item mosquitto_passwd

# Server
kubectl create secret generic server-secret -n apps `
  --from-literal=DB_USERNAME=plant `
  --from-literal=DB_PASSWORD=<db-password> `
  --from-literal=JWT_SECRET=<random, at least 32 characters> `
  --from-literal=APP_PASSWORD=<app-login-password> `
  --from-literal=MQTT_USERNAME=plant `
  --from-literal=MQTT_PASSWORD=<mqtt-password>

# GHCR image pull (GitHub PAT with read:packages)
kubectl create secret docker-registry ghcr-pull-secret -n apps `
  --docker-server=ghcr.io `
  --docker-username=<github-user> `
  --docker-password=<github-pat>
```

### 3. Data layer & broker

```powershell
kubectl apply -f infra/database/postgres/postgres.yaml
kubectl apply -f infra/apps/mosquitto/mosquitto.yaml

kubectl get pods -n database   # wait until Running
kubectl get pods -n apps
```

### 3b. Restore the database (when migrating)

Restore **before** Flux deploys the server — otherwise Flyway creates an empty schema and MQTT writes race the restore.

```bash
P=$(kubectl get pod -n database -l app=postgres -o jsonpath='{.items[0].metadata.name}')
MSYS_NO_PATHCONV=1 kubectl cp plantdb.dump database/$P:/tmp/plantdb.dump
MSYS_NO_PATHCONV=1 kubectl exec -n database $P -- pg_restore -U plant -d plantdb --clean --if-exists --no-owner --exit-on-error /tmp/plantdb.dump
```

### 4. Flux → server deploys itself

```powershell
kubectl apply -k infra/flux/flux-system                 # Flux controllers (pinned version, trimmed)
kubectl -n flux-system rollout status deployment/source-controller
kubectl -n flux-system rollout status deployment/kustomize-controller
kubectl apply -f infra/flux/k3s-homelab-source.yaml     # tells Flux which repo to watch
kubectl apply -f infra/flux/plant-watering-server-kustomization.yaml
```

Flux now applies `apps/plant-watering-system-server/` on its own. Verify:

```powershell
kubectl get kustomization -n flux-system              # READY should be True
kubectl rollout status deployment/plant-watering-server -n apps
```

> [!NOTE]
> The server needs roughly 30–90 seconds until its readiness probe turns green. On a fresh database Flyway creates the schema on first start; after a restore (step 3b) Flyway only reports `Schema "public" is up to date`.

<details>
<summary><b>📊 Optional: monitoring stack (Helm)</b></summary>

Not running by default — too heavy for a 2 GB Pi.

```powershell
helm repo add grafana https://grafana.github.io/helm-charts
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm upgrade --install grafana grafana/grafana -n monitoring --create-namespace -f infra/monitoring/grafana/values.yaml
helm upgrade --install prometheus prometheus-community/prometheus -n monitoring -f infra/monitoring/prometheus/values.yaml
```

</details>

---

## 💾 Manual Database Backup

Run from Git Bash; store the file outside the repo.

```bash
P=$(kubectl get pod -n database -l app=postgres -o jsonpath='{.items[0].metadata.name}')
MSYS_NO_PATHCONV=1 kubectl exec -n database $P -- pg_dump -U plant -Fc -f /tmp/plantdb.dump plantdb
MSYS_NO_PATHCONV=1 kubectl cp database/$P:/tmp/plantdb.dump plantdb-$(date +%F).dump
```

Secrets are not in Git — keep their values in a password manager.

## ➕ Adding Another App via Flux

1. Create `apps/<app-name>/` with `deployment.yaml` + `service.yaml` (namespace `apps`, pick a free NodePort).
2. Create any secrets it needs by hand (see above).
3. Add `flux/<app-name>-kustomization.yaml` — copy `plant-watering-server-kustomization.yaml` and change `metadata.name` and `spec.path`.
4. `kubectl apply -f infra/flux/<app-name>-kustomization.yaml` once — from then on, Git is the source of truth.

## 🔍 Troubleshooting

| Symptom | Check |
|---|---|
| New image not rolling out | `kubectl get kustomization -n flux-system` — a `READY False` message explains why Flux can't apply |
| Pod stuck in `ImagePullBackOff` | `ghcr-pull-secret` missing or PAT expired |
| Pod in `CreateContainerConfigError` | A secret or secret key referenced in the manifest doesn't exist |
| Server can't reach the DB/broker | `kubectl get pods -A`, then `kubectl logs deployment/plant-watering-server -n apps` |
