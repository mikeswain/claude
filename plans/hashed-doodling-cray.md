# Nav Registry — `nav-entry` Subchart

## Context

Consumer apps (Ontology-Manager, GraphSandbox, FHIR-Mapper) need to discover sibling components to render a cross-app navigation bar (links + icons). Today there's no central nav registry — each consumer only knows about services it integrates with directly. The goal is a self-service, decoupled registration pattern (like `keycloak-client`) where consumers declare nav metadata and all apps receive an aggregated nav manifest via environment/file injection — no K8s API awareness in consumers.

## Approach

New `nav-entry` subchart in `k8s/nav-entry/`. Consumers add it as a Helm dependency and declare their nav metadata. On install/upgrade, a post-install Job in the platform namespace:

1. Creates/updates an individual labeled ConfigMap (`grl-platform-nav-{appId}`)
2. Aggregates all labeled nav ConfigMaps into a single `grl-platform-nav` ConfigMap
3. Reflector mirrors `grl-platform-nav` to all consumer namespaces

Consumers mount `grl-platform-nav` as `nav.json` — a static file the frontend fetches at startup.

## New Files: `k8s/nav-entry/`

### `Chart.yaml`
- `name: nav-entry`, `version: 0.1.0`, `type: application`
- Mirror structure of `k8s/keycloak-client/Chart.yaml`

### `values.yaml`
```yaml
global:
  host: ""
  platformNamespace: grl-platform

appId: ""           # Required — unique app identifier (used in ConfigMap name)
name: ""            # Display name (defaults to appId)
hostname: ""        # Public hostname (falls back to global.host)
icon: "apps"        # Material Icons name (e.g., "schema", "hub", "transform")
order: 100          # Sort order (lower = earlier)

image:
  repository: bitnami/kubectl
  tag: "1.33"
backoffLimit: 3
resources:
  requests:
    cpu: 50m
    memory: 32Mi
  limits:
    cpu: 100m
    memory: 64Mi
```

### `templates/_helpers.tpl`
- `nav-entry.name`, `nav-entry.fullname`, `nav-entry.labels` — standard helpers (copy pattern from `keycloak-client`)
- `nav-entry.platformNamespace` — `global.platformNamespace | default "grl-platform"`
- `nav-entry.hostname` — `.Values.hostname | default .Values.global.host`
- `nav-entry.url` — `https://{hostname}` (derive from hostname helper; use `https` since gateway terminates TLS)

### `templates/_validate.tpl`
- Fail if `.Values.appId` is empty

### `templates/configmap.yaml`
Individual nav entry ConfigMap. Runs as a Helm hook in the **platform namespace**.

```yaml
metadata:
  name: grl-platform-nav-{{ .Values.appId }}
  namespace: {{ platformNamespace }}
  labels:
    grl-platform/nav-entry: "true"
  annotations:
    helm.sh/hook: post-install,post-upgrade
    helm.sh/hook-delete-policy: before-hook-creation
data:
  appId: {{ .Values.appId }}
  name: {{ .Values.name | default .Values.appId }}
  url: {{ nav-entry.url }}
  icon: {{ .Values.icon }}
  order: {{ .Values.order }}
```

### `templates/rbac.yaml`
ServiceAccount, Role (get/list/create/update/patch ConfigMaps), RoleBinding — all scoped to platform namespace. All are Helm hooks with `hook-weight: "-5"` so they're created before the Job.

### `templates/job.yaml`
Post-install/upgrade Job (hook-weight: `"10"`, runs after the ConfigMap hook).

Script logic:
1. `kubectl get configmaps -l grl-platform/nav-entry=true` in platform namespace
2. Build JSON array sorted by `order`: `[{"appId":"...","name":"...","url":"...","icon":"..."}]`
3. `kubectl apply` the aggregate `grl-platform-nav` ConfigMap with Reflector annotations

### `templates/cleanup-job.yaml`
Pre-delete hook: deletes `grl-platform-nav-{appId}`, then re-runs aggregation.

### `templates/NOTES.txt`
Brief post-install message showing the registered nav entry.

### `tests/`
Helm unit tests covering:
- ConfigMap renders with correct labels and namespace
- Job runs with correct hook annotations and weight ordering
- RBAC scoped correctly
- Validation fails on empty appId
- Cleanup hook renders

## Consumer Changes

Each consumer adds the dependency and configures nav metadata.

### Ontology-Manager
- `k8s/ontology-manager/Chart.yaml`: add `nav-entry` dependency (same pattern as `keycloak-client`)
- `k8s/ontology-manager/values.yaml`: add `nav-entry:` section
  ```yaml
  nav-entry:
    appId: ontology-manager
    name: Ontology Manager
    icon: schema
    order: 10
  ```

### GraphSandbox
- `k8s/values.yaml`: add `nav-entry` dependency + values
  ```yaml
  nav-entry:
    appId: graph-sandbox
    name: Graph Sandbox
    icon: hub
    order: 20
  ```

### FHIR-Message-Mapper-Service
- `k8s/fhir/Chart.yaml`: add `nav-entry` dependency + values
  ```yaml
  nav-entry:
    appId: fhir-mapper
    name: FHIR Mapper
    icon: transform
    order: 30
  ```

## Aggregate ConfigMap Shape

`grl-platform-nav` (Reflector-mirrored to all namespaces):

```yaml
data:
  nav.json: |
    [
      {"appId":"ontology-manager","name":"Ontology Manager","url":"https://om.staging.test","icon":"schema"},
      {"appId":"graph-sandbox","name":"Graph Sandbox","url":"https://gs.staging.test","icon":"hub"},
      {"appId":"fhir-mapper","name":"FHIR Mapper","url":"https://fhir.staging.test","icon":"transform"}
    ]
```

## Consumer Frontend Consumption

Each consumer's Deployment mounts the aggregate:
```yaml
volumes:
  - name: nav
    configMap:
      name: grl-platform-nav
      optional: true          # graceful degradation if no nav entries exist yet
volumeMounts:
  - name: nav
    mountPath: /usr/share/nginx/html/nav.json
    subPath: nav.json
    readOnly: true
```

Frontend fetches `/nav.json` at startup, renders icon links. Missing file = no nav (graceful).

## CI

No CI changes needed — `pages.yml` iterates `k8s/*/Chart.yaml` and will automatically discover, test, package, and publish `nav-entry`.

## Key Reference Files
- `k8s/keycloak-client/` — primary pattern to follow for subchart structure
- `k8s/keycloak-client/templates/job.yaml` — hook Job pattern
- `k8s/keycloak-client/templates/_helpers.tpl` — helpers pattern
- `k8s/keycloak-client/values.yaml` — values structure
- `k8s/oxigraph/templates/discovery-configmap.yaml` — Reflector annotation pattern

## Verification

1. `helm unittest k8s/nav-entry` — all unit tests pass
2. `helm lint k8s/nav-entry` — no warnings
3. `helm template` the nav-entry subchart with test values — verify ConfigMap, Job, RBAC render correctly
4. Verify hook ordering: RBAC (weight -5) → ConfigMap (weight 0) → Job (weight 10)
5. Verify the aggregation script builds valid JSON from multiple labeled ConfigMaps

## Implementation Order

1. Create `k8s/nav-entry/` subchart (Chart.yaml, values.yaml, _helpers.tpl, _validate.tpl)
2. Add templates (configmap.yaml, job.yaml, rbac.yaml, cleanup-job.yaml, NOTES.txt)
3. Write helm-unittest tests
4. Add `nav-entry` dependency to each consumer chart (OM, GS, FHIR) — just Chart.yaml + values.yaml changes
5. (Future, separate PR) Add `nav.json` volume mount to consumer Deployments + frontend nav component
