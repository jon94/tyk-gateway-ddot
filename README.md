# Tyk Gateway + Datadog DDOT Collector on GKE

End-to-end OTel tracing from Tyk Gateway OSS through the Datadog Distribution of OpenTelemetry (DDOT) Collector into Datadog APM, running on a GKE Standard cluster.

---

## Architecture

```
Tyk Gateway (tyk namespace)
    │  OTLP gRPC
    ▼
datadog-otel ClusterIP Service (port 4317)
    │
    ▼
DDOT Collector (otel-agent container in Datadog Agent DaemonSet)
    │  otel-config.yaml pipeline
    ▼
Datadog APM (service: tyk)
```

---

## Prerequisites

- GKE Standard cluster (non-autopilot) — Ubuntu node image required (COS blocks eBPF writes to `/usr/src`)
- `kubectl` and `helm` configured
- Datadog API key

---

## 1. Create GKE Cluster

```bash
gcloud container clusters create tyk-otel-demo \
  --zone=asia-east2-a \
  --machine-type=e2-standard-2 \
  --num-nodes=2 \
  --cluster-version=latest

# Create Ubuntu node pool (required — COS image blocks Datadog system-probe)
gcloud container node-pools create ubuntu-pool \
  --cluster=tyk-otel-demo \
  --zone=asia-east2-a \
  --machine-type=e2-standard-2 \
  --num-nodes=2 \
  --image-type=UBUNTU_CONTAINERD

# Delete the default COS pool
gcloud container node-pools delete default-pool \
  --cluster=tyk-otel-demo \
  --zone=asia-east2-a \
  --quiet

gcloud container clusters get-credentials tyk-otel-demo --zone=asia-east2-a
```

---

## 2. Create Namespaces

```bash
kubectl create namespace tyk
kubectl create namespace datadog
```

---

## 3. Add Helm Repos

```bash
helm repo add tyk-helm https://helm.tyk.io/public/helm/charts/
helm repo add datadog https://helm.datadoghq.com
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
```

---

## 4. Deploy Datadog Agent with DDOT Collector

The DDOT Collector runs as a sidecar container (`otel-agent`) inside the Datadog Agent DaemonSet. The pipeline is defined in `otel-config.yaml` and mounted via a ConfigMap.

```bash
# Fill in your API key in ddot-values.yaml first, then:

kubectl create configmap ddot-otel-config \
  --from-file=otel-config.yaml=otel-config.yaml \
  -n datadog

helm install datadog-agent datadog/datadog \
  --namespace datadog \
  -f ddot-values.yaml
```

### Expose DDOT OTLP receiver as a ClusterIP Service

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: datadog-otel
  namespace: datadog
spec:
  selector:
    app: datadog-agent
  ports:
    - name: otlp-grpc
      port: 4317
      targetPort: 4317
      protocol: TCP
    - name: otlp-http
      port: 4318
      targetPort: 4318
      protocol: TCP
EOF
```

---

## 5. Deploy Redis + Tyk Gateway

```bash
helm install redis bitnami/redis \
  --namespace tyk \
  --set auth.enabled=false \
  --set architecture=standalone

helm install tyk-gateway tyk-helm/tyk-oss \
  --namespace tyk \
  -f tyk-values.yaml
```

---

## 6. Verify Spans are Flowing

```bash
# Port-forward Tyk
kubectl port-forward -n tyk svc/gateway-svc-tyk-gateway 8080:8080

# Create a test API
curl -s -X POST http://localhost:8080/tyk/apis \
  -H "x-tyk-authorization: CHANGEME" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "httpbin-test",
    "api_id": "httpbin-test",
    "org_id": "default",
    "use_keyless": true,
    "version_data": {
      "not_versioned": true,
      "versions": { "Default": { "name": "Default", "use_extended_paths": true } }
    },
    "proxy": {
      "listen_path": "/httpbin/",
      "target_url": "https://httpbin.org/",
      "strip_listen_path": true
    },
    "active": true
  }'

curl -s http://localhost:8080/tyk/reload -H "x-tyk-authorization: CHANGEME"

# Send traffic
for i in $(seq 1 10); do curl -s -o /dev/null -w "req $i: %{http_code}\n" http://localhost:8080/httpbin/get; done

# Check DDOT received and exported spans
DDPOD=$(kubectl get pods -n datadog -l app=datadog-agent -o jsonpath='{.items[0].metadata.name}')
kubectl port-forward -n datadog pod/$DDPOD 8888:8888 &
curl -s http://localhost:8888/metrics | grep -E "accepted_spans|sent_spans|refused_spans" | grep -v "^#"
```

Expected output:
```
otelcol_exporter_sent_spans{...exporter="datadog"...}       <N>
otelcol_receiver_accepted_spans{...receiver="otlp"...}      <N>
otelcol_receiver_refused_spans{...receiver="otlp"...}       0
```

View traces in Datadog: **APM → Traces → filter `service:tyk`**

---

## Gotchas

| Issue | Cause | Fix |
|---|---|---|
| Datadog Agent pods `CreateContainerError` | COS node image blocks `/usr/src` writes (eBPF) | Use `--image-type=UBUNTU_CONTAINERD` node pool |
| Tyk OTel endpoint `$(HOST_IP)` resolves literally | Kubernetes env var substitution order — `HOST_IP` injected after the OTel env var | Use a ClusterIP Service DNS name instead of `hostIP` |
| Tyk initialized but no spans exported | gRPC connection gone idle between init and first request | Set `connectionTimeout: 10` and `spanProcessorType: simple` |
| API definitions lost on pod restart | Tyk OSS stores APIs ephemerally | Mount API definitions as a ConfigMap under `/mnt/tyk-gateway/apps/` or use Tyk Operator |
| Tyk API 404 after reload | Pod restarted and lost in-memory API | Re-POST to `/tyk/apis` and call `/tyk/reload` |
