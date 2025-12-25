Architecture Modules

Kernel.Core
- OpenNURBS-backed primitives and operations.
- Pure-by-default operators, explicit context in/out.
- No history, no evaluation scheduling, no UI.

Kernel.Model
- Geometry instances, topology, metadata, identity, provenance.
- Identity graph and strict reference resolution.
- Append-only, versioned storage schema.

Kernel.Operators
- Declarative command library and parameter schema.
- Operator contracts, typing, validation.
- Side effects only when declared.

Synth.Tree
- Parametric synthesizer: trees, ranges, combinators, conditionals.
- Outputs sets/families of geometry or operator specs.
- Debugging and introspection surfaces.

History.Timeline
- Linear, ordered history of steps with explicit inputs/outputs.
- Rollback, replay, branching, deterministic reevaluation.
- Stores evaluation results and failure records.

Eval.Engine
- Dependency extraction across tree and timeline.
- Scoped recomputation and invalidation.
- Diagnostics and explainability queries.

Storage.Repository
- Append-only storage of geometry, specs, provenance, evaluation artifacts.
- Content-addressable geometry blobs.
- Identity map for stable references.

API.Bindings
- Python bindings over Model/Operators/Timeline.
- Explicit session/context; no implicit globals.
