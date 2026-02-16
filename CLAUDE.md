# Woodhouse Infra

Flux-managed Kubernetes infrastructure repo for the Woodhouse cluster.

## Repo Structure

- `clusters/woodhouse/` — Flux bootstrap and top-level Kustomizations
  - `flux-system/` — Flux controllers and GitRepository source
  - `infra.yaml` — Kustomization pointing to `./infra`
  - `apps.yaml` — Kustomization pointing to `./apps` (depends on `infra`)
- `infra/` — Cluster infrastructure components (storage, ingress, etc.)
- `apps/` — Application workloads
- `docs/` — Cluster documentation and planning notes

## How It Works

Flux watches the `main` branch and syncs `clusters/woodhouse/`. The `infra` Kustomization must reconcile successfully before `apps` begins syncing. Each subdirectory under `infra/` and `apps/` has its own `kustomization.yaml` and is included by the parent.

## Adding Infrastructure Components

New infrastructure goes under `infra/<component-name>/` with a `kustomization.yaml`. Add the directory to `infra/kustomization.yaml` resources list. Helm-based components use a `HelmRepository` + `HelmRelease`; raw manifests are applied directly.

## Key Details

- Git source: `mwhite7112/woodhouse-infra`, branch `main`
- Cluster: 3 bare-metal machines running Talos Linux — 1 control plane (8GB), 2 workers (16GB each)
- Flux version: v2.7.5
- See `docs/cluster-overview.md` for the full cluster plan and component details
