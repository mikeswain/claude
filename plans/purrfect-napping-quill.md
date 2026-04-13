# grl-frontend-web-ui: Auth0 → Generic OIDC with Runtime Config

## Context

grl-frontend-web-ui is a React SPA that browses deployed REST APIs. It's deployed inside GraphSandbox per-sandbox Helm charts. Auth is currently hardcoded to Auth0 (`@auth0/auth0-react` with credentials baked into `globalAppConfig.ts`). It also supports API key auth as an alternative.

**Two deployment scenarios need auth to work differently:**
1. **GraphSandbox K8s** — frontend inherits Keycloak auth from the platform automatically
2. **Downloaded Helm chart** — users configure their own OIDC provider or use API key only

**Approach:** Replace Auth0 with `react-oidc-context` (generic OIDC), inject auth config at runtime via nginx-served `/config.json` generated from env vars at container startup. Frontend-only auth — graph-server stays API key for now.

---

## Phase 1: Runtime Config Injection (grl-frontend-web-ui)

### 1.1 New file: `docker-entrypoint.sh`

Generates `/config.json` from env vars before nginx starts:

```sh
#!/bin/sh
cat > /usr/share/nginx/html/config.json <<EOF
{
  "oidc": {
    "authority": "${OIDC_AUTHORITY:-}",
    "clientId": "${OIDC_CLIENT_ID:-}",
    "scope": "${OIDC_SCOPE:-openid profile email}"
  }
}
EOF
exec "$@"
```

Empty `OIDC_AUTHORITY`/`OIDC_CLIENT_ID` = API-key-only mode (no OIDC provider mounted).

### 1.2 New file: `nginx.conf` (production)

```nginx
server {
    listen 80;
    root /usr/share/nginx/html;
    location /config.json { add_header Cache-Control "no-cache"; }
    location / { try_files $uri $uri/ /index.html; }
}
```

No `/api/` or `/openapi/` proxy — that's handled by the sandbox's HTTPRoute/nginx pod.

### 1.3 Modify: `Dockerfile`

- Copy `nginx.conf` and `docker-entrypoint.sh` into the image
- Remove `REACT_APP_API_URL` build arg (no longer needed — `serverBase` defaults to `window.origin`)
- Set `ENTRYPOINT ["/docker-entrypoint.sh"]` and `CMD ["nginx", "-g", "daemon off;"]`

### 1.4 New file: `src/config/loadRuntimeConfig.ts`

Fetches `/config.json` once. Returns `{ oidc?: { authority, clientId, scope } }`. Treats empty authority/clientId as "not configured" → returns `{ oidc: undefined }`.

### 1.5 Modify: `src/index.tsx`

Load config before rendering:
```typescript
loadRuntimeConfig().then(config => {
  root.render(<App runtimeConfig={config} />);
});
```

### 1.6 Dev: `public/config.json`

For local webpack dev server, a static file with dev Keycloak config or empty OIDC for API-key-only mode.

---

## Phase 2: Auth Rewrite (grl-frontend-web-ui)

### 2.1 Dependencies

- Remove: `@auth0/auth0-react`
- Add: `react-oidc-context`, `oidc-client-ts`

### 2.2 Rename `src/auth0/` → `src/auth/`

New files:

| File | Replaces | Purpose |
|------|----------|---------|
| `OidcAuthProvider.tsx` | `Auth0ProviderWithNavigate.tsx` | Wraps app in `react-oidc-context` AuthProvider |
| `ApiKeyAuthProvider.tsx` | `AnonymousAuth0Provider.tsx` | Provides auth context for API key mode |
| `useAuthContext.ts` | (new) | Unified hook abstracting OIDC vs API key |
| `AuthenticationGuard.tsx` | same | Simplified — uses `useAuthContext` |
| `LoginButton.tsx` | same | OIDC login hidden when not configured |
| `UserMenu.tsx` | same | Uses `useAuthContext` instead of `useAuth0` |

### 2.3 `useAuthContext.ts` — unified auth hook

```typescript
type AuthContext = {
  isAuthenticated: boolean;
  isLoading: boolean;
  user?: { name?: string; picture?: string; email?: string };
  login: () => void;
  logout: () => void;
  getAccessToken: () => Promise<string | undefined>;
  authMode: 'oidc' | 'apikey' | 'none';
};
```

- **OIDC mode**: delegates to `useAuth()` from `react-oidc-context`
- **API key mode**: always authenticated, no token (API key sent via `X-Api-Key` header separately)
- **None mode**: not authenticated, login is no-op

### 2.4 `OidcAuthProvider.tsx`

Uses `react-oidc-context`'s `AuthProvider`:
- `authority`: from runtime config
- `client_id`: from runtime config
- `redirect_uri`: `window.location.origin`
- `scope`: from runtime config (default: `openid profile email`)
- `onSigninCallback`: strips OIDC params from URL via `window.history.replaceState`
- `automaticSilentRenew: true`

### 2.5 Modify: `src/App.tsx` — `AuthLayout`

```typescript
function AuthLayout({ runtimeConfig }) {
  const [apiKey] = useLocalStorage("apiKey", "");
  const children = <AppContextProvider ...>...</AppContextProvider>;

  if (apiKey) return <ApiKeyAuthProvider>{children}</ApiKeyAuthProvider>;
  if (runtimeConfig.oidc) return <OidcAuthProvider config={runtimeConfig.oidc}>{children}</OidcAuthProvider>;
  return <NoAuthProvider>{children}</NoAuthProvider>;  // API-key-only mode, shows login card
}
```

### 2.6 Modify: `src/api/useHeaders.ts`

Replace `useAuth0` with `useAuthContext`. Same dual-path logic:
- API key → `X-Api-Key` header
- OIDC → `Authorization: Bearer <token>` via `getAccessToken()`

### 2.7 Modify: `src/SharedLayout.tsx`

Replace `import { useAuth0 } from "@auth0/auth0-react"` with `useAuthContext`. Same `isLoading`/`isAuthenticated` checks.

### 2.8 Modify: `src/globalAppConfig.ts`

- Remove `auth` field and `AuthConfig` type (replaced by runtime config)
- Remove hardcoded Auth0 credentials
- Keep `serverBase` defaulting to `window.origin`

### 2.9 Update tests

- `AppContext.test.tsx`, `ModelDialog.test.tsx` — replace `jest.mock("@auth0/auth0-react")` with mock of `useAuthContext`

---

## Phase 3: Per-Sandbox Helm Chart (GraphSandbox)

### 3.1 Modify: `builder/k8s/chart/templates/frontend.yaml`

Add env vars to the frontend container:
```yaml
env:
  {{- with .Values.frontend.oidc }}
  - name: OIDC_AUTHORITY
    value: {{ .authority | quote }}
  - name: OIDC_CLIENT_ID
    value: {{ .clientId | quote }}
  {{- end }}
```

### 3.2 Modify: `builder/k8s/chart/values.yaml`

```yaml
frontend:
  enabled: false
  image: frontend-web-ui
  port: 80
  oidc: {}    # Set by builder when deploying in GRL platform
```

### 3.3 Modify: `shared/src/helm.ts`

Add to `SandboxHelmValues` / frontend type:
```typescript
frontend: {
  enabled: boolean;
  oidc?: { authority?: string; clientId?: string; scope?: string };
};
```

### 3.4 Modify: `builder/src/deployToK8s.ts`

When deploying a sandbox, read the Keycloak issuer from the `grl-platform-keycloak` ConfigMap and pass it as Helm values:
```typescript
--set frontend.oidc.authority=<keycloakIssuer>
--set frontend.oidc.clientId=graph-sandbox
```

Uses existing `@kubernetes/client-node` already imported in the builder.

---

## Phase 4: Keycloak Client Config (grl-platform)

### 4.1 Redirect URI wildcards

The `graph-sandbox` client in grl-platform values needs a wildcard hostname to cover per-sandbox subdomains:
```yaml
keycloak:
  clients:
    graph-sandbox:
      hostname: "*.sandbox.example.com"  # Covers all sandbox subdomains
```

This is a deploy-time config change, not a code change. The existing realm configmap template already appends `/*` to hostnames for redirect URIs.

---

## Files Summary

**grl-frontend-web-ui** (new/modified):
- `docker-entrypoint.sh` (new)
- `nginx.conf` (new)
- `Dockerfile` (modify)
- `public/config.json` (new, dev only)
- `src/config/loadRuntimeConfig.ts` (new)
- `src/index.tsx` (modify)
- `src/auth/OidcAuthProvider.tsx` (new, replaces auth0/Auth0ProviderWithNavigate.tsx)
- `src/auth/ApiKeyAuthProvider.tsx` (new, replaces auth0/AnonymousAuth0Provider.tsx)
- `src/auth/useAuthContext.ts` (new)
- `src/auth/AuthenticationGuard.tsx` (rewrite)
- `src/auth/LoginButton.tsx` (modify)
- `src/auth/UserMenu.tsx` (modify)
- `src/api/useHeaders.ts` (modify)
- `src/App.tsx` (modify)
- `src/SharedLayout.tsx` (modify)
- `src/globalAppConfig.ts` (modify)
- `src/__tests__/AppContext.test.tsx` (modify)
- `src/form/__tests__/ModelDialog.test.tsx` (modify)
- Delete: `src/auth0/` directory

**GraphSandbox**:
- `builder/k8s/chart/templates/frontend.yaml` (modify)
- `builder/k8s/chart/values.yaml` (modify)
- `shared/src/helm.ts` (modify)
- `builder/src/deployToK8s.ts` (modify)

---

## Verification

1. `npm test` in grl-frontend-web-ui passes
2. `helm lint` passes for both sandbox-generator and per-sandbox charts
3. Local dev with `public/config.json` pointing to dev Keycloak: login redirects to Keycloak, callback works, user menu shows name, logout works
4. Local dev with empty OIDC config: only API key login shown, API key auth works
5. K8s deploy: sandbox frontend inherits Keycloak config, OIDC login works
6. Downloaded chart without OIDC config: API-key-only mode works

## Key Design Decisions

- **No `audience` field**: Auth0-specific concept. Standard OIDC doesn't require it. Can be added later if a specific provider needs it.
- **`react-oidc-context` handles callback on current page**: The SPA's catch-all nginx route (`try_files → /index.html`) serves the callback URL. `oidc-client-ts` reads `code` and `state` from URL params automatically. No dedicated callback route needed.
- **Silent token refresh**: `oidc-client-ts` handles this via `automaticSilentRenew`. Keycloak supports refresh tokens for public clients.
- **No backend JWT validation**: Graph-server continues using API keys. The frontend auth gates the UI only. JWT validation on the backend is a separate future task.
