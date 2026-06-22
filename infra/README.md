# infra/

Kubernetes manifests for all services running on the homelab cluster.

## Structure

```
infra/
├── database/
│   ├── postgres/
│   │   └── postgres.yaml             # PostgreSQL 16, namespace: database, NodePort 32432
│   └── influxdb/
│       └── values.yaml               # InfluxDB 2.7 via Helm, namespace: database, NodePort 32086
├── monitoring/
│   ├── grafana/
│   │   └── values.yaml               # Grafana via Helm (not running — uninstalled to save RAM)
│   └── prometheus/
│       └── values.yaml               # Prometheus via Helm (not running — uninstalled to save RAM)
├── apps/
│   ├── mosquitto/
│   │   └── mosquitto.yaml            # Eclipse Mosquitto 2 MQTT broker, namespace: apps, NodePort 31883
│   └── plant-watering-system-server/
│       ├── deployment.yaml           # Spring Boot server, namespace: apps, NodePort 30080
│       └── service.yaml              # NodePort service
└── flux/
    ├── k3s-homelab-source.yaml       # Flux GitRepository — watches this repo on GitHub
    └── plant-watering-server-kustomization.yaml  # Flux Kustomization — deploys infra/apps/plant-watering-system-server/
```

## Namespaces

| Namespace   | Purpose                                          |
|-------------|--------------------------------------------------|
| database    | PostgreSQL, InfluxDB                             |
| monitoring  | Grafana, Prometheus (manifests saved, not running) |
| apps        | Mosquitto, Plant Watering System Server          |
| flux-system | Flux CD GitOps controller                        |

## Deploying

Plain YAML manifests:
```powershell
kubectl apply -f infra/database/postgres/postgres.yaml
kubectl apply -f infra/apps/mosquitto/mosquitto.yaml
```

Helm charts:
```powershell
helm upgrade --install influxdb influxdata/influxdb2 -n database -f infra/database/influxdb/values.yaml
helm upgrade --install grafana grafana/grafana -n monitoring -f infra/monitoring/grafana/values.yaml
helm upgrade --install prometheus prometheus-community/prometheus -n monitoring -f infra/monitoring/prometheus/values.yaml
```

Flux resources (applied once manually — Flux then self-manages):
```powershell
kubectl apply -f infra/flux/k3s-homelab-source.yaml
kubectl apply -f infra/flux/plant-watering-server-kustomization.yaml
```

## CI/CD

The Plant Watering System Server is deployed automatically via GitOps:
1. Push to `server/` in the plant-watering-system repo triggers GitHub Actions
2. Pipeline builds an arm64 Docker image and pushes it to GHCR
3. Pipeline commits the new image SHA to `infra/apps/plant-watering-system-server/deployment.yaml`
4. Flux picks up the commit within ~1 minute and rolls out the new version
