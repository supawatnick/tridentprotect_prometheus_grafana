# 17 - Custom Grafana Dashboard

Status: NOT TESTED (ยังไม่ได้ import รอบที่แก้ไข)

Custom dashboard สำหรับดูสถานะ Trident Protect: Backup, Snapshot, AppVault และ AppMirrorRelationship

## ไฟล์ dashboard

| ไฟล์ | ใช้กับ | ลิงก์ Open Prometheus |
|---|---|---|
| `17-trident-protect-dashboard.json` | `k8s-clus1` | `http://10.10.110.111:9090` |
| `17-trident-protect-dashboard-clus2.json` | `k8s-clus2` | `http://10.10.110.121:9090` |

สองไฟล์เนื้อหาเหมือนกันทุกอย่าง ต่างกันแค่ URL ของ Prometheus

```text
title   = Trident Protect Overview
uid     = trident-protect-overview
panels  = 27 (แบ่ง 6 rows)
tags    = ["trident-protect", "trident", "netapp"]
refresh = 30s
time    = now-24h -> now
```

## ตัวแปร (Template variables)

| ตัวแปร | ชนิด | ค่า | ใช้กับ |
|---|---|---|---|
| `DS_PROMETHEUS` | datasource | `prometheus` | ทุก panel ตอน import จะถามเลือก datasource |
| `app` | query | `label_values({__name__=~"kube_customresource_(backup|snapshot)_info"}, appReference)` | กรอง panel Backup/Snapshot — เลือกได้หลายค่า + All |
| `amr` | query | `label_values(kube_customresource_app_mirror_relationship_info, amr_name)` | กรอง panel AMR — เลือกได้หลายค่า + All |
| `vault` | query | `label_values(kube_customresource_appvault_info, appvault_name)` | กรอง panel AppVault — เลือกได้หลายค่า + All |

ค่าที่มีจริงในแต่ละ cluster:

```text
k8s-clus1  app = auto-auto-backup-01, auto-auto-backup-02, auto-auto-backup-03, auto-auto-backup-05, demo201
k8s-clus2  app = demo203
            amr = amr-demo202, auto-amr-k8s-clus1-auto-backup-01-p0 ... -05-p1
            vault = k8s-clus1-backup, k8s-clus1-source-manifests, k8s-clus2-dest-manifests, k8s-clus2-source-manifests
```

> ตัวแปร `app` ใช้ selector ที่รวม backup และ snapshot เพื่อให้ใช้ได้ทั้งสอง cluster
> บน `k8s-clus2` ไม่มี Backup CR เลย ถ้าใช้แค่ `kube_customresource_backup_info` ตัวแปรจะว่าง
> และ panel ที่กรองด้วย `{appReference=~"$app"}` จะขึ้น No data

## Panel ทั้งหมด

### Row 1 — Trident Protect Summary

| Panel | Query | ค่าปัจจุบัน | สี |
|---|---|---:|---|
| Total Backups | `count(kube_customresource_backup_info{appReference=~"$app"})` | 18 | เขียว |
| Total Snapshots | `count(kube_customresource_snapshot_info{appReference=~"$app"})` | 76 | เขียว |
| Failed Backups | `count(kube_customresource_backup_info{status="Error",appReference=~"$app"}) or vector(0)` | 0 | **แดงเมื่อ > 0** |
| Failed Snapshots | `count(kube_customresource_snapshot_info{status="Error",appReference=~"$app"}) or vector(0)` | 0 | **แดงเมื่อ > 0** |
| Running Backups | `count(kube_customresource_backup_info{status="Running"}) or vector(0)` | 0 | เหลืองเมื่อ > 0 |
| AppVaults Available | `count(kube_customresource_appvault_info{state="Available",appvault_name=~"$vault"})` | 4 | เขียว |
| AppVaults in Error | `count(kube_customresource_appvault_info{state="Error"}) or vector(0)` | 0 | **แดงเมื่อ > 0** |
| AMR Established | `count(kube_customresource_app_mirror_relationship_info{state="Established"})` | 1 | เขียว |

### Row 2 — Status Breakdown

| Panel | Query |
|---|---|
| Backups by status | `count by (status) (kube_customresource_backup_info{appReference=~"$app"})` |
| Snapshots by status | `count by (status) (kube_customresource_snapshot_info{appReference=~"$app"})` |
| AppVaults by state | `count by (state) (kube_customresource_appvault_info{appvault_name=~"$vault"})` |
| AMR by state | `count by (state) (kube_customresource_app_mirror_relationship_info)` |

### Row 3 — Growth Over Time

| Panel | Query |
|---|---|
| Backups over time | `count(kube_customresource_backup_info{appReference=~"$app"})` |
| Snapshots over time | `count(kube_customresource_snapshot_info{appReference=~"$app"})` |

### Row 4 — Per Application

| Panel | Query | Legend |
|---|---|---|
| Backups per application | `count by (appReference) (kube_customresource_backup_info)` | `{{appReference}}` |
| Snapshots per application | `count by (appReference) (kube_customresource_snapshot_info)` | `{{appReference}}` |

### Row 5 — Details

| Panel | Query | คอลัมน์ |
|---|---|---|
| Latest Backups | `kube_customresource_backup_info` | Date, Application, Backup, Status, AppVault |
| Latest Snapshots | `kube_customresource_snapshot_info` | Date, Application, Snapshot, Status, AppVault |
| AppVaults | `kube_customresource_appvault_info` | AppVault, State, Error, Message |
| AMR Configuration | `kube_customresource_app_mirror_relationship_info` | Relationship, Source App, Source Vault, Dest Vault, Storage Class, Recurrence, Dest Namespace, Namespace Mapping |

### Row 6 — AppMirrorRelationship Detail

| Panel | Type | Query | ค่า |
|---|---|---|---:|
| AMRs Total | stat | `count(kube_customresource_app_mirror_relationship_info{amr_name=~"$amr"})` | 1 |
| AMRs Established | stat | `count(...{state="Established",amr_name=~"$amr"})` | 1 |
| AMRs Not Established | stat | `count(...{state!="Established",amr_name=~"$amr"}) or vector(0)` | 0 → **แดงเมื่อ > 0** |
| AMRs Promoted | stat | `count(...{desiredState="Promoted",amr_name=~"$amr"}) or vector(0)` | 0 → เหลืองเมื่อ > 0 |
| AMRs by Desired State | pie | `count by (desiredState) (...)` | |
| AMR Transfer Detail | table | Relationship, Transfer Start, Transfer End, Last Snapshot, Snapshot Done, State | |
| AMR Conditions | table | Relationship, State, Desired State, Conditions, Created | |

ทุก table เรียงตามวันที่ใหม่สุด จำกัด 10 แถว และมีลิงก์ "Explore in Prometheus" ที่ชื่อ panel

## สีสถานะ

ใช้ value mapping เดียวกับ dashboard ของ Blog:

```text
Completed   = dark-green
Available   = dark-green
Established = dark-green
Running     = super-light-yellow
Promoting   = super-light-yellow
Error       = dark-red
Failed      = dark-red
```

ผูกกับคอลัมน์ `Status` และ `State` ผ่าน `fieldConfig.overrides` + `custom.cellOptions` (โหมด `color-background`)

## Import ใน Grafana UI

```text
Dashboards -> New -> Import -> Upload JSON file
```

เลือกไฟล์ `17-trident-protect-dashboard.json` แล้วกด `Load`

> ถ้า Grafana ถามให้เลือก data source ให้เลือก `Prometheus` ที่สร้างไว้ใน `15-grafana-datasource.md`
> เพราะ dashboard ใช้ตัวแปร `${DS_PROMETHEUS}`

กด `Import`

> เนื่องจาก uid `trident-protect-overview` ถูกใช้ไปแล้ว ให้เลือก overwrite dashboard เดิม
> หรือลบ dashboard เดิมออกก่อนแล้วค่อย import ใหม่

## ตรวจผลหลัง import

| ตรวจ | ผลที่คาด |
|---|---|
| Row 1 | ตัวเลข 8 ตัว ตรงกับ Prometheus |
| Row 2 | กราฟวงกลม 4 อัน แยกตาม status/state |
| Row 3 | กราฟเส้น 2 อัน มีเส้นข้อมูล |
| Row 4 | bar gauge 2 อัน ราย application |
| Row 5 | table 4 อัน มีข้อมูลและคอลัมน์ Status มีสี |
| Row 6 | stat 4 ตัว, pie 1 อัน, table 2 อัน แสดงรายละเอียด AMR |
| ตัวแปร Application | เลือกได้ 5 ค่า + All |
| ตัวแปร Relationship | เลือกได้ 1 ค่า + All |
| ตัวแปร AppVault | เลือกได้ 4 ค่า + All |

## ข้อจำกัด

- metric ของ kube-state-metrics **ไม่มี label ของ namespace ของ CR** จึงกรองตาม namespace ของ application ไม่ได้ มีได้แค่ `appReference`
- label `namespace` บน metric หมายถึง namespace ของ pod ที่ทำการ scrape (คือ `prometheus`) ไม่ใช่ namespace ของ CR
- `creation_time` เป็น label จึงนำมาทำ arithmetic กับเวลาไม่ได้ ใช้ได้แค่เรียงลำดับ
- panel ที่ไม่มีข้อมูล (เช่น ไม่มี Backup ที่ Error) ใช้ `or vector(0)` เพื่อให้แสดง 0 แทน `No data`
- คอลัมน์ `Error` และ `Message` ของตาราง AppVaults จะปรากฏเฉพาะเมื่อมี AppVault อยู่ในสถานะ Error เท่านั้น
- ลิงก์ "Open Prometheus" ชี้ไปที่ external IP ที่ MetalLB จ่าย ถ้า Service ถูกสร้างใหม่และได้ IP อื่น ต้องแก้ในไฟล์นี้แล้ว import ใหม่

## หมายเหตุการแก้ไขรอบแรก

รอบแรก table panel ตั้ง `format: time_series` ทำให้ Prometheus คืนผลแยก frame ต่อ series
ซึ่งมีเพียง field `Time` และ `Value` แล้ว `organize` transformation กลับ exclude ทั้งสองช่อง
ทำให้ตารางไม่มีข้อมูลและแก้ได้โดยเปลี่ยนเป็น `format: table`

นอกจากนี้ `limit` transformation ใช้ `limitField` เป็น **จำนวนแถว** (ตัวเลข)
ค่าเดิม `"limit"` จะถูก `parseInt` เป็น `NaN` จึงไม่จำกัดแถวเลย แก้เป็น `10` แล้ว

## สิ่งที่ยังไม่รวม

| ตัวเลือก | Query ที่ใช้ได้ |
|---|---|
| ExecHook | `count by (stage, action, status) (kube_customresource_ehr_info)` |
| Trident CSI | `trident_volume_count`, `trident_volume_allocated_bytes`, `trident_operation_duration_milliseconds`, `trident_rest_ops_total` |
