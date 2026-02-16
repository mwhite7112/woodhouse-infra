# Longhorn

Distributed replicated block storage — the cluster's default StorageClass.

- **Namespace:** `longhorn-system` (pod-security: `privileged`)
- **Install method:** HelmRelease (chart v1.9.x from `https://charts.longhorn.io`)
- **StorageClass:** `longhorn` (cluster default)
- **Replica count:** 2
- **Data path:** `/var/lib/longhorn`
- **Disk selection:** Only nodes labeled `node.longhorn.io/create-default-disk=true` (worker nodes)

## Key files

- `infra/longhorn/helmrelease.yaml`
- `infra/longhorn/helmrepository.yaml`
- `infra/longhorn/namespace.yaml`

## Design notes

Distributed block storage with 2-replica redundancy across worker nodes. Requires `iscsi-tools` and `util-linux-tools` Talos system extensions and a `/var/lib/longhorn` kubelet extra mount on all nodes. Replaces the previous Local Path Provisioner which had no replication.
