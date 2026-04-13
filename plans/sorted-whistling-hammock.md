# Plan: Refactor ontology editing into a tabbed workspace

## Context

The ontology editing flow currently uses three separate routes (`/ontology/upload/:name`, `/ontology/annotate/:name`, `/ontology/asg/:name`) with forward/back links between them. This feels like a rigid pipeline but users actually jump between tasks freely. Replacing them with a single workspace at `/ontology/:name` with tabs (Source, Annotate, API Spec) gives a more natural editing experience.

## Route structure change

**Before:**
```
/ontology (context.tsx)
  ├── index → ontologies.tsx (list)
  ├── upload → ontology.tsx (create new)
  ├── upload/:name? → ontology.tsx (edit existing)
  ├── asg/:name → asg.tsx
  ├── annotate/:name → annotate.tsx
  └── query/:name → query.ts
```

**After:**
```
/ontology (context.tsx)
  ├── index → ontologies.tsx (list)
  ├── upload → ontology.tsx (create new only)
  ├── :name (workspace.tsx — tab bar + shared loader)
  │   ├── index → sourceTab.tsx
  │   ├── annotate → annotateTab.tsx
  │   └── spec → specTab.tsx
  └── query/:name → query.ts (unchanged)
```

`upload` is listed before `:name` in routes.ts — React Router matches static segments first.

## Files to create

### `frontend/app/routes/ontology/workspace.tsx` — layout route

**Loader:** `loadOntologyDescription(user, name)` — lightweight (file read, no SPARQL). Returns `OntologyDescription`.

**Component:** Renders:
- Header with ontology name
- Tab bar using `<NavLink>` elements styled as Flowbite underline tabs (match `border-b-2` pattern)
- Three tabs: **Source** (always enabled), **Annotate** (enabled when `isPrepared`), **API Spec** (enabled when `isPrepared`)
- `<Outlet context={{ ontology }} />`

**Outlet context type** (exported):
```typescript
export type WorkspaceContext = { ontology: OntologyDescription };
```

Tab NavLinks use relative paths: `"."` (end=true), `"annotate"`, `"spec"`. Disabled tabs render as `<span>` with muted styling.

### `frontend/app/routes/ontology/sourceTab.tsx` — Source tab

Extract the **edit-existing** path from `ontology.tsx`. Always has a name (from parent `:name` param).

**Loader:** Fetches `publishedOntologies` + `omAvailable` (same as current ontology.tsx loader minus the existing ontology lookup — that comes from workspace context).

**Action:** Same as current `ontology.tsx` action but:
- Remove `create-new` handling
- `saveUpload` redirects to `/ontology/${name}/spec` (was `/ontology/asg/${name}`)
- `handleFormAction`'s "already prepared" redirect goes to `/ontology/${name}/spec`

**Component:** Reuses `UploadForm` and `OmPicker` (keep inline or extract to shared file). Removes "Step 1" header. Gets `ontology` from `useOutletContext<WorkspaceContext>()`.

### `frontend/app/routes/ontology/annotateTab.tsx` — Annotate tab

Copy of current `annotate.tsx` with minimal changes:
- Remove header ("Annotate Ontology")
- Footer transform link: `../spec?transform=${apiId}` (was `/ontology/asg/${name}?transform=...`)
- Loader/action/state management identical — heavy SPARQL queries only run when tab is active (child route lazy loading)

### `frontend/app/routes/ontology/specTab.tsx` — API Spec tab

Copy of current `asg.tsx` with minimal changes:
- Remove header ("Step 2: Transform...")
- Remove Back/Annotation Editor buttons (tabs replace these)
- Keep "Finish" button → `/generate`
- `?transform=` auto-trigger still works (this is a route with its own URL)

## Files to modify

### `frontend/app/routes.ts`

Replace:
```ts
route("upload/:name?", "routes/ontology/ontology.tsx", { id: "routes/upload.existing" }),
route("asg/:name", "routes/ontology/asg.tsx"),
route("annotate/:name", "routes/ontology/annotate.tsx"),
```
With:
```ts
route(":name", "routes/ontology/workspace.tsx", [
  index("routes/ontology/sourceTab.tsx"),
  route("annotate", "routes/ontology/annotateTab.tsx"),
  route("spec", "routes/ontology/specTab.tsx"),
]),
```

### `frontend/app/routes/ontology/ontologies.tsx`

Edit button: `/ontology/upload/${name}` → `/ontology/${name}`

### `frontend/app/routes/ontology/ontology.tsx`

Now create-new only (no `:name` param variant). Update redirects:
- `saveUpload`: `/ontology/asg/${existing}` → `/ontology/${existing}/spec`
- "Already prepared" shortcut: `/ontology/asg/${existing}` → `/ontology/${existing}/spec`

## Files to delete

- `frontend/app/routes/ontology/asg.tsx` (content moved to specTab.tsx)
- `frontend/app/routes/ontology/annotate.tsx` (content moved to annotateTab.tsx)

## Key design decisions

1. **Each tab keeps its own loader/action** — annotate's heavy SPARQL queries only fire when that tab is clicked. No upfront loading penalty.
2. **Workspace loader is lightweight** — just reads the ontology YAML manifest for phase gating and display.
3. **Tab state is URL-driven** — browser back/forward works, tab persists on refresh.
4. **No backward-compat redirects needed initially** — the old URLs aren't public API. Can add redirect routes later if needed.
5. **Upload JobTracker** — stays in sourceTab. If user switches tab mid-upload, the job continues server-side and the source tab reloads state on return. Simpler than lifting modal to workspace.

## Verification

1. `npm run build -w shared && npm run build -w frontend && npm run build -w builder`
2. `npm test -w frontend` — all tests pass
3. Manual testing:
   - Ontology list → Edit → lands on Source tab at `/ontology/:name`
   - Upload a new file → JobTracker modal → redirects to Spec tab
   - Click Annotate tab → loads annotation tree
   - Click API Spec tab → shows config fields + transform button
   - `?transform=` auto-trigger from annotate works
   - Annotate/Spec tabs disabled for non-Prepared ontologies
   - Create new ontology at `/ontology/upload` → redirects to workspace
   - `/ontology/query/:name` still works
