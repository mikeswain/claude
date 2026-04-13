# Plan: Add Keycloak JWT Auth to FHIR Mapper + Use Shared Oxigraph

## Context

FHIR Mapper has a complete but disabled local auth system (`mapper.auth.enabled=false`).
The codebase already has an `AuthenticationProvider` interface with `jwt` provider planned,
config fields for `issuerUri`/`clientId`/`jwksUri` stubbed out, and a `AuthenticationFilter`
that extracts Bearer tokens. We need to:

1. Implement the JWT provider so Keycloak tokens are validated
2. Keep auth optional (default off) via a single `keycloak.enabled` Helm toggle
3. Switch to grl-platform shared Oxigraph (auto-discovered via Reflector)
4. Document how to curl the API with a service account token

**Decisions:** Resource-server only (no outbound Keycloak admin calls). Role mapping:
`admin`→ADMIN, everything else→USER. Project assignments not used in JWT mode.

## Changes

### 1. Implement `JwtAuthenticationProvider`

**New file:** `src/main/java/com/grl/mapper/auth/JwtAuthenticationProvider.java`

Implements `AuthenticationProvider`. Validates Keycloak-issued JWTs using nimbus-jose-jwt
(already a transitive dependency via Spring Boot).

```java
@Slf4j
@Component
@ConditionalOnProperty(name = "mapper.auth.provider", havingValue = "jwt")
public class JwtAuthenticationProvider implements AuthenticationProvider {
    private final JWSKeySelector<SecurityContext> keySelector;
    private final String expectedIssuer;

    // Constructor: build RemoteJWKSet from jwksUri (or derive from issuerUri + well-known)
    // Cache JWKS with 5-min TTL (NimbusJWKSet default)

    @Override
    public Mono<AuthSession> authenticate(String username, String password) {
        return Mono.error(new UnsupportedOperationException(
            "JWT provider does not support password auth — authenticate via Keycloak"));
    }

    @Override
    public Mono<AuthSession> validateSession(String token) {
        // 1. Parse and verify JWT signature via JWKS
        // 2. Check exp, iss claims
        // 3. Extract: sub, preferred_username, realm_access.roles
        // 4. Map roles: "admin" in roles → ADMIN, else → USER
        // 5. Return AuthSession(token, sub, username, role, exp, emptyList)
    }

    @Override
    public Mono<Void> invalidateSession(String token) {
        return Mono.empty(); // JWTs expire naturally
    }

    @Override
    public String getProviderType() { return "jwt"; }
}
```

**Key points:**
- Uses `com.nimbusds.jose.jwk.source.RemoteJWKSet` for JWKS fetching + caching
- Derives JWKS URI from `issuerUri + /protocol/openid-connect/certs` if `jwksUri` not set
- No Spring Security dependency needed — just nimbus-jose-jwt (already available)
- `assignedProjects` is empty list — all authenticated users access all projects in JWT mode

**Dependency:** Verify `nimbus-jose-jwt` is available transitively. If not, add to `pom.xml`:
```xml
<dependency>
    <groupId>com.nimbusds</groupId>
    <artifactId>nimbus-jose-jwt</artifactId>
</dependency>
```

### 2. Condition `AuthController` on local provider

**File:** `src/main/java/com/grl/mapper/auth/AuthController.java`

Add `@ConditionalOnProperty(name = "mapper.auth.provider", havingValue = "local", matchIfMissing = true)`

The login/logout/change-password endpoints only make sense for local auth. In JWT mode,
users authenticate at Keycloak directly.

### 3. Extend `AuthStatusController` response

**File:** `src/main/java/com/grl/mapper/auth/AuthStatusController.java`

When auth is enabled with JWT provider, include Keycloak discovery info so the designer
UI (or any client) knows where to authenticate:

```json
{
  "enabled": true,
  "provider": "jwt",
  "issuerUri": "https://keycloak.example.com/auth/realms/grl-platform",
  "clientId": "fhir-mapper"
}
```

When auth is disabled: `{"enabled": false}` (unchanged).
When local auth: `{"enabled": true, "provider": "local"}`.

Inject `MapperProperties` to read the provider config.

### 4. Add `keycloak-client` subchart dependency

**File:** `k8s/fhir/Chart.yaml`

```yaml
dependencies:
  - name: mapper
    version: 0.1.0
  - name: designer
    version: 0.1.0
  - name: keycloak-client
    version: "0.1.0"
    repository: "https://graph-research-labs.github.io/grl-platform/staging"
    condition: keycloak.enabled
```

### 5. Add umbrella chart helpers + keycloak env ConfigMap

**New file:** `k8s/fhir/templates/_helpers.tpl`

Discovery helpers (same pattern as Ontology-Manager):

```
fhir-mapper.host          — global.host fallback
fhir-mapper.oxigraphFqdn  — from mirrored grl-platform-oxigraph ConfigMap
fhir-mapper.keycloakIssuer — from mirrored grl-platform-keycloak ConfigMap (issuer field)
fhir-mapper.keycloakJwksUri — derived: internal FQDN + /auth/realms/.../certs
```

**New file:** `k8s/fhir/templates/configmap-keycloak-env.yaml`

Conditional on `keycloak.enabled`. Renders a ConfigMap with env overrides:
- `MAPPER_AUTH_ENABLED: "true"`
- `MAPPER_AUTH_PROVIDER: jwt`
- `MAPPER_AUTH_ISSUER_URI: <from mirrored ConfigMap — external issuer>`
- `MAPPER_AUTH_JWKS_URI: <from mirrored ConfigMap — internal FQDN + certs path>`
- `MAPPER_AUTH_CLIENT_ID: fhir-mapper`

### 6. Update mapper deployment for conditional envFrom

**File:** `k8s/fhir/charts/mapper/templates/deployment.yaml`

Add conditional envFrom for keycloak env. Since the env vars load in order, the
keycloak ConfigMap's `MAPPER_AUTH_ENABLED=true` overrides the base ConfigMap's `false`.

```yaml
envFrom:
  - configMapRef:
      name: {{ .Release.Name }}-mapper-env
  - secretRef:
      name: {{ .Release.Name }}-mapper-secrets
  {{- if .Values.global.keycloak }}
  {{- if .Values.global.keycloak.enabled }}
  - configMapRef:
      name: {{ .Release.Name }}-keycloak-env
  {{- end }}
  {{- end }}
```

### 7. Update umbrella values for Oxigraph discovery + keycloak toggle

**File:** `k8s/fhir/values.yaml`

```yaml
global:
  host: ""
  environment: latest
  keycloak:
    enabled: false    # flows to subchart conditions

keycloak:
  enabled: false      # controls keycloak-client subchart + keycloak env ConfigMap

keycloak-client:
  clientId: fhir-mapper
  name: FHIR Mapper

mapper:
  env:
    MAPPER_AUTH_ENABLED: "false"       # overridden by keycloak env ConfigMap when enabled
    MAPPER_AUTH_PROVIDER: "local"      # overridden to "jwt" when keycloak enabled
    # Oxigraph — remove hardcoded URLs, use discovery helper in configmap
```

For Oxigraph: remove the hardcoded `MAPPER_GRAPH_DB_ENDPOINT` URLs from the umbrella
values and instead set them in a new umbrella-level configmap template (or override in
the mapper subchart configmap). Simplest: keep them in the umbrella values but use the
helper to resolve the FQDN.

Actually, since the mapper subchart's configmap.yaml just renders `env` values from
`values.yaml`, the simplest approach is to set the Oxigraph URLs in the umbrella
values.yaml using plain values (not templates — values.yaml can't template). The
discovery helper can be used in a separate umbrella-level configmap that the deployment
also loads via envFrom, similar to the keycloak env approach.

**New file:** `k8s/fhir/templates/configmap-platform-env.yaml`

Always rendered. Sets platform-discovered env vars:
- `MAPPER_GRAPH_DB_ENDPOINT: <oxigraph FQDN>/store`
- `MAPPER_GRAPH_DB_UPDATE_ENDPOINT: <oxigraph FQDN>/update`

This overrides the defaults in the mapper subchart's configmap.

### 8. Update CI workflow

**File:** `.github/workflows/pages.yml`

Add grl-platform repo + dependency build step (same pattern as other consumer projects).

### 9. Update docs

**File:** `docs/deployment/K8S-DEPLOYMENT-GUIDE.md`

Major updates:
- Quick start with minimal `--set global.host=...` command
- Section on enabling Keycloak auth (`--set keycloak.enabled=true`)
- Service token instructions:

```bash
# Get a service account token from Keycloak
TOKEN=$(curl -s -X POST \
  "https://<keycloak-host>/auth/realms/grl-platform/protocol/openid-connect/token" \
  -d "grant_type=client_credentials" \
  -d "client_id=grl-platform-service" \
  -d "client_secret=$(kubectl get secret grl-platform-service-account \
       -o jsonpath='{.data.clientSecret}' | base64 -d)" \
  | jq -r .access_token)

# Call the API
curl -H "Authorization: Bearer $TOKEN" https://fhir.example.com/projects
```

- Remove manual regcred step (mirrored by Reflector)
- Update config reference table

### 10. Tests

**Java:**
- `JwtAuthenticationProviderTest` — valid/expired/invalid-sig/missing-claims JWTs
- Use nimbus-jose-jwt to create test JWTs with a local RSA key pair

**Helm:**
- Conditional rendering of keycloak-env ConfigMap
- Platform-env ConfigMap with Oxigraph discovery
- keycloak-client subchart not rendered when `keycloak.enabled=false`

## Files Summary

| File | Action | Purpose |
|------|--------|---------|
| `src/.../auth/JwtAuthenticationProvider.java` | New | JWT token validation |
| `src/.../auth/AuthController.java` | Modify | Condition on `provider=local` |
| `src/.../auth/AuthStatusController.java` | Modify | Include provider/issuer in response |
| `pom.xml` | Modify | Verify nimbus-jose-jwt dep |
| `k8s/fhir/Chart.yaml` | Modify | Add keycloak-client dependency |
| `k8s/fhir/values.yaml` | Modify | Add keycloak toggle, global.host, remove hardcoded Oxigraph |
| `k8s/fhir/templates/_helpers.tpl` | New | Discovery helpers |
| `k8s/fhir/templates/configmap-keycloak-env.yaml` | New | Conditional Keycloak env vars |
| `k8s/fhir/templates/configmap-platform-env.yaml` | New | Oxigraph discovery env vars |
| `k8s/fhir/charts/mapper/templates/deployment.yaml` | Modify | Additional envFrom refs |
| `k8s/fhir/charts/mapper/values.yaml` | Modify | Remove hardcoded Oxigraph URLs |
| `.github/workflows/pages.yml` | Modify | Add grl-platform repo |
| `docs/deployment/K8S-DEPLOYMENT-GUIDE.md` | Modify | Auth docs, service token, minimal install |
| `src/test/.../auth/JwtAuthenticationProviderTest.java` | New | Unit tests |

## Deploy Commands

```bash
# Without auth (default)
helm install fhir fhir-platform/fhir -n fhir --create-namespace \
  --set global.host=fhir.example.com

# With Keycloak auth
helm install fhir fhir-platform/fhir -n fhir --create-namespace \
  --set global.host=fhir.example.com \
  --set keycloak.enabled=true
```

## Verification

```bash
# Deploy without auth
helm install fhir ./k8s/fhir -n fhir --create-namespace --set global.host=fhir.local
curl http://fhir.local/fhir/health          # 200
curl http://fhir.local/projects              # 200 (no auth)
curl http://fhir.local/auth/status           # {"enabled":false}

# Deploy with auth
helm upgrade fhir ./k8s/fhir -n fhir \
  --set global.host=fhir.local --set keycloak.enabled=true
curl http://fhir.local/fhir/health          # 200 (public path)
curl http://fhir.local/projects              # 401
curl http://fhir.local/auth/status           # {"enabled":true,"provider":"jwt","issuerUri":"..."}

# Service token
TOKEN=$(curl -s -X POST ...)
curl -H "Authorization: Bearer $TOKEN" http://fhir.local/projects   # 200

# keycloak-client Job
kubectl get jobs -n fhir

# Java tests
mvn test -Dtest="JwtAuthenticationProviderTest"

# Helm
helm unittest k8s/fhir
helm lint k8s/fhir --set global.host=x
```
