# SchemaVisitor Refactoring Plan

## Step 1: Encapsulation & cleanup fixes
**Files:** `SchemaVisitor.java`

- Make `propertyName`, `owlThing`, `range` fields `private`
- Remove unused `range` field and constructor parameter entirely (never read after assignment)
- Change `Boolean range` → removed from constructor, update all call sites
- Make logger `private static final`
- Inline `cleanName()` — it just calls `Objects.requireNonNull()`, name is misleading

**Call sites to update (remove `range` param):**
- `ResourceSchemaMapper.java:200-201` (2 calls)
- `SchemaVisitorTest.java` (~30 calls)
- `SchemaVisitorTaxonomyDetectionTest.java` (~17 calls)

## Step 2: Eliminate VisitorPair — use single SchemaVisitor
**Files:** `ResourceSchemaMapper.java`, `SchemaVisitor.java`

The mapper creates two identical visitors and dispatches object-property axioms to one and data-property axioms to the other. This is unnecessary — `SchemaVisitor` already separates `objectPropertyNames`/`dataPropertyNames` internally, and the OWL visitor dispatch is type-safe (`visit(OWLObjectSomeValuesFrom)` vs `visit(OWLDataSomeValuesFrom)`).

- Delete the `VisitorPair` record
- Use a single `SchemaVisitor` instance
- `visitSuperClass()` just calls `superClass.accept(visitor)` unconditionally
- `mergeVisitorRestrictions()` simplifies to `visitor.getRestrictions()`
- `processDataProperties()` and `processObjectProperties()` both use the same visitor
- Taxonomy patterns: single `visitor.getTaxonomyPatterns()` call

## Step 3: Deduplicate SomeValuesFrom / AllValuesFrom visit methods
**File:** `SchemaVisitor.java`

`visit(OWLObjectSomeValuesFrom)` and `visit(OWLObjectAllValuesFrom)` are nearly identical (~25 lines each). Same for the data variants. Extract shared logic:

```java
private void handleObjectValuesFrom(OWLClassExpression ce,
    OWLClassExpression filler, RestrictionInfo info) {
  // resolve property, add to names, store restriction, detect taxonomy, handle filler
}
```

Similarly consolidate `setPropertiesWithCardinality` and `setDataPropertiesWithCardinality` into one method where possible.

## Verification
- `mvn clean test` — all 258+ unit tests pass
- `mvn clean verify` — integration tests pass (if submodule available)
