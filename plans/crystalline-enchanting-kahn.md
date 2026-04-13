# Configurable Graph Context URI Pattern

## Context

Currently, the graph context IRI is constructed rigidly: `ApiController` extracts `organizationId` from JWT and passes it as `graphName` to rest2graph, which prefixes it with the namespace from the OpenAPI spec (`x-grl-graph-namespace`), producing e.g. `https://www.graphresearchlabs.com/orgA`.

This works for multi-tenant isolation but some tools need more control — e.g. a fixed domain, user-scoped graphs, or header-driven graph selection.

**Goal:** A configurable `GRAPH_CONTEXT_PATTERN` env var (e.g. `http://adomain.com/{namespace}/{org}/{apiname}`) with placeholder substitution, producing a fully-resolved IRI that rest2graph uses as-is.

## Analysis of Current Flow

```
JWT → ApiController.resolveOrganization() → orgId string
  → ApiHandler.getResource(orgId, path, params)
  → Controller → OpenApiHelper.forRequest(orgId)
  → new ApiGraphContext(namespace, orgId, apiLabel)
  → createIRI(namespace.getName(), orgId)  ← always prefixes
  → IRI used in SPARQL FROM clauses
```

## Changes

### 1. rest2graph: `ApiGraphContext.java` — detect full IRIs

**File:** `rest2graph/src/main/java/com/grl/component/rdf/ApiGraphContext.java`

Add a private static helper and modify both String constructors:

```java
private static IRI resolveGraphContextIRI(Namespace namespace, String graphContext) {
    return graphContext.contains("://")
        ? SimpleValueFactory.getInstance().createIRI(graphContext)
        : SimpleValueFactory.getInstance().createIRI(namespace.getName(), graphContext);
}
```

Both string constructors call this instead of hardcoding namespace prefixing. Bare org IDs (`orgA`) never contain `://` so existing behavior is preserved. Full IRIs pass through directly.

**Test:** New `ApiGraphContextTest` — bare string produces namespaced IRI, full IRI passes through unchanged.

### 2. graph-server: `YamlConfig.java` — add config property

**File:** `server/config/YamlConfig.java`

Add field `String graphContextPattern` + setter. Maps to `application.graphContextPattern` / env `APPLICATION_GRAPHCONTEXTPATTERN`.

### 3. graph-server: `ApiController.java` — resolve pattern

**File:** `server/controller/ApiController.java`

- Inject `YamlConfig`, `OpenAPI` bean, `apiName` bean
- Add static `resolveGraphContext(pattern, orgId, sub, namespace, apiName, headerLookup)` method
- Call it in `parseRequest()` after resolving orgId

When pattern is null/blank → returns raw orgId (exact current behavior, zero-risk default).

**Available placeholders:**

| Placeholder | Source |
|---|---|
| `{org}` | JWT org claim (existing logic) |
| `{sub}` | JWT `sub` claim |
| `{namespace}` | `APIUtils.namespaceFrom(openAPI).getName()` |
| `{apiname}` | `OpenApiUtils.getApiName(openAPI)` (existing bean) |
| `{http.header.X}` | Request header value |

**Example configs:**
```yaml
# Current behavior (default when unset)
# graphContextPattern:

# Explicit namespace + org (equivalent to current)
graphContextPattern: "{namespace}{org}"

# Custom domain with api scoping
graphContextPattern: "https://data.example.com/graphs/{apiname}/{org}"

# User-scoped
graphContextPattern: "{namespace}{org}/{sub}"

# Header-driven
graphContextPattern: "{http.header.X-Graph-URI}"
```

**Test:** New `ApiControllerTest` — unit test the static method with various patterns, null pattern, missing headers, etc.

### 4. Rename `GraphApiRequest.organizationId` → `graphContext`

Since this field now carries either a raw org ID or a full resolved URI, rename for clarity. Update all references in the controller.

## Sequencing

1. **rest2graph first** — `ApiGraphContext` change + tests, bump version
2. **graph-server second** — bump rest2graph dep, add config + resolution logic + tests

## Verification

1. `mvn test` in rest2graph — existing tests pass, new `ApiGraphContextTest` passes
2. `mvn test` in graph-server — existing tests pass, new `ApiControllerTest` passes
3. Manual: deploy with no `graphContextPattern` set → identical behavior to today
4. Manual: deploy with `graphContextPattern: "{namespace}{org}/{apiname}"` → verify SPARQL FROM clause uses the full IRI
