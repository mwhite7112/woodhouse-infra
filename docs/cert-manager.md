# Cert-manager

Automated TLS certificate provisioning.

- **Namespace:** `cert-manager`
- **Install method:** HelmRelease (chart v1.17.x from `https://charts.jetstack.io`)
- **CRDs:** Installed and kept by Helm (`crds.enabled: true`, `crds.keep: true`)
- **ClusterIssuers:** `self-signed` (deployed via separate `cert-issuers` Kustomization that depends on `infra`)

## Key files

- `infra/cert-manager/helmrelease.yaml`
- `infra/cert-manager/helmrepository.yaml`
- `infra/cert-manager/namespace.yaml`
- `infra/cert-issuers/self-signed-issuer.yaml`

## Design notes

ClusterIssuers are in a separate Kustomization (`cert-issuers`) because they depend on cert-manager CRDs existing first. The `cert-issuers` Kustomization depends on `infra`, which ensures cert-manager is installed before issuers are applied.

For services exposed via [Cloudflare Tunnel](cloudflare-tunnel.md), TLS is handled at Cloudflare's edge — cert-manager is not needed for those. The `self-signed` issuer is available for cluster-internal TLS between services.
