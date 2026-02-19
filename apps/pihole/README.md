# Pi-hole

Network-wide ad blocking via DNS. DNS is exposed via a MetalLB `LoadBalancer` service at the static IP `192.168.1.2` (port 53 UDP/TCP), independent of any node.

## Network-wide ad blocking setup

Point your router's DHCP DNS server to `192.168.1.2`. Clients will automatically use Pi-hole for DNS resolution.

The admin UI is accessible at `pihole.woodlab.work`.

MetalLB is configured in `infra/metallb/` (Helm) and `infra/metallb-config/` (IP pool: `192.168.1.2-192.168.1.9`).

## Secret

`secret.yaml` must be SOPS-encrypted before committing. To update the password:

```bash
sops apps/pihole/secret.yaml
```
