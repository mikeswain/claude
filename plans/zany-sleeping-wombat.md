# Plan: Remove Namespace Flattening

## Context

The ASG pipeline flattens all OWL entity IRIs into a single namespace before mapping to OpenAPI. This exists because downstream code reconstructs `OWLClass`/`OWLProperty` objects by concatenating `defaultOntologyPrefixIRI + shortName`, which only works when all entities share one namespace. This requires a fragile, per-transform exclusion list (`namespaceMappings`) to avoid remapping standard vocabularies (OWL, RDF, XSD, SKOS, gist, etc.) — an unnecessary configuration burden that could produce odd results if incomplete.

**Fix**: Pass OWL entity objects through the pipeline instead of string short names. Convert to strings only at the OpenAPI output boundary. This eliminates the single-namespace assumption and removes the need for namespace mapping configuration.

## Implementation Steps

### Step 1: Add OWL property reverse-lookup maps to SchemaVisitor

**File**: `src/main/java/com/grl/component/schema/SchemaVisitor.java`

- Add `Map<String, OWLObjectProperty> objectPropertiesByName` and `Map<String, OWLDataProperty> dataPropertiesByName`
- Populate with `putIfAbsent` wherever `objectPropertyNames.add()`/`dataPropertyNames.add()` are called (7 object property sites, 5 data property sites)
- Add unmodifiable getters
- **Purely additive** — no callers yet, existing tests pass unchanged

### Step 2: Change MappingContext.addSchema() to accept OWLClass

**File**: `src/main/java/com/grl/component/mapper/MappingContext.java`

- Change `addSchema(String shortName)` → `addSchema(OWLClass cls)`, derive short name internally via `nameResolver.resolve(cls)`
- Add a **temporary deprecated** `addSchema(String)` overload that searches ontology signature by short name (keeps ObjectPropertyMapper compiling until Step 4)
- Remove `defaultOntologyPrefixIRI` field and getter
- Remove `OWLOntologyManager manager` field and `getManager()` — only used for IRI reconstruction in `addSchema` and by ObjectPropertyMapper (Step 4)
- Add `PropertyNameResolver` field

### Step 3: Eliminate findDataProperty/findObjectProperty in ResourceSchemaMapper

**File**: `src/main/java/com/grl/component/mapper/ResourceSchemaMapper.java`

- In `processDataProperties()`: replace `findDataProperty(propertyName)` with `visitor.getDataPropertiesByName().get(propertyName)`
- In `processObjectProperties()`: replace `findObjectProperty(propertyName)` with `visitor.getObjectPropertiesByName().get(propertyName)`
- In `processTaxonomyProperties()`: same for `findObjectProperty`
- Delete the `findDataProperty()` and `findObjectProperty()` methods
- For `createSchemaVisitor()`: replace `context.getManager().getOWLDataFactory()` with static `OWLManager.getOWLDataFactory()`

### Step 4: Change ObjectPropertyMapper to accept OWLClass ranges

**Files**: `ObjectPropertyMapper.java`, `ResourceSchemaMapper.java`, `RestrictionInfo.java`

**ObjectPropertyMapper**: Change `List<String> ref` → `List<OWLClass> rangeClasses`. Derive `List<String> ref` internally via `nameResolver.resolve()`. Pass `OWLClass` to `context.addSchema()` and `mapLinkedResourceClass()` directly. Remove all `IRI.create(prefix + ref)` reconstruction.

**ResourceSchemaMapper**: Add `extractObjectPropertyRangeClasses()` returning `List<OWLClass>` — gets OWLClass objects from visitor's `objectPropertyRanges` multimap and from a new `RestrictionInfo.getRangeClasses()`. Update `processObjectProperties()` to call this and pass result to ObjectPropertyMapper.

**RestrictionInfo**: Add `getRangeClasses()` returning `List<OWLClass>` (extract filler from SomeValuesFrom/AllValuesFrom). Keep existing `getRangeNames()` for data property XSD types.

After this step, remove the deprecated `addSchema(String)` overload from MappingContext.

### Step 5: Remove namespace flattening from OntologyMapper

**File**: `src/main/java/com/grl/component/mapper/OntologyMapper.java`

**`loadOntologies()`**: Remove `AsgUtils.applyNamespaceMappings()` call (line 133-134). Remove saving `ontology-flat.owl` (or save merged ontology with that filename for backward compat).

**`mapApiRoot()`**:
- Find APIRoot by searching ontology signature: `ontology.getClassesInSignature().stream().filter(cls -> nameResolver.resolve(cls).equals("APIRoot"))` instead of `IRI.create(defaultIRIPrefix, "APIRoot")`
- `x-grl-graph-namespace`: derive from ontology IRI instead of namespace mapping config
- Remove `defaultIRIPrefix` local variable and `defaultOntologyPrefixIRI` from MappingContext constructor
- Remove import of `NamespaceMappingsDeserializer.DEFAULT_NS`

### Step 6: Update Asg CLI

**File**: `src/main/java/com/grl/component/asg/Asg.java`

- Rename `flattenOnly()` → `mergeOnly()` or update log message
- Update CLI help text for `--no-generate` option

### Step 7: Delete dead code

- `AsgUtils.java`: Delete `applyNamespaceMappings()` and `InvertablePattern` inner record
- `NamespaceMappingsDeserializer.java`: Delete entire file
- `YamlConfig.java`: Remove `namespaceMappings` field, `@JsonDeserialize`, `@Size`, and getter

### Step 8: Update tests and configs

**Delete**:
- `AsgUtilsTest.java` namespace mapping tests
- `NamespaceMappingsDeserializerTest.java`

**Update**:
- `ConfigTest.java`: remove `namespaceMappings` assertions
- `SimpleApiIT.java`: update `x-grl-graph-namespace` assertion to match ontology IRI
- All test config YAML files: remove `namespaceMappings` entries
  - `regression/config_base.yaml`
  - `regression/imports/business-ontology/OpenAPI/config_employment.yaml`
  - Config inheritance test files

## Verification

```bash
mvn clean verify    # All unit + integration tests pass
```

Check that generated OpenAPI output (schema names, $ref values, property names) is unchanged — only `x-grl-graph-namespace` extension value changes.
