# Separate Display Name from Machine Name for APIs/Ontologies

## Context

API and ontology names flow through many GRL components but serve two purposes:
1. **Display name** — human-readable, free-format (e.g. "My Pet API v2", "Health Data Model")
2. **Machine name** — used in config.yaml, K8s labels, URL paths, filesystem paths, IRIs

Currently these are conflated. A single name is either silently sanitised (losing user intent)
or passes through unchecked and breaks downstream in ASG, grl-frontend-web-ui, or K8s.

The fix: separate them explicitly. Display name is free-format. Machine name is derived via
`cleanLabel()` and always safe for all consumers.

## Current Data Flow

```
Ontology uploaded or imported from OM
  → ontology "name" (free-format today, used as directory name)
    → annotation editor adds APIs with:
        - apiName (IRI local name, e.g. "MyApi" — derived via toApiName() from label)
        - prefLabel (display label, e.g. "My Pet API")
        - displayLabel (query result fallback)
    → step3.tsx uses ontologyName as key in apiNames Record
    → generate.ts uses apiNames keys in filesystem paths
    → config.yaml "name" field → ASG (requires ^[a-zA-Z0-9_]+$)
    → K8s labels via cleanLabel()
    → URL paths in grl-frontend-web-ui (requires ^\w+$)
```

## Revised Model

### Two names at each level:

| Level | Display name | Machine name | Source |
|-------|-------------|-------------|--------|
| **Ontology** | `displayName` (free-format) | `name` (sanitised via `cleanLabel()`) | Display: filename or OM project name. Machine: `cleanLabel(displayName)` |
| **API** | `prefLabel` / `displayLabel` (free-format) | `name` (IRI short form from `toApiName()`) | Already exists in `ApiInfo` |

### Key insight: `ApiInfo` already has this separation

```typescript
type ApiInfo = { id: string; name: string; displayLabel: string; };
```

- `name` = IRI short form (e.g. `MyApi`) — always PascalCase from `toApiName()`, safe for machine use
- `displayLabel` = human-readable (e.g. "My Pet API") — free-format

The annotation editor's `toApiName()` produces PascalCase output that satisfies all downstream
consumers (`^\w+$`). This is the machine name. The `prefLabel` entered by the user is the
display name.

### Config.yaml `name` field across phases

The `name` in config.yaml (set to the ontology name) is used in all three phases:
- **Prepare** (uploadOntology.ts:80): `resolve(prepareDir, asgConfig.name, "ontology-merged.owl")`
- **Transform** (transformOntology.ts:38): `resolve(outputDir, asgConfig.name, "openapi.yaml")`
- **Generate** (generate.ts:68): `${path}/target/${api}/openapi.yaml` — `api` from `apiNames` key

The prepare and transform phases use `asgConfig.name` from config.yaml. The generate
phase uses the `apiNames` key from step3.tsx. Currently both are the ontology name, so
they match. But the prepare phase only needs a legal name — it ignores the actual value
for output discovery. The transform phase uses it to find ASG's output directory.

For generate, the `apiNames` key just needs to match what ASG wrote. Since `config.yaml`
controls what ASG writes, the key must match `asgConfig.name`. Whether that's the ontology
name or the API name doesn't matter, as long as it's consistent.

### What needs to change:

The **ontology** level doesn't have this separation yet. `ontology.name` is used as both
the display name and the directory/label name.

## Machine Name Pattern

Used for: config.yaml `name`, K8s label values, URL paths, filesystem directories, IRIs.

```
^[a-zA-Z][a-zA-Z0-9_]{0,61}[a-zA-Z0-9]$
```

Safe across all consumers:

| Consumer | Current constraint | Satisfies? |
|----------|-------------------|-----------|
| ASG YamlConfig.java:63 | `^[a-zA-Z0-9_]+$` | Yes |
| grl-frontend-web-ui schema.ts:101 | `^\w+$` | Yes |
| grl-frontend-web-ui SchemaContext.tsx:24 | `^\/(\w+)` in URL | Yes |
| rest2graph SchemaUtils.java:72 | strips non-`\w` | No stripping needed |
| K8s label values | alnum start/end, `[a-zA-Z0-9._-]` middle, max 63 | Yes |
| Filesystem paths | No special chars | Yes |

## Changes

### 1. GraphSandbox shared — add machine name validator

**File:** `GraphSandbox/shared/src/utils.ts`

```typescript
/** Machine name pattern — safe for ASG, K8s labels, URL paths, filesystem, IRIs. */
export const MACHINE_NAME_PATTERN = /^[a-zA-Z][a-zA-Z0-9_]{0,61}[a-zA-Z0-9]$/;
export function isValidMachineName(name: string): boolean {
  return MACHINE_NAME_PATTERN.test(name);
}
```

Add unit tests.

### 2. GraphSandbox shared — fix `cleanLabel()` trailing underscore bug

Current `cleanLabel()` can produce labels ending in `_` (K8s rejects this). Fix:

```typescript
export function cleanLabel(label: string): string {
  return label
    .replaceAll('@', '_at_')
    .replaceAll(/^[^a-zA-Z0-9]+/g, '')     // strip leading non-alnum
    .replaceAll(/[^a-zA-Z0-9._-]/g, '_')   // replace invalid middle chars
    .slice(0, 63)
    .replaceAll(/[^a-zA-Z0-9]+$/g, '');     // strip trailing non-alnum (after truncation)
}
```

### 3. GraphSandbox ontology — separate displayName from name

**File:** `GraphSandbox/frontend/server/ontology.server.ts`

The `OntologyDescription` (or equivalent type) needs both:
- `displayName` — original filename / OM project name (shown in UI)
- `name` — `cleanLabel(displayName)` (used for directories, config.yaml, labels)

**File:** `GraphSandbox/frontend/app/routes/ontology/ontology.tsx`

Replace `sanitiseOntologyName()` with `cleanLabel()` to derive `name` from the display name.
Store both. Show `displayName` in the UI, use `name` for filesystem/config operations.

When importing from OM: `displayName` = OM project name, `name` = `cleanLabel(projectName)`.
When uploading file: `displayName` = filename, `name` = `cleanLabel(filename)`.

### 4. GraphSandbox annotation editor — default API name from ontology name

**File:** `GraphSandbox/frontend/app/routes/ontology/annotateTab.tsx`

When adding a new API, default the API machine name (`newApiName`) to the ontology's
machine `name`. The user can override it, but the default is sensible for the common case
(one API per ontology).

`toApiName()` already produces safe names. Add `isValidMachineName()` as a safety net
in `canSubmitAddApi()`.

### 5. GraphSandbox generate step — use API machine name, not ontology name

**File:** `GraphSandbox/frontend/app/routes/generate/step3.tsx`

Currently line 43: `const apiName = rawForm.ontologyName.toString()` — uses the ontology
name as the `apiNames` key.

This should use the API's machine `name` (from `ApiInfo.name`) instead, since that's what
ASG and the generate pipeline expect. The ontology may contain multiple APIs; the user
selects which one to generate (step3 UI already shows API packages).

### 6. GraphSandbox builder — validate machine names server-side

**File:** `GraphSandbox/builder/src/generate.ts`

At the start of `generatePackage()`, validate all `apiNames` keys against
`MACHINE_NAME_PATTERN`. Throw with a clear message before touching the filesystem.

### 7. Ontology-Manager — keep project name free-format

No pattern restriction on OM project names. They're display names. When GraphSandbox
imports from OM, it derives the machine name via `cleanLabel()`. OM doesn't need to know
about machine name constraints.

### 8. FHIR Mapper — no change needed

FHIR Mapper project IDs are internal identifiers (UUIDs). When ontology names flow to
GraphSandbox, the same `cleanLabel()` derivation applies at the GraphSandbox boundary.

## Implementation Order

1. `shared/src/utils.ts` — add `isValidMachineName()` + fix `cleanLabel()` + tests
2. Ontology data model — add `displayName` alongside `name`, update ontology.server.ts
3. UI — show `displayName`, use `name` for operations
4. annotateTab — default new API name from ontology machine name, validate in `canSubmitAddApi()`
5. step3.tsx — use API machine name as `apiNames` key
6. generate.ts — server-side validation gate

## Verification

- `npm test` in GraphSandbox (shared utils tests, annotate.logic tests, generate tests)
- Manual: upload ontology with hyphens in filename → displayName preserves hyphens, name is sanitised
- Manual: import OM project "My Data Model" → displayName="My Data Model", name="My_Data_Model"
- Manual: add API in annotation editor → name defaults to ontology machine name
- Manual: generate → config.yaml `name` field is the machine name, ASG accepts it
- Existing data: `cleanLabel()` is already applied in most paths, so existing sandboxes
  should continue working. Audit for edge cases where raw names leaked through.
