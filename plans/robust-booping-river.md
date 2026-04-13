# Plan: OM Ontology Selection in FHIR and GraphSandbox

## Context

Users of FHIR-Message-Mapper-Service and GraphSandbox need to select ontologies managed by Ontology Manager for processing. FHIR already has manual OM integration (user enters endpoint URL + token), but it requires manual configuration. GraphSandbox has no OM integration at all.

**Goal:** Auto-discover OM via a Reflector-mirrored ConfigMap so both consumers can browse and select published ontologies without manual endpoint entry. Scope is **published ontologies only, read-only, public ones need no auth**.

---

## Phase 1: OM Discovery ConfigMap (Ontology-Manager repo)

The OM chart at `~/grl/Ontology-Manager/k8s/ontology-manager/` already has templates, helpers, and values. Add a discovery ConfigMap following the pattern from `grl-platform/k8s/keycloak/templates/discovery-configmap.yaml`.

### 1a. Create discovery ConfigMap template

**New file:** `~/grl/Ontology-Manager/k8s/ontology-manager/templates/discovery-configmap.yaml`

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: grl-platform-ontology-manager
  labels:
    {{- include "ontology-manager.labels" . | nindent 4 }}
  annotations:
    reflector.v1.k8s.emberstack.com/reflection-allowed: "true"
    reflector.v1.k8s.emberstack.com/reflection-auto-enabled: "true"
data:
  ONTOLOGY_MANAGER_URL: {{ printf "https://%s" (include "ontology-manager.host" .) | quote }}
  ONTOLOGY_MANAGER_INTERNAL_URL: {{ printf "http://%s-backend.%s.svc.cluster.local:8080" (include "ontology-manager.fullname" .) .Release.Namespace | quote }}
```

- `ONTOLOGY_MANAGER_URL` — browser-facing URL (via gateway), for frontend display/redirects
- `ONTOLOGY_MANAGER_INTERNAL_URL` — cluster-internal FQDN, for server-to-server API calls
- Keys are ALL_CAPS env-var-shaped per established convention

### 1b. Add Helm unit tests

**New file:** `~/grl/Ontology-Manager/k8s/ontology-manager/tests/discovery_configmap_test.yaml`

Test: correct ConfigMap name, env-var keys, Reflector annotations, URL construction from `gateway.host`.

### Verification
```bash
helm unittest ~/grl/Ontology-Manager/k8s/ontology-manager
```

---

## Phase 2: FHIR-Message-Mapper-Service — Auto-discover OM

FHIR already has full OM integration (`ExternalOntologyController`, `LoadOntologyDialog`). Changes are minimal: auto-populate the endpoint URL from the env var so the user doesn't have to type it.

### 2a. Backend: Expose OM URL to SPA via config endpoint

The designer-ui SPA can't read env vars — it gets config from the mapper backend. Add a public endpoint.

**New file:** `~/grl/FHIR-Message-Mapper-Service/src/main/java/com/grl/mapper/controller/AppConfigController.java`

```java
@RestController
@RequestMapping("/config")
public class AppConfigController {
    @Value("${ONTOLOGY_MANAGER_URL:}")
    private String ontologyManagerUrl;

    @GetMapping(value = "/integrations", produces = MediaType.APPLICATION_JSON_VALUE)
    public Mono<Map<String, Object>> getIntegrations() {
        // Return non-blank values only
    }
}
```

**Modify:** `AuthenticationFilter.java` (line 64) — add `/config/integrations` to `PUBLIC_PATHS`

### 2b. Backend: Default endpoint in ExternalOntologyController

**Modify:** `~/grl/FHIR-Message-Mapper-Service/src/main/java/com/grl/mapper/controller/ExternalOntologyController.java`

- Add `@Value("${ONTOLOGY_MANAGER_INTERNAL_URL:}") private String defaultEndpoint;`
- In `listProjects`/`fetchOntology`/`importOntology`: if `request.endpoint()` is blank and `defaultEndpoint` is set, use `defaultEndpoint`
- Skip SSRF validation for the default endpoint (trusted cluster-internal URL)
- Make token optional (public ontologies don't need auth)
- Switch `listProjects` to call `/api/v1/published` instead of `/api/v1/projects` when using the default endpoint (public discovery endpoint, no auth needed)

### 2c. Frontend: Auto-populate endpoint from backend config

**Modify:** `~/grl/FHIR-Message-Mapper-Service/designer-ui/src/api/client.ts`
- Add `getIntegrations()` method

**Modify:** `~/grl/FHIR-Message-Mapper-Service/designer-ui/src/components/ontology/LoadOntologyDialog.tsx`
- On open, call `getIntegrations()` to check for discovered OM URL
- If `ontologyManagerUrl` is returned: skip the connection form, auto-connect and show project list
- If not: show existing manual entry form (unchanged fallback)
- Make token field optional (empty string allowed for public ontologies)
- Pass empty endpoint to backend when using auto-discovered URL (backend uses its own default)

### 2d. K8s: Mount ConfigMap in mapper deployment

**Modify:** `~/grl/FHIR-Message-Mapper-Service/k8s/fhir/charts/mapper/templates/deployment.yaml` (after line 46)

Add to `envFrom`:
```yaml
            - configMapRef:
                name: grl-platform-ontology-manager
                optional: true
```

### 2e. Tests

- Unit test for `AppConfigController` (returns URL when set, empty when not)
- Unit test for `ExternalOntologyController` default endpoint fallback
- Helm test for optional configMapRef in mapper deployment

### Verification
```bash
# Backend tests
cd ~/grl/FHIR-Message-Mapper-Service && ./gradlew test
# Helm tests
helm unittest k8s/fhir
# Manual: start with ONTOLOGY_MANAGER_URL=http://localhost:8080 env var, verify LoadOntologyDialog auto-connects
```

---

## Phase 3: GraphSandbox — OM Ontology Browsing

GraphSandbox has no OM integration. The `OntologyEndpoints` abstraction is source-agnostic — we need to add a browsing UI and an import flow for OM published ontologies.

### 3a. Server config: expose OM availability

**Modify:** `~/grl/GraphSandbox/frontend/server/serverConfig.ts`

Add:
```typescript
ontologyManagerUrl: process.env.ONTOLOGY_MANAGER_INTERNAL_URL || "",
ontologyManagerAvailable: !!process.env.ONTOLOGY_MANAGER_INTERNAL_URL,
```

Follows existing `platformOxigraphAvailable` pattern.

### 3b. Server module: OM API client

**New file:** `~/grl/GraphSandbox/frontend/server/ontologyManager.server.ts`

Thin server-side module:
- `listPublishedOntologies()` — calls `GET {internalUrl}/api/v1/published`, returns list
- `fetchPublishedOntologyTtl(projectId)` — calls `GET {internalUrl}/api/v1/projects/{id}/published/ontology` with `Accept: text/turtle`
- Returns `[]` when URL not configured (graceful degradation)

Uses internal URL for server-to-server calls. No auth token (public ontologies only).

### 3c. Extend OntologyDescription with source metadata

**Modify:** `~/grl/GraphSandbox/shared/src/types.ts`

Add optional `ontologySource` field to `OntologyDescriptionPrepared`:

```typescript
export type OntologySource =
  | { kind: "uploaded" }
  | { kind: "om-project"; projectId: string; projectName: string; version: number };
```

Existing ontologies without the field are implicitly "uploaded" — backwards compatible.

### 3d. Ontology list: show OM published ontologies

**Modify:** `~/grl/GraphSandbox/frontend/app/routes/ontology/ontologies.tsx`

- Loader: also call `listPublishedOntologies()` (conditional on `ontologyManagerAvailable`)
- UI: add a second section "From Ontology Manager" below the existing table showing published ontologies with name, description, version, and "Import" button
- Import button submits a form POST to a new action/route

### 3e. Import route: fetch OM ontology and upload to Oxigraph

**New file:** `~/grl/GraphSandbox/frontend/app/routes/ontology/import-om.tsx`

Action-only route that:
1. Receives OM project ID + name via form POST
2. Server-side: calls `fetchPublishedOntologyTtl(projectId)`
3. Submits `UploadOntology` job via existing BullMQ queue (reuse existing pattern from `ontology.tsx`)
4. Saves ontology description with `phase: "Prepared"` and `ontologySource: { kind: "om-project", ... }`
5. Redirects to the ontology page

Uses existing `UploadOntologyRequest` + BullMQ job — consistent with current upload flow. The builder service already has `buildEndpoints()` and the Oxigraph upload logic.

### 3f. K8s: Add env vars to UI container

**Modify:** `~/grl/GraphSandbox/k8s/templates/deployment.yaml`

Add to the `ui` container's `env` section:
```yaml
            - name: ONTOLOGY_MANAGER_INTERNAL_URL
              valueFrom:
                configMapKeyRef:
                  name: grl-platform-ontology-manager
                  key: ONTOLOGY_MANAGER_INTERNAL_URL
                  optional: true
```

Using `configMapKeyRef` with `optional: true` (since GS uses individual `env` entries, not `envFrom`).

### 3g. Tests

- Unit test for `ontologyManager.server.ts` (mock fetch, verify URL construction, empty array when unconfigured)
- Test for import route action
- Helm test for optional configMapKeyRef

### Verification
```bash
# Frontend tests
cd ~/grl/GraphSandbox && npm test
# Helm lint
helm lint k8s/
# Manual: start with ONTOLOGY_MANAGER_INTERNAL_URL=http://localhost:8080, verify OM ontologies appear in list
```

---

## Implementation Order

All three phases can be developed in parallel (the `optional: true` ConfigMap mount means consumers work without OM). Deploy order:

1. **Phase 1** (OM ConfigMap) — deploy with next OM release
2. **Phase 2** (FHIR) and **Phase 3** (GraphSandbox) — deploy independently, they gracefully degrade when ConfigMap absent

---

## Key Files

| File | Action |
|------|--------|
| `~/grl/Ontology-Manager/k8s/ontology-manager/templates/discovery-configmap.yaml` | Create |
| `~/grl/Ontology-Manager/k8s/ontology-manager/tests/discovery_configmap_test.yaml` | Create |
| `~/grl/Ontology-Manager/k8s/ontology-manager/templates/_helpers.tpl` | Existing (has `ontology-manager.host`, `ontology-manager.fullname`) |
| `~/grl/FHIR-Message-Mapper-Service/src/.../controller/AppConfigController.java` | Create |
| `~/grl/FHIR-Message-Mapper-Service/src/.../auth/AuthenticationFilter.java` | Modify (add public path) |
| `~/grl/FHIR-Message-Mapper-Service/src/.../controller/ExternalOntologyController.java` | Modify (default endpoint) |
| `~/grl/FHIR-Message-Mapper-Service/designer-ui/src/api/client.ts` | Modify (add getIntegrations) |
| `~/grl/FHIR-Message-Mapper-Service/designer-ui/src/components/ontology/LoadOntologyDialog.tsx` | Modify (auto-connect) |
| `~/grl/FHIR-Message-Mapper-Service/k8s/fhir/charts/mapper/templates/deployment.yaml` | Modify (envFrom) |
| `~/grl/GraphSandbox/frontend/server/serverConfig.ts` | Modify (add OM URL) |
| `~/grl/GraphSandbox/frontend/server/ontologyManager.server.ts` | Create |
| `~/grl/GraphSandbox/shared/src/types.ts` | Modify (OntologySource) |
| `~/grl/GraphSandbox/frontend/app/routes/ontology/ontologies.tsx` | Modify (OM browse UI) |
| `~/grl/GraphSandbox/frontend/app/routes/ontology/import-om.tsx` | Create |
| `~/grl/GraphSandbox/k8s/templates/deployment.yaml` | Modify (env var) |
