# 16 - Add Grafana Dashboard

Status: NOT TESTED

Blog: "Add a dashboard for the Trident protect metrics"

ไฟล์ `16-grafana-dashboard.json` เตรียมไว้ให้แล้ว โดยดึงจากตัวอย่างของ Blog และแก้ 2 จุด

## ไฟล์ dashboard

```text
title    = Trident Protect Global view
uid      = ce54kquleoe80c
panels   = 6
```

## สิ่งที่แก้จากตัวอย่างของ Blog

| จุด | เดิม | ใหม่ | เหตุผล |
|---|---|---|---|
| value mapping (2 จุด) | `"Failed"` | `"Error"` | status ของ Trident Protect ใน cluster นี้เป็น `Error` ไม่ใช่ `Failed` |
| tags | `['24.10','25.02','25.06','25.10']` | เพิ่ม `'26.06'` | Trident ใน cluster นี้เป็น 26.06.0 |

query ทั้ง 6 panel ใช้ metric ที่ระบบเรามีอยู่แล้ว:

```promql
count(kube_customresource_snapshot_info) by (status)
count(kube_customresource_backup_info) by (status)
count(kube_customresource_snapshot_info)
count(kube_customresource_backup_info)
kube_customresource_snapshot_info
kube_customresource_backup_info
```

## 1. ดาวน์โหลดไฟล์ dashboard (ถ้ายังไม่มี)

```bash
curl -sSL \
  "https://raw.githubusercontent.com/YvosOnTheHub/LabNetApp/master/Kubernetes_v6/Trident_Protect_Scenarios/Scenario04/dashboard.json" \
  -o /tmp/dashboard-original.json
```

ไฟล์ที่ต้อง import คือ `16-grafana-dashboard.json` ในโฟลเดอร์นี้ (แก้ไว้เรียบร้อย)

## 2. Import ใน Grafana UI

```text
Dashboards → New → Import → Upload JSON file
```

เลือกไฟล์ `16-grafana-dashboard.json` แล้วกด `Load`

> ถ้า Grafana ถามให้เลือก data source ให้เลือก `Prometheus` ที่สร้างใน `15-grafana-datasource.md`
> เพราะ JSON อ้างอิง datasource uid `prometheus`

กด `Import`

## 3. ตรวจผล

Expected:

```text
Dashboard "Trident Protect Global view" imported
```

เปิด dashboard ต้องเห็น 6 panel:

| Panel | ต้องเห็น |
|---|---|
| Snapshots by status | กราฟแท่ง แยกตาม `status` (Completed = เขียว, Error = แดง) |
| Backups by status | กราฟแท่ง แยกตาม `status` |
| Total Number of Snapshots | ตัวเลข snapshot ทั้งหมด |
| Total Number of Backups | ตัวเลข backup ทั้งหมด |
| Last 10 Snapshots | ตาราง snapshot 10 รายการล่าสุด |
| Last 10 Backups | ตาราง backup 10 รายการล่าสุด |

ตัวเลขต้องตรงกับ Prometheus:

```promql
count(kube_customresource_snapshot_info)
count(kube_customresource_backup_info)
```
