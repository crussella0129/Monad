Storage Format Specification

Purpose
Define content-addressable storage scheme for geometry blobs, entity metadata, provenance records, and evaluation artifacts.

Storage Architecture

Overview
```
Storage.Repository
├── Blobs/          # Content-addressable geometry blobs
├── Entities/       # Entity metadata and version history
├── Provenance/     # Immutable provenance records
├── Timeline/       # Timeline steps and evaluation results
├── Cache/          # Evaluation cache (ephemeral)
└── Index/          # Lookup indices
```

Content-Addressable Blob Storage

Blob Identification
```
blob_id = blake3(content)
```
- BLAKE3 chosen for speed and security.
- 256-bit hash provides collision resistance.
- Example: `blob_a1b2c3d4e5f6...` (64 hex chars)

Blob Format
```
Header (fixed size):
  magic: 4 bytes ("MNDB")
  version: 2 bytes
  content_type: 2 bytes (0x01=OpenNURBS, 0x02=mesh, etc.)
  compression: 1 byte (0x00=none, 0x01=zstd)
  reserved: 7 bytes

Content:
  [compressed or raw bytes]

Footer:
  checksum: 32 bytes (blake3 of header + content)
```

Content Types
```
0x01: OpenNURBS 3dm blob
0x02: Triangle mesh (custom format)
0x03: Point cloud
0x04: Curve data
0x05: Metadata blob
```

Deduplication
- Identical geometry produces identical blob_id.
- Storage layer stores each unique blob once.
- Multiple entities can reference same blob.

Entity Storage

Entity Record Format
```yaml
entity_id: ent_01HXYZ
current_version: 3
created_at: 2024-01-15T10:30:00Z
deleted_at: null

versions:
  - version: 1
    geom_blob_ref: blob_abc123
    topology_id: topo_001
    metadata_blob_ref: blob_meta1
    provenance_id: prov_001
    created_at: 2024-01-15T10:30:00Z

  - version: 2
    geom_blob_ref: blob_def456
    topology_id: topo_002
    metadata_blob_ref: blob_meta2
    provenance_id: prov_002
    created_at: 2024-01-15T11:00:00Z
    previous_version: 1

  - version: 3
    geom_blob_ref: blob_ghi789
    topology_id: topo_003
    metadata_blob_ref: blob_meta3
    provenance_id: prov_003
    created_at: 2024-01-15T11:30:00Z
    previous_version: 2
```

Topology Metadata
```yaml
topology_id: topo_003
entity_id: ent_01HXYZ
element_count:
  faces: 6
  edges: 12
  vertices: 8
elements:
  faces:
    - index: 0
      semantic: "top"
      edge_indices: [0, 1, 2, 3]
    - index: 1
      semantic: null
      edge_indices: [4, 5, 6, 7]
    # ...
  edges:
    - index: 0
      vertex_indices: [0, 1]
    # ...
  vertices:
    - index: 0
      position_ref: [x, y, z]  # For debugging only
    # ...
```

Provenance Storage

Provenance Record Format
```yaml
provenance_id: prov_003
source_type: "parametric"  # or "imported", "generated"
source_ref: "step_042"     # Timeline step or import record
timestamp: 2024-01-15T11:30:00Z
operator_chain:
  - operator_id: op_extrude
    operator_version: "1.2.0"
    params:
      height: 10.0
      direction: [0, 0, 1]
    input_refs:
      - ent_01XYZ:version=2
parent_provenance: prov_002
```

Import Provenance
```yaml
provenance_id: prov_import_001
source_type: "imported"
source_ref:
  file_path: "/imports/part.3dm"
  file_hash: blob_file123
  import_timestamp: 2024-01-15T09:00:00Z
  import_settings:
    units: "mm"
    tolerance: 0.001
timestamp: 2024-01-15T09:00:00Z
operator_chain: []
parent_provenance: null
```

Timeline Storage

Timeline Step Record
```yaml
step_id: step_042
order_index: 42
created_at: 2024-01-15T11:30:00Z

operator_spec:
  operator_id: op_extrude
  operator_version: "1.2.0"
  name: "Extrude"
  determinism: "deterministic"

inputs:
  - ref_id: ref_001
    target: "ent_01XYZ:face/0"
    resolved_version: 2

params:
  height: 10.0
  direction: [0, 0, 1]

outputs:
  - entity_id: ent_01ABC
    version: 1

status: "success"  # or "failed", "invalidated"

diagnostics:
  execution_time_ms: 150
  warnings: []
  errors: []

reproducibility:
  status: "reproducible"

cache_key: "cache_abc123def456"
```

Evaluation Cache

Cache Entry Format
```yaml
cache_key: cache_abc123def456
created_at: 2024-01-15T11:30:00Z
last_accessed: 2024-01-15T14:00:00Z
ttl: 86400  # seconds, or null for permanent

operator_id: op_extrude
operator_version: "1.2.0"
inputs_hash: hash_inputs
params_hash: hash_params

result:
  output_blob_refs:
    - blob_ghi789
  output_entity_versions:
    - entity_id: ent_01ABC
      version: 1
  execution_metadata:
    duration_ms: 150
    peak_memory_mb: 256
```

Cache Eviction Policy
1. LRU eviction when cache exceeds size limit.
2. TTL-based expiration for time-sensitive caches.
3. Version-based invalidation when operator version changes.
4. Explicit purge on user request.

Index Structures

Entity Index
```
entity_id -> entity_record_location
```

Blob Reference Index
```
blob_id -> [entity_ids using this blob]
```

Timeline Index
```
step_id -> step_record_location
order_index -> step_id
```

Provenance Index
```
provenance_id -> provenance_record_location
entity_id -> [provenance_ids]
```

Search Index (Optional)
```
semantic_qualifier -> [topology_paths]
operator_id -> [step_ids]
```

File Layout (On-Disk)

Directory Structure
```
<project_root>/
├── monad.project           # Project metadata
├── blobs/
│   ├── a1/
│   │   └── b2c3d4...       # Blob files (sharded by prefix)
│   └── ...
├── entities/
│   └── entities.db         # SQLite or similar for entity records
├── timeline/
│   └── timeline.db         # SQLite for timeline steps
├── provenance/
│   └── provenance.db       # SQLite for provenance records
├── cache/
│   └── eval_cache.db       # Ephemeral evaluation cache
└── indices/
    └── indices.db          # Lookup indices
```

Serialization Formats

Primary: MessagePack
- Chosen for compactness and speed.
- Schema-less but with documented structure.

Secondary: JSON
- For human-readable exports and debugging.
- Not used for primary storage.

Geometry: OpenNURBS 3DM
- Native format for NURBS geometry.
- Wrapped in Monad blob envelope.

Integrity and Recovery

Write-Ahead Log
- All mutations logged before commit.
- Enables recovery on crash.

Checksums
- Every blob has embedded checksum.
- Periodic integrity scans detect corruption.

Backup Format
- Full project export as single archive.
- Includes all blobs, records, and indices.
- Importable as new project or for recovery.
