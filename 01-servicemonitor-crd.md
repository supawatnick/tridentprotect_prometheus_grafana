# 01 - ServiceMonitor CRD

Status: TESTED

ใช้โดย kube-state-metrics เพื่อสร้าง `ServiceMonitor`

## Apply

```bash
kubectl apply --server-side -f https://raw.githubusercontent.com/prometheus-operator/prometheus-operator/v0.95.1/example/prometheus-operator-crd/monitoring.coreos.com_servicemonitors.yaml
```

## Verify

```bash
kubectl wait --for=condition=Established crd/servicemonitors.monitoring.coreos.com --timeout=60s
kubectl get crd servicemonitors.monitoring.coreos.com
```

Expected: `Established=True`
