# ASG Project Memory

## Key Lessons
- When removing namespace flattening, OWL class iteration order from `Set<OWLClass>` becomes non-deterministic. Sort by resolved name before iterating to ensure deterministic OpenAPI output (tags, paths, etc.).
- `MappingContext` constructor changed: no longer takes `defaultOntologyPrefixIRI` or `OWLOntologyManager` params. Takes `(OWLOntology, String schemaLabel, OWLClass apiRoot, Set<OWLClass> resources, YamlConfig)`.
- `ObjectPropertyMapper` now takes `List<OWLClass>` range classes instead of `List<String>` ref names.
- `SchemaVisitor` has reverse-lookup maps: `getObjectPropertiesByName()` and `getDataPropertiesByName()`.
- `namespaceMappings` config field, `NamespaceMappingsDeserializer`, and `AsgUtils.applyNamespaceMappings()` are all deleted.
- Build command: `mvn clean verify` runs all unit + integration tests (~89 tests, ~25s).
- Test configs are in `src/test/resources/regression/` and `src/test/resources/com/grl/component/asg/configuration/`.
