# Tallgeese Deployment Notes

Customizations and workarounds applied to the upstream `rh-aiservices-bu/rhoai-maas-guide` for the tallgeese lab cluster (OCP 4.22, bare-metal, 3-node compact, no GPUs).

## Pre-existing Infrastructure

These components were already installed before running the guide — the setup script's Phase 1 operator installs are mostly no-ops:

| Component | Notes |
|-----------|-------|
| RHOAI 3.4 | `rhods-operator.3.4.2` |
| cert-manager | `cert-manager-operator.v1.20.0` |
| MetalLB | `metallb-operator.v4.22.0`, IPAddressPool `ultra-lab-pool` (192.168.8.80-99) |
| Tempo Operator | Installed cluster-wide via `openshift-operators` |
| OpenTelemetry Operator | Same |
| COO | Same |

## Customizations

### 1. PostgreSQL Image

The guide uses `registry.redhat.io/rhel9/postgresql-16:latest`. We initially tried `registry.access.redhat.com/hi/postgresql:17` but it fails with OpenShift's restricted SCC (`mkdir: cannot create directory '/var/lib/pgsql': Permission denied`). Reverted to the guide's original image.

### 2. DSC — Minimal Component Set

The guide's DSC enables many components as Managed. We only enabled `kserve.modelsAsService: Managed` and kept everything else matching tallgeese's existing state to minimize resource usage:

**Managed:** dashboard, kserve (+ modelsAsService, nim), llamastackoperator, workbenches
**Removed:** feastoperator, mlflowoperator, modelregistry, ray, sparkoperator, trainer, trainingoperator, trustyai, kueue

> **Note:** Model Registry is Removed — if MaaS features break, enabling it may be required.

### 3. RHCL Version

The guide pins RHCL to v1.3.4 via Manual install plan approval (WASM bug in v1.4.0). On OCP 4.22, v1.3.4 doesn't exist in the catalog — only v1.4.2 is available. We patched the subscription's `startingCSV` to `v1.4.2` and approved the install plan.

### 4. Duplicate OperatorGroups

The setup script's Phase 1 kustomize manifests create OperatorGroups for cert-manager, RHOAI, and (in Phase 7) Tempo. When these operators are already installed, the duplicate OGs cause CSVs to fail with "csv created in namespace with multiple operatorgroups." **Fix:** Delete the duplicate OG created by the script; the original one stays.

### 5. MetalLB — Reuse Existing Pool

No separate `maas-pool` was created. The MaaS gateway uses the existing `ultra-lab-pool` (192.168.8.80-99) and received IP 192.168.8.83.

### 6. DNS

Manually added `maas.apps.tallgeese.ultra.lab → 192.168.8.83` on the GL.iNet router (192.168.8.1).

### 7. Qwen 2.5 0.5B Model (CPU)

Added a lightweight real LLM as an alternative to the simulator. Key learnings:

- **vLLM CPU ignores `--kv-cache-memory-bytes`** — must use `VLLM_CPU_KVCACHE_SPACE` env var (value in GB). Without it, vLLM defaults to using ALL available node memory for KV cache (~46GB), causing immediate OOMKill.
- Set `VLLM_CPU_KVCACHE_SPACE=2` (2GB KV cache)
- `--max-model-len=4096` works within the same ~4.9GB memory footprint
- Resources: requests 2 CPU / 6Gi, limits 4 CPU / 12Gi

### 8. Playground (LlamaStackDistribution)

The "Add to playground" button in the RHOAI UI creates a `LlamaStackDistribution` CR. Key settings:

- `VLLM_MAX_TOKENS=512` — must be lower than the smallest model's context length (simulator is 1024). Setting it too high causes 400 errors that LlamaStack surfaces as 500s.
- `VLLM_TLS_VERIFY=false` — required for self-signed certs
- API tokens are set to `fake` — the playground authenticates via the MaaS subscription, not direct vLLM tokens

### 9. DSCI Monitoring — Metrics and Traces

The guide's DSCI intentionally omits `metrics` and `traces` config (Phase 7 patches it in). We added it directly with 5-day retention:

```yaml
monitoring:
  metrics:
    replicas: 1
    storage:
      size: 5Gi
      retention: 5d
  traces:
    sampleRatio: "0.1"
    storage:
      backend: pv
      retention: 120h
```

### 10. Missing `prometheus-web-tls-ca` Secret

The RHOAI operator creates a ConfigMap named `prometheus-web-tls-ca` but the MonitoringStack Prometheus pod spec mounts it as a Secret. **Fix:** Manually create a Secret from the ConfigMap's data:

```bash
CA_DATA=$(oc get configmap prometheus-web-tls-ca -n redhat-ods-monitoring -o jsonpath='{.data.service-ca\.crt}')
oc create secret generic prometheus-web-tls-ca -n redhat-ods-monitoring --from-literal=service-ca.crt="$CA_DATA"
```

### 11. payload-processing Network Policy

The `openshift-ingress-deny-all` NetworkPolicy blocks the `payload-processing` pod from reaching the kube API, causing CrashLoopBackOff. **Fix:** Add an allow policy:

```bash
oc apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: payload-processing-allow
  namespace: openshift-ingress
spec:
  podSelector:
    matchLabels:
      app: payload-processing
  policyTypes: [Ingress, Egress]
  ingress: [{}]
  egress: [{}]
EOF
```

### 12. Phase 7 Operator Conflicts

Phase 7 tries to install Tempo/OpenTelemetry/COO into dedicated namespaces, but these operators are already installed cluster-wide via `openshift-operators`. The duplicate subscriptions and OperatorGroups cause CSV failures. **Fix:** Skip operator installs and apply only the telemetry resources:

```bash
oc apply -k manifests/07-observability/telemetry/
```

### 13. MaaS Gateway HPA — Limit Replicas

The OpenShift Gateway Controller creates an HPA with min=2 max=10 by default. On a lab cluster this scales to 10 replicas on startup CPU spikes. The HPA is owned by the Gateway and can't be patched directly — the controller reverts it.

**Fix:** The parametersRef ConfigMap supports a `horizontalPodAutoscaler` key (same pattern as `deployment` and `service`). Added to `manifests/02-platform-config/gateway-resources.yaml`:

```yaml
data:
  horizontalPodAutoscaler: |
    spec:
      minReplicas: 1
      maxReplicas: 2
```

> **Note:** Gateway annotations like `gateway.istio.io/autoscale-min` and `autoscaling.istio.io/minReplicas` do NOT work — the OpenShift Gateway Controller ignores them. Only the ConfigMap key works.

### 14. Missing Perses Datasources (RHOAI 3.4.x Bug — Fixed in 3.5 EA)

The RHOAI 3.4.x operator tries to create `PersesDatasource` CRs but fails because its manifests omit the `namespace` field in `spec.client.tls.caCert`, which the v1alpha2 CRD validation requires. This causes the monitoring controller to spam errors every ~500ms.

**Fix:** Upgrade to RHOAI 3.5.0-ea.2 (beta channel). The beta channel has no upgrade path from 3.4.x — you must delete the subscription and recreate it:

```bash
oc delete subscription rhods-operator -n redhat-ods-operator
oc delete csv rhods-operator.3.4.3 -n redhat-ods-operator
oc apply -f - <<EOF
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: rhods-operator
  namespace: redhat-ods-operator
spec:
  channel: beta
  installPlanApproval: Automatic
  name: rhods-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
  startingCSV: rhods-operator.3.5.0-ea.2
EOF
```

The 3.5 operator creates all datasources correctly and the observability dashboard works.

### 15. Missing `prometheus-web-tls-ca` Secret

The RHOAI operator creates a ConfigMap named `prometheus-web-tls-ca` but the MonitoringStack Prometheus pod spec mounts it as a Secret. **Fix:**

```bash
CA_DATA=$(oc get configmap prometheus-web-tls-ca -n redhat-ods-monitoring -o jsonpath='{.data.service-ca\.crt}')
oc create secret generic prometheus-web-tls-ca -n redhat-ods-monitoring --from-literal=service-ca.crt="$CA_DATA"
```

## Open Issues

- **istio-pod-monitor** scrapes port 15021 (health port, not metrics) causing permanent 33% TargetDown alert — Kuadrant-managed, can't patch without operator overwriting
- **MaaS Usage tab** shows 0 — may need time for metrics to accumulate, or a wiring issue
- **LlamaStack 500 errors on rate limit** — LlamaStack surfaces 429 (Too Many Requests) from MaaS gateway as 500 Internal Server Error
