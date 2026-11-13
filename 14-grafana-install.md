# 14 - Install Grafana

Status: NOT TESTED

Blog: "Install Grafana" + "Enable persistent storage for Grafana"

คลัสเตอร์ : `k8s-clus1`
namespace : `grafana`

รันทีละ block จากบนลงล่าง

> ⚠️ PRODUCTION:
> - เปลี่ยน `storageClassName` เป็น storage class จริงของ cluster
> - ใช้ admin login ตามขั้นตอนด้านล่าง; สำหรับ production ให้กำหนด HTTPS และสิทธิ์เข้าถึงตามนโยบายองค์กร
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

## 4. เปิด external access แยกจาก Helm

values ไม่กำหนดส่วน `service` เพื่อใช้ค่า default ของ chart:
Helm สร้าง Service `my-grafana` แบบ ClusterIP สำหรับใช้ภายใน cluster
ไฟล์ external ด้านล่างเป็นตัวอย่างสำหรับ MetalLB ใน lab เท่านั้น
ถ้า production ใช้ LB อื่น ให้ลบ/เปลี่ยน MetalLB annotation ให้ตรงกับผู้ให้บริการ
ถ้าใช้ Ingress ให้ข้ามไฟล์ external แล้วส่ง traffic ไป Service `my-grafana` แทน
เลือก apply External Service เพียงไฟล์เดียวตาม cluster:

```bash
# cluster 1
kubectl apply -f 14-grafana-external-service-clus1.yaml
```

```bash
# cluster 2
kubectl apply -f 14-grafana-external-service-clus2.yaml
```

ทั้งสองไฟล์สร้าง Service ชื่อ `grafana-external` แต่ใช้ MetalLB pool ต่างกัน
ถ้าเปลี่ยนชื่อ Helm release จาก `my-grafana` ต้องแก้ selector `app.kubernetes.io/instance` ในไฟล์ Service ด้วย

ถ้ามี Grafana ติดตั้งแบบ LoadBalancer เดิมอยู่แล้ว ต้อง `helm upgrade` ด้วย values ใหม่ก่อน
ตรวจ effective values ว่า type เป็น ClusterIP และไม่มี MetalLB annotation เดิมค้างอยู่
อย่าใช้ `--reuse-values` ที่คงค่า service เดิมไว้
Service `my-grafana` จะเปลี่ยนเป็น ClusterIP และ IP เดิมอาจถูกคืนให้ MetalLB
จากนั้น apply External Service และตรวจ IP ใหม่; การย้ายนี้อาจทำให้ external access ขาดช่วง
ไม่ต้อง uninstall Grafana หรือ PVC

## 5. ตรวจผล

```bash
kubectl -n grafana rollout status deployment/my-grafana --timeout=180s
kubectl -n grafana get pods,service,pvc
```

Expected:

```text
pod/my-grafana-...          1/1   Running
service/my-grafana          ClusterIP      ...   80/TCP
service/grafana-external    LoadBalancer   ...   80:.../TCP
persistentvolumeclaim/my-grafana   Bound   10Gi   RWO   sc-clus1-nfs
```

> ⚠️ ถ้าเปลี่ยน `storageClassName` ของ release ที่ติดตั้งไปแล้ว จะ `helm upgrade` ไม่ได้
> เพราะ `spec.storageClassName` ของ PVC เป็น immutable field
> ต้อง `helm uninstall` แล้ว `helm install` ใหม่ (ยังไม่มีข้อมูล Grafana ที่ต้องเก็บ)

## 6. ดึงรหัสผ่าน admin

```bash
kubectl -n grafana get secret my-grafana \
  -o jsonpath="{.data.admin-password}" | base64 --decode; echo
```

ผู้ใช้คือ `admin`

## 7. เข้าใช้งาน

```bash
kubectl -n grafana get service grafana-external \
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
