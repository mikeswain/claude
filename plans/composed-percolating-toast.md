# Plan: Switch GraphSandbox builder store from RDF4J to shared Oxigraph

## Context

GraphSandbox's builder uses a standalone RDF4J Workbench instance to store ontologies during the upload/annotation workflow. Every other GRL product uses the shared Oxigraph instance from grl-platform. This creates an unnecessary dependency on a different graph database with different APIs.

The integration tests already use Oxigraph successfully, confirming SPARQL compatibility. Scope is builder store only — per-sandbox GraphDB instances are unchanged.

## Named graph URI scheme (replacing per-user repositories)

**Current RDF4J model** uses two levels of isolation:
- **Per-user repository**: `/rdf4j-server/repositories/{cleanLabel(userName)}` — e.g. `repositories/mike_at_example.com`
- **Per-ontology named graph**: `https://graphresearchlabs.com/{ontologyName}` — e.g. `https://graphresearchlabs.com/pizza`

SPARQL queries hit the repository URL and scope to the named graph via `GRAPH <uri> { ... }`.

**New Oxigraph model** — single store, no repository concept. We keep the same named graph URIs:
- **Graph URI**: `https://graphresearchlabs.com/{ontologyName}` — unchanged

This works because:
1. Every SPARQL query in `frontend/app/routes/ontology/sparql.ts` already scopes with `GRAPH <${graphUri}>` — no cross-graph leakage
2. Ontology names are unique per user (enforced by the builder's directory structure: `sandboxes/{userName}/ontologies/{ontologyName}`)
3. The `OntologyEndpoints` URLs are stored per-ontology and used directly by the frontend — the frontend never constructs graph store URLs itself

If two users upload ontologies with the same name, the graph URIs would collide. To prevent this, we include the username in the graph URI:
- **Graph URI**: `https://graphresearchlabs.com/{cleanLabel(userName)}/{ontologyName}`

This replaces the per-user repository as the isolation boundary. The `graphContextUrl` is the only thing that changes — all SPARQL queries already use it.

**Endpoint mapping per ontology:**

| Endpoint | RDF4J (current) | Oxigraph (new) |
|----------|-----------------|----------------|
| `downloadUrl` | `{repoUrl}/statements?context=<{graphCtx}>` | `{base}/store?graph={graphCtx}` |
| `sparqlQueryUrl` | `{repoUrl}` | `{base}/query` |
| `sparqlUpdateUrl` | `{repoUrl}/statements` | `{base}/update` |
| `graphContextUrl` | `https://graphresearchlabs.com/{ontName}` | `https://graphresearchlabs.com/{user}/{ontName}` |

Where `{base}` is `http://{host}:7878` (no per-user path segment needed).

## API mapping: RDF4J → Oxigraph

| Operation | RDF4J | Oxigraph |
|-----------|-------|----------|
| Create repository | `PUT /repositories/{name}` (Turtle config body) | Not needed — single store, use named graphs |
| Upload to named graph | `POST /repositories/{name}/statements?context=<uri>` | `POST /store?graph=<uri>` |
| Delete named graph | `DELETE /repositories/{name}/statements?context=<uri>` | `DELETE /store?graph=<uri>` |
| SPARQL query | `POST /repositories/{name}` (Content-Type: application/sparql-query) | `POST /query` |
| SPARQL update | `POST /repositories/{name}/statements` (Content-Type: application/sparql-update) | `POST /update` |

## Changes

### 1. `shared/src/generatorConfig.ts` — change base URL pattern

Current: `http://{host}:{port}/rdf4j-server` (port 8080)
New: `http://{host}:{port}` (port 7878, Oxigraph default)

Change `GRAPH_PORT` default from `'8080'` to `'7878'` and remove the `/rdf4j-server` path suffix.

### 2. `builder/src/uploadOntology.ts` — rewrite for Oxigraph endpoints

- **Remove `ensureRepositoryExists()`** — Oxigraph has no repository concept; named graphs are implicit.
- **Rewrite `upload()`** — Change URL construction:
  - `downloadUrl`: `{graphBase}/store?graph={graphContextUrl}` (was `{repoUrl}/statements?context=<{graphContextUrl}>`)
  - `sparqlQueryUrl`: `{graphBase}/query` (was `{repoUrl}`)
  - `sparqlUpdateUrl`: `{graphBase}/update` (was `{repoUrl}/statements`)
  - `graphContextUrl`: unchanged
- **Rewrite `uploadOntology()`** — Remove the `ensureRepositoryExists` call. Pass `graphBase` directly to `upload()`.
- The DELETE + POST pattern for uploading stays the same, just different URLs.

### 3. `dev.docker-compose.yml` — replace RDF4J with Oxigraph

Replace:
```yaml
rdf4j-server:
  image: eclipse/rdf4j-workbench:latest
  ports: ["8080:8080"]
  volumes: [rdf4j-data:/var/rdf4j]
```
With:
```yaml
oxigraph:
  image: ghcr.io/oxigraph/oxigraph:latest
  ports: ["7878:7878"]
  volumes: [oxigraph-data:/data]
  command: serve --location /data --bind 0.0.0.0:7878
```

Update volumes: `rdf4j-data` → `oxigraph-data`.

### 4. `k8s/values.yaml` + `k8s/templates/_helpers.tpl` — add Oxigraph discovery (K8s)

Follow the same pattern as Ontology-Manager:
- Add `oxigraph.serviceFqdn` and `oxigraph.port` values (default empty, looked up from `grl-platform-oxigraph` ConfigMap)
- Add a `sandbox-generator.oxigraphServiceFqdn` helper that reads from the discovery ConfigMap with fallback to explicit value
- Wire `GRAPH_HOST` and `GRAPH_PORT` env vars in `k8s/templates/deployment.yaml` to use the resolved Oxigraph FQDN/port

### 5. `shared/src/types.ts` — no change

`OntologyEndpoints` type is URL-based and transport-agnostic. No change needed.

### 6. Tests

- **`builder/src/__tests__/uploadOntology.test.ts`** — update mocked URLs and remove repository creation expectations
- **`frontend/server/__tests__/integration.globalSetup.ts`** — already uses Oxigraph, no change needed
- **`frontend/server/__tests__/annotations.integration.test.ts`** — already uses Oxigraph endpoints, verify no change needed

## Files to modify

| File | Change |
|------|--------|
| `shared/src/generatorConfig.ts` | Port 8080→7878, remove `/rdf4j-server` path |
| `builder/src/uploadOntology.ts` | Remove repo creation, rewrite URLs for Oxigraph |
| `dev.docker-compose.yml` | Replace RDF4J service with Oxigraph |
| `k8s/values.yaml` | Add `oxigraph` config block |
| `k8s/templates/_helpers.tpl` | Add `oxigraphServiceFqdn` helper |
| `k8s/templates/deployment.yaml` | Wire `GRAPH_HOST`/`GRAPH_PORT` from Oxigraph discovery |
| `builder/src/__tests__/uploadOntology.test.ts` | Update mocked URLs |

## Verification

1. `npm run build -w shared && npm run build -w builder` — compiles
2. `docker compose -f dev.docker-compose.yml up -d` — Oxigraph starts on 7878
3. Run builder tests: `npm test -w builder`
4. Run frontend integration tests: `npm test -w frontend` (already Oxigraph-based)
5. Manual: upload an ontology via the UI, verify SPARQL queries work in annotation view
