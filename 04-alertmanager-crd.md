# 04 - Alertmanager CRD

Status: TESTED

เตรียมไว้สำหรับเพิ่ม Alertmanager ภายหลัง

## Apply

```bash
kubectl apply --server-side -f https://raw.githubusercontent.com/prometheus-operator/prometheus-operator/v0.95.1/example/prometheus-operator-crd/monitoring.coreos.com_alertmanagers.yaml
```

## Verify

```bash
kubectl wait --for=condition=Established crd/alertmanagers.monitoring.coreos.com --timeout=60s
kubectl get crd alertmanagers.monitoring.coreos.com
```

Expected: `Established=True`
