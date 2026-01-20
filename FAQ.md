# Frequently Asked Questions

## General

### What does FEBO stand for?

**F**unctional **E**nergy-**B**ased **O**ptimization.

### Why another optimization format?

Existing formats (MPS, QUBO, LP) are solver-specific and limited:

- **MPS**: Linear/quadratic only, Sum aggregator only
- **QUBO**: Binary quadratic only, no constraints
- **Pyomo/AMPL**: Modeling languages, not portable specifications

FEBO is a **universal** format that can represent any optimization problem and be consumed by any solver.

### Is FEBO a solver?

No. FEBO is a **specification language**—like HTML for web pages or SQL for databases. You need a solver to actually find solutions.

### What solvers support FEBO?

Currently:
- [Xtellix](https://xtellix.com) — Native FEBO support

Coming soon:
- Converters for Gurobi, CPLEX, D-Wave, and others

---

## Technical

### Why "Hamiltonian"?

The term comes from physics, where Hamiltonians represent total energy. In FEBO, each Hamiltonian represents one "concern" (objective, constraint, or coupling). The name emphasizes that:

1. We're minimizing energy
2. The formulation maps to physics-based computing
3. It's a well-defined mathematical object

### What's the difference between a Component and a Hamiltonian?

- **Component**: A single aggregated term (e.g., `Sum over i of x_i²`)
- **Hamiltonian**: A weighted sum of components representing one concern

Example:
```
Component C₁ = Σᵢ cost[i] * x[i]     (shipping cost)
Component C₂ = Σᵢ penalty[i] * y[i]  (delay penalty)

Hamiltonian H_logistics = 0.8 * C₁ + 1.2 * C₂
```

### What's a Coupling Hamiltonian?

A Hamiltonian whose interaction structure spans multiple subsystems. It defines how subsystems interact.

Example: Production facility must equal shipments
```
H_coupling(x_prod, x_ship) = Σᵢ (production[i] - shipments[i])²
```

### Why are non-Sum aggregators important?

**Sum** (the only aggregator in most formats) can't naturally express:

- **Max**: Worst-case / robust optimization
- **Min**: Best-case / resource floors  
- **Prod**: Multiplicative structures (likelihoods, Griewank)
- **LogSumExp**: Smooth approximations

With FEBO, you write what you mean directly.

### What does "functional term" mean?

A term computed by an arbitrary function rather than a fixed coefficient:

- **Static term**: `Q[i,j] * x[i] * x[j]` (coefficient lookup)
- **Functional term**: `neural_network(x[i], x[j])` (computed)

Functional terms enable ML integration, simulation coupling, and black-box optimization.

---

## Practical

### How do I convert my MPS file to FEBO?

Coming soon: `mps2febo` converter.

Conceptually:
- Objective → Sum aggregator, linear/quadratic terms
- Constraints → Constraint Hamiltonians with penalty weights

### How do I convert my QUBO to FEBO?

Coming soon: `qubo2febo` converter.

Conceptually:
- Q matrix → Sparse quadratic interaction
- All terms → Sum aggregator
- Single Hamiltonian in ensemble

### What file format does FEBO use?

FEBO specifications can be written in:
- **YAML** (recommended for readability)
- **JSON** (for programmatic generation)

### Is there a schema for validation?

Coming in v0.2.

---

## Philosophy

### Why energy minimization?

1. **Unification**: Objectives and constraints become the same thing
2. **Physical intuition**: Systems naturally minimize energy
3. **Hardware mapping**: Direct correspondence to physics-based computers
4. **Auditability**: Energy contributions are measurable

### What does "Everything else is execution" mean?

FEBO separates **specification** from **solving**:

- **Specification** (FEBO): What problem you want solved
- **Execution** (Solver): How to find the solution

This is like SQL separating query specification from query execution. You describe WHAT, the solver figures out HOW.

---

## More Questions?

Open an issue on [GitHub](https://github.com/xtellix/febo-spec/issues).
