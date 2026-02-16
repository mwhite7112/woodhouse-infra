# Traefik

Ingress controller for the Woodhouse cluster.

- **Namespace:** `traefik` (pod-security: `privileged`)
- **Install method:** HelmRelease (chart v34.x from `https://traefik.github.io/charts`)
- **Deployment mode:** DaemonSet — runs on every node for distributed ingress
- **Service type:** ClusterIP with `hostPort` on 80 (HTTP) and 443 (HTTPS)
- **Dashboard:** Disabled

## Key files

- `infra/traefik/helmrelease.yaml`
- `infra/traefik/helmrepository.yaml`
- `infra/traefik/namespace.yaml`

## Design notes

Using a DaemonSet with hostPorts instead of a LoadBalancer service. This is well-suited for bare-metal / on-prem clusters where there is no cloud load balancer. Every node directly handles ingress traffic on ports 80 and 443.

Traefik reads standard Kubernetes Ingress resources — no Traefik-specific CRDs (IngressRoute) are required, though they are available if needed.

## Routing

External traffic reaches Traefik via the [Cloudflare Tunnel](cloudflare-tunnel.md). See [Exposing Services](exposing-services.md) for how to create Ingress resources that route to your app.
