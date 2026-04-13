# Import Samples Feature

## Context

Global/sample ontologies are currently read-only and not very useful — users can see them in the list but can't configure, annotate, or re-transform them. The `_global_` sentinel value also causes ongoing `cleanLabel()` problems. We want to replace this with a user-driven import flow where samples become fully editable user ontologies.

**Constraint**: The grl-apis source directory structure (submodule from external ontology library) cannot change.

## Approach

Add a **"Samples" tab** on the ontology creation page (`ontology.tsx`) alongside "Upload" and "Ontology Manager". Users browse available sample APIs, select one, and import it. The imported sample becomes a normal user ontology (configurable, annotatable, transformable).

The key insight: each sample's `target/<name>/ontology-merged.owl` is **fully self-contained** (all `owl:imports` inlined, zero external dependencies). This is the portable artifact we copy into the user's workspace. No catalog.xml, no extends chain, no relative import paths needed.

---

## 1. Changes to grl-apis build (`../grl-apis`)

**No source directory changes needed.** Only enrich the zip packaging.

### Add `manifest.json` to repo root

Static metadata index (changes rarely — only when samples added/removed). Contains everything needed to create a normal user config on import:

```json
[
  {
    "name": "nzcs1",
    "title": "NZ Climate Disclosure Standard 1 API",
    "description": "API to support Climate Disclosures under NZ Climate Disclosure Standard 1",
    "rootClass": "https://www.graphresearchlabs.com/SustainabilityFinancialDisclosure/NZCS1APIV1.0.0"
  },
  { "name": "admin", "title": "Administration API", "description": "...", "rootClass": "..." },
  { "name": "logging", "title": "Logging API", "description": "...", "rootClass": "..." },
  { "name": "employment", "title": "Employment", "description": "Minimal Test API" },
  { "name": "insuranceCDR", "title": "Insurance Common Data Record API", "description": "..." },
  { "name": "sensor", "title": "Sensor API", "description": "...", "rootClass": "..." }
]
```

### Update `.github/workflows/build.yml` packaging

Add to each sample directory in the zip:
- `ontology-merged.owl` — from `target/<name>/` (self-contained, ~1-2MB each)
- `manifest.json` — copied to zip root

No need for source-config.yaml — the manifest has the metadata (title, description, rootClass) and the imported ontology just gets a normal `config.yaml` from `defaultConfig.json`.

Resulting zip:
```
manifest.json
nzcs1/
  openapi.yaml, openapi.json, ontology-merged.owl
admin/
  openapi.yaml, openapi.json, ontology-merged.owl
...
```

---

## 2. Init container: `_samples_/` staging area

**File: `k8s/templates/init-scripts-configmap.yaml`**

- Change destination from `_global_/ontologies/` to `_samples_/`
- Extract flat (no `target/<name>` nesting — files land directly in `_samples_/<name>/`)
- Clean up legacy `_global_/` on upgrade

Result on PVC:
```
/mnt/data/data/sandboxes/_samples_/
  manifest.json
  nzcs1/
    openapi.yaml, openapi.json, ontology-merged.owl, source-config.yaml
  ...
```

**File: `frontend/server/serverConfig.ts`** — add:
```typescript
samplesRoot: path.resolve(buildRoot, "data/sandboxes/_samples_"),
```

---

## 3. New server module: `frontend/server/samples.server.ts`

### Types
```typescript
export type SampleInfo = {
  name: string;
  title: string;
  description: string;
  rootClass?: string;
};
```

### `listAvailableSamples(): Promise<SampleInfo[]>`
- Read and cache `manifest.json` from `samplesRoot`
- Gracefully return `[]` if manifest missing (old zip format)

### `importSample(user: string, sampleName: string): Promise<UploadJob>`

Users can import the same sample multiple times (e.g. to experiment with different annotations). If the name already exists, auto-suffix like the OM import does (e.g. `nzcs1_v2`).

1. **Resolve unique name**: Check existing user ontologies, suffix if collision
2. **Copy `ontology-merged.owl`** from `{samplesRoot}/{sampleName}/` to `{userOntologyDir}/{name}/`
3. **Create normal `config.yaml`** from `defaultConfig.json` + manifest metadata:
   - `name`: the resolved unique name
   - `ontologies: ["ontology-merged.owl"]` (local file, imports already inlined)
   - `outputDir: "target"`
   - `rootClass`: from manifest (if present)
   - `templates.openapi.info`: title + description from manifest
   - `templates.openapi.servers`: `[{ url: name }]`
   - No `extends`, no `catalogFile` — not needed with merged OWL
4. **Create `ontology.yaml` manifest**: `{ name, displayName: title, phase: "Loaded", scope: "User", ontologyFile: "ontology-merged.owl" }`
5. **Queue Upload job** (same as OM import pattern):
   ```typescript
   await addBuildJob({ userName: user, request: "Upload Ontology", ontologyName: name })
   ```
   Upload runs ASG `--no-generate` on the merged OWL. Since imports are already inlined, this is fast — it just uploads to the SPARQL store and returns `ontologyEndpoints`. Phase becomes "Prepared".

---

## 4. Frontend: Samples tab

**File: `frontend/app/routes/ontology/ontology.tsx`**

### Loader
Add `samples` to loader data:
```typescript
const samples = await listAvailableSamples(user);
return { existingOntology, publishedOntologies, omAvailable, samples };
```

### Tab visibility
```typescript
const showTabs = (omAvailable || samples.length > 0) && !existingOntology;
```

### New TabItem
Add `<TabItem title="Samples">` with a `SamplePicker` component following the existing `OmPicker` pattern:
- Card-based list (scrollable, max-height), following the `OmPicker` pattern
- Each card: title, description
- Select one → click "Import" → triggers action
- Can import same sample multiple times (auto-suffixed name for duplicates)

### Action handler
New `"import-sample"` case in the existing `action()` function:
```typescript
if (action === "import-sample") {
  const sampleName = body.sampleName;
  const job = await importSample(user, sampleName);
  return { resultType: "Upload", job };
}
```

This returns an `UploadJob` — the existing `JobTracker` modal handles progress display. On success, `onSucceeded` calls `save-upload` action which redirects to `/ontology/{name}/spec`.

---

## 5. Global ontology system — natural phase-out

**This change**: Init container writes to `_samples_/` not `_global_/ontologies/`. The existing `initializeGlobalData()` exits early when `_global_/ontologies/` doesn't exist (line 85 `existsSync` check). Global ontologies simply stop appearing.

**Phase 2 cleanup** (follow-up, not this PR):
- Remove `globalUser`, `initializeGlobalData()`, `scope: "Global"` type branch
- Remove `_global_` bypass in `ontologyDir()`
- Remove read-only badges in UI

---

## Files to modify

| File | Change |
|------|--------|
| `../grl-apis/manifest.json` | **New** — sample metadata index |
| `../grl-apis/.github/workflows/build.yml` | Include merged OWL, source config, manifest in zip |
| `k8s/templates/init-scripts-configmap.yaml` | Write to `_samples_/`, clean up `_global_/` |
| `frontend/server/serverConfig.ts` | Add `samplesRoot` |
| `frontend/server/samples.server.ts` | **New** — `listAvailableSamples`, `importSample` |
| `frontend/app/routes/ontology/ontology.tsx` | Add Samples tab + loader + action |
| `frontend/server/defaultConfig.json` | Reference — used as base for adapted config |

## Verification

1. **grl-apis**: Run `mvn generate-sources`, verify `ontology-merged.owl` exists per sample, create enriched zip manually
2. **Init container**: Deploy, verify `_samples_/manifest.json` and per-sample files on PVC
3. **Samples tab**: Navigate to `/ontology/upload`, verify tab with all 6 samples listed
4. **Import**: Select a sample, import, verify Upload job completes → ontology in list as User/Prepared
5. **Full lifecycle**: Open imported sample → spec tab → Transform → verify OpenAPI generated
6. **Re-import**: Import same sample twice, verify auto-suffixed name (e.g. `nzcs1_2`), both usable independently
7. **Config correctness**: Verify imported config has no `extends`/`catalogFile`, local ontology path works
8. **Tests**: Unit tests for `listAvailableSamples` and config adaptation in `samples.server.ts`
