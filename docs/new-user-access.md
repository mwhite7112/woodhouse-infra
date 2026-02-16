# New User Access Runbook

How to create a new user with `cluster-admin` access to the Woodhouse cluster.

## 1. Create the RBAC resources

Add the following to `infra/rbac/`:

- A `ServiceAccount` for the user in the `cluster-admin-sa` namespace
- A `Secret` of type `kubernetes.io/service-account-token` annotated with the ServiceAccount name
- A `ClusterRoleBinding` binding the ServiceAccount to the `cluster-admin` ClusterRole

Add the new resources to `infra/rbac/kustomization.yaml`, merge to `main`, and wait for Flux to reconcile.

## 2. Verify the resources synced

```bash
kubectl get serviceaccount <name> -n cluster-admin-sa
kubectl get secret <token-secret-name> -n cluster-admin-sa
kubectl get clusterrolebinding <crb-name>
```

If the secret's `token` field is empty, wait a moment — Kubernetes populates it once it finds the matching ServiceAccount.

## 3. Build the kubeconfig

```bash
set TOKEN (kubectl get secret <token-secret-name> -n cluster-admin-sa -o jsonpath='{.data.token}' | base64 -d)

set SERVER (kubectl config view --raw -o jsonpath='{.clusters[0].cluster.server}')
kubectl config view --raw -o jsonpath='{.clusters[0].cluster.certificate-authority-data}' | base64 -d > /tmp/woodhouse-ca.crt

kubectl config set-cluster woodhouse \
  --server="$SERVER" \
  --certificate-authority=/tmp/woodhouse-ca.crt \
  --embed-certs=true \
  --kubeconfig=<name>.kubeconfig

rm /tmp/woodhouse-ca.crt

kubectl config set-credentials <name> \
  --token="$TOKEN" \
  --kubeconfig=<name>.kubeconfig

kubectl config set-context woodhouse \
  --cluster=woodhouse \
  --user=<name> \
  --kubeconfig=<name>.kubeconfig

kubectl config use-context woodhouse \
  --kubeconfig=<name>.kubeconfig
```

## 4. Verify

```bash
kubectl get nodes --kubeconfig=<name>.kubeconfig
```

Should list the 3 cluster nodes.

## 5. Hand off

Give the user the `<name>.kubeconfig` file. Transfer it securely — do not commit it to git. The user places it at `~/.kube/config` and kubectl will pick it up automatically.
