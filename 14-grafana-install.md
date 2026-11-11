# 14 - Install Grafana

Status: NOT TESTED

Blog: "Install Grafana" + "Enable persistent storage for Grafana"

คลัสเตอร์ : `k8s-clus1`
namespace : `grafana`

รันทีละ block จากบนลงล่าง

> ⚠️ PRODUCTION:
> - เปลี่ยน `storageClassName` เป็น storage class จริงของ cluster
> - Grafana ไม่มี authentication ติดตั้งมา เมื่อเปิดผ่าน MetalLB ให้จำกัดเครือข่ายที่เข้าถึงได้
> - เปิด persistence ตั้งแต่ครั้งแรก ถ้าเปิดทีหลังข้อมูลเดิมจะหาย
>
> ℹ️ `helm install` จะขึ้นคำเตือน `this chart is deprecated` เพราะ chart ย้ายไป OCI registry
> แต่ OCI ต้อง authenticate จึงยังใช้ HTTP repo ได้อย่างเดียวใน environment นี้
> chart `10.5.15` เป็นเวอร์ชันล่าสุดที่มีใน repo และ pin ไว้แล้ว จึงไม่กระทบ

## 1. เพิ่ม Helm repository

```bash
helm repo add grafana https://grafana.github.io/helm-charts
```

```bash
helm repo update
```

## 2. สร้าง namespace

```bash
kubectl create namespace grafana --dry-run=client -o yaml | kubectl apply -f -
```

## 3. ติดตั้ง Grafana

```bash
helm install my-grafana grafana/grafana \
  -f 14-grafana-values.yaml \
  --version 10.5.15 \
  -n grafana
```

> Blog ติดตั้งก่อนแล้วค่อย `helm upgrade` เพื่อเปิด persistence
> ที่นี่เปิด persistence ตั้งแต่ครั้งแรกเพื่อไม่ให้ข้อมูลหาย
>
> สำหรับ `k8s-clus2` ให้ใช้ไฟล์ค่าอีกชุด เพราะ storage class และ MetalLB pool ต่างกัน:
>
> ```bash
> helm install my-grafana grafana/grafana \
>   -f 14-grafana-values-clus2.yaml \
>   --version 10.5.15 \
>   -n grafana
> ```

## 4. ตรวจผล

```bash
kubectl -n grafana rollout status deployment/my-grafana --timeout=180s
kubectl -n grafana get pods,service,pvc
```

Expected:

```text
pod/my-grafana-...          1/1   Running
service/my-grafana          LoadBalancer   ...   80:.../TCP
persistentvolumeclaim/my-grafana   Bound   10Gi   RWO   sc-clus1-nfs
```

> ⚠️ ถ้าเปลี่ยน `storageClassName` ของ release ที่ติดตั้งไปแล้ว จะ `helm upgrade` ไม่ได้
> เพราะ `spec.storageClassName` ของ PVC เป็น immutable field
> ต้อง `helm uninstall` แล้ว `helm install` ใหม่ (ยังไม่มีข้อมูล Grafana ที่ต้องเก็บ)

## 5. ดึงรหัสผ่าน admin

```bash
kubectl -n grafana get secret my-grafana \
  -o jsonpath="{.data.admin-password}" | base64 --decode; echo
```

ผู้ใช้คือ `admin`

## 6. เข้าใช้งาน

```bash
kubectl -n grafana get service my-grafana \
  -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
```

เปิดในเบราว์เซอร์:

```text
http://<EXTERNAL-IP>/
```

ทางเลือกแบบ port-forward ตาม Blog:

```bash
kubectl -n grafana port-forward svc/my-grafana 3000:80
```

เปิด `http://localhost:3000`

## ถัดไป

`15-grafana-datasource.md` (เพิ่ม Prometheus เป็น data source) → `16-grafana-dashboard.md` (import dashboard)
