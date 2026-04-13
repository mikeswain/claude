# GRL Platform Home Page + Nav Entry Enhancements

## Context

The platform needs a static landing page on the bare `global.host` domain (e.g. `staging.dev`) showing cards for all installed products plus greyed-out placeholders for future ones. This is driven by a mockup from leadership showing a 3x2 card grid with icons, titles, and descriptions.

The bare domain is currently unclaimed — GraphSandbox uses its own hostname (e.g. `gs.staging.dev`) via `gateway.host` at install time.

## Part 1: Nav Entry Schema Changes

**Files:** `k8s/nav-entry/values.yaml`, `k8s/nav-entry/templates/configmap.yaml`, `k8s/nav-entry/templates/job.yaml`, `k8s/nav-entry/templates/cleanup-job.yaml`

Add two new fields to nav-entry:

- **`description`** (string, optional) — short blurb for the card
- **`enabled`** (bool, default `true`) — `false` renders as a greyed-out placeholder

The `icon` field remains a string but is now dual-purpose:
- If it starts with `http` → treated as an image URL
- Otherwise → treated as a Material Symbols icon name

Update the individual ConfigMap to include the new fields. Update the aggregation job script to extract and pass through `description` and `enabled` into the nav.json array.

**Umbrella chart nav entries** (`nav-oxigraph.yaml`, `nav-keycloak.yaml`) also need updating to include the new fields.

**Tests:** Update existing helm-unittest tests, add cases for the new fields.

## Part 2: Placeholder Entries in Umbrella Chart

**Files:** `k8s/grl-platform/values.yaml`, new template `k8s/grl-platform/templates/nav-placeholders.yaml`

Define future/placeholder products in umbrella chart values:

```yaml
home:
  enabled: true
  placeholders:
    - appId: ingest-data
      name: Ingest Data
      description: Upload data from source systems...
      icon: https://...or-material-icon-name
      order: 10
      enabled: false
    - appId: generate-insights
      name: Generate Insights
      description: Select and customise metrics...
      icon: insights
      order: 40
      enabled: false
    # etc.
```

Template renders a nav-entry ConfigMap for each placeholder (same label pattern so they get aggregated).

## Part 3: Home Page (Static nginx)

**New files in `k8s/grl-platform/templates/`:**

- **`home-configmap.yaml`** — HTML/CSS/JS in a ConfigMap. The page:
  - Fetches `/nav.json` at runtime
  - Renders colourful cards for `enabled: true` entries
  - Renders greyed-out cards for `enabled: false` entries
  - Icons: `<img>` for URL icons, Material Symbols `<span>` for named icons
  - Sorted by `order`
  - Header with "GRL Semantic AI Fabric" branding (configurable title/tagline in values)

- **`home-deployment.yaml`** — `nginx:alpine`, two volume mounts:
  - ConfigMap `home-html` → `/usr/share/nginx/html/`
  - ConfigMap `grl-platform-nav` → `/usr/share/nginx/html/nav.json` (subPath)
  - Reloader annotation for `grl-platform-nav` so cards update when apps are installed/removed

- **`home-service.yaml`** — ClusterIP on port 80

- **`home-httproute.yaml`** — bare `global.host`, PathPrefix `/`, no auth filter

All gated on `home.enabled` (default `true`).

## Part 4: Consumer Updates (Follow-up)

Not in this PR, but consumers should add `description` to their nav-entry values:
- GraphSandbox: `description: "Visually explore transactional data and ontologies"`
- FHIR Mapper: `description: "Generate REST APIs, React apps, or MCP servers..."`

Existing consumers without `description` will render cards with title + icon only (no description text).

## Verification

1. `helm unittest k8s/nav-entry` — all tests pass including new field coverage
2. `helm lint k8s/grl-platform` — no lint errors
3. `helm template grl-platform k8s/grl-platform --set global.host=staging.dev` — verify:
   - Home page templates render (ConfigMap, Deployment, Service, HTTPRoute)
   - Placeholder nav entries render with `enabled: "false"`
   - HTTPRoute claims `staging.dev` with no auth filter
4. Deploy to k3d and verify:
   - `curl https://staging.dev` returns the landing page
   - Cards render from nav.json (colourful for installed apps, greyed for placeholders)
   - Installing a new consumer auto-updates the cards via Reloader
