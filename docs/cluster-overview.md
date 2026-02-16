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
│   └── traefik/               #   Traefik ingress controller (HelmRelease)
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
| Traefik | **Deployed** | Ingress controller — see [details below](#traefik) |
| Cert-manager | **Deployed** | Automated SSL/TLS certificate provisioning — see [details below](#cert-manager) |
| Cilium | *Planned* | eBPF-based CNI for container networking |
| Cloudflare Tunnel | **Deployed** | Outbound-only tunnel to Cloudflare edge — see [details below](#cloudflare-tunnel) |

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
- **Dashboard:** Disabled

Key files:
- `infra/traefik/helmrelease.yaml`
- `infra/traefik/helmrepository.yaml`
- `infra/traefik/namespace.yaml`

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

### Cert-manager

- **Namespace:** `cert-manager`
- **Install method:** HelmRelease (chart v1.17.x from `https://charts.jetstack.io`)
- **CRDs:** Installed and kept by Helm (`crds.enabled: true`, `crds.keep: true`)
- **ClusterIssuers:** `self-signed` (deployed via separate `cert-issuers` Kustomization that depends on `infra`)

Key files:
- `infra/cert-manager/helmrelease.yaml`
- `infra/cert-manager/helmrepository.yaml`
- `infra/cert-manager/namespace.yaml`
- `infra/cert-issuers/self-signed-issuer.yaml`

Design notes: ClusterIssuers are in a separate Kustomization (`cert-issuers`) because they depend on cert-manager CRDs existing first. The `cert-issuers` Kustomization depends on `infra`, which ensures cert-manager is installed before issuers are applied.

### Cloudflare Tunnel

- **Namespace:** `cloudflare-tunnel` (pod-security: `baseline`)
- **Install method:** Raw manifests (Deployment + ConfigMap + Secret)
- **Replicas:** 2
- **Image:** `cloudflare/cloudflared:2025.2.1`
- **Backend:** `http://traefik.traefik.svc.cluster.local:80` (catch-all)
- **Metrics:** Exposed on port 2000 (`/ready` endpoint)

Key files:
- `infra/cloudflare-tunnel/deployment.yaml`
- `infra/cloudflare-tunnel/tunnel-config.yaml`
- `infra/cloudflare-tunnel/tunnel-credentials.yaml` (SOPS-encrypted)
- `infra/cloudflare-tunnel/namespace.yaml`

Design notes: Cloudflare Tunnel creates outbound-only connections from the cluster to Cloudflare's edge network — no inbound ports, firewall rules, or router changes needed. All traffic flows through the tunnel to Traefik, which handles per-service routing via IngressRoutes. Adding a new service only requires a Traefik IngressRoute and a DNS CNAME; no cloudflared config changes are needed.

Traffic flow: `User → Cloudflare Edge → Tunnel → cloudflared pod → Traefik → IngressRoute → App`

### SOPS + age

- **Encryption:** SOPS with age keypair
- **Decryption Secret:** `sops-age` in `flux-system` namespace
- **Scope:** All three Kustomizations (`infra`, `apps`, `cert-issuers`) have SOPS decryption configured
- **Encrypted fields:** Only `data` and `stringData` on Secrets (via `encrypted_regex` in `.sops.yaml`)

Key files:
- `.sops.yaml`

Design notes: Secrets are encrypted locally with `sops --encrypt --in-place` before committing. Flux decrypts at apply time using the age private key stored in the `sops-age` cluster Secret. See [managing-secrets.md](managing-secrets.md) for the workflow.

## Flux Configuration

- **Flux version:** v2.7.5
- **Git source:** `main` branch, synced every 1 minute
- **Infrastructure sync:** every 30 minutes, pruning enabled
- **Apps sync:** every 30 minutes, depends on `infra` and `cert-issuers` Kustomizations
- **Cert-issuers sync:** every 30 minutes, depends on `infra` Kustomization
- **SOPS decryption:** enabled on all Kustomizations via `sops-age` Secret
- **Controllers:** source, kustomize, helm, notification, image-reflector, image-automation
- **Helm sources:** Traefik charts repo (1h refresh), Longhorn charts repo (1h refresh), cert-manager charts repo (1h refresh)
