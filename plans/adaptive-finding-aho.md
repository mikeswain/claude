# Fix: Union Ranges & Inherited Data Properties in API Generation

## Context

Two bugs prevent full API coverage for ontologies with inherited properties and polymorphic ranges:

1. **Union ranges in cardinality restrictions** produce empty schemas because `SchemaVisitor.setPropertiesWithCardinality()` only handles single-class fillers, not `OWLObjectUnionOf`
2. **Inherited data properties** (e.g., `nameText` on `:Person`) are missing from subclass schemas because `ResourceSchemaMapper.getSuperClassAxioms()` filters all axioms by `apiLabel` annotation — abstract superclasses don't carry that annotation

## Fix 1: Union fillers in cardinality restrictions

**File:** `src/main/java/com/grl/component/schema/SchemaVisitor.java` — `setPropertiesWithCardinality()` (line 348)

**Root cause:** Line 348 checks `ce.getFiller().isOWLClass()` which returns false for `OWLObjectUnionOf`, so no range classes are added to `objectPropertyRanges`.

**Why not use visitor delegation:** The `setBooleanCombinationObjectProperties()` else branch (line 384-394) when `propertyName` is already set does NOT add single-class operands to `objectPropertyRanges` — it has a no-op path at line 389-390. Delegating via `accept(this)` would silently lose the ranges.

**Fix:** Extract union members directly from the filler:

```java
// Replace lines 348-350
if (ce.getFiller().isOWLClass()) {
    if (!ce.getFiller().asOWLClass().equals(owlThing)) {
        objectPropertyRanges.put(propertyName, ce.getFiller().asOWLClass());
    }
} else if (ce.getFiller() instanceof OWLObjectUnionOf unionOf) {
    for (OWLClassExpression operand : unionOf.getOperands()) {
        if (operand.isOWLClass() && !operand.asOWLClass().equals(owlThing)) {
            objectPropertyRanges.put(propertyName, operand.asOWLClass());
        }
    }
}
```

**Tests:**
- Unit test in `SchemaVisitorTest`: cardinality restriction with union filler — verify all union members appear in `objectPropertyRanges`
- Integration test: ontology with `maxQualifiedCardinality + unionOf` filler — verify generated schema has properties for each union member

## Fix 2: Inherited data properties from superclasses

**File:** `src/main/java/com/grl/component/mapper/ResourceSchemaMapper.java` — `getSuperClassAxioms()` (line 662)

**Root cause:** Line 662-663 filters all axioms by `apiLabel` annotation. Data properties on abstract superclasses (like `:Person`) don't carry this annotation, so they're excluded.

**Why data properties only:** Object properties from superclasses represent structural relationships that may not be appropriate for every subclass. Data properties are simple scalars (names, dates) that are almost always wanted.

**Fix:** Include unannotated axioms if they are data property restrictions:

```java
// Replace lines 662-664
boolean hasApiLabel = owlSubClassOfAxiom.annotations()
    .anyMatch(a -> a.getValue().toString().equals(schemaAnnotation));
boolean isInheritedDataProperty = !hasApiLabel
    && superClass instanceof OWLDataRestriction;
if (hasApiLabel || isInheritedDataProperty) {
    superClassAxioms.add(owlSubClassOfAxiom);
}
```

**Tests:**
- Unit test: verify data property from superclass without `apiLabel` is collected
- Integration test: class whose superclass has unannotated data properties — verify they appear in schema
- Regression: run existing tests to verify no unwanted properties leak

## Execution Order

1. Fix 1 (union fillers) — smaller change, isolated to one method
2. Fix 2 (inherited data properties) — broader impact, needs regression pass
3. `anyOf` improvement deferred — current multi-range output via `getComposedSchemaObject()` works

## Verification

```bash
mvn clean verify   # All unit + integration tests pass (~89 tests)
```

## Files to Modify

- `src/main/java/com/grl/component/schema/SchemaVisitor.java` (lines 348-350)
- `src/main/java/com/grl/component/mapper/ResourceSchemaMapper.java` (lines 662-664)
- `src/test/java/com/grl/component/schema/SchemaVisitorTest.java` (new tests)
- `src/test/resources/regression/ontologies/cardinality-test.owl` (add union filler test data)
- New integration test or extend `CardinalityRestrictionsIT` for union filler
- New test ontology + config + integration test for inherited data properties
