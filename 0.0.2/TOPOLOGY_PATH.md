Topology Path Encoding Specification

Purpose
Define a stable, human-auditable, and machine-parseable schema for addressing topological elements (faces, edges, vertices) within geometric entities.

Requirements
1. Stability: Paths must remain valid across serialization/deserialization cycles.
2. Auditability: Paths must be human-readable for debugging and diagnostics.
3. Versioning: Schema changes must be backward-compatible or explicitly versioned.
4. Uniqueness: Each topological element within an entity has exactly one canonical path.

Path Schema

Format
```
<entity_id>:<element_type>/<index_or_qualifier>[/<sub_path>]
```

Components
- entity_id: The stable identifier of the parent GeometryEntity (UUID v7 for time-ordering).
- element_type: One of `face`, `edge`, `vertex`, `loop`, `shell`, `solid`.
- index_or_qualifier: Numeric index or semantic qualifier (e.g., `top`, `bottom`, `outer`).
- sub_path: Optional nested path for sub-elements (e.g., edge within face).

Examples
```
ent_01HXYZ:face/0                      # First face of entity
ent_01HXYZ:face/top                    # Semantically named face
ent_01HXYZ:face/0/edge/2               # Third edge of first face
ent_01HXYZ:edge/3                      # Fourth edge at entity level
ent_01HXYZ:shell/0/face/2/loop/outer   # Outer loop of face 2 in shell 0
```

Element Type Hierarchy
```
solid
  └── shell
        └── face
              └── loop
                    └── edge
                          └── vertex
```

Index Stability Rules
1. Indices are assigned at geometry creation time and stored in metadata.
2. Indices are NOT recomputed from geometric properties (no auto-reordering).
3. If topology changes (e.g., face split), new elements get new indices.
4. Deleted elements leave index gaps; indices are never reused within an entity version.

Semantic Qualifiers
- Reserved qualifiers: `top`, `bottom`, `front`, `back`, `left`, `right`, `inner`, `outer`.
- Semantic qualifiers are optional annotations set by operators or users.
- If a qualifier becomes ambiguous (e.g., multiple "top" faces), resolution fails.

Serialization Format

Text Format (Human-Readable)
```
path: "ent_01HXYZ:face/0/edge/2"
```

Binary Format (Compact)
```
[entity_id: 16 bytes][path_length: 2 bytes][path_segments: variable]
path_segment := [type: 1 byte][qualifier_type: 1 byte][value: 4 bytes]
```

Versioning
- Schema version is embedded in serialized paths: `v1:ent_01HXYZ:face/0`.
- Version prefix is optional in text format (defaults to current version).
- Binary format includes schema version in header.

Resolution Rules
1. Exact match: Path must resolve to exactly one element.
2. Ambiguity error: If multiple elements match, emit `PathAmbiguousError`.
3. Missing error: If no elements match, emit `PathNotFoundError`.
4. Stale error: If entity version changed and path invalid, emit `PathStaleError`.

Diagnostics
On resolution failure, diagnostic must include:
- The attempted path.
- The entity version at reference creation.
- The current entity version.
- Available paths in the current topology (for debugging).

Migration
When an operator modifies topology:
1. Record a topology diff: old_paths -> new_paths mapping.
2. Store diff in TimelineStep diagnostics.
3. Downstream references can use diff for explicit rebind UI.

Open Questions (Deferred)
- Should we support regex or wildcard path queries for bulk selection?
- How to handle curved edge parameterization in path (e.g., point at t=0.5)?
