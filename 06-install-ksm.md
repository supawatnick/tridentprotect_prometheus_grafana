# 06 - Install kube-state-metrics

ขั้นตอนนี้ทดสอบผ่านแล้วกับ chart `kube-state-metrics` version `9.0.0` (KSM app `v2.20.0`)

> Blog ใช้ chart `5.21.0` (KSM `v2.12.0`) ซึ่งมีบั๊กส่ง custom-resource metrics ซ้ำสองชุด
> ทำให้ Prometheus ตกข้อมูล (`Error on ingesting samples with different value but same timestamp`)
> บั๊กนี้ถูกแก้ใน KSM `v2.14.0` (PR #2502) จึงใช้ `9.0.0` แทน

## Add Helm Repository

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
```

```bash
helm repo update
```

## Create Namespace

```bash
kubectl create namespace prometheus --dry-run=client -o yaml | kubectl apply -f -
```

คำสั่งนี้รันซ้ำได้ ถ้ามี namespace อยู่แล้วผลจะเป็น `unchanged`

## Install

```bash
helm install trident-protect \
  -f 05-ksm-values.yaml \
  prometheus-community/kube-state-metrics \
  --version 9.0.0 \
  -n prometheus
```

## Upgrade (แก้ค่าใน 05-ksm-values.yaml)

ใช้เมื่อเปลี่ยน custom resource หรือ label ใน `05-ksm-values.yaml`

```bash
helm upgrade trident-protect \
  -f 05-ksm-values.yaml \
  prometheus-community/kube-state-metrics \
  --version 9.0.0 \
  -n prometheus
```

```bash
kubectl -n prometheus rollout status deployment/trident-protect-kube-state-metrics --timeout=180s
```

โหลด KSM ใหม่ทำให้ scrape gap ประมาณ 10-20 วินาที

## Verify

```bash
helm status trident-protect -n prometheus
kubectl -n prometheus get pods,service,servicemonitor
```

Expected:

```text
trident-protect-kube-state-metrics   1/1   Running
```

ตรวจ RBAC สำหรับ Trident Protect CRs:

```bash
kubectl get clusterrole trident-protect-kube-state-metrics \
  -o jsonpath='{range .rules[?(@.apiGroups[0]=="protect.trident.netapp.io")]}{.resources}{" => "}{.verbs}{"\n"}{end}'
```

ต้องมี `list` และ `watch` สำหรับ `backups`, `snapshots`, `exechooksruns`, `appvaults` และ `appmirrorrelationships`

ตรวจว่า custom-resource metrics ไม่ถูกส่งซ้ำ (ผลต้องเป็น `1` ทุกบรรทัด):

```bash
kubectl -n prometheus port-forward svc/trident-protect-kube-state-metrics 18090:8080 &
curl -s http://127.0.0.1:18090/metrics \
  | grep '^kube_customresource_backup_info' | sort | uniq -c | awk '{print $1}' | sort -u
```

ตรวจว่า Prometheus ไม่ตก samples:

```bash
kubectl -n prometheus logs prometheus-prometheus-0 -c prometheus --since=1m \
  | grep -c "different value but same timestamp"
```

ผลต้องเป็น `0`
