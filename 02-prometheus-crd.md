# 02 - Prometheus CRD

Status: TESTED

ใช้โดย `08-prometheus.yaml`

## Apply

```bash
kubectl apply --server-side -f https://raw.githubusercontent.com/prometheus-operator/prometheus-operator/v0.95.1/example/prometheus-operator-crd/monitoring.coreos.com_prometheuses.yaml
```

## Verify

```bash
kubectl wait --for=condition=Established crd/prometheuses.monitoring.coreos.com --timeout=60s
kubectl get crd prometheuses.monitoring.coreos.com \
  -o jsonpath='{range .spec.versions[*]}{.name}{" served="}{.served}{" storage="}{.storage}{" schema="}{.schema.openAPIV3Schema.type}{"\n"}{end}'
```

Expected: `v1 served=true storage=true schema=object`
