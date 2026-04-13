# Licensing Support Plan

## Context

GRL platform needs per-component licensing with usage metering, air-gap support, and UI display across all three consumer products. GraphSandbox already has a working keygen.sh integration (`shared/src/license.ts`, `LicenseSummary.tsx`). Ontology-Manager and FHIR-Mapper have none. The license server will NOT live inside grl-platform — it's an external service.

## Solution Recommendation: keygen.sh

**Continue with keygen.sh.** GraphSandbox already integrates it, and it covers all requirements:

| Requirement | keygen.sh | Cryptlex | Custom JWT |
|---|---|---|---|
| Already integrated | Yes (GraphSandbox) | No | No |
| Usage metering | Native (`uses`/`maxUses`) | Yes | Must build |
| Air-gap (offline) | Ed25519 signed license files | Offline activation | Trivial but must build everything |
| Self-hosting | keygen-ee Docker (same API) | Cloud only | N/A |
| Per-component | Products + policies + entitlements | Features + licenses | Must build |
| Cost | Free tier: 5 licenses / 25 machines | $499+/mo | Dev time |

**Deployment recommendation:** Start with keygen.sh cloud (free tier covers dev). Document self-hosted keygen-ee as an option for enterprise/air-gapped customers. The API is identical — switching is a URL change.

## Air-Gap Strategy

Three configurable modes (`license.mode`):

1. **connected** (default) — periodic HTTP validation (60-min interval, as GraphSandbox does today). Usage reported via API.
2. **semi-connected** — validated at startup via API. Usage buffered locally, synced when connected. Configurable grace period (default 72h) for connectivity loss.
3. **air-gapped** — signed license file (Ed25519) mounted as K8s Secret. Validated cryptographically, no network calls. Usage written to PVC as JSON, extractable for manual submission. License renewal via file replacement.

## Architecture

### Validation: application-level, not gateway-level
Each consumer has different license requirements (different product, entitlements). The gateway handles authentication (oauth2-proxy); licensing is per-component authorization that belongs in the app.

### License claims in JWTs
GraphSandbox already reads `license_expires_at`, `license_issued_at`, `license_key`, `license_name` from JWT custom claims (`User.ts:2-7`). These flow via Keycloak user attributes → protocol mappers. Extend this to all consumers by adding a `license` client scope to the Keycloak realm config.

### Usage reporting
- **Connected:** in-memory counter, periodic flush (every 5 min) to keygen.sh `POST .../licenses/{id}/actions/increment-usage`
- **Air-gapped:** same accumulation, flushed to local JSON file on PVC

---

## Operational Workflows

### Initial License Activation at Deploy Time

**Connected / semi-connected:**
1. Vendor (us) creates a license in keygen.sh for the customer, scoped to the purchased products/entitlements
2. Vendor provides the customer a license key (string)
3. Customer sets the key in their Helm values at deploy time:
   ```yaml
   license:
     key: "XXXX-XXXX-XXXX-XXXX"
   ```
4. On first pod startup, the license client validates the key against keygen.sh API and activates the machine
5. App starts normally if valid; fails fast with clear error if invalid/expired

**Air-gapped:**
1. Vendor creates the license in keygen.sh and exports a signed license file (Ed25519-signed JSON)
2. Vendor provides the file to the customer out-of-band (email, secure transfer, USB)
3. Customer creates a K8s Secret from the file:
   ```bash
   kubectl create secret generic grl-license --from-file=license.lic=./license.lic -n <namespace>
   ```
4. Customer references the secret in Helm values:
   ```yaml
   license:
     mode: air-gapped
     licenseSecret: grl-license
   ```
5. On startup, the license client reads the mounted file and validates the Ed25519 signature against the embedded public key — no network call

### Updating a License

**Connected — remote update (no customer action):**
1. Vendor updates the license in keygen.sh (extend expiry, add entitlements, change product scope)
2. Next periodic validation (within 60 min) picks up the new license state automatically
3. No Helm upgrade, no pod restart required
4. Frontend `LicenseSummary` reflects updated expiry/status on next JWT refresh

**Connected — adding/removing a component:**
1. Vendor adds or removes a product entitlement on the license in keygen.sh
2. The license client in each consumer validates its own product scope — if the entitlement is removed, that consumer enters enforcement mode (warn → block)
3. To add a newly licensed component: customer deploys it via Helm with the same license key; it self-validates on startup

**Semi-connected — same as connected**, but changes are picked up at next startup or connectivity window rather than on a periodic timer.

**Air-gapped — license file replacement:**
1. Vendor generates a new signed license file with updated terms
2. Vendor delivers the file to the customer out-of-band
3. Customer updates the K8s Secret:
   ```bash
   kubectl create secret generic grl-license --from-file=license.lic=./new-license.lic \
     -n <namespace> --dry-run=client -o yaml | kubectl apply -f -
   ```
4. Reloader detects the Secret change and triggers a rolling restart
5. Pods pick up the new license file on startup

### Revoking / Suspending a License

**Connected:**
1. Vendor suspends the license in keygen.sh
2. Next periodic validation returns `SUSPENDED` status
3. Consumer enters enforcement mode — write operations blocked, UI shows warning
4. To reinstate: vendor unsuspends in keygen.sh; next validation restores access

**Air-gapped:**
- No remote revocation possible (by design — air-gapped means no call home)
- Licenses are time-bound; revocation happens by not providing a renewal file
- For urgent revocation: customer must manually delete the license Secret

### Transitioning from Air-Gapped to Connected

1. Customer opens network access from the cluster to keygen.sh (or self-hosted keygen-ee)
2. Customer updates Helm values:
   ```yaml
   license:
     mode: connected  # was: air-gapped
     key: "XXXX-XXXX-XXXX-XXXX"
   ```
3. `helm upgrade` triggers restart; license client switches to HTTP validation
4. License file Secret can be removed (no longer needed)

### Usage Data Collection (Air-Gapped)

1. Each consumer writes usage data to a JSON file on its PVC:
   ```json
   {"product": "ontology-manager", "period": "2026-04-14", "counts": {"mutations": 142, "sparql-queries": 891}}
   ```
2. Customer extracts usage files periodically (e.g., monthly):
   ```bash
   kubectl cp <namespace>/<pod>:/data/usage/ ./usage-export/
   ```
3. Customer sends the files to vendor (email, portal upload)
4. Vendor imports usage data into keygen.sh for reconciliation/billing

---

## Documentation Plan

### Customer Documentation (on-prem deployment guide)

Target audience: customer IT/DevOps teams deploying GRL products in their own infrastructure.

**`docs/licensing.md`** — single document covering:

1. **Overview** — what licensing controls (per-component access + usage metering), what modes are available
2. **Initial setup**
   - Connected: setting `license.key` in Helm values, verifying activation
   - Air-gapped: receiving the license file, creating the K8s Secret, setting `license.mode: air-gapped`
3. **License renewal**
   - Connected: automatic (no action needed), or how to verify updated terms
   - Air-gapped: updating the K8s Secret with the new license file
4. **Adding/removing components** — deploy/remove the consumer chart; license scope controls access
5. **Usage reporting**
   - Connected: automatic, no action needed
   - Air-gapped: how to extract and submit usage files
6. **Transitioning between modes** — air-gapped → connected and vice versa
7. **Troubleshooting**
   - License validation failures (common status codes and what they mean)
   - Expired license behaviour (grace period, warn vs block)
   - How to check license status (kubectl logs, UI indicators)
8. **Network requirements** — outbound HTTPS to keygen.sh (or self-hosted URL), ports, domains to allowlist

### Vendor Documentation (internal operations guide)

Target audience: GRL team managing customer licenses.

**`docs/licensing-vendor.md`** — internal document covering:

1. **keygen.sh account structure** — account, products (one per consumer), policies (per licensing tier), entitlements
2. **Creating a new customer license**
   - Which product(s) and policy to assign
   - Setting expiry, maxUses, maxMachines
   - Generating the license key
3. **Exporting signed license files** — for air-gapped customers, how to generate and deliver the Ed25519-signed file
4. **Modifying a license** — extending expiry, changing entitlements, adjusting usage limits
5. **Suspending / revoking** — when and how, connected vs air-gapped implications
6. **Usage reconciliation** — importing air-gapped usage files, reviewing connected usage in keygen.sh dashboard
7. **Self-hosted keygen-ee** — deployment guide for customers who want to host their own license server (Docker image, config, same API)
8. **Pricing tier mapping** — how keygen.sh policies map to GRL product tiers (TBD with business)
9. **Trust model & circumvention boundaries** (see below)

### Trust Model & Circumvention Boundaries (vendor docs)

This is a high-trust, commercial-relationship licensing model — not DRM. The goal is honest-customer compliance tracking, not adversarial tamper-proofing. Customers have root access to their clusters; we don't try to prevent that.

**What the license system enforces:**
- Time-bound access — licenses expire, apps check expiry
- Component scope — each consumer validates its own product entitlement
- Usage metering — operations counted and reported (connected) or logged (air-gapped)
- Machine binding (optional) — keygen.sh can bind a license to a machine fingerprint

**What a customer with cluster admin access could do:**

| Circumvention | Mode | Difficulty | Mitigation |
|---|---|---|---|
| Disable license check env vars / remove ConfigMap | All | Trivial | App fails to start without valid config (fail-closed) |
| Set `license.enabled: false` in Helm values | All | Trivial | Only available if we expose this flag — **don't**; license check is always-on, not optional |
| Patch container image to skip validation | All | Moderate | Out of scope — same as pirating the software |
| Block outbound traffic to keygen.sh | Connected | Trivial | Grace period expires → enforcement kicks in |
| Replay a valid license response (MITM) | Connected | Moderate | keygen.sh responses include nonce + timestamp; client should reject stale responses |
| Copy license key to another cluster | Connected | Easy | Machine fingerprint binding (optional policy); or accept this as a trust model trade-off |
| Copy signed license file to another cluster | Air-gapped | Easy | Machine binding in the signed payload; or accept — file-based licenses are inherently portable |
| Edit usage data files before submission | Air-gapped | Trivial | **Accepted risk** — air-gapped usage is self-reported by design; contractual, not technical enforcement |
| Run beyond licensed user/seat count | All | Easy | Seat enforcement requires Keycloak user counting; possible future enhancement |

**Design stance:**
- **Fail-closed, not fail-open** — missing or invalid license blocks startup / write operations. A customer has to actively circumvent, not passively benefit from a misconfiguration.
- **No obfuscation** — license validation code is readable. We don't hide the logic; the protection is contractual + relationship-based.
- **Air-gapped usage is trust-based** — we acknowledge self-reported usage data can be edited. The license file controls the time window; usage data is for billing reconciliation under the contract.
- **Machine binding is optional** — for enterprise customers who want portability across clusters (DR failover, migration), strict machine binding is counterproductive. Enable it per-policy when warranted.
- **Grace periods are intentional** — a 72h grace period on connectivity loss is a feature, not a vulnerability. It prevents legitimate outages from blocking production workloads.

**Recommendations:**
- Keep `license.enabled` out of Helm values — the license client library is always active when the library is present. The only way to "disable" licensing is to not include the library dependency (i.e., build a custom image).
- For high-value enterprise accounts, enable machine fingerprint binding in the keygen.sh policy.
- For usage-metered contracts, prefer connected mode and make air-gapped a contractual discussion (shorter renewal cycles, manual audit rights).
- Log all license validation results (success and failure) to stdout — gives both us and the customer an audit trail.

---

## Phases

### Phase 0: Cleanup
Remove the old keygen placeholder from grl-platform.

- Delete `k8s/keygen/` directory (just `.gitkeep`)
- Remove `keygen: enabled: false` from `k8s/grl-platform/values.yaml:125-126`
- Clean references in `CLAUDE.md`, `README.md`, `k8s/README.md`, `docs/k8s-deployment.md`

### Phase 1: Platform shared infrastructure

**New `k8s/license-client/` subchart** (follows `nav-entry` pattern):
- ConfigMap `grl-platform-license` with Reflector annotations, containing:
  - `LICENSE_SERVER_URL`, `LICENSE_ACCOUNT`, `LICENSE_MODE`, `LICENSE_PUBLIC_KEY` (Ed25519 for air-gap)
- Values: `serverUrl`, `account`, `mode`, `publicKey`
- `_helpers.tpl`, `_validate.tpl`, helm unit tests

**Keycloak realm changes:**
- Add `license` client scope with protocol mappers for the four license claim fields to `k8s/keycloak/templates/realm-config-job.yaml`
- Update `k8s/keycloak-client/` subchart to include `license` in `defaultClientScopes` for all consumer clients

**Umbrella chart:**
- Add `license-client` dependency to `k8s/grl-platform/Chart.yaml` (condition: `license.enabled`)
- Add `license:` config block to `k8s/grl-platform/values.yaml`

**Files to modify:**
- `k8s/grl-platform/Chart.yaml`
- `k8s/grl-platform/values.yaml`
- `k8s/keycloak/templates/realm-config-job.yaml`
- `k8s/keycloak-client/templates/configmap.yaml`

**Files to create:**
- `k8s/license-client/Chart.yaml`
- `k8s/license-client/values.yaml`
- `k8s/license-client/templates/_helpers.tpl`
- `k8s/license-client/templates/_validate.tpl`
- `k8s/license-client/templates/configmap.yaml`
- `k8s/license-client/templates/NOTES.txt`
- `k8s/license-client/tests/`

### Phase 2: GraphSandbox alignment (reference implementation)

Align existing GraphSandbox integration with the new shared infra.

- Refactor `shared/src/license.ts` to read from shared env vars (`LICENSE_SERVER_URL`, `LICENSE_ACCOUNT`) with fallback to existing `KEYGEN_*` vars for backward compat
- Add air-gap mode: Ed25519 signature validation of mounted license file
- Add usage reporting: increment keygen.sh usage on BullMQ job completion in `builder/src/index.ts`
- Update helm chart to add `license-client` subchart dependency
- Update deployment template to `envFrom` the shared ConfigMap
- Move `keygen:` values to `license:` section

**Files to modify:**
- `~/grl/GraphSandbox/shared/src/license.ts`
- `~/grl/GraphSandbox/builder/src/index.ts`
- `~/grl/GraphSandbox/k8s/Chart.yaml`
- `~/grl/GraphSandbox/k8s/values.yaml`
- `~/grl/GraphSandbox/k8s/templates/` (deployment envFrom)

### Phase 3: Java license client library

Create `grl-license-client` Maven artifact (new repo or module — TBD):

- **`LicenseProperties`** — Spring Boot config properties (`grl.license.*`)
- **`LicenseAutoConfiguration`** — `@ConditionalOnProperty("grl.license.enabled")`
- **`LicenseService`** — HTTP validation (keygen.sh API), file-based validation (air-gap), result caching, periodic revalidation
- **`UsageMeterService`** — accumulate counts, periodic flush to keygen.sh or local file
- **`LicenseInterceptor`** — Spring `HandlerInterceptor`, rejects requests when license invalid/expired
- **`@Metered` annotation + AOP aspect** — declarative usage counting on service methods
- **Enforcement modes:** `warn` | `block` | `warn-then-block` (configurable grace period)
- Unit tests with WireMock

### Phase 4: FHIR-Mapper integration

- Add `grl-license-client` Maven dependency
- Add `license-client` subchart dependency to `k8s/fhir/Chart.yaml`
- Configure `LicenseInterceptor` in WebFlux filter chain (note: reactive — needs `Mono`/`Flux` aware AOP)
- Add `@Metered("fhir-message")` to `FhirStreamController.processStream()` — count = batch message count
- Add `@Metered("fhir-message")` to `FhirStreamController.processSingle()` — count = 1
- Add `LicenseSummary` component to `designer-ui/src/components/layout/GlobalAppBar.tsx` user dropdown
- Update frontend to read license claims from JWT (follow GraphSandbox `User.ts` pattern)

**Key files:**
- `~/grl/FHIR-Message-Mapper-Service/src/main/java/com/grl/mapper/controller/FhirStreamController.java`
- `~/grl/FHIR-Message-Mapper-Service/designer-ui/src/components/layout/GlobalAppBar.tsx`
- `~/grl/FHIR-Message-Mapper-Service/k8s/fhir/Chart.yaml`
- `~/grl/FHIR-Message-Mapper-Service/k8s/fhir/values.yaml`

### Phase 5: Ontology-Manager integration

- Add `grl-license-client` Maven dependency
- Add `license-client` subchart dependency to `k8s/ontology-manager/Chart.yaml`
- Configure `LicenseInterceptor` in `SecurityConfig` filter chain (standard Spring MVC)
- Add `@Metered` to mutating service methods — reuse pointcut patterns from existing `ProvenanceAspect.java` which already enumerates all mutating operations
- Add `LicenseSummary` component to frontend app bar/user dropdown
- Update frontend to read license claims from JWT

**Key files:**
- `~/grl/Ontology-Manager/backend/src/main/java/com/grl/ontology/config/SecurityConfig.java`
- `~/grl/Ontology-Manager/backend/src/main/java/com/grl/ontology/aspect/ProvenanceAspect.java` (pattern reference)
- `~/grl/Ontology-Manager/frontend/src/components/layout/NavTabBar.tsx`
- `~/grl/Ontology-Manager/k8s/ontology-manager/Chart.yaml`

### Phase 6: E2E testing & documentation

**Testing:**
- E2E tests in `grl-e2e` for license validation across all consumers
- Test air-gap mode with signed license files
- Test license expiry behavior (warn → block transition)
- Test usage reporting accuracy
- Test remote license update (modify in keygen.sh, verify consumer picks up change)
- Test component add/remove (deploy new consumer with existing license, verify entitlement check)

**Documentation:**
- `docs/licensing.md` — customer-facing on-prem deployment guide (see Documentation Plan above)
- `docs/licensing-vendor.md` — internal vendor operations guide (see Documentation Plan above)
- Update CLAUDE.md onboarding checklist with licensing section
- Update `docs/k8s-deployment.md` values table with license config
- Helm NOTES.txt for license-client subchart (post-install instructions)

## Metering Points Summary

| Consumer | What to meter | Where | Count |
|---|---|---|---|
| GraphSandbox | Sandbox builds | `builder/src/index.ts` BullMQ job completion | 1 per build |
| FHIR-Mapper | FHIR messages | `FhirStreamController.processStream()` | batch size |
| FHIR-Mapper | Single transforms | `FhirStreamController.processSingle()` | 1 |
| Ontology-Manager | Mutations | Service methods per `ProvenanceAspect` pointcuts | 1 per operation |
| Ontology-Manager | SPARQL queries | `DraftSparqlController` endpoints | 1 per query |

## Verification

- `helm unittest k8s/license-client` — subchart template tests
- `helm lint k8s/license-client` and `helm lint k8s/grl-platform` — chart validation
- Deploy to k3d, verify `grl-platform-license` ConfigMap is mirrored to consumer namespaces
- Verify license claims appear in JWT after Keycloak realm update
- GraphSandbox: start builder with valid/invalid/expired license keys, confirm behaviour
- FHIR-Mapper + Ontology-Manager: hit API endpoints, confirm license enforcement and usage increment
- Air-gap: mount signed license file, disable network, confirm validation passes
