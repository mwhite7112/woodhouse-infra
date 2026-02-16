# Exposing Services on woodlab.work

How to make an application accessible on the internet via `woodlab.work`.

## How it works

Traffic flows through three layers:

```
User → Cloudflare Edge (TLS termination) → Cloudflare Tunnel → cloudflared pod → Traefik → Ingress → Your app
```

- **Cloudflare** handles DNS, TLS certificates, and DDoS protection automatically — you don't need to manage certs for public-facing services.
- **Cloudflare Tunnel** runs inside the cluster and creates an outbound-only connection to Cloudflare's edge. No ports are opened on the firewall/router.
- **Traefik** is the cluster's ingress controller. It reads Kubernetes Ingress resources and routes traffic to the right Service.

## Exposing a new service

You need two things: a **DNS record** and a **Kubernetes Ingress**.

### 1. Create the DNS record

A wildcard CNAME (`*.woodlab.work`) is already configured, so any subdomain automatically routes through the tunnel. If you need a record for the apex domain or a specific subdomain that isn't covered, ask a cluster admin to run:

```bash
cloudflared tunnel route dns woodhouse myapp.woodlab.work
```

### 2. Create a Kubernetes Ingress

Add an Ingress resource in your app's namespace. The `host` field must match the DNS record.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app
  namespace: my-app
spec:
  rules:
    - host: my-app.woodlab.work
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: my-app
                port:
                  number: 80
```

That's it. Once Flux syncs the Ingress, your app is live at `https://my-app.woodlab.work`.

## Full example

The `test-app` in `apps/test-app/` is a working reference. It has:

| File | Purpose |
|------|---------|
| `namespace.yaml` | Creates the `test-app` namespace |
| `deployment.yaml` | Deploys the container with a health check |
| `service.yaml` | Exposes the container on port 80 inside the cluster |
| `ingress.yaml` | Routes `test-app.woodlab.work` to the Service |
| `kustomization.yaml` | Lists all resources for Flux |

To deploy a new app, copy this structure under `apps/<your-app>/`, update the names and image, and add the directory to `apps/kustomization.yaml`.

## Multiple paths / routes

You can serve multiple paths from a single Ingress:

```yaml
spec:
  rules:
    - host: my-app.woodlab.work
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 80
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend-service
                port:
                  number: 80
```

## TLS / HTTPS

Cloudflare terminates TLS at the edge and forwards traffic through the encrypted tunnel to the cluster. Your Ingress resources do **not** need a `tls` block or cert-manager annotations for services exposed through the tunnel — HTTPS just works.

Cert-manager and the `self-signed` ClusterIssuer are available if you need TLS for cluster-internal traffic between services.

## Checklist

- [ ] DNS record exists (wildcard covers most cases)
- [ ] App has a Deployment and Service in its namespace
- [ ] Ingress `host` matches the DNS record (e.g. `my-app.woodlab.work`)
- [ ] Ingress backend points to the correct Service name and port
- [ ] App directory is listed in `apps/kustomization.yaml`
- [ ] Commit and push to `main` — Flux handles the rest
