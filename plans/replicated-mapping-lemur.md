# Config Consistency Fixes Across GRL Repos

## Context

Rapid feature development across grl-platform, GraphSandbox, and Ontology-Manager has introduced configuration inconsistencies — confusing defaults, hardcoded values, and naming divergence. This batch fixes 8 issues to reduce the config surface and make deployments less error-prone. (Image tag pattern #6 needs no changes — both repos already support `--set global.environment=dev`.)

---

## Phase 1: grl-platform changes

### 1a. Fix gateway.host default + validation
**`k8s/keycloak/values.yaml`** line 4: change `host: ontology.example.com` → `host: ""`
**`k8s/keycloak/templates/_validate.tpl`** line 6: simplify to `{{- if not .Values.gateway.host -}}`

### 1b. Add client hostname validation
**`k8s/keycloak/templates/_validate.tpl`** lines 9-10: replace empty loop with hostname check:
```tpl
{{- range $clientId, $client := .Values.clients -}}
  {{- if not $client.hostname -}}
    {{- fail (printf "clients.%s.hostname is required ..." $clientId) -}}
  {{- end -}}
{{- end -}}
```

### 1c. Extract token lifespans + seed users to values
**`k8s/keycloak/values.yaml`**: add `realm:` block with lifespans and seed users list
**`k8s/keycloak/templates/keycloak-realm-configmap.yaml`**:
- Lines 215-217: reference `.Values.realm.accessTokenLifespan` etc.
- Lines 156-204: replace hardcoded users with `range .Values.realm.seedUsers`
- Service account user (lines 205-212) stays hardcoded — it's structural

### 1d. Expand discovery ConfigMap
**`k8s/keycloak/templates/discovery-configmap.yaml`**: add `realm`, `serviceFqdn`, `servicePort`
**`k8s/oxigraph/templates/discovery-configmap.yaml`** (NEW): export per-instance FQDNs and port

### 1e. Rebuild subchart dependencies
`helm dependency build k8s/grl-platform`

---

## Phase 2: GraphSandbox changes

### 2a. Set explicit platformNamespace default
**`k8s/values.yaml`** line 63: `platformNamespace: grl-platform`

### 2b. Flatten gateway ref naming
Standardize to `gateway.name`/`gateway.namespace` matching grl-platform and OM.

**Files:**
- `k8s/values.yaml` lines 46-48: flatten `gatewayRef:` to `name:`/`namespace:` directly under `gateway:`
- `k8s/templates/httproute.yaml` lines 12-13: `.Values.gateway.name`/`.Values.gateway.namespace`
- `builder/k8s/chart/values.yaml` lines 64-66: same flattening
- `builder/k8s/chart/templates/httproute.yaml` lines 15-16, 161-162, 189-190: same
- `shared/src/helm.ts` lines 13-16: flatten `GatewayConfig` type (remove `gatewayRef` nesting, add `name`/`namespace` directly)

---

## Phase 3: Ontology-Manager changes

### 3a. Add platformNamespace + clear FQDN defaults
**`k8s/ontology-manager/values.yaml`**:
- Add `platformNamespace: grl-platform`
- Change `keycloak.serviceFqdn` and `oxigraph.serviceFqdn` to `""` (looked up if empty)
- Change `gateway.host` from `ontology.local` to `""`

### 3b. Add ConfigMap lookup helpers
**`k8s/ontology-manager/templates/_helpers.tpl`**: add 3 helpers:
- `ontology-manager.keycloakServiceFqdn` — lookup from `grl-platform-keycloak` ConfigMap, fall back to explicit value
- `ontology-manager.keycloakExternalUrl` — derive from ConfigMap issuer, fall back to explicit value
- `ontology-manager.oxigraphServiceFqdn` — lookup from `grl-platform-oxigraph` ConfigMap, fall back to explicit value

### 3c. Use helpers in templates
**`k8s/ontology-manager/templates/backend-deployment.yaml`**: lines 36, 42, 44 — use helpers instead of `.Values.*.serviceFqdn`
**`k8s/ontology-manager/templates/frontend-config-configmap.yaml`**: line 10 — use keycloakExternalUrl helper

### 3d. Relax validation
**`k8s/ontology-manager/templates/_validate.tpl`**:
- Line 5: simplify to `{{- if not .Values.gateway.host -}}`
- Lines 8-9: remove `keycloak.externalUrl` required check (helper will fail if neither value nor ConfigMap provides it)

---

## Phase 4: FHIR-Message-Mapper-Service changes

### 4a. Replace hardcoded Oxigraph FQDNs
**`k8s/fhir/values.yaml`** lines 17-18: hardcoded `oxigraph-default.grl-platform.svc.cluster.local`
**`k8s/fhir/charts/mapper/values.yaml`** lines 28-29: same

These have the same namespace coupling issue. Since FHIR-Mapper doesn't use Keycloak yet, only the Oxigraph FQDN needs fixing. Two options:
- **Minimal**: add `platformNamespace: grl-platform` and construct the FQDN in values with a comment
- **Full**: add a ConfigMap lookup helper like OM

Going with minimal since FHIR-Mapper's env vars are passed as a flat map (not templated individually), and it only needs Oxigraph. The gateway ref naming is already correct (`gateway.name`/`gateway.namespace`).

---

## Phase 5: Verification

```bash
# grl-platform — must render cleanly
helm template grl-platform k8s/grl-platform \
  --set keycloak.secrets.kcDbPassword=test \
  --set keycloak.secrets.kcAdminPassword=test \
  --set keycloak.gateway.host=keycloak.example.com \
  --set keycloak.clients.ontology-manager.hostname=ontology.example.com \
  --set keycloak.clients.graph-sandbox.hostname=sandbox.example.com

# GraphSandbox — verify gateway ref works
helm template generator GraphSandbox/k8s \
  --set global.environment=dev

# Ontology-Manager — with explicit overrides (ConfigMap lookup fails in dry-run)
helm template ontology-manager Ontology-Manager/k8s/ontology-manager \
  --set gateway.host=ontology.example.com \
  --set keycloak.externalUrl=https://keycloak.example.com/auth \
  --set keycloak.serviceFqdn=kc.grl-platform.svc.cluster.local \
  --set oxigraph.serviceFqdn=ox.grl-platform.svc.cluster.local \
  --set secrets.serviceAccountSecret=test
```

---

## Files modified (18 total)

| Repo | File | Change |
|------|------|--------|
| grl-platform | `k8s/keycloak/values.yaml` | gateway.host default, realm block |
| grl-platform | `k8s/keycloak/templates/_validate.tpl` | Fix host check, add hostname validation |
| grl-platform | `k8s/keycloak/templates/keycloak-realm-configmap.yaml` | Template lifespans + seed users |
| grl-platform | `k8s/keycloak/templates/discovery-configmap.yaml` | Add realm, serviceFqdn, servicePort |
| grl-platform | `k8s/oxigraph/templates/discovery-configmap.yaml` | **NEW** — instance FQDNs |
| GraphSandbox | `k8s/values.yaml` | platformNamespace default, flatten gateway ref |
| GraphSandbox | `k8s/templates/httproute.yaml` | gateway.name/namespace |
| GraphSandbox | `builder/k8s/chart/values.yaml` | Flatten gateway ref |
| GraphSandbox | `builder/k8s/chart/templates/httproute.yaml` | gateway.name/namespace (3 places) |
| GraphSandbox | `shared/src/helm.ts` | Flatten GatewayConfig type |
| Ontology-Manager | `k8s/ontology-manager/values.yaml` | platformNamespace, empty FQDN defaults, empty host |
| Ontology-Manager | `k8s/ontology-manager/templates/_helpers.tpl` | 3 ConfigMap lookup helpers |
| Ontology-Manager | `k8s/ontology-manager/templates/backend-deployment.yaml` | Use helpers for FQDNs |
| Ontology-Manager | `k8s/ontology-manager/templates/frontend-config-configmap.yaml` | Use helper for externalUrl |
| Ontology-Manager | `k8s/ontology-manager/templates/_validate.tpl` | Relax checks |
| FHIR-Mapper | `k8s/fhir/values.yaml` | Replace hardcoded Oxigraph FQDN |
| FHIR-Mapper | `k8s/fhir/charts/mapper/values.yaml` | Replace hardcoded Oxigraph FQDN |
