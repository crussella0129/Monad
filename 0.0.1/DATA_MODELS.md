Core Data Models and Invariants

GeometryEntity
- Fields: entity_id, geom_blob_ref, topology_id, metadata, provenance_id.
- Invariant: entity_id stable across edits unless explicitly recreated.
- Failure: MissingGeometryBlob when geom_blob_ref is absent.

Provenance
- Fields: source_type, source_ref, timestamp, operator_chain.
- Invariant: provenance is immutable once recorded.
- Failure: OrphanedSource when source_ref is missing.

OperatorSpec
- Fields: operator_id, name, version, inputs (typed), params, declared_side_effects.
- Invariant: same inputs + params + version => same outputs.
- Failure: OperatorValidationError on invalid params or type mismatch.

TimelineStep
- Fields: step_id, order_index, operator_spec, input_refs, output_refs, status, diagnostics.
- Invariant: steps are append-only; edits create new versions.
- Failure: StepInvalidated with dependency chain when refs break.

SynthNode
- Fields: node_id, op, inputs, outputs, tree_context.
- Invariant: deterministic evaluation given inputs.
- Failure: TreeNondeterministicError if nondeterminism detected.

Reference
- Fields: ref_id, target_entity_id, target_topology_path, resolution_rules.
- Invariant: resolve exactly or fail; no guessing.
- Failure: ReferenceResolutionError on ambiguity or missing targets.
