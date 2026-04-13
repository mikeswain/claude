# Data Isolation Analysis & Design Document

## Context

OM and FHIR-Mapper currently expose all projects to all authenticated users — there is no user-level or organization-level data isolation. GraphSandbox already isolates by user (filesystem paths + K8s labels keyed on JWT `sub`). The goal is configurable isolation across all three apps, controlled by a single deploy-time Helm value, so small on-prem installs can run "open" while multi-tenant deployments enforce org boundaries.

---

## Current State

### Ontology-Manager (Java/Spring, SPARQL/TDB2)
- **Identity**: JWT → `sub`, `preferred_username`, `email`, `realm_access.roles` (ADMIN/EDITOR/VIEWER)
- **Data model**: Projects have `ownerId` (`Project.java:14`), `ProjectAssignment` tracks userId + role per project
- **Gaps**:
  - `ProjectController.list()` returns **all** non-archived projects — ignores `ownerId` and assignments
  - `OntologySummaryService.listOntologySummaries()` — same, returns all
  - No `orgId` on any entity, no org claims extracted from JWT

### FHIR-Message-Mapper-Service (Java/Spring WebFlux, SPARQL/TDB2)
- **Identity**: JWT → `sub`, `preferred_username`, `realm_access.roles` → ADMIN/USER
- **Data model**: Projects stored as RDF in TDB2; no ownership fields on projects
- **`AuthSession`** has `assignedProjects` but it's **never populated** (hardcoded `List.of()`)
- **Partial filtering**: `ProjectController.filterProjectsByUserAccess()` exists but is a no-op for USER role since assignments are empty
- **Gaps**:
  - Mutation endpoints (destination, ontology, mapping controllers) — **no** per-project authorization
  - No `ownerId` or `orgId` on projects, no org claims extracted

### GraphSandbox (TypeScript/Remix, filesystem + K8s)
- **Identity**: Keycloak OIDC → `sub`, `email`, `organization[]`, `org_name`, `org_id` (`User.ts:44-55`)
- **Already user-scoped**: All paths use `cleanLabel(user.sub)` for filesystem dirs, K8s namespace labels, Helm repo paths
- **Gaps**: `org_id`/`org_name` present in token but only used cosmetically (API key prefix, UI card)

### Keycloak (shared, grl-platform)
- KC 26 with built-in organizations feature
- GraphSandbox already requests `organization` scope
- `keycloak-client` subchart handles client registration per consumer

### Service Account: `grl-platform-service`
- **Client credentials** grant, `serviceAccountsEnabled: true`, `publicClient: false`
- **Realm roles**: `admin` (via SA user `service-account-grl-platform-service`)
- **JWT `sub`**: `service-account-grl-platform-service` — not a real user
- **No KC org membership** — KC 26 doesn't support SA org membership (known issue)
- **Workaround**: hardcoded `oidc-hardcoded-claim-mapper` on the client injects an `organization` claim with a configured org alias/ID
- **Used by**:
  - OM → Keycloak Admin API (user directory queries)
  - GS → OM published ontology API (`/api/v1/published`)
  - Both apps treat SA tokens identically to user tokens — role extraction from `realm_access.roles`, no special SA handling
- **Current behavior**: SA has `admin` role, so it bypasses all role-based restrictions in both OM and FHIR

---

## Isolation Model

### Single Configuration Knob

A Helm value `isolation.mode` propagated to all apps via a Reflector-mirrored ConfigMap:

| Mode | Scope key | Who sees what | KC orgs required? |
|------|-----------|--------------|-------------------|
| `none` | n/a | All authenticated users see everything | No |
| `user` | JWT `sub` | Users see only projects they own / created | No |
| `organization` | JWT `org_id` | Users see only their org's projects; org members share visibility | Yes |

**Default: `none`** — backwards compatible, existing deploys unchanged.

### Identity Source: JWT Claims Only

No cross-service calls, no duplicated assignment stores. The JWT already carries everything needed:
- **`sub`** — user identity (available in all three apps today)
- **`org_id`** — organization identity (KC 26 organizations feature, already in GS token type)

This mirrors what GraphSandbox already does. OM and FHIR need to adopt the same pattern.

### Service Account (Machine-to-Machine) Handling

Two-tier SA model: a platform-internal SA for infrastructure operations, and per-org SAs for tenant-facing API access.

**Tier 1 — Platform SA** (`grl-platform-service`, existing)
- Used internally: OM → KC Admin API, GS → OM published ontology API
- Has `superadmin` realm role — bypasses all isolation checks
- Never exposed to tenants; credentials stay within the platform namespace
- No org claim needed — superadmin skips org filtering

**Tier 2 — Per-org SA** (`grl-platform-service-{org-alias}`, new)
- One per organization, provisioned when an org is created
- Each has a hardcoded `organization` claim mapper injecting that org's `org_id` (KC 26 workaround for SA org membership)
- Has `admin` role (scoped to their org by isolation enforcement)
- Credentials distributed to the org for their external integrations / pipelines
- Org-scoped: can only see/modify data belonging to their org, same as any org member

| Mode | Platform SA | Per-org SA | User tokens |
|------|------------|-----------|-------------|
| `none` | Sees all (superadmin) | n/a (not needed) | See all |
| `user` | Sees all (superadmin) | n/a (not needed) | See own projects |
| `organization` | Sees all (superadmin) | Sees own org only | See own org only |

**Enforcement logic**:
```java
// In TenantFilterService / TenantIsolationFilter
if (ctx.hasRole("superadmin")) {
    return repo.findByArchivedFalse(pageable); // platform SA — unrestricted
}
// ... normal isolation logic (org_id filtering applies to per-org SAs and users alike)
```

**Per-org SA provisioning**: In `organization` mode, the realm config iterates over `realm.organizations[]` in values.yaml to generate a client + SA user per org. Each client gets a hardcoded claim mapper with that org's alias/ID. Secrets are Reflector-mirrored like the existing platform SA secret.

### Adding a New Organization (operational workflow)

When `isolation.mode: organization`, adding a new org (e.g. a department):

1. **Keycloak UI/API**: Create the organization, add users as members
2. **Helm values**: Add the org to `realm.organizations[]` (name, alias). Example:
   ```yaml
   realm:
     organizations:
       - name: Graph Research Labs
         alias: graph-research-labs
       - name: Radiology Dept        # new
         alias: radiology-dept        # new
   ```
3. **`helm upgrade`**: Provisions the per-org SA client + hardcoded claim mapper + mirrored secret

No per-app changes, no data migration, no new namespaces. User tokens carry `org_id` as soon as they're added to the KC org; filtering is automatic.

---

## Phase 0: Security Hardening (needed regardless of isolation mode)

Existing authorization gaps that are bugs independent of multi-tenancy.

### 0a. OM — project list ignores ownership
- `ProjectController.list()` and `OntologySummaryService.listOntologySummaries()` return all projects
- Fix: When `isolation.mode != none`, filter by `ownerId == sub` (user mode) or `orgId == org_id` (org mode)
- OM already has `ownerId` on `Project` and `ProjectAssignment` — these just aren't consulted at list time
- Files: `ProjectController.java`, `ProjectService.java`, `SparqlProjectRepository.java`, `OntologySummaryService.java`

### 0b. FHIR — projects have no ownership
- Projects have no `ownerId` field — need to add one (set from JWT `sub` on create)
- `AuthSession.assignedProjects` concept can be **removed** — ownership via `sub` on the project itself replaces it
- Files: `ProjectService.java` (SPARQL INSERT/SELECT), `JwtAuthenticationProvider.java`, `AuthSession.java`

### 0c. FHIR — mutation endpoints unguarded
- `ProjectDestinationController`, `OntologyController`, `MappingVersionController` allow any authenticated user to modify any project
- Fix: check project ownership before mutations (same mechanism as list filtering)
- Files: all controllers under `com.grl.mapper.controller`

---

## Phase 1: Platform Configuration & Tenant Context

### 1a. Helm value + Reflector-mirrored ConfigMap

Add to `k8s/grl-platform/values.yaml`:
```yaml
isolation:
  mode: none  # none | user | organization
```

New template `templates/isolation-configmap.yaml`:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: grl-platform-isolation
  annotations:
    reflector.v1.k8s.emberstack.com/reflection-allowed: "true"
    reflector.v1.k8s.emberstack.com/reflection-auto-enabled: "true"
data:
  GRL_ISOLATION_MODE: {{ .Values.isolation.mode | quote }}
```

Consumer Deployment templates add `envFrom` referencing this ConfigMap.

### 1b. Tenant context per app

**OM (Spring MVC)**: New `TenantContext` record extracted from JWT in a servlet filter. Fields: `userId` (sub), `orgId` (nullable), `isolationMode` (from env). Stored as request attribute.

**FHIR (WebFlux)**: Extend `AuthSession` with `orgId`/`orgName`. Extract from JWT in `JwtAuthenticationProvider`. Add `ownerId` to project creation. Read isolation mode from `GRL_ISOLATION_MODE` env var / Spring config.

**GraphSandbox (Remix)**: Introduce `scopeKey()` utility that replaces direct `cleanLabel(user)` calls:
```typescript
function scopeKey(token: AccessToken, mode: IsolationMode): string {
  switch (mode) {
    case 'none': return 'shared';
    case 'user': return cleanLabel(token.sub);
    case 'organization': return cleanLabel(token.org_id ?? token.sub);
  }
}
```

### 1c. Keycloak org claims
- Verify KC 26 `organization` scope puts `org_id` in the **access token** (not just ID token / userinfo)
- GS already requests the scope; OM and FHIR keycloak-client configs need it too
- If claims don't land in the access token, add a protocol mapper in `keycloak-client` subchart

### 1d. Service account changes (organization mode only)

**Platform SA** (`grl-platform-service`):
- Add `superadmin` to its realm roles (currently has `admin`)
- No org claim needed — superadmin bypasses isolation
- No other changes; continues to be used for internal infra calls

**Per-org SAs** (new, `grl-platform-service-{org-alias}`):
- Generated in `realm-config-job.yaml` by iterating `realm.organizations[]`
- Each client: `serviceAccountsEnabled: true`, `publicClient: false`, realm role `admin`
- Each gets a hardcoded `organization` claim mapper with `{"org-alias":{"id":"org-uuid"}}`
- Secrets stored in per-org K8s Secrets, Reflector-mirrored to consumer namespaces
- In `none`/`user` modes, per-org SAs are not provisioned (gated by `isolation.mode == organization`)

---

## Phase 2: Isolation Enforcement

### 2a. OM — service-layer filtering

New method in `ProjectService` (or a `TenantFilterService`):
```java
public Page<Project> listAccessible(TenantContext ctx, Pageable pageable) {
    return switch (ctx.isolationMode()) {
        case NONE -> repo.findByArchivedFalse(pageable);
        case USER -> repo.findByArchivedFalseAndOwnerId(ctx.userId(), pageable);
        case ORGANIZATION -> repo.findByArchivedFalseAndOrgId(ctx.orgId(), pageable);
    };
}
```

- Wire into `ProjectController.list()` and `OntologySummaryService`
- Individual project GET/PUT/DELETE: verify ownership/org match before proceeding
- ADMIN role: in `user` mode sees own + assigned (via `ProjectAssignment`); in `organization` mode sees all within org

### 2b. FHIR — ownership-based filtering

Replace `filterProjectsByUserAccess()` with isolation-mode-aware logic:
- `none` → return all (current ADMIN behavior for everyone)
- `user` → filter by `ownerId == session.userId`
- `organization` → filter by `orgId == session.orgId`

Add a centralized `TenantIsolationFilter` (WebFilter after `AuthenticationFilter`) that checks project ownership on any request carrying a `projectId` parameter — covers all mutation controllers without per-endpoint changes.

### 2c. GraphSandbox — `scopeKey()` replaces `cleanLabel(user)`

All data-path functions switch from `cleanLabel(user)` to `scopeKey(token, mode)`:
- `ontology.server.ts` — `ontologyDir()`, `fetchUserOntologies()`
- `assets.server.ts` — `queryNamespaces()`, `fetchNamespace()` (K8s label selector)
- `packages.server.ts` — `queryPackages()`, `fetchHelmPackage()` (Helm repo path)

In `organization` mode, all org members share the same filesystem root and K8s label, meaning they **see and manage each other's sandboxes** — a shared workspace model.

---

## Phase 3: Data Model Changes

### 3a. Add fields

| App | Field | Storage | Set on create | Nullable |
|-----|-------|---------|---------------|----------|
| OM | `orgId` | SPARQL `grl:orgId` triple | From JWT `org_id` when mode=organization | Yes |
| FHIR | `ownerId` | SPARQL `dcterms:creator` triple | From JWT `sub` always | Yes (null for legacy) |
| FHIR | `orgId` | SPARQL `grl:orgId` triple | From JWT `org_id` when mode=organization | Yes |
| GS | (none) | Filesystem path IS the scope | n/a | n/a |

### 3b. Migration for existing data

When upgrading an existing deployment from `none` → `user` or `organization`:
- **OM**: Existing projects already have `ownerId` — user mode works immediately. For org mode, a SPARQL UPDATE backfills `orgId` on projects without one.
- **FHIR**: Existing projects have no `ownerId` — a SPARQL UPDATE stamps a default owner. For org mode, same `orgId` backfill.
- **GS**: In `none` → `user`, no migration needed (already user-scoped). In `none` → `organization`, existing user dirs need to be merged under org dirs (filesystem operation).

Deliver migrations as a one-time Helm hook Job (post-upgrade) with configurable default org/owner parameters.

---

## Phase 4: Helm Wiring Summary

| Component | Change |
|-----------|--------|
| `grl-platform/values.yaml` | Add `isolation.mode: none` |
| `grl-platform/templates/isolation-configmap.yaml` | New ConfigMap with Reflector annotations |
| Consumer Deployments | `envFrom` for `grl-platform-isolation` + Reloader annotation |
| `keycloak-client` subchart | Ensure `organization` scope in `defaultClientScopes` for all clients |
| `keycloak` realm config | Add `superadmin` realm role; assign to platform SA user |
| `keycloak` realm config | Generate per-org SA clients from `realm.organizations[]` (org mode only) |

---

## Verification Strategy

1. **Unit tests**: Tenant filter with all three modes × ADMIN/USER/SA token types
2. **Helm unit tests**: ConfigMap rendering, env var injection, SA mapper templating
3. **k3d integration**:
   - `isolation.mode: none` → all users see all projects (current behavior preserved)
   - `isolation.mode: user` → user sees only owned projects; SA sees all
   - `isolation.mode: organization` → org A user sees only org A projects; SA with superadmin sees all
4. **Negative tests**: Cross-org access denied; cross-user access denied in user mode; unauthenticated rejected
5. **SA regression**: GS → OM published ontology API still works in all modes; OM → KC admin API unaffected

---

## Key Design Decisions

1. **JWT as sole identity source** — `sub` for user scope, `org_id` for org scope. No cross-service calls, no duplicated stores. Mirrors existing GS pattern.
2. **Enforcement at service/query layer** — gateway can't understand per-entity scoping; each app filters at the data access layer.
3. **`orgId` is nullable/additive** — existing data works unchanged; `none` mode never reads or writes it.
4. **Single config knob** — all apps read the same `isolation.mode`, preventing inconsistent enforcement.
5. **Logical isolation** — all tenants share stores. For physical isolation, deploy separate Helm releases.
6. **Org = shared workspace in GS** — org members share ontologies, sandboxes, and packages.

---

## Implementation Order (recommended for separate sessions)

1. **Phase 0** — Fix security gaps in OM and FHIR (independent of isolation feature)
2. **Phase 1a** — Platform ConfigMap + Helm wiring
3. **Phase 1b+2** — Per-app tenant context + enforcement (can be done in parallel across repos)
4. **Phase 3** — Data model additions + migration tooling
5. **Phase 4** — End-to-end k3d verification
