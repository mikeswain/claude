# Documentation Cleanup & Standardisation Plan

## Context

The GRL repos have accumulated ~260 markdown files / ~100K lines across Ontology-Manager and
FHIR-Message-Mapper-Service. Most are stale AI-generated specs, point-in-time test reports,
bug fix journals, and research docs that don't belong in source control. GraphSandbox is clean.

This plan follows the recommendations in `/home/mike/grl/documentation-report.md`, with user
decisions on handling: tag before deleting, push security/research/presentations to Confluence,
delete everything else that doesn't fit the new standard structure.

## Target Documentation Structure (per repo)

```
README.md                  # Developer-facing: what it is, how to build/test, links to docs/
docs/
├── quick-start.md         # Minimal install: K8s (primary), then Docker. Links to deeper docs
├── k8s-deployment.md      # All Helm values, defaults, configuration reference
├── features.md            # Product features, design principles, architecture decisions
├── api.md                 # Auth guide, endpoint table with links to full descriptions
└── user.md                # UI guide, task-oriented use cases with screenshots/examples
```

- **README.md** — Developer README: brief project description, how to build, how to run tests,
  tech stack, links to docs/ (quick-start, k8s-deployment, api, user). Not a product page —
  aimed at developers who need to clone and contribute.
- **quick-start.md** — Get running in 5 mins. K8s first (helm install), Docker second.
  Links to k8s-deployment.md for full config, api.md for endpoints, user.md for usage
- **k8s-deployment.md** — Complete Helm values reference table (value, type, default, description),
  prerequisites, upgrade notes, troubleshooting
- **features.md** — Product features and design principles. Condensed from the existing feature specs,
  design docs, and architecture decisions. More verbose than other docs — captures the "what and why"
  of key features, design patterns, supported standards (OWL constructs, FHIR profiles, etc.),
  and architectural choices. Linked from README and quick-start.
- **api.md** — Authentication (how to get a token), endpoint summary table (method, path, description),
  then full description of each endpoint with request/response bodies, status codes, examples
- **user.md** — Task-oriented UI guide: common workflows, use cases, screenshots/descriptions

All docs link to each other. README links to all five. quick-start links to k8s-deployment, features, api, user.

## Execution Steps

### Phase 1: Preserve history

1. Tag both repos at current HEAD: `docs-archive-2026-03`
2. Tag message: "Documentation archive before cleanup — see this tag for removed docs"

### Phase 2: Export to Confluence (security + research + presentations)

Create a script that reads the following markdown files and pushes them to Confluence via REST API:

**FHIR-Message-Mapper-Service** (~17 files):
- `docs/security/` — NIST assessment, HISO compliance, code review, remediation, CORS, input validation, error handling (~8 files)
- `docs/research/` — NZ healthcare challenges, ontology patterns, OWL patterns (~5 files)
- `docs/presentations/` — Semantic web presentation, healthcare use cases, SPARQL demos (~4 files)

**Ontology-Manager** (~1 file):
- `docs/design/pluggable-style-guides.md` + `docs/documentation/gistStyleGuide.md` (reference material)

Script approach: iterate files, convert markdown to Confluence storage format (XHTML), POST to
`/rest/api/content` under a "GRL Documentation Archive" space. User provides Confluence base URL,
space key, and API token.

### Phase 3: Delete docs that don't belong in git

**Ontology-Manager — DELETE** (~80 files):
- `docs/features/` — All 48 P0-P5 feature specs
- `docs/plans/` — All 5 dated design plans
- `docs/design/` — All 3 design docs (collaborative editing, style guides)
- `docs/bug-reports/` — All 6 bug fix journals
- `docs/backlog/` — All 5 backlog items
- `docs/temp/` — oxigraph evaluation
- `docs/refactoring/` — postgres-to-tdb2 migration
- `docs/documentation/` — gist style guide (after Confluence export)
- `backend/TEST_SUMMARY.md`, `backend/src/**/TEST_*.md`, `backend/src/**/*_README.md` — test docs

**FHIR-Message-Mapper-Service — DELETE** (~130 files):
- `docs/features/` — All 28 feature specs
- `docs/bugs/` — All 16 bug fix journals
- `docs/testing/` — All point-in-time test reports (~20 files), keep only: README.md, unit-tests.md, integration-tests.md, e2e-tests.md, performance-tests.md, infrastructure-setup.md, maven-profiles.md, frontend-tests.md
- `docs/backlog/` — All 12 roadmap/backlog docs
- `docs/api/` — Delete duplicates: API-REMEDIATION-ANALYSIS, API-DOCUMENTATION-UPDATE-PLAN, API-DESIGN-REVIEW, API-COMPARISON, API-SUMMARY (keep API-DOCUMENTATION.md as source for new api.md)
- `docs/security/` — All 8 files (after Confluence export)
- `docs/research/` — All 5 files (after Confluence export)
- `docs/presentations/` — All 4 files (after Confluence export)
- `docs/refactoring/` — Both work logs
- `docs/design/` — PHASE0-CARML-SPIKE-REPORT and other spike docs
- `docs/patterns/` — ENVIRONMENTS-PATTERN, HELP-CENTRE-PATTERN
- `docs/oracle/` — DECLARATIVE-API-GENERATION
- `docs/temp/` — oxigraph evaluation
- `docs/bug-reports/` — skipped-tests-analysis
- `docs/decisions/` — All 3 ADRs (content folded into features.md first)
- `test-data/generated-fhir/*.md` — All test execution reports
- `designer-ui/docs/` — DEAD-CODE-AUDIT, UNUSED-CLIENT-METHODS (keep API-CONTRACT.md)
- `.claude/api-doc-update-summary.md` — work log

### Phase 4: Write new standard docs

#### Ontology-Manager

| File | Source material | Notes |
|------|----------------|-------|
| `README.md` | Rewrite existing | Brief: what it is, build instructions, test commands, tech stack, links to docs/ |
| `docs/quick-start.md` | Existing `docs/quick-start.md` | Update for current setup, K8s primary, Docker secondary |
| `docs/k8s-deployment.md` | Existing `docs/k8s-deployment.md` + Helm values | Full values reference table |
| `docs/features.md` | Condense from `docs/features/`, `docs/design/`, `docs/backlog/` | OWL2 support matrix, design principles, architecture decisions, key capabilities |
| `docs/api.md` | Existing `docs/api/README.md` (76 endpoints) | Auth guide + endpoint table + full descriptions |
| `docs/user.md` | New — derive from UI code | Task-oriented UI guide |

#### FHIR-Message-Mapper-Service

| File | Source material | Notes |
|------|----------------|-------|
| `README.md` | Rewrite existing | Brief: what it is, build instructions, test commands, tech stack, links to docs/ |
| `docs/quick-start.md` | Existing deployment docs | Consolidate from 5+ deployment docs into one |
| `docs/k8s-deployment.md` | Existing K8S-DEPLOYMENT-GUIDE.md + Helm values | Full values reference table |
| `docs/features.md` | Condense from `docs/features/`, `docs/design/`, `docs/decisions/` | FHIR profiles, mapping pipeline, data lifecycle, design principles, architecture |
| `docs/api.md` | Existing API-DOCUMENTATION.md (3805 lines) | Restructure: auth → summary table → full descriptions |
| `docs/user.md` | Existing designer-ui docs + UI code | Task-oriented UI guide |

#### GraphSandbox

| File | Source material | Notes |
|------|----------------|-------|
| `README.md` | Rewrite existing | Brief: what it is, build instructions, test commands, tech stack, links to docs/ |
| `docs/quick-start.md` | Existing `docs/quick-start.md` | Already good — update links to match new structure |
| `docs/k8s-deployment.md` | Existing `docs/k8s-deployment.md` | Already good — verify completeness |
| `docs/features.md` | New — derive from code + builder/README.md | Sandbox generation, template system, multi-tenant architecture |
| `docs/api.md` | New — derive from backend routes | Auth guide + endpoint table + full descriptions |
| `docs/user.md` | New — derive from UI code | Task-oriented: creating sandboxes, managing templates, etc. |

#### Housekeeping (all repos)

- Update CLAUDE.md files to remove references to deleted doc paths
- Fold `docs/owl-standards-support.md` (OM) into `features.md` as OWL2 support matrix
- Fold FHIR `docs/decisions/` ADR content into `features.md`, then delete originals
- Confluence export script is a one-off — run, verify, discard

#### What stays beyond the standard 5

- **FHIR**: `docs/decisions/` ADR content folded into features.md, then originals deleted
- **FHIR**: `docs/testing/` (guides only, not reports) — linked from README under "Contributing"
- **OM**: `docs/testing/` (guides only) — same
- **Both**: `CLAUDE.md` files — untouched
- **Both**: Component READMEs (backend/, frontend/) — keep if useful

### Phase 5: Cleanup empty directories

Remove any now-empty `docs/` subdirectories after deletion.

## Verification

1. Both repos tagged `docs-archive-2026-03` before any deletions
2. Confluence export script run and pages verified in Confluence
3. All remaining docs follow the standard structure
4. All cross-links work (README → quick-start → k8s-deployment, api, user)
5. `docs/decisions/` preserved in FHIR
6. `docs/testing/` guides (not reports) preserved in both
7. No broken links in READMEs
8. Helm values tables in k8s-deployment.md match actual chart values
9. API docs match actual endpoints (cross-check with running service or code)
