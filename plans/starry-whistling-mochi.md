# Add Oxigraph Graph Store Options

## Context

The per-sandbox Helm chart currently supports two graph store backends: GraphDB and Fluree. Each deploys a dedicated pod. We need two additional options:

- **Standalone Oxigraph** — deploys an Oxigraph pod per sandbox (like graphdb/fluree)
- **Internal Store** — uses the shared grl-platform Oxigraph instance (no pod deployed, just points the graph-server at the platform service). Data isolation is handled by graph-server's existing JWT-scoped named graphs.

## Approach

Follow existing patterns exactly: standalone oxigraph mirrors graphdb.yaml structure; internal store uses the keycloak-style ConfigMap discovery at deploy time.

## Files to Change

### 1. `shared/src/helm.ts` — Add types

- Add `PlatformServiceConfig` type: `{ enabled: boolean; serviceFqdn?: string; port?: number }`
- Add to `SandboxHelmValues`:
  - `oxigraph: ContainerConfig`
  - `platformOxigraph: PlatformServiceConfig`

### 2. `builder/k8s/chart/values.yaml` — Default values

Add after graphdb block:

```yaml
oxigraph:
  enabled: false
  image: ghcr.io/oxigraph/oxigraph
  tag: "latest"
  port: 7878
  podLabels:
    app.kubernetes.io/component: oxigraph
  podAnnotations: {}
  livenessProbe: {}
  readinessProbe: {}

platformOxigraph:
  enabled: false
  serviceFqdn: ""    # Resolved from grl-platform-oxigraph ConfigMap at deploy time
  port: 7878
```

### 3. NEW: `builder/k8s/chart/templates/oxigraph.yaml` — Standalone Deployment + Service

- Guard: `{{ if .Values.oxigraph.enabled }}`
- Container: `oxigraph serve --location /data --bind 0.0.0.0:7878`
- Volume: reuse existing `graph-pvc` mounted at `/data`
- Service: ClusterIP, selector `app.kubernetes.io/component: oxigraph`
- Pattern: simplified version of graphdb.yaml (no init container, no configmap)

### 4. `builder/k8s/chart/templates/graph-server.yaml` — Extend env var chain

Add two `else if` branches after the fluree block (around line 90):

```yaml
{{ else if .Values.oxigraph.enabled }}
- name: APPLICATION_REPOSITORYTYPE
  value: sparql
- name: APPLICATION_QUERY_ENDPOINT
  value: "http://{{ include "sandbox.fullname" . }}-oxigraph:{{ .Values.oxigraph.port }}/query"
- name: APPLICATION_UPDATE_ENDPOINT
  value: "http://{{ include "sandbox.fullname" . }}-oxigraph:{{ .Values.oxigraph.port }}/update"
{{ else if .Values.platformOxigraph.enabled }}
- name: APPLICATION_REPOSITORYTYPE
  value: sparql
- name: APPLICATION_QUERY_ENDPOINT
  value: "http://{{ .Values.platformOxigraph.serviceFqdn }}:{{ .Values.platformOxigraph.port }}/query"
- name: APPLICATION_UPDATE_ENDPOINT
  value: "http://{{ .Values.platformOxigraph.serviceFqdn }}:{{ .Values.platformOxigraph.port }}/update"
{{ end }}
```

### 5. `builder/k8s/chart/templates/httproute.yaml` — Standalone oxigraph route

Add after Fluree HTTPRoute block (before final `{{- end }}`):

```yaml
{{- if .Values.oxigraph.enabled }}
# Oxigraph HTTPRoute — oxigraph-<namespace>.<domain>
{{- end }}
```

No HTTPRoute for internal store (accessed within cluster only).

### 6. `builder/src/deployToK8s.ts` — Resolve platform oxigraph ConfigMap

- Add `resolveOxigraphOverrides(): Promise<string[]>` following the keycloak pattern
  - Read `grl-platform-oxigraph` ConfigMap from `platformNamespace`
  - Return `["--set", "platformOxigraph.serviceFqdn=X", "--set", "platformOxigraph.port=Y"]`
  - Return `[]` if not found (harmless when platformOxigraph not enabled)
- Call from `deployTok8s()` alongside OIDC overrides
- Merge both override sets into the `applyHelmChart` call

### 7. `frontend/server/serverConfig.ts` — Expose platform availability

Add to config object:
```ts
platformOxigraphAvailable: !!process.env.GRAPH_HOST,
```

`GRAPH_HOST` is set by the sandbox-generator Helm chart when it resolves the `grl-platform-oxigraph` ConfigMap. If empty/unset, the platform store isn't available.

### 8. `frontend/app/routes/generate/step2.tsx` — UI

Pass `platformOxigraphAvailable` from loader to component (already imports `serverConfig`).

Update Graph Database dropdown — "Internal Store" only shown when available:
```tsx
<option value="fluree">Fluree</option>
<option value="graphdb">GraphDB</option>
<option value="oxigraph">Oxigraph</option>
{platformOxigraphAvailable && <option value="platformOxigraph">Internal Store</option>}
```

Extend action form-to-overrides mapping:
```ts
oxigraph: { enabled: rawForm.repositoryType === "oxigraph" },
platformOxigraph: { enabled: rawForm.repositoryType === "platformOxigraph" },
```

### 9. `builder/k8s/chart/templates/storage.yaml` — Conditional graph-pvc

Guard the graph-pvc so it's not created for internal store (no local storage needed):
```yaml
{{- if or .Values.graphdb.enabled .Values.fluree.enabled .Values.oxigraph.enabled }}
# graph-pvc
{{- end }}
```

## Implementation Order

1. `shared/src/helm.ts` (types — dependencies first)
2. `builder/k8s/chart/values.yaml`
3. `builder/k8s/chart/templates/oxigraph.yaml` (new file)
4. `builder/k8s/chart/templates/graph-server.yaml`
5. `builder/k8s/chart/templates/httproute.yaml`
6. `builder/k8s/chart/templates/storage.yaml`
7. `builder/src/deployToK8s.ts`
8. `frontend/server/serverConfig.ts`
9. `frontend/app/routes/generate/step2.tsx`

## Verification

1. `npm run build -w shared && npm run build -w builder` — types compile
2. `npm test -w builder` — existing tests pass
3. Deploy a sandbox with each store option and verify:
   - graph-server gets correct env vars (`kubectl exec` or pod logs)
   - standalone oxigraph pod starts and is reachable
   - internal store correctly resolves FQDN from ConfigMap
4. Check `helm template` output for each enabled store to verify correct YAML generation
