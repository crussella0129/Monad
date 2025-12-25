Failure Semantics and Guardrails

Reference and Identity
- Identity is explicit and versioned; geometry blobs are immutable.
- References resolve by entity_id + topology path.
- If resolution fails, evaluation stops with diagnostics.
- No silent remapping of faces/edges.

Imported Geometry
- Imported entities are first-class with provenance.
- Import changes trigger reference revalidation.
- Broken references fail explicitly.

Hard Problems and Guardrails
1) Topology instability
- Guardrail: no auto-matching; explicit rebind only.
- Diagnostics include old vs new topology paths.

2) Mixed provenance
- Guardrail: same reference model for imported and parametric geometry.
- Failure if import is missing or shape changes invalidate refs.

3) Nondeterminism from plugins
- Guardrail: operators declare determinism; nondeterministic steps flagged.
- Non-reproducible steps are explicitly marked.

4) Partial recomputation correctness
- Guardrail: strict dependency graph; no hidden globals.
- Eval.Engine refuses skip if metadata incomplete.

5) State leakage
- Guardrail: operators are pure unless declared side effects.
- Side effects are sandboxed and logged.
