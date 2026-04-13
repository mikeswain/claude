# Global App Bar — Implementation Plan

## Context

The three GRL consumer apps (GraphSandbox, FHIR Mapper, Ontology Manager) have inconsistent
top bars. Boss approved a consistent global app bar across all apps. GraphSandbox already has
the correct layout — the other two need to match it.

This is a copy-paste pattern (not shared package). Each app keeps its native UI framework.

## Reference: GraphSandbox root.tsx:284-301

```
┌─────────────────────────────────────────────────────────────────┐
│ [GRL Logo]  [☰]                    [⊞ Nav] [☀/☾] [👤 User ▾]  │
└─────────────────────────────────────────────────────────────────┘
```

- Left: GRL logo (light/dark SVG swap) + hamburger toggle
- Right: PlatformNav dropdown, ThemeToggle, UserMenu dropdown
- Bar: `h-12, bg-secondary, border-bottom, px-4 py-2`
- Sidebar: `w-0`/`w-60` transition, state in localStorage

## Phase 1: FHIR Mapper (closest to reference)

**Create:**
- `designer-ui/src/components/layout/GlobalAppBar.tsx` — Tailwind + existing Icon component
  - Hamburger: `<Icon name="menu" />`
  - ThemeToggle: extract from Sidebar.tsx:108-115 (localStorage + `.dark` class toggle)
  - UserMenu: extract from Sidebar.tsx:286-303 (JWT decode from `localStorage("auth_token")`)
  - Imports existing `PlatformNav`
  - Logo: two `<img>` with `dark:hidden` / `dark:block`
- `designer-ui/public/HeaderLogoGreen.svg` — copy from GraphSandbox
- `designer-ui/public/HeaderLogoWhite.svg` — copy from GraphSandbox

**Modify:**
- `designer-ui/src/components/layout/Sidebar.tsx`
  - Add `collapsed` prop → `w-0 overflow-hidden` / `w-60` with transition
  - Remove logo block (~line 200-212)
  - Remove user dropdown + theme toggle + logout (~line 283-334)
  - Extract `getJwtClaim` to a shared util (used by both Sidebar and GlobalAppBar)
- `designer-ui/src/components/layout/Header.tsx`
  - Remove `<PlatformNav />` (moved to GlobalAppBar)
- `designer-ui/src/App.tsx`
  - Change outer div from `flex` (horizontal) to `flex flex-col` (vertical)
  - Insert `<GlobalAppBar>` at top
  - Add `sidebarCollapsed` state with localStorage persistence
  - Pass `collapsed` to Sidebar

## Phase 2: Ontology Manager

**Create:**
- `frontend/src/components/layout/GlobalAppBar.tsx` + `GlobalAppBar.css`
  - Pure CSS + Material Symbols icons (matching OM conventions)
  - ThemeToggle: add/remove `.dark` on `<html>`, persist to localStorage
  - UserMenu: avatar circle with initial, dropdown with sign-out (reuse `useAuth()`)
  - Imports existing `PlatformNav`
- `frontend/public/HeaderLogoGreen.svg` — copy from GraphSandbox
- `frontend/public/HeaderLogoWhite.svg` — copy from GraphSandbox

**Modify:**
- `frontend/src/styles/tokens.css`
  - Move current `:root` values to `.dark`
  - Add new `:root` with light-mode palette (matching GS/FHIR light theme)
  - This is the biggest risk — all existing components assume dark. May need to defer
    light mode and ship with toggle hidden initially.
- `frontend/src/components/layout/EditorHeader.tsx`
  - Remove logo + title (keep back button + project selectors)
  - Remove `<PlatformNav />`
- `frontend/src/components/landing/LandingSidebar.tsx`
  - Remove logo/title header section
  - Remove `<UserZone />` (now in GlobalAppBar)
  - Add `collapsed` prop with width transition
- `frontend/src/App.tsx`
  - Insert GlobalAppBar at top of both landing + editor render paths
  - Add sidebarCollapsed state with localStorage
  - Landing: hamburger toggles LandingSidebar
  - Editor: hamburger hidden or toggles left nav panel collapse

## Phase 3: GraphSandbox (already done)

No changes needed — GS is the reference. Just verify consistency after the other two are done.

## Phase 4: Visual QA

- All three apps: identical bar height, spacing, icon sizes
- Light/dark mode toggle works in all apps
- PlatformNav dropdown aligns consistently
- Sidebar toggle persists across page loads
- User menu shows initials + sign-out works

## Key Risk: OM Light Mode

Adding light mode to OM affects every component. Two options:
1. **Ship it** — add light palette to tokens.css, fix any visual issues
2. **Defer it** — ship GlobalAppBar with theme toggle hidden, dark-only for now

Recommend option 2 for initial delivery, light mode as follow-up.

## Auth Patterns Per App

| App | User info source | Logout mechanism |
|-----|-----------------|-----------------|
| GraphSandbox | `useUserStore()` → user object | NavLink to `/logout` |
| FHIR Mapper | `getJwtClaim(localStorage("auth_token"), "preferred_username")` | Keycloak redirect |
| Ontology Mgr | `useAuth()` → user object with firstName/lastName/roles | `useAuth().logout()` |

## Verification

For each app:
1. `npm run dev` — bar renders with logo, hamburger, nav, theme, user
2. Click hamburger — sidebar toggles, state persists on reload
3. Click PlatformNav — dropdown shows all apps from nav.json
4. Click theme toggle — light/dark switches (FHIR + GS; OM deferred)
5. Click user avatar — dropdown shows name + sign-out
6. Click sign-out — redirects to Keycloak logout
