# Plan: GS UI Harmonization

## Context

The three GRL frontends (OM, FHIR, GS) grew independently with divergent styling.
OM and FHIR share a navy dark palette and orange accent; GS uses Tailwind defaults
(gray/blue) with Space Grotesk font, top navbar, and `prefers-color-scheme` dark mode.

**Goal:** Align GS visually with OM/FHIR — shared color tokens, system fonts,
dark-by-default, and a collapsible sidebar drawer nav instead of top navbar.
Flowbite and Heroicons stay.

**Decisions:**
- Accent color: `#c76919` (OM's burnt orange)
- Keep Flowbite components, theme them with CSS tokens
- Keep Heroicons (no Material Symbols migration)

---

## Phase 1: Design Tokens + Tailwind Theme

Replace GS's default Tailwind gray/blue palette with OM/FHIR navy dark tokens.

### Modify: `frontend/app/app.css`

1. **Remove** Space Grotesk `@font-face` and the Syne font file
2. **Add** JetBrains Mono via Google Fonts link (in `root.tsx`) or `@import`
3. **Replace** `@theme static` block with token-aligned theme:

```css
@theme static {
  /* Typography */
  --font-sans: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, "SF Pro Display", sans-serif;
  --font-mono: "JetBrains Mono", "SF Mono", "Fira Code", monospace;

  /* Brand */
  --color-accent: #c76919;
  --color-accent-hover: #a85614;

  /* Backgrounds (navy dark palette, matching OM/FHIR) */
  --color-bg-primary: #0d1921;
  --color-bg-secondary: #122538;
  --color-bg-surface: #1a2d3d;
  --color-bg-input: #0f1e2b;

  /* Text */
  --color-text-primary: #ffffff;
  --color-text-secondary: #94a3b8;
  --color-text-muted: #64748b;

  /* Borders */
  --color-border: #2a3f52;
  --color-border-light: #1e3345;

  /* Status */
  --color-success: #5cc98d;
  --color-warning: #f59e0b;
  --color-error: #ef4444;
  --color-info: #5b9bd5;
}
```

4. **Replace** `html, body` block — make dark-by-default:

```css
html, body {
  background-color: var(--color-bg-primary);
  color: var(--color-text-primary);
  color-scheme: dark;
}
```

5. **Update** link base layer to use accent:

```css
@layer base {
  a { color: var(--color-accent); }
  a:hover { color: var(--color-accent-hover); }
}
```

### Modify: `frontend/app/components/tree.css`

Update tree CSS variables to use the new tokens instead of Tailwind gray/blue
defaults. Remove the `prefers-color-scheme` media query (dark-only now):

```css
:root {
  --tree-accent: var(--color-accent);
  --tree-selected-bg: var(--color-bg-surface);
  --tree-hover-bg: var(--color-bg-secondary);
  --tree-indicator: var(--color-accent);
  --tree-drop-bg: var(--color-bg-secondary);
  --tree-search-bg: var(--color-bg-secondary);
  --tree-desc-bg: var(--color-bg-surface);
  --tree-border: var(--color-border);
  --tree-text: var(--color-text-secondary);
  --tree-assist-bg: var(--color-bg-secondary);
}
```

---

## Phase 2: Flowbite Theme + Root Layout

### Modify: `frontend/app/root.tsx`

1. **Expand `customTheme`** — theme all Flowbite components with token colors:

```typescript
const customTheme = createTheme({
  button: {
    color: {
      primary: "bg-[var(--color-accent)] hover:bg-[var(--color-accent-hover)] text-white",
      secondary: "bg-[var(--color-bg-surface)] hover:bg-[var(--color-bg-secondary)] text-[var(--color-text-primary)] border border-[var(--color-border)]"
    }
  },
  navbar: {
    root: { base: "bg-[var(--color-bg-secondary)] border-b border-[var(--color-border-light)]" },
    link: { active: { on: "text-[var(--color-accent)]", off: "text-[var(--color-text-secondary)] hover:text-[var(--color-text-primary)]" } }
  },
  dropdown: {
    floating: { base: "bg-[var(--color-bg-surface)] border border-[var(--color-border)] text-[var(--color-text-primary)]" },
    item: { base: "hover:bg-[var(--color-bg-secondary)] text-[var(--color-text-secondary)]" }
  },
  textInput: {
    field: { input: { base: "bg-[var(--color-bg-input)] border-[var(--color-border)] text-[var(--color-text-primary)] focus:border-[var(--color-accent)]" } }
  },
  select: {
    field: { select: { base: "bg-[var(--color-bg-input)] border-[var(--color-border)] text-[var(--color-text-primary)]" } }
  },
  modal: {
    content: { base: "bg-[var(--color-bg-surface)] border border-[var(--color-border)]" },
    header: { base: "border-b border-[var(--color-border)]" }
  },
  spinner: {
    color: { info: "fill-[var(--color-accent)]" }
  },
  badge: {
    root: { color: { info: "bg-[var(--color-accent)] text-white" } }
  },
  alert: {
    root: { color: {
      success: "bg-[var(--color-success)]/10 text-[var(--color-success)]",
      failure: "bg-[var(--color-error)]/10 text-[var(--color-error)]",
      warning: "bg-[var(--color-warning)]/10 text-[var(--color-warning)]"
    } }
  },
  // Table theme added in Phase 4
});
```

The exact theme keys will need validation against the Flowbite React API at
implementation time — the structure above is directional. Inspect the Flowbite
source or use `console.log(getTheme())` to confirm the correct nesting.

2. **Update Google Fonts link** — replace Inter with JetBrains Mono:

```typescript
export const links: Route.LinksFunction = () => [
  { rel: "preconnect", href: "https://fonts.googleapis.com" },
  { rel: "preconnect", href: "https://fonts.gstatic.com", crossOrigin: "anonymous" },
  { rel: "stylesheet", href: "https://fonts.googleapis.com/css2?family=JetBrains+Mono:wght@400;500;600&display=swap" }
];
```

3. **Update layout classes** — replace Tailwind gray/sky defaults with tokens:
   - Body: remove `dark:` prefixed classes (dark-only now)
   - Main container: use token-based colors
   - Footer: `bg-[var(--color-bg-secondary)]` instead of `bg-sky-200 dark:bg-sky-900`
   - Remove dual logo switching (`block dark:hidden` / `hidden dark:block`) — use white logo only

4. **Remove** `dark:` class variants from navbar/footer (single dark theme)

---

## Phase 3: Top Bar + Sidebar Drawer Navigation

Two-tier navigation matching FHIR's pattern:
- **Top app bar** — persistent, full width: GRL logo (left), platform/global nav (center-right), user menu (far right)
- **Sidebar drawer** — collapsible, for in-app section nav

### Top App Bar

Restyle the existing Flowbite `<Navbar>` (keep the component):
- **Left**: GRL logo (white variant only, dark theme)
- **Center-right**: Platform nav links to sibling GRL apps. Port FHIR's `PlatformNav`
  pattern — fetch `/nav.json` at runtime, render as icon+text links. Uses Heroicons
  instead of Material Symbols (GS keeps Heroicons).
  Reference: `/home/mike/grl/FHIR-Message-Mapper-Service/designer-ui/src/components/layout/PlatformNav.tsx`
- **Far right**: User avatar + dropdown menu (existing Flowbite `Avatar`/`Dropdown`, moved from current `UserNav`)
- Styling: `bg-[var(--color-bg-secondary)]`, `border-b border-[var(--color-border-light)]`

### New file: `frontend/app/components/PlatformNav.tsx`

Port from FHIR's `PlatformNav.tsx` — fetches `/nav.json`, renders cross-app links.
Adapt to use Heroicons instead of Material Symbols `<Icon>`.

### New file: `frontend/app/components/Sidebar.tsx`

Collapsible drawer for in-app section navigation:
- **Expanded** (~240px): icon + text label for each nav item
- **Collapsed** (0px / hidden): sidebar off-screen
- **Toggle**: hamburger button (`Bars3Icon` from Heroicons) in top-left of app bar
- **State**: `useState` in root layout, persisted to `localStorage`

Nav items (from current `UserNav`):
| Label | Icon (Heroicon) | Route |
|-------|-----------------|-------|
| Home | `HomeIcon` | `/` |
| Assets | `ServerStackIcon` | `/asset` |
| Builds | `WrenchScrewdriverIcon` | `/build` |
| Ontologies | `OntologyIcon` (custom) | `/ontology` |
| OWL Catalog | `BuildingLibraryIcon` | `/library` |

Styling:
- Background: `var(--color-bg-secondary)`
- Active item: `bg-[var(--color-bg-surface)]` with left accent bar `var(--color-accent)`
- Hover: `bg-[var(--color-bg-surface)]`
- Text: `var(--color-text-primary)`, muted for inactive `var(--color-text-secondary)`
- Border-right: `var(--color-border-light)`
- Transition: `transition-all duration-200`

### Modify: `frontend/app/root.tsx`

1. **Rework layout** — top bar + sidebar + main content:

```tsx
<body className="min-h-dvh flex flex-col">
  <header>
    {/* Top app bar: hamburger, logo, platform nav, user menu */}
  </header>
  <div className="flex flex-1">
    <Sidebar collapsed={collapsed} />
    <main className="flex-1 p-4">
      <ThemeProvider theme={customTheme}>{children}</ThemeProvider>
    </main>
  </div>
  <footer>...</footer>
</body>
```

2. **Remove** `UserNav` function — split into: section nav items → `Sidebar`, user dropdown → top bar right
3. **Remove** `NavbarCollapse`, `NavbarToggle` imports (replaced by sidebar)
4. **Keep** `Navbar`, `NavbarBrand` for top bar structure
5. **Remove** dual logo switching — white logo only
6. **Keep** `Avatar`, `Dropdown`, `DropdownItem`, `DropdownDivider`, `DropdownHeader`, `Spinner`, `ThemeProvider`, `createTheme`

---

## Phase 4: Table Styling via Flowbite Theme

Theme Flowbite's `Table` components via `createTheme` in `root.tsx` to match OM's
table pattern (reference: OM's `LandingPage.css` lines 184–289). No component
replacement needed — just theme overrides.

Add to `customTheme` in `root.tsx`:

```typescript
table: {
  root: {
    wrapper: "border border-[var(--color-border)] rounded-lg overflow-hidden",
    base: "w-full border-collapse"
  },
  head: {
    base: "bg-[var(--color-bg-secondary)]",
    cell: {
      base: "px-4 py-2.5 text-xs font-semibold uppercase tracking-wider text-[var(--color-text-muted)] border-b border-[var(--color-border)]"
    }
  },
  body: {
    base: "",
    cell: {
      base: "px-4 py-3 text-sm text-[var(--color-text-secondary)] border-b border-[var(--color-border-light)] [tr:last-child_&]:border-b-0"
    }
  },
  row: {
    base: "transition-colors duration-150 hover:bg-white/[0.03]"
  }
}
```

Files using Flowbite Table (will pick up theme automatically):
- `frontend/app/routes/ontology/ontologies.tsx`
- `frontend/app/routes/assets/assetGroups.tsx`
- `frontend/app/routes/assets/assetGroup.tsx`
- `frontend/app/routes/builds/assetBuilds.tsx`
- `frontend/app/routes/library/library.tsx`

---

## Phase 5: Component-by-Component Color Migration

Grep for `dark:` prefixed classes and Tailwind default color classes (`bg-gray-`,
`text-gray-`, `border-gray-`, `bg-blue-`, `text-blue-`) across all route/component
files. Replace with token-based equivalents.

### Mapping guide

| Current Tailwind | Token replacement |
|-----------------|-------------------|
| `bg-white` / `bg-gray-50` / `dark:bg-gray-900` | `bg-[var(--color-bg-primary)]` |
| `bg-gray-100` / `dark:bg-gray-800` | `bg-[var(--color-bg-secondary)]` |
| `bg-gray-200` / `dark:bg-gray-700` | `bg-[var(--color-bg-surface)]` |
| `text-gray-900` / `dark:text-white` | `text-[var(--color-text-primary)]` |
| `text-gray-500` / `dark:text-gray-400` | `text-[var(--color-text-secondary)]` |
| `text-gray-400` / `dark:text-gray-500` | `text-[var(--color-text-muted)]` |
| `border-gray-300` / `dark:border-gray-600` | `border-[var(--color-border)]` |
| `bg-blue-600` / `hover:bg-blue-700` | `bg-[var(--color-accent)]` / `hover:bg-[var(--color-accent-hover)]` |
| `text-blue-600` / `dark:text-blue-400` | `text-[var(--color-accent)]` |
| `text-green-500` | `text-[var(--color-success)]` |
| `text-red-500` | `text-[var(--color-error)]` |
| `text-yellow-500` | `text-[var(--color-warning)]` |

### Files to update (grep for `dark:` and `bg-gray-`):
- `app/routes/ontology/ontologies.tsx`
- `app/routes/ontology/ontology.tsx`
- `app/routes/ontology/annotate.tsx`
- `app/routes/assets/assetGroups.tsx`
- `app/routes/builds/assetBuilds.tsx`
- `app/routes/welcome.tsx`
- `app/components/SubtaskList.tsx`
- `app/components/UserCard.tsx`
- `app/components/JobTracker.tsx`
- `app/components/AnnotationFilter.tsx`
- Plus any others found by grep

---

## Phase 6: Cleanup

1. **Delete** `frontend/public/SpaceGrotesk-VariableFont_wght.ttf`
2. **Delete** `frontend/public/Syne-VariableFont_wght.ttf` (if present)
3. **Remove** light-mode logo (`HeaderLogoGreen.svg`) if no longer referenced
4. **Verify** no remaining `dark:` classes (should be none — single dark theme)
5. **Verify** no remaining raw Tailwind gray/blue color classes
6. **Verify** Flowbite Table theme renders correctly against OM reference

---

## Implementation Order

1. Phase 1 (tokens + app.css) — foundation
2. Phase 2 (root.tsx + Flowbite theme) — font swap, theme colors
3. Phase 3 (top bar + sidebar nav) — new layout structure
4. Phase 4 (tables) — theme Flowbite tables to match OM style
5. Phase 5 (component color migration) — bulk `dark:`/gray/blue replacement
6. Phase 6 (cleanup) — remove unused assets

Phases 1–2 together. Phase 3 as a focused session. Phases 4–5 iteratively. Phase 6 as final sweep.

---

## Verification

After each phase:
- `npx tsc --noEmit` in frontend — type check
- `npm run dev -w frontend` — visual inspection in browser
- Compare side-by-side with OM (dark navy, burnt orange accent, system fonts)

Final checks:
```bash
# No remaining dark: prefixes (should be zero or near-zero)
grep -r "dark:" frontend/app/ --include="*.tsx" --include="*.css" | wc -l

# No raw Tailwind grays in component files
grep -rE "bg-gray-|text-gray-|border-gray-" frontend/app/ --include="*.tsx" | wc -l

# No old font references
grep -r "Grotesk\|Inter" frontend/app/ --include="*.tsx" --include="*.css"
```
