# Tyk Gateway + Datadog DDOT Collector on GKE

End-to-end OTel tracing from Tyk Gateway OSS through the Datadog Distribution of OpenTelemetry (DDOT) Collector into Datadog APM, running on GKE Standard.

---

## How it works

```
Tyk Gateway Pod (tyk namespace)
        │
        │  OTLP gRPC to datadog-otel:4317
        │  (internalTrafficPolicy: Local ensures same-node routing)
        ▼
DDOT Collector — otel-agent container
(inside Datadog Agent DaemonSet, same node as Tyk)
        │
        │  otel-config.yaml pipeline:
        │  otlp receiver
        │    → k8sattributes   (adds k8s.pod.name, k8s.node.name, etc.)
        │    → resourcedetection (adds GCP/host metadata)
        │    → resource        (fixes host.name, ensures container name)
        │    → infraattributes (maps attributes to Datadog infra tags)
        │    → batch           (buffers before export)
        │    → datadog exporter
        ▼
Datadog APM
  service: tyk
  host tags: from the GKE node
  container tags: from the Tyk pod
```

### Why same-node routing matters

The `infraattributes` processor enriches spans with Datadog infrastructure tags (host tags, container tags, pod labels) by looking up containers in the **local Datadog Agent's container cache**. That cache only knows about containers running on the **same node**. If spans are routed to a DDOT pod on a different node, `infraattributes` cannot find the Tyk container and no infrastructure tags are added.

This is solved by setting `internalTrafficPolicy: Local` on the `datadog-otel` Kubernetes Service. kube-proxy then only routes traffic to the DDOT pod on the same node as the source pod.

---

## Repository structure

```
.
├── otel-config.yaml   # DDOT Collector pipeline — what to receive, process, and export
├── ddot-values.yaml   # Datadog Agent Helm values — enables DDOT, sets RBAC, resources
├── tyk-values.yaml    # Tyk Gateway Helm values — enables OTel, sets resource attributes
└── README.md
```

---

## File explanations

### `otel-config.yaml`

This is the OpenTelemetry Collector pipeline configuration mounted into the DDOT container via a Kubernetes ConfigMap. It defines what the collector receives, how it processes spans, and where it sends them.

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
```
**Receiver**: Listens on all interfaces at port 4317 for OTLP/gRPC. Tyk Gateway sends spans here. Port 4318 is available for OTLP/HTTP if needed.

---

```yaml
processors:
  k8sattributes:
    auth_type: serviceAccount
    passthrough: false
    extract:
      metadata:
        - k8s.pod.name
        - k8s.pod.uid
        - k8s.deployment.name
        - k8s.namespace.name
        - k8s.node.name
        - k8s.container.name
    pod_association:
      - sources:
          - from: resource_attribute
            name: k8s.pod.ip
      - sources:
          - from: connection
```
**k8sattributes processor**: Enriches every span with Kubernetes metadata by looking up the source pod in the Kubernetes API.

- `auth_type: serviceAccount` — uses the DaemonSet's service account to query the Kubernetes API. The RBAC rules in `ddot-values.yaml` grant the required permissions.
- `passthrough: false` — if the pod cannot be identified, the span is still forwarded (not dropped), just without k8s metadata.
- `extract.metadata` — the list of Kubernetes fields to add as resource attributes on each span.
- `pod_association` — how the processor identifies which pod sent the span:
  1. First tries `k8s.pod.ip` resource attribute (set by Tyk via `OTEL_RESOURCE_ATTRIBUTES`)
  2. Falls back to the gRPC connection's source IP address

---

```yaml
  resourcedetection:
    detectors: [env, system, gcp]
    timeout: 5s
    override: false
```
**resourcedetection processor**: Adds host and cloud metadata.

- `env` — reads `OTEL_RESOURCE_ATTRIBUTES` from the collector's own environment.
- `system` — adds `host.name` from the OS hostname of the DDOT container.
- `gcp` — queries the GCP metadata server for project ID, zone, instance name.
- `override: false` — will not overwrite attributes already set by `k8sattributes`. Order matters: run this after `k8sattributes`.

---

```yaml
  resource:
    attributes:
      - key: host.name
        from_attribute: k8s.node.name
        action: upsert
      - key: k8s.container.name
        value: gateway-tyk-gateway
        action: insert
```
**resource processor**: Two targeted attribute fixes.

- `host.name` upsert from `k8s.node.name`: **Critical fix**. By default, Tyk (like any Go process in Kubernetes) reports the pod name as its kernel hostname. If `host.name` equals the pod name, Datadog treats each pod restart as a new host, inflating your host count and preventing host tag correlation. Overwriting with `k8s.node.name` (set by `k8sattributes`) maps spans to the correct GKE node, which the Datadog Agent already monitors with its tags.
- `k8s.container.name` insert: Safety fallback. The `infraattributes` processor requires `k8s.pod.uid` + `k8s.container.name` to look up container metadata. `insert` only sets the value if it is not already present — so if `k8sattributes` already set it correctly, this is a no-op.

---

```yaml
  infraattributes:
    cardinality: 2
```
**infraattributes processor**: Datadog-specific processor that maps OTel resource attributes to Datadog infrastructure tags. It looks up the container in the local Datadog Agent's tagger cache and adds tags like `kube_namespace`, `kube_deployment`, `image_name`, pod labels, etc.

To identify the container it tries (in order):
1. `process.pid` — Tyk exposes this automatically as a Go process
2. `datadog.container.cgroup_inode`
3. `k8s.pod.uid` + `k8s.container.name`

**Requirement**: The span must be processed by the DDOT on the same node as the source container. This is why `internalTrafficPolicy: Local` is essential.

---

```yaml
  batch:
    send_batch_max_size: 100
    send_batch_size: 10
    timeout: 10s
```
**batch processor**: Groups spans before exporting to reduce API calls. These settings are from [Tyk's recommended OTel configuration](https://tyk.io/docs/api-management/traces#opentelemetry-backends-for-tracing).

---

```yaml
exporters:
  datadog:
    api:
      site: ${env:DD_SITE}
      key: ${env:DD_API_KEY}
```
**Datadog exporter**: Sends spans to Datadog APM. Reads the API key and site from environment variables injected by the Datadog Agent Helm chart — no secrets hardcoded in the config file.

---

```yaml
service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [k8sattributes, resourcedetection, resource, infraattributes, batch]
      exporters: [datadog]
```
**Pipeline**: Processor order is deliberate:
1. `k8sattributes` — must run first to populate `k8s.node.name` before `resource` copies it to `host.name`
2. `resourcedetection` — adds GCP metadata, `override: false` preserves what `k8sattributes` set
3. `resource` — fixes `host.name` and ensures `k8s.container.name` is present
4. `infraattributes` — requires all the above attributes to already be on the span
5. `batch` — always last before the exporter

---

### `ddot-values.yaml`

Helm values for the `datadog/datadog` chart. Configures the Datadog Agent DaemonSet and enables the DDOT Collector sidecar container.

```yaml
datadog:
  apiKey: <YOUR_DATADOG_API_KEY>
  site: datadoghq.com
  clusterName: tyk-otel-demo
```
Standard Datadog Agent config — API key, site, and cluster name (used for tagging).

---

```yaml
  otelCollector:
    enabled: true
    ports:
      - containerPort: 4317
        hostPort: 4317
        name: otel-grpc
      - containerPort: 4318
        hostPort: 4318
        name: otel-http
```
Enables the DDOT Collector as a sidecar container (`otel-agent`) inside each Datadog Agent pod. `hostPort` exposes the OTLP ports on the node's network interface (alternative ingress path if needed).

---

```yaml
    configMap:
      name: ddot-otel-config
      key: otel-config.yaml
```
Tells the DDOT container to load its pipeline configuration from the `ddot-otel-config` ConfigMap (created from `otel-config.yaml`). This is the correct pattern — keep the OTel config in a standalone file rather than embedding it inline in Helm values.

---

```yaml
    rbac:
      create: true
      rules:
        - apiGroups: [""]
          resources: ["pods", "namespaces", "nodes", ...]
          verbs: ["get", "list", "watch"]
        - apiGroups: ["apps"]
          resources: ["replicasets", "deployments", ...]
          verbs: ["get", "list", "watch"]
```
**Critical**: The `k8sattributes` processor needs to call the Kubernetes API to look up pod metadata. Without these ClusterRole rules, it silently fails to enrich spans and span processing stalls entirely. These rules are added here so they survive Helm upgrades — patching the ClusterRole directly is overwritten on the next `helm upgrade`.

---

### `tyk-values.yaml`

Helm values for the `tyk-helm/tyk-oss` chart.

```yaml
    opentelemetry:
      enabled: true
      exporter: grpc
      endpoint: datadog-otel.datadog.svc.cluster.local:4317
      connectionTimeout: 10
      spanProcessorType: simple
      samplingType: AlwaysOn
      contextPropagation: tracecontext
```
Tyk's native OTel configuration.

- `endpoint` — the Kubernetes DNS name of the `datadog-otel` ClusterIP service in the `datadog` namespace. Using a DNS name avoids the `$(HOST_IP)` env var substitution problem (Tyk's chart generates `TYK_GW_OPENTELEMETRY_ENDPOINT` before `extraEnvs` are evaluated, so `$(HOST_IP)` can never be substituted).
- `connectionTimeout: 10` — increased from the default 1 second. The default caused spans to be silently dropped when the gRPC connection had any startup latency.
- `spanProcessorType: simple` — exports each span immediately instead of batching. In combination with Tyk's batch settings being on the collector side, this ensures spans are not held in Tyk's memory.
- `samplingType: AlwaysOn` — sends 100% of spans. Adjust for production.
- `contextPropagation: tracecontext` — uses W3C Trace Context headers for distributed tracing correlation.

---

```yaml
    extraEnvs:
      - name: OTEL_K8S_POD_UID
        valueFrom:
          fieldRef:
            fieldPath: metadata.uid
      - name: OTEL_K8S_POD_IP
        valueFrom:
          fieldRef:
            fieldPath: status.podIP
      - name: OTEL_RESOURCE_ATTRIBUTES
        value: "k8s.pod.uid=$(OTEL_K8S_POD_UID),k8s.pod.ip=$(OTEL_K8S_POD_IP),k8s.container.name=gateway-tyk-gateway"
```
Injects Kubernetes identity into spans as OTel resource attributes via the downward API.

- `k8s.pod.uid` — required by the `infraattributes` processor (detection method 3: `k8s.pod.uid` + `k8s.container.name`).
- `k8s.pod.ip` — gives `k8sattributes` a reliable way to look up the pod beyond the connection source IP.
- `k8s.container.name` — hardcoded to `gateway-tyk-gateway` (the Tyk container name in the pod spec). Required alongside `k8s.pod.uid` for `infraattributes`.

---

## Kubernetes Service — same-node routing

The `datadog-otel` ClusterIP Service is created with `internalTrafficPolicy: Local`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: datadog-otel
  namespace: datadog
spec:
  selector:
    app: datadog-agent
  internalTrafficPolicy: Local   # key setting
  ports:
    - name: otlp-grpc
      port: 4317
      targetPort: 4317
```

`internalTrafficPolicy: Local` tells kube-proxy to only route traffic to endpoints on the **same node** as the source pod. Since the Datadog Agent is a DaemonSet (one pod per node), this guarantees Tyk's spans always reach the DDOT pod on the same node — which is required for `infraattributes` to find the container in the local tagger cache.

---

## Setup steps

### 1. Create GKE cluster

> Use `UBUNTU_CONTAINERD` image type. The default COS image has a read-only `/usr/src` that blocks Datadog system-probe startup (exit code 203/EXEC).

```bash
gcloud container clusters create tyk-otel-demo \
  --zone=asia-east2-a \
  --machine-type=e2-standard-2 \
  --num-nodes=2

gcloud container node-pools create ubuntu-pool \
  --cluster=tyk-otel-demo \
  --zone=asia-east2-a \
  --machine-type=e2-standard-2 \
  --num-nodes=2 \
  --image-type=UBUNTU_CONTAINERD

gcloud container node-pools delete default-pool \
  --cluster=tyk-otel-demo --zone=asia-east2-a --quiet

gcloud container clusters get-credentials tyk-otel-demo --zone=asia-east2-a
```

### 2. Add Helm repos

```bash
helm repo add tyk-helm https://helm.tyk.io/public/helm/charts/
helm repo add datadog https://helm.datadoghq.com
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
```

### 3. Create namespaces

```bash
kubectl create namespace tyk
kubectl create namespace datadog
```

### 4. Deploy Datadog Agent with DDOT Collector

```bash
# Fill in your API key in ddot-values.yaml, then:

kubectl create configmap ddot-otel-config \
  --from-file=otel-config.yaml=otel-config.yaml \
  --namespace datadog

helm install datadog-agent datadog/datadog \
  --namespace datadog \
  --values ddot-values.yaml
```

### 5. Create the same-node routing Service

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
  internalTrafficPolicy: Local
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

### 6. Deploy Redis and Tyk Gateway

```bash
helm install redis bitnami/redis \
  --namespace tyk \
  --set auth.enabled=false \
  --set architecture=standalone

helm install tyk-gateway tyk-helm/tyk-oss \
  --namespace tyk \
  --values tyk-values.yaml
```

### 7. Create a test API and send traffic

```bash
kubectl port-forward -n tyk svc/gateway-svc-tyk-gateway 8080:8080

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

curl http://localhost:8080/tyk/reload -H "x-tyk-authorization: CHANGEME"

for i in $(seq 1 20); do curl -s -o /dev/null http://localhost:8080/httpbin/get; done
```

### 8. Verify spans are flowing

```bash
# Find the DDOT pod on the same node as Tyk
TYKNODE=$(kubectl get pod -n tyk -l app=gateway-tyk-gateway \
  -o jsonpath='{.items[0].spec.nodeName}')
DDPOD=$(kubectl get pods -n datadog -l app=datadog-agent \
  -o wide --no-headers | grep "$TYKNODE" | awk '{print $1}')

# Check span counters
kubectl port-forward -n datadog pod/$DDPOD 8888:8888 &
curl -s http://localhost:8888/metrics | grep -E "accepted_spans|sent_spans" | grep -v "^#"
```

Expected:
```
otelcol_exporter_sent_spans{exporter="datadog",...}         <N>
otelcol_receiver_accepted_spans{receiver="otlp",...}        <N>
otelcol_receiver_refused_spans{receiver="otlp",...}         0
```

View traces: **Datadog → APM → Traces → filter `service:tyk`**

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Agent pods `CreateContainerError` | COS node image blocks `/usr/src` writes needed by system-probe | Use `--image-type=UBUNTU_CONTAINERD` node pool |
| No spans in DDOT metrics at all | `k8sattributes` RBAC missing — pod list forbidden, blocks span processing | Re-run `helm upgrade` with `ddot-values.yaml` which includes RBAC rules |
| Spans flowing but no host or container tags | Spans routing to DDOT on wrong node — `infraattributes` cannot find container | Ensure `datadog-otel` Service has `internalTrafficPolicy: Local` |
| `host.name` shows pod name instead of node name | Tyk uses kernel hostname (= pod name in K8s) as `host.name` | `resource` processor in `otel-config.yaml` copies `k8s.node.name` → `host.name` |
| `TYK_GW_OPENTELEMETRY_ENDPOINT` shows `$(HOST_IP)` literally | Kubernetes env var substitution requires `HOST_IP` to be defined before the referencing var; Tyk chart generates the env var before `extraEnvs` are evaluated | Use ClusterIP DNS name instead — do not rely on `$(HOST_IP)` substitution |
| Tyk initialized but no spans exported | gRPC connection timed out (default timeout is 1 second) | `connectionTimeout: 10` is set in `tyk-values.yaml` |
| API returns 404 after pod restart | Tyk OSS stores API definitions ephemerally in the pod filesystem | Re-POST to `/tyk/apis` and call `/tyk/reload` after every restart |
