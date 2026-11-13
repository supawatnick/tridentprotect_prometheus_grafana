# 09 - Install Prometheus Operator

Status: TESTED on `k8s-clus1` with Prometheus Operator `v0.95.1`

Prometheus CR ใน `08-prometheus-alert.yaml` ต้องมี Operator เพื่อสร้าง StatefulSet, Service และ Prometheus pod

ใช้ Operator version `v0.95.1` ให้ตรงกับ CRD ที่ติดตั้งไว้ ห้ามใช้ `LATEST` เพื่อให้การติดตั้งทำซ้ำได้ผลเดิม

## Install

```bash
export NAMESPACE=prometheus
export OPERATOR_VERSION=v0.95.1
```

```bash
curl -sSL \
  "https://github.com/prometheus-operator/prometheus-operator/releases/download/${OPERATOR_VERSION}/bundle.yaml" \
  | sed "s/namespace: default/namespace: ${NAMESPACE}/g" \
  | kubectl apply --server-side -f -
```

ใช้ `apply` แทน `create` เพราะ CRD บางตัวมีอยู่แล้ว และใช้ `--server-side` เพื่อหลีกเลี่ยง annotation limit ของ CRD ขนาดใหญ่

## Verify Operator

```bash
kubectl -n prometheus rollout status deployment/prometheus-operator --timeout=180s
kubectl -n prometheus get deployment,pod,service -l app.kubernetes.io/name=prometheus-operator
```

Expected: Operator pod `1/1 Running`

## Verify Prometheus Reconciliation

ถ้า `08-prometheus-alert.yaml` ถูก apply ไว้แล้ว Operator จะ reconcile ให้อัตโนมัติ

```bash
kubectl -n prometheus get prometheus prometheus
kubectl -n prometheus get statefulset prometheus-prometheus
kubectl -n prometheus get pod prometheus-prometheus-0
kubectl -n prometheus get service prometheus-operated
```

Expected: `prometheus-prometheus-0` เป็น `2/2 Running`

ทดสอบ Prometheus ก่อนสร้าง External Service ในขั้นที่ 10 ด้วย port-forward:

```bash
kubectl -n prometheus port-forward service/prometheus-operated 9090:9090
```

เปิด `http://localhost:9090/targets` และตรวจว่า target `kube-state-metrics` เป็น `UP`
