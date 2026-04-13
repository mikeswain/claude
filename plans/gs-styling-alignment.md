# Plan: Unify GRL UI Styling

## Context

The three GRL frontends grew independently and have divergent styling:
- **OM**: Pure CSS variables (50+ tokens across 4 files), no component library, dark-only
- **FHIR**: Tailwind v4 + CSS variables + custom components, light/dark toggle
- **GS**: Tailwind v4 + Flowbite React, Heroicons, no CSS variable system

OM and FHIR are roughly aligned (CSS vars, Material Symbols, orange accent) but both
have organic growth — OM's tokens are spread across files, FHIR duplicates some patterns.

**Goal:** First, clean up and consolidate the OM/FHIR CSS into a clean shared token
system and reference component patterns. Then align GS to that standard. The end result
is all three apps looking like one product suite, with the styling approach being
maintainable and succinct enough to serve as the reference going forward.

---

## Phase 0: Consolidate OM/FHIR Shared Token System

Before touching GS, clean up the reference standard.

### 0a. Audit and unify tokens

Compare OM's `tokens.css` with FHIR's `:root` / `.dark` variables in `index.css`.
Produce a single canonical token set that both can adopt:

**OM tokens file:** `~/grl/Ontology-Manager/frontend/src/styles/tokens.css`
**FHIR tokens:** `~/grl/FHIR-Message-Mapper-Service/designer-ui/src/index.css`

Reconcile differences:
- Accent: OM `#c76919` vs FHIR `#FF8400` — pick one
- Background naming: OM `--bg-primary` vs FHIR `--background` — standardise
- Status colors: should be identical
- Borders, surfaces, muted text — unify naming

Output: a reference `tokens.css` with clear sections (backgrounds, text, accents,
status, borders, typography, spacing). Both OM and FHIR adopt it.

### 0b. Clean up OM's CSS structure

OM has 4 CSS files (`tokens.css`, `reset.css`, `globals.css`, `app-layout.css`) with
component-level classes (`.btn`, `.btn-primary`, `.form-input`, etc.) mixed into globals.

- Keep `tokens.css` (cleaned up from 0a)
- Keep `reset.css` (minimal, fine as-is)
- Evaluate `globals.css` — what's reusable vs what should be component-scoped?
- Consider whether FHIR's approach (Tailwind utilities + CSS vars) is more maintainable
  than OM's hand-rolled component classes

### 0c. Evaluate shared package

Decide whether the canonical tokens should live in:
1. Each repo independently (copy, risk drift)
2. A shared npm package in grl-platform (extra build step but single source of truth)
3. Just documented convention (lightest touch)

Option 3 is fine to start — extract to a package later if drift becomes a problem.

---

## Phase 1: Design Tokens

Create a shared CSS variable system matching FHIR/OM conventions.

**New file:** `frontend/app/styles/tokens.css`

Source values from:
- OM: `~/grl/Ontology-Manager/frontend/src/styles/tokens.css` (50+ variables)
- FHIR: `~/grl/FHIR-Message-Mapper-Service/designer-ui/src/index.css` (CSS vars in `:root` / `.dark`)

Key tokens to adopt:
```css
--bg-primary: #0d1921;        /* navy dark background */
--bg-secondary: #162736;
--bg-surface: #1a2f42;
--accent: #c76919;             /* burnt orange — or #FF8400 to match FHIR */
--text-primary: #e8edf2;
--text-secondary: #8fa3b8;
--text-muted: #5a7a94;
--border: #1e3a52;
--success: #22c55e;
--warning: #eab308;
--error: #ef4444;
--font-mono: 'JetBrains Mono', monospace;
```

**Modify:** `frontend/app/app.css` — import tokens, replace existing color definitions.

### Decision needed
- Match OM's `#c76919` or FHIR's `#FF8400` for accent? They're close but not identical.
  FHIR is brighter. Pick one and use it everywhere.

---

## Phase 2: Icon System

Replace Heroicons with Material Symbols Sharp (matching OM and FHIR).

**New file:** `frontend/app/components/ui/Icon.tsx`

Port from FHIR's `designer-ui/src/components/ui/Icon.tsx` — thin wrapper around
Material Symbols Sharp font with `name`, `size`, `weight`, `className` props.

**Add:** Material Symbols Sharp font (Google Fonts import or self-hosted).

**Then:** Find-and-replace all Heroicons imports across the codebase:
```
@heroicons/react/24/solid → Icon component
@heroicons/react/24/outline → Icon component
```

Each icon needs a name mapping (e.g. `PencilIcon` → `<Icon name="edit" />`).
Do this file-by-file to avoid breaking things.

**Remove:** `@heroicons/react` from package.json once complete.

### Files to update (grep for `@heroicons/react`):
- `app/routes/ontology/ontologies.tsx`
- `app/routes/ontology/ontology.tsx`
- `app/routes/assets/assetGroups.tsx`
- `app/routes/builds/assetBuilds.tsx`
- Plus any others

---

## Phase 3: Replace Flowbite Components

This is the biggest phase. Replace Flowbite React components with custom ones that
use the CSS token system. Build each replacement, swap usage, then remove Flowbite.

### Components to replace

| Flowbite Component | Used In | Replacement Approach |
|-------------------|---------|---------------------|
| `Button` | Everywhere | Custom `Button` with token-based styling |
| `Card` | ontology.tsx, others | Simple div with `bg-[var(--bg-surface)]` + border |
| `Table`, `TableHead`, `TableBody`, `TableRow`, `TableCell`, `TableHeadCell` | ontologies.tsx, assets | Custom table with token styling |
| `Badge` | ontologies.tsx | Span with accent colors |
| `ButtonGroup` | ontologies.tsx | Flex container |
| `TextInput` | ontology.tsx | Input with `var(--bg-secondary)` + border |
| `Label` | ontology.tsx | Styled label |
| `HelperText` | ontology.tsx | Small muted text |
| `Modal` | ontology.tsx | Dialog/overlay |
| `Spinner` | ontology.tsx | CSS animation |
| `Tabs`, `TabItem` | ontology.tsx | Custom tabs with underline variant |
| `Navbar`, `NavbarBrand`, `NavbarLink` | root.tsx / layout | Custom nav |
| `Avatar` | root.tsx | Image/initials circle |
| `Dropdown`, `DropdownItem` | root.tsx | Custom dropdown |
| `Select` | annotate.tsx | Styled select |
| `Alert`, `Toast` | annotate.tsx | Custom alert |

### Approach
1. Create `frontend/app/components/ui/` directory
2. Build each component one at a time, using CSS vars from tokens.css
3. Reference FHIR's components for patterns: `~/grl/FHIR-Message-Mapper-Service/designer-ui/src/components/ui/`
4. Swap imports file-by-file
5. After all Flowbite usage removed, delete the dependency

**Remove from package.json:** `flowbite-react`
**Remove from app.css:** `@source` directive for flowbite class list
**Remove from root.tsx:** `ThemeProvider`, `createTheme`

---

## Phase 4: Typography

**Fonts to adopt:**
- Body: System font stack (`-apple-system, BlinkMacSystemFont, 'Segoe UI', ...`) — matches OM
- Code: JetBrains Mono — matches both OM and FHIR
- Remove: Inter (Google Fonts import), Grotesk (custom font)

**Modify:** `frontend/app/app.css` — update font-family declarations.
**Remove:** Grotesk font files from `frontend/public/fonts/` if present.

---

## Phase 5: Dark Theme

OM is dark-only. FHIR has light/dark toggle. GS currently uses Flowbite's theme provider.

**Recommended:** Dark-by-default (matches OM), with optional light mode toggle later.

After replacing Flowbite components with token-based ones, dark theme comes for free —
the tokens define the dark palette. Remove Flowbite's `ThemeProvider` and `useThemeMode`.

---

## Phase 6: Layout Polish

Once tokens + components are in place, do a visual pass:
- Sidebar/nav styling to match OM/FHIR patterns
- Consistent spacing (OM/FHIR use tighter spacing than Flowbite defaults)
- Status colors (success/warning/error) aligned with tokens
- Loading states (replace Flowbite Spinner with CSS-based spinner)

---

## Implementation Order

Phases are sequential — each builds on the previous:
1. **Tokens** — foundation, no visual change yet
2. **Icons** — independent, mechanical
3. **Components** — biggest effort, do incrementally
4. **Typography** — quick win after components
5. **Dark theme** — falls out naturally from tokens + components
6. **Layout polish** — final visual pass

Phases 1-2 can be done in a single session. Phase 3 is the bulk of the work and
should be done component-by-component over multiple sessions.

---

## Verification

After each phase:
- `npx tsc --noEmit` — type check
- Visual inspection in browser — compare side-by-side with OM/FHIR
- No Flowbite imports remaining (after Phase 3 complete):
  `grep -r "flowbite" frontend/app/`
