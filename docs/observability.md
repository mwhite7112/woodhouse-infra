# Observability

Metrics collection, log aggregation, and dashboards for the Woodhouse cluster.

## Components

| Component | Chart | Namespace | Description |
|-----------|-------|-----------|-------------|
| VictoriaMetrics (VMSingle) | victoria-metrics-k8s-stack | monitoring | Time-series metrics database (30-day retention, 20Gi Longhorn PVC) |
| VMAgent | victoria-metrics-k8s-stack | monitoring | Scrapes metrics from all cluster targets (30s interval) |
| VM Operator | victoria-metrics-k8s-stack | monitoring | Manages VM CRDs (VMSingle, VMAgent, scrape configs) |
| Grafana | victoria-metrics-k8s-stack | monitoring | Dashboards at `grafana.woodlab.work` |
| node-exporter | victoria-metrics-k8s-stack | monitoring | DaemonSet — host-level metrics (CPU, memory, disk, network) |
| kube-state-metrics | victoria-metrics-k8s-stack | monitoring | Kubernetes object metrics (pods, deployments, nodes) |
| VictoriaLogs | victoria-logs-single | monitoring | Log aggregation database (7-day retention, 10Gi Longhorn PVC) |
| Vector | victoria-logs-single | monitoring | DaemonSet — collects container logs from all nodes, ships to VictoriaLogs |

## Architecture

```
                  ┌──────────────┐
                  │   Grafana    │──── grafana.woodlab.work
                  └──────┬───────┘
                   ┌─────┴─────┐
            ┌──────┴──┐   ┌────┴────────┐
            │VMSingle │   │VictoriaLogs │
            │(metrics)│   │  (logs)     │
            └────┬────┘   └──────┬──────┘
                 │               │
            ┌────┴────┐   ┌──────┴──────┐
            │ VMAgent │   │   Vector    │
            │(scraper)│   │ (DaemonSet) │
            └────┬────┘   └──────┬──────┘
                 │               │
         ┌───────┴───────┐  ┌───┴────────┐
         │ /metrics      │  │ /var/log/  │
         │ endpoints     │  │ pods/      │
         └───────────────┘  └────────────┘
```

## Accessing Grafana

- **URL:** https://grafana.woodlab.work
- **Credentials:** admin user + password from `grafana-admin` Secret in the `monitoring` namespace

To retrieve the current admin password:

```bash
kubectl get secret grafana-admin -n monitoring -o jsonpath='{.data.admin-password}' | base64 -d
```

## Pre-built Dashboards

The victoria-metrics-k8s-stack chart includes dashboards for:

- Cluster overview (nodes, pods, resource usage)
- Node metrics (CPU, memory, disk, network per node)
- Pod/container metrics
- Kubernetes API server
- etcd
- CoreDNS

## Querying Logs

VictoriaLogs is pre-configured as a Grafana datasource. In Grafana:

1. Go to **Explore**
2. Select the **VictoriaLogs** datasource
3. Use [LogsQL](https://docs.victoriametrics.com/victorialogs/logsql/) to query logs

Example queries:

```
# All logs from a specific namespace
_namespace:"monitoring"

# Logs from a specific pod
_pod:"victoria-metrics-k8s-stack-vmsingle-0"

# Error logs across the cluster
error OR ERROR
```

## File Layout

```
infra/victoria-metrics/
├── namespace.yaml        # monitoring namespace (privileged — node-exporter needs host access)
├── helmrepository.yaml   # VictoriaMetrics Helm chart repo (shared by both charts)
├── grafana-secret.yaml   # SOPS-encrypted Grafana admin credentials
├── helmrelease.yaml      # victoria-metrics-k8s-stack HelmRelease
└── kustomization.yaml

infra/victoria-logs/
├── helmrelease.yaml      # victoria-logs-single HelmRelease (with Vector)
└── kustomization.yaml
```

## Notes

- **Vector on Talos:** Talos mounts container logs at `/var/log/pods/` (the standard path). Vector's `kubernetes_logs` source uses this path. Talos does not create `/var/log/containers/` symlinks — this should not matter for Vector but is worth noting if troubleshooting.
- **Shared HelmRepository:** Both charts reference the same `victoriametrics` HelmRepository in the `monitoring` namespace. The victoria-logs kustomization does not re-declare the namespace or HelmRepository — these come from the victoria-metrics kustomization.
- **Grafana plugins:** The VictoriaLogs datasource plugin (`victoriametrics-logs-datasource`) downloads from grafana.com at Grafana pod startup. Outbound connectivity is provided via the Cloudflare Tunnel.

## Verification

```bash
# Check HelmReleases
flux get helmreleases -n monitoring

# Check all pods are running
kubectl get pods -n monitoring

# Check VictoriaMetrics is receiving metrics
kubectl port-forward -n monitoring svc/victoria-metrics-k8s-stack-vmsingle 8429:8429
# Then visit http://localhost:8429/vmui

# Check VictoriaLogs is receiving logs
kubectl port-forward -n monitoring svc/victoria-logs-server 9428:9428
# Then visit http://localhost:9428/select/vmui
```
