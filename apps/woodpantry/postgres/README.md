# woodpantry-postgres

Shared CloudNativePG cluster for all woodpantry services. One database per service, created manually after the cluster is running.

## Metrics

`enablePodMonitor` is disabled in `cluster.yaml`. The CNPG chart's built-in pod monitor targets `monitoring.coreos.com/v1/PodMonitor` (Prometheus Operator), which does not exist in this cluster — Victoria Metrics uses its own CRD types instead.

To wire up CNPG metrics later, add a `VMPodScrape` resource to this directory targeting the `woodpantry-postgres` pods.
