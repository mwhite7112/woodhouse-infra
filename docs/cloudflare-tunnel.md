# Cloudflare Tunnel

Outbound-only tunnel connecting the cluster to Cloudflare's edge network. Exposes services on `woodlab.work` without opening any inbound ports.

- **Namespace:** `cloudflare-tunnel` (pod-security: `baseline`)
- **Install method:** Raw manifests (Deployment + ConfigMap + Secret)
- **Replicas:** 2
- **Image:** `cloudflare/cloudflared:2025.2.1`
- **Backend:** `http://traefik.traefik.svc.cluster.local:80` (catch-all)
- **Metrics:** Exposed on port 2000 (`/ready` endpoint)
- **Domain:** `woodlab.work` (wildcard `*.woodlab.work` included)

## Key files

- `infra/cloudflare-tunnel/deployment.yaml`
- `infra/cloudflare-tunnel/tunnel-config.yaml`
- `infra/cloudflare-tunnel/tunnel-credentials.yaml` (SOPS-encrypted)
- `infra/cloudflare-tunnel/namespace.yaml`

## Traffic flow

```
User → Cloudflare Edge (TLS) → Tunnel → cloudflared pod → Traefik → Ingress → App
```

## Design notes

Cloudflare Tunnel creates outbound-only connections from the cluster to Cloudflare's edge network — no inbound ports, firewall rules, or router changes needed. All traffic flows through the tunnel to [Traefik](traefik.md), which handles per-service routing via Ingress resources.

Adding a new service only requires a Kubernetes Ingress and (if the wildcard doesn't cover it) a DNS CNAME. No cloudflared config changes are needed. See [Exposing Services](exposing-services.md) for the full walkthrough.

## Adding a DNS record

The wildcard `*.woodlab.work` covers all subdomains. For any additional records:

```bash
cloudflared tunnel route dns woodhouse <hostname>.woodlab.work
```

## Tunnel management

The tunnel was created with `cloudflared tunnel create woodhouse`. To list or inspect:

```bash
cloudflared tunnel list
cloudflared tunnel info woodhouse
```

Tunnel credentials are stored as a SOPS-encrypted Secret in the cluster. See [Managing Secrets](managing-secrets.md) if you need to rotate them.

## Remote kubectl access

The K8s API is exposed through the tunnel at `k8s.woodlab.work`, protected by Cloudflare Access (email whitelist).

### Traffic flow

```
kubectl → cloudflared access tcp (local :6444) → Cloudflare Edge → Tunnel → cloudflared pod → kubernetes.default.svc.cluster.local:443
```

### Ingress rule

The tunnel config includes a hostname-specific rule for `k8s.woodlab.work` that routes TCP traffic directly to the in-cluster K8s API, before the catch-all Traefik rule.

### Setup (manual steps)

**1. Cloudflare Access application**

In the Cloudflare Zero Trust dashboard, create a self-hosted application:

- **Application domain:** `k8s.woodlab.work`
- **Policy:** Allow — identity rule using email list
- **Session duration:** 24h

**2. DNS (if not already covered by wildcard)**

```bash
cloudflared tunnel route dns woodhouse k8s.woodlab.work
```

**3. Client-side `cloudflared` proxy**

Run this on your local machine to proxy kubectl traffic through the tunnel:

```bash
cloudflared access tcp --hostname k8s.woodlab.work --url 127.0.0.1:6444
```

For persistent access, run this as a background service / systemd unit.

**4. Kubeconfig**

Update your kubeconfig to point at the local proxy:

```yaml
clusters:
  - cluster:
      server: https://127.0.0.1:6444
      # Keep the existing certificate-authority-data or use
      # insecure-skip-tls-verify: true if the API cert doesn't
      # cover 127.0.0.1
    name: woodhouse
```

**5. Verify**

```bash
kubectl get nodes
```
