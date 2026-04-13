# OM Layout Restructuring: Persistent AppBar + Project Toolbar

## Context

The Ontology Manager currently has no persistent top-level navigation. The landing page has its own sidebar with logo/user, and the editor has a monolithic `EditorHeader` that crams branding, project controls, platform nav, and lifecycle controls into one cluttered bar. The goal is to match GraphSandbox's pattern: a persistent AppBar with logo + hamburger + PlatformNav + user menu, and demote the editor header to a lighter contextual project toolbar.

## Changes

### 1. New `AppBar` component
**Create**: `src/components/layout/AppBar.tsx` + `AppBar.css`

- Persistent across all views (landing, editor, admin)
- Left: hamburger toggle (Material Symbols `menu` icon) + GRL logo (clickable → landing)
- Right: `PlatformNav` + `UserZone` (compact mode — avatar only)
- CSS: `var(--bg-secondary)`, `border-bottom`, compact height (~48px)
- Hamburger hidden in editor mode (panels have their own collapse via resize handles)

### 2. `UserZone` — add compact + dropdown placement
**Modify**: `src/platform/user-zone/UserZone.tsx` + `UserZone.css`

- Add `compact?: boolean` prop — when true, show only the avatar circle (no name/role text)
- Add `placement?: 'above' | 'below'` prop (default `'above'`) — controls popover direction
- CSS: `.user-zone--compact` for width/layout, `.user-zone-popover--below` for `top` instead of `bottom` positioning
- Backward compatible — existing landing sidebar usage unchanged

### 3. Refactor `EditorHeader` → `ProjectToolbar`
**Rename**: `EditorHeader.tsx` → `ProjectToolbar.tsx`, create `ProjectToolbar.css`

Remove from toolbar (now in AppBar):
- Logo, "// ontology manager" title, back button (`.header-left`)
- `PlatformNav`

Keep (as a single-row flex bar):
- ProjectSelector, VersionSelector, StyleGuideSelector
- Mode toggle (Editor/Browse)
- Validate, Imports, Help buttons
- Lifecycle controls (badge, branch pill, action buttons)

Style: lighter than AppBar — `var(--bg-panel)` background, smaller padding (`6px 16px`), subtle `border-bottom: 1px solid var(--border-light)`

### 4. Modify `LandingSidebar`
**Modify**: `src/components/landing/LandingSidebar.tsx` + `LandingPage.css`

- Remove logo/title header block (now in AppBar)
- Remove `UserZone` from footer (now in AppBar)
- Add `collapsed: boolean` prop — collapses to `width: 0` with `transition: width 0.2s`
- Remove `onManageUsers`, `onOpenCommandPalette` props

### 5. Modify `LandingPage`
**Modify**: `src/components/landing/LandingPage.tsx`

- Accept `sidebarCollapsed: boolean` prop, pass to `LandingSidebar`
- Remove `onManageUsers`, `onOpenCommandPalette` props (moved to AppBar)
- CSS: change `.landing-page` from `height: 100vh` to `flex: 1`

### 6. Modify `UserManagementHub`
**Modify**: `src/platform/users/UserManagementHub.tsx`

- Keep `onBack` and the back button (as in-content breadcrumb) — simpler than routing through AppBar
- Remove the full `<header className="hub-header">` wrapper; just render back link + title inline in the content area

### 7. Wire up in `App.tsx`
**Modify**: `src/App.tsx`

- Add `sidebarCollapsed` state with localStorage persistence (`om-sidebar-collapsed`)
- Wrap all three modes in a shared shell:
  ```
  <div className="app-shell">
    <AppBar ... />
    <div className="app-shell-body">
      {mode content}
    </div>
  </div>
  ```
- Pass `sidebarCollapsed` to `LandingPage`
- Replace `EditorHeader` import with `ProjectToolbar`
- The existing `.app` class stays as the inner editor flex container (no longer needs `height: 100vh`)

### 8. CSS cleanup
- **`app-layout.css`**: Add `.app-shell`, `.app-shell-body`; change `.app` from `height: 100vh` to `flex: 1; display: flex; flex-direction: column`
- **`Header.css`**: Remove `.header`, `.header-left`, `.header-right`, `.header-logo`, `.header-title`, `.btn-home`; keep `.mode-toggle`, `.lifecycle-*`, selector styles (or move to `ProjectToolbar.css`)
- **`LandingPage.css`**: Remove `.landing-sidebar-header`, `.landing-sidebar-logo`, `.landing-sidebar-title`, `.landing-sidebar-user` styles; add `.landing-sidebar.collapsed` transition

## Files to create
- `src/components/layout/AppBar.tsx`
- `src/components/layout/AppBar.css`
- `src/components/layout/ProjectToolbar.css`

## Files to modify
- `src/App.tsx`
- `src/components/layout/EditorHeader.tsx` → rename to `ProjectToolbar.tsx`
- `src/components/layout/Header.css`
- `src/components/landing/LandingPage.tsx`
- `src/components/landing/LandingSidebar.tsx`
- `src/components/landing/LandingPage.css`
- `src/platform/user-zone/UserZone.tsx`
- `src/platform/user-zone/UserZone.css`
- `src/platform/users/UserManagementHub.tsx`
- `src/styles/app-layout.css`

## Verification
1. `npm run dev` — all three modes render with AppBar at top
2. Landing: hamburger toggles sidebar, PlatformNav + user avatar visible in AppBar
3. Editor: ProjectToolbar shows below AppBar with project controls, no duplicate logo/nav
4. Admin: AppBar visible, back button inside hub content works
5. Logo click in AppBar returns to landing (with unsaved-changes guard in editor)
6. UserZone popover opens downward in AppBar, upward in landing sidebar (if kept there — it's removed from sidebar in this plan)
7. Sidebar collapse state persists across page refresh (localStorage)
