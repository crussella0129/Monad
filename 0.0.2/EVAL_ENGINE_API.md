Eval.Engine API Specification

Purpose
Define interfaces for DAG construction, dependency metadata, incremental recomputation, and cache policy for the evaluation engine.

Core Interfaces

EvalContext
```cpp
class EvalContext {
public:
    // Session identification
    SessionId session_id() const;

    // Entity resolution
    Result<GeometryEntity> resolve_entity(EntityId id, Version version = LATEST);
    Result<SubEntity> resolve_path(TopologyPath path);

    // Blob access
    Result<GeomBlob> get_blob(BlobId id);
    BlobId store_blob(GeomBlob blob);

    // Cache access
    Optional<CacheEntry> cache_lookup(CacheKey key);
    void cache_store(CacheKey key, CacheEntry entry);

    // Diagnostics
    DiagnosticsCollector& diagnostics();

    // Configuration
    EvalConfig config() const;
};
```

EvalEngine
```cpp
class EvalEngine {
public:
    // Construction
    static EvalEngine create(EvalConfig config);

    // DAG operations
    Result<EvalDAG> build_dag(Timeline& timeline, SynthTree& tree);
    Result<EvalDAG> build_dag_partial(Timeline& timeline, StepRange range);

    // Evaluation
    Result<EvalResult> evaluate(EvalDAG& dag, EvalContext& ctx);
    Result<EvalResult> evaluate_incremental(
        EvalDAG& dag,
        EvalContext& ctx,
        ChangeSet changes
    );

    // Invalidation
    InvalidationSet compute_invalidation(EvalDAG& dag, ChangeSet changes);

    // Explainability
    Result<ExplainResult> explain(EvalDAG& dag, EntityId entity);
    Result<DependencyChain> trace_dependency(EvalDAG& dag, TopologyPath path);
};
```

DAG Construction

EvalDAG Structure
```cpp
class EvalDAG {
public:
    // Node access
    size_t node_count() const;
    EvalNode& node(NodeId id);
    Iterator<EvalNode> nodes();

    // Edge access
    Iterator<EvalEdge> incoming_edges(NodeId id);
    Iterator<EvalEdge> outgoing_edges(NodeId id);

    // Topology
    Iterator<NodeId> roots();           // Nodes with no inputs
    Iterator<NodeId> leaves();          // Nodes with no outputs
    Iterator<NodeId> topological_order();

    // Subgraph extraction
    EvalDAG subgraph(Set<NodeId> nodes);
    EvalDAG affected_subgraph(Set<NodeId> changed_nodes);
};

class EvalNode {
public:
    NodeId id() const;
    NodeType type() const;              // TimelineStep, SynthNode, Input

    // For TimelineStep nodes
    Optional<StepId> step_id() const;

    // For SynthNode nodes
    Optional<SynthNodeId> synth_node_id() const;

    // State
    EvalState state() const;            // Pending, Valid, Invalid, Error
    Optional<CacheKey> cache_key() const;

    // Metadata
    DependencyMetadata dependencies() const;
};

class EvalEdge {
public:
    NodeId source() const;
    NodeId target() const;
    EdgeType type() const;              // DataFlow, Control, Reference

    // For DataFlow edges
    Optional<OutputSlot> source_slot() const;
    Optional<InputSlot> target_slot() const;
};
```

DAG Construction Algorithm
```
BuildDAG(timeline, tree):
    dag = new EvalDAG()

    # Add tree nodes
    for node in tree.topological_order():
        eval_node = dag.add_node(SynthNode(node))
        for input in node.inputs:
            if input.is_tree_reference:
                dag.add_edge(input.source_node, eval_node, DataFlow)
            else:
                input_node = dag.add_node(Input(input))
                dag.add_edge(input_node, eval_node, DataFlow)

    # Add timeline steps
    for step in timeline.steps:
        step_node = dag.add_node(TimelineStep(step))
        for input_ref in step.input_refs:
            if input_ref.from_tree:
                tree_node = tree.find_output_node(input_ref)
                dag.add_edge(tree_node, step_node, DataFlow)
            else:
                prev_step = timeline.find_producer(input_ref)
                if prev_step:
                    dag.add_edge(prev_step, step_node, DataFlow)

        # Add ordering edges for timeline linearity
        if step.order_index > 0:
            prev = timeline.step_at(step.order_index - 1)
            dag.add_edge(prev, step_node, Control)

    return dag
```

Dependency Metadata

DependencyMetadata Structure
```cpp
class DependencyMetadata {
public:
    // Input dependencies
    Vector<EntityRef> entity_inputs() const;
    Vector<ParamRef> param_inputs() const;
    Vector<NodeRef> upstream_nodes() const;

    // Output declarations
    Vector<EntityRef> entity_outputs() const;

    // Sensitivity
    Set<ParamId> sensitive_params() const;      // Params that affect output
    Set<EntityId> sensitive_entities() const;   // Entities that affect output

    // Hashing
    Hash inputs_hash() const;
    Hash params_hash() const;
    Hash combined_hash() const;
};

class ParamRef {
public:
    ParamId id() const;
    String name() const;
    ParamType type() const;
    Any value() const;
    Hash value_hash() const;
};

class EntityRef {
public:
    EntityId entity_id() const;
    Version version() const;
    Optional<TopologyPath> sub_path() const;
    Hash content_hash() const;
};
```

Incremental Recomputation

ChangeSet
```cpp
class ChangeSet {
public:
    // Constructors
    static ChangeSet param_change(ParamId id, Any old_val, Any new_val);
    static ChangeSet entity_change(EntityId id, Version old_ver, Version new_ver);
    static ChangeSet step_edit(StepId id, OperatorSpec old_spec, OperatorSpec new_spec);
    static ChangeSet step_insert(StepId id, size_t order_index);
    static ChangeSet step_delete(StepId id);

    // Combination
    ChangeSet merge(ChangeSet other);

    // Queries
    Set<ParamId> changed_params() const;
    Set<EntityId> changed_entities() const;
    Set<StepId> changed_steps() const;
};
```

Invalidation Algorithm
```
ComputeInvalidation(dag, changes):
    invalid = Set<NodeId>()

    # Find directly affected nodes
    for node in dag.nodes():
        if node.is_affected_by(changes):
            invalid.add(node.id)

    # Propagate invalidation downstream
    worklist = Queue(invalid)
    while not worklist.empty():
        node_id = worklist.pop()
        for edge in dag.outgoing_edges(node_id):
            if edge.target not in invalid:
                invalid.add(edge.target)
                worklist.push(edge.target)

    return InvalidationSet(invalid)
```

Incremental Evaluation
```
EvaluateIncremental(dag, ctx, changes):
    invalid = ComputeInvalidation(dag, changes)

    # Collect valid cached results
    cached = {}
    for node in dag.nodes():
        if node.id not in invalid:
            if node.cache_key:
                cached[node.id] = ctx.cache_lookup(node.cache_key)

    # Evaluate in topological order, skipping cached
    results = {}
    for node in dag.topological_order():
        if node.id in cached and cached[node.id].is_some():
            results[node.id] = cached[node.id].unwrap()
        else:
            inputs = gather_inputs(node, results, ctx)
            result = evaluate_node(node, inputs, ctx)
            results[node.id] = result
            if node.is_cacheable():
                ctx.cache_store(node.cache_key, result)

    return EvalResult(results)
```

Cache Policy

CacheConfig
```cpp
struct CacheConfig {
    size_t max_size_bytes = 1 << 30;    // 1 GB default
    Duration default_ttl = Duration::hours(24);
    bool cache_deterministic = true;
    bool cache_conditional = true;
    bool cache_nondeterministic = false;
    EvictionPolicy eviction = LRU;
};

enum EvictionPolicy {
    LRU,        // Least recently used
    LFU,        // Least frequently used
    FIFO,       // First in first out
    SIZE,       // Largest entries first
};
```

CacheKey Construction
```cpp
CacheKey compute_cache_key(EvalNode& node, EvalContext& ctx) {
    Hasher h;

    // Operator identity
    h.update(node.operator_id());
    h.update(node.operator_version());

    // Input content hashes (sorted for determinism)
    auto inputs = node.dependencies().entity_inputs();
    sort(inputs.begin(), inputs.end(), by_entity_id);
    for (auto& input : inputs) {
        h.update(input.entity_id());
        h.update(input.content_hash());
    }

    // Parameter hashes (sorted for determinism)
    auto params = node.dependencies().param_inputs();
    sort(params.begin(), params.end(), by_param_id);
    for (auto& param : params) {
        h.update(param.id());
        h.update(param.value_hash());
    }

    return CacheKey(h.finalize());
}
```

Explainability API

ExplainResult
```cpp
class ExplainResult {
public:
    EntityId target_entity() const;

    // Direct producer
    NodeId producer_node() const;
    StepId producer_step() const;

    // Full dependency chain
    Vector<DependencyLink> dependency_chain() const;

    // Parameter influences
    Vector<ParamInfluence> param_influences() const;

    // Visualization
    String to_text() const;
    String to_dot() const;     // GraphViz format
};

class DependencyLink {
public:
    NodeId from() const;
    NodeId to() const;
    DependencyType type() const;    // Direct, Transitive
    String description() const;
};

class ParamInfluence {
public:
    ParamId param() const;
    String name() const;
    Any value() const;
    NodeId affects_node() const;
    float sensitivity() const;      // Optional: how much output changes
};
```

Explainability Query
```
Explain(dag, entity_id):
    # Find producer node
    producer = find_node_producing(dag, entity_id)
    if not producer:
        return Error("Entity not found in DAG")

    # Trace backwards
    chain = []
    visited = Set()
    worklist = Queue([producer])

    while not worklist.empty():
        node = worklist.pop()
        if node in visited:
            continue
        visited.add(node)

        for edge in dag.incoming_edges(node):
            chain.append(DependencyLink(edge.source, node))
            worklist.push(edge.source)

    # Collect parameter influences
    params = []
    for node in visited:
        for param in node.dependencies().param_inputs():
            params.append(ParamInfluence(param, node))

    return ExplainResult(entity_id, producer, chain, params)
```

Error Handling

EvalError Types
```cpp
enum EvalErrorType {
    MissingInput,           // Required input not available
    InvalidReference,       // Reference resolution failed
    OperatorFailed,         // Operator execution threw
    TypeMismatch,           // Input/output type incompatible
    CyclicDependency,       // DAG contains cycle (should not happen)
    Timeout,                // Evaluation exceeded time limit
    ResourceExhausted,      // Memory or other resource limit
    Cancelled,              // User cancelled evaluation
};

class EvalError {
public:
    EvalErrorType type() const;
    NodeId failing_node() const;
    String message() const;
    Optional<DependencyChain> dependency_chain() const;
    DiagnosticsSnapshot diagnostics() const;
};
```

Error Recovery
1. On error, mark failing node and all downstream as Error state.
2. Preserve successfully computed upstream results in cache.
3. Return partial result with error diagnostics.
4. Allow retry after error is fixed without full re-evaluation.
