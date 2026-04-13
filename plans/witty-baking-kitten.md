# Move Keycloak Realm Config to keycloak-config-cli Job

## Context

The `--import-realm` built-in importer silently ignores `organizations` and only runs once (SKIP strategy). Rather than layering workarounds (`--override=true`, separate curl Jobs), move all entity configuration to a keycloak-config-cli Job. This is the same tool already used by the `keycloak-client` subchart (`adorsys/keycloak-config-cli:6.5.0-26.5.4`) — it uses the Admin API and handles idempotent upserts.

## Changes

### 1. Strip realm-export.json to bootstrap-only

**File:** `k8s/keycloak/templates/keycloak-realm-configmap.yaml`

Keep only the bare realm shell:
```json
{
  "realm": "grl-platform",
  "enabled": true,
  "sslRequired": "external",
  "organizationsEnabled": true,
  "registrationAllowed": false,
  "loginWithEmailAllowed": true,
  "duplicateEmailsAllowed": false,
  "resetPasswordAllowed": true,
  "editUsernameAllowed": false,
  "bruteForceProtected": true,
  "accessTokenLifespan": ...,
  "ssoSessionIdleTimeout": ...,
  "ssoSessionMaxLifespan": ...
}
```

Remove: `roles`, `clients`, `scopeMappings`, `users`.

### 2. Remove `--override=true` from deployment

**File:** `k8s/keycloak/templates/keycloak-deployment.yaml`

Remove `--override=true` from args — bootstrap only needs to create the realm once.

### 3. Create config-cli Job + ConfigMap

**New file:** `k8s/keycloak/templates/realm-config-job.yaml`

Post-install/post-upgrade hook Job using `adorsys/keycloak-config-cli`, same pattern as `keycloak-client/templates/job.yaml`. Manages:

- **Roles**: admin, editor, viewer
- **Clients**: grl-platform-service (with secret from `keycloak.serviceAccountSecret` helper)
- **Scope mappings**: grl-platform-service → viewer
- **Users**: seed users from `.Values.realm.seedUsers` + service account user
- **Organizations**: "Graph Research Labs" with seed users as members

The ConfigMap contains the realm JSON for config-cli to import. Uses `IMPORT_MANAGED_CLIENT=no-delete` to avoid removing consumer-registered clients.

Config-cli image/tag and resource values reused from new `configCli` section in `values.yaml` (same defaults as keycloak-client).

### 4. Delete curl-based org-seed-job.yaml

**Delete:** `k8s/keycloak/templates/org-seed-job.yaml`

### 5. Add configCli values

**File:** `k8s/keycloak/values.yaml`

```yaml
configCli:
  image:
    repository: adorsys/keycloak-config-cli
    tag: "6.5.0-26.5.4"
  backoffLimit: 5
  resources:
    requests:
      cpu: 100m
      memory: 128Mi
    limits:
      cpu: 250m
      memory: 256Mi
```

### 6. Update tests

**File:** `k8s/keycloak/tests/realm_configmap_test.yaml`

- Remove tests for roles, clients, users, service account, scope mappings, organizations (no longer in this template)
- Keep: basic realm settings, token lifespans, organizationsEnabled

**New file:** `k8s/keycloak/tests/realm_config_job_test.yaml`

- Job is post-install/post-upgrade hook
- ConfigMap contains realm JSON with roles, clients, users, organizations
- Seed users appear in both users and organization members
- Service account client present with correct scopes
- Organization "Graph Research Labs" present

## Files

| File | Action |
|------|--------|
| `k8s/keycloak/templates/keycloak-realm-configmap.yaml` | Edit (strip to bootstrap) |
| `k8s/keycloak/templates/keycloak-deployment.yaml` | Edit (remove --override=true) |
| `k8s/keycloak/templates/realm-config-job.yaml` | Create |
| `k8s/keycloak/templates/org-seed-job.yaml` | Delete |
| `k8s/keycloak/values.yaml` | Edit (add configCli section) |
| `k8s/keycloak/tests/realm_configmap_test.yaml` | Edit (reduce scope) |
| `k8s/keycloak/tests/realm_config_job_test.yaml` | Create |

## Verification

```bash
helm unittest k8s/keycloak
helm lint k8s/keycloak
helm template test k8s/keycloak --set gateway.host=kc.example.com \
  --set secrets.kcDbPassword=x --set secrets.kcAdminPassword=x \
  | grep -A5 'realm-config'
```
