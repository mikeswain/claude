# Plan: Copy API Token from User Menu

## Context

Getting a Bearer token to use in curl/scripts currently requires a manual Keycloak curl
command + kubectl secret lookup. But the UI already holds the user's current session token
in localStorage (`auth_token`) — regardless of whether it's a local session token or a
Keycloak JWT. Exposing it via a one-click "Copy API Token" menu item eliminates that friction.

## Approach

Add a **"Copy API Token"** item to the user menu in `Sidebar.tsx`. On click it reads
`localStorage.getItem('auth_token')`, writes it to the clipboard, and briefly shows
"Copied!" feedback. Show the item only when the user is authenticated (i.e. a token exists).

No new API calls, no new backend endpoints, no new components — the token is already there.

## Files to Modify

- `designer-ui/src/components/layout/Sidebar.tsx` — only file that changes

## Implementation Detail

The user dropdown menu is built inline in `Sidebar.tsx` (lines ~310-350, after the Profile
Settings item). Add one new menu button immediately after Profile Settings:

```tsx
{/* Copy API Token */}
{(() => {
  const token = localStorage.getItem('auth_token');
  if (!token) return null;
  return (
    <button
      onClick={() => {
        navigator.clipboard.writeText(token).then(() => {
          setCopiedToken(true);
          setTimeout(() => setCopiedToken(false), 2000);
        }).catch(() => {});
      }}
      className="... same classes as other menu items ..."
    >
      <ClipboardIcon className="w-4 h-4" />
      <span>{copiedToken ? 'Copied!' : 'Copy API Token'}</span>
    </button>
  );
})()}
```

State: add `const [copiedToken, setCopiedToken] = useState(false)` alongside existing
state in the Sidebar component.

Icon: reuse whatever icon library is already imported (check existing imports for clipboard
or copy icons; if none, use an inline SVG matching the existing icon style).

This follows the identical pattern used in `HelpCodeBlock` (navigator.clipboard +
2-second timeout + state toggle).

## What it looks like

User dropdown:
- Profile Settings
- **Copy API Token** → shows "Copied!" for 2s after click
- Dark Mode toggle
- Sign Out

Hidden automatically when no token exists (e.g. if auth is disabled and no token in storage).

## Verification

1. `cd designer-ui && npm run dev`
2. Log in with local auth — open user menu, click "Copy API Token", paste into terminal:
   `curl -H "Authorization: Bearer <pasted>" http://localhost:8091/projects`  → 200
3. Check "Copied!" feedback appears and reverts after 2s
4. Log out — verify menu item disappears (no token in localStorage)
5. If Keycloak JWT auth available in staging: repeat step 2 with JWT token
