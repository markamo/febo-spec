# FEBO: Functional Energy-Based Optimization

> A universal specification language for optimization problems

[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](LICENSE)
[![Version](https://img.shields.io/badge/version-0.1.1-green.svg)](CHANGELOG.md)
[![Spec](https://img.shields.io/badge/spec-YAML-orange.svg)](YAML_SPEC.md)

---

## What is FEBO?

**FEBO** (Functional Energy-Based Optimization) is a specification language that represents any optimization problem as an energy landscape to be minimized. Think of it as **"HTML for optimization"** — a portable, human-readable format that any solver can consume.

```
FEBO Model (.febo.yaml)  →  Parser  →  Any Solver (Xtellix, Gurobi, CVXPY, D-Wave, ...)
```

### Why FEBO?

| Problem | Traditional Approach | FEBO Approach |
|---------|---------------------|---------------|
| Worst-case optimization | Reformulate with epigraph variables | Native `agg: max` |
| Multi-objective | Manual Pareto or weighted sum | Explicit Hamiltonian weights |
| Coupled subsystems | Monolithic flat model | Coupling Hamiltonians with `couples:` |
| Debugging infeasibility | "Infeasible" | "H_coupling violated at Plant-3 → Customer-15" |
| Billion-scale problems | Out of memory | Structure-aware solvers exploit hierarchy |

---

## Quick Example: Portfolio Optimization

```yaml
febo_version: "0.1.1"
name: portfolio

parameters:
  n: {type: int, default: 100}
  risk_aversion: {type: float, default: 0.5}

variables:
  w: {type: continuous, shape: [n], bounds: [0, 1]}

data:
  mu: {source: "returns.csv", shape: [n]}
  Sigma: {source: "covariance.csv", shape: [n, n]}

interactions:
  all_i:  {type: all_indices, range: [1, n]}
  all_ij: {type: all_indices, ranges: {i: [1, n], j: [1, n]}}

terms:
  return: {type: analytic, arity: unary, expr: "mu[i] * w[i]"}
  risk:   {type: analytic, arity: binary, expr: "Sigma[i,j] * w[i] * w[j]"}
  budget: {type: analytic, arity: n-ary, expr: "(sum(w) - 1)^2"}

components:
  total_return: {term: return, interaction: all_i, agg: sum}
  total_risk:   {term: risk, interaction: all_ij, agg: sum}
  budget_pen:   {term: budget}

hamiltonians:
  H_return: {components: [{use: total_return, alpha: -1}]}
  H_risk:   {components: [{use: total_risk}]}
  H_budget: {components: [{use: budget_pen}]}

ensemble:
  - {use: H_return}
  - {use: H_risk, weight: risk_aversion}
  - {use: H_budget, weight: 10000}
```

**Result**: Minimize `E(w) = -Σᵢ μᵢwᵢ + λ·wᵀΣw + 10000·(Σwᵢ - 1)²`

---

## The FEBO Hierarchy

```
┌─────────────────────────────────────────────────────────────┐
│  Ensemble                                                    │
│  └── Weighted sum of Hamiltonians                           │
├─────────────────────────────────────────────────────────────┤
│  Hamiltonians                                                │
│  └── H_objective, H_constraints, H_coupling                 │
├─────────────────────────────────────────────────────────────┤
│  Components                                                  │
│  └── term + interaction + aggregator                        │
├─────────────────────────────────────────────────────────────┤
│  Terms              │  Interactions      │  Aggregators      │
│  └── x[i]²          │  └── all_indices   │  └── sum          │
│  └── Q[i,j]x[i]x[j] │  └── sparse        │  └── prod         │
│  └── dock(a,b)      │  └── groups        │  └── max/min      │
└─────────────────────────────────────────────────────────────┘
```

---

## Key Features

### Native Aggregators Beyond Sum

```yaml
# Traditional: requires epigraph reformulation (1 variable + S constraints)
# FEBO: native max aggregator
components:
  worst_case: {term: violation, interaction: scenarios, agg: max}
```

### Explicit Subsystem Coupling

```yaml
hamiltonians:
  H_production:
    components: [{use: prod_cost}]
    
  H_distribution:
    components: [{use: ship_cost}]
    
  H_coupling:
    couples: [production, shipment]
    components: [{use: interface_balance}]
```

### Realtime / Streaming Optimization

```yaml
metadata:
  realtime:
    warm_start: true
    streaming_updates:
      - data: candidates
      - data: conflicts
    latency_target_ms: 50
```

### Groups for Constraint Structures

```yaml
interactions:
  by_item: {type: groups, groups: candidates_by_item, bind: p}

terms:
  exactly_one: {type: analytic, arity: n-ary, expr: "(sum_p(y[p]) - 1)^2"}
```

---

## Examples

| Example | Description | Key Features |
|---------|-------------|--------------|
| [griewank.febo.yaml](examples/griewank.febo.yaml) | Benchmark function | `agg: prod`, billion-scale |
| [portfolio.febo.yaml](examples/portfolio.febo.yaml) | Mean-variance optimization | Multi-objective, parameters |
| [supply_chain.febo.yaml](examples/supply_chain.febo.yaml) | Multi-echelon logistics | Coupling Hamiltonians |
| [robust.febo.yaml](examples/robust.febo.yaml) | Minimax robust optimization | `agg: max` |
| [binpack3d_realtime.febo.yaml](examples/binpack3d_realtime.febo.yaml) | 3D bin packing | `groups`, `sparse`, realtime |
| [qubo.febo.yaml](examples/qubo.febo.yaml) | Quadratic unconstrained binary | D-Wave compatible |

---

## Documentation

| Document | Description |
|----------|-------------|
| [YAML_SPEC.md](YAML_SPEC.md) | **Normative specification** — syntax, semantics, validation rules |
| [SPECIFICATION.md](SPECIFICATION.md) | Conceptual foundations — Terms, Aggregators, Hamiltonians |
| [COMPARISON.md](COMPARISON.md) | FEBO vs Pyomo, AMPL, Gurobi, CVXPY, MPS |
| [APPLICATIONS.md](APPLICATIONS.md) | Real-world formulations |
| [schema/febo-v0.1.1.schema.json](schema/febo-v0.1.1.schema.json) | JSON Schema for validation |
| [FAQ.md](FAQ.md) | Frequently asked questions |
| [CHANGELOG.md](CHANGELOG.md) | Version history |

---

## Comparison with Traditional Frameworks

| Feature | FEBO | Pyomo | AMPL | Gurobi | CVXPY |
|---------|------|-------|------|--------|-------|
| Sum aggregator | ✅ | ✅ | ✅ | ✅ | ✅ |
| Prod aggregator | ✅ Native | ❌ Log transform | ✅ | ❌ | ❌ |
| Max aggregator | ✅ Native | ❌ Epigraph | ❌ | ❌ | ❌ |
| Multi-objective | ✅ Native | ⚠️ Manual | ⚠️ | ⚠️ | ⚠️ |
| Subsystem structure | ✅ | ❌ | ❌ | ❌ | ❌ |
| Auditability | ✅ Built-in | ❌ | ❌ | ❌ | ❌ |
| Billion-scale | ✅ | ❌ | ❌ | ❌ | ❌ |

See [COMPARISON.md](COMPARISON.md) for detailed side-by-side examples.

---

## Getting Started

```bash
# Clone the repository
git clone https://github.com/xtellix/febo-spec.git
cd febo-spec

# Explore examples
ls examples/

# Validate a FEBO file (requires jsonschema)
pip install jsonschema pyyaml
python -c "
import yaml, jsonschema, json
schema = json.load(open('schema/febo-v0.1.1.schema.json'))
doc = yaml.safe_load(open('examples/portfolio.febo.yaml'))
jsonschema.validate(doc, schema)
print('Valid FEBO file!')
"
```

---

## Roadmap

- [x] **v0.1** — Core specification (Terms, Aggregators, Interactions, Hamiltonians, Ensemble)
- [x] **v0.1.1** — Groups, bind, realtime hints, indexed reductions
- [ ] **v0.2** — Reference parser (Python)
- [ ] **v0.3** — Converters (QUBO ↔ FEBO, MPS → FEBO)
- [ ] **v0.4** — Solver adapters (Gurobi, CVXPY, OR-Tools)
- [ ] **v1.0** — Stable specification

---

## Design Philosophy

> **FEBO is to optimization what SQL is to databases.**

1. **Declarative**: Describe *what* to optimize, not *how*
2. **Portable**: One model, any backend
3. **Compositional**: Build complex from simple
4. **Auditable**: Every energy term is traceable
5. **Scalable**: Structure enables billion-variable problems

---

## Contributing

We welcome contributions! See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

- 🐛 **Bug reports**: Open an issue
- 💡 **Feature requests**: Open a discussion
- 📝 **Spec improvements**: Submit a PR

---

## License

This specification is released under the [Apache License 2.0](LICENSE).

The FEBO specification is open. Implementations (parsers, converters, solvers) may have their own licenses.

---

## About

FEBO is developed by [Xtellix, Inc.](https://xtellix.com)

**Author**: Mark Amo-Boateng, PhD

---

<p align="center">
<i>Terms contribute. Interactions select. Aggregators combine.<br>Hamiltonians structure. Ensembles unify.<br><b>Minimize the energy.</b></i>
</p>
