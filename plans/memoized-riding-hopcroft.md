# Plan: Migrate from Heroicons to Material Symbols Outlined

## Context

Peer GRL projects (Ontology Manager, FHIR designer, grl-platform home page) already use Google Material Symbols. GraphSandbox currently uses `@heroicons/react` (32 files, ~40 unique icons) plus 5 custom SVG icons. Switching to Material Symbols aligns the icon system across the platform.

## Approach

Load Material Symbols Outlined via Google Fonts CDN (same as peer projects). Create a thin `<Icon>` wrapper component. Replace all Heroicons imports file-by-file. Also replace `OntologyIcon` with Material Symbol `graph_3`.

### 1. Infrastructure

**`frontend/app/components/Icon.tsx`** (new) — Thin wrapper, modelled on FHIR's `Icon.tsx`:
```tsx
interface IconProps {
  name: string;
  size?: number;       // fontSize in px (default 20, equiv to size-5)
  weight?: number;     // font weight (default 400)
  fill?: boolean;      // FILL axis (default true — matches solid Heroicons)
  className?: string;
  title?: string;
}
```
Renders `<span class="material-symbols-outlined ...">name</span>` with `fontVariationSettings`.

**`frontend/app/root.tsx`** — Add font link to `links` export:
```
https://fonts.googleapis.com/css2?family=Material+Symbols+Outlined:opsz,wght,FILL,GRAD@20..48,100..700,0..1,-50..200&display=swap
```

### 2. Icon Name Mapping

| Heroicon | Material Symbol |
|---|---|
| ArrowPathIcon | refresh |
| ArrowPathRoundedSquareIcon | sync |
| ArrowRightIcon | arrow_forward |
| Bars3Icon | menu |
| BuildingLibraryIcon | local_library |
| BuildingOffice2Icon | apartment |
| CheckBadgeIcon | verified |
| CheckCircleIcon | check_circle |
| ChevronDoubleDownIcon | keyboard_double_arrow_down |
| ChevronDoubleUpIcon | keyboard_double_arrow_up |
| ChevronDownIcon | keyboard_arrow_down |
| ChevronLeftIcon | chevron_left |
| ChevronRightIcon | chevron_right |
| ChevronUpIcon | keyboard_arrow_up |
| ClipboardDocumentCheckIcon | assignment_turned_in |
| ClipboardDocumentIcon | assignment |
| DocumentCheckIcon | task |
| EllipsisVerticalIcon | more_vert |
| EnvelopeIcon | mail |
| ExclamationCircleIcon | error |
| ExclamationTriangleIcon | warning |
| EyeIcon | visibility |
| EyeSlashIcon | visibility_off |
| FaceFrownIcon | sentiment_dissatisfied |
| HomeIcon | home |
| InformationCircleIcon | info |
| MoonIcon | dark_mode |
| PencilIcon | edit |
| PlayIcon | play_arrow |
| PlusIcon | add |
| ServerStackIcon | dns |
| ShieldCheckIcon | verified_user |
| ShieldExclamationIcon | gpp_maybe |
| Squares2X2Icon | apps |
| StopCircleIcon | stop_circle |
| SunIcon | light_mode |
| TrashIcon | delete |
| UserIcon | person |
| XCircleIcon | cancel |
| XMarkIcon | close |

**Custom SVG → Material Symbol:**
| Custom Icon | Material Symbol | Used in |
|---|---|---|
| OntologyIcon | graph_3 | Sidebar, SelectedOntologySidebar, workspace, ontology, ontologies (5 files) |
| ApiPackageIcon | deployed_code | Sidebar, step1, step2, step3, build, assetBuilds (7 files) |
| AnnotateIcon | linked_services | Sidebar only (1 file) |

**Custom SVGs kept as-is** (no Material equivalent):
- HelmIcon (Helm logo)
- GRLLogo (company logo)

### 3. Special Cases

**Flowbite Alert `icon` prop** (5 call sites) — Flowbite's `icon` prop expects `FC<ComponentProps<"svg">>`. Replace by removing the `icon` prop and rendering `<Icon>` inline inside Alert children with flex layout.
- `routes/generate/build.tsx`
- `routes/queue/JobTracker.tsx` (2 sites)
- `components/ExternalPanel.tsx`
- `routes/ontology/annotateTab.tsx`

**Sidebar polymorphic icons** — `navItems` currently stores Heroicon/custom SVG components. Nearly all icons are now Material Symbols. Only HelmIcon remains as a custom SVG component. Use `materialIcon: string` for most items, keep `icon: FC` for HelmIcon only. Branch in render.

**Rotation transitions** — `ChevronRightIcon` with `rotate-90` class (AnnotationTree, PropertyAnnotationTree) works fine on the `<span>` — CSS transforms apply to any element.

**Outline variant** — 3 files used `@heroicons/react/24/outline`: AnnotationFilter, GraphContextInfo, UserCard. These pass `fill={false}` to get the unfilled look.

### 4. Files to Modify (32 files + 1 new + 1 delete)

**New:** `frontend/app/components/Icon.tsx`

**Infrastructure:** `frontend/app/root.tsx` (add font link + replace 3 icons)

**Simple replacements (24 files):**
- `app/form/EyeToggle.tsx`, `app/form/Clipboard.tsx`
- `app/auth/LicenseSummary.tsx`
- `app/welcome/welcome.tsx`
- `app/components/PlatformNav.tsx`, `app/components/PropertyAnnotationTree.tsx`
- `app/components/AnnotationTree.tsx`, `app/components/SubtaskList.tsx`
- `app/routes/Imports.tsx`
- `app/routes/generate/step1.tsx`, `step2.tsx`, `step3.tsx`, `build.tsx`
- `app/routes/auth/login.tsx`, `app/routes/layouts/authLayout.tsx`
- `app/routes/ontology/ontology.tsx`, `ontologies.tsx`, `sourceTab.tsx`, `annotateTab.tsx`, `TransformResults.tsx`
- `app/routes/library/library.tsx`, `mapping.tsx`
- `app/routes/assets/assetGroup.tsx`, `assetGroups.tsx`
- `app/routes/builds/assetBuilds.tsx`

**Outline → fill={false} (3 files):** `AnnotationFilter.tsx`, `GraphContextInfo.tsx`, `UserCard.tsx`

**Alert icon refactor (4 files):** `build.tsx`, `JobTracker.tsx`, `ExternalPanel.tsx`, `annotateTab.tsx`

**Sidebar refactor:** `app/components/Sidebar.tsx`

**Custom icon replacements (11 files):**
- `app/components/Sidebar.tsx` — OntologyIcon→`graph_3`, ApiPackageIcon→`deployed_code`, AnnotateIcon→`linked_services`
- `app/components/SelectedOntologySidebar.tsx` — `<Icon name="graph_3" size={24} />`
- `app/routes/ontology/workspace.tsx` — `<Icon name="graph_3" size={48} />`
- `app/routes/ontology/ontology.tsx` — `<Icon name="graph_3" size={48} />`
- `app/routes/ontology/ontologies.tsx` — `<Icon name="graph_3" size={48} />`
- `app/routes/generate/step1.tsx` — `<Icon name="deployed_code" size={40} />`
- `app/routes/generate/step2.tsx` — `<Icon name="deployed_code" size={40} />`
- `app/routes/generate/step3.tsx` — `<Icon name="deployed_code" size={40} />`
- `app/routes/generate/build.tsx` — `<Icon name="deployed_code" size={40} />`
- `app/routes/builds/assetBuilds.tsx` — `<Icon name="deployed_code" size={48} />`

**Delete:** `frontend/app/icons/OntologyIcon.tsx`, `ApiPackageIcon.tsx`, `AnnotateIcon.tsx`

**Cleanup:**
- `frontend/package.json` — remove `@heroicons/react`
- `CLAUDE.md` — update icon convention from Heroicons to Material Symbols
- Move `IconProps` type from `OntologyIcon.tsx` to `HelmIcon.tsx` (only remaining custom SVG that needs it)

### 5. Verification

1. `npm run build -w frontend` — no TS errors, no heroicons imports
2. `grep -r "@heroicons" frontend/app/` — zero results
3. `npm test -w frontend` — all tests pass (update snapshots if needed)
4. Visual check in browser: header, sidebar, ontology pages, asset pages, generate wizard, library, annotations, login/auth error, user menu
5. Check browser console for hydration warnings (SSR)
