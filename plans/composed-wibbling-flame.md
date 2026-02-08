# ASG Documentation and Integration Test Plan

## Overview

Create comprehensive end-user documentation for ASG (API Specification Generator), then use that documentation to drive creation of exhaustive integration tests with purpose-built test ontologies.

## Phase 1: Documentation

### Directory Structure

```
docs/
├── index.md                    # Landing page and navigation
├── getting-started.md          # Quick start (5-minute setup)
├── cli-reference.md            # Command-line options
├── configuration/
│   ├── index.md               # Config overview
│   ├── config-reference.md    # All YAML options
│   ├── templates.md           # OpenAPI templates
│   └── security.md            # Security configuration
├── ontology-design/
│   ├── index.md               # Ontology requirements
│   ├── structure.md           # Required structure (APIRoot, etc.)
│   ├── annotations.md         # All supported annotations
│   ├── restrictions.md        # OWL restrictions → OpenAPI
│   └── datatypes.md           # XSD → OpenAPI type mapping
├── mapping-reference/
│   ├── owl-to-openapi-matrix.md  # Quick reference matrix
│   └── property-mappings.md      # Data/Object/Taxonomy properties
├── examples/
│   ├── minimal-example.md     # Simplest working example
│   └── common-patterns.md     # Recipe-style patterns
└── troubleshooting.md         # Common issues and solutions
```

### Key Documentation Content

#### Mapping and Ontology To an API
1. **API How to identify an API**
  - use APIRoot
  - subclass of hasAPI some `api-name`
  - If more than one APIRoot in ontology, see rootClass in config
2. ** How to specify resources in the API **
  - Resource as in REsource/REpresentational State Transfer
  - Each such class becomes a resource in the Open API spec
  - Resource classes marked by being subclass of apiID annotated with apiLabel `api-name`
  - Instances of the class in a graph will have an `apiID` data-property to uniquely identify them
  - Produces collection endpoints /`resource-name` with methods:
    - POST: create new
    - GET: list existing (with pagination)
    - Produces single item endpoints /`resource-name`/`apiID` with methods:
    - PUT: update by replacing
    - PATCH: update by merging
    - GET: fetch
    - DELETE: remove
  - Each resource class will have a schema in components/schemas
3. ** How to Specify Resource Properties
  - Data and Object properties annotated with apiLabel `api-name` are candidates
  - Restrictions specify cardinality etc in json schema (open api schema components)
  - Individuals become enums
1. **Configuration Reference** - Document all YamlConfig fields:
   - `name`, `ontologies`, `outputDir` (required)
   - `extends`, `rootClass`, `catalogFile`, `enforceRequiredness`
   - `namespaceMappings`, `securitySchemeName`, `securityTemplate`
   - `templates.openapi`, `templates.collectionPathItem`, `templates.singlePathItem`
   - Use the existing documentation in README.md as a starting point and add the configuration reference to this document

2. **OWL Restrictions** - Document all 16 RestrictionInfo types:
   - Object: MinCardinality, MaxCardinality, ExactCardinality, SomeValuesFrom, AllValuesFrom, HasValue, OneOf
   - Data: DataMinCardinality, DataMaxCardinality, DataExactCardinality, DataSomeValuesFrom, DataAllValuesFrom, DataHasValue, DataOneOf
   - Boolean: UnionOf, IntersectionOf, ComplementOf

3. **XSD Datatypes** - Document all 46 type mappings from DataPropertyMapper

4. **Annotations** - Document apiLabel, apiDescription, apiSummary, apiTag, apiID, security, skos:prefLabel, skos:definition, skos:example

### Critical Source Files for Documentation

- `/home/mike/grl/ASG/src/main/java/com/grl/component/asg/configuration/YamlConfig.java` - All config options
- `/home/mike/grl/ASG/src/main/java/com/grl/component/schema/RestrictionInfo.java` - 16 restriction types
- `/home/mike/grl/ASG/src/main/java/com/grl/component/mapper/DataPropertyMapper.java` - XSD mappings
- `/home/mike/grl/ASG/src/main/java/com/grl/component/asg/OntologyConstants.java` - Annotation IRIs

---

## Phase 2: Integration Tests

### Test Organization

```
src/test/java/regression/features/
├── restrictions/
│   ├── CardinalityRestrictionsIT.java    # Min/Max/Exact cardinality
│   ├── ValueRestrictionsIT.java          # SomeValuesFrom, AllValuesFrom, etc.
│   └── BooleanRestrictionsIT.java        # UnionOf, IntersectionOf, ComplementOf
├── datatypes/
│   ├── StringDatatypesIT.java            # string, anyURI, base64Binary, etc.
│   ├── NumericDatatypesIT.java           # integer, decimal, float, double
│   └── TemporalDatatypesIT.java          # date, dateTime, time, duration
├── properties/
│   ├── DataPropertyIT.java               # Data property scenarios
│   ├── ObjectPropertyIT.java             # $ref, x-grl-model-link
│   └── TaxonomyPropertyIT.java           # Enum from individuals
├── configuration/
│   ├── ConfigInheritanceIT.java          # extends, variable substitution
│   └── SecurityConfigIT.java             # OAuth2, API key, scopes
└── annotations/
    └── AnnotationMappingIT.java          # All annotation types
```

### Test Ontologies (Purpose-Built)

```
src/test/resources/regression/
├── ontologies/
│   ├── cardinality-test.owl              # All 6 cardinality restrictions
│   ├── value-restrictions-test.owl       # SomeValuesFrom, AllValuesFrom, etc.
│   ├── boolean-combinations-test.owl     # UnionOf, IntersectionOf, ComplementOf
│   ├── string-datatypes-test.owl         # All string-family XSD types
│   ├── numeric-datatypes-test.owl        # All numeric XSD types
│   ├── temporal-datatypes-test.owl       # date, dateTime, time, duration
│   ├── object-property-test.owl          # $ref, arrays, x-grl-model-link
│   ├── taxonomy-test.owl                 # Named individuals as enums
│   └── annotation-test.owl               # All annotation types
├── config_base.yaml                  # all extend from this (sas in existing tests)
└── configs/
    └── config_<feature>.yaml             # One config per ontology
```

### Test Patterns (Follow Existing)

From `SimpleApiIT.java:28-30`:
```java
ReadContext spec = TestUtil.getGeneratedSpec(
    TestUtil.asFilePath(FeatureIT.class, "config_feature.yaml"));
```

JSONPath assertions:
```java
assertEquals("expected", spec.read("$.components.schemas.Resource.properties.field.type"));
assertEquals(1, spec.read("$.components.schemas.Resource.properties.field.minItems", Integer.class));
```

### Coverage Matrix (High Priority Tests)

| Feature | OpenAPI Assertion |
|---------|-------------------|
| MinCardinality(n) | `minItems: n`, required if n>=1 |
| MaxCardinality(n) | `maxItems: n` |
| ExactCardinality(n) | `minItems: n, maxItems: n` |
| SomeValuesFrom | `minItems: 1`, required |
| AllValuesFrom | Type constraint only, no cardinality |
| DataOneOf | `enum: [values]` |
| DataHasValue | `default: value` |
| UnionOf | `anyOf: [schemas]` |
| IntersectionOf | `allOf: [schemas]` |
| xsd:string | `type: string` |
| xsd:integer | `type: integer` |
| xsd:dateTime | `type: string, format: date-time` |
| xsd:anyURI | `type: string, format: uri` |
| Object to resource | `x-grl-model-link: ResourceName` |
| Taxonomy | `enum: [individual labels]` |

---

## Execution Order

1. **Create docs/ structure** - Empty files with headers
2. **Write configuration documentation** - config-reference.md, templates.md, security.md
3. **Write ontology design documentation** - restrictions.md, datatypes.md, annotations.md
4. **Write examples** - minimal-example.md, common-patterns.md
5. **Create test ontologies** - One per feature area
6. **Create test configs** - One per ontology
7. **Implement integration tests** - Follow documentation systematically

---

## Verification

1. **Documentation**: Review each doc file covers its source file completely
2. **Ontologies**: Run with `-G` flag to verify namespace flattening
3. **Tests**: Run `mvn verify` to execute all integration tests
4. **Coverage**: Ensure each documented feature has a corresponding test

```bash
# Build and verify
mvn clean verify

# Run specific test class
mvn test -Dtest=CardinalityRestrictionsIT

# Run all feature tests
mvn verify -Dit.test="regression.features.**"
```
