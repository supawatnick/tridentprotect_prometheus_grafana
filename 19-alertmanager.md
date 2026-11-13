# 19 - Alertmanager และ Email Alert

Status: TESTED

คลัสเตอร์ : `k8s-clus1`
namespace : `prometheus` (Alertmanager, Prometheus, PrometheusRule), `mailpit` (Mailpit)

เพิ่มการแจ้งเตือนจาก Prometheus ผ่าน Alertmanager ไปยังอีเมล

> ใน lab ใช้ Mailpit เป็น SMTP ปลายทางและดูอีเมลผ่าน web UI
> ใน production ให้เปลี่ยน `smtp_smarthost` และเพิ่ม credential ในไฟล์ `19-alertmanager-config.yaml`

## ไฟล์ที่เกี่ยวข้อง

| ไฟล์ | หน้าที่ |
|---|---|
| `21-mailpit.yaml` | debug SMTP server + web UI |
| `19-alertmanager-config.yaml` | Secret ที่มี `alertmanager.yaml` (route + email receiver) |
| `19-alertmanager.yaml` | ServiceAccount + Alertmanager CR |
| `19-alertmanager-external-service.yaml` | เปิด Alertmanager UI ผ่าน MetalLB |
| `08-prometheus-alert.yaml` | Prometheus CR ที่เปิด `alerting` และ `ruleSelector` |
| `20-alert-rules.yaml` | PrometheusRule ของ Backup, Snapshot และ AppVault |

## 1. ติดตั้ง Mailpit

```bash
kubectl apply -f 21-mailpit.yaml
```

```bash
kubectl -n mailpit rollout status deployment/mailpit --timeout=180s
```

ดู IP ของ web UI:

```bash
kubectl -n mailpit get service mailpit-http \
  -o jsonpath='{.status.loadBalancer.ingress[0].ip}{"\n"}'
```

เปิด `http://<EXTERNAL-IP>:8025`

## 2. ใส่ config ของ Alertmanager

```bash
kubectl apply -f 19-alertmanager-config.yaml
```

> ต้องใส่ Secret ก่อน Alertmanager CR เพื่อไม่ให้ช่วงแรก operator
> สร้าง config ขั้นต่ำที่ไม่มี receiver

## 3. ติดตั้ง Alertmanager

```bash
kubectl apply -f 19-alertmanager.yaml
```

```bash
kubectl -n prometheus rollout status statefulset/alertmanager-alertmanager --timeout=300s
```

> ชื่อ StatefulSet คือ `alertmanager-<ชื่อ CR>` ซึ่ง Operator ตั้งจาก
> `prefixedName()` ใน `pkg/alertmanager/statefulset.go`
> CR ชื่อ `alertmanager` จึงได้ StatefulSet ชื่อ `alertmanager-alertmanager`
> และ Secret config ที่ Operator สร้างชื่อ `alertmanager-alertmanager-generated`
> ตรวจจาก `kubectl -n prometheus get statefulset,secret | grep alertmanager`

ตรวจว่า Alertmanager ตั้งค่าแล้ว:

```bash
kubectl -n prometheus get alertmanager alertmanager -o wide
```

## 4. เปิด Alertmanager UI ผ่าน MetalLB

```bash
kubectl apply -f 19-alertmanager-external-service.yaml
```

```bash
kubectl -n prometheus get service alertmanager-external \
  -o jsonpath='{.status.loadBalancer.ingress[0].ip}{"\n"}'
```

เปิด `http://<EXTERNAL-IP>:9093`

## 5. เชื่อม Prometheus เข้ากับ Alertmanager

```bash
kubectl apply -f 08-prometheus-alert.yaml
```

> ไฟล์นี้เป็น superset ของ `08-prometheus.yaml` จึงต้อง apply ทับ
> และต้องทำหลังขั้นที่ 3 เสมอ
> ถ้า apply `08-prometheus.yaml` ทีหลังจะทำให้ `alerting` และ `ruleSelector` หายไป

ตรวจว่า Prometheus ต่อ Alertmanager สำเร็จ:

```bash
curl -s http://<PROMETHEUS-IP>:9090/api/v1/alertmanagers | jq '.data.activeAlertmanagers'
```

Expected:

```json
[
  {
    "url": "http://<POD-IP>:9093/api/v2/alerts"
  }
]
```

## 6. ใส่ alert rules

```bash
kubectl apply -f 20-alert-rules.yaml
```

ตรวจว่า rule ถูกโหลด:

```bash
curl -s http://<PROMETHEUS-IP>:9090/api/v1/rules | jq '.data.groups[].rules[].name'
```

Expected:

```text
BackupFailed
BackupCompleted
SnapshotFailed
SnapshotCompleted
AppVaultError
AMRError
AMRNotEstablished
AMRTransferCompleted
```

## 7. ทดสอบ

### 7.1 ทดสอบ pipeline ด้วย rule ชั่วคราว

เพิ่ม rule ที่ fire ทันทีชั่วคราว:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: pipeline-test
  namespace: prometheus
spec:
  groups:
    - name: test
      rules:
        - alert: PipelineTest
          expr: vector(1)
          for: 10s
          labels:
            severity: info
          annotations:
            summary: "Pipeline test"
```

```bash
kubectl apply -f - <<'EOF'
<เนื้อหาข้างบน>
EOF
```

รอประมาณ 1-2 นาที แล้วตรวจ:

```bash
curl -s http://<PROMETHEUS-IP>:9090/api/v1/alerts | jq '.data.alerts[] | {name:.labels.alertname,state:.state}'
```

ดูอีเมลใน Mailpit UI ต้องมี `PipelineTest` ที่ subject ขึ้นต้นด้วย `[TRIDENT PROTECT] FIRING PipelineTest`

ลบ rule ชั่วคราว:

```bash
kubectl delete prometheusrule pipeline-test -n prometheus
```

### 7.2 ทดสอบด้วย backup จริง

```bash
tridentctl-protect create backup <ชื่อ backup> --app demo201 --appvault k8s-clus1-backup -n demo201
```

เมื่อ backup เสร็จ `BackupCompleted` จะ fire หนึ่งครั้ง และจะได้อีเมล

> ไม่ควรมี Backup ที่อยู่ในสถานะ Error ใน cluster ตอนนี้
> ถ้ามี จะได้อีเมล `BackupFailed` ด้วย

## 8. ปรับเป็น production

สิ่งที่ต้องแก้มี 3 จุด อยู่ในไฟล์ `19-alertmanager-config.yaml` ไฟล์เดียว

| สิ่งที่เปลี่ยน | ฟิลด์ | ค่าใน lab | ตัวอย่างค่าใน production |
|---|---|---|---|
| SMTP server | `global.smtp_smarthost` | `mailpit-smtp.mailpit.svc:1025` | `smtp.corp.example.com:587` |
| คนส่ง (From) | `global.smtp_from` | `alertmanager@trident-protect.local` | `alertmanager@example.com` |
| คนรับ (To) | `receivers[].email_configs[].to` | `trident-protect@example.com` | `noc@example.com, sre-team@example.com` |

เพิ่มเติมที่ควรตั้งเมื่อ relay ต้อง authentication

```yaml
global:
  smtp_auth_username: 'alertmanager@example.com'
  smtp_auth_password: '<SMTP_PASSWORD>'
  smtp_require_tls: true
  smtp_hello: 'alertmanager@example.com'
```

หมายเหตุเรื่องพอร์ต

| พอร์ต | วิธีเข้ารหัส |
|---|---|
| `25` | ไม่เข้ารหัส (relay ภายในเครื่องเดียวกัน) |
| `587` | STARTTLS — ควรใช้ค่านี้ |
| `465` | implicit TLS — ต้องตั้ง `force_implicit_tls: true` |

`smtp_hello` ค่า default ของ Alertmanager คือ `localhost` ซึ่ง relay ส่วนใหญ่จะ reject
ควรตั้งเป็น FQDN ของเครื่อง Alertmanager

### เก็บ password อย่างปลอดภัย

ห้าม commit password จริงลง Git ให้เก็บเป็นไฟล์แทน โดย mount Secret ผ่าน `spec.secrets`
ของ Alertmanager CR (ไฟล์ `19-alertmanager.yaml`)

```yaml
spec:
  secrets:
    - smtp-credentials
```

Operator จะ mount ไว้ที่ `/etc/alertmanager/secrets/smtp-credentials`
แล้วอ้างด้วย `smtp_auth_password_file` ใน `19-alertmanager-config.yaml`

```yaml
global:
  smtp_auth_username: 'alertmanager@example.com'
  smtp_auth_password_file: '/etc/alertmanager/secrets/smtp-credentials/password'
```

สร้าง Secret นอก Git

```bash
kubectl -n prometheus create secret generic smtp-credentials \
  --from-literal=password='<SMTP_PASSWORD>'
```

### อื่น ๆ ที่ควรปรับใน production

ดูรายละเอียดในไฟล์ `19-alertmanager.yaml` ที่บล็อก `⚠️ PRODUCTION`

```text
replicas: 3                 HA
storage:                    alert และ silence ไม่หายเมื่อ pod restart
retention: 120h             เก็บประวัติย้อนหลังได้นานขึ้น
externalUrl:                ลิงก์ในอีเมลชี้ถูกที่
web.tlsConfig:              เปิด UI ผ่าน HTTPS
```

### ตรวจหลังแก้

```bash
kubectl apply -f 19-alertmanager-config.yaml
```

ไม่ต้อง restart pod Operator mount Secret นี้ไว้ที่ `/etc/alertmanager/config`
และ config-reloader จะ reload ให้อัตโนมัติภายในประมาณ 1 นาที

```bash
kubectl -n prometheus logs alertmanager-alertmanager-0 -c alertmanager --tail=20
```

ต้องไม่มี error และเห็นข้อความ reload config

## 9. รูปแบบอีเมล

ไฟล์ `19-alertmanager-config.yaml` ใส่ `text:` และ `html:` ไว้ใน `email_configs[0]`
เพื่อแทน template default ของ Alertmanager

### ทำไมต้องเปลี่ยน

template default แสดง **label ทั้งหมด** ของทุก alert รวมถึงที่ไม่มีความหมายสำหรับคนอ่าน

```text
container = kube-state-metrics
customresource_group = protect.trident.netapp.io
customresource_kind = Snapshot
customresource_version = v1
endpoint = http
instance = 10.244.38.248:8080
job = kube-state-metrics
namespace = prometheus
pod = trident-protect-kube-state-metrics-558985498c-rzplv
prometheus = prometheus/prometheus
service = trident-protect-kube-state-metrics
```

บวกกับ label `conditions` ของ AppMirrorRelationship ที่ยาวราว 2 KB
ทำให้อีเมลใหญ่มากและอ่านยาก

### แจ้งเตือนเฉพาะรอบปัจจุบัน

อีเมลแบบ default ของ Alertmanager จะสะสม alert ที่ผ่านมาแล้วไว้ในกลุ่มเดียวกัน
ทำให้ทุกฉบับมีประวัติของรอบก่อน ๆ ติดมาด้วย สิ่งที่ทำเพื่อแก้:

| ที่แก้ | ผล |
|---|---|
| `20-alert-rules.yaml` เปลี่ยน `offset 10m` → `offset 5m` | alert อยู่ในสถานะ firing 5 นาทีพอดี เท่ากับรอบของ schedule → firing เหลือ 1 รอบ (3 alerts) แทน 2 รอบ (6 alerts) |
| template แสดงเฉพาะ `.Alerts.Firing` | ตัดประวัติของ alert ที่ resolve แล้วออก |
| แยก route ตาม `severity` | event (`info`) แจ้งเฉพาะรอบปัจจุบัน ส่วนปัญหา (`critical`/`warning`) แจ้งด้วยเมื่อหายแล้ว |

```yaml
route:
  group_by: ['alertname']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 12h
  receiver: email-alert
  routes:
    - receiver: email-event
      matchers:
        - 'severity="info"'
```

| receiver | ใช้กับ | `send_resolved` | เนื้อหา |
|---|---|---|---|
| `email-event` | `severity=info` (SnapshotCompleted, BackupCompleted, AMRTransferCompleted) | `false` | เฉพาะ event ของรอบปัจจุบัน |
| `email-alert` | `severity=critical\|warning` (BackupFailed, SnapshotFailed, AppVaultError, AMRError, AMRNotEstablished) | `true` | แจ้งด้วยเมื่อปัญหาหายแล้ว |

### รูปแบบที่ได้

event ปกติ — มีแค่ของรอบเดียว

```text
Snapshot completed
13/11/2026 11:40 (Asia/Bangkok)

COMPLETED  custom-48daa-20261113043500
   App        auto-auto-backup-01
   Namespace  auto-backup-01
   Task       Snapshot
   Status     Completed
   Time       13/11/2026 11:35

COMPLETED  custom-1fb8e-20261113043500
   App        auto-auto-backup-02
   Namespace  auto-backup-02
   Task       Snapshot
   Status     Completed
   Time       13/11/2026 11:35

COMPLETED  custom-f91a1-20261113043500
   App        auto-auto-backup-03
   Namespace  auto-backup-03
   Task       Snapshot
   Status     Completed
   Time       13/11/2026 11:35

View all: http://10.10.110.114:9093/#/alerts?receiver=email-event
```

ปัญหาที่เกิดแล้วหาย

```text
Backup failed
A backup has failed.
13/11/2026 11:51 (Asia/Bangkok)

CLEARED  zz-recover-test
   App        zz-app
   Namespace  zz-ns
   Task       Backup
   Status     Error
   Time       13/11/2026 11:40
   Error      Failed to create snapshot: bucket unreachable

View all: http://10.10.110.114:9093/#/alerts?receiver=email-alert
```

Subject: `[Trident Protect] Snapshot completed - 3 events`

### badge ที่ใช้

| badge | เงื่อนไข | สี |
|---|---|---|
| `COMPLETED` | alert ที่ยังเกิดอยู่ และชื่อไม่เข้าเงื่อนไขของ failure | เขียว `#1e8e3e` |
| `FAILED` | ชื่อ alert ตรงกับ `Failed`\|`Error`\|`NotEstablished` | แดง `#d93025` |
| `CLEARED` | alert ที่ resolve แล้ว (แสดงเฉพาะใน receiver `email-alert`) | เทา `#5f6368` |

### รายละเอียดที่แสดง

| ช่อง | ที่มาของข้อมูล |
|---|---|
| ชื่อ object | `backup_name` / `snapshot_name` / `appvault_name` / `amr_name` |
| App | `appReference`; AMR ใช้ `sourceApplicationName -> destinationApplicationRef` |
| Namespace | `cr_namespace` (ดูหัวข้อ "namespace ของ CR" ด้านล่าง) |
| Task | `customresource_kind`; `AppMirrorRelationship` แสดงเป็น `Replication`; ExecHooksRun แสดง `<stage>-<action> hook` |
| Status | label `status` ถ้าไม่มีใช้ `state` |
| Time | `creation_time` ของ CR (เวลาที่งานเกิดขึ้นจริง) แปลงด้วย `toDate` แล้ว format ด้วย `date`; AMR ใช้ `lastTransferredSnapshotCompletionTimestamp`; ถ้าไม่มีใช้เวลาที่ alert ถูกตรวจเจอ |
| Error | label `error` (`status.error`) แสดงเฉพาะเมื่อไม่ว่าง |

หัวเรื่องและคำอธิบายของอีเมลใช้ annotation `summary` และ `description` ของ alert
ถ้าไม่มี annotation จะ fallback เป็นชื่อ alert

### namespace ของ CR

label `namespace` บน metric ของ custom resource **ไม่ใช่** namespace ของ CR
เพราะ Prometheus Operator เติม label นี้ให้เสมอจาก namespace ของ target ที่ scrape

```yaml
# config ที่ Operator generate ให้
- source_labels: [__meta_kubernetes_namespace]
  target_label: namespace
  action: replace
```

แม้เอกสารของ kube-state-metrics จะแนะนำให้ map `namespace: [metadata, namespace]`
แต่จะถูก Operator เขียนทับเสมอ

วิธีที่ใช้จริงคือ map เป็นชื่ออื่นในไฟล์ `05-ksm-values.yaml`

```yaml
          labelsFromPath:
            backup_name: [metadata, name]
            creation_time: [metadata, creationTimestamp]
            cr_namespace: [metadata, namespace]      # ← ชื่อนี้
```

ผลที่ได้

```text
kube_customresource_backup_info     cr_namespace = auto-backup-01, auto-backup-02, demo201
kube_customresource_appvault_info   cr_namespace = trident-protect
kube_customresource_app_mirror_relationship_info   cr_namespace = demo203
```

> หลังแก้ `05-ksm-values.yaml` ต้อง `helm upgrade` แล้ว `rollout restart` kube-state-metrics
> เพราะมันอ่าน config ตอน startup เท่านั้น และรอประมาณ 5 นาทีให้ series เก่าหมดอายุ

### กฎที่ใช้ใน template

| เรื่อง | ค่า |
|---|---|
| เวลา | `$tz := "Asia/Bangkok"` เปลี่ยนเป็น `"UTC"` ได้ที่จุดเดียว |
| จำนวน firing ที่แสดง | สูงสุด 20 รายการ |
| จำนวน resolved ที่แสดง | สูงสุด 3 รายการ (เฉพาะ receiver `email-alert`) |
| ลำดับ | firing อยู่ก่อน resolved เสมอ |
| Subject | `[Trident Protect] {{ summary }} - {{ count }} event(s)` |

### ข้อควรระวัง

| เรื่อง | รายละเอียด |
|---|---|
| template เขียนซ้ำ 2 ชุด | receiver ทั้งสองใช้ template เดียวกันแต่ YAML block scalar ใช้ anchor ไม่ได้ จึงต้องเขียนซ้ำในไฟล์เดียว **ต้องแก้ให้ตรงกันทั้งสองชุด** |
| `send_resolved` ไม่ได้ตัดเนื้อหาอีเมล | flag นี้แค่ไม่ trigger การแจ้งเตือนตอน alert resolve เท่านั้น template ยังได้รับทั้ง firing และ resolved เสมอ ต้องซ่อนใน template เอง |
| `offset` ต้องเท่ากับรอบของ schedule | ถ้า schedule ทุก 10 นาที ต้องใช้ `offset 10m` มิฉะนั้นอีเมลจะรวมหลายรอบไว้ด้วยกัน |
| alert มีชีวิตสั้นลง | หลังลด `offset` เหลือ 5 นาที alert จะอยู่ได้ 5 นาที (~20 รอบ evaluate ที่ scrape ทุก 15 วินาที) ต้องไม่ให้ Prometheus restart ในช่วงนั้น |
| ไม่มีเลขกำกับ | Alertmanager 0.34.2 ไม่มีฟังก์ชันบวกลบ (`add`/`sub`) ใน template จึงข้ามเลขกำกับ |
| `amtool check-config` ไม่ตรวจ inline template | ตรวจได้แค่ schema ของ config ส่วน `text:` และ `html:` จะ parse ตอนส่งอีเมลเท่านั้น ต้องทดสอบด้วยอีเมลจริง |
| นับ `{{ end }}` ให้ครบ | template ที่ซับซ้อนมีโอกาสเพิ่ม `{{ end }}` เกิน ทำให้ Alertmanager ตอบ error ตอนส่งอีเมล ไม่ใช่ตอนโหลด config |
| ช่อง Error อาจไม่มีค่า | label `error` ถูก KSM ตัดทิ้งเมื่อค่าว่าง ตอนนี้ใน cluster ยังไม่มี CR ที่ error จึงยังไม่เห็นค่าในช่องนี้ |
| ลิงก์ View all ชี้ URL ภายใน | ตั้ง `spec.externalUrl` ใน `19-alertmanager.yaml` เป็น `http://10.10.110.114:9093` (IP ที่ MetalLB จ่ายให้ `alertmanager-external`) ถ้า Service ถูกลบแล้วสร้างใหม่อาจได้ IP อื่น |
| ทดสอบ template โดยไม่รอ alert จริง | `amtool alert add --alertmanager.url=http://127.0.0.1:9093 --end=<เวลาในอนาคต> alertname=X key=value` ใน pod แล้วรอ notification (alert ที่แทรกจะหมดอายุเอง) |

## ข้อควรระวัง

| เรื่อง | รายละเอียด |
|---|---|
| `name: alertmanager-operated` | ต้องตรงกับชื่อ Service แบบ exact match เพราะ Operator สร้าง relabel `keep` ด้วย anchored regex; ชื่อ Alertmanager CR คือ `alertmanager` แต่ไม่ใช่ชื่อ Service |
| `port: web` | ต้องตรงกับชื่อ port บน Service `alertmanager-operated`; Operator ใช้ `web` เป็นค่า default ของ `spec.portName` |
| `ruleSelector: {}` | ค่า null จะไม่เลือก PrometheusRule ใดเลย โดยไม่มี error |
| `ruleNamespaceSelector: {}` | ค่า null (default) เลือกแค่ namespace ตัวเอง |
| Operator ไม่ auto-configure | `spec.alerting` ต้องตั้งเองเสมอ ไม่มีการค้นหา Alertmanager อัตโนมัติ |
| `unless ... offset 5m` | ใช้เพื่อแจ้งเฉพาะ backup ที่เพิ่งเกิด ถ้าใช้ expression ตรงๆ จะแจ้งของเก่าทั้งหมดที่ยังอยู่ ค่า offset ต้องเท่ากับรอบของ schedule |
| Mailpit | ใช้ได้เฉพาะ lab ห้ามใช้ใน production |

## Cleanup

ดูรายละเอียดวิธีถอนทั้งหมดใน `18-cleanup-uninstall.md`
