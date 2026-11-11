# NetApp Trident Protect — Prometheus and Grafana Monitoring

Monitoring and visualization for NetApp Trident and Trident Protect with Prometheus,
Grafana and kube-state-metrics, following the NetApp Community guide.

## Source

- Community blog: <https://community.netapp.com/community/discussion/463692/netapp-trident-protect-metrics-and-monitoring>
- NetApp docs: <https://docs.netapp.com/us-en/trident/trident-protect/monitor-trident-protect-resources.html>

## What it collects

Trident Protect custom resources, scraped by kube-state-metrics through the Prometheus
Operator:

| Resource | Metric |
|---|---|
| Backup | `kube_customresource_backup_info` |
| Snapshot | `kube_customresource_snapshot_info` |
| ExecHooksRun | `kube_customresource_ehr_info` |
| AppVault | `kube_customresource_appvault_info` |
| AppMirrorRelationship | `kube_customresource_app_mirror_relationship_info` |

Trident CSI exposes its own metrics on port `8001` at `/metrics`, collected through a
separate ServiceMonitor:

| Purpose | Metric examples |
|---|---|
| Backend and volume usage | `trident_backend_info`, `trident_volume_count`, `trident_volume_allocated_bytes` |
| Performance | `trident_operation_duration_milliseconds`, `trident_incoming_api_request_duration_seconds`, `trident_rest_ops_total` |

## Prerequisites

- Kubernetes cluster, tested on v1.36.4
- Trident and Trident Protect, tested on 26.06
- Helm and kubectl
- MetalLB, if Prometheus and Grafana should be reachable from outside the cluster
- A Trident NFS storage class for Grafana persistence

## Versions used

| Component | Version |
|---|---|
| Prometheus Operator | `v0.95.1` |
| Prometheus | `v3.14.0` |
| kube-state-metrics chart | `9.0.0` (app `v2.20.0`) |
| Grafana chart | `10.5.15` (app `12.3.1`) |

## Run order

Run the files in this order. Each step has its own verify block.

| Step | File | What it does |
|---|---|---|
| 1 | `01-servicemonitor-crd.md` | Install ServiceMonitor CRD |
| 2 | `02-prometheus-crd.md` | Install Prometheus CRD |
| 3 | `03-prometheusrule-crd.md` | Install PrometheusRule CRD |
| 4 | `04-alertmanager-crd.md` | Install Alertmanager CRD |
| 5 | `05-ksm-values.yaml` | KSM configuration and RBAC for the Trident Protect CRs |
| 6 | `06-install-ksm.md` | Install kube-state-metrics |
| 7 | `07-prometheus-rbac.yaml` | Prometheus ServiceAccount, ClusterRole and binding |
| 8 | `08-prometheus.yaml` | Prometheus custom resource |
| 9 | `09-prometheus-operator-installation.md` | Install Prometheus Operator and the remaining CRDs |
| 10 | `10-prometheus-external-service.yaml` | Expose Prometheus through MetalLB |
| 11 | `13-trident-servicemonitor.yaml` | ServiceMonitor for the Trident CSI metrics |
| 12 | `14-grafana-install.md`, `14-grafana-values.yaml` | Install Grafana with persistent storage |
| 13 | `15-grafana-datasource.md` | Add Prometheus as a Grafana data source |
| 14 | `16-grafana-dashboard.md`, `16-grafana-dashboard.json` | Import the dashboard from the blog |
| 15 | `17-trident-protect-dashboard.md`, `17-trident-protect-dashboard.json` | Import the custom dashboard |

`12-nginx-test-app-clus2.yaml` is an optional test application used to verify the
AppMirrorRelationship replication path.

## Quick start

```bash
export KUBECONFIG=/root/.kube/config
export NS=prometheus

# 1. CRDs
kubectl apply --server-side -f https://raw.githubusercontent.com/prometheus-operator/prometheus-operator/v0.95.1/example/prometheus-operator-crd/monitoring.coreos.com_servicemonitors.yaml
kubectl apply --server-side -f https://raw.githubusercontent.com/prometheus-operator/prometheus-operator/v0.95.1/example/prometheus-operator-crd/monitoring.coreos.com_prometheuses.yaml
kubectl apply --server-side -f https://raw.githubusercontent.com/prometheus-operator/prometheus-operator/v0.95.1/example/prometheus-operator-crd/monitoring.coreos.com_prometheusrules.yaml
kubectl apply --server-side -f https://raw.githubusercontent.com/prometheus-operator/prometheus-operator/v0.95.1/example/prometheus-operator-crd/monitoring.coreos.com_alertmanagers.yaml

# 2. Prometheus Operator
curl -sSL "https://github.com/prometheus-operator/prometheus-operator/releases/download/v0.95.1/bundle.yaml" \
  | sed "s/namespace: default/namespace: ${NS}/g" \
  | kubectl apply --server-side -f -

# 3. kube-state-metrics
kubectl create namespace ${NS} --dry-run=client -o yaml | kubectl apply -f -
helm install trident-protect -f 05-ksm-values.yaml \
  prometheus-community/kube-state-metrics --version 9.0.0 -n ${NS}

# 4. Prometheus
kubectl apply -f 07-prometheus-rbac.yaml
kubectl apply -f 08-prometheus.yaml

# 5. ServiceMonitors and external access
kubectl apply -f 13-trident-servicemonitor.yaml
kubectl apply -f 10-prometheus-external-service.yaml

# 6. Grafana
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
kubectl create namespace grafana --dry-run=client -o yaml | kubectl apply -f -
helm install my-grafana grafana/grafana -f 14-grafana-values.yaml --version 10.5.15 -n grafana
```

## Cluster specific values

The two clusters use different Trident storage classes and different MetalLB pools, so a
few files have a `-clus2` variant.

| Value | Cluster 1 | Cluster 2 |
|---|---|---|
| kubeconfig | `/root/.kube/config` | `/root/.kube/config-clus2` |
| StorageClass | `sc-clus1-nfs` | `sc-clus2-nfs` |
| MetalLB pool | `external-pool` = 10.10.110.101-120 | `ai-pool` = 10.10.110.121-140 |

| File | Cluster 1 | Cluster 2 |
|---|---|---|
| External Service | `10-prometheus-external-service.yaml` | `10-prometheus-external-service-clus2.yaml` |
| Grafana values | `14-grafana-values.yaml` | `14-grafana-values-clus2.yaml` |
| Custom dashboard | `17-trident-protect-dashboard.json` | `17-trident-protect-dashboard-clus2.json` |

The two MetalLB pools do not overlap, so both clusters can advertise on the same network
at the same time.

## Service URLs

Service DNS names use the standard Kubernetes form:

```
<service>.<namespace>.svc.cluster.local:<port>
```

| Service | URL |
|---|---|
| Prometheus | `http://prometheus-operated.prometheus.svc.cluster.local:9090` |
| Grafana | `http://my-grafana.grafana.svc.cluster.local` |
| kube-state-metrics | `http://trident-protect-kube-state-metrics.prometheus.svc.cluster.local:8080` |
| Trident CSI metrics | `http://trident-csi.trident.svc.cluster.local:9220` |

These names resolve inside the cluster only. From a workstation use the MetalLB address
instead.

## Dashboards

`16-grafana-dashboard.json` is the dashboard from the NetApp Community blog, with two
adjustments: the value mapping is changed from `Failed` to `Error` because that is the
status value Trident Protect uses, and the tag list gained `26.06`.

`17-trident-protect-dashboard.json` is a custom dashboard with 27 panels across 6 rows:

- Summary: totals, failed and running counts, AppVault and AppMirrorRelationship state
- Status breakdown: pie charts per status
- Growth over time: backup and snapshot counts
- Per application: bar gauges grouped by `appReference`
- Details: tables for backups, snapshots, AppVaults and replication relationships
- AppMirrorRelationship detail: transfer timestamps, snapshots, conditions and configuration

Both use the `${DS_PROMETHEUS}` data source variable, so Grafana asks which data source to
use during import.

## Cleanup

See `18-cleanup-uninstall.md` for the full teardown of the Helm releases, the custom
resources, the Operator, the CRDs and the namespaces.

## Notes

- **Large CRDs need `--server-side`.** Client-side apply stores the whole manifest in the
  `kubectl.kubernetes.io/last-applied-configuration` annotation, which fails for the
  Prometheus and Alertmanager CRDs with `metadata.annotations: Too long`.
- **kube-state-metrics chart 9.0.0, not 5.21.0.** The version used in the blog sends every
  custom resource series twice, which makes Prometheus drop samples with
  `Error on ingesting samples with different value but same timestamp`. The duplicate was
  fixed in kube-state-metrics v2.14.0.
- **The ServiceMonitor is the integration point.** The `prometheus-config` ConfigMap from the
  blog is not referenced by the Prometheus custom resource and is not mounted into the pod,
  so it has no effect. The kube-state-metrics chart creates a ServiceMonitor and the
  Prometheus custom resource selects it with `serviceMonitorSelector: {}`.
- **Custom resource metrics carry no namespace label.** The `namespace` label on these
  metrics is the namespace of the pod that was scraped, not the namespace of the custom
  resource. Only `appReference` identifies the application.
- **`creation_time` and `lastTransfer*` are labels.** They cannot be used in arithmetic
  against timestamps, only for ordering and grouping.
- **Grafana has no authentication.** When it is exposed through MetalLB, restrict network
  access or place an authenticating reverse proxy in front of it.

## Deployment log

`999-deployment-log.md` records every command and result, including the failures that
were found and fixed along the way.
