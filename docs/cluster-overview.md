# Woodhouse Cluster Overview

Kubernetes cluster managed via [Flux](https://fluxcd.io/) GitOps from this repository.

## Repo Structure

```
woodhouse-infra/
├── clusters/woodhouse/       # Flux bootstrap & top-level Kustomizations
│   ├── flux-system/           #   Flux controllers, GitRepository source
│   ├── infra.yaml             #   Kustomization → ./infra
│   └── apps.yaml              #   Kustomization → ./apps (depends on infra)
├── infra/                     # Cluster infrastructure components
│   ├── storage/               #   Local Path Provisioner
│   └── ingress/               #   Traefik (HelmRelease)
├── apps/                      # Application workloads
└── docs/                      # You are here
```

Flux syncs `clusters/woodhouse/` from the `main` branch of `mwhite7112/woodhouse-infra`. Infrastructure reconciles before apps via a dependency chain.

## Cluster Plan

Status key: **Deployed** | *Planned* | *Exploring*

### Core

| Component | Status | Notes |
|-----------|--------|-------|
| Secrets management | *Planned* | TBD — SOPS, Sealed Secrets, or external provider |
| Cluster RBAC | *Planned* | Beyond what individual components define |

### Networking

| Component | Status | Notes |
|-----------|--------|-------|
| Traefik | **Deployed** | Ingress controller — see [details below](#traefik) |
| Cert-manager | *Planned* | Automated SSL/TLS certificate provisioning |
| Cilium | *Planned* | eBPF-based CNI for container networking |
| Cloudflare Tunnel | *Planned* | Expose services to the internet without open ports |

### Storage

| Component | Status | Notes |
|-----------|--------|-------|
| Local Path Provisioner | **Deployed** | Current default StorageClass — to be replaced by Longhorn |
| Longhorn | *Planned* | Rancher distributed storage — will replace Local Path Provisioner as default StorageClass |
| Central database | *Exploring* | Likely a shared PostgreSQL instance |
| Object storage | *Exploring* | Likely MinIO for S3-compatible blob storage |

### Observability

| Component | Status | Notes |
|-----------|--------|-------|
| VictoriaMetrics | *Planned* | Metrics collection — chosen over Prometheus for lower memory footprint |
| Grafana | *Exploring* | Dashboards |
| VictoriaLogs | *Planned* | Log aggregation — chosen over Loki for lower memory footprint |

### App Development Services

| Component | Status | Notes |
|-----------|--------|-------|
| Redis | *Planned* | Caching / session store |
| Message queue | *Exploring* | TBD — NATS, RabbitMQ, etc. |

### Misc

| Component | Status | Notes |
|-----------|--------|-------|
| Service dashboard / homepage | *Exploring* | e.g. Homepage, Heimdall, Homer |

---

## Deployed Components

### Traefik

- **Namespace:** `traefik` (pod-security: `privileged`)
- **Install method:** HelmRelease (chart v34.x from `https://traefik.github.io/charts`)
- **Deployment mode:** DaemonSet — runs on every node for distributed ingress
- **Service type:** ClusterIP with `hostPort` on 80 (HTTP) and 443 (HTTPS)
- **Dashboard:** Enabled at `http://traefik.local` on the web entrypoint

Key files:
- `infra/ingress/helmrelease.yaml`
- `infra/ingress/helmrepository.yaml`
- `infra/ingress/namespace.yaml`

Design notes: Using a DaemonSet with hostPorts instead of a LoadBalancer service. This is well-suited for bare-metal / on-prem clusters where there is no cloud load balancer. Every node directly handles ingress traffic on ports 80 and 443.

### Local Path Provisioner

- **Namespace:** `local-path-storage`
- **Version:** v0.0.34 (Rancher)
- **StorageClass:** `local-path` (cluster default)
- **Volume binding:** `WaitForFirstConsumer` — volumes bind to the node where the pod is scheduled
- **Reclaim policy:** Delete
- **Host path:** `/var/local-path-provisioner`

Key files:
- `infra/storage/local-path-provisioner.yaml`
- `infra/storage/configmap.yaml`

Design notes: Simple node-local provisioner suitable for single-node or dev clusters. No replication — data lives on one node. Will be replaced by Longhorn for replicated, distributed storage.

## Flux Configuration

- **Flux version:** v2.7.5
- **Git source:** `main` branch, synced every 1 minute
- **Infrastructure sync:** every 30 minutes, pruning enabled
- **Apps sync:** every 30 minutes, depends on `infra` Kustomization
- **Controllers:** source, kustomize, helm, notification, image-reflector, image-automation
- **Helm sources:** Traefik charts repo (1h refresh)
