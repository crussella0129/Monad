# Monad CAD 0.0.2 Summary

## What I built

Created `0.0.2/` with detailed design specifications addressing all five roadmap items from 0.0.1:

### 1. TOPOLOGY_PATH.md
- Defined stable path encoding: `<entity_id>:<element_type>/<index>[/<sub_path>]`
- UUID v7 for entity IDs (time-ordered)
- Index stability rules: indices assigned at creation, never reused
- Semantic qualifiers (top, bottom, etc.) with explicit ambiguity handling
- Text and binary serialization formats with schema versioning
- Diagnostic schema for resolution failures with topology diffs

### 2. OPERATOR_DETERMINISM.md
- Three determinism levels: deterministic, conditional, nondeterministic
- Declaration schema in OperatorSpec with conditions and seed parameters
- Evaluation rules: cache deterministic, execute always for nondeterministic
- Reproducibility tracking in TimelineStep with execution context
- Cache key construction from operator + inputs + params hashes
- Sandbox model for isolated operators with external dependencies

### 3. IDENTITY_LIFECYCLE.md
- Entity ID format using ULID for uniqueness and time-ordering
- Immutable vs versioned fields (entity_id, provenance never change)
- Soft delete with GC eligibility rules
- Reference states: Valid, Stale, Broken, Orphaned
- Manual and batch repair workflows with diagnostic presentation
- GC safety: never automatic, requires user confirmation

### 4. STORAGE_FORMAT.md
- Content-addressable blobs using BLAKE3 (256-bit)
- Blob format with header (magic, version, compression), content, footer (checksum)
- Entity record format with version history
- Topology metadata schema with element indices and semantics
- Provenance records for parametric and imported geometry
- Timeline step records with reproducibility metadata
- File layout: SQLite for records, sharded directories for blobs
- Write-ahead log for crash recovery

### 5. EVAL_ENGINE_API.md
- EvalContext: session management, entity resolution, blob/cache access
- EvalEngine: DAG build, evaluate, invalidate, explain
- EvalDAG: nodes, edges, topological ordering, subgraph extraction
- DAG construction algorithm from timeline + synth tree
- DependencyMetadata: entity/param inputs, sensitivity tracking
- ChangeSet and invalidation propagation algorithm
- Incremental evaluation with cache lookup
- CacheConfig with eviction policies (LRU, LFU, FIFO, SIZE)
- ExplainResult for full dependency chain visualization
- Error types and recovery strategies

## Why

The 0.0.1 roadmap identified five critical design areas that needed detailed specification before implementation could begin. These specs transform high-level architectural principles into concrete, implementable contracts with:
- Explicit data formats
- Algorithm pseudocode
- Error handling strategies
- Interface definitions in C++-style syntax

This ensures future agents and contributors have unambiguous guidance for implementation.

## Problems encountered

None. This was a pure design specification phase with no code execution or compilation required. The specifications were derived logically from the 0.0.1 architectural constraints.

## How to run or test

No runnable code yet. Validation is by architectural review:
1. Review each spec for internal consistency
2. Cross-reference specs (e.g., TOPOLOGY_PATH referenced in IDENTITY_LIFECYCLE)
3. Verify alignment with 0.0.1 invariants and failure semantics

## Next steps

1. **Prototype implementation**: Begin C++ scaffolding for core data structures
   - GeometryEntity, Provenance, OperatorSpec structs
   - TopologyPath parser and resolver
   - Basic blob storage with BLAKE3 hashing

2. **OpenNURBS integration**: Evaluate OpenNURBS capabilities and limitations
   - Test geometry blob serialization/deserialization
   - Identify any extensions needed

3. **Eval.Engine skeleton**: Implement minimal DAG construction
   - Node and edge types
   - Topological sort
   - Simple evaluation loop (no caching)

4. **Python bindings design**: Define pybind11 surface for scripting
   - Minimal EvalContext exposure
   - Entity creation and inspection

5. **Test harness**: Create validation framework
   - Determinism verification tests
   - Reference resolution tests
   - Cache correctness tests
