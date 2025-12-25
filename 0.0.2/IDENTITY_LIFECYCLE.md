Identity Lifecycle Specification

Purpose
Define rules for entity and sub-entity identity creation, mutation, deletion, and reference repair workflows.

Identity Hierarchy

Levels
1. Entity Identity: Top-level GeometryEntity (solid, surface, curve, etc.).
2. Sub-Entity Identity: Topological elements within an entity (face, edge, vertex).
3. Transient Identity: Temporary identities during evaluation (not persisted).

Identity Generation

Entity ID Format
```
ent_<ulid>
```
- ULID provides time-ordering and uniqueness.
- Example: `ent_01HXYZ7890ABCDEF1234567890`

Sub-Entity ID Format
```
<entity_id>:<element_type>/<index>
```
- Sub-entities are identified by their parent entity and topology path.
- See TOPOLOGY_PATH.md for full path specification.

Creation Rules

Entity Creation
1. Operator creates new geometry blob.
2. System assigns new entity_id (never reused).
3. Entity is registered in identity map with initial version = 1.
4. Provenance records creation operator and inputs.

Sub-Entity Creation
1. Sub-entities are implicitly created when geometry is created.
2. Indices are assigned sequentially at creation time.
3. Sub-entity identities are stored in entity metadata.
4. No explicit "create sub-entity" operation exists.

Mutation Rules

Immutable Fields
- entity_id: Never changes after creation.
- geom_blob_ref: Blobs are immutable; mutations create new blobs.
- provenance_id: Provenance records are append-only.

Mutable Fields (Versioned)
- topology_id: Changes when topology is modified (new version created).
- metadata: Can be updated (new version created).

Version Semantics
```
GeometryEntity:
  entity_id: ent_01HXYZ
  version: 3
  geom_blob_ref: blob_ABC123
  previous_version: version_ref_2
```

Modification Workflow
1. Operator receives entity at version N.
2. Operator produces new geometry blob.
3. System creates entity version N+1 with new blob_ref.
4. Version N remains accessible for rollback.
5. References to version N continue to resolve to version N.

Deletion Rules

Soft Delete
- Entities are never physically deleted during active session.
- Deleted entities are marked with `deleted_at` timestamp.
- Deleted entities remain resolvable with `DeletedEntityError`.

Hard Delete
- Occurs only during garbage collection.
- Entities with no live references are candidates.
- GC runs only on explicit user action or session close.

Sub-Entity Deletion
- When topology changes, old sub-entities are marked obsolete.
- Obsolete sub-entities resolve with `ObsoleteSubEntityError`.
- Topology diff records which elements were removed.

Reference Lifecycle

Reference States
```
enum ReferenceState:
  Valid          # Resolves to live entity/sub-entity
  Stale          # Entity version changed; may need update
  Broken         # Target deleted or topology changed
  Orphaned       # Source entity deleted
```

State Transitions
```
Valid -> Stale:     Target entity version incremented
Stale -> Valid:     Reference explicitly rebind to new version
Stale -> Broken:    Topology path no longer exists
Valid -> Broken:    Target deleted
Valid -> Orphaned:  Source entity deleted (reference itself gone)
```

Reference Resolution
1. Lookup entity_id in identity map.
2. If entity deleted, return `DeletedEntityError`.
3. Check reference's expected_version vs current_version.
4. If version mismatch, attempt path resolution on current topology.
5. If path resolves, return `StaleReferenceWarning` with resolved element.
6. If path fails, return `BrokenReferenceError`.

Reference Repair Workflows

Automatic Repair (Disabled by Default)
- NOT SUPPORTED in initial design.
- Violates principle of explicit failures.

Manual Repair
1. User receives `BrokenReferenceError` with diagnostic.
2. System presents current topology and available paths.
3. User selects new target path.
4. System creates new reference with current entity version.
5. Old reference is marked superseded.

Batch Repair
1. User initiates repair workflow for timeline step.
2. System collects all broken references in step.
3. System suggests repairs based on topology diff (if available).
4. User confirms or modifies each suggestion.
5. System applies all repairs atomically.

Diagnostics Schema
```
BrokenReferenceError:
  reference_id: ref_123
  original_path: "ent_01HXYZ:face/2"
  original_entity_version: 1
  current_entity_version: 3
  topology_diff:
    removed: ["face/2", "face/3"]
    added: ["face/2_split_a", "face/2_split_b"]
    unchanged: ["face/0", "face/1"]
  suggested_repairs:
    - path: "face/2_split_a"
      confidence: "low"
      reason: "Index proximity"
```

Garbage Collection

GC Eligibility
Entity versions are GC-eligible when:
1. No timeline step references that version.
2. No active reference points to that version.
3. User has not pinned the version.

GC Process
1. Mark phase: Traverse all live references and timeline steps.
2. Sweep phase: Collect unreferenced entity versions and blobs.
3. Compact phase: Reclaim storage space.

GC Safety
- GC is never automatic during active editing.
- GC requires explicit user confirmation.
- GC creates a checkpoint before execution.
