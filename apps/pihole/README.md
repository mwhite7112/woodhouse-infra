# Pi-hole

Network-wide ad blocking via DNS. The pod is pinned to the control plane node using the standard `node-role.kubernetes.io/control-plane` label, with `hostPort: 53` so the node's IP acts as the DNS server.

## Network-wide ad blocking setup

Once deployed, point your router's DHCP DNS server to the control plane node's IP. Clients will automatically use Pi-hole for DNS resolution.

The admin UI is accessible at `pihole.woodlab.work`.

## Multiple control plane nodes

The current `nodeSelector` matches **any** control plane node. With a single control plane this is fine, but if you add a second control plane node the pod could reschedule to either one, changing the DNS IP and breaking client DNS until the router is updated.

To fix this, label one specific node and use that instead:

```bash
kubectl label node <node-name> pihole-dns=true
```

Then update the `nodeSelector` in `deployment.yaml`:

```yaml
nodeSelector:
  pihole-dns: "true"
```

This pins Pi-hole to exactly one node without exposing any hostname or IP in the repo.

## Secret

`secret.yaml` must be SOPS-encrypted before committing. To update the password:

```bash
sops apps/pihole/secret.yaml
```
