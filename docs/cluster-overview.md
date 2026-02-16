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
│   ├── longhorn/              #   Longhorn distributed storage (HelmRelease)
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
| Longhorn | **Deployed** | Distributed replicated storage — default StorageClass |
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

### Longhorn

- **Namespace:** `longhorn-system` (pod-security: `privileged`)
- **Install method:** HelmRelease (chart v1.9.x from `https://charts.longhorn.io`)
- **StorageClass:** `longhorn` (cluster default)
- **Replica count:** 2
- **Data path:** `/var/lib/longhorn`
- **Disk selection:** Only nodes labeled `node.longhorn.io/create-default-disk=true` (worker nodes)

Key files:
- `infra/longhorn/helmrelease.yaml`
- `infra/longhorn/helmrepository.yaml`
- `infra/longhorn/namespace.yaml`

Design notes: Distributed block storage with 2-replica redundancy across worker nodes. Requires `iscsi-tools` and `util-linux-tools` Talos system extensions and a `/var/lib/longhorn` kubelet extra mount on all nodes. Replaces the previous Local Path Provisioner which had no replication.

## Flux Configuration

- **Flux version:** v2.7.5
- **Git source:** `main` branch, synced every 1 minute
- **Infrastructure sync:** every 30 minutes, pruning enabled
- **Apps sync:** every 30 minutes, depends on `infra` Kustomization
- **Controllers:** source, kustomize, helm, notification, image-reflector, image-automation
- **Helm sources:** Traefik charts repo (1h refresh), Longhorn charts repo (1h refresh)
