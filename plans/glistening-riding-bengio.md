# Plan: Refactor ResourceSchemaMapper for Legibility

## Summary

Refactor the 362-line `ResourceSchemaMapper` class to improve legibility by extracting the 228-line `createSchema()` method into focused helper methods. The refactoring will be done in stages, starting with low-risk extractions and progressing to more complex ones.

## Current Problems

1. **God Method**: `createSchema()` has 228 lines with 5+ distinct responsibilities
2. **Code Duplication**: Data and object property processing loops share ~80% identical structure
3. **Poor Variable Naming**: `rangesDP`, `propertyFromDataRestrictions`, misleading type names
4. **Complex Inline Logic**: 17-line stream chain for example value extraction
5. **No Intermediate Abstractions**: Everything happens in one method

## File to Modify

`src/main/java/com/grl/component/mapper/ResourceSchemaMapper.java`

## Refactoring Stages

### Stage 1: Low-Risk Extract Methods (Lines → ~10 methods)

Extract simple, self-contained blocks that have clear boundaries:

| Method | Current Lines | Purpose |
|--------|---------------|---------|
| `initializeResourceSchema()` | 78-86 | Create and configure base ObjectSchema |
| `createSchemaVisitors()` | 87-93 | Factory for visitor pair |
| `collectRelevantAxioms()` | 95-99 | Gather taxonomy, subclass, superclass axioms |
| `mergeVisitorRestrictions()` | 121-131 | Combine restrictions from both visitors |
| `logMissingPropertiesWarning()` | 291-302 | Diagnostic logging |

### Stage 2: Property Lookup Helpers

Extract duplicated property lookup patterns:

```java
private Optional<OWLDataProperty> findDataProperty(String propertyName)
private Optional<OWLObjectProperty> findObjectProperty(String propertyName)
```

### Stage 3: Range Extraction Helper

Unify the duplicated range extraction pattern (appears in both loops):

```java
private List<String> extractPropertyRanges(
    String propertyName,
    Multimap<String, ? extends OWLEntity> rangesFromVisitor,
    Map<String, Set<RestrictionInfo>> restrictions)
```

### Stage 4: Example Value Extraction

Extract the complex 17-line stream chain into a readable method:

```java
private String extractExampleValue(
    OWLDataProperty property,
    List<OWLSubClassOfAxiom> subClassAxioms)
```

### Stage 5: Property Processing Methods

Extract the main processing loops:

```java
private void processDataProperties(
    Schema<?> schema,
    Set<String> propertyNames,
    Map<String, Set<RestrictionInfo>> restrictions,
    Multimap<String, OWLDatatype> dataRanges,
    List<OWLSubClassOfAxiom> subClassAxioms)

private void processObjectProperties(
    Schema<?> schema,
    Set<String> propertyNames,
    Map<String, Set<RestrictionInfo>> restrictions,
    Multimap<String, OWLClass> objectRanges)

private void processTaxonomyProperties(
    Schema<?> schema,
    List<OWLSubClassOfAxiom> taxonomyAxioms)
```

### Stage 6: Variable Renaming

Improve clarity throughout:

| Old Name | New Name |
|----------|----------|
| `rangesDP` | `dataPropertyRangeTypes` |
| `rangesOP` | `objectPropertyRangeClasses` |
| `propertyFromDataRestrictions` | `dataProperty` |
| `propertyFromObjectRestrictions` | `objectProperty` |
| `objectSchemaVisitor` | `objectPropertyVisitor` |
| `dataSchemaVisitor` | `dataPropertyVisitor` |

### Stage 7: Documentation

Add comprehensive Javadoc and inline comments throughout.

#### Class-Level Javadoc

```java
/**
 * Maps an OWL class to an OpenAPI Schema object.
 *
 * <p>This mapper analyzes an OWL class from the ontology and generates a corresponding
 * OpenAPI 3.1 ObjectSchema with properties derived from:
 * <ul>
 *   <li><b>Data properties</b> - mapped to primitive schema types (string, integer, etc.)</li>
 *   <li><b>Object properties</b> - mapped to $ref references or inline schemas</li>
 *   <li><b>Taxonomy properties</b> - mapped to enum schemas based on named individuals</li>
 * </ul>
 *
 * <h2>Processing Pipeline</h2>
 * <pre>
 * OWLClass → SchemaVisitors → Axiom Collection → Restriction Merging
 *                                    ↓
 *         ┌────────────────────────────────────────────┐
 *         │                                            │
 *         ▼                    ▼                       ▼
 *   DataPropertyMapper   ObjectPropertyMapper   TaxonomyPropertyMapper
 *         │                    │                       │
 *         └────────────────────┴───────────────────────┘
 *                              ↓
 *                      OpenAPI ObjectSchema
 * </pre>
 *
 * <h2>Annotation Filtering</h2>
 * <p>Only axioms annotated with {@code apiLabel} matching the schema label from
 * {@link MappingContext#getSchemaLabel()} are included in the generated schema.
 *
 * <h2>Thread Safety</h2>
 * <p>This class is <b>not thread-safe</b>. Each instance should be used to map
 * a single OWL class and then discarded.
 *
 * @see DataPropertyMapper
 * @see ObjectPropertyMapper
 * @see TaxonomyPropertyMapper
 * @see SchemaVisitor
 */
```

#### Method Javadocs for Extracted Methods

```java
/**
 * Creates and configures the base ObjectSchema with standard properties.
 *
 * @return a new ObjectSchema configured with name, type, description, and OpenAPI 3.1 spec version
 */
private Schema<?> initializeResourceSchema()

/**
 * Creates paired SchemaVisitors for analyzing object and data property restrictions.
 *
 * <p>Both visitors traverse the same class hierarchy but collect different types
 * of restrictions:
 * <ul>
 *   <li>Object visitor: collects SomeValuesFrom, AllValuesFrom, cardinality on object properties</li>
 *   <li>Data visitor: collects DataSomeValuesFrom, DataAllValuesFrom, cardinality on data properties</li>
 * </ul>
 *
 * @param analyzedClass the OWL class to analyze
 * @return a record containing both configured visitors
 */
private VisitorPair createSchemaVisitors(OWLClass analyzedClass)

/**
 * Collects all relevant axioms for the analyzed class from the ontology.
 *
 * <p>Gathers three categories of axioms:
 * <ul>
 *   <li><b>Taxonomy axioms</b>: SubClassOf axioms referencing named individuals (enum values)</li>
 *   <li><b>SubClass axioms</b>: Direct SubClassOf axioms for the analyzed class</li>
 *   <li><b>SuperClass axioms</b>: Inherited axioms from the class hierarchy</li>
 * </ul>
 *
 * @param analyzedClass the OWL class to collect axioms for
 * @return a record containing categorized axiom lists
 */
private AxiomCollection collectRelevantAxioms(OWLClass analyzedClass)

/**
 * Merges typed restrictions from both schema visitors into a unified map.
 *
 * <p>Combines restrictions keyed by property name. A single property may have
 * multiple restrictions (e.g., both minCardinality and maxCardinality).
 *
 * @param objectVisitor visitor containing object property restrictions
 * @param dataVisitor visitor containing data property restrictions
 * @return map of property name → set of RestrictionInfo for that property
 */
private Map<String, Set<RestrictionInfo>> mergeVisitorRestrictions(
    SchemaVisitor objectVisitor, SchemaVisitor dataVisitor)

/**
 * Finds a data property in the ontology by its resolved name.
 *
 * @param propertyName the short-form property name (e.g., "firstName")
 * @return the matching OWLDataProperty, or empty if not found
 */
private Optional<OWLDataProperty> findDataProperty(String propertyName)

/**
 * Finds an object property in the ontology by its resolved name.
 *
 * @param propertyName the short-form property name (e.g., "hasAddress")
 * @return the matching OWLObjectProperty, or empty if not found
 */
private Optional<OWLObjectProperty> findObjectProperty(String propertyName)

/**
 * Extracts range type/class names for a property from multiple sources.
 *
 * <p>Ranges are collected from:
 * <ol>
 *   <li>The SchemaVisitor's property range multimap (from restriction fillers)</li>
 *   <li>The RestrictionInfo objects themselves (via getRangeNames)</li>
 * </ol>
 *
 * @param propertyName the property to extract ranges for
 * @param rangesFromVisitor multimap of property → range classes/datatypes from visitor
 * @param restrictions map of property → RestrictionInfo set
 * @return deduplicated list of range type names
 */
private List<String> extractPropertyRanges(...)

/**
 * Extracts the example value for a data property from annotations.
 *
 * <p>Looks for skos:example annotations in two places (in order):
 * <ol>
 *   <li>On the SubClassOf axiom containing the property restriction</li>
 *   <li>On the data property itself (as a default)</li>
 * </ol>
 *
 * @param property the data property to find an example for
 * @param subClassAxioms axioms that may contain restriction-level annotations
 * @return the example value string, or null if none found
 */
private String extractExampleValue(OWLDataProperty property, List<OWLSubClassOfAxiom> subClassAxioms)

/**
 * Processes all data properties and adds them to the schema.
 *
 * <p>For each data property discovered by the SchemaVisitor:
 * <ol>
 *   <li>Looks up the OWLDataProperty in the ontology</li>
 *   <li>Extracts description, label, example, ranges, and enum values</li>
 *   <li>Creates a DataPropertyMapper to generate the schema</li>
 *   <li>Adds the property to the resource schema</li>
 *   <li>Marks as required if restrictions indicate requiredness</li>
 * </ol>
 *
 * @param schema the resource schema to add properties to
 * @param propertyNames set of data property names from the visitor
 * @param restrictions merged restrictions map
 * @param dataRanges multimap of property → OWLDatatype ranges
 * @param subClassAxioms axioms for extracting example annotations
 */
private void processDataProperties(...)

/**
 * Processes all object properties and adds them to the schema.
 *
 * <p>For each object property discovered by the SchemaVisitor:
 * <ol>
 *   <li>Looks up the OWLObjectProperty in the ontology</li>
 *   <li>Extracts description, label, and range classes</li>
 *   <li>Creates an ObjectPropertyMapper to generate the schema</li>
 *   <li>Adds the property to the resource schema</li>
 *   <li>Marks as required if restrictions indicate requiredness</li>
 * </ol>
 *
 * @param schema the resource schema to add properties to
 * @param propertyNames set of object property names from the visitor
 * @param restrictions merged restrictions map
 * @param objectRanges multimap of property → OWLClass ranges
 */
private void processObjectProperties(...)

/**
 * Processes taxonomy axioms and adds enum properties to the schema.
 *
 * <p>Taxonomy properties are object properties whose range is a set of named
 * individuals (enum values). These are mapped to StringSchema with enum constraints.
 *
 * <p>Two patterns are supported:
 * <ul>
 *   <li><b>Direct individuals</b>: The restriction directly references named individuals</li>
 *   <li><b>Class-based</b>: The restriction references a class, and individuals are
 *       looked up via EntitySearcher</li>
 * </ul>
 *
 * @param schema the resource schema to add taxonomy properties to
 * @param taxonomyAxioms axioms identified as taxonomy definitions
 */
private void processTaxonomyProperties(Schema<?> schema, List<OWLSubClassOfAxiom> taxonomyAxioms)

/**
 * Logs a warning when no properties were found for a class.
 *
 * <p>This typically indicates that the ontology is missing apiLabel annotations
 * on the property restrictions, or that the schema label doesn't match.
 *
 * @param analyzedClass the class that had no mappable properties
 * @param axioms all collected axioms (for diagnostic output)
 */
private void logMissingPropertiesWarning(OWLClass analyzedClass, AxiomCollection axioms)
```

#### Inline Comments for Complex Logic

Add inline comments for:
- The axiom filtering logic (why we filter on apiLabel)
- The taxonomy detection heuristic (individuals vs class lookup)
- The example value lookup order (restriction annotation → property annotation)
- The visitor application pattern (why we check property signatures)

Example inline comments:
```java
// Filter axioms to only those annotated with the target apiLabel
// This allows multiple API schemas to be generated from the same ontology
subClassOfAxioms.stream()
    .filter(WithAnnotation.of(OntologyConstants.apiLabel, schemaLabel))
    ...

// Taxonomy detection: axioms with named individuals are taxonomy enums
// These map to StringSchema with enum values rather than object references
if (!individualsInSignature.isEmpty()) {
    // Direct individual reference - use individual labels as enum values
    ...
} else {
    // Indirect reference via class - look up individuals belonging to the class
    ...
}
```

## Target Structure

After refactoring, `createSchema()` should read as:

```java
private Schema createSchema(OWLClass analyzedClass) {
    Schema<?> schema = initializeResourceSchema();

    var visitors = createSchemaVisitors(analyzedClass);
    var axioms = collectRelevantAxioms(analyzedClass);

    visitAxiomsWithVisitors(visitors, axioms);

    var restrictions = mergeVisitorRestrictions(visitors);

    processDataProperties(schema, visitors.data(), restrictions, axioms.subClass());
    processObjectProperties(schema, visitors.object(), restrictions);
    processTaxonomyProperties(schema, axioms.taxonomy());

    if (schema.getProperties() == null || schema.getProperties().isEmpty()) {
        logMissingPropertiesWarning(analyzedClass, axioms);
    }

    return schema;
}
```

**Target: ~30 lines for createSchema(), down from 228**

## Execution Order

1. **Stage 1** - Extract 5 simple methods (low risk)
2. Run tests: `mvn test`
3. **Stage 2** - Extract property lookup helpers
4. Run tests
5. **Stage 3** - Extract range extraction helper
6. Run tests
7. **Stage 4** - Extract example value extraction
8. Run tests
9. **Stage 5** - Extract property processing methods
10. Run tests
11. **Stage 6** - Rename variables for clarity
12. Run tests
13. **Stage 7** - Add Javadoc and inline comments
14. Run full test suite: `mvn test`

## Verification

After each stage:
1. Run `mvn test` - all 208 tests must pass
2. Run `mvn verify` - integration tests must pass
3. Verify generated OpenAPI specs are unchanged (diff output files)

## Risk Mitigation

- **Small commits**: One stage per commit allows easy rollback
- **Test after each stage**: Catch regressions immediately
- **Preserve behavior**: No functional changes, only structural
- **Keep original method signatures**: Public API unchanged

## Out of Scope

- Changes to other mapper classes
- New functionality
- Performance optimizations
- Adding unit tests for ResourceSchemaMapper (covered by integration tests)
