# Bugfix Requirements Document

## Introduction

Stored analysis JSON objects produced by older API versions are deserialized without any check against the current schema version. When the API evolves and the shape of the analysis object changes, previously stored data is silently loaded and used as-is, leading to subtle data corruption, incorrect computations, or undefined behavior — all without any error being surfaced to the caller. The fix introduces a `schemaVersion` field on every stored analysis object and enforces a version check at deserialization time, either migrating compatible data or discarding (and optionally re-fetching) incompatible data.

## Bug Analysis

### Current Behavior (Defect)

1.1 WHEN a stored analysis object was serialized with an older API version and is loaded by the current runtime THEN the system deserializes it without inspecting any version field, silently accepting structurally incompatible data.

1.2 WHEN the deserialized analysis object is missing fields that were added in a newer API version THEN the system proceeds with default-zero or null values for those fields, producing incorrect analysis results without emitting any warning or error.

1.3 WHEN the deserialized analysis object contains fields that were removed or renamed in a newer API version THEN the system ignores or misinterprets those fields, causing silent data corruption in downstream logic that depends on the analysis output.

1.4 WHEN multiple stored analysis objects with different implicit schema versions coexist in storage THEN the system treats them all as current-version objects, making it impossible to distinguish which records are safe to use.

### Expected Behavior (Correct)

2.1 WHEN a stored analysis object is loaded THEN the system SHALL inspect the `schemaVersion` field of the stored object before deserializing its payload into the current type.

2.2 WHEN the `schemaVersion` field is absent from a stored analysis object THEN the system SHALL treat the record as version 0 (legacy/unknown) and apply the configured incompatibility policy (migrate or discard) rather than silently accepting the data.

2.3 WHEN the stored `schemaVersion` is lower than the current version but a migration path exists THEN the system SHALL apply the appropriate migration transforms to bring the data to the current schema version before use.

2.4 WHEN the stored `schemaVersion` is incompatible with the current version and no migration path exists THEN the system SHALL discard the stored record, return an explicit error or a defined sentinel value to the caller, and log a structured warning indicating the schema mismatch.

2.5 WHEN a new analysis object is written to storage THEN the system SHALL embed the current `schemaVersion` value in the serialized representation.

### Unchanged Behavior (Regression Prevention)

3.1 WHEN a stored analysis object carries the current `schemaVersion` THEN the system SHALL CONTINUE TO deserialize and return it correctly without any additional latency or behavioral change.

3.2 WHEN a stored analysis object carries a `schemaVersion` that is newer than the runtime's known version THEN the system SHALL CONTINUE TO surface an explicit error rather than silently processing unknown future-format data.

3.3 WHEN analysis objects are written and immediately read back within the same runtime version THEN the system SHALL CONTINUE TO produce identical results to the pre-fix behavior for all valid, current-version inputs.

3.4 WHEN the storage layer contains no analysis objects THEN the system SHALL CONTINUE TO behave as if the store is empty, with no errors or panics introduced by the schema-version logic.

---

## Bug Condition (Pseudocode)

```pascal
FUNCTION isBugCondition(X)
  INPUT: X of type StoredAnalysisRecord
  OUTPUT: boolean

  // The bug triggers whenever a stored record lacks a schema version
  // OR carries a version that differs from the current runtime version
  RETURN X.schemaVersion IS NULL
      OR X.schemaVersion != CURRENT_SCHEMA_VERSION
END FUNCTION
```

### Fix-Checking Property

```pascal
// Property: Fix Checking — incompatible stored data is handled explicitly
FOR ALL X WHERE isBugCondition(X) DO
  result ← loadAnalysis'(X)
  ASSERT (result IS migrated_record AND result.schemaVersion = CURRENT_SCHEMA_VERSION)
      OR (result IS explicit_error AND no_silent_corruption(result))
END FOR
```

### Preservation Property

```pascal
// Property: Preservation Checking — current-version records are unaffected
FOR ALL X WHERE NOT isBugCondition(X) DO
  ASSERT loadAnalysis(X) = loadAnalysis'(X)
END FOR
```
