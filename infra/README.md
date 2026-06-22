# infra/

Kubernetes manifests for all services running on the homelab cluster.

## Structure

```
infra/
├── database/
│   ├── postgres/
│   │   └── postgres.yaml       # PostgreSQL 16, namespace: database, NodePort 32432
│   └── influxdb/
│       └── values.yaml         # InfluxDB 2.7 via Helm, namespace: database, NodePort 32086
├── monitoring/
│   ├── grafana/
│   │   └── values.yaml         # Grafana via Helm, namespace: monitoring, NodePort 32000
│   └── prometheus/
│       └── values.yaml         # Prometheus via Helm, namespace: monitoring, cluster-internal
└── apps/
    └── mosquitto/
        └── mosquitto.yaml      # Eclipse Mosquitto 2 MQTT broker, namespace: apps, NodePort 31883
```

## Namespaces

| Namespace  | Purpose                        |
|------------|--------------------------------|
| database   | PostgreSQL, InfluxDB           |
| monitoring | Grafana, Prometheus            |
| apps       | Application services (Mosquitto, Spring Boot, ...) |

## Deploying

Plain YAML manifests (database, apps):
```powershell
kubectl apply -f infra/database/postgres/postgres.yaml
kubectl apply -f infra/apps/mosquitto/mosquitto.yaml
```

Helm charts (monitoring, InfluxDB):
```powershell
helm upgrade --install grafana grafana/grafana -n monitoring -f infra/monitoring/grafana/values.yaml
```
