---
name: verify-guide
description: "End-to-end verification of the rhoai-maas-guide content on a live OpenShift cluster. Installs MaaS, runs verify.sh, 8 blind-spot regression checks, and validates guide commands produce correct output."
argument-hint: "<API_URL> <USERNAME> <PASSWORD> [--skip-install] [--skip-verify-sh] [--skip-blindspot] [--skip-guide-content] [--with-observability] [--walkthrough-number N]"
allowed-tools: Bash(oc *), Bash(./*), Bash(envsubst *), Bash(curl *), Bash(jq *), Bash(grep *), Bash(dig *), Bash(wc *), Bash(awk *), Bash(sed *), Bash(head *), Bash(tail *), Bash(tr *), Bash(sort *), Bash(openssl *), Bash(python3 *), Bash(ls *), Bash(cat *), Bash(date *), Bash(mkdir *), Bash(echo *), Bash(bash *), Write, Read, AskUserQuestion
---

# Verify Guide Content on Live Cluster

End-to-end verification that the rhoai-maas-guide content is correct and working on a live OpenShift cluster. Runs installation, the 15-point E2E verify.sh, 8 blind-spot regression checks learned from 7 manual walkthroughs, and validates that AsciiDoc guide commands produce documented output.

## Arguments

Parse `$ARGUMENTS` for these values:

- `<API_URL>` -- (required) OpenShift API server URL (e.g. `https://api.cluster-kjcbs.dyn.redhatworkshops.io:6443`)
- `<USERNAME>` -- (required) Cluster admin username (e.g. `admin` or `kubeadmin`)
- `<PASSWORD>` -- (required) Cluster admin password
- `--skip-install` -- Skip Phase 1 (setup-maas.sh). Use when MaaS is already installed.
- `--skip-verify-sh` -- Skip Phase 2 (verify.sh E2E).
- `--skip-blindspot` -- Skip Phase 3 (8 blind-spot checks).
- `--skip-guide-content` -- Skip Phase 4 (AsciiDoc content verification).
- `--with-observability` -- Pass `--with-observability` to setup-maas.sh in Phase 1.
- `--walkthrough-number N` -- Label this run as walkthrough N (default: auto-increment from existing memory files).

If any of the three required arguments are missing, use `AskUserQuestion` to request them. Do NOT proceed without all three.

## Phases

| Phase | What it checks | Time |
|-------|---------------|------|
| 0 | Cluster login, preflight detection (platform, OCP version, GPU, nodes) | instant |
| 1 | Full MaaS installation via `./scripts/setup-maas.sh` | 15-30 min |
| 2 | 15-point E2E via `./manifests/06-verification/verify.sh --no-cleanup` | 3-5 min |
| 3 | 8 blind-spot regression checks (gateway OOM, MetalLB IP, RHCL version, etc.) | 1-2 min |
| 4 | AsciiDoc guide commands produce documented output | 1-2 min |
| 5 | Report generation + save findings to memory | instant |

## CRITICAL: Continuation Policy

**NEVER stop on failure.** Every phase and every individual check MUST run regardless of previous failures. Accumulate all results (PASS/FAIL/WARN with details) and report them together in Phase 5. The entire purpose of this skill is to find ALL issues in a single run, not to stop at the first one.

Track results using these counters (initialize at the start):
- `TOTAL_PASSED=0`
- `TOTAL_FAILED=0`
- `TOTAL_WARNED=0`
- Keep a list of all individual results with their phase, check name, and status.

## Instructions

Run from the guide repo root (`rhoai-maas-guide/`). Make sure your working directory is correct before starting.

---

### Phase 0: Cluster Login & Preflight

**Step 0a: Login**

```bash
oc login --server=<API_URL> -u <USERNAME> -p '<PASSWORD>' --insecure-skip-tls-verify=true
```

Verify the login succeeded with `oc whoami`. If it fails, report the error and ask the user to check credentials.

**Step 0b: Verify cluster-admin**

```bash
oc auth can-i '*' '*' --all-namespaces
```

Must return `yes`. If not, FAIL and warn the user that cluster-admin is required.

**Step 0c: Detect cluster characteristics**

Run these in parallel and record all values:

```bash
# Platform type (AWS, None, BareMetal, VSphere, etc.)
oc get infrastructure cluster -o jsonpath='{.status.platformStatus.type}'

# OCP version
oc get clusterversion version -o jsonpath='{.status.desired.version}'

# Node count and roles
oc get nodes --no-headers -o custom-columns='NAME:.metadata.name,ROLES:.metadata.labels.node-role\.kubernetes\.io/worker,STATUS:.status.conditions[-1].type'

# GPU detection
oc get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.metadata.labels.nvidia\.com/gpu\.present}{"\t"}{.metadata.labels.nvidia\.com/gpu\.memory}{"\n"}{end}'

# Cluster domain
oc get ingresses.config/cluster -o jsonpath='{.spec.domain}'
```

Record these values for the report:
- `PLATFORM_TYPE` (e.g. "None", "AWS")
- `OCP_VERSION` (e.g. "4.20.26")
- `NODE_COUNT`
- `HAS_GPU` (true/false)
- `GPU_VRAM` (in MiB, if present)
- `CLUSTER_DOMAIN`
- `IS_CLOUD` = true if PLATFORM_TYPE is AWS, Azure, or GCP; false otherwise

**Step 0d: Report preflight to user**

Print a summary of detected cluster characteristics before proceeding. Example:

```
Cluster: api.cluster-kjcbs.dyn.redhatworkshops.io:6443
Platform: None (non-cloud, MetalLB required)
OCP: 4.20.26
Nodes: 1 (SNO)
GPU: None
Domain: apps.cluster-kjcbs.dyn.redhatworkshops.io
```

---

### Phase 1: Install MaaS via setup-maas.sh

Skip this phase if `--skip-install` was passed. Report "Phase 1: SKIPPED (--skip-install)" and move to Phase 2.

**Step 1a: Run the installer**

```bash
cd /Users/rcarrata/Code/maas/rhoai-maas-guide
./scripts/setup-maas.sh 2>&1
```

If `--with-observability` was passed, add that flag:
```bash
./scripts/setup-maas.sh --with-observability 2>&1
```

**IMPORTANT**: This script can take 15-30 minutes. Run it and monitor the output. The script is idempotent and detects what's already installed.

**Step 1b: Handle known failures**

The script has a known `pipefail+grep` bug that can silently kill it during Phase 5 pod wait (when all pods are Running, `grep -v "Running|Completed"` returns exit code 1). If the script exits without completing:

1. Check if the script reached Phase 5 or later
2. If yes, resume: `./scripts/setup-maas.sh --from-phase 6`
3. Record the bug occurrence as a WARN

**Step 1c: Record result**

- If setup-maas.sh completes successfully (exit code 0): PASS
- If it fails but MaaS is partially installed: WARN with details of which phase failed
- If it fails completely: FAIL with error details, but CONTINUE to Phase 2

---

### Phase 2: Run verify.sh (15-point E2E)

Skip this phase if `--skip-verify-sh` was passed.

**Step 2a: Run verification**

```bash
cd /Users/rcarrata/Code/maas/rhoai-maas-guide
./manifests/06-verification/verify.sh --no-cleanup 2>&1
```

We use `--no-cleanup` so the test resources stay deployed for Phase 3 and 4 checks.

**Step 2b: Parse results**

From the output, extract:
- Every line containing `[PASS]` or `[FAIL]` - record each individually
- The final summary line showing total PASSED/FAILED counts
- Any `[WARN]` or `[ERROR]` lines

Add the verify.sh PASSED count to `TOTAL_PASSED` and FAILED count to `TOTAL_FAILED`.

**Step 2c: Handle failures**

If verify.sh itself crashes (not individual test failures, but the script itself), record it as a FAIL and continue to Phase 3. Common crash causes:
- Gateway pod restarting (HTTP 000 on health endpoint)
- DNS not propagated yet for cloud LB

---

### Phase 3: Blind Spot Checks

Skip this phase if `--skip-blindspot` was passed.

These 8 checks were developed across 7 manual walkthroughs (v1-v7). They catch regressions that verify.sh does not test. Run ALL 8 regardless of individual failures.

#### Check 1: MetalLB/Gateway IP Correctness

```bash
PLATFORM=$(oc get infrastructure cluster -o jsonpath='{.status.platformStatus.type}')
GATEWAY_ADDR=$(oc get gateway maas-default-gateway -n openshift-ingress -o jsonpath='{.status.addresses[0].value}' 2>/dev/null || echo "NOT_FOUND")
```

**If non-cloud (None/BareMetal/VSphere/OpenStack):**
- Verify MetalLB IPAddressPool exists: `oc get ipaddresspool maas-pool -n metallb-system`
- Verify Gateway address is NOT a node InternalIP:
  ```bash
  NODE_IPS=$(oc get nodes -o jsonpath='{.items[*].status.addresses[?(@.type=="InternalIP")].address}')
  ```
  If `GATEWAY_ADDR` is in `NODE_IPS`, FAIL (MetalLB IP collision - this breaks OAuth/console)
- PASS if Gateway has a valid address that is not a node IP

**If cloud (AWS/Azure/GCP):**
- Verify Gateway address contains a hostname (e.g. `.elb.amazonaws.com`)
- PASS if hostname present, FAIL if bare IP or NOT_FOUND

#### Check 2: Gateway Memory & Restart Count

```bash
DEPLOY_NAME=$(oc get deployment -n openshift-ingress -l gateway.networking.k8s.io/gateway-name=maas-default-gateway --no-headers -o custom-columns='NAME:.metadata.name' 2>/dev/null | head -1)

MEM_LIMIT=$(oc get deployment "$DEPLOY_NAME" -n openshift-ingress -o jsonpath='{.spec.template.spec.containers[0].resources.limits.memory}' 2>/dev/null || echo "UNKNOWN")

RESTART_COUNT=$(oc get pods -n openshift-ingress -l gateway.networking.k8s.io/gateway-name=maas-default-gateway -o jsonpath='{.items[0].status.containerStatuses[0].restartCount}' 2>/dev/null || echo "0")

LAST_TERM_REASON=$(oc get pods -n openshift-ingress -l gateway.networking.k8s.io/gateway-name=maas-default-gateway -o jsonpath='{.items[0].status.containerStatuses[0].lastState.terminated.reason}' 2>/dev/null || echo "")
```

- FAIL if `MEM_LIMIT` is `1Gi` (should be `2Gi` after OOMKill fix)
- FAIL if `RESTART_COUNT` >= 3
- WARN if `LAST_TERM_REASON` is `OOMKilled`
- PASS if memory is 2Gi AND restarts < 3 AND no OOMKill

#### Check 3: OAuth/Console Accessible

```bash
CONSOLE_HOST=$(oc get route console -n openshift-console -o jsonpath='{.spec.host}' 2>/dev/null || echo "")
if [ -n "$CONSOLE_HOST" ]; then
    CONSOLE_CODE=$(curl -sk --connect-timeout 10 --max-time 15 -o /dev/null -w '%{http_code}' "https://${CONSOLE_HOST}" 2>/dev/null || echo "000")
fi
```

- PASS if HTTP code is 200, 301, 302, or 403
- FAIL if HTTP 000 (connection refused/reset - strong indicator of MetalLB IP collision)
- WARN if any other HTTP code

#### Check 4: RHOAI Dashboard Accessible

```bash
DASHBOARD_HOST=$(oc get route rhods-dashboard -n redhat-ods-applications -o jsonpath='{.spec.host}' 2>/dev/null || echo "")
if [ -n "$DASHBOARD_HOST" ]; then
    DASHBOARD_CODE=$(curl -sk --connect-timeout 10 --max-time 15 -o /dev/null -w '%{http_code}' "https://${DASHBOARD_HOST}" 2>/dev/null || echo "000")
fi
```

- PASS if HTTP code is 200, 301, 302, or 403 (redirect to login is expected)
- FAIL if 000 or empty
- WARN if route does not exist (RHOAI dashboard not deployed)

#### Check 5: llm Namespace Labels

```bash
NS_LABELS=$(oc get namespace llm -o jsonpath='{.metadata.labels}' 2>/dev/null || echo "{}")
```

Verify these labels exist:
- `opendatahub.io/dashboard` = `"true"`
- `maas.opendatahub.io/gateway-access` = `"true"`
- `opendatahub.io/generated-namespace` = `"true"`

- PASS if all 3 labels present
- FAIL for each missing label (report which ones)
- If namespace llm does not exist, WARN (model may not have been deployed)

#### Check 6: RHCL Version and Approval Mode

```bash
RHCL_CSV=$(oc get csv -n openshift-operators --no-headers 2>/dev/null | grep rhcl || echo "")
RHCL_VERSION=$(echo "$RHCL_CSV" | awk '{print $1}' | grep -oE 'v[0-9]+\.[0-9]+\.[0-9]+' || echo "UNKNOWN")
RHCL_APPROVAL=$(oc get subscription -n openshift-operators -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.installPlanApproval}{"\n"}{end}' 2>/dev/null | grep rhcl || echo "")
```

- PASS if RHCL version is v1.4.2+ AND installPlanApproval is Automatic
- WARN if installPlanApproval is Manual (old pin - should be Automatic now)
- FAIL if RHCL is not installed or version cannot be determined

#### Check 7: HF Xet Storage Behavior

```bash
INIT_STUCK=$(oc get pods -n llm --no-headers 2>/dev/null | grep -E 'Init:[0-9]' || true)
```

- If any pods are stuck in Init phase, check how long:
  ```bash
  oc get pods -n llm -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.phase}{"\t"}{.status.startTime}{"\n"}{end}'
  ```
- PASS if no pods stuck in Init, or all pods Running/Completed
- WARN if pods in Init but started < 2 minutes ago (may still be pulling)
- FAIL if pods stuck in Init > 5 minutes (possible Xet hang)

Also check for the fix env vars:
```bash
oc get pods -n llm -o jsonpath='{range .items[*].spec.initContainers[*]}{.name}{"\t"}{range .env[*]}{.name}={.value}{" "}{end}{"\n"}{end}' 2>/dev/null || true
```
Report whether `HF_XET_HIGH_PERFORMANCE` or `HF_HUB_DISABLE_XET` is set (informational).

#### Check 8: Independent Inference Test

This tests inference OUTSIDE of verify.sh, using the guide's documented subscription names.

```bash
CLUSTER_DOMAIN=$(oc get ingresses.config/cluster -o jsonpath='{.spec.domain}')
MAAS_URL="https://maas.${CLUSTER_DOMAIN}"
TOKEN=$(oc whoami -t)
```

**Step 8a: Create API key with simulator-free subscription**

```bash
API_KEY_RESPONSE=$(curl -sk --connect-timeout 10 --max-time 30 \
    -X POST "${MAAS_URL}/maas-api/v1/api-keys" \
    -H "Authorization: Bearer ${TOKEN}" \
    -H "Content-Type: application/json" \
    -d '{"name":"verify-guide-test","subscription":"simulator-free","expiresIn":"10m"}')
API_KEY=$(echo "$API_KEY_RESPONSE" | jq -r '.key // empty')
API_KEY_ID=$(echo "$API_KEY_RESPONSE" | jq -r '.id // empty')
```

If API key creation fails (empty key), try with the verify.sh subscription name `simulator-subscription` as fallback. Record which subscription name worked.

- FAIL if neither subscription name works

**Step 8b: List models**

```bash
MODELS_RESPONSE=$(curl -sk --connect-timeout 10 --max-time 30 \
    "${MAAS_URL}/maas-api/v1/models" \
    -H "Authorization: Bearer ${API_KEY}")
MODEL_COUNT=$(echo "$MODELS_RESPONSE" | jq '.data | length' 2>/dev/null || echo "0")
MODEL_URL=$(echo "$MODELS_RESPONSE" | jq -r '.data[0].url // empty' 2>/dev/null)
```

- PASS if MODEL_COUNT > 0
- FAIL if no models returned

**Step 8c: Send inference request**

```bash
INFERENCE_RESPONSE=$(curl -sk --connect-timeout 10 --max-time 60 \
    -w '\n%{http_code}' \
    "${MODEL_URL}/v1/chat/completions" \
    -H "Authorization: Bearer ${API_KEY}" \
    -H "Content-Type: application/json" \
    -d '{"model":"facebook/opt-125m","messages":[{"role":"user","content":"Say hello"}],"max_tokens":20}')
INFERENCE_CODE=$(echo "$INFERENCE_RESPONSE" | tail -1)
INFERENCE_BODY=$(echo "$INFERENCE_RESPONSE" | sed '$d')
```

- PASS if HTTP 200 and response body contains `choices`
- FAIL if HTTP != 200 or response is empty/invalid

**Step 8d: Cleanup test API key**

```bash
curl -sk --connect-timeout 10 --max-time 15 \
    -X DELETE "${MAAS_URL}/maas-api/v1/api-keys/${API_KEY_ID}" \
    -H "Authorization: Bearer ${TOKEN}" 2>/dev/null || true
```

---

### Phase 4: Guide Content Verification

Skip this phase if `--skip-guide-content` was passed.

These checks verify that commands documented in the AsciiDoc guide pages produce the expected output. If the guide says "run X and you should see Y", we run X and check for Y.

#### Content Check 1: Subscription Names (05-maas-models.adoc)

The guide documents `simulator-free` and `simulator-premium` subscription names.

```bash
SUBS=$(oc get maassubscription -n models-as-a-service --no-headers -o custom-columns='NAME:.metadata.name' 2>/dev/null || echo "")
```

- PASS if output contains `simulator-free` (at minimum - premium may or may not be deployed)
- FAIL if `simulator-free` is not present (subscription name mismatch bug - was a real bug in v1)

#### Content Check 2: MaaS CRDs (04-rhoai-config.adoc)

The guide says `oc get crd | grep maas.opendatahub.io` should show MaaS CRDs.

```bash
MAAS_CRDS=$(oc get crd --no-headers 2>/dev/null | grep maas.opendatahub.io || echo "")
```

Expected CRDs: `maasmodelrefs`, `maasauthpolicies`, `maassubscriptions`

- PASS if all 3 CRDs found
- FAIL for each missing CRD

#### Content Check 3: Health Endpoint Body (06-verification.adoc)

The guide documents that the health endpoint returns `{"status":"healthy"}`.

```bash
HEALTH_BODY=$(curl -sk --connect-timeout 10 --max-time 30 "https://maas.${CLUSTER_DOMAIN}/maas-api/health" 2>/dev/null || echo "")
HEALTH_STATUS=$(echo "$HEALTH_BODY" | jq -r '.status // empty' 2>/dev/null || echo "")
```

- PASS if `HEALTH_STATUS` is `healthy`
- FAIL if response body is different from documented output

#### Content Check 4: Manual Verification Commands (06-verification.adoc)

Run the exact commands from the verification guide and check expected values:

```bash
# Gateway Programmed
GW_STATUS=$(oc get gateway maas-default-gateway -n openshift-ingress -o jsonpath='{.status.conditions[?(@.type=="Programmed")].status}' 2>/dev/null || echo "UNKNOWN")

# Key deployments available
PG_AVAIL=$(oc get deployment postgres -n redhat-ods-applications -o jsonpath='{.status.availableReplicas}' 2>/dev/null || echo "0")
API_AVAIL=$(oc get deployment maas-api -n redhat-ods-applications -o jsonpath='{.status.availableReplicas}' 2>/dev/null || echo "0")
CTRL_AVAIL=$(oc get deployment maas-controller -n redhat-ods-applications -o jsonpath='{.status.availableReplicas}' 2>/dev/null || echo "0")
AUTH_AVAIL=$(oc get deployment authorino -n kuadrant-system -o jsonpath='{.status.availableReplicas}' 2>/dev/null || echo "0")

# ModelsAsServiceReady
MAAS_READY=$(oc get datasciencecluster default-dsc -o jsonpath='{.status.conditions[?(@.type=="ModelsAsServiceReady")].status}' 2>/dev/null || echo "UNKNOWN")
```

- PASS for each value that matches expected (Gateway=True, each deployment >= 1, ModelsAsServiceReady=True)
- FAIL for each mismatch

#### Content Check 5: Operator CSV Status (01-prerequisites.adoc)

The guide documents 4 required operators that must reach Succeeded:

```bash
# RHOAI
oc get csv -n redhat-ods-operator --no-headers -o custom-columns='NAME:.metadata.name,PHASE:.status.phase' 2>/dev/null | grep rhods

# RHCL
oc get csv -n openshift-operators --no-headers -o custom-columns='NAME:.metadata.name,PHASE:.status.phase' 2>/dev/null | grep rhcl

# cert-manager
oc get csv -n cert-manager-operator --no-headers -o custom-columns='NAME:.metadata.name,PHASE:.status.phase' 2>/dev/null | grep cert-manager

# LWS
oc get csv -n openshift-lws-operator --no-headers -o custom-columns='NAME:.metadata.name,PHASE:.status.phase' 2>/dev/null | grep leader-worker-set
```

- PASS for each operator with phase Succeeded
- FAIL for each operator not in Succeeded or not found

---

### Phase 5: Report Generation

#### Step 5a: Collect operator versions

```bash
RHOAI_VERSION=$(oc get csv -n redhat-ods-operator --no-headers 2>/dev/null | grep rhods | awk '{print $1}' | grep -oE '[0-9]+\.[0-9]+\.[0-9]+' || echo "UNKNOWN")
RHCL_VERSION=$(oc get csv -n openshift-operators --no-headers 2>/dev/null | grep rhcl | awk '{print $1}' | grep -oE 'v[0-9]+\.[0-9]+\.[0-9]+' || echo "UNKNOWN")
CERTMGR_VERSION=$(oc get csv -n cert-manager-operator --no-headers 2>/dev/null | grep cert-manager | awk '{print $1}' | grep -oE '[0-9]+\.[0-9]+\.[0-9]+' || echo "UNKNOWN")
LWS_VERSION=$(oc get csv -n openshift-lws-operator --no-headers 2>/dev/null | grep leader-worker-set | awk '{print $1}' | grep -oE '[0-9]+\.[0-9]+\.[0-9]+' || echo "UNKNOWN")
SMESH_VERSION=$(oc get csv -n openshift-operators --no-headers 2>/dev/null | grep servicemesh | awk '{print $1}' | grep -oE '[0-9]+\.[0-9]+\.[0-9]+' || echo "UNKNOWN")
COO_VERSION=$(oc get csv -n openshift-cluster-observability-operator --no-headers 2>/dev/null | grep cluster-observability | awk '{print $1}' | grep -oE '[0-9]+\.[0-9]+\.[0-9]+' || echo "N/A")
```

#### Step 5b: Print the report

Print a structured report to the user with these sections:

**Header:**
```
=== VERIFY-GUIDE REPORT ===
Date: <today's date>
Cluster: <API_URL>
Platform: <PLATFORM_TYPE> | OCP: <OCP_VERSION> | Nodes: <NODE_COUNT> | GPU: <yes/no>
RHOAI: <version> | RHCL: <version> | Service Mesh: <version>
```

**Summary:**
```
Total: X checks | PASSED: Y | FAILED: Z | WARNINGS: W
```

**Phase Results Table:**
For each phase, list every individual check with PASS/FAIL/WARN status.

**Failures (if any):**
List every FAIL with its phase, check name, expected value, and actual value.

**Warnings (if any):**
List every WARN with details.

**Operator Versions Table:**
| Operator | Version |
|----------|---------|
| RHOAI | X.Y.Z |
| RHCL | vX.Y.Z |
| ... | ... |

#### Step 5c: Save findings to memory

Use the `Write` tool to save the findings as a memory file. Determine the walkthrough number:
- If `--walkthrough-number N` was passed, use N
- Otherwise, check existing memory files and auto-increment

Save to: `/Users/rcarrata/.claude/projects/-Users-rcarrata-Code-maas/memory/project_naive-user-guide-findings-v<N>.md`

Use the same format as previous walkthrough findings (see existing memory files for reference). Include:
- Frontmatter with name, description, metadata (type: project)
- Date, cluster, OCP version, platform, RHOAI version, RHCL version
- Result summary (X/Y verification, X/Y blind spots)
- Issues found (numbered, with severity and details)
- Positive findings
- Blind spots verified table
- Operator versions table
- Comparison with previous walkthroughs

#### Step 5d: Cleanup

Run cleanup to remove test resources left by `--no-cleanup`:

```bash
cd /Users/rcarrata/Code/maas/rhoai-maas-guide
./manifests/06-verification/verify.sh --cleanup-only 2>&1
```

Report cleanup status.

---

## Known Bugs to Watch For

This list was built from 7 walkthroughs. The blind-spot checks target these specifically:

| Bug | Walkthrough | Status |
|-----|------------|--------|
| Gateway OOMKill at 1Gi | v1-v3 | FIXED (ConfigMap 2Gi) |
| Wrong subscription name (simulator-subscription vs simulator-free) | v1 | FIXED |
| HF Xet storage-initializer hang | v4 | FIXED (RHOAI 3.4.2+) |
| pipefail+grep kills setup-maas.sh silently | v7 | OPEN |
| Phase 1 CSV timeout too short (900s) | v3, v7 | OPEN |
| Help text phase numbering mismatch | v7 | OPEN |
| MetalLB IP collision with node IP | v6 | FIXED (PR #40) |
| Kuadrant Wasm plugin startup race | v3 | UPSTREAM |
| RHCL v1.4.0 Wasm/UI bugs | v5 | FIXED (v1.4.2, pin removed) |

## Troubleshooting

- **`oc login` fails**: Verify the API URL includes the port (`:6443`). Try `--insecure-skip-tls-verify=true`. Check if the cluster is reachable.
- **setup-maas.sh fails at Phase 1**: Operator CSVs may need more time. Check `oc get csv -A --no-headers | grep -v Succeeded`.
- **setup-maas.sh exits silently at Phase 5**: Known pipefail+grep bug. Resume with `./scripts/setup-maas.sh --from-phase 6`.
- **verify.sh gets HTTP 000**: Gateway pod likely restarting. Check `oc get pods -n openshift-ingress | grep maas-default-gateway`.
- **Blind spot check 3 fails (OAuth unreachable)**: MetalLB IP collision. The MetalLB IP shares port 443 with the node, intercepting traffic.
- **Check 8 fails on API key creation**: Try both `simulator-free` and `simulator-subscription` subscription names. The verify.sh creates its own subscription while the guide manifests use different names.
- **Content check 1 fails on subscription names**: If verify.sh was the only thing that ran (no Phase 1), only `simulator-subscription` exists, not `simulator-free`. This is expected when using `--skip-install`.
