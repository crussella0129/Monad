Operator Determinism Contract Specification

Purpose
Define how operators declare and enforce determinism, and how nondeterministic operations are handled within the evaluation pipeline.

Definitions

Deterministic Operator
An operator where identical inputs, parameters, and operator version always produce bit-identical outputs.

Conditionally Deterministic Operator
An operator that is deterministic given certain constraints (e.g., fixed random seed, specific platform).

Nondeterministic Operator
An operator that may produce different outputs on re-evaluation (e.g., random sampling without seed, external data fetch).

Declaration Schema

OperatorSpec Extension
```
OperatorSpec:
  ...
  determinism:
    type: "deterministic" | "conditional" | "nondeterministic"
    conditions: [string]?          # For conditional: list of required conditions
    seed_param: string?            # Parameter name for seed (if applicable)
    isolation_required: bool       # Must run in isolated sandbox
```

Examples
```yaml
# Deterministic operator
name: CreateLine
determinism:
  type: deterministic

# Conditional operator (deterministic with seed)
name: RandomPoints
determinism:
  type: conditional
  conditions: ["seed_provided"]
  seed_param: "random_seed"

# Nondeterministic operator
name: FetchExternalMesh
determinism:
  type: nondeterministic
  isolation_required: true
```

Evaluation Rules

Deterministic Operators
1. Cache outputs keyed by (operator_id, version, inputs_hash, params_hash).
2. On cache hit, skip execution and return cached result.
3. Cache is valid until operator version changes or inputs invalidate.

Conditional Operators
1. If conditions are met (e.g., seed provided), treat as deterministic.
2. If conditions unmet, treat as nondeterministic.
3. Emit warning if conditions are not met: `ConditionalDeterminismWarning`.

Nondeterministic Operators
1. Always execute; never return cached results for new evaluations.
2. Store execution result with timestamp and execution context.
3. Mark TimelineStep as `non_reproducible`.
4. Downstream steps inherit `non_reproducible` flag.
5. Emit diagnostic: `NondeterministicStepWarning`.

Reproducibility Tracking

TimelineStep Extension
```
TimelineStep:
  ...
  reproducibility:
    status: "reproducible" | "non_reproducible" | "conditionally_reproducible"
    reasons: [string]              # List of nondeterministic dependencies
    last_execution: timestamp
    execution_context:
      platform: string
      operator_version: string
      seed_values: {param: value}
```

Cache Policy

Cache Key Construction
```
cache_key = hash(
  operator_id,
  operator_version,
  sorted(inputs.map(i => i.content_hash)),
  sorted(params.map(p => (p.name, p.value)))
)
```

Cache Invalidation
1. Operator version change: invalidate all cached results for that operator.
2. Input content change: invalidate cached results with that input.
3. Param value change: invalidate cached results with that param set.
4. Explicit purge: user-initiated cache clear.

Isolation Requirements

Sandbox Model
Operators with `isolation_required: true` must:
1. Execute in a sandboxed environment (no shared state).
2. Have no access to filesystem except explicit inputs.
3. Have no network access except through declared external dependencies.
4. Log all external interactions.

External Dependency Declaration
```
OperatorSpec:
  ...
  external_dependencies:
    - type: "http"
      url_pattern: "https://api.example.com/*"
    - type: "file"
      path_pattern: "/data/external/*"
```

Verification and Auditing

Determinism Verification
1. On debug/audit mode, re-execute deterministic operators and compare outputs.
2. If outputs differ, emit `DeterminismViolationError`.
3. Log violation with full execution context.

Audit Log Schema
```
{
  step_id: string,
  operator_id: string,
  expected_hash: string,
  actual_hash: string,
  execution_context: {...},
  violation_type: "output_mismatch" | "timing_variance" | "external_change"
}
```

Migration Path
For operators transitioning from unknown to declared determinism:
1. Default to `nondeterministic` if undeclared.
2. Emit `UndeclaredDeterminismWarning`.
3. Require explicit declaration before operator can be used in production timelines.
