# 18 - Cleanup / Uninstall

Status: TESTED (`k8s-clus1`, 2026-11-12)

ถอน monitoring stack ทั้งหมดออกจาก cluster

คลัสเตอร์ : `k8s-clus1` และ `k8s-clus2`

## ภาพรวมสิ่งที่ติดตั้ง

| กลุ่ม | Resource | วิธีติดตั้ง | วิธีถอน |
|---|---|---|---|
| Grafana | `my-grafana` (ns `grafana`) | `helm install` | `helm uninstall` |
| Grafana External Service | `grafana-external` (ns `grafana`) | `kubectl apply` ตาม cluster | `kubectl delete -f` ตาม cluster |
| KSM | `trident-protect` (ns `prometheus`) | `helm install` | `helm uninstall` |
| Prometheus CR | `prometheus/prometheus` | `kubectl apply` | `kubectl delete -f` |
| Prometheus RBAC | SA + ClusterRole + CRB `prometheus` | `kubectl apply` | `kubectl delete -f` |
| External Service | `prometheus-external` | `kubectl apply` | `kubectl delete -f` |
| ServiceMonitor | `trident-sm` | `kubectl apply` | `kubectl delete -f` |
| Alertmanager | `alertmanager` + config + external Service | `kubectl apply` | `kubectl delete -f` |
| Alert rules | `trident-protect-alerts` | `kubectl apply` | `kubectl delete -f` |
| Mailpit | namespace `mailpit` | `kubectl apply` | ลบ namespace |
| Operator + CRD | `prometheus-operator` + CRD 10 ตัว | `bundle.yaml` | `kubectl delete` + ลบ CRD |

> ลำดับสำคัญ: **ลบ Prometheus CR ก่อน Operator และ CRD**
> ถ้าลบ CRD ก่อน CR ตัว StatefulSet และ Service อาจค้าง เพราะ owner reference
> garbage collection ไม่ได้เก็บทุกกรณี

## 0. ตรวจสถานะก่อนลบ

```bash
export KUBECONFIG=/root/.kube/config
```

สำหรับ `k8s-clus2` ใช้:

```bash
export KUBECONFIG=/root/.kube/config-clus2
```

```bash
helm list -A
kubectl -n prometheus get deploy,svc,sa,statefulset,servicemonitor,prometheus
kubectl -n grafana get deploy,svc,sa,pvc
kubectl get clusterrole,clusterrolebinding -o name \
  | grep -E '/(prometheus|prometheus-operator|trident-protect-kube-state-metrics)$'
kubectl get crd -o name | grep monitoring.coreos.com
```

> ⚠️ ถ้ามี `ClusterRole` หรือ `ClusterRoleBinding` ชื่อ `prometheus` อยู่ก่อนติดตั้ง
> ให้ตรวจสอบก่อนว่าเป็นของเราหรือของเดิม ก่อนจะลบ

## 1. ถอน Grafana

ลบ External Service ที่สร้างแยกจาก Helm โดยเลือกตาม cluster:

```bash
# cluster 1
kubectl delete -f 14-grafana-external-service-clus1.yaml --ignore-not-found
# cluster 2 ใช้ไฟล์ 14-grafana-external-service-clus2.yaml แทน
```

```bash
helm uninstall my-grafana -n grafana
```

> ⚠️ คำสั่งนี้ลบข้อมูล Grafana **ถาวร**
> PVC และ PV จะหายไปด้วย เพราะ chart ไม่ได้ตั้ง `helm.sh/resource-policy: keep`
> และ StorageClass เป็น `reclaimPolicy: Delete`
> dashboard และ data source ที่สร้างผ่าน UI จะหายไปด้วย

ตรวจ:

```bash
kubectl -n grafana get pvc,pod,deploy,svc,secret
```

ต้องไม่เหลืออะไร

## 2. ถอน kube-state-metrics

```bash
helm uninstall trident-protect -n prometheus
```

> ⚠️ **มี Helm release ชื่อ `trident-protect` อยู่ 2 ตัว**
>
> | release | namespace | คืออะไร | ต้องลบไหม |
> |---|---|---|---|
> | `trident-protect` | `prometheus` | kube-state-metrics ที่เราติดตั้ง | ✅ ลบ |
> | `trident-protect` | `trident-protect` | Trident Protect ตัวจริง | ❌ **ห้ามลบ** |
>
> คำสั่ง `-n prometheus` เป็นตัวกำหนด namespace จึงปลอดภัย
> แต่ถ้าลืมใส่ `-n` จะไปลบ Trident Protect ตัวจริงทันที

ลบ: Deployment, Service, ServiceMonitor, ConfigMap, ServiceAccount, ClusterRole, ClusterRoleBinding

> ServiceMonitor ของ KSM จะหายไปในขั้นนี้ แต่ ServiceMonitor `trident-sm` ที่เรา apply เอง
> ยังอยู่ ต้องลบในขั้นที่ 3

## 3. ถอน resource ที่ apply ด้วย kubectl

ไฟล์เหล่านี้ระบุ namespace ไว้ในตัวแล้ว จึงไม่ต้องใส่ `-n`

```bash
kubectl delete -f 08-prometheus-alert.yaml --ignore-not-found
kubectl delete -f 08-prometheus.yaml --ignore-not-found
```

```bash
kubectl delete -f 20-alert-rules.yaml --ignore-not-found
kubectl delete -f 19-alertmanager-external-service.yaml --ignore-not-found
kubectl delete -f 19-alertmanager.yaml --ignore-not-found
kubectl delete -f 19-alertmanager-config.yaml --ignore-not-found
```

```bash
kubectl delete -f 13-trident-servicemonitor.yaml --ignore-not-found
```

External Service ใช้ไฟล์ต่างกันตาม cluster:

```bash
# k8s-clus1
kubectl delete -f 10-prometheus-external-service.yaml --ignore-not-found
```

```bash
# k8s-clus2
kubectl delete -f 10-prometheus-external-service-clus2.yaml --ignore-not-found
```

```bash
kubectl delete -f 07-prometheus-rbac.yaml --ignore-not-found
```

ลบ: Prometheus CR, ServiceMonitor `trident-sm`, Service `prometheus-external`,
ServiceAccount + ClusterRole + ClusterRoleBinding `prometheus`

## 4. ถอน Prometheus Operator

มาจาก `bundle.yaml` จึงไม่ใช่ Helm release และ `helm uninstall` ลบไม่ได้

```bash
kubectl -n prometheus delete deployment/prometheus-operator
```

```bash
kubectl -n prometheus delete service/prometheus-operator
```

```bash
kubectl -n prometheus delete serviceaccount/prometheus-operator
```

```bash
kubectl delete clusterrolebinding prometheus-operator
```

```bash
kubectl delete clusterrole prometheus-operator
```

## 5. ถอน CRD ทั้ง 10 ตัว

```bash
kubectl get crd -o name | grep monitoring.coreos.com | xargs -r kubectl delete
```

จะลบ:

```text
alertmanagerconfigs.monitoring.coreos.com
alertmanagers.monitoring.coreos.com
podmonitors.monitoring.coreos.com
probes.monitoring.coreos.com
prometheusagents.monitoring.coreos.com
prometheuses.monitoring.coreos.com
prometheusrules.monitoring.coreos.com
scrapeconfigs.monitoring.coreos.com
servicemonitors.monitoring.coreos.com
thanosrulers.monitoring.coreos.com
```

## 6. ถอน namespace

```bash
kubectl delete namespace prometheus
```

```bash
kubectl delete namespace grafana
```

```bash
kubectl delete namespace mailpit
```

## 7. ตรวจว่าสะอาด

```bash
helm list -n prometheus
```

```bash
helm list -n grafana
```

> ทั้งสองคำสั่งต้องไม่มี release เหลือ
> ถ้าใช้ `helm list -A` จะเห็น release `trident` และ `trident-protect` ที่อยู่ใน namespace
> `trident` และ `trident-protect` ซึ่งเป็นของเดิมและ**ไม่ได้ถูกลบ** ไม่ใช่ว่า cleanup ไม่สมบูรณ์

```bash
kubectl get clusterrole,clusterrolebinding -o name \
  | grep -E '/(prometheus|prometheus-operator|trident-protect-kube-state-metrics)$'
```

```bash
kubectl get crd -o name | grep monitoring.coreos.com
```

```bash
kubectl get ns | grep -E 'prometheus|grafana|mailpit'
```

ทั้ง 5 คำสั่งต้องไม่มีผลลัพธ์

## 8. ทางเลือก: ลบ Nginx test app (เฉพาะ k8s-clus2)

```bash
kubectl delete -f 12-nginx-test-app-clus2.yaml
```

ลบ: PVC `nginx-data`, Deployment `nginx`, Service `nginx` ใน namespace `demo203-dr`

> ⚠️ **อย่าลบ namespace `demo203-dr`**
> namespace นี้มี Application `demo203` ของ Trident Protect
> และเป็นต้นทางของ AMR `amr-demo203` ที่ replicate ไปยัง `k8s-clus1`
> ลบแล้วจะทำให้การทดสอบ AMR หยุดทำงาน

## ไฟล์ที่ต่างกันระหว่างสอง cluster

| Resource | `k8s-clus1` | `k8s-clus2` |
|---|---|---|
| External Service | `10-prometheus-external-service.yaml` | `10-prometheus-external-service-clus2.yaml` |
| Grafana values | `14-grafana-values.yaml` | `14-grafana-values-clus2.yaml` |
| Grafana External Service | `14-grafana-external-service-clus1.yaml` | `14-grafana-external-service-clus2.yaml` |
| Grafana dashboard | `17-trident-protect-dashboard.json` | `17-trident-protect-dashboard-clus2.json` |
| Blog dashboard | `16-grafana-dashboard.json` | `16-grafana-dashboard.json` (เหมือนกัน) |

เหตุผลที่ต่างกัน:

| ค่า | `k8s-clus1` | `k8s-clus2` |
|---|---|---|
| StorageClass | `sc-clus1-nfs` | `sc-clus2-nfs` |
| MetalLB pool | `external-pool` = 10.10.110.101-120 | `ai-pool` = 10.10.110.121-140 |

สองช่วง IP ไม่ทับกัน จึง deploy ทั้งสอง cluster พร้อมกันได้

## สิ่งที่ไม่ถูกลบ

ขั้นตอนนี้ไม่แตะ:

- Trident และ Trident Protect ใน namespace `trident` และ `trident-protect`
  (Helm release `trident` และ `trident-protect` ใน namespace ของตัวเอง)
- AppVault, Application, Snapshot, ExecHooksRun, AppMirrorRelationship ทั้งหมด
- namespace `demo203-dr` และ `demo203` บน `k8s-clus2`
- MetalLB และ IPAddressPool
- StorageClass และ VolumeSnapshotClass
