Evaluation Contract and Dependency Model

Contract
- Deterministic: identical inputs, operator versions, and context yield identical results.
- Scoped: only affected nodes/steps recompute; unaffected results reused.
- Explainable: every output traced to upstream inputs and operators.
- Inspectable: failures emit structured diagnostics with dependency chains.

Dependency Extraction
- Timeline steps declare input refs and output refs.
- Tree nodes declare parameter dependencies and upstream nodes.
- Eval.Engine builds a DAG: Tree DAG feeds Timeline chain.

Change Propagation
- Command param change: invalidate affected steps and recompute downstream.
- Tree param change: recompute affected subtrees and dependent timeline steps.
- Timeline edit: invalidate downstream steps; preserve upstream caches.
