# cert-manager Integration for RHOAI MaaS

How to use cert-manager instead of OpenShift service-ca for TLS certificates in the MaaS stack.

## Current State

All components use OpenShift's built-in service serving certificates (`service.beta.openshift.io/serving-cert-secret-name` annotation). cert-manager is installed but only used by the service mesh gateway certs in `istio-system`.

## Feasibility by Component

| Component | Current Mechanism | cert-manager? | Recommended? |
|---|---|---|---|
| Authorino gRPC TLS | service-ca annotation | Yes — official upstream CRs | Yes |
| MaaS Gateway | Router wildcard cert | Yes — gateway-shim | Yes |
| KServe webhook server | RHOAI operator / service-ca | Possible but operator overwrites | No |
| Serving pod TLS (`/var/run/kserve/tls/`) | Self-signed in Go code | No integration point | Not possible |

## Prerequisite: ClusterIssuer

Create a CA chain. For lab (self-signed):

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned-bootstrap
spec:
  selfSigned: {}
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: internal-root-ca
  namespace: cert-manager
spec:
  isCA: true
  commonName: internal-root-ca
  secretName: internal-root-ca-secret
  issuerRef:
    name: selfsigned-bootstrap
    kind: ClusterIssuer
---
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: internal-ca-issuer
spec:
  ca:
    secretName: internal-root-ca-secret
```

For production, replace `internal-ca-issuer` with an ACME (Let's Encrypt) or Vault-backed ClusterIssuer.

### Trusting the CA Cluster-Wide

If using self-signed, inject the CA into the cluster trust store:

```bash
oc get secret internal-root-ca-secret -n cert-manager \
  -o jsonpath='{.data.ca\.crt}' | base64 -d > internal-ca-bundle.crt
oc create configmap custom-ca \
  --from-file=ca-bundle.crt=internal-ca-bundle.crt -n openshift-config
oc patch proxy/cluster --type=merge \
  --patch='{"spec":{"trustedCA":{"name":"custom-ca"}}}'
```

> **Note:** On tallgeese, `ultra-lab-ca` is already in the proxy trusted CA. You could potentially reuse it as the cert-manager CA issuer instead of creating a new chain, avoiding a node rollout.

> **Warning:** Changing the Proxy `trustedCA` triggers a node rollout via the MCO. Always confirm this is acceptable before applying.

## Component 1: Authorino TLS Listener

Official upstream cert-manager manifests exist at:
`https://raw.githubusercontent.com/Kuadrant/authorino/main/deploy/certs.yaml`

### Step 1: Remove service-ca annotation

Remove `service.beta.openshift.io/serving-cert-secret-name: authorino-server-cert` from the Authorino Service in `manifests/02-platform-config/kuadrant/service-annotation.yaml`.

### Step 2: Create cert-manager resources

```yaml
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: authorino-ca-root
  namespace: kuadrant-system
spec:
  selfSigned: {}
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: authorino-ca-cert
  namespace: kuadrant-system
spec:
  commonName: "*.kuadrant-system.svc"
  isCA: true
  issuerRef:
    kind: Issuer
    name: authorino-ca-root
  secretName: authorino-ca-cert
---
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: authorino-ca
  namespace: kuadrant-system
spec:
  ca:
    secretName: authorino-ca-cert
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: authorino-server-cert
  namespace: kuadrant-system
spec:
  dnsNames:
    - authorino-authorino-authorization
    - authorino-authorino-authorization.kuadrant-system.svc
    - authorino-authorino-authorization.kuadrant-system.svc.cluster.local
  issuerRef:
    kind: Issuer
    name: authorino-ca
  secretName: authorino-server-cert
```

The Authorino CR itself doesn't need changes — it already references `certSecretRef.name: authorino-server-cert`, which cert-manager will populate.

## Component 2: MaaS Gateway

Uses cert-manager's gateway-shim controller, which is **disabled by default** on the OpenShift cert-manager operator.

### Step 1: Enable gateway-shim

```yaml
apiVersion: operator.openshift.io/v1alpha1
kind: CertManager
metadata:
  name: cluster
spec:
  controllerConfig:
    overrideArgs:
    - '--enable-gateway-api'
```

Then restart: `oc rollout restart deployment cert-manager -n cert-manager`

### Step 2: Annotate the Gateway

Add `cert-manager.io/cluster-issuer` annotation and change the TLS secret reference:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: maas-default-gateway
  namespace: openshift-ingress
  annotations:
    cert-manager.io/cluster-issuer: internal-ca-issuer
spec:
  listeners:
   - name: https
     hostname: maas.apps.tallgeese.ultra.lab
     port: 443
     protocol: HTTPS
     tls:
       certificateRefs:
       - name: maas-gateway-tls  # cert-manager creates this Secret
       mode: Terminate
```

cert-manager auto-creates a Certificate CR and populates `maas-gateway-tls` with the cert/key.

### For Let's Encrypt (production)

Use an ACME ClusterIssuer with `gatewayHTTPRoute` solver:

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@example.com
    privateKeySecretRef:
      name: letsencrypt-prod-account-key
    solvers:
      - http01:
          gatewayHTTPRoute:
            parentRefs:
              - name: maas-default-gateway
                namespace: openshift-ingress
```

## Components NOT Feasible

### KServe Webhook Server

The RHOAI operator manages these certs. You'd need to set resources to `opendatahub.io/managed: "false"` and manually create certs matching the upstream KServe pattern. Not worth it — these are internal-only webhook certs.

### Serving Pod TLS (`/var/run/kserve/tls/`)

Generated by the KServe controller in Go code (`workload_tls_self_signed.go`). There is no integration point for cert-manager. The code has a comment suggesting future extensibility ("Distro hooks may supply a separate CA cert") but nothing is implemented.

## Sources

- Authorino cert-manager manifests: https://raw.githubusercontent.com/Kuadrant/authorino/main/deploy/certs.yaml
- cert-manager Gateway API docs: https://cert-manager.io/docs/usage/gateway/
- Red Hat OCP cert-manager operator: https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/security_and_compliance/cert-manager-operator-for-red-hat-openshift
- KServe self-signed TLS code: https://github.com/kserve/kserve/blob/master/pkg/controller/v1alpha2/llmisvc/workload_tls_self_signed.go
