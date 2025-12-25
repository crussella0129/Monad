Roadmap (Near-Term Design Tasks)

1) Topology path encoding
- Define a stable, human-auditable topology path schema.
- Specify serialization and versioning strategy.

2) Operator determinism contract
- Define deterministic vs nondeterministic operator tags.
- Define how nondeterministic results are cached and invalidated.

3) Identity lifecycle
- Define sub-entity identity rules (face/edge) and creation/deletion policies.
- Define reference repair workflows and diagnostics UX.

4) Storage format details
- Choose content-addressable scheme for geometry blobs.
- Define provenance and evaluation artifact schemas.

5) Eval.Engine interfaces
- Define DAG construction APIs and dependency metadata requirements.
- Define incremental recomputation strategy and cache policy.
