# 15 - Add Grafana Data Source

Status: NOT TESTED

Blog: "Add a data source"

เพิ่ม Prometheus ที่ติดตั้งด้วย Prometheus Operator เป็น data source ของ Grafana ผ่าน UI

## 1. ตรวจ URL ของ Prometheus

Service ของ Prometheus ที่ Operator สร้างชื่อ `prometheus-operated`

```bash
kubectl -n prometheus get service | grep operated
```

Expected:

```text
prometheus-operated   ClusterIP   None   <none>   9090/TCP
```

## 2. เพิ่ม data source ใน Grafana UI

เปิด Grafana แล้วทำตามลำดับ:

```text
Connections → Data sources → Add new data source → Prometheus
```

| ช่อง | ค่า |
|---|---|
| Name | `Prometheus` |
| URL | `http://prometheus-operated.prometheus.svc.cluster.local:9090` |

กด `Save & test`

Expected:

```text
Successfully queried the Prometheus API.
```

## 3. ตรวจว่า datasource ใช้งานได้

เปิดเมนู `Drilldown` หรือ `Explore` ใน Grafana แล้วเลือก data source `Prometheus`
แล้ว query:

```promql
kube_customresource_backup_info
```

ตรวจจำนวนและ labels ให้ตรงกับ Backup CR จริง; ถ้าไม่มี Backup อาจไม่มี series นี้

```promql
kube_customresource_app_mirror_relationship_info
```

ตรวจให้ตรงกับ AppMirrorRelationship CR จริง; ถ้าไม่มี AMR อาจไม่มี series นี้

## ถัดไป

`16-grafana-dashboard.md` (import dashboard)
