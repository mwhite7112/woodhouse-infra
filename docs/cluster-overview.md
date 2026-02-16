# Woodhouse Cluster Overview

Kubernetes cluster managed via [Flux](https://fluxcd.io/) GitOps from this repository.

## Repo Structure

```
woodhouse-infra/
├── clusters/woodhouse/       # Flux bootstrap & top-level Kustomizations
│   ├── flux-system/           #   Flux controllers, GitRepository source
│   ├── infra.yaml             #   Kustomization → ./infra
│   ├── cert-issuers.yaml      #   Kustomization → ./infra/cert-issuers (depends on infra)
│   └── apps.yaml              #   Kustomization → ./apps (depends on infra, cert-issuers)
├── infra/                     # Cluster infrastructure components
│   ├── cert-manager/          #   cert-manager (HelmRelease)
│   ├── cert-issuers/          #   ClusterIssuers (depends on cert-manager CRDs)
│   ├── cloudflare-tunnel/     #   Cloudflare Tunnel (raw manifests)
│   ├── longhorn/              #   Longhorn distributed storage (HelmRelease)
│   ├── rbac/                  #   Namespaces, ServiceAccounts, ClusterRoleBindings
│   ├── traefik/               #   Traefik ingress controller (HelmRelease)
│   ├── victoria-logs/         #   VictoriaLogs log aggregation (HelmRelease)
│   └── victoria-metrics/      #   VictoriaMetrics + Grafana observability stack (HelmRelease)
├── apps/                      # Application workloads
├── .githooks/                 # Git hooks (pre-commit: block unencrypted Secrets)
├── docs/                      # You are here
└── .sops.yaml                 # SOPS encryption config (age public key)
```

Flux syncs `clusters/woodhouse/` from the `main` branch of `mwhite7112/woodhouse-infra`. Infrastructure reconciles before apps via a dependency chain.

## Cluster Plan

Status key: **Deployed** | *Planned* | *Exploring*

### Core

| Component | Status | Notes |
|-----------|--------|-------|
| Secrets management | **Deployed** | SOPS + age — see [managing-secrets.md](managing-secrets.md) |
| Cluster RBAC | **Deployed** | ServiceAccount + ClusterRoleBinding in `infra/rbac/` |

### Networking

| Component | Status | Notes |
|-----------|--------|-------|
| Traefik | **Deployed** | Ingress controller — see [traefik.md](traefik.md) |
| Cert-manager | **Deployed** | Automated SSL/TLS certificate provisioning — see [cert-manager.md](cert-manager.md) |
| Cilium | *Planned* | eBPF-based CNI for container networking |
| Cloudflare Tunnel | **Deployed** | Outbound-only tunnel to Cloudflare edge — see [cloudflare-tunnel.md](cloudflare-tunnel.md) |

### Storage

| Component | Status | Notes |
|-----------|--------|-------|
| Longhorn | **Deployed** | Distributed replicated storage — see [longhorn.md](longhorn.md) |
| Central database | *Exploring* | Likely a shared PostgreSQL instance |
| Object storage | *Exploring* | Likely MinIO for S3-compatible blob storage |

### Observability

| Component | Status | Notes |
|-----------|--------|-------|
| VictoriaMetrics | **Deployed** | Metrics collection (VMSingle + VMAgent) — see [observability.md](observability.md) |
| Grafana | **Deployed** | Dashboards at `grafana.woodlab.work` — see [observability.md](observability.md) |
| VictoriaLogs | **Deployed** | Log aggregation with Vector shipper — see [observability.md](observability.md) |

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

## Component Docs

| Doc | Description |
|-----|-------------|
| [Traefik](traefik.md) | Ingress controller — DaemonSet with hostPorts |
| [Cloudflare Tunnel](cloudflare-tunnel.md) | Outbound-only tunnel to Cloudflare edge for `woodlab.work` |
| [Longhorn](longhorn.md) | Distributed replicated block storage |
| [Cert-manager](cert-manager.md) | Automated TLS certificate provisioning |
| [Observability](observability.md) | VictoriaMetrics, VictoriaLogs, Grafana |
| [Managing Secrets](managing-secrets.md) | SOPS + age encryption workflow |
| [Exposing Services](exposing-services.md) | How to connect an app to `woodlab.work` |
| [New User Access](new-user-access.md) | Creating cluster-admin kubeconfigs |

## Flux Configuration

- **Flux version:** v2.7.5
- **Git source:** `main` branch, synced every 1 minute
- **Infrastructure sync:** every 30 minutes, pruning enabled
- **Apps sync:** every 30 minutes, depends on `infra` and `cert-issuers` Kustomizations
- **Cert-issuers sync:** every 30 minutes, depends on `infra` Kustomization
- **SOPS decryption:** enabled on all Kustomizations via `sops-age` Secret
- **Controllers:** source, kustomize, helm, notification, image-reflector, image-automation
- **Helm sources:** Traefik charts repo (1h refresh), Longhorn charts repo (1h refresh), cert-manager charts repo (1h refresh), VictoriaMetrics charts repo (1h refresh)
