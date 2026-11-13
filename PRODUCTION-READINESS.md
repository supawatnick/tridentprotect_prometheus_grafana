# Production readiness review

วันที่ตรวจ: 2026-11-13 (Asia/Bangkok)

> รายงานด้านล่างเป็น baseline ก่อนแก้ไข ไม่ใช่สถานะ configuration ล่าสุด
> การเปลี่ยนแปลงหลัง review: จำกัด ServiceMonitor ด้วย `monitoring: trident-protect`, แยก Grafana External Services ออกจาก values, เอา dashboard lab links และ Alertmanager email links/externalUrl ออก และแยก email groups ตาม namespace/application แล้ว
> ข้อเสนอที่เหลือยังไม่ได้ implement ทั้งหมด; ตรวจ manifests และคู่มือแต่ละ phase สำหรับขั้นตอนปัจจุบัน

ขอบเขต: ตรวจไฟล์เดิมทั้ง 32 ไฟล์แบบ static review และเทียบเอกสารทางการ ไม่ได้ deploy, ติดต่อ cluster, เปลี่ยน manifests หรือรับรอง version compatibility/CVE status ของชุดนี้

## ข้อสรุปและระดับความสำคัญ

ชุดนี้เหมาะกับ demo lab แต่ยังไม่ควรนำไป apply กับ production โดยตรง

- **P0 — ต้องแก้ก่อนใช้งาน:** จำกัดการเข้าถึง UI/API, ใช้ TLS/authentication ตามนโยบายองค์กร, เปลี่ยน SMTP lab, ป้องกัน cleanup ลบ shared resources และกำหนด persistent storage สำหรับข้อมูลที่ต้องรักษา
- **P1 — ต้องเสร็จก่อนรับรอง production:** จำกัด discovery/RBAC, แก้ alert identity และทดสอบ history gaps, แยก environment configuration, provision Grafana แบบทำซ้ำได้ และตรวจสุขภาพ monitoring เอง
- **P2 — ตาม SLO และ scale:** HA, long-term storage, capacity tuning และการปรับ dashboard/เอกสาร ทั้งนี้ HA เป็น P1 หาก SLO ต้องทน node failure

ข้อเท็จจริงคือค่าที่พบในไฟล์ ส่วนจำนวน replicas, disk, retention และเวลา alert ด้านล่างเป็นข้อเสนอ ต้องเลือกจาก SLO/RPO/RTO, workload และนโยบายองค์กร ห้ามถือว่าค่าที่เคยผ่านใน lab เป็น production sizing

## ประเด็นสำคัญที่ตรวจพบ

1. `08-prometheus*.yaml` ไม่มี `storage`, retention, version pin หรือ HA ที่ระบุชัด; มีเพียง memory request 400Mi ข้อมูล TSDB จึงไม่ถูกกำหนดให้คงอยู่ผ่าน PVC ใน manifests นี้
2. `19-alertmanager.yaml` ใช้ replica เดียว ไม่มี PVC/resources/version pin และ external URL เป็น lab IP ผ่าน HTTP การสูญเสีย persistent state กระทบ silences/notification state; Alertmanager ไม่ใช่ระบบเก็บ alert history ระยะยาว
3. `10-*`, `14-grafana-values*` และ `19-alertmanager-external-service.yaml` เปิด LoadBalancer ผ่าน HTTP โดยไม่มี access restrictions ในไฟล์เหล่านี้ ยังไม่ทราบว่ามี firewall ภายนอกหรือไม่
4. `19-alertmanager-config.yaml` ส่ง Mailpit โดย `smtp_require_tls: false` ส่วน `21-mailpit.yaml` เปิดรับ SMTP แบบไม่ใช้ credential สำหรับ lab
5. `18-cleanup-uninstall.md` และ historical log มีคำสั่งลบ CRDs ทั้ง `monitoring.coreos.com` และ namespaces การลบ CRD ลบ custom resources ของระบบอื่นที่ใช้ CRD เดียวกันได้ ห้ามใช้เป็น production teardown
6. `serviceMonitorSelector`, `serviceMonitorNamespaceSelector`, `ruleSelector`, `ruleNamespaceSelector` ในฉบับ alert เป็น `{}` ซึ่งเลือกกว้าง; `podMonitorSelector: {}` ก็เลือกกว้างภายใน namespace scope ของมัน ไม่ควรเปิดให้ resource จากทีมอื่นถูกนำมา scrape/ประเมินโดยไม่มี ownership policy
7. `20-alert-rules.yaml` ใช้ชื่ออย่างเดียวใน `unless on(backup_name)`, `on(snapshot_name)` และ AMR matching ไม่รวม namespace/UID จึงชนกันได้เมื่อมีชื่อซ้ำข้าม namespace หรือมีการสร้างใหม่ชื่อเดิม
8. Completed rules เทียบ series ปัจจุบันกับ `offset 5m` ไม่ได้ตรวจอายุจาก timestamp จริง เมื่อไม่มี history อาจแจ้ง Completed เก่าเป็นเหตุการณ์ใหม่ ช่วง `group_interval: 5m` ใกล้กับหน้าต่าง event 5m ต้องทดสอบว่ามี event หลุดก่อนส่งหรือไม่ ไม่รับประกันส่งครั้งเดียว
9. labels เช่น `error`, `message`, AMR `conditions`, timestamps และ snapshot name ที่เปลี่ยนได้เพิ่ม series churn และอาจเปิดเผยรายละเอียดระบบผ่าน metrics/email
10. dashboard มี query ที่ไม่ใช้ `$app`, `$vault`, `$amr` ทุก panel และไม่มี namespace/cluster scope ที่สม่ำเสมอ; `or vector(0)` อาจทำให้ metrics หายดูเหมือนสถานะปกติ

## รายการที่ต้องเปลี่ยนรายไฟล์

| ไฟล์ | ระดับ | สิ่งที่ต้องปรับสำหรับ production |
|---|---|---|
| `01-servicemonitor-crd.md` | P1 | ตรวจ CRD/Operator เดิมและเจ้าของก่อนติดตั้ง; pin artifact/checksum และ version ที่ทดสอบร่วมกัน; ห้ามเขียนทับ shared CRD โดยไม่ผ่าน upgrade plan |
| `02-prometheus-crd.md` | P1 | ใช้นโยบายเดียวกับ 01; ตรวจ stored/served versions และ backup CRs ก่อน schema upgrade; วาง rollback ที่ไม่ใช่ลบ CRD |
| `03-prometheusrule-crd.md` | P1 | ใช้นโยบายเดียวกับ 01; เพิ่ม admission/schema และ rule validation; ถ้ามี platform monitoring อยู่แล้วให้ใช้ CRD ที่ทีม platform ดูแล |
| `04-alertmanager-crd.md` | P1 | ใช้นโยบายเดียวกับ 01; ตรวจ compatibility ของ Alertmanager fields ก่อนอ้างตัวอย่าง TLS จาก comment |
| `05-ksm-values.yaml` | P1 | แก้ comment chart 5.21.0 ให้ตรงกับ version ที่เลือก; ทบทวน namespaces ที่จำเป็นและ RBAC ที่ Helm render จริง; กำหนด resources/securityContext/scheduling ตาม chart schema; ลด labels ที่ยาว/เปลี่ยนบ่อยหรือมีข้อมูลอ่อนไหว; เพิ่ม numeric timestamp metric สำหรับ freshness หาก KSM schema รองรับ conversion ของ field จริง; เก็บ UID และ `cr_namespace`; เปลี่ยน ServiceMonitor label ให้ตรงกับ selector แบบเจาะจง |
| `06-install-ksm.md` | P1 | ตรวจ Helm repo และ pin chart/app/image ที่ผ่าน compatibility/security review; preflight context/namespace/CRDs; ใช้ upgrade/install workflow พร้อม timeout/wait ตาม Helm version ที่เลือก; หลีกเลี่ยง restart ซ้ำโดยไม่จำเป็นและพิสูจน์ config rollout; ตรวจ metrics ของทั้ง 5 CR types, RBAC และ duplicate samples |
| `07-prometheus-rbac.yaml` | P1 | ตั้งชื่อ ClusterRole/Binding ที่มี ownership ชัดเจน; ลด permissions ที่ไม่ใช้ เช่น nodes/ingresses/configmaps ตาม discovery ที่ render จริง; ใช้ namespace Role/Binding สำหรับส่วนที่จำกัด namespace ได้; ตรวจ EndpointSlice access และ negative RBAC tests; อย่า reuse/ยึด resource เดิมโดยอัตโนมัติ |
| `08-prometheus.yaml` | P0/P1 | ทำเป็น base เดียวแล้วใช้ production overlay; เพิ่ม PVC, retention เวลา/ขนาดพร้อม disk headroom, version pin, CPU/memory sizing, securityContext และ external labels เช่น environment/cluster; จำกัด monitor selectors; กำหนด HA ตาม SLO และ replica-specific labels/dedup; ไม่แชร์ TSDB directory ระหว่าง replicas |
| `08-prometheus-alert.yaml` | P0/P1 | ใช้ production base เดียวกับ 08 ไม่คัดลอกค่าคู่กัน; เพิ่ม alerting และ rule selectors ที่จำกัด ownership/namespace; ตรวจ service/port จริงของ Alertmanager; ระบุ evaluation interval ให้ชัด; ทดสอบ reload และ HA alert dedup; ไม่ apply base ที่ลบ alerting ตามหลัง |
| `09-prometheus-operator-installation.md` | P1 | ถ้ามี Operator เดิมให้ integrate แทนติดตั้งซ้ำ; สร้าง namespace ก่อน apply; vendor/pin/checksum bundle และตรวจ RBAC/watch scope/resources/security ของ manifest ที่ดาวน์โหลด; แยก CRD lifecycle จาก app lifecycle; ติดตาม compatibility และ upgrade/rollback |
| `10-prometheus-external-service.yaml` | P0 | production ไม่เปิด HTTP API แบบ unrestricted; ใช้ internal ClusterIP + authenticated TLS gateway หรือ private LB ที่จำกัด network; เปลี่ยน pool/DNS ตาม environment และทดสอบว่าเข้าตรงเพื่อ bypass auth ไม่ได้ |
| `10-prometheus-external-service-clus2.yaml` | P0 | เช่นเดียวกับ 10; ย้าย pool `ai-pool` ออกจาก shared base ไป cluster overlay; ไม่ถือว่า IP ranges ของ lab มีอยู่ใน production |
| `12-nginx-test-app-clus2.yaml` | P1 | แยก lab-only ออกจาก production deployment set; ถ้าจะทำ synthetic test ให้ใช้ namespace/data แยกเฉพาะ, non-root image, securityContext, disabled SA token และ image pin; ไม่ใช้ PVC/application namespace ของ workload จริง |
| `13-trident-servicemonitor.yaml` | P1 | ยืนยัน label/port/namespace ของ Trident version จริง; เปลี่ยน ownership label ให้ตรงกับ Prometheus selector; ตั้ง scrape timeout ไม่เกิน interval และ sample/label limits ที่วัดแล้ว; ใช้ TLS/auth ถ้า endpoint รองรับ มิฉะนั้นจำกัด network; พิสูจน์ว่าไม่ scrape ซ้ำกับ platform stack |
| `14-grafana-install.md` | P0/P1 | แก้ข้อความไม่มี authentication ซึ่งขัดกับขั้นตอนดึง admin password และ values ที่ไม่ได้ปิด auth; กำหนด SSO/role mapping, admin secret/rotation, TLS และ backup/restore; ตรวจ chart source/deprecation กับ registry จริงก่อนสรุป; เปลี่ยน PVC migration จาก uninstall/reinstall เป็นแผนย้ายข้อมูลที่กู้คืนได้ |
| `14-grafana-values.yaml` | P0/P1 | ปิด anonymous อย่าง explicit, ใช้ SSO ตามองค์กรและ secret reference; ตั้ง domain/root_url/secure cookies หลัง TLS gateway; provision datasource/dashboard; กำหนด resources/securityContext/storage backup และ network exposure; ไม่เพิ่ม replicas บน SQLite/shared PVC ตรง ๆ |
| `14-grafana-values-clus2.yaml` | P0/P1 | ใช้มาตรฐานเดียวกับฉบับหลัก; แยกเฉพาะ StorageClass, hostname และ network values ใน cluster overlay เพื่อลด drift |
| `15-grafana-datasource.md` | P1 | เปลี่ยนจาก UI-only เป็น datasource provisioning พร้อม UID คงที่; เลือก query endpoint ที่ชัดสำหรับ single/HA Prometheus และ secret/TLS หากเปิดใช้; ค่า 18/1 series เป็น lab sample ไม่ใช่ acceptance criteria ของ production |
| `16-grafana-dashboard.json` | P1/P2 | เพิ่ม namespace/cluster scope, แสดง CR namespace และ no-data/stale state; ตรวจ datasource variable ตอน provisioning; count เป็นจำนวน CR ที่ยังมีอยู่ ไม่ใช่ยอดสำรองข้อมูลสะสมตลอดเวลา; ยืนยัน sorting/limit และ latest timestamps |
| `16-grafana-dashboard.md` | P1/P2 | ใช้ version-controlled dashboard ที่ provision แล้ว; ถ้าดึง upstream ให้ pin commit/checksum แทน master; แก้คำอธิบาย datasource ให้ตรง JSON; ระบุ semantic ของ counts และข้อมูลที่หายจาก CR cleanup |
| `17-trident-protect-dashboard.json` | P1 | ใช้ filter namespace/cluster/app/vault/AMR ให้สอดคล้องทั้ง summary และ details; ใช้ regex escaping ที่เหมาะกับ multi-select; เพิ่ม missing-data indicators แยกจาก zero; แสดง identity ข้าม namespace และ timezone ที่กำหนด; ไม่ใช้เป็นหลักฐานว่า backup restore ได้ |
| `17-trident-protect-dashboard-clus2.json` | P1 | ใช้ dashboard template เดียวและ environment parameters; ตรวจ hardcoded links/UID; หาก import สอง cluster ใน Grafana เดียวต้องออกแบบ UID/cluster filter ป้องกัน overwrite และสับสน |
| `17-trident-protect-dashboard.md` | P1 | อธิบาย filters/permissions/no-data, ความต่างของ inventory กับ successful events และเวลา creation/transfer; นับ 27 data panels + 6 row entries; เปลี่ยนค่าตัวอย่างเป็น dynamic verification; export/provision workflow ไม่ overwrite โดยไม่ตรวจ ownership |
| `18-cleanup-uninstall.md` | P0 | แยก lab teardown กับ production decommission; production ห้าม blanket-delete CRDs/namespaces/shared RBAC; ตรวจ context, ownership และ dependencies, backup/export/restore test, retention/PV reclaim policy และอนุมัติ destructive steps; แก้ข้อความลืม `-n` แล้วลบ Protect ทันที เพราะผลจริงขึ้นกับ current namespace; ลบเฉพาะ resources ที่โครงการเป็นเจ้าของ |
| `19-alertmanager-config.yaml` | P0/P1 | เปลี่ยน relay/from/to/TLS/auth และ reference secret ภายนอก Git; จัด route ตาม cluster/namespace/team/severity; ทบทวน timing, retries, info opt-in, inhibit และ maintenance silence; แยก shared templates เพื่อลด duplication; ตรวจ empty/malformed timestamps และ escaping ของ error fields; ทดสอบ resolved/firing หลายรายการและ SMTP failures; ไม่รับประกัน exactly-once email |
| `19-alertmanager.yaml` | P0/P1 | เพิ่ม PVC/version/resources/securityContext และ HTTPS DNS `externalUrl`; ใช้ HA ตาม SLO พร้อม spread/anti-affinity, disruption budget และ gossip TCP/UDP connectivity; mount SMTP secret; ตรวจ TLS fields ตาม CRD ที่ติดตั้ง; retention ไม่ใช่การเก็บ alert audit history |
| `19-alertmanager-external-service.yaml` | P0 | เอา direct unrestricted HTTP UI ออกจาก production; เปิดผ่าน TLS/auth gateway หรือ private access policy; ป้องกัน bypass และจำกัดการสร้าง silences/เปลี่ยนสถานะจากผู้ไม่มีสิทธิ์; เปลี่ยน pool ตาม environment |
| `19-alertmanager.md` | P0/P1 | แยก lab กับ prod ชัด; เพิ่ม SMTP trust/credential rotation, HA connectivity, backups และ incident ownership; แก้คำกล่าวว่า offset ตัดประวัติได้แน่นอน; acceptance ต้องทดสอบ routing/delivery/failure/recovery และ reload ของ version จริง |
| `20-alert-rules.yaml` | P1 | เปลี่ยน identity matching เป็น UID + namespace + cluster ตาม labels ที่มีจริง; หากใช้ numeric completion timestamp ให้เลือก window ตาม semantics ไม่ใช่ creation time; ตั้ง for/evaluation ให้เหมาะกับ scrape interval; เพิ่ม stale backup/last-success, stuck tasks, missing metrics/target down, rule evaluation/notification failures; เพิ่ม runbook/dashboard annotations และ team labels; ทดสอบ restarts/history gaps/deletion-recreation/HA dedup; completed notification แยกจาก incident และทำ opt-in |
| `21-mailpit.yaml` | P0 | ใช้เฉพาะ lab/testing; exclude จาก production render/apply โดยอัตโนมัติ ไม่ใช่แค่มี comment เตือน; ใช้ relay จริงแทน และตรวจว่าไม่มี Mailpit endpoints/routes ติดไปด้วย |
| `999-deployment-log.md` | P1 | เก็บเป็น historical lab evidence พร้อมป้ายห้าม copy production; ไม่แก้ประวัติให้ดูเหมือนทดสอบ production; ตรวจข้อมูลภายในก่อนเผยแพร่และแยก production change/test evidence; ไม่ใช้ Status PASS เดิมรับรอง version/config ใหม่ |
| `README.md` | P0/P1 | แยก lab/prod entrypoints และ existing-stack integration; แก้ quick start สร้าง namespace ก่อน Operator และเพิ่ม repo/preflight; เลิกใช้ apply ทั้ง directory; แก้ auth/offset semantics/version claims; เพิ่ม compatibility matrix, required inputs, test gates, upgrade/rollback/restore และ link รายงานนี้ |

## โครงสร้างที่เสนอสำหรับขั้น implementation

คง lab ไว้ใช้งานได้ แยก shared base กับ environment overlays เช่น `environments/lab`, `environments/production/<cluster>` หรือ Helm values layers ตามเครื่องมือที่ทีมดูแลอยู่ ไม่สร้าง framework ใหม่หาก platform มีอยู่แล้ว

มีสอง deployment modes ที่ต้องเลือกก่อน implementation:

1. **Integrate กับ monitoring เดิม:** ติดตั้งเฉพาะ KSM custom-resource configuration, ServiceMonitors, rules และ dashboard ให้ platform Prometheus/Grafana ใช้งาน ห้ามยึด Helm ownership หรือ Operator/CRD ของเดิม
2. **Dedicated stack:** ดูแล Operator/CRDs/Prometheus/Grafana/Alertmanager ครบวงจรและรับภาระ capacity, security, backup, upgrades และ on-call เพิ่ม

สำหรับ Prometheus ใช้ TSDB-compatible POSIX filesystem ที่เอกสารรองรับ ไม่ยก NFS StorageClass ของ Grafana ไปใช้กับ TSDB โดยอัตโนมัติ และ PVC ของแต่ละ replica ต้องแยกกัน [Prometheus storage](https://prometheus.io/docs/prometheus/latest/storage/), [Operator storage](https://prometheus-operator.dev/docs/platform/storage/)

ถ้าต้องทำ Grafana HA ให้ใช้ shared PostgreSQL/MySQL และวาง HA/backup ของ database ด้วย การเพิ่ม replicas บน SQLite เดิมไม่เพียงพอ [Grafana HA](https://grafana.com/docs/grafana/latest/setup-grafana/set-up-for-high-availability/)

HA Prometheus แต่ละ replica เก็บข้อมูลของตัวเอง ต้องเลือก query/dedup/long-term storage architecture ให้เหมาะ ไม่ถือว่าจำนวน replicas ทำให้ข้อมูลถูก replicate [Operator HA](https://prometheus-operator.dev/docs/platform/high-availability/)

สิ่งที่ต้องเพิ่มนอกเหนือจากไฟล์เดิม:

- NetworkPolicies: default-deny ที่วางแผน rollout แล้ว พร้อมอนุญาต DNS, Kubernetes API, scrape targets, Grafana query, Prometheus → Alertmanager, SMTP และ gossip ของ Alertmanager ตาม CNI/ports จริง
- TLS/auth gateway และ certificate lifecycle; ป้องกัน backend bypass; Grafana SSO/roles และ break-glass admin secret
- Namespace ownership/labels, resource quota ตามเหมาะสม, Pod Security compatibility, image/dependency scan และ immutable artifact provenance
- CI: render/schema validation, PromQL/rule tests, Alertmanager config/template tests และ policy ที่ห้าม Mailpit/unrestricted UI/secret literals ใน production output
- การเฝ้าระวัง monitoring เอง: exporter down, missing CR metrics, Prometheus disk/rule failures, notification failures และ end-to-end heartbeat ที่ตรวจจากภายนอก stack
- Backup/restore runbook และ staged rollout/rollback ที่รักษา PVC/database/CRDs; `helm rollback` ไม่ย้อน schema/database migration ให้อัตโนมัติ

## ข้อมูลที่ต้องกำหนดก่อนลงค่า production

| เรื่อง | ต้องทราบ |
|---|---|
| Platform | ใช้ stack เดิมหรือ dedicated; Kubernetes/Trident/Protect/Operator/chart versions ที่ทีมรับรอง |
| Scope | cluster/namespace/team ที่ monitor และชื่อ resources/labels ที่เป็นเจ้าของ |
| SLO | tolerable outage, HA, RPO/RTO ของ metrics/Grafana/notification state |
| Storage | TSDB-compatible StorageClass, capacity/IOPS, retention, snapshots และ restore target |
| Access | DNS, ingress/gateway, IdP/roles, certificate issuer และ private network policy |
| Notifications | SMTP trust/auth, recipients/on-call, expected backup/AMR schedules และ escalation |
| Scale | จำนวน CR/active series, churn, scrape/evaluation interval และ capacity budget |

## Verification และ acceptance gates

ผลรอบนี้:

- ตรวจ Git ก่อนทำงาน: working tree สะอาดบน `main`; สร้าง `chore/production-readiness-review`
- ตรวจรายการไฟล์เดิม 32 ไฟล์และ configuration/คำแนะนำที่เกี่ยวข้อง; ไม่มี local AGENTS.md ใน repository ที่ค้นพบ
- Dashboard JSON ทั้ง 3 ไฟล์ parse ด้วย PowerShell `ConvertFrom-Json` ได้; ไม่ได้ตรวจ runtime หรือรับรองความถูกต้องของ PromQL ด้วยผล parse นี้
- ไม่มี `helm`, `kubectl`, `promtool`, `amtool` ใน PATH ที่ตรวจพบ; Python launcher ใช้งานไม่ได้เพราะ PermissionError ใน AppData จึงไม่ได้ parse YAML ด้วย PyYAML และไม่ได้ติดตั้งเครื่องมือเพิ่ม
- ไม่ได้รัน lint/schema validation, Helm render, server dry-run, rule unit tests, SMTP tests หรือ cluster/security integration tests
- เพิ่มเฉพาะรายงานนี้ ไม่เปลี่ยน manifests/credentials และไม่ commit/push/deploy

ก่อนรับรอง production ต้องมีหลักฐานอย่างน้อย:

1. Render production artifacts ด้วย versions ที่ pin; ตรวจ schemas ตาม CRDs จริง, duplicate resources, lab IPs/receivers, secret leakage และ policy; server-side dry-run/diff บน staging
2. Extract `spec.groups` เป็น Prometheus rules format ก่อนใช้ `promtool check rules` และ `promtool test rules` — ไม่ส่ง Kubernetes CR ทั้งไฟล์ให้ promtool
3. Unit cases: Error → Completed, missing status/metrics, duplicate names ต่าง namespace, UID เปลี่ยน, history ขาด 5m, outage/restart, completion ระหว่าง notification batching และ schedule ที่ไม่ใช่ 5m
4. Validate extracted `alertmanager.yaml` ด้วย amtool version เดียวกับ production; render templates พร้อม labels ว่าง/อักขระ HTML/เวลาผิด; ทดสอบ route, TLS/auth failures, resend และ resolved email
5. ทดสอบ unauthorized access/SSO roles และ backend bypass, RBAC allow/deny, NetworkPolicy รวม gossip, secret rotation และ Pod Security ผ่าน rendered workloads
6. Restart/reschedule/node drain และ restore จริง: metrics ตาม RPO, Grafana dashboards/users, Alertmanager silences; วัด downtime และ duplicate notifications ตาม SLO
7. ทดสอบขนาดข้อมูลจริง: sample count/churn, memory, disk growth/query latency และ config reload; ตรวจตัวเลข dashboard เทียบ Kubernetes CRs ใน scope เดียวกัน
8. ตรวจ end-to-end notification โดย simulated failure ใน staging และ external heartbeat; rollback ได้โดยไม่ลบ CRDs/shared resources/PVCs

## แหล่งอ้างอิงทางการ

อ่านเมื่อ 2026-11-13; เอกสาร latest อธิบายแนวทาง ไม่ใช่หลักฐานว่า fields ทุกตัวรองรับใน pinned versions ของ repository ต้องตรวจ schema ของ release ที่เลือกซ้ำ

- [Prometheus Operator API](https://prometheus-operator.dev/docs/api-reference/api/): selectors, storage, security และ workload fields
- [Alertmanager configuration](https://prometheus.io/docs/alerting/latest/configuration/): routes, grouping, SMTP/TLS และ receivers
- [Prometheus rule unit tests](https://prometheus.io/docs/prometheus/latest/configuration/unit_testing_rules/): test harness สำหรับ rule semantics
- [Grafana authentication](https://grafana.com/docs/grafana/latest/setup-grafana/configure-access/configure-authentication/): authentication และ role mapping options
- [Grafana security](https://grafana.com/docs/grafana/latest/setup-grafana/configure-security/): datasource proxy/network access controls
- [Kubernetes Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/): baseline/restricted workload controls; ทดสอบกับ chart และ storage permissions จริงก่อน enforce
