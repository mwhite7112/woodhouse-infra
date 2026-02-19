# Pi-hole

Network-wide ad blocking via DNS. The pod is pinned to the control plane node using the standard `node-role.kubernetes.io/control-plane` label, with `hostPort: 53` so the node's IP acts as the DNS server.

## Network-wide ad blocking setup

Pi-hole runs on a worker node labeled `pihole-dns=true`. Label your chosen worker before deploying:

```bash
kubectl label node <node-name> pihole-dns=true
```

Then point your router's DHCP DNS server to that worker node's IP. Clients will automatically use Pi-hole for DNS resolution.

The admin UI is accessible at `pihole.woodlab.work`.

## Changing which node Pi-hole runs on

Remove the label from the current node and add it to the new one:

```bash
kubectl label node <old-node> pihole-dns-
kubectl label node <new-node> pihole-dns=true
```

Update your router's DNS to point to the new node's IP.

## Secret

`secret.yaml` must be SOPS-encrypted before committing. To update the password:

```bash
sops apps/pihole/secret.yaml
```
