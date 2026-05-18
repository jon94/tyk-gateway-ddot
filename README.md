# Tyk Gateway + Datadog DDOT Collector on GKE

End-to-end OTel tracing from Tyk Gateway OSS through the Datadog Distribution of OpenTelemetry (DDOT) Collector into Datadog APM, running on GKE Standard.

## Architecture

```
Tyk Gateway  ──OTLP gRPC──▶  datadog-otel (ClusterIP :4317)
                                       │
                               DDOT Collector
                            (otel-agent container
                             in Datadog DaemonSet)
                                       │
                              otel-config.yaml pipeline
                           k8sattributes → resourcedetection
                            → infraattributes → batch
                                       │
                              Datadog APM (service: tyk)
```

## Repository Structure

```
.
├── otel-config.yaml   # DDOT Collector pipeline config
├── ddot-values.yaml   # Datadog Agent Helm values
├── tyk-values.yaml    # Tyk Gateway Helm values
└── README.md
```

---

## Prerequisites

- GCP project with GKE API enabled
- `gcloud` CLI authenticated (`gcloud auth login`)
- `kubectl` and `helm` installed
- Datadog account — API key ready

---

## Step 1 — Create GKE Cluster

> **Important:** Use `UBUNTU_CONTAINERD` node image. The default COS image has a read-only `/usr/src` that blocks Datadog system-probe startup.

```bash
# Create cluster
gcloud container clusters create tyk-otel-demo \
  --zone=asia-east2-a \
  --machine-type=e2-standard-2 \
  --num-nodes=2 \
  --cluster-version=latest

# Add Ubuntu node pool
gcloud container node-pools create ubuntu-pool \
  --cluster=tyk-otel-demo \
  --zone=asia-east2-a \
  --machine-type=e2-standard-2 \
  --num-nodes=2 \
  --image-type=UBUNTU_CONTAINERD

# Remove default COS pool
gcloud container node-pools delete default-pool \
  --cluster=tyk-otel-demo \
  --zone=asia-east2-a \
  --quiet

# Get credentials
gcloud container clusters get-credentials tyk-otel-demo --zone=asia-east2-a
```

---

## Step 2 — Add Helm Repositories

```bash
helm repo add tyk-helm https://helm.tyk.io/public/helm/charts/
helm repo add datadog https://helm.datadoghq.com
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
```

---

## Step 3 — Create Namespaces

```bash
kubectl create namespace tyk
kubectl create namespace datadog
```

---

## Step 4 — Deploy Datadog Agent with DDOT Collector

**4a.** Fill in your Datadog API key in `ddot-values.yaml`:
```yaml
datadog:
  apiKey: <YOUR_DATADOG_API_KEY>
```

**4b.** Create the DDOT config ConfigMap:
```bash
kubectl create configmap ddot-otel-config \
  --from-file=otel-config.yaml=otel-config.yaml \
  --namespace datadog
```

**4c.** Install the Datadog Agent:
```bash
helm install datadog-agent datadog/datadog \
  --namespace datadog \
  --values ddot-values.yaml
```

**4d.** Expose the DDOT OTLP receiver as a ClusterIP Service:
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

## Step 5 — Deploy Redis and Tyk Gateway

```bash
# Redis (required by Tyk)
helm install redis bitnami/redis \
  --namespace tyk \
  --set auth.enabled=false \
  --set architecture=standalone

# Tyk Gateway
helm install tyk-gateway tyk-helm/tyk-oss \
  --namespace tyk \
  --values tyk-values.yaml
```

---

## Step 6 — Verify Everything is Running

```bash
kubectl get pods -n datadog
kubectl get pods -n tyk
```

Expected output:
```
NAMESPACE   NAME                                      READY   STATUS
datadog     datadog-agent-xxxxx                       4/4     Running
datadog     datadog-agent-cluster-agent-xxxxx         1/1     Running
tyk         gateway-tyk-gateway-xxxxx                 1/1     Running
tyk         redis-master-0                            1/1     Running
```

---

## Step 7 — Create a Test API and Send Traffic

```bash
# Port-forward Tyk Gateway
kubectl port-forward -n tyk svc/gateway-svc-tyk-gateway 8080:8080

# Create a test API (proxies to httpbin.org)
curl -X POST http://localhost:8080/tyk/apis \
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

# Reload Tyk to activate the API
curl http://localhost:8080/tyk/reload -H "x-tyk-authorization: CHANGEME"

# Send test traffic
for i in $(seq 1 20); do
  curl -s -o /dev/null http://localhost:8080/httpbin/get
  curl -s -o /dev/null http://localhost:8080/httpbin/status/404
  curl -s -o /dev/null http://localhost:8080/httpbin/status/500
done
```

---

## Step 8 — Confirm Spans are Reaching Datadog

```bash
# Find the Datadog Agent pod on the same node as Tyk
TYKNODE=$(kubectl get pod -n tyk -l app=gateway-tyk-gateway -o jsonpath='{.items[0].spec.nodeName}')
DDPOD=$(kubectl get pods -n datadog -l app=datadog-agent -o wide --no-headers | grep "$TYKNODE" | awk '{print $1}')

# Port-forward DDOT metrics endpoint
kubectl port-forward -n datadog pod/$DDPOD 8888:8888 &

# Check span counters
curl -s http://localhost:8888/metrics | grep -E "accepted_spans|sent_spans|refused_spans" | grep -v "^#"
```

Expected output:
```
otelcol_exporter_sent_spans{exporter="datadog",...}          <N>
otelcol_receiver_accepted_spans{receiver="otlp",...}         <N>
otelcol_receiver_refused_spans{receiver="otlp",...}          0
```

View traces in Datadog: **APM → Traces → filter `service:tyk`**

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Agent pods `CreateContainerError` | COS node image blocks `/usr/src` writes | Use `--image-type=UBUNTU_CONTAINERD` node pool |
| No spans in DDOT metrics | `k8sattributes` RBAC missing pod list permissions | RBAC rules are included in `ddot-values.yaml` — re-run `helm upgrade` |
| Tyk logs `lookup $(HOST_IP): no such host` | `$(HOST_IP)` env var not substituted | Use ClusterIP Service DNS name (`datadog-otel.datadog.svc.cluster.local:4317`) instead of `hostIP` |
| Tyk initialized but no spans exported | gRPC connection gone idle | `spanProcessorType: simple` and `connectionTimeout: 10` are set in `tyk-values.yaml` |
| API returns 404 after pod restart | Tyk OSS stores API definitions ephemerally | Re-POST to `/tyk/apis` and call `/tyk/reload` after every restart |
| SSH to VM timing out | Firewall rule missing or wrong source IP | Use `--tunnel-through-iap` with `gcloud compute ssh` |
