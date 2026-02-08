# Plan: Enhance Pattern B to Support Individual Graphs

## Summary

Enhance Pattern B taxonomy handling to support individuals with object property assertions, generating `oneOf` schemas with `const` objects instead of simple string enums.

## Requirements

| Decision | Choice |
|----------|--------|
| Output structure | `oneOf` with `const` objects |
| Nesting depth | Configurable via `maxIndividualDepth`, default 2 |
| Data properties | Include as `const` with `type` |
| Code/identifier | Fallback: `skos:notation` → IRI local name |
| Nested individuals | Expand to nested objects recursively (up to max depth) |

## Example Transformation

**OWL Input:**
```turtle
sensor:SensorTypeAnalog a owl:NamedIndividual, sensor:SensorTypeCode ;
    skos:notation "analog" ;
    skos:prefLabel "Analog Sensor" ;
    sensor:hasCategory sensor:MeasurementCategory ;
    sensor:voltage "5V" .

sensor:MeasurementCategory a owl:NamedIndividual ;
    skos:notation "measurement" ;
    sensor:unit "volts" .
```

**OpenAPI Output (maxIndividualDepth: 2):**
```yaml
oneOf:
  - type: object
    properties:
      code:
        const: "analog"
        type: string
      hasCategory:
        type: object
        properties:
          code:
            const: "measurement"
            type: string
          unit:
            const: "volts"
            type: string
      voltage:
        const: "5V"
        type: string
```

## Files to Modify

| File | Changes |
|------|---------|
| `YamlConfig.java` | Add `maxIndividualDepth` field (default: 2) |
| `TaxonomyInfo.java` | Add `IndividualGraph` record for property assertions |
| `SchemaVisitor.java` | Extract object/data property assertions for Pattern B |
| `TaxonomyPatternMapper.java` | Generate `oneOf` with nested const objects |
| `MappingContext.java` | Expose `maxIndividualDepth` from config |
| `docs/configuration/config-reference.md` | Document `maxIndividualDepth` option |
| `docs/ontology-design/taxonomy-patterns.md` | Update Pattern B section with new output format |

## Implementation Details

### 1. Configuration: YamlConfig.java

**File:** `src/main/java/com/grl/component/asg/configuration/YamlConfig.java`

Add new field:
```java
@Builder.Default
private int maxIndividualDepth = 2;

public int getMaxIndividualDepth() {
    return maxIndividualDepth;
}
```

### 2. Data Model: TaxonomyInfo.java

**File:** `src/main/java/com/grl/component/schema/TaxonomyInfo.java`

Add new record to represent individual's property graph:
```java
public record PropertyAssertion(
    String propertyName,
    @Nullable String literalValue,      // For data properties
    @Nullable String literalType,       // XSD type for data properties
    @Nullable OWLNamedIndividual objectValue  // For object properties
) {
    public boolean isDataProperty() {
        return literalValue != null;
    }
    public boolean isObjectProperty() {
        return objectValue != null;
    }
}

public record IndividualGraph(
    String code,                        // skos:notation or IRI fallback
    @Nullable String displayName,       // skos:prefLabel
    @Nullable String definition,        // skos:definition
    List<PropertyAssertion> assertions  // object + data property assertions
) {}
```

Update `IndividualMetadata` or replace with `IndividualGraph` in Pattern B handling.

### 3. Detection: SchemaVisitor.java

**File:** `src/main/java/com/grl/component/schema/SchemaVisitor.java`

Modify Pattern B detection to extract property assertions:

```java
private IndividualGraph extractIndividualGraph(OWLNamedIndividual individual) {
    String code = getCodeValue(individual);  // notation → IRI fallback
    String displayName = getAnnotationValue(individual, SKOS_PREFLABEL);
    String definition = getAnnotationValue(individual, SKOS_DEFINITION);

    List<PropertyAssertion> assertions = new ArrayList<>();

    // Extract data property assertions
    EntitySearcher.getDataPropertyValues(individual, ontology)
        .forEach((prop, values) -> {
            for (OWLLiteral lit : values) {
                assertions.add(new PropertyAssertion(
                    nameResolver.resolve(prop),
                    lit.getLiteral(),
                    lit.getDatatype().getIRI().getShortForm(),
                    null
                ));
            }
        });

    // Extract object property assertions
    EntitySearcher.getObjectPropertyValues(individual, ontology)
        .forEach((prop, values) -> {
            for (OWLIndividual target : values) {
                if (target.isNamed()) {
                    assertions.add(new PropertyAssertion(
                        nameResolver.resolve(prop),
                        null,
                        null,
                        target.asOWLNamedIndividual()
                    ));
                }
            }
        });

    return new IndividualGraph(code, displayName, definition, assertions);
}

private String getCodeValue(OWLNamedIndividual individual) {
    // Fallback: skos:notation → IRI local name
    String notation = getAnnotationValue(individual, SKOS_NOTATION);
    return notation != null ? notation : nameResolver.resolve(individual);
}
```

### 4. Schema Generation: TaxonomyPatternMapper.java

**File:** `src/main/java/com/grl/component/mapper/TaxonomyPatternMapper.java`

Replace `mapPatternB()` with graph-aware generation:

```java
private Schema<?> mapPatternB() {
    int maxDepth = context.getMaxIndividualDepth();

    // Backward compatibility: depth 0 = simple string enum (old behavior)
    if (maxDepth == 0) {
        return mapPatternBLegacy();
    }

    List<Schema<?>> individualSchemas = new ArrayList<>();
    for (IndividualGraph graph : info.individualGraphs()) {
        Schema<?> indSchema = buildIndividualSchema(graph, maxDepth, new HashSet<>());
        individualSchemas.add(indSchema);
    }

    ComposedSchema oneOfSchema = new ComposedSchema();
    oneOfSchema.oneOf(individualSchemas);
    oneOfSchema.setDescription(description);

    return wrapInArrayIfNeeded(oneOfSchema);
}

private Schema<?> mapPatternBLegacy() {
    // Original Pattern B: simple string enum
    List<String> enumValues = info.individualGraphs().stream()
        .map(IndividualGraph::code)
        .toList();

    StringSchema enumSchema = new StringSchema();
    enumSchema.setDescription(description);
    enumSchema.setEnum(enumValues);

    return wrapInArrayIfNeeded(enumSchema);
}

private Schema<?> buildIndividualSchema(IndividualGraph graph, int remainingDepth, Set<String> visited) {
    ObjectSchema schema = new ObjectSchema();

    // Add code field
    Schema<?> codeSchema = new StringSchema();
    codeSchema.setConst(graph.code());
    schema.addProperty("code", codeSchema);

    // Add displayName if present
    if (graph.displayName() != null) {
        Schema<?> displaySchema = new StringSchema();
        displaySchema.setConst(graph.displayName());
        schema.addProperty("displayName", displaySchema);
    }

    // Add definition if present
    if (graph.definition() != null) {
        Schema<?> defSchema = new StringSchema();
        defSchema.setConst(graph.definition());
        schema.addProperty("definition", defSchema);
    }

    // Add property assertions
    for (PropertyAssertion assertion : graph.assertions()) {
        Schema<?> propSchema;

        if (assertion.isDataProperty()) {
            propSchema = new Schema<>();
            propSchema.setConst(assertion.literalValue());
            propSchema.setType(mapXsdToJsonType(assertion.literalType()));
        } else if (assertion.isObjectProperty() && remainingDepth > 0) {
            OWLNamedIndividual target = assertion.objectValue();
            String targetCode = getCodeValue(target);

            if (visited.contains(targetCode)) {
                // Cycle detection - use code reference
                propSchema = new StringSchema();
                propSchema.setConst(targetCode);
            } else {
                visited.add(targetCode);
                IndividualGraph targetGraph = extractIndividualGraph(target);
                propSchema = buildIndividualSchema(targetGraph, remainingDepth - 1, visited);
                visited.remove(targetCode);
            }
        } else {
            // Max depth reached or object property - use code reference
            propSchema = new StringSchema();
            propSchema.setConst(getCodeValue(assertion.objectValue()));
        }

        schema.addProperty(assertion.propertyName(), propSchema);
    }

    return schema;
}
```

### 5. Context: MappingContext.java

**File:** `src/main/java/com/grl/component/mapper/MappingContext.java`

Add accessor:
```java
public int getMaxIndividualDepth() {
    return config.getMaxIndividualDepth();
}
```

## Documentation Updates

### 1. Config Reference: docs/configuration/config-reference.md

Add new section under "Optional Fields":

```markdown
### `maxIndividualDepth`

**Type:** `integer`
**Default:** `2`

Controls how deeply object property assertions on Pattern B taxonomy individuals are traversed when generating `oneOf` schemas.

```yaml
maxIndividualDepth: 2
```

| Value | Behavior |
|-------|----------|
| `0` | Legacy mode: generates simple string enum (backward compatible) |
| `1` | Single level: only direct property assertions on individuals |
| `2+` | Recursive: expands nested individuals up to specified depth |

**Example with depth 2:**

OWL individual with nested object property:
```turtle
sensor:SensorTypeAnalog a sensor:SensorTypeCode ;
    skos:notation "analog" ;
    sensor:hasCategory sensor:MeasurementCategory .

sensor:MeasurementCategory a owl:NamedIndividual ;
    skos:notation "measurement" ;
    sensor:unit "volts" .
```

Generates `oneOf` with nested objects:
```yaml
oneOf:
  - type: object
    properties:
      code:
        const: "analog"
        type: string
      hasCategory:
        type: object
        properties:
          code:
            const: "measurement"
            type: string
          unit:
            const: "volts"
            type: string
```
```

### 2. Taxonomy Patterns: docs/ontology-design/taxonomy-patterns.md

Update the Pattern B section. Replace the current "OpenAPI Output" showing simple enum with:

```markdown
### OpenAPI Output

With `maxIndividualDepth >= 1` (default), Pattern B generates `oneOf` schemas with const objects:

```yaml
hasSensorType:
  type: array
  minItems: 1
  items:
    oneOf:
      - type: object
        properties:
          code:
            const: "analog"
            type: string
          displayName:
            const: "Analog"
            type: string
          definition:
            const: "Analog sensor type."
            type: string
      - type: object
        properties:
          code:
            const: "digital"
            type: string
          displayName:
            const: "Digital"
            type: string
          definition:
            const: "Digital sensor type."
            type: string
```

### Object Property Assertions

When Pattern B individuals have object property assertions, these are expanded as nested objects (up to `maxIndividualDepth`):

**OWL:**
```xml
<owl:NamedIndividual rdf:about="https://example.com/SensorTypeAnalog">
    <rdf:type rdf:resource="https://example.com/SensorTypeCode"/>
    <skos:notation>analog</skos:notation>
    <example:hasCategory rdf:resource="https://example.com/MeasurementCategory"/>
</owl:NamedIndividual>
```

**OpenAPI (depth 2):**
```yaml
oneOf:
  - type: object
    properties:
      code:
        const: "analog"
        type: string
      hasCategory:
        type: object
        properties:
          code:
            const: "measurement"
            type: string
```

### Cycle Detection

If object property assertions form a cycle (A → B → A), ASG detects this and uses a string reference instead of infinite nesting.

### Legacy Mode

Set `maxIndividualDepth: 0` to generate simple string enums (pre-enhancement behavior):

```yaml
hasSensorType:
  type: array
  minItems: 1
  items:
    type: string
    enum:
      - analog
      - digital
```
```

Also update the Pattern Overview table at the top:

```markdown
| Pattern | OWL Structure | OpenAPI Output | Use Case |
|---------|---------------|----------------|----------|
| **A** | `owl:oneOf` with untyped individuals | Inline string enum | Simple, closed enumeration |
| **B** | `owl:oneOf` with typed individuals | `oneOf` with const objects | Closed enumeration with graphs |
| **C** | Named class (no SKOS) | `$ref` to schema | Extensible, hierarchical taxonomy |
| **D** | SKOS-based class | Inline string enum | Standards-compliant vocabulary |
```

Update the Pattern A vs B Comparison table:

```markdown
| Aspect | Pattern A | Pattern B |
|--------|-----------|-----------|
| Individual typing | `owl:NamedIndividual` only | Domain class + metadata |
| OpenAPI output | Simple string enum | `oneOf` with const objects |
| Property assertions | Not supported | Expanded as nested objects |
| SPARQL query | Parse `owl:oneOf` | `?x a SensorTypeCode` |
| Metadata support | `rdfs:label` only | Full annotation + property support |
```

## Test Updates

### Unit Tests

**File:** `src/test/java/com/grl/component/mapper/TaxonomyPatternMapperTest.java`

Add tests for:
- Pattern B generates oneOf with const objects
- Nested object properties expand to nested objects
- Data properties include type information
- Cycle detection prevents infinite recursion
- maxIndividualDepth=0 falls back to simple enum behavior
- Code fallback (notation → IRI)

**File:** `src/test/java/com/grl/component/schema/SchemaVisitorTaxonomyDetectionTest.java`

Add tests for:
- extractIndividualGraph captures data property assertions
- extractIndividualGraph captures object property assertions
- Code value uses notation when present
- Code value falls back to IRI when notation absent

### Integration Tests

**File:** `src/test/resources/regression/ontologies/taxonomy-patterns.owl`

Add Pattern B individuals with:
- Data property assertions (various XSD types)
- Object property assertions (to other individuals)
- Nested individuals (2+ levels deep)
- Cycle scenario (A → B → A)

**File:** `src/test/resources/regression/configs/config_taxonomy_patterns.yaml`

Add `maxIndividualDepth: 2` to test config.

**File:** `src/test/java/regression/features/properties/TaxonomyPatternsIT.java`

Add tests for:
- Pattern B generates oneOf structure
- Nested properties are expanded
- Const values match expected codes

## Verification

```bash
# Run unit tests
mvn test -Dtest=TaxonomyPatternMapperTest,SchemaVisitorTaxonomyDetectionTest

# Run integration tests
mvn verify -Dit.test=TaxonomyPatternsIT

# Full test suite
mvn clean verify
```

## Backward Compatibility

- `maxIndividualDepth: 0` preserves old behavior: generates simple `enum: ['analog', 'digital']`
- `maxIndividualDepth: 1+` generates `oneOf` with const objects
- Default `maxIndividualDepth: 2` is a **breaking change** for existing Pattern B users
- Document migration path in release notes

## Metadata Handling

When `maxIndividualDepth >= 1`, include `displayName` and `definition` in const objects if annotations are present:

```yaml
oneOf:
  - type: object
    properties:
      code:
        const: "analog"
        type: string
      displayName:
        const: "Analog Sensor"
        type: string
      definition:
        const: "Analog sensor type"
        type: string
      hasCategory:
        # ... nested object
```
