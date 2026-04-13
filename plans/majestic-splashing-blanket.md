# Combined GRL Platform Installer

## Context

Currently, installing the full GRL stack requires 5+ manual `helm install` commands across
multiple namespaces, each with overlapping `--set` flags. The quick-start docs walk through
this step-by-step, but it's error-prone and slow. We want a single `curl ... | bash` (or
`./grl-install.sh`) that stands up the entire platform + all products with minimal input.

## Approach

A **single self-contained bash script** (`scripts/grl-install.sh`) that:

1. Prompts for only what it must: **domain** and **GitHub PAT** (with `read:packages` scope)
2. Auto-generates Keycloak passwords (prints them at the end)
3. Installs Gateway API CRDs
4. Installs grl-platform from its published Helm chart
5. Installs each enabled consumer product from their published Helm charts
6. Prints a summary with URLs and credentials

### Distribution

Hosted on the grl-platform GitHub Pages site (same place charts are published).
The repo is public, and the script is safe to expose — it requires a GH PAT
(and later a license key) to do anything useful.

```bash
# One-liner
curl -fsSL https://graph-research-labs.github.io/grl-platform/install.sh | bash

# Or download-and-run (supports flags for non-interactive use)
./install.sh --host staging.dev --gh-token ghp_xxx
```

CI publishes it alongside the chart index — see Files to Create/Modify below.

## Script Design

### Input Handling

**Interactive mode** (stdin is a tty and required flags missing):
```
GRL Platform Installer
======================
Domain (e.g. staging.dev): ___
GitHub username: ___
GitHub PAT (read:packages): ___
```

**Flag mode** (CI / non-interactive):
```
--host <domain>              Root domain (required)
--gh-username <user>         GHCR username (or GH_USERNAME env)
--gh-token <pat>             GHCR PAT (or GH_TOKEN env)
--kc-admin-password <pw>     Keycloak admin password (auto-generated if omitted)
--kc-db-password <pw>        Keycloak DB password (auto-generated if omitted)
--products <csv>             Comma-separated list (default: all)
                             Valid: ontology-manager, graphsandbox, fhir
--channel <staging|prod>     Chart channel (default: staging)
--tls                        Run mkcert TLS setup (requires mkcert on PATH)
--scheme <https|http>        Keycloak gateway scheme (default: https)
--dry-run                    Print commands without executing
--set <key=val>              Helm --set for platform (no prefix)
--set <component:key=val>    Helm --set routed to a specific component
--values <file>              Helm -f for platform (no prefix)
--values <component:file>    Helm -f routed to a specific component
--help                       Usage info
```

**Per-component overrides** use a `component:` prefix to route `--set` and `--values`
to the correct `helm upgrade --install` call. Unprefixed values go to grl-platform.

```bash
# Example: custom platform + component overrides
./install.sh --host staging.dev --gh-token ghp_xxx \
  --set keycloak.replicas=2 \
  --set fhir:mapper.resources.limits.memory=4Gi \
  --set graphsandbox:keygen.enabled=false \
  --values ontology-manager:my-om-values.yaml
```

Valid component prefixes: `ontology-manager`, `graphsandbox`, `fhir` (must match
product registry names).

Implementation: accumulate into associative arrays during arg parsing:
```bash
declare -A COMPONENT_SETS COMPONENT_VALUES
# --set fhir:foo=bar  →  COMPONENT_SETS[fhir]+=" --set foo=bar"
# --set foo=bar       →  PLATFORM_SETS+=" --set foo=bar"
```

Passwords auto-generated via `openssl rand -base64 24` when not provided.

### Prerequisites Check

Fail fast with clear messages:
- `helm` >= 3.12
- `kubectl` connected to a cluster
- `mkcert` (only if `--tls`)

### Product Registry

```bash
# name|release|namespace|repo-url-suffix|chart-name|host-prefix
PRODUCTS=(
  "ontology-manager|ontology-manager|ontology-manager|Ontology-Manager|ontology-manager|om"
  "graphsandbox|graphsandbox|graphsandbox|GraphSandbox|sandbox-generator|gs"
  "fhir|fhir|fhir|FHIR-Message-Mapper-Service|fhir|fhir"
)
```

### Install Sequence

```
1. kubectl apply Gateway API CRDs (v1.2.1) — idempotent
2. helm repo add + update for platform + each enabled consumer
3. helm upgrade --install grl-platform (--wait --timeout 5m)
4. [optional] mkcert TLS setup (if --tls)
5. For each enabled consumer:
     helm upgrade --install <release> <repo>/<chart> \
       --namespace <ns> --create-namespace \
       --set global.host=<prefix>.<domain> \
       --set keycloak.enabled=true \
       ${COMPONENT_SETS[<name>]} ${COMPONENT_VALUES[<name>]} \
       --wait --timeout 3m
6. Print summary
```

`helm upgrade --install` makes the script fully idempotent — rerunning upgrades
existing releases.

### Idempotency on Re-run

```bash
if helm status grl-platform -n grl-platform &>/dev/null; then
  # Existing install — add --reuse-values so passwords aren't lost
  # Explicit --set flags still override
  REUSE="--reuse-values"
fi
```

### TLS Setup (--tls flag)

Inlined mkcert logic (from dev-tls.sh) — ~10 lines:
```bash
mkcert -install
mkcert -cert-file /tmp/tls.crt -key-file /tmp/tls.key "*.${HOST}" "${HOST}"
kubectl -n nginx-gateway create secret tls staging-tls \
  --cert=/tmp/tls.crt --key=/tmp/tls.key \
  --dry-run=client -o yaml | kubectl apply -f -
rm /tmp/tls.crt /tmp/tls.key
```

### Summary Output

```
GRL Platform — Install Complete
================================
  Keycloak:          https://keycloak.staging.dev
  Oxigraph:          https://graph.staging.dev
  Ontology Manager:  https://om.staging.dev
  GraphSandbox:      https://gs.staging.dev
  FHIR Mapper:       https://fhir.staging.dev

  Keycloak Admin:    admin / <generated-password>

  Keycloak DB password has been stored in the Helm release.
  To retrieve later: helm get values grl-platform -n grl-platform
```

### Error Handling

- `set -euo pipefail`
- Trap on EXIT to print "Installation failed at step X. Re-run to resume."
- No rollback on partial failure — re-run is idempotent
- Each step prints a status line: `[1/6] Installing Gateway API CRDs...`

## Files to Create/Modify

| File | Action | Description |
|------|--------|-------------|
| `scripts/grl-install.sh` | **Create** | The installer script |
| `.github/workflows/pages.yml` | **Update** | Copy `scripts/grl-install.sh` to gh-pages root as `install.sh` |
| `docs/quick-start.md` | **Update** | Add one-liner install at the top |

### CI Change (`.github/workflows/pages.yml`)

Add after the "Publish charts" step, before "Commit & push":

```yaml
- name: Copy installer to gh-pages root
  run: cp scripts/grl-install.sh gh-pages/install.sh
```

This publishes it at `https://graph-research-labs.github.io/grl-platform/install.sh`
on every push to main or develop (same trigger as chart publishing).
The `paths:` filter needs `scripts/grl-install.sh` added so changes to the
installer also trigger a publish.

## Verification

1. **Dry run**: `./grl-install.sh --host test.dev --gh-token fake --dry-run` — prints all commands without executing
2. **Fresh install on k3d**: Create clean cluster, run installer, verify all pods Running
3. **Re-run (idempotent)**: Run again — should upgrade without errors, passwords preserved
4. **Subset install**: `--products ontology-manager,fhir` — only installs those two
5. **Helm unit tests**: Existing chart tests still pass (no chart changes)
