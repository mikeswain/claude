# Introduction
We are adding some new patterns for defining taxonomies, which may be more complex than just the string enums we currently support.

## Modifications
- ResourceSchemaMapper should recognize these patterns and use a new TaxonomyPatternMapper to generate the schemas
- Recognition uses OWL classes and the SchemaVisitor (which may need extending). The SPARQL examples are just a guide.
- Modify ResourceSchemaMapper in keeping with its current structure
- Add unit and integration tests following current patterns

---

# Implementation Plan

## Phase 1: Extend SchemaVisitor for Pattern Detection

### 1.1 Add TaxonomyPattern enum
**File:** `src/main/java/com/grl/component/schema/TaxonomyPattern.java` (new)

```java
public enum TaxonomyPattern {
    PATTERN_A,  // Inline enum (untyped individuals)
    PATTERN_B,  // Typed individuals with metadata
    PATTERN_C,  // Class-based taxonomy (extensible)
    PATTERN_D,  // SKOS taxonomy
    NONE        // Not a taxonomy pattern
}
```

### 1.2 Add TaxonomyInfo record
**File:** `src/main/java/com/grl/component/schema/TaxonomyInfo.java` (new)

Record to hold detected taxonomy information:
- `TaxonomyPattern pattern`
- `List<OWLNamedIndividual> individuals` (for A/B)
- `OWLClass targetClass` (for C/D)
- `boolean hasHierarchy` (for C)
- `OWLIndividual conceptScheme` (for D - the skos:ConceptScheme)

### 1.3 Extend RestrictionInfo
**File:** `src/main/java/com/grl/component/schema/RestrictionInfo.java`

Add method to detect taxonomy patterns from OneOf/SomeValuesFrom restrictions:
- `Optional<TaxonomyInfo> detectTaxonomyPattern(OWLOntology ontology)`

### 1.4 Add detection logic to SchemaVisitor
**File:** `src/main/java/com/grl/component/schema/SchemaVisitor.java`

Add new tracking:
- `Map<String, TaxonomyInfo> taxonomyPatterns` - detected patterns per property
- Detection in `visit(OWLObjectSomeValuesFrom)` and `visit(OWLObjectAllValuesFrom)`:
  - If filler is blank node with owl:oneOf → Pattern A or B
  - If filler is named class → check for SKOS (D) or class-based (C)

**Detection Logic (OWL API equivalent of SPARQL):**
```java
private TaxonomyPattern detectPattern(OWLClassExpression filler, OWLOntology ontology) {
    // Step 1: Is target a blank node with owl:oneOf?
    if (filler.isAnonymous() && filler instanceof OWLObjectOneOf oneOf) {
        List<OWLNamedIndividual> individuals = oneOf.individuals().toList();
        // Check if individuals have domain class beyond owl:NamedIndividual
        boolean hasTypedIndividuals = individuals.stream()
            .anyMatch(ind -> hasDomainClass(ind, ontology));
        return hasTypedIndividuals ? PATTERN_B : PATTERN_A;
    }

    // Step 2: Is target a named class?
    if (filler.isNamed() && filler instanceof OWLClass targetClass) {
        // Check for SKOS
        if (isSubClassOfSkosConcept(targetClass, ontology) ||
            hasInSchemeRestriction(targetClass, ontology)) {
            return PATTERN_D;
        }
        // Check for class-based taxonomy (no object property restrictions)
        if (!hasObjectPropertyRestrictions(targetClass, ontology)) {
            return PATTERN_C;
        }
    }
    return NONE;
}
```

## Phase 2: Create TaxonomyPatternMapper

### 2.1 New mapper class
**File:** `src/main/java/com/grl/component/mapper/TaxonomyPatternMapper.java` (new)

Generates OpenAPI schemas based on detected pattern:

```java
public class TaxonomyPatternMapper {
    public Schema<?> mapTaxonomy(TaxonomyInfo info, MappingContext context) {
        return switch (info.pattern()) {
            case PATTERN_A -> mapInlineEnum(info);
            case PATTERN_B -> mapTypedEnum(info);
            case PATTERN_C -> mapClassBasedTaxonomy(info, context);
            case PATTERN_D -> mapSkosTaxonomy(info, context);
            case NONE -> throw new IllegalArgumentException("Not a taxonomy");
        };
    }
}
```

**Pattern A output:** Inline StringSchema with enum
**Pattern B output:** StringSchema with enum only (string values from codeValue/label, no x-code-metadata)
**Pattern C output:** $ref to schema in components/schemas with `x-extensible: true` and optional `x-hierarchy`
**Pattern D output:** $ref to Coding schema in components/schemas with `x-fhir-valueset`, `x-skos-hierarchy`

**Decision Notes:**
- Pattern B: String enum values only, no metadata extension
- All 4 patterns implemented together
- Patterns C/D generate separate schemas in components/schemas (reusable)

## Phase 3: Integrate into ResourceSchemaMapper

### 3.1 Modify processTaxonomyProperties()
**File:** `src/main/java/com/grl/component/mapper/ResourceSchemaMapper.java`

Current flow:
1. `getTaxonomyAxioms()` identifies axioms with named individuals
2. `addTaxonomyPropertyToSchema()` creates enum from individual labels

New flow:
1. After `mergeVisitorRestrictions()`, check for `taxonomyPatterns` from visitors
2. For each detected taxonomy pattern, use `TaxonomyPatternMapper` instead of existing `TaxonomyPropertyMapper`
3. Keep existing flow as fallback for simple cases

### 3.2 Add helper methods
- `extractTaxonomyPatterns(VisitorPair visitors)` - collect taxonomy info from both visitors
- Integrate with existing required/array detection logic

## Phase 4: Add SKOS Constants

### 4.1 Extend OntologyConstants
**File:** `src/main/java/com/grl/component/owl/OntologyConstants.java`

Add SKOS IRIs:
- `SKOS_CONCEPT` = skos:Concept
- `SKOS_CONCEPT_SCHEME` = skos:ConceptScheme
- `SKOS_IN_SCHEME` = skos:inScheme
- `SKOS_BROADER` = skos:broader
- `SKOS_NARROWER` = skos:narrower
- `SKOS_NOTATION` = skos:notation
- `SKOS_COLLECTION` = skos:Collection

## Phase 5: Unit Tests

### 5.1 TaxonomyPatternMapperTest
**File:** `src/test/java/com/grl/component/mapper/TaxonomyPatternMapperTest.java` (new)

Test nested classes:
- `PatternATests` - inline enum generation
- `PatternBTests` - typed enum with metadata extraction
- `PatternCTests` - class-based schema with $ref and x-hierarchy
- `PatternDTests` - SKOS Coding schema generation

### 5.2 SchemaVisitorTaxonomyDetectionTest
**File:** `src/test/java/com/grl/component/schema/SchemaVisitorTaxonomyDetectionTest.java` (new)

Test pattern detection logic:
- Detect Pattern A (blank node, untyped individuals)
- Detect Pattern B (blank node, typed individuals)
- Detect Pattern C (named class, no restrictions)
- Detect Pattern D (SKOS subclass or inScheme)
- Distinguish C from domain class (has object property restrictions)

## Phase 6: Integration Tests

### 6.1 Test ontology
**File:** `src/test/resources/regression/ontologies/taxonomy-patterns.owl` (new)

Contains examples of all 4 patterns with variations (someValuesFrom, allValuesFrom, cardinality)

### 6.2 Config file
**File:** `src/test/resources/regression/configs/config_taxonomy_patterns.yaml` (new)

### 6.3 Integration test class
**File:** `src/test/java/regression/features/properties/TaxonomyPatternsIT.java` (new)

Tests using JSONPath:
- Pattern A generates inline enum
- Pattern B generates enum with string values
- Pattern C generates $ref with x-extensible
- Pattern C with hierarchy generates x-hierarchy
- Pattern D generates Coding schema with x-fhir-valueset

## Files to Create/Modify

| File | Action |
|------|--------|
| `TaxonomyPattern.java` | Create |
| `TaxonomyInfo.java` | Create |
| `TaxonomyPatternMapper.java` | Create |
| `SchemaVisitor.java` | Modify (add taxonomy detection) |
| `RestrictionInfo.java` | Modify (add taxonomy helper methods) |
| `ResourceSchemaMapper.java` | Modify (integrate TaxonomyPatternMapper) |
| `OntologyConstants.java` | Modify (add SKOS constants) |
| `TaxonomyPatternMapperTest.java` | Create |
| `SchemaVisitorTaxonomyDetectionTest.java` | Create |
| `taxonomy-patterns.owl` | Create |
| `config_taxonomy_patterns.yaml` | Create |
| `TaxonomyPatternsIT.java` | Create |

## Verification

1. Run unit tests: `mvn test -Dtest=TaxonomyPatternMapperTest,SchemaVisitorTaxonomyDetectionTest`
2. Run integration tests: `mvn verify -Dtest=TaxonomyPatternsIT`
3. Run full test suite: `mvn clean verify`
4. Manual verification: Generate spec from test ontology and inspect output


# GRL Generator: Taxonomy Pattern Reference

This document provides a complete reference for taxonomy patterns supported by the GRL Generator (ASG), including discriminators for pattern detection, OWL variations, and expected OpenAPI outputs.

---

## Pattern Summary

| Pattern | Description | Discriminator | OpenAPI Output |
|---------|-------------|---------------|----------------|
| A | Inline Enumeration (untyped) | Blank node with `owl:oneOf`; individuals have no domain class | Inline `enum` |
| B | Typed Individuals | Blank node with `owl:oneOf`; individuals have domain class + metadata | `enum` with string values |
| C | Class-based Taxonomy | Named class IRI target; no `owl:oneOf`; no object property restrictions; flat or hierarchical | `$ref` to extensible schema (with `x-hierarchy` if hierarchical) |
| D | SKOS Taxonomy | Class is `subClassOf skos:Concept` or has `skos:inScheme` restriction | `$ref` to Coding schema |

---

## Pattern A: Inline Enumeration (No Type)

### Discriminator

The restriction target (`owl:onClass`, `owl:someValuesFrom`, or `owl:allValuesFrom`) is a **blank node** (anonymous class) containing `owl:oneOf`. The individuals referenced in the `owl:oneOf` list have no domain-specific class—they are just `owl:NamedIndividual`.

The restriction can be attached via either `rdfs:subClassOf` or `owl:equivalentClass`.

**ASG detection (SPARQL):**
```sparql
# Target is blank node with oneOf, via either subClassOf or equivalentClass
{
    ?class rdfs:subClassOf ?restriction .
} UNION {
    ?class owl:equivalentClass ?restriction .
}
?restriction owl:onProperty ?prop .
?restriction (owl:someValuesFrom|owl:allValuesFrom|owl:onClass) ?target .
FILTER(isBlank(?target))
?target owl:oneOf ?list .
```

### Individuals Definition

```turtle
sensor:Active a owl:NamedIndividual ; rdfs:label "Active" .
sensor:Cancelled a owl:NamedIndividual ; rdfs:label "Cancelled" .
sensor:Hold a owl:NamedIndividual ; rdfs:label "Hold" .
```

### All Variations

```turtle
sensor:PatternA rdf:type owl:Class ;
    rdfs:subClassOf
        # A1: Required array (1..*) via someValuesFrom
        [ rdf:type owl:Restriction ;
          owl:onProperty sensor:isCategorisedBy ;
          owl:someValuesFrom [ rdf:type owl:Class ;
                               owl:oneOf ( sensor:Active
                                           sensor:Cancelled )
                             ]
        ] ,
        # A2: Type constraint (0..*) via allValuesFrom
        [ rdf:type owl:Restriction ;
          owl:onProperty sensor:isCategorisedBy ;
          owl:allValuesFrom [ rdf:type owl:Class ;
                              owl:oneOf ( sensor:Active
                                          sensor:Cancelled )
                            ]
        ] ,
        # A3: Required array (1..*) via minQualifiedCardinality
        [ rdf:type owl:Restriction ;
          owl:onProperty sensor:isCategorisedBy ;
          owl:minQualifiedCardinality "1"^^xsd:nonNegativeInteger ;
          owl:onClass [ rdf:type owl:Class ;
                        owl:oneOf ( sensor:Active
                                    sensor:Cancelled )
                      ]
        ] ,
        # A4: Required single (1..1) via qualifiedCardinality
        [ rdf:type owl:Restriction ;
          owl:onProperty sensor:isCategorisedBy ;
          owl:qualifiedCardinality "1"^^xsd:nonNegativeInteger ;
          owl:onClass [ rdf:type owl:Class ;
                        owl:oneOf ( sensor:Active
                                    sensor:Cancelled
                                    sensor:Hold )
                      ]
        ] ,
        # A5: Bounded array (0..3) via maxQualifiedCardinality
        [ rdf:type owl:Restriction ;
          owl:onProperty sensor:isCategorisedBy ;
          owl:maxQualifiedCardinality "3"^^xsd:nonNegativeInteger ;
          owl:onClass [ rdf:type owl:Class ;
                        owl:oneOf ( sensor:Active
                                    sensor:Cancelled )
                      ]
        ] ;
    skos:prefLabel "Pattern A" .
```

### Variation A1: Required Array (1..*) via someValuesFrom

**OWL:**
```turtle
owl:someValuesFrom [ rdf:type owl:Class ;
                     owl:oneOf ( sensor:Active sensor:Cancelled ) ]
```

**OpenAPI:**
```yaml
PatternA:
  type: object
  required:
    - isCategorisedBy
  properties:
    isCategorisedBy:
      type: array
      minItems: 1
      items:
        type: string
        enum: [Active, Cancelled]
```

### Variation A2: Type Constraint / Optional Array (0..*) via allValuesFrom

**OWL:**
```turtle
owl:allValuesFrom [ rdf:type owl:Class ;
                    owl:oneOf ( sensor:Active sensor:Cancelled ) ]
```

**OpenAPI:**
```yaml
PatternA:
  type: object
  properties:
    isCategorisedBy:
      type: array
      items:
        type: string
        enum: [Active, Cancelled]
```

### Variation A3: Required Array (1..*) via minQualifiedCardinality

**OWL:**
```turtle
owl:minQualifiedCardinality "1"^^xsd:nonNegativeInteger ;
owl:onClass [ rdf:type owl:Class ;
              owl:oneOf ( sensor:Active sensor:Cancelled ) ]
```

**OpenAPI:**
```yaml
PatternA:
  type: object
  required:
    - isCategorisedBy
  properties:
    isCategorisedBy:
      type: array
      minItems: 1
      items:
        type: string
        enum: [Active, Cancelled]
```

### Variation A4: Required Single (1..1) via qualifiedCardinality

**OWL:**
```turtle
owl:qualifiedCardinality "1"^^xsd:nonNegativeInteger ;
owl:onClass [ rdf:type owl:Class ;
              owl:oneOf ( sensor:Active sensor:Cancelled sensor:Hold ) ]
```

**OpenAPI:**
```yaml
PatternA:
  type: object
  required:
    - isCategorisedBy
  properties:
    isCategorisedBy:
      type: string
      enum: [Active, Cancelled, Hold]
```

### Variation A5: Bounded Array (0..3) via maxQualifiedCardinality

**OWL:**
```turtle
owl:maxQualifiedCardinality "3"^^xsd:nonNegativeInteger ;
owl:onClass [ rdf:type owl:Class ;
              owl:oneOf ( sensor:Active sensor:Cancelled ) ]
```

**OpenAPI:**
```yaml
PatternA:
  type: object
  properties:
    isCategorisedBy:
      type: array
      maxItems: 3
      items:
        type: string
        enum: [Active, Cancelled]
```

### Variation A6: Optional Single (0..1) via maxQualifiedCardinality 1

**OWL:**
```turtle
owl:maxQualifiedCardinality "1"^^xsd:nonNegativeInteger ;
owl:onClass [ rdf:type owl:Class ;
              owl:oneOf ( sensor:Active sensor:Cancelled ) ]
```

**OpenAPI:**
```yaml
PatternA:
  type: object
  properties:
    isCategorisedBy:
      type: string
      enum: [Active, Cancelled]
```

### Variation A7: Via owl:equivalentClass (instead of rdfs:subClassOf)

The restriction can be attached using `owl:equivalentClass` rather than `rdfs:subClassOf`. This changes the semantics (necessary AND sufficient vs necessary only) but the ASG traversal and output are identical.

**OWL:**
```turtle
sensor:PatternA rdf:type owl:Class ;
    owl:equivalentClass [ rdf:type owl:Restriction ;
                          owl:onProperty sensor:isCategorisedBy ;
                          owl:someValuesFrom [ rdf:type owl:Class ;
                                               owl:oneOf ( sensor:Active
                                                           sensor:Cancelled )
                                             ]
                        ] .
```

**OpenAPI:**
```yaml
PatternA:
  type: object
  required:
    - isCategorisedBy
  properties:
    isCategorisedBy:
      type: array
      minItems: 1
      items:
        type: string
        enum: [Active, Cancelled]
```

> **Note:** This variation applies equally to Pattern B. The `owl:equivalentClass` wrapper can be used with any of the restriction types (someValuesFrom, allValuesFrom, qualified cardinality, etc.).

---

## Pattern B: Typed Individuals (Class Enumeration)

### Discriminator

The restriction structure is **identical to Pattern A**—a blank node containing `owl:oneOf`. The distinction is in the **individuals themselves**:

| Pattern | Individual typing | Metadata |
|---------|-------------------|----------|
| A | `owl:NamedIndividual` only | Just `rdfs:label` |
| B | `owl:NamedIndividual` + domain class | `codeValue`, `displayName`, `definition`, etc. |

The restriction can be attached via either `rdfs:subClassOf` or `owl:equivalentClass` (see Pattern A, Variation A7).

**ASG detection:**
```sparql
# After extracting individuals from owl:oneOf list:
SELECT ?individual ?domainClass WHERE {
    ?individual a ?domainClass .
    FILTER(?domainClass != owl:NamedIndividual)
}
# Results? → Pattern B
# No results? → Pattern A
```

### Individuals Definition

```turtle
sensor:_sensor_type_analog rdf:type owl:NamedIndividual ,
                                    sensor:SensorClass ;
    sensor:hasSuperCategory sensor:_sensor_type ;
    sensor:name "analog" ;
    sensor:codeValue "analog" ;
    sensor:displayName "Analog" ;
    skos:definition "Analog sensor type." ;
    skos:prefLabel "analog" ;
    www:apiLabel "SensorAPIV1.0.0" .

sensor:_sensor_type_digital rdf:type owl:NamedIndividual ,
                                     sensor:SensorClass ;
    sensor:hasSuperCategory sensor:_sensor_type ;
    sensor:name "digital" ;
    sensor:codeValue "digital" ;
    sensor:displayName "Digital" ;
    skos:definition "Digital sensor type." ;
    skos:prefLabel "digital" ;
    www:apiLabel "SensorAPIV1.0.0" .
```

### All Variations

```turtle
sensor:PatternB rdf:type owl:Class ;
    rdfs:subClassOf
        # B1: Required array (1..*) via someValuesFrom
        [ rdf:type owl:Restriction ;
          owl:onProperty sensor:isCategorisedBy ;
          owl:someValuesFrom [ rdf:type owl:Class ;
                               owl:oneOf ( sensor:_sensor_type_analog
                                           sensor:_sensor_type_digital )
                             ]
        ] ,
        # B2: Type constraint (0..*) via allValuesFrom
        [ rdf:type owl:Restriction ;
          owl:onProperty sensor:isCategorisedBy ;
          owl:allValuesFrom [ rdf:type owl:Class ;
                              owl:oneOf ( sensor:_sensor_type_analog
                                          sensor:_sensor_type_digital )
                            ]
        ] ,
        # B3: Required array (1..*) via minQualifiedCardinality
        [ rdf:type owl:Restriction ;
          owl:onProperty sensor:isCategorisedBy ;
          owl:minQualifiedCardinality "1"^^xsd:nonNegativeInteger ;
          owl:onClass [ rdf:type owl:Class ;
                        owl:oneOf ( sensor:_sensor_type_analog
                                    sensor:_sensor_type_digital )
                      ]
        ] ,
        # B4: Required single (1..1) via qualifiedCardinality
        [ rdf:type owl:Restriction ;
          owl:onProperty sensor:isCategorisedBy ;
          owl:qualifiedCardinality "1"^^xsd:nonNegativeInteger ;
          owl:onClass [ rdf:type owl:Class ;
                        owl:oneOf ( sensor:_sensor_type_analog
                                    sensor:_sensor_type_digital )
                      ]
        ] ,
        # B5: Optional single (0..1) via maxQualifiedCardinality
        [ rdf:type owl:Restriction ;
          owl:onProperty sensor:isCategorisedBy ;
          owl:maxQualifiedCardinality "1"^^xsd:nonNegativeInteger ;
          owl:onClass [ rdf:type owl:Class ;
                        owl:oneOf ( sensor:_sensor_type_analog
                                    sensor:_sensor_type_digital )
                      ]
        ] ;
    skos:prefLabel "Pattern B" .
```

### Variation B1: Required Array (1..*) via someValuesFrom

**OWL:**
```turtle
owl:someValuesFrom [ rdf:type owl:Class ;
                     owl:oneOf ( sensor:_sensor_type_analog
                                 sensor:_sensor_type_digital ) ]
```

**OpenAPI:**
```yaml
PatternB:
  type: object
  required:
    - isCategorisedBy
  properties:
    isCategorisedBy:
      type: array
      minItems: 1
      items:
        type: string
        enum: [analog, digital]
      x-code-metadata:
        analog:
          displayName: Analog
          definition: Analog sensor type.
        digital:
          displayName: Digital
          definition: Digital sensor type.
```

### Variation B2: Type Constraint / Optional Array (0..*) via allValuesFrom

**OWL:**
```turtle
owl:allValuesFrom [ rdf:type owl:Class ;
                    owl:oneOf ( sensor:_sensor_type_analog
                                sensor:_sensor_type_digital ) ]
```

**OpenAPI:**
```yaml
PatternB:
  type: object
  properties:
    isCategorisedBy:
      type: array
      items:
        type: string
        enum: [analog, digital]
      x-code-metadata:
        analog:
          displayName: Analog
          definition: Analog sensor type.
        digital:
          displayName: Digital
          definition: Digital sensor type.
```

### Variation B3: Required Array (1..*) via minQualifiedCardinality

**OWL:**
```turtle
owl:minQualifiedCardinality "1"^^xsd:nonNegativeInteger ;
owl:onClass [ rdf:type owl:Class ;
              owl:oneOf ( sensor:_sensor_type_analog
                          sensor:_sensor_type_digital ) ]
```

**OpenAPI:**
```yaml
PatternB:
  type: object
  required:
    - isCategorisedBy
  properties:
    isCategorisedBy:
      type: array
      minItems: 1
      items:
        type: string
        enum: [analog, digital]
      x-code-metadata:
        analog:
          displayName: Analog
          definition: Analog sensor type.
        digital:
          displayName: Digital
          definition: Digital sensor type.
```

### Variation B4: Required Single (1..1) via qualifiedCardinality

**OWL:**
```turtle
owl:qualifiedCardinality "1"^^xsd:nonNegativeInteger ;
owl:onClass [ rdf:type owl:Class ;
              owl:oneOf ( sensor:_sensor_type_analog
                          sensor:_sensor_type_digital ) ]
```

**OpenAPI:**
```yaml
PatternB:
  type: object
  required:
    - isCategorisedBy
  properties:
    isCategorisedBy:
      type: string
      enum: [analog, digital]
      x-code-metadata:
        analog:
          displayName: Analog
          definition: Analog sensor type.
        digital:
          displayName: Digital
          definition: Digital sensor type.
```

### Variation B5: Optional Single (0..1) via maxQualifiedCardinality

**OWL:**
```turtle
owl:maxQualifiedCardinality "1"^^xsd:nonNegativeInteger ;
owl:onClass [ rdf:type owl:Class ;
              owl:oneOf ( sensor:_sensor_type_analog
                          sensor:_sensor_type_digital ) ]
```

**OpenAPI:**
```yaml
PatternB:
  type: object
  properties:
    isCategorisedBy:
      type: string
      enum: [analog, digital]
      x-code-metadata:
        analog:
          displayName: Analog
          definition: Analog sensor type.
        digital:
          displayName: Digital
          definition: Digital sensor type.
```

### Pattern B Advantages Over Pattern A

| Capability | Pattern A | Pattern B |
|------------|-----------|-----------|
| Enumerate valid values via SPARQL | ✗ (parse ontology) | ✓ (`?x a SensorClass`) |
| Rich metadata per value | ✗ | ✓ |
| `apiLabel` on individuals | ✗ | ✓ |
| Hierarchical grouping via `hasSuperCategory` | ✗ | ✓ |

---

## Pattern C: Class-based Taxonomy (Extensible/Hierarchical)

### Discriminator

The restriction target is a **named class IRI** (not a blank node). The class:
- Has **no `owl:oneOf`** (open, not closed)
- Is **NOT `rdfs:subClassOf skos:Concept`** (not SKOS)
- Has **no outgoing object property restrictions** (it's a taxonomy, not a domain class)
- **May have subclasses** forming a hierarchy

**ASG detection (SPARQL):**
```sparql
# Step 1: Target is named class IRI (not blank node)
?restriction (owl:someValuesFrom|owl:allValuesFrom|owl:onClass) ?target .
FILTER(isIRI(?target))

# Step 2: Target has apiLabel
?target www:apiLabel ?label .

# Step 3: Not SKOS
FILTER NOT EXISTS { ?target rdfs:subClassOf skos:Concept }

# Step 4: No owl:oneOf
FILTER NOT EXISTS { ?target owl:oneOf ?list }
FILTER NOT EXISTS { ?target owl:equivalentClass/owl:oneOf ?list }

# Step 5: No outgoing object property restrictions (taxonomy, not domain class)
FILTER NOT EXISTS {
    ?target rdfs:subClassOf ?restriction2 .
    ?restriction2 owl:onProperty ?prop .
    ?prop a owl:ObjectProperty .
    # Exclude properties to datatypes or to taxonomy classes themselves
}
```

### Class Hierarchy Definition

```turtle
sensor:EnvironmentCode rdf:type owl:Class ;
    rdfs:comment "Operating environment classification for sensors"@en ;
    rdfs:label "Environment Code"@en ;
    skos:prefLabel "Environment Code"@en .

sensor:IndoorEnvironment rdf:type owl:Class ;
    rdfs:subClassOf sensor:EnvironmentCode ;
    rdfs:label "Indoor Environment"@en .

sensor:IndustrialEnvironment rdf:type owl:Class ;
    rdfs:subClassOf sensor:EnvironmentCode ;
    rdfs:label "Industrial Environment"@en .

sensor:OutdoorEnvironment rdf:type owl:Class ;
    rdfs:subClassOf sensor:EnvironmentCode ;
    rdfs:label "Outdoor Environment"@en .
```

### Individuals Definition (instances of subclasses)

```turtle
sensor:env_cleanroom rdf:type owl:NamedIndividual ,
                              sensor:IndoorEnvironment ;
    rdfs:label "Cleanroom"@en ;
    skos:definition "Controlled cleanroom environment"@en ;
    sensor:codeValue "cleanroom" ;
    sensor:displayName "Cleanroom" .

sensor:env_factory_floor rdf:type owl:NamedIndividual ,
                                  sensor:IndustrialEnvironment ;
    rdfs:label "Factory Floor"@en ;
    skos:definition "Manufacturing factory floor"@en ;
    sensor:codeValue "factory-floor" ;
    sensor:displayName "Factory Floor" .

sensor:env_hazardous rdf:type owl:NamedIndividual ,
                              sensor:IndustrialEnvironment ;
    rdfs:label "Hazardous Area"@en ;
    skos:definition "Hazardous or explosive atmosphere zone"@en ;
    sensor:codeValue "hazardous" ;
    sensor:displayName "Hazardous Area" .

sensor:env_office rdf:type owl:NamedIndividual ,
                           sensor:IndoorEnvironment ;
    rdfs:label "Office"@en ;
    skos:definition "Climate-controlled office environment"@en ;
    sensor:codeValue "office" ;
    sensor:displayName "Office" .

sensor:env_outdoor_extreme rdf:type owl:NamedIndividual ,
                                    sensor:OutdoorEnvironment ;
    rdfs:label "Outdoor Extreme"@en ;
    skos:definition "Outdoor extreme weather conditions"@en ;
    sensor:codeValue "outdoor-extreme" ;
    sensor:displayName "Outdoor Extreme" .

sensor:env_outdoor_temperate rdf:type owl:NamedIndividual ,
                                      sensor:OutdoorEnvironment ;
    rdfs:label "Outdoor Temperate"@en ;
    skos:definition "Outdoor temperate climate conditions"@en ;
    sensor:codeValue "outdoor-temperate" ;
    sensor:displayName "Outdoor Temperate" .

sensor:env_warehouse rdf:type owl:NamedIndividual ,
                              sensor:IndoorEnvironment ;
    rdfs:label "Warehouse"@en ;
    skos:definition "Indoor warehouse or storage facility"@en ;
    sensor:codeValue "warehouse" ;
    sensor:displayName "Warehouse" .
```

### Flat Taxonomy Example (TagCode)

Pattern C can also be **flat** (no subclass hierarchy)—just direct instances of the taxonomy class:

**Class Definition:**
```turtle
sensor:TagCode rdf:type owl:Class ;
    rdfs:comment "Tags for additional sensor classification"@en ;
    rdfs:label "Tag Code"@en .
```

**Individuals (direct instances, no subclasses):**
```turtle
sensor:tag_critical_asset rdf:type owl:NamedIndividual ,
                                   sensor:TagCode ;
    rdfs:label "Critical Asset"@en ;
    sensor:codeValue "critical-asset" ;
    sensor:displayName "Critical Asset" .

sensor:tag_iot_enabled rdf:type owl:NamedIndividual ,
                                sensor:TagCode ;
    rdfs:label "IoT Enabled"@en ;
    sensor:codeValue "iot-enabled" ;
    sensor:displayName "IoT Enabled" .

sensor:tag_legacy rdf:type owl:NamedIndividual ,
                           sensor:TagCode ;
    rdfs:label "Legacy"@en ;
    sensor:codeValue "legacy" ;
    sensor:displayName "Legacy" .

sensor:tag_redundant rdf:type owl:NamedIndividual ,
                              sensor:TagCode ;
    rdfs:label "Redundant"@en ;
    sensor:codeValue "redundant" ;
    sensor:displayName "Redundant" .

sensor:tag_safety_related rdf:type owl:NamedIndividual ,
                                   sensor:TagCode ;
    rdfs:label "Safety Related"@en ;
    sensor:codeValue "safety-related" ;
    sensor:displayName "Safety Related" .
```

### All Variations

```turtle
sensor:PatternC rdf:type owl:Class ;
    owl:equivalentClass
        # C1: Required single (1..1) via equivalentClass
        [ rdf:type owl:Restriction ;
          owl:onProperty sensor:hasEnvironment ;
          owl:qualifiedCardinality "1"^^xsd:nonNegativeInteger ;
          owl:onClass sensor:EnvironmentCode
        ] ;
    rdfs:subClassOf
        # C2: Required array (1..*) via someValuesFrom
        [ rdf:type owl:Restriction ;
          owl:onProperty sensor:hasEnvironment ;
          owl:someValuesFrom sensor:EnvironmentCode
        ] ,
        # C3: Type constraint (0..*) via allValuesFrom
        [ rdf:type owl:Restriction ;
          owl:onProperty sensor:hasEnvironment ;
          owl:allValuesFrom sensor:EnvironmentCode
        ] ,
        # C4: Required array (1..*) via minQualifiedCardinality
        [ rdf:type owl:Restriction ;
          owl:onProperty sensor:isCategorisedBy ;
          owl:minQualifiedCardinality "1"^^xsd:nonNegativeInteger ;
          owl:onClass sensor:EnvironmentCode
        ] ,
        # C5: Required single (1..1) via qualifiedCardinality
        [ rdf:type owl:Restriction ;
          owl:onProperty sensor:hasEnvironment ;
          owl:qualifiedCardinality "1"^^xsd:nonNegativeInteger ;
          owl:onClass sensor:EnvironmentCode
        ] ,
        # C6: Bounded array (0..3) via maxQualifiedCardinality
        [ rdf:type owl:Restriction ;
          owl:onProperty sensor:hasEnvironment ;
          owl:maxQualifiedCardinality "3"^^xsd:nonNegativeInteger ;
          owl:onClass sensor:EnvironmentCode
        ] ,
        # C7: Optional array (0..*) via allValuesFrom - flat taxonomy (no subclasses)
        [ rdf:type owl:Restriction ;
          owl:onProperty sensor:hasTag ;
          owl:allValuesFrom sensor:TagCode
        ] ;
    skos:definition "Class-Based Restrictions. Best for: Extensible categories" ;
    skos:prefLabel "Pattern C" .
```

### Variation C1: Required Single (1..1) via qualifiedCardinality

**OWL:**
```turtle
owl:qualifiedCardinality "1"^^xsd:nonNegativeInteger ;
owl:onClass sensor:EnvironmentCode
```

**OpenAPI:**
```yaml
PatternC:
  type: object
  required:
    - hasEnvironment
  properties:
    hasEnvironment:
      $ref: '#/components/schemas/EnvironmentCode'
```

### Variation C2: Required Array (1..*) via someValuesFrom

**OWL:**
```turtle
owl:someValuesFrom sensor:EnvironmentCode
```

**OpenAPI:**
```yaml
PatternC:
  type: object
  required:
    - hasEnvironment
  properties:
    hasEnvironment:
      type: array
      minItems: 1
      items:
        $ref: '#/components/schemas/EnvironmentCode'
```

### Variation C3: Type Constraint / Optional Array (0..*) via allValuesFrom

**OWL:**
```turtle
owl:allValuesFrom sensor:EnvironmentCode
```

**OpenAPI:**
```yaml
PatternC:
  type: object
  properties:
    hasEnvironment:
      type: array
      items:
        $ref: '#/components/schemas/EnvironmentCode'
```

### Variation C4: Required Array (1..*) via minQualifiedCardinality

**OWL:**
```turtle
owl:minQualifiedCardinality "1"^^xsd:nonNegativeInteger ;
owl:onClass sensor:EnvironmentCode
```

**OpenAPI:**
```yaml
PatternC:
  type: object
  required:
    - hasEnvironment
  properties:
    hasEnvironment:
      type: array
      minItems: 1
      items:
        $ref: '#/components/schemas/EnvironmentCode'
```

### Variation C5: Optional Single (0..1) via maxQualifiedCardinality 1

**OWL:**
```turtle
owl:maxQualifiedCardinality "1"^^xsd:nonNegativeInteger ;
owl:onClass sensor:EnvironmentCode
```

**OpenAPI:**
```yaml
PatternC:
  type: object
  properties:
    hasEnvironment:
      $ref: '#/components/schemas/EnvironmentCode'
```

### Variation C6: Bounded Array (0..3) via maxQualifiedCardinality

**OWL:**
```turtle
owl:maxQualifiedCardinality "3"^^xsd:nonNegativeInteger ;
owl:onClass sensor:EnvironmentCode
```

**OpenAPI:**
```yaml
PatternC:
  type: object
  properties:
    hasEnvironment:
      type: array
      maxItems: 3
      items:
        $ref: '#/components/schemas/EnvironmentCode'
```

### Variation C7: Optional Array (0..*) via allValuesFrom — Flat Taxonomy

This variation demonstrates Pattern C with a **flat taxonomy** (no subclass hierarchy). The `TagCode` class has direct instances with no intermediate subclasses.

**OWL:**
```turtle
owl:allValuesFrom sensor:TagCode
```

**OpenAPI:**
```yaml
PatternC:
  type: object
  properties:
    hasTag:
      type: array
      items:
        $ref: '#/components/schemas/TagCode'
```

**TagCode Schema (flat, no hierarchy):**
```yaml
components:
  schemas:
    TagCode:
      type: object
      properties:
        code:
          type: string
          enum: [critical-asset, iot-enabled, legacy, redundant, safety-related]
        displayName:
          type: string
      x-extensible: true
```

### EnvironmentCode Schema (with hierarchy)

**OpenAPI:**
```yaml
components:
  schemas:
    EnvironmentCode:
      type: object
      properties:
        code:
          type: string
        displayName:
          type: string
      x-extensible: true
      x-hierarchy:
        IndoorEnvironment:
          - cleanroom
          - office
          - warehouse
        IndustrialEnvironment:
          - factory-floor
          - hazardous
        OutdoorEnvironment:
          - outdoor-extreme
          - outdoor-temperate
```

### Pattern B vs C Comparison

| Aspect | Pattern B | Pattern C |
|--------|-----------|-----------|
| Restriction target | Blank node with `owl:oneOf` | Named class IRI |
| Set closure | Closed (explicit list) | Open (query instances) |
| Class structure | Flat (one class) | Flat OR hierarchical (subclasses) |
| Adding values | Requires changing `owl:oneOf` | Just add instance |
| Discovery | Parse `owl:oneOf` | Query `?x a/rdfs:subClassOf* <Class>` |
| OpenAPI output | Inline enum or `$ref` to enum | `$ref` to extensible schema |

### Pattern C Variants

| Variant | Example | Class Structure | OpenAPI |
|---------|---------|-----------------|---------|
| Hierarchical | `EnvironmentCode` | Has subclasses (`IndoorEnvironment`, etc.) | Schema with `x-hierarchy` |
| Flat | `TagCode` | Direct instances only | Schema without `x-hierarchy` |

### Pattern C Advantages

| Capability | Pattern A/B | Pattern C |
|------------|-------------|-----------|
| Add values without schema change | ✗ | ✓ |
| Hierarchical grouping | ✗ | ✓ |
| Query by category type | ✗ | ✓ (`?x a IndoorEnvironment`) |
| Inheritance of properties | ✗ | ✓ (subclasses inherit) |

---

## Pattern D: SKOS Taxonomy (FHIR-Compatible)

### Discriminator

Restriction target is a class that:
- Is `rdfs:subClassOf skos:Concept`, OR
- Has `owl:equivalentClass` with restriction on `skos:inScheme`

**ASG detection (SPARQL):**
```sparql
# Check for SKOS subclass
{ ?target rdfs:subClassOf skos:Concept }
UNION
# Check for equivalentClass with inScheme restriction
{ ?target owl:equivalentClass ?equiv .
  ?equiv owl:onProperty skos:inScheme ;
         owl:hasValue ?scheme . }
```

### FHIR Terminology Concepts

| FHIR Resource | Purpose | OWL/SKOS Mapping |
|---------------|---------|------------------|
| **CodeSystem** | Defines codes and their meanings | `skos:ConceptScheme` |
| **ValueSet** | Selects codes from CodeSystem(s) for use in a context | `skos:Collection` with `skos:member` |
| **Coding** | Single code instance: system + code + display | `skos:Concept` with `skos:notation`, `skos:prefLabel` |
| **CodeableConcept** | Multiple Codings representing same concept | Array of Coding objects |
| **Binding** | Links element to ValueSet with strength | OWL restriction + `fhir:binding` annotation |

### Full FHIR-Compatible Model

#### 1. CodeSystem (defines the codes)

```turtle
sensor:MeasurementContextCodeSystem a skos:ConceptScheme ;
    rdfs:label "Measurement Context Code System"@en ;
    skos:definition "Code system defining measurement context types"@en ;
    fhir:url "https://www.graphresearchlabs.com/sensor/CodeSystem/measurement-context"^^xsd:anyURI ;
    fhir:version "1.0.0" ;
    fhir:status "active" ;
    skos:hasTopConcept sensor:context_continuous,
                       sensor:context_discrete .
```

#### 2. Concepts (defined by the CodeSystem)

```turtle
sensor:context_continuous a skos:Concept ;
    skos:inScheme sensor:MeasurementContextCodeSystem ;
    skos:topConceptOf sensor:MeasurementContextCodeSystem ;
    skos:notation "continuous" ;
    skos:prefLabel "Continuous Measurement"@en ;
    skos:definition "Ongoing continuous measurement"@en ;
    skos:narrower sensor:context_realtime,
                  sensor:context_sampled .

sensor:context_realtime a skos:Concept ;
    skos:inScheme sensor:MeasurementContextCodeSystem ;
    skos:notation "realtime" ;
    skos:prefLabel "Real-time"@en ;
    skos:altLabel "Live"@en ;
    skos:definition "Real-time continuous streaming"@en ;
    skos:broader sensor:context_continuous ;
    skos:narrower sensor:context_realtime_high_freq,
                  sensor:context_realtime_standard .

sensor:context_realtime_high_freq a skos:Concept ;
    skos:inScheme sensor:MeasurementContextCodeSystem ;
    skos:notation "realtime-hf" ;
    skos:prefLabel "Real-time High Frequency"@en ;
    skos:definition "High-frequency real-time (>1000 samples/sec)"@en ;
    skos:broader sensor:context_realtime .

sensor:context_realtime_standard a skos:Concept ;
    skos:inScheme sensor:MeasurementContextCodeSystem ;
    skos:notation "realtime-std" ;
    skos:prefLabel "Real-time Standard"@en ;
    skos:definition "Standard frequency real-time (1-1000 samples/sec)"@en ;
    skos:broader sensor:context_realtime .

sensor:context_discrete a skos:Concept ;
    skos:inScheme sensor:MeasurementContextCodeSystem ;
    skos:topConceptOf sensor:MeasurementContextCodeSystem ;
    skos:notation "discrete" ;
    skos:prefLabel "Discrete Measurement"@en ;
    skos:definition "Point-in-time discrete measurements"@en ;
    skos:narrower sensor:context_scheduled,
                  sensor:context_triggered .

sensor:context_scheduled a skos:Concept ;
    skos:inScheme sensor:MeasurementContextCodeSystem ;
    skos:notation "scheduled" ;
    skos:prefLabel "Scheduled"@en ;
    skos:altLabel "Batch"@en ;
    skos:definition "Measurement at scheduled times"@en ;
    skos:broader sensor:context_discrete .

sensor:context_triggered a skos:Concept ;
    skos:inScheme sensor:MeasurementContextCodeSystem ;
    skos:notation "triggered" ;
    skos:prefLabel "Event Triggered"@en ;
    skos:altLabel "On-Demand"@en ;
    skos:definition "Measurement triggered by external event"@en ;
    skos:broader sensor:context_discrete ;
    skos:narrower sensor:context_triggered_alarm,
                  sensor:context_triggered_threshold .
```

#### 3. ValueSet (selects codes for a specific use)

**Full CodeSystem inclusion:**
```turtle
sensor:AllMeasurementContextsValueSet a skos:Collection ;
    rdfs:label "All Measurement Contexts Value Set"@en ;
    skos:definition "Value set including all measurement context codes"@en ;
    fhir:url "https://www.graphresearchlabs.com/sensor/ValueSet/all-measurement-contexts"^^xsd:anyURI ;
    fhir:version "1.0.0" ;
    fhir:status "active" .
    # Membership defined via bridge class equivalentClass
```

**Subset ValueSet:**
```turtle
sensor:RealtimeMeasurementValueSet a skos:Collection ;
    rdfs:label "Realtime Measurement Value Set"@en ;
    skos:definition "Value set for real-time measurement contexts only"@en ;
    fhir:url "https://www.graphresearchlabs.com/sensor/ValueSet/realtime-measurement"^^xsd:anyURI ;
    fhir:version "1.0.0" ;
    fhir:status "active" ;
    skos:member sensor:context_realtime,
                sensor:context_realtime_high_freq,
                sensor:context_realtime_standard .
```

#### 4. Bridge Class (makes ValueSet usable in OWL restrictions)

**Bridge for all contexts (full CodeSystem):**
```turtle
sensor:MeasurementContextCode a owl:Class ;
    rdfs:label "Measurement Context Code"@en ;
    rdfs:subClassOf skos:Concept ;
    owl:equivalentClass [
        a owl:Restriction ;
        owl:onProperty skos:inScheme ;
        owl:hasValue sensor:MeasurementContextCodeSystem
    ] ;
    fhir:binding [
        fhir:valueSet "https://www.graphresearchlabs.com/sensor/ValueSet/all-measurement-contexts"^^xsd:anyURI ;
        fhir:strength "required"
    ] .
```

**Bridge for subset (Realtime only):**
```turtle
sensor:RealtimeMeasurementCode a owl:Class ;
    rdfs:label "Realtime Measurement Code"@en ;
    rdfs:subClassOf sensor:MeasurementContextCode ;
    owl:equivalentClass [
        a owl:Class ;
        owl:intersectionOf (
            skos:Concept
            [ a owl:Restriction ;
              owl:onProperty skos:inScheme ;
              owl:hasValue sensor:MeasurementContextCodeSystem ]
            [ a owl:Restriction ;
              owl:onProperty skos:broader ;
              owl:someValuesFrom [ owl:oneOf ( sensor:context_realtime ) ] ]
        )
    ] ;
    fhir:binding [
        fhir:valueSet "https://www.graphresearchlabs.com/sensor/ValueSet/realtime-measurement"^^xsd:anyURI ;
        fhir:strength "required"
    ] .
```

#### 5. Binding (restriction on domain class)

```turtle
sensor:Sensor rdfs:subClassOf [
    a owl:Restriction ;
    owl:onProperty sensor:hasMeasurementContext ;
    owl:someValuesFrom sensor:MeasurementContextCode  # All contexts allowed
] .

sensor:RealtimeSensor rdfs:subClassOf
    sensor:Sensor ,
    [ a owl:Restriction ;
      owl:onProperty sensor:hasMeasurementContext ;
      owl:someValuesFrom sensor:RealtimeMeasurementCode  # Only realtime subset
    ] .
```

### All Variations

```turtle
sensor:PatternD rdf:type owl:Class ;
    rdfs:subClassOf
        # D1: Required array (1..*) via someValuesFrom
        [ rdf:type owl:Restriction ;
          owl:onProperty sensor:hasMeasurementContext ;
          owl:someValuesFrom sensor:MeasurementContextCode
        ] ,
        # D2: Type constraint (0..*) via allValuesFrom
        [ rdf:type owl:Restriction ;
          owl:onProperty sensor:hasMeasurementContext ;
          owl:allValuesFrom sensor:MeasurementContextCode
        ] ,
        # D3: Required single (1..1) via qualifiedCardinality
        [ rdf:type owl:Restriction ;
          owl:onProperty sensor:hasMeasurementContext ;
          owl:qualifiedCardinality "1"^^xsd:nonNegativeInteger ;
          owl:onClass sensor:MeasurementContextCode
        ] ,
        # D4: Optional single (0..1) via maxQualifiedCardinality
        [ rdf:type owl:Restriction ;
          owl:onProperty sensor:hasMeasurementContext ;
          owl:maxQualifiedCardinality "1"^^xsd:nonNegativeInteger ;
          owl:onClass sensor:MeasurementContextCode
        ] ,
        # D5: Bounded array (1..3) via min + max
        [ rdf:type owl:Restriction ;
          owl:onProperty sensor:hasMeasurementContext ;
          owl:minQualifiedCardinality "1"^^xsd:nonNegativeInteger ;
          owl:onClass sensor:MeasurementContextCode
        ] ,
        [ rdf:type owl:Restriction ;
          owl:onProperty sensor:hasMeasurementContext ;
          owl:maxQualifiedCardinality "3"^^xsd:nonNegativeInteger ;
          owl:onClass sensor:MeasurementContextCode
        ] ;
    skos:definition "SKOS Taxonomy. Best for: Standards-compliant vocabularies" ;
    skos:prefLabel "Pattern D" .
```

### Variation D1: Required Array (1..*) via someValuesFrom

**OWL:**
```turtle
owl:someValuesFrom sensor:MeasurementContextCode
```

**OpenAPI (Coding style):**
```yaml
PatternD:
  type: object
  required:
    - hasMeasurementContext
  properties:
    hasMeasurementContext:
      type: array
      minItems: 1
      items:
        $ref: '#/components/schemas/MeasurementContextCoding'
```

### Variation D2: Type Constraint / Optional Array (0..*) via allValuesFrom

**OWL:**
```turtle
owl:allValuesFrom sensor:MeasurementContextCode
```

**OpenAPI:**
```yaml
PatternD:
  type: object
  properties:
    hasMeasurementContext:
      type: array
      items:
        $ref: '#/components/schemas/MeasurementContextCoding'
```

### Variation D3: Required Single (1..1) via qualifiedCardinality

**OWL:**
```turtle
owl:qualifiedCardinality "1"^^xsd:nonNegativeInteger ;
owl:onClass sensor:MeasurementContextCode
```

**OpenAPI:**
```yaml
PatternD:
  type: object
  required:
    - hasMeasurementContext
  properties:
    hasMeasurementContext:
      $ref: '#/components/schemas/MeasurementContextCoding'
```

### Variation D4: Optional Single (0..1) via maxQualifiedCardinality

**OWL:**
```turtle
owl:maxQualifiedCardinality "1"^^xsd:nonNegativeInteger ;
owl:onClass sensor:MeasurementContextCode
```

**OpenAPI:**
```yaml
PatternD:
  type: object
  properties:
    hasMeasurementContext:
      $ref: '#/components/schemas/MeasurementContextCoding'
```

### Variation D5: Bounded Array (1..3) via min + max

**OWL:**
```turtle
owl:minQualifiedCardinality "1"^^xsd:nonNegativeInteger ;
owl:onClass sensor:MeasurementContextCode
# Combined with:
owl:maxQualifiedCardinality "3"^^xsd:nonNegativeInteger ;
owl:onClass sensor:MeasurementContextCode
```

**OpenAPI:**
```yaml
PatternD:
  type: object
  required:
    - hasMeasurementContext
  properties:
    hasMeasurementContext:
      type: array
      minItems: 1
      maxItems: 3
      items:
        $ref: '#/components/schemas/MeasurementContextCoding'
```

### OpenAPI Schemas for FHIR-Compatible Output

**Coding (single code):**
```yaml
components:
  schemas:
    MeasurementContextCoding:
      type: object
      required:
        - code
      properties:
        system:
          type: string
          const: "https://www.graphresearchlabs.com/sensor/CodeSystem/measurement-context"
          description: The CodeSystem URL
        code:
          type: string
          description: The code value (skos:notation)
        display:
          type: string
          description: Human-readable display (skos:prefLabel)
      x-fhir-valueset: "https://www.graphresearchlabs.com/sensor/ValueSet/all-measurement-contexts"
      x-fhir-binding-strength: required
      x-skos-hierarchy:
        continuous:
          children: [realtime, sampled]
        realtime:
          parent: continuous
          children: [realtime-hf, realtime-std]
        realtime-hf:
          parent: realtime
        realtime-std:
          parent: realtime
        sampled:
          parent: continuous
        discrete:
          children: [scheduled, triggered]
        scheduled:
          parent: discrete
        triggered:
          parent: discrete
          children: [triggered-alarm, triggered-threshold]
```

**CodeableConcept (multiple codings):**
```yaml
    MeasurementContextCodeableConcept:
      type: object
      properties:
        coding:
          type: array
          items:
            $ref: '#/components/schemas/MeasurementContextCoding'
          description: Code(s) from controlled vocabularies
        text:
          type: string
          description: Plain text representation chosen by user
```

### Pattern D Discovery (ASG)

```sparql
# Find all concepts in a scheme
SELECT ?concept ?code ?display ?broader WHERE {
    ?concept skos:inScheme sensor:MeasurementContextCodeSystem ;
             skos:notation ?code ;
             skos:prefLabel ?display .
    OPTIONAL { ?concept skos:broader ?broader }
}
ORDER BY ?broader ?code
```

### FHIR Binding Strengths

| Strength | Meaning | OWL Pattern |
|----------|---------|-------------|
| **required** | Must be from ValueSet | `owl:someValuesFrom` + `owl:allValuesFrom` |
| **extensible** | Should be from ValueSet, can extend | `owl:someValuesFrom` only |
| **preferred** | Encouraged but not required | `owl:someValuesFrom` with broader class |
| **example** | Illustrative only | `rdfs:range` only (no restriction) |

### Pattern C vs D Comparison

| Aspect | Pattern C (Class-based) | Pattern D (SKOS) |
|--------|------------------------|------------------|
| Standard | OWL only | OWL + SKOS + FHIR |
| Hierarchy | `rdfs:subClassOf` | `skos:broader`/`skos:narrower` |
| Membership | `rdf:type` | `skos:inScheme` |
| Subset selection | Subclass hierarchy | `skos:Collection` + `skos:member` |
| Multi-language | Not native | `skos:prefLabel`/`skos:altLabel` with `@lang` |
| External mapping | Not native | `skos:exactMatch`, `skos:closeMatch` |
| Interoperability | OWL tools only | FHIR, SKOS, terminology servers |
| Discovery | `?x a/rdfs:subClassOf* Class` | `?x skos:inScheme Scheme` |

### Pattern D Advantages

| Capability | Pattern A/B/C | Pattern D |
|------------|---------------|-----------|
| FHIR compatibility | ✗ | ✓ |
| Terminology server support | ✗ | ✓ |
| Multi-language labels | ✗ | ✓ |
| Alternative labels | ✗ | ✓ (`skos:altLabel`) |
| External code mapping | ✗ | ✓ (`skos:exactMatch`) |
| Transitive hierarchy queries | Limited | ✓ (`skos:broader+`) |
| ValueSet subsetting | ✗ | ✓ |
| Binding strength | ✗ | ✓ |

### Comparison: Pattern D vs FHIR RDF Ontology

The official FHIR RDF specification takes a different approach to terminology than our SKOS-based Pattern D. Here's how they compare:

#### FHIR RDF Ontology Approach

FHIR RDF represents coded values as **structured objects** with properties, not as direct concept references:

```turtle
# FHIR RDF Coding structure
fhir:CodingBase.system rdf:type owl:ObjectProperty .
fhir:CodingBase.code rdf:type owl:ObjectProperty .
fhir:CodingBase.display rdf:type owl:ObjectProperty .
fhir:CodingBase.version rdf:type owl:ObjectProperty .

# Example instance (FHIR RDF style)
:observation1 fhir:Observation.status [
    fhir:Coding.system [ fhir:value "http://hl7.org/fhir/observation-status"^^xsd:anyURI ] ;
    fhir:Coding.code [ fhir:value "final" ] ;
    fhir:Coding.display [ fhir:value "Final" ]
] .
```

#### Pattern D (SKOS) Approach

Pattern D uses **direct concept references** with SKOS vocabulary:

```turtle
# Pattern D structure
sensor:context_realtime a skos:Concept ;
    skos:inScheme sensor:MeasurementContextCodeSystem ;
    skos:notation "realtime" ;
    skos:prefLabel "Real-time"@en .

# Example instance (Pattern D style)
:sensor1 sensor:hasMeasurementContext sensor:context_realtime .
```

#### Key Differences

| Aspect | FHIR RDF Ontology | Pattern D (SKOS) |
|--------|-------------------|------------------|
| **Code representation** | Anonymous node with properties | Named individual (IRI) |
| **Value access** | `fhir:Coding.code/fhir:value` | Direct object property |
| **System identification** | Explicit `fhir:Coding.system` property | Implicit via `skos:inScheme` |
| **Display text** | Explicit `fhir:Coding.display` property | `skos:prefLabel` on concept |
| **Hierarchy** | Not in Coding structure | `skos:broader/narrower` on concepts |
| **Round-tripping** | Optimized for JSON/XML ↔ RDF | Native RDF/OWL semantics |
| **Reasoning** | Limited (data-oriented) | Full OWL DL reasoning |
| **External ontologies** | Optional `rdf:type` extension | `skos:exactMatch`, `skos:closeMatch` |

#### FHIR RDF Extensions for Ontology Integration

FHIR RDF provides optional extensions to bridge to external ontologies:

```json
{
  "code": {
    "extension": {
      "url": "http://hl7.org/fhir/StructureDefinition/rdftype",
      "valueUri": "http://snomed.info/id/24484000"
    },
    "coding": {
      "system": "http://snomed.info/sct",
      "code": "24484000",
      "display": "Severe"
    }
  }
}
```

This allows FHIR instances to carry an `rdf:type` assertion linking to external ontology concepts, while maintaining JSON/XML compatibility.

#### When to Use Each Approach

| Use Case | Recommended Approach |
|----------|---------------------|
| Exchanging FHIR resources | FHIR RDF (for round-tripping) |
| Internal ontology modeling | Pattern D (SKOS) |
| Terminology server integration | Either (both support FHIR API) |
| OWL reasoning over codes | Pattern D (SKOS) |
| Multi-system code mapping | Pattern D (via `skos:exactMatch`) |
| Generating APIs from ontology | Pattern D (cleaner mapping) |

#### Hybrid Approach for FHIR Interoperability

For maximum interoperability, you can support both:

```turtle
# Define concepts in SKOS (Pattern D)
sensor:context_realtime a skos:Concept ;
    skos:inScheme sensor:MeasurementContextCodeSystem ;
    skos:notation "realtime" ;
    skos:prefLabel "Real-time"@en ;
    # Map to FHIR coding
    fhir:system "https://graphresearchlabs.com/sensor/CodeSystem/measurement-context"^^xsd:anyURI ;
    fhir:code "realtime" ;
    fhir:display "Real-time" .

# In instance data, use direct reference (Pattern D style)
:sensor1 sensor:hasMeasurementContext sensor:context_realtime .

# For FHIR export, generate Coding structure
# {
#   "system": "https://graphresearchlabs.com/sensor/CodeSystem/measurement-context",
#   "code": "realtime",
#   "display": "Real-time"
# }
```

This allows:
1. **Native OWL reasoning** via direct concept references
2. **FHIR JSON/XML export** via properties on concepts
3. **Terminology server queries** via SKOS structure

---

## ASG Decision Tree

```
1. EXTRACT RESTRICTION from class definition
   Check both rdfs:subClassOf and owl:equivalentClass

2. EXTRACT TARGET from restriction
   ├─ someValuesFrom → target
   ├─ allValuesFrom → target
   └─ onClass → target

3. IS TARGET A BLANK NODE?
   └─ YES → Check for owl:oneOf inside blank node
            └─ Found owl:oneOf? Extract individuals from list
               └─ Do individuals have a domain class type (not just owl:NamedIndividual)?
                  ├─ YES → Pattern B (typed individuals)
                  │        Parse oneOf, emit enum with x-code-metadata
                  └─ NO  → Pattern A (inline enum)
                           Parse oneOf, emit simple inline enum
   └─ NO (target is named class IRI) → Continue to step 4

4. DOES TARGET HAVE apiLabel MATCHING CURRENT API SUFFIX?
   └─ NO  → Stop traversal (not included in this API)
   └─ YES → Continue to step 5

5. IS TARGET rdfs:subClassOf skos:Concept?
   OR HAS owl:equivalentClass WITH skos:inScheme RESTRICTION?
   └─ YES → Pattern D (SKOS)
            Query: ?x skos:inScheme <scheme>
            Emit: Coding schema with hierarchy
   └─ NO  → Continue to step 6

6. DOES TARGET HAVE owl:oneOf (direct or via equivalentClass)?
   └─ YES → Pattern B variant (named class enum)
            Parse oneOf list, emit enum schema with x-code-metadata
   └─ NO  → Continue to step 7

7. DOES TARGET HAVE OUTGOING OBJECT PROPERTY RESTRICTIONS?
   (i.e., rdfs:subClassOf [ owl:onProperty ?p ] where ?p is owl:ObjectProperty
    pointing to another domain class)
   └─ YES → Domain class, not taxonomy
            Continue property traversal into this class
   └─ NO  → Pattern C (class-based taxonomy)
            Query: ?x a/rdfs:subClassOf* <TargetClass>
            Emit: $ref to extensible schema with x-hierarchy
```

---

## Cardinality Summary

| OWL Pattern | Cardinality | Required? | Array? |
|-------------|-------------|-----------|--------|
| `owl:qualifiedCardinality 1` | 1..1 | Yes | No |
| `owl:maxQualifiedCardinality 1` | 0..1 | No | No |
| `owl:someValuesFrom` | 1..* | Yes | Yes |
| `owl:allValuesFrom` (alone) | 0..* | No | Yes |
| `owl:someValuesFrom` + `owl:allValuesFrom` | 1..* (constrained) | Yes | Yes |
| `owl:minQualifiedCardinality n` | n..* | Yes (if n≥1) | Yes |
| `owl:maxQualifiedCardinality n` | 0..n | No | Yes (bounded) |

---

## Annotation Requirements

| Pattern | What needs `apiLabel` matching API suffix |
|---------|------------------------------------------|
| A | Property only (inline `owl:oneOf` parsed, individuals have no class) |
| B | Property + individuals (individuals have `apiLabel` for API inclusion) |
| C | Property + taxonomy class + subclasses (if hierarchical) |
| D | Property + bridge class (concepts discovered via `skos:inScheme`) |

**Pattern A vs B annotation difference:**
- Pattern A: Individuals discovered from `owl:oneOf`, no `apiLabel` needed on individuals
- Pattern B: Individuals have domain class and `apiLabel` for multi-API support

**Pattern C annotation notes:**
- The taxonomy class (`EnvironmentCode`) needs `apiLabel`
- Subclasses (`IndoorEnvironment`, `IndustrialEnvironment`, `OutdoorEnvironment`) need `apiLabel` if they should appear in the API hierarchy
- Individuals are discovered via query, so they don't need `apiLabel` unless filtered by API

**Pattern D annotation notes:**
- The bridge class (`MeasurementContextCode`) needs `apiLabel`
- The `skos:ConceptScheme` does NOT need `apiLabel` (it's not an OWL class)
- `skos:Concept` instances do NOT need `apiLabel` — discovered via `skos:inScheme` query
- ValueSet collections do NOT need `apiLabel` — referenced via bridge class binding
