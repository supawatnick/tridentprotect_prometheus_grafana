# Deployment Log

Environment: `k8s-clus1`

> หมายเหตุ: Step 1-13 เป็นบันทึกจากรอบทดสอบแรก ซึ่งใช้ชื่อไฟล์เดิมก่อนจัดชุดไฟล์ใหม่
> (เช่น `03-ksm-values.yaml` ปัจจุบันคือ `05-ksm-values.yaml`, `02-prometheus-rbac.yaml` ปัจจุบันคือ `07-prometheus-rbac.yaml`,
> `05-prometheus-cr.yaml` ปัจจุบันคือ `08-prometheus.yaml`) ใช้แค่อ้างอิงประวัติเท่านั้น

## Step 1: Add Prometheus Community Helm Repository

- Time: `2026-11-10 14:28:46 +07`
- Status: PASS

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
```

Result:

```text
"prometheus-community" already exists with the same configuration, skipping
```

The repository was already configured with the same URL. No change was needed.

## Step 2: Update Helm Repositories

- Time: `2026-11-10 14:28:46 +07`
- Status: PASS

```bash
helm repo update
```

Result:

```text
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "netapp-trident" chart repository
...Successfully got an update from the "netapp-trident-protect" chart repository
...Successfully got an update from the "prometheus-community" chart repository
...Successfully got an update from the "kube-prometheus-stack" chart repository
...Successfully got an update from the "grafana" chart repository
Update Complete. Happy Helming!
```

All configured Helm repositories were updated successfully.

## Step 3: Install ServiceMonitor CRD

- Time: `2026-11-10 14:30:37 +07`
- Status: PASS

```bash
kubectl apply -f https://raw.githubusercontent.com/prometheus-operator/prometheus-operator/main/example/prometheus-operator-crd/monitoring.coreos.com_servicemonitors.yaml
```

Result:

```text
customresourcedefinition.apiextensions.k8s.io/servicemonitors.monitoring.coreos.com created
```

Verification:

```text
NAME                                    CREATED
servicemonitors.monitoring.coreos.com   2026-11-10T07:30:18Z
```

The ServiceMonitor CRD was created successfully.

## Step 4: Create Prometheus Namespace

- Time: `2026-11-10 14:32 +07`
- Status: PASS

```bash
kubectl create ns prometheus
```

Result:

```text
namespace/prometheus created
```

## Step 5: Install Trident Protect kube-state-metrics

- Time: `2026-11-10 14:32:43 +07`
- Status: PASS

```bash
helm install trident-protect -f 03-ksm-values.yaml prometheus-community/kube-state-metrics --version 5.21.0 -n prometheus
```

Result:

```text
NAME: trident-protect
NAMESPACE: prometheus
STATUS: deployed
REVISION: 1
DESCRIPTION: Install complete
```

Verification:

```text
pod/trident-protect-kube-state-metrics-5dd75b8787-7mmzs   1/1   Running
service/trident-protect-kube-state-metrics                ClusterIP   8080/TCP
servicemonitor/trident-protect-kube-state-metrics         created
```

The generated KSM ClusterRole includes `list` and `watch` access for all configured Trident Protect resources:

```text
backups
snapshots
exechooksruns
appvaults
appmirrorrelationships
```

## Step 6: Install Prometheus RBAC

- Time: `2026-11-10 14:34:54 +07`
- Status: PASS

```bash
kubectl apply -f 02-prometheus-rbac.yaml
```

Result:

```text
serviceaccount/prometheus created
clusterrole.rbac.authorization.k8s.io/prometheus created
clusterrolebinding.rbac.authorization.k8s.io/prometheus created
```

Verification:

```text
ServiceAccount/prometheus       present in namespace prometheus
ClusterRole/prometheus          present
ClusterRoleBinding/prometheus   present

list services                               yes
list endpoints                              yes
list endpointslices.discovery.k8s.io        yes
list servicemonitors.monitoring.coreos.com  yes
```

The Prometheus ServiceAccount is bound successfully and has the required discovery permissions.

## Step 7: Test Prometheus CR Before Installing Its CRD

- Time: Not recorded
- Status: EXPECTED FAIL

```bash
kubectl apply -f 05-prometheus-cr.yaml
```

Result:

```text
error: resource mapping not found for name: "prometheus" namespace: "prometheus" from "05-prometheus-cr.yaml": no matches for kind "Prometheus" in version "monitoring.coreos.com/v1"
ensure CRDs are installed first
```

Cause: `prometheuses.monitoring.coreos.com` was not installed. No resource from the minimal `05-prometheus-cr.yaml` was created.

## Step 8: Test CRD Installation With Client-Side Apply

- Time: Not recorded
- Status: PARTIAL

Commands and results:

```text
prometheuses.monitoring.coreos.com  FAILED: metadata.annotations too long
prometheusrules.monitoring.coreos.com  created
alertmanagers.monitoring.coreos.com  FAILED: metadata.annotations too long
```

Client-side apply stores the manifest in the `kubectl.kubernetes.io/last-applied-configuration` annotation. The Prometheus and Alertmanager CRD schemas exceed the annotation limit of 262144 bytes.

## Step 9: Roll Back Additional Monitoring CRDs

- Time: Not recorded
- Status: PASS

Removed:

```text
prometheuses.monitoring.coreos.com
prometheusrules.monitoring.coreos.com
```

`alertmanagers.monitoring.coreos.com` did not exist because its earlier apply failed.

Retained for the running KSM release:

```text
servicemonitors.monitoring.coreos.com
servicemonitor/trident-protect-kube-state-metrics
```

The KSM pod remained `1/1 Running`. Prometheus RBAC was also retained.

## Step 10: Install Required CRDs With Server-Side Apply

- Time: `2026-11-10 15:22-15:23 +07`
- Status: PASS

The CRDs were installed individually from the Prometheus Operator `v0.95.1` manifests. Server-side apply was used for the large CRDs to avoid the client-side annotation limit.

Final verification:

```text
servicemonitors.monitoring.coreos.com   Established=True
prometheuses.monitoring.coreos.com      Established=True
prometheusrules.monitoring.coreos.com   Established=True
alertmanagers.monitoring.coreos.com     Established=True
```

Prometheus CRD schema verification:

```text
v1 served=true storage=true schema=object
```

All four CRDs required by this monitoring stack are now installed and established.

## Step 11: Install Prometheus Operator

- Time: `2026-11-10 15:45 +07`
- Status: PASS

```bash
curl -sSL \
  "https://github.com/prometheus-operator/prometheus-operator/releases/download/${OPERATOR_VERSION}/bundle.yaml" \
  | sed "s/namespace: default/namespace: ${NAMESPACE}/g" \
  | kubectl apply --server-side -f -
```

Result:

```text
Prometheus Operator CRDs (10)       serverside-applied
ClusterRole/prometheus-operator     serverside-applied
ClusterRoleBinding/prometheus-operator serverside-applied
Deployment/prometheus-operator      serverside-applied
ServiceAccount/prometheus-operator  serverside-applied
Service/prometheus-operator         serverside-applied
```

Verification:

```text
prometheus-operator       1/1 Running
prometheus-prometheus-0   2/2 Running
Prometheus READY=True RECONCILED=True AVAILABLE=True
prometheus-operated       port 9090
```

## Step 12: Expose Prometheus Through MetalLB

- Time: `2026-11-10 15:54 +07`
- Status: PASS

```bash
kubectl apply -f 10-prometheus-external-service.yaml
```

Result:

```text
service/prometheus-external created
EXTERNAL-IP: 10.10.110.111
PORT: 9090
```

MetalLB selected the address automatically from `external-pool`; the manifest does not lock a specific IP.

## Step 13: Verify KSM Integration

- Time: `2026-11-10 16:00 +07`
- Status: PASS

```text
Target health: up
Target URL: http://10.244.38.223:8080/metrics
Job: serviceMonitor/prometheus/trident-protect-kube-state-metrics/0
```

Metrics observed:

```text
kube_customresource_backup_info                    14 series
kube_customresource_snapshot_info                  49 series
kube_customresource_ehr_info                       130 series
kube_customresource_appvault_info                  4 series
kube_customresource_app_mirror_relationship_info   0 series (no AMR resource)
```

## Step 14: Clean Up Before Fresh Deployment

- Time: `2026-11-10 16:06:00 +07`
- Status: PASS

Removed:

```text
Helm release trident-protect in namespace prometheus
Namespace prometheus
ClusterRole and ClusterRoleBinding for prometheus
ClusterRole and ClusterRoleBinding for prometheus-operator
All 10 monitoring.coreos.com CRDs
```

The main Trident Protect installation in namespace `trident-protect` was not modified. The cluster is ready for a fresh monitoring deployment.

## Step 15: Add Blog Labels to KSM Configuration

- Time: `2026-11-11 10:30 +07`
- Status: PASS

Updated `05-ksm-values.yaml`:

```yaml
# resource level labelsFromPath
creation_time: [metadata, creationTimestamp]
```

Added to Snapshot, Backup, ExecHooksRun and AppMirrorRelationship. AppVault was left unchanged to match the blog exactly.

```yaml
# AppMirrorRelationship info.labelsFromPath
destinationNamespaces: ["status", "destinationNamespaces"]
promotedSnapshot: ["spec", "promotedSnapshot"]
namespaceMapping: ["spec", "namespaceMapping"]
conditions: ["status", "conditions"]
```

Validation:

```text
YAML OK
Helm template OK (chart 5.21.0)
Helm template OK (chart 9.0.0)
```

## Step 16: Upgrade kube-state-metrics and Fix Duplicate Metrics

- Time: `2026-11-11 10:34:14 +07`
- Status: PASS

```bash
helm upgrade trident-protect prometheus-community/kube-state-metrics \
  -f 05-ksm-values.yaml \
  --version 9.0.0 \
  -n prometheus
```

Result:

```text
Release "trident-protect" has been upgraded. Happy Helming!
STATUS: deployed
REVISION: 2
```

```text
trident-protect   prometheus   2   deployed   kube-state-metrics-9.0.0   2.20.0
pod/trident-protect-kube-state-metrics-558985498c-pm52q   1/1   Running
image=registry.k8s.io/kube-state-metrics/kube-state-metrics:v2.20.0
```

Chart `5.21.0` (KSM `v2.12.0`) sent every custom-resource series twice. KSM `v2.14.0` fixed it in PR #2502, so the chart was moved to `9.0.0` (KSM `v2.20.0`).

Raw KSM response after the upgrade:

```text
kube_customresource_backup_info                    total=18   unique=18   OK
kube_customresource_snapshot_info                  total=73   unique=73   OK
kube_customresource_ehr_info                       total=186  unique=186  OK
kube_customresource_appvault_info                  total=4    unique=4    OK
kube_customresource_app_mirror_relationship_info   total=1    unique=1    OK
```

Prometheus ingestion warnings after the new pod became ready:

```text
Error on ingesting samples with different value but same timestamp   count=0
```

The 35 warnings seen in the previous 10 minutes all came from the old pod `10.244.38.220` before the upgrade.

KSM logs showed no error, warning or panic.

## Step 17: Verify New Labels in Prometheus

- Time: `2026-11-11 10:35 +07`
- Status: PASS

```text
creation_time          2026-11-11T03:04:50Z (and values for every Snapshot, Backup and ExecHooksRun)
destinationNamespaces  [demo203]
namespaceMapping       [map[destination:demo203 source:demo203-dr]]
conditions             [map[lastTransitionTime:2026-11-11T03:06:03Z ... type:Established] ...]
promotedSnapshot       absent (spec.promotedSnapshot is not set until the relationship is promoted)
```

Current series counts:

```text
kube_customresource_snapshot_info                    73
kube_customresource_backup_info                      18
kube_customresource_ehr_info                         186
kube_customresource_appvault_info                    4
kube_customresource_app_mirror_relationship_info     1
```

KSM target after the upgrade:

```text
health=up  url=http://10.244.229.76:8080/metrics  lastError=
```

## Step 18: Monitor Trident CSI Metrics

- Time: `2026-11-11 10:52 +07`
- Status: PASS

Created `13-trident-servicemonitor.yaml` matching the blog exactly:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: trident-sm
  namespace: prometheus
  labels:
    release: prom-operator
spec:
  jobLabel: trident
  selector:
    matchLabels:
      app: controller.csi.trident.netapp.io
  namespaceSelector:
    matchNames:
      - trident
  endpoints:
    - port: metrics
      interval: 15s
```

Pre-check on `k8s-clus1`:

```text
Service trident/trident-csi
Labels  app=controller.csi.trident.netapp.io, k8s_version=v1.36.4, trident_version=v26.06.0
Port    name=metrics port=9220 targetPort=8001
```

Apply:

```text
servicemonitor.monitoring.coreos.com/trident-sm created (server dry run)
servicemonitor.monitoring.coreos.com/trident-sm created
```

The target appeared after the operator re-synced the Prometheus resource and the kubelet
synced the regenerated config secret into the pod, which takes roughly 1-2 minutes.

Generated scrape job:

```text
job= serviceMonitor/prometheus/trident-sm/0 | interval= 15s | role= endpoints | ns= ['trident']
```

Target:

```text
health=up job=trident-csi service=trident-csi ns=trident endpoint=metrics
url=http://10.244.38.228:8001/metrics err= scrapeDuration=0.0036s
```

The `job` label is `trident-csi` rather than `trident`. The Prometheus Operator always adds a
default relabel rule that sets `job` to the Service name, and the `jobLabel: trident` rule
only overrides it when the label actually exists. The `trident-csi` Service has no `trident`
label, so the default rule wins (prometheus-operator v0.95.1, pkg/prometheus/promcfg.go:2217-2232).

31 `trident_*` metric families are being scraped. Real label names are
`backend_name`, `backend_type`, `backend_uuid`, `volume_state`, `volume_type`.

Health and configuration:

```text
trident_backend_count              1, 1, 2, 1
trident_node_count                 3
trident_storageclass_count         2
trident_autogrow_policy_count      0
trident_build_info                 trident_version=26.06.0 build_type=stable
```

Backend usage:

```text
backend=k8s_clus1_active_nfs type=ontap-nas
backend=k8s_clus1_iscsi      type=ontap-san
backend=k8s_clus1_nfs        type=ontap-nas
backend=k8s_clus2_iscsi      type=ontap-san
backend=k8s_clus2_nfs        type=ontap-nas
```

```text
trident_volume_count       ontap-nas/3ce08622... = 25, ontap-nas/eba8c81f... = 21
trident_volume_total_bytes 210453397504
trident_snapshot_count     73
```

Performance:

```text
trident_operation_duration_milliseconds_count          38 series
trident_incoming_api_request_duration_seconds_count    12 series
trident_outgoing_api_request_duration_seconds_count    21 series
trident_rest_ops_total                                   6 series
```

## Step 19: Install Grafana

- Time: `2026-11-11 11:51:48 +07`
- Status: PASS

Files created:

```text
14-grafana-values.yaml
14-grafana-install.md
15-grafana-datasource.md
16-grafana-dashboard.md
16-grafana-dashboard.json
```

Commands:

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
kubectl create namespace grafana --dry-run=client -o yaml | kubectl apply -f -
helm install my-grafana grafana/grafana \
  -f 14-grafana-values.yaml \
  --version 10.5.15 \
  -n grafana
```

Result:

```text
NAME            NAMESPACE  REVISION  UPDATED                             STATUS    CHART            APP VERSION
my-grafana      grafana    1         2026-11-11 11:51:48 +0700 +07       deployed  grafana-10.5.15  12.3.1
```

Verification:

```text
deployment.apps/my-grafana          1/1   1   1
pod/my-grafana-55794444bb-kbt8f    1/1   Running   0
service/my-grafana                 LoadBalancer   10.98.88.253   10.10.110.112   80:31114/TCP
endpoints/my-grafana               10.244.38.243:3000
persistentvolumeclaim/my-grafana   Bound   10Gi   RWO   sc-clus1-nfs
```

Grafana health API:

```text
{"database":"ok","version":"12.3.1","commit":"3a1c80ca7ce612f309fdc99338dd3c5e486339be"}
```

Notes:

- The chart is marked deprecated upstream and moved to an OCI registry, but the OCI
  registry requires authentication (`401 unauthorized`), so the HTTP repository is still
  used. `10.5.15` is the newest version in that repository and is pinned in the file.
- The admin password in the secret no longer authenticates against the Grafana API
  (`invalid password`), which means the password was changed after the first login.
  The data source and dashboard steps are therefore done through the UI.

## Step 20: Build Custom Trident Protect Dashboard

- Time: `2026-11-11 12:10 +07`
- Status: PASS (file created and validated, not yet imported)

Files created:

```text
17-trident-protect-dashboard.json
17-trident-protect-dashboard.md
```

Dashboard specification:

```text
title   = Trident Protect Overview
uid     = trident-protect-overview
panels  = 20 across 5 rows
tags    = ["trident-protect", "trident", "netapp"]
refresh = 30s
time    = now-24h -> now
variables = DS_PROMETHEUS, app, vault
```

Every query was executed against the live Prometheus instance before writing the file:

```text
Total Backups              stat         1   18
Total Snapshots            stat         1   76
Failed Backups             stat         1   0
Failed Snapshots           stat         1   0
Running Backups            stat         1   0
AppVaults Available        stat         1   4
AppVaults in Error         stat         1   0
AMR Established            stat         1   1
Backups by status          piechart     1   18
Snapshots by status        piechart     1   76
AppVaults by state         piechart     1   4
AMR by state               piechart     1   1
Backups over time          timeseries   1   53 points
Snapshots over time        timeseries   1   53 points
Backups per application    bargauge     5   3
Snapshots per application  bargauge     7   7
Latest Backups             table        18  1
Latest Snapshots           table        76  1
AppVaults                  table        4   1
AppMirrorRelationships     table        1   1
```

No panel returned an empty result. Panel layout was checked for overlaps and fits
inside 24 columns.

Dashboard JSON is ready to import through the Grafana UI.

## Step 21: Fix Custom Dashboard Table Panels and Add AMR Detail

- Time: `2026-11-11 12:40 +07`
- Status: PASS

The first import succeeded but the table panels rendered empty. Grafana logs showed no query
errors, so the problem was the response format rather than the query.

Root cause found by comparing with the working blog dashboard:

```text
                 format        limit options
blog (works)     "table"       {}
ours (empty)     "time_series" {"limitField": "limit"}
```

With `format: time_series` the Prometheus datasource returns one frame per series that only
contains the `Time` and `Value` fields. The `organize` transformation excluded both of them,
which left every table with zero fields.

The `limit` transformation option is a row count, not a field name
(grafana/packages/grafana-data/src/transformations/transformers/limit.ts):

```ts
export interface LimitTransformerOptions {
  limitField?: number | string;
}
```

`parseInt("limit", 10)` returns `NaN`, so the previous value never limited anything.

Fixes applied:

```text
table target format   = "table"
limit transformation  = {"id": "limit", "options": {"limitField": 10}}
```

Added a dedicated AppMirrorRelationship section:

```text
Row 6 - AppMirrorRelationship Detail
  AMRs Total / AMRs Established / AMRs Not Established / AMRs Promoted   stat
  AMRs by Desired State                                                    piechart
  AMR Transfer Detail                                                      table
  AMR Conditions                                                           table
```

New template variable:

```text
amr = label_values(kube_customresource_app_mirror_relationship_info, amr_name)
```

Dashboard now has 27 panels across 6 rows and 4 template variables.
All 27 queries were executed against the live Prometheus instance, no panel returned an
empty result. The `organize` transformation was simulated against the real label sets to
confirm the visible columns of every table.

```text
Latest Backups         Date, Application, Backup, Status, AppVault
Latest Snapshots       Date, Application, Snapshot, Status, AppVault
AppVaults              AppVault, State, Error, Message
AMR Configuration      Relationship, Source App, Source Vault, Dest Vault, Storage Class, Recurrence, Dest Namespace, Namespace Mapping
AMR Transfer Detail    Relationship, Transfer Start, Transfer End, Last Snapshot, Snapshot Done, State
AMR Conditions         Relationship, State, Desired State, Conditions, Created
```

The `Error` and `Message` columns of the AppVaults table only appear when an AppVault is
actually in the Error state, which is the expected behaviour.

## Step 22: Deploy Monitoring Stack on k8s-clus2

- Time: `2026-11-11 13:55-14:12 +07`
- Status: PASS

The full monitoring stack was deployed on `k8s-clus2` to prove the runbook works on a
second cluster. Steps 01 to 13 were applied with the same files as `k8s-clus1`.

Two files needed cluster specific values because the two clusters use different Trident
storage classes and different MetalLB pools:

```text
                          k8s-clus1                    k8s-clus2
storage class             sc-clus1-nfs                 sc-clus2-nfs
MetalLB pool              external-pool (.101-.120)     ai-pool (.121-.140)
```

The pools do not overlap, so both clusters can advertise on the same LAN at the same time.

Files created:

```text
10-prometheus-external-service-clus2.yaml
14-grafana-values-clus2.yaml
17-trident-protect-dashboard-clus2.json
```

Issue found during the deployment: the first Grafana install on k8s-clus2 used the k8s-clus1
values, so the PVC stayed Pending and the Service never received an address.

```text
persistentvolumeclaim/my-grafana   Pending   sc=sc-clus1-nfs
  Warning ProvisioningFailed: storageclass.storage.k8s.io "sc-clus1-nfs" not found
service/my-grafana                  LoadBalancer   EXTERNAL-IP=<pending>
```

The PVC could not be patched because `spec.storageClassName` is an immutable field, so the
release was uninstalled and reinstalled with the corrected values:

```bash
helm uninstall my-grafana -n grafana
kubectl -n grafana delete pvc my-grafana --ignore-not-found
helm install my-grafana grafana/grafana \
  -f 14-grafana-values-clus2.yaml \
  --version 10.5.15 \
  -n grafana
```

Final state on k8s-clus2:

```text
deployment.apps/my-grafana          1/1   1   1
pod/my-grafana-75f9d7d7cb-9mcrq    1/1   Running   0
service/my-grafana                 LoadBalancer   10.108.157.20   10.10.110.122   80:32174/TCP
persistentvolumeclaim/my-grafana   Bound   10Gi   RWO   sc-clus2-nfs
```

Prometheus external service on k8s-clus2:

```text
service/prometheus-external   LoadBalancer   10.109.52.90   10.10.110.121   9090:31132/TCP
```

Targets:

```text
health=up job=gpu-operator        ns=gpu-operator
health=up job=kube-state-metrics  ns=prometheus
health=up job=trident-csi         ns=trident
```

Metric series on k8s-clus2:

```text
kube_customresource_backup_info                    0
kube_customresource_snapshot_info                  3
kube_customresource_ehr_info                       6
kube_customresource_appvault_info                  4
kube_customresource_app_mirror_relationship_info   6
```

Grafana health:

```text
{"database":"ok","version":"12.3.1"}
```

## Step 23: Make the Application Variable Work on Both Clusters

- Time: `2026-11-11 14:15 +07`
- Status: PASS

`k8s-clus2` has no Backup CR at all, so the original variable query produced an empty
dropdown and every panel filtered with `{appReference=~"$app"}` would show No data.

The variable now uses a selector that covers both metrics:

```yaml
label_values({__name__=~"kube_customresource_(backup|snapshot)_info"}, appReference)
```

Grafana passes the metric argument to `/api/v1/label/<label>/values?match[]=<selector>`
(confirmed in packages/grafana-prometheus/src/metric_find_query.ts and migrations/variableMigration.ts).

Values returned:

```text
k8s-clus1  auto-auto-backup-01, auto-auto-backup-02, auto-auto-backup-03, auto-auto-backup-05, demo201
k8s-clus2  demo203
```

## Step 24: Document Cleanup and Uninstall Procedure

- Time: `2026-11-11 14:45 +07`
- Status: PASS (documented and validated with dry-run, not executed)

Created `18-cleanup-uninstall.md` covering both clusters.

What gets removed and how:

```text
Grafana               my-grafana (ns grafana)          helm uninstall
kube-state-metrics    trident-protect (ns prometheus)  helm uninstall
Prometheus CR         prometheus/prometheus           kubectl delete -f 08-prometheus.yaml
Prometheus RBAC       SA + ClusterRole + CRB         kubectl delete -f 07-prometheus-rbac.yaml
External Service      prometheus-external             kubectl delete -f 10-prometheus-external-service*.yaml
ServiceMonitor        trident-sm                      kubectl delete -f 13-trident-servicemonitor.yaml
Prometheus Operator   deployment + service + SA       kubectl delete
Operator RBAC         ClusterRole + CRB               kubectl delete
CRDs                  10 monitoring.coreos.com CRDs   kubectl delete
Namespaces            prometheus, grafana            kubectl delete namespace
```

Every command was validated with `--dry-run=client` against k8s-clus1 and produced no
errors. The read-only inspection commands were executed against the live cluster and
returned the expected resources.

Two safety points were found while writing the file:

- There are two Helm releases named `trident-protect`. The one in namespace `prometheus`
  is the kube-state-metrics install and must be removed. The one in namespace
  `trident-protect` is the real Trident Protect installation and must never be removed.
  The `-n prometheus` argument makes the command unambiguous, and the file now warns
  about it explicitly.
- `helm uninstall my-grafana` removes the PVC and the PV as well, because the chart does
  not set `helm.sh/resource-policy: keep` and the StorageClass uses
  `reclaimPolicy: Delete`. Dashboards and data sources created through the UI are lost.

The file also documents that namespace `demo203-dr` on k8s-clus2 must not be deleted, since
it holds the `demo203` Application and is the source of the `amr-demo203` relationship.
