# Changelog

All notable changes to the FEBO specification will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Planned
- JSON Schema for validation
- Reference parser (Python)
- MPS → FEBO converter
- QUBO → FEBO converter

---

## [0.1.1] - 2025-01-20

### Added
- **Functions registry**: External function hooks for `functional` terms (python, wasm, shared_lib, http)
- **`groups` interaction**: Generalized group-based indexing for constraint structures
- **`none` interaction**: Explicit scalar interaction type
- **`bind` field**: Explicit index symbol binding for interactions (`bind: p`, `bind: [p, q]`)
- **Data types**: `dtype` field (float64, float32, int64, int32, bool) and `storage` hints
- **Ragged shapes**: `[K, "*"]` notation for variable-length dimensions
- **Indexed reductions spec**: Deterministic semantics for `sum_p()`, `prod_j()`, etc.
- **Reduction domains**: Explicit rules per interaction type (all_indices, groups, sparse)
- **Realtime metadata**: `warm_start`, `streaming_updates`, `latency_target_ms` hints
- **Example**: `binpack3d_realtime.febo.yaml` demonstrating candidate-placement formulation

### Changed
- `one_hot` is now an alias for `groups`
- Clarified that `groups` generates ordered lists of unique indices (duplicates invalid)
- `source`/`groups` resolution: if value matches `data:` key → data reference, else file path or inline
- Index bounds validated by `index_base` (1-based: `1..P`, 0-based: `0..P-1`)
- n-ary terms: clarified that validity determined by bound-symbol checks
- Two-level aggregation for groups: within-group reduction + across-groups aggregation

## [0.1.0] - 2025-01-20

### Added

**Core Specification**
- Five-level hierarchy: Terms → Aggregators/Interactions → Components → Hamiltonians → Ensemble
- Term types: constant, unary, binary, n-ary (analytic and functional)
- Aggregators: Sum, Prod, Max, Min, LogSumExp, LogProd, Integral, Identity
- Interaction structures: all_indices, Sparse, Low-Rank, Laplacian, One-Hot, Index-Set
- Subsystem and Coupling Hamiltonians
- Ensemble formulation
- Parameters section for model/instance separation

**YAML Specification**
- Normative YAML format specification (YAML_SPEC.md)
- JSON Schema for validation (febo-v0.1.schema.json)
- File extension: `.febo.yaml`
- Expression grammar: febo_expr_v1

**Documentation**
- Conceptual specification (SPECIFICATION.md)
- Real-world applications (APPLICATIONS.md)
- Comparison with Pyomo, AMPL, Gurobi, CVXPY, MPS (COMPARISON.md)
- Quick-start README
- FAQ
- Contributing guidelines

**Examples**
- `griewank.febo.yaml` — Benchmark with Prod aggregator
- `portfolio.febo.yaml` — Multi-objective with constraints
- `qubo.febo.yaml` — Quantum annealer compatible
- `supply_chain.febo.yaml` — Multi-echelon with Coupling Hamiltonians
- `robust.febo.yaml` — Minimax with Max aggregator

### Design Decisions
- `all_indices` with `ranges` covers cartesian products (no separate `dense` type)
- Sparse interactions use `source` (file) or `pairs` (inline)
- `alpha` and `weight` default to 1 when omitted
- `one_hot` generates index groups; constraint enforcement is user-provided
- Hamiltonian type determined by presence of `couples` field (no inference)

### Notes
- This is the initial public release
- Specification is subject to change before v1.0

---

## Version History

| Version | Date | Status |
|---------|------|--------|
| 0.1.0 | 2025-01-20 | Current |
| 1.0.0 | TBD | Planned stable release |
