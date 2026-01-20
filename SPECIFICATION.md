# Foundations of Functional Optimization

> **A Universal Paradigm for Optimization as Energy Minimization**

**Author**: Mark Amo-Boateng, PhD  
**Affiliation**: Xtellix, Inc.  
**Version**: 0.1.1  
**Date**: January 2025

---

## Introduction

Functional Optimization is a paradigm shift in how we formulate and solve optimization problems. Rather than treating objectives and constraints as fundamentally different mathematical objects requiring separate handling, Functional Optimization unifies everything under a single principle:

**Every optimization problem is an energy landscape. The optimal solution is the state of minimum energy.**

This document establishes the mathematical foundations of this paradigm, including the complete hierarchy from **Terms** through **Ensemble**.

> **Note**: This document uses conceptual terminology. For the normative YAML specification, see [YAML_SPEC.md](YAML_SPEC.md). Terminology mapping:
> - **Static term** → `type: analytic` in YAML
> - **Functional term** → `type: functional` in YAML  
> - **Dense interaction** → `type: all_indices` with `ranges:` in YAML
> - **Sparse interaction** → `type: sparse` in YAML
> - **Expression** → `expr:` in YAML
> - **Aggregator** → `agg:` in YAML

---

## The Core Philosophy

> **Terms** define the basic energy contribution per variable set.
> **Interaction Structures** define which variable sets to include.
> **Aggregators** define how to combine term outputs.
> **Coupling Hamiltonians** define how subsystems interact.
> **Everything else is execution.**

---

## Part I: The Energy Paradigm

### 1.1 The Fundamental Principle

In physics, systems naturally evolve toward states of minimum energy. A ball rolls downhill. Heat flows from hot to cold. Electrons fill the lowest available orbitals. This principle—**energy minimization**—governs the physical universe.

Functional Optimization applies this same principle to mathematical optimization:

$$\boxed{x^* = \arg\min_{x \in \mathcal{X}} H_{\text{total}}(x)}$$

Where:
- $x$ is the vector of decision variables
- $\mathcal{X}$ is the decision space (continuous, discrete, or hybrid)
- $H_{\text{total}}(x)$ is the **total energy function**
- $x^*$ is the optimal solution

### 1.2 Why Energy?

The energy formulation provides four fundamental advantages:

**Unification**: Objectives, constraints, and subsystem interactions all become weighted Hamiltonians.

**Composition**: Complex problems are built by combining Hamiltonians into weighted ensembles.

**Auditability**: Every term, component, and Hamiltonian contributes a measurable quantity to the total energy.

**Physical Correspondence**: The formulation maps directly to physics-based computing paradigms.

---

## Part II: The FEBO Hierarchy

FEBO organizes optimization problems into five levels:

```
┌─────────────────────────────────────────────────────────┐
│  Level 5: ENSEMBLE                                      │
│           H_total = Σ wₖ · Hₖ(x)                        │
│           (Subsystem Hs + Coupling Hs)                  │
└─────────────────────────────────────────────────────────┘
                          ↑
┌─────────────────────────────────────────────────────────┐
│  Level 4: HAMILTONIANS                                  │
│           H(x) = Σ αₘ · Componentₘ(x)                   │
│           • Subsystem H: one subsystem's variables      │
│           • Coupling H: spans multiple subsystems       │
└─────────────────────────────────────────────────────────┘
                          ↑
┌─────────────────────────────────────────────────────────┐
│  Level 3: COMPONENTS                                    │
│           Component = Aggregator(Terms over Interaction)│
└─────────────────────────────────────────────────────────┘
                          ↑
┌─────────────────────────────────────────────────────────┐
│  Level 2: PRIMITIVES (orthogonal)                       │
│           • Interaction Structures: which variable sets │
│           • Aggregators: how to combine term outputs    │
└─────────────────────────────────────────────────────────┘
                          ↑
┌─────────────────────────────────────────────────────────┐
│  Level 1: TERMS                                         │
│           Basic unit functions: term(xᵢ), term(xᵢ,xⱼ)  │
│           Static or Functional                          │
└─────────────────────────────────────────────────────────┘
```

---

## Part III: Terms (Level 1)

A **term** is the atomic building block of FEBO—a basic mathematical function that computes an energy contribution from a specific set of variables.

### 3.1 Term Arity

Terms are classified by how many variables they involve:

| Arity | Name | Form | Examples |
|-------|------|------|----------|
| 0 | Constant | $\text{term}()$ | $c$, $h()$ |
| 1 | Unary | $\text{term}(x_i)$ | $x_i^2$, $\cos(x_i/\sqrt{i})$, $g(x_i)$ |
| 2 | Binary | $\text{term}(x_i, x_j)$ | $x_i x_j$, $(x_i - x_j)^2$, $\phi(x_i, x_j)$ |
| 3 | Ternary | $\text{term}(x_i, x_j, x_k)$ | $x_i x_j x_k$, $T_{ijk} x_i x_j x_k$ |
| n | n-ary | $\text{term}(x_{i_1}, \ldots, x_{i_n})$ | $\prod_{i \in S} x_i$ |

### 3.2 Static vs. Functional Terms

| Type | Description | Example |
|------|-------------|---------|
| **Static** | Fixed mathematical form with coefficients | $Q_{ij} \cdot x_i x_j$ |
| **Functional** | Computed by an arbitrary function | $\text{neural\_net}(x_i, x_j)$ |

**Static terms** are defined by coefficients:
- Unary: $q_i \cdot x_i$ or $q_i \cdot x_i^2$
- Binary: $Q_{ij} \cdot x_i x_j$
- Ternary: $T_{ijk} \cdot x_i x_j x_k$

**Functional terms** are computed:
- Unary: $g(x_i)$ — any function of one variable
- Binary: $\phi(x_i, x_j)$ — neural network, simulation, learned interaction
- n-ary: $f(x_{i_1}, \ldots, x_{i_n})$ — black-box, physics simulation

### 3.3 Common Term Types

| Term | Form | Use Case |
|------|------|----------|
| Linear | $c_i \cdot x_i$ | Linear objectives, costs |
| Quadratic (diagonal) | $q_i \cdot x_i^2$ | Regularization, penalties |
| Quadratic (off-diagonal) | $Q_{ij} \cdot x_i x_j$ | Interactions, correlations |
| Laplacian | $(x_i - x_j)^2$ | Smoothness, similarity |
| Distance | $\|x_i - x_j\|^p$ | Spatial, clustering |
| Trigonometric | $\cos(x_i / \sqrt{i})$ | Oscillatory (Griewank) |
| Indicator | $\mathbf{1}[x_i \neq x_j]$ | Constraint violation |
| Neural | $\text{MLP}(x_i, x_j; \theta)$ | Learned interactions |
| Simulation | $\text{simulate}(x)$ | Physics, black-box |

### 3.4 Term Examples by Problem

| Problem | Term Type | Specific Form |
|---------|-----------|---------------|
| QUBO | Binary, static | $Q_{ij} x_i x_j$ |
| Griewank | Unary, functional | $\cos(x_i / \sqrt{i})$ |
| Graph cut | Binary, static | $w_{ij} (1 - x_i x_j)$ |
| TSP | Binary, static | $d_{ij} x_{ij}$ |
| Portfolio | Binary, static | $\Sigma_{ij} w_i w_j$ |
| Neural loss | Unary, functional | $\ell(f_\theta(x_i), y_i)$ |
| PDE residual | n-ary, functional | $\mathcal{L}[u](x)$ |

---

## Part IV: Interaction Structures (Level 2a)

An **interaction structure** defines which variable sets the terms are applied to—the topology of variable interactions.

### 4.1 The Role of Interaction Structures

Terms define *what* computation happens. Interaction structures define *where* (which variable combinations).

```
Term:          xᵢxⱼ (binary product)
Interaction:   Sparse graph E = {(1,2), (1,3), (2,4), ...}
Result:        Apply term to each edge → {x₁x₂, x₁x₃, x₂x₄, ...}
```

### 4.2 Quadratic Interaction Structures

#### Dense

All pairs $(i,j)$ for $i,j \in \{1, \ldots, n\}$.

$$\{(i,j) : i,j \in [n]\}$$

**Storage**: $O(n^2)$

**Use cases**: Full covariance, small problems, exact models.

---

#### Sparse (Graph / Edge List)

Only specified pairs $(i,j) \in E$.

$$\{(i,j) : (i,j) \in E\}$$

**Storage**: $O(|E|)$

**Use cases**: QUBO, Ising, network optimization, graph problems.

**This is the most common interaction structure in combinatorial optimization.**

---

#### Laplacian / Neighbor

Pairs connected by edges, typically in a grid or graph.

$$\{(i,j) : (i,j) \in \text{neighbors}\}$$

**Use cases**: Grids, PDEs, image processing, spatial smoothness.

---

#### Low-Rank

Implicit dense structure via factorization: $Q = U^\top U$.

**Storage**: $O(nk)$ where $k$ is rank.

**Use cases**: Factor models, embeddings, covariance approximation.

---

#### Diagonal + Low-Rank

$Q = D + U^\top U$

**Use cases**: Per-variable penalties + global structure.

---

#### Block / Tiled

Block-sparse structure in $Q$.

**Use cases**: Modular systems, hierarchical problems.

---

### 4.3 Constraint-Induced Interaction Structures

These specify variable groups for constraint encoding.

#### Groups (One-Hot, At-Most-One, Exactly-K)

$$\{G_1, G_2, \ldots, G_K\}$$ where each $G_k$ is a set of indices.

Apply constraint term to each group. The term decides the constraint type:
- **Exactly-one**: $(\sum_{i \in G_k} x_i - 1)^2$
- **At-most-one**: $\sum_{i < j, i,j \in G_k} x_i x_j$
- **Exactly-K**: $(\sum_{i \in G_k} x_i - K)^2$

**Use cases**: Assignment, scheduling, routing, bin packing candidate selection.

> **YAML**: `type: groups` with `groups:` (data reference or inline) and `bind:` for index symbol.

---

#### Flow Networks

$$\{(i,j) : (i,j) \in \text{arcs}\}$$

Apply flow conservation term at each node.

**Use cases**: Network flow, logistics, routing.

---

#### Resource Sets

$$\{i : i \in \text{resource users}\}$$

Apply capacity constraint term per resource.

**Use cases**: Knapsack, bin packing, scheduling.

---

### 4.4 Higher-Order Interaction Structures

#### k-uniform Hypergraph

$$\{S : S \subseteq [n], |S| = k\}$$

All subsets of size $k$.

**Use cases**: 3-body interactions, polynomial optimization.

---

#### Sparse Hypergraph

$$\{S : S \in \mathcal{E}\}$$

Specified hyperedges only.

**Use cases**: Higher-order Markov random fields, tensor problems.

---

### 4.5 FEBO v1 Required Interaction Structures

| Interaction Structure | Required | YAML `type:` |
|-----------------------|----------|--------------|
| Sparse (edge list) | ✅ | `sparse` |
| All indices (dense) | ✅ | `all_indices` |
| Low-Rank | ✅ | `low_rank` |
| Laplacian / Neighbor | ✅ | `laplacian` |
| Groups | ✅ | `groups` |
| None (scalar) | ✅ | `none` |
| Higher-Order (v2) | Optional | — |

---

## Part V: Aggregators (Level 2b)

An **aggregator** combines the outputs of terms (applied over an interaction structure) into a single scalar energy.

### 5.1 The Role of Aggregators

Terms produce values. Interaction structures specify where. Aggregators combine results.

```
Terms applied:    {t₁, t₂, t₃, t₄, ...}
Aggregator:       Sum
Result:           t₁ + t₂ + t₃ + t₄ + ...
```

### 5.2 Discrete Aggregators

#### Sum (Weighted or Unweighted)

$$A_{\text{sum}} = \sum_{i} w_i \, t_i$$

**Use cases**: General objectives, constraint penalties, composite Hamiltonians.

**This is the default.** When no aggregator is specified, Sum is assumed.

---

#### Mean

$$A_{\text{mean}} = \frac{1}{n}\sum_i t_i$$

**Use cases**: Scale-invariant aggregation, batch averaging, empirical risk.

---

#### Product

$$A_{\text{prod}} = \prod_i t_i$$

**Use cases**: Multiplicative structure, likelihoods, reliability, Griewank.

**Note**: Numerically sensitive. Use LogProd for stability.

---

#### Log-Product (Sum of Logs)

$$A_{\log\text{prod}} = \sum_i \log(\max(t_i, \epsilon))$$

**Use cases**: Numerically stable multiplicative aggregation, log-likelihoods.

---

#### Maximum

$$A_{\max} = \max_i t_i$$

**Use cases**: Worst-case optimization, bottleneck constraints, robust optimization, N-1 security.

**Key insight**: Enables minimax without epigraph reformulation.

---

#### Minimum

$$A_{\min} = \min_i t_i$$

**Use cases**: Best-case selection, resource floors (e.g., minimum battery level).

---

### 5.3 Smooth / Soft Aggregators

#### LogSumExp (Soft Maximum)

$$A_{\text{LSE}} = \frac{1}{\beta}\log\left(\sum_i e^{\beta t_i}\right)$$

**Use cases**: Differentiable approximation to max, robust constraints, entropy regularization.

**Parameter**: $\beta$ controls sharpness. As $\beta \to \infty$, approaches hard max.

---

#### Power Mean

$$A_{\text{pmean}} = \left(\frac{1}{n}\sum_i t_i^r\right)^{1/r}$$

**Use cases**: Continuous interpolation between min ($r \to -\infty$), mean ($r = 1$), and max ($r \to \infty$).

---

### 5.4 Continuous Aggregators

#### Integral

$$A_{\text{int}} = \int_{\Omega} t(s) \, d\mu(s)$$

**Use cases**: Time integrals, spatial integrals, path integrals, continuous domains.

---

#### Expectation

$$A_{\mathbb{E}} = \mathbb{E}_{\xi}[t(\xi)]$$

**Use cases**: Stochastic programming, robust optimization under uncertainty.

---

### 5.5 Indexed Reductions

Within a term expression, **indexed reductions** allow summing or multiplying over a bound index:

$$\text{sum}_p(y_p) = \sum_{p \in \text{domain}} y_p$$
$$\text{prod}_i(t_i) = \prod_{i \in \text{domain}} t_i$$

**Reduction domains by interaction type:**

| Interaction | Domain for $\text{sum}_p(\cdot)$ |
|-------------|----------------------------------|
| All indices | The declared range(s) |
| Groups | Current group's index list |
| Sparse | **Not defined** (parse error) |

**Use cases**: 
- Within groups: `(sum_p(y[p]) - 1)²` for exactly-one constraints
- Within all_indices: `sum_i(cost[i] * x[i])` for linear objectives

> **YAML**: `sum_p(...)`, `prod_i(...)` where the index must match a `bind:` symbol from the interaction.

---

### 5.6 FEBO v1 Required Aggregators

| Aggregator | Symbol | Required |
|------------|--------|----------|
| Sum | $\sum$ | ✅ |
| Prod | $\prod$ | ✅ |
| Max | $\max$ | ✅ |
| Min | $\min$ | ✅ |
| LogSumExp | LSE | ✅ |
| LogProd | $\sum \log$ | ✅ |
| Integral | $\int$ | ✅ |
| Mean | — | Optional (= Sum / n) |
| Power Mean | — | Optional (v2) |
| CVaR | — | Optional (v2) |

---

## Part VI: Components (Level 3)

A **component** is what you get when you apply an aggregator to terms over an interaction structure:

$$\text{Component}(x) = \text{Aggregator}\Big(\text{Term}(x_I) \text{ for } I \in \text{Interaction}\Big)$$

### 6.1 The Formula

$$C(x) = \text{Agg}_{I \in \mathcal{I}}\big[\text{term}(x_I)\big]$$

Where:
- $\mathcal{I}$ is the interaction structure (which variable sets)
- $\text{term}(x_I)$ is the term applied to variable set $I$
- $\text{Agg}$ is the aggregator

### 6.2 The Aggregator × Interaction × Term Matrix

Any aggregator can combine with any interaction structure and any term type:

| Aggregator | Interaction | Term | Result | Name |
|------------|-------------|------|--------|------|
| Sum | Sparse | $x_i x_j$ | $\sum_{(i,j) \in E} x_i x_j$ | QUBO |
| Sum | Dense | $Q_{ij} x_i x_j$ | $x^\top Q x$ | Quadratic form |
| Sum | Laplacian | $(x_i - x_j)^2$ | $\sum_{(i,j)} (x_i - x_j)^2$ | Smoothness |
| Sum | All $i$ | $c_i x_i$ | $c^\top x$ | Linear |
| Prod | All $i$ | $\cos(x_i/\sqrt{i})$ | $\prod_i \cos(x_i/\sqrt{i})$ | Griewank coupling |
| Max | All $i$ | $f_i(x)$ | $\max_i f_i(x)$ | Minimax |
| Max | Sparse | $w_{ij} x_i x_j$ | $\max_{(i,j)} w_{ij} x_i x_j$ | Robust QUBO |
| LSE | All $i$ | $g_i(x)$ | $\frac{1}{\beta}\log\sum_i e^{\beta g_i}$ | Soft max |
| Sum | One-hot groups | $(\sum_p y_p - 1)^2$ | $\sum_{\text{groups}} (\sum_{p \in G} y_p - 1)^2$ | Assignment |

### 6.3 Example Components in Detail

**QUBO Component**:
```
Term:          xᵢxⱼ (binary, static)
Interaction:   Sparse edge list E
Aggregator:    Sum
Component:     Σ_{(i,j)∈E} Qᵢⱼ xᵢxⱼ
```

**Griewank Product Component**:
```
Term:          cos(xᵢ/√i) (unary, functional)
Interaction:   All i ∈ {1,...,n}
Aggregator:    Prod
Component:     ∏ᵢ cos(xᵢ/√i)
```

**Minimax Component**:
```
Term:          fᵢ(x) (functional)
Interaction:   All i ∈ {1,...,m}
Aggregator:    Max
Component:     max_i fᵢ(x)
```

**Smoothness Component**:
```
Term:          (xᵢ - xⱼ)² (binary, static)
Interaction:   Laplacian (neighbor pairs)
Aggregator:    Sum
Component:     Σ_{(i,j)∈neighbors} (xᵢ - xⱼ)²
```

---

## Part VII: Hamiltonians (Level 4)

A **Hamiltonian** is a weighted sum of components representing one coherent "concern":

$$H(x) = \sum_m \alpha_m \cdot C_m(x)$$

### 7.1 Subsystem Hamiltonians

A **subsystem Hamiltonian** operates on one subsystem's variables only:

$$H_1(x_1) = \alpha_1 \cdot C_1(x_1) + \alpha_2 \cdot C_2(x_1)$$

**Examples**:
- $H_{\text{production}}(x_{\text{prod}})$: Production costs and constraints
- $H_{\text{logistics}}(x_{\text{ship}})$: Shipping costs and constraints
- $H_{\text{risk}}(x_{\text{portfolio}})$: Portfolio risk terms

### 7.2 Coupling Hamiltonians

A **coupling Hamiltonian** spans variables from multiple subsystems:

$$H_{12}(x_1, x_2) = \alpha \cdot C(x_1, x_2)$$

**Key insight**: Coupling Hamiltonians are not a special primitive—they are simply Hamiltonians whose interaction structures span multiple variable sets.

**Examples**:
- $H_{\text{interface}}(x_{\text{prod}}, x_{\text{ship}})$: "Production must equal shipments"
- $H_{\text{MHD}}(I_{\text{coil}}, n_{\text{plasma}}, T_{\text{plasma}})$: Physics coupling in fusion
- $H_{\text{echelon}}(x_{\text{tier1}}, x_{\text{tier2}})$: Supply chain tier linkage

### 7.3 Constraint Hamiltonians

Constraints become Hamiltonians with constraint-induced terms and interaction structures:

**Equality** $h(x) = 0$:
$$H_{\text{eq}}(x) = [h(x)]^2$$

**Inequality** $g(x) \leq 0$:
$$H_{\text{ineq}}(x) = [\max(0, g(x))]^2$$

**Property**: Zero energy when satisfied, positive when violated.

### 7.4 Example: Griewank as Hamiltonian

The Griewank function:
$$f(x) = 1 + \frac{1}{4000}\sum_i x_i^2 - \prod_i \cos(x_i/\sqrt{i})$$

As a Hamiltonian:
$$H_{\text{Griewank}}(x) = \alpha_1 \cdot C_{\text{quadratic}} + \alpha_2 \cdot C_{\text{product}} + \alpha_3 \cdot C_{\text{constant}}$$

Where:
- $C_{\text{quadratic}} = \sum_i x_i^2$ (Sum aggregator, unary quadratic term, all-$i$ interaction)
- $C_{\text{product}} = \prod_i \cos(x_i/\sqrt{i})$ (Prod aggregator, cosine term, all-$i$ interaction)
- $C_{\text{constant}} = 1$

With $\alpha_1 = 1/4000$, $\alpha_2 = -1$, $\alpha_3 = 1$.

---

## Part VIII: Ensemble (Level 5)

The **ensemble** is the total energy—a weighted combination of all Hamiltonians:

$$H_{\text{total}}(x) = \sum_k w_k \cdot H_k(x)$$

### 8.1 Structure

The ensemble typically contains:

$$H_{\text{total}} = \underbrace{w_1 H_1(x_1) + w_2 H_2(x_2) + \cdots}_{\text{Subsystem Hamiltonians}} + \underbrace{w_{12} H_{12}(x_1, x_2) + \cdots}_{\text{Coupling Hamiltonians}}$$

### 8.2 Example: Production-Logistics System

```
Variables:
  x₁ = production quantities (by product, facility, time)
  x₂ = shipment quantities (by product, origin, destination, time)

Terms:
  t_prod(x₁) = production cost per unit
  t_ship(x₂) = shipping cost per unit
  t_interface(x₁,x₂) = (production - shipments)²

Components:
  C₁ = Sum over products/facilities/times of t_prod
  C₂ = Sum over shipments of t_ship
  C₃ = Sum over facilities of t_interface

Subsystem Hamiltonians:
  H₁(x₁) = α₁·C₁ (production costs + capacity constraints)
  H₂(x₂) = α₂·C₂ (shipping costs + transport constraints)

Coupling Hamiltonian:
  H₁₂(x₁, x₂) = α₃·C₃ ("Everything produced must be shipped")

Ensemble:
  H_total = w₁·H₁(x₁) + w₂·H₂(x₂) + w₁₂·H₁₂(x₁,x₂)
```

### 8.3 Why Coupling Hamiltonians Matter

**Without coupling H** (sequential optimization):
1. Optimize $H_1(x_1)$ → get $x_1^*$
2. Optimize $H_2(x_2)$ given $x_1^*$ → get $x_2^*$
3. Result: Local optimum, missed interactions

**With coupling H** (simultaneous optimization):
1. Optimize $H_{\text{total}}(x_1, x_2)$ → get $(x_1^*, x_2^*)$
2. Result: Global optimum respecting all interactions

---

## Part IX: The Universal Form

The complete FEBO formulation:

$$\boxed{H_{\text{total}}(x) = \sum_k w_k \cdot \left[ \sum_m \alpha_m^{(k)} \cdot \text{Agg}_m^{(k)}\Big(\text{term}_m^{(k)}(x_I) \text{ for } I \in \mathcal{I}_m^{(k)}\Big) \right]}$$

Where:
- $w_k$: weight for Hamiltonian $k$ (static or functional)
- $\alpha_m^{(k)}$: weight for component $m$ in Hamiltonian $k$
- $\text{Agg}_m^{(k)}$: aggregator for component $m$
- $\text{term}_m^{(k)}$: term function for component $m$
- $\mathcal{I}_m^{(k)}$: interaction structure for component $m$

**Every element can be static or functional.**

---

## Part X: Auditability

### 10.1 The Principle

Every level contributes measurable energy. At any solution $x^*$:

```
H_total(x*) = 157.3
├── w₁·H₁(x₁*): 45.2              [Subsystem 1: Production]
│   ├── α₁·C₁: 32.1                 [Component: Production cost]
│   │   └── Terms: t(F1)=12.1, t(F2)=8.3, t(F3)=11.7
│   └── α₂·C₂: 13.1                 [Component: Capacity penalty]
├── w₂·H₂(x₂*): 67.8              [Subsystem 2: Logistics]
│   ├── α₃·C₃: 52.3                 [Component: Shipping cost]
│   └── α₄·C₄: 15.5                 [Component: Route constraints]
├── w₁₂·H₁₂(x₁*,x₂*): 0.0         [Coupling: satisfied ✓]
│   └── α₅·C₅: 0.0                  [Interface constraint]
└── w₃·H₃(x*): 44.3               [Demand satisfaction]
```

### 10.2 Audit at Every Level

| Level | What You Can Audit |
|-------|-------------------|
| **Term** | Individual variable contributions |
| **Component** | Aggregated energy from one concern |
| **Hamiltonian** | Total energy for one subsystem or coupling |
| **Ensemble** | Overall energy and decomposition |

### 10.3 Benefits

| Use Case | What Auditability Provides |
|----------|---------------------------|
| **Debugging** | Which term/component/H is driving the solution? |
| **Verification** | Are constraints satisfied (H = 0)? |
| **Explanation** | Why was this solution chosen? |
| **Sensitivity** | How do weights affect the outcome? |
| **Compliance** | Auditable trail for regulators |
| **Decomposition** | Energy contribution by subsystem |

### 10.4 Coupling Diagnosis

When subsystems don't coordinate well:

```
H₁₂(x₁*,x₂*) = 23.7  ✗ COUPLING VIOLATED

Component breakdown:
├── C_interface: 23.7
│   ├── term(F3): 15.0  ← production=100, shipments=85, gap²=225
│   ├── term(F7): 8.7   ← production=50, shipments=62, gap²=144
│   └── term(others): 0.0

Diagnosis: Production-logistics mismatch at F3, F7
Action: Increase coupling weight w₁₂ or add capacity
```

---

## Part XI: Comparison to Incumbent Formats

| Feature | MPS | QUBO | Pyomo | AMPL | **FEBO** |
|---------|-----|------|-------|------|----------|
| **Terms: Static** | ✅ | ✅ | ✅ | ✅ | ✅ |
| **Terms: Functional** | ❌ | ❌ | ⚠️ | ⚠️ | ✅ |
| **Terms: Higher-order** | ❌ | ❌ | ⚠️ | ⚠️ | ✅ |
| **Interaction: Sparse** | ⚠️ | ✅ | ✅ | ✅ | ✅ |
| **Interaction: Low-Rank** | ❌ | ❌ | ⚠️ | ⚠️ | ✅ |
| **Aggregator: Sum** | ✅ | ✅ | ✅ | ✅ | ✅ |
| **Aggregator: Max** | ❌ | ❌ | ❌ | ❌ | ✅ |
| **Aggregator: Prod** | ❌ | ❌ | ❌ | ❌ | ✅ |
| **Aggregator: LogSumExp** | ❌ | ❌ | ❌ | ❌ | ✅ |
| **Subsystem Hamiltonians** | ❌ | ❌ | ❌ | ❌ | ✅ |
| **Coupling Hamiltonians** | ❌ | ❌ | ❌ | ❌ | ✅ |
| **Auditability** | ❌ | ❌ | ⚠️ | ⚠️ | ✅ |

**FEBO is the only format with explicit terms, first-class aggregators, and coupling Hamiltonians.**

---

## Part XII: FEBO as Universal Hub

### 12.1 The Converter Architecture

```
     MPS ─────┐
     QUBO ────┼───► FEBO ───┬──► Xtellix
     Pyomo ───┤             ├──► Gurobi
     JuMP ────┤             ├──► D-Wave
     AMPL ────┘             └──► Future solvers
```

**O(n) converters instead of O(n²).**

### 12.2 Why FEBO Works as Hub

| Source Format | FEBO Representation |
|---------------|---------------------|
| MPS (linear) | Sum aggregator, linear terms, all-variable interaction |
| QUBO | Sum aggregator, quadratic terms, sparse interaction |
| Pyomo | Map expressions to terms, constraints to Hamiltonians |
| Higher-order | Native support via n-ary terms |

FEBO is a superset. Lossless import from any source.

---

## Part XIII: Mapping Existing Approaches

| Problem Type | Term | Interaction | Aggregator | Coupling H? |
|--------------|------|-------------|------------|-------------|
| Linear Programming | $c_i x_i$ | All $i$ | Sum | No |
| Quadratic Programming | $Q_{ij} x_i x_j$ | Dense | Sum | No |
| QUBO / Ising | $Q_{ij} x_i x_j$ | Sparse | Sum | No |
| Robust Optimization | $f(x, \xi)$ | Uncertainty set | Max | No |
| Multi-Objective | Multiple | Multiple | Sum | No |
| Multi-System | Per-subsystem | Per-subsystem | Sum | **Yes** |
| Supply Chain | Echelon-specific | Echelon-specific | Sum | **Yes** |
| Fusion Control | Physics-based | Physics-based | Sum + Max | **Yes** |
| Griewank | $x_i^2$, $\cos(x_i/\sqrt{i})$ | All $i$ | Sum + Prod | No |

---

## Summary

### The Five Levels

| Level | Name | What It Is |
|-------|------|------------|
| 1 | **Terms** | Basic unit functions: $\text{term}(x_i)$, $\text{term}(x_i, x_j)$, ... |
| 2a | **Interaction Structures** | Which variable sets to apply terms to |
| 2b | **Aggregators** | How to combine term outputs |
| 3 | **Components** | Aggregator(Terms over Interaction) |
| 4 | **Hamiltonians** | Weighted sum of components (subsystem or coupling) |
| 5 | **Ensemble** | Weighted sum of Hamiltonians |

### The Hierarchy

```
Level 1: Terms
         term(xᵢ), term(xᵢ,xⱼ), term(xᵢ,xⱼ,xₖ), ...
              ↓
Level 2: Primitives (orthogonal)
         ├── Interaction Structures: which variable sets
         └── Aggregators: how to combine
              ↓
Level 3: Components = Agg(Terms over Interaction)
              ↓
Level 4: Hamiltonians = Σ αₘ · Componentₘ
         ├── Subsystem H (single subsystem)
         └── Coupling H (multiple subsystems)
              ↓
Level 5: Ensemble = Σ wₖ · Hₖ
              ↓
         Minimize → Solution
```

### The Philosophy

> **Terms** define the basic energy contribution per variable set.
> **Interaction Structures** define which variable sets to include.
> **Aggregators** define how to combine term outputs.
> **Coupling Hamiltonians** define how subsystems interact.
> **Everything else is execution.**

---

## Notation Reference

| Symbol | Meaning |
|--------|---------|
| $x$ | Decision variables |
| $x_k$ | Variables for subsystem $k$ |
| $\text{term}(x_I)$ | Term applied to variable set $I$ |
| $\mathcal{I}$ | Interaction structure (which variable sets) |
| $\text{Agg}$ | Aggregator (Sum, Prod, Max, Min, LSE, etc.) |
| $C(x)$ | Component = Agg(terms over interaction) |
| $\alpha$ | Component weight |
| $H(x)$ | Hamiltonian (subsystem or coupling) |
| $H_k(x_k)$ | Subsystem Hamiltonian |
| $H_{jk}(x_j, x_k)$ | Coupling Hamiltonian |
| $w$ | Hamiltonian weight in ensemble |
| $H_{\text{total}}$ | Ensemble (total energy) |
| $x^*$ | Optimal solution |

---

## FEBO v1 Required Primitives

### Terms

| Term Type | Required | YAML `type:` |
|-----------|----------|--------------|
| Constant | ✅ | `constant` |
| Unary (analytic) | ✅ | `analytic` |
| Unary (functional) | ✅ | `functional` |
| Binary (analytic) | ✅ | `analytic` |
| Binary (functional) | ✅ | `functional` |
| n-ary | ✅ | `analytic` or `functional` |
| Higher-order | Optional (v2) | — |

### Interaction Structures

| Interaction Structure | Required | YAML `type:` |
|-----------------------|----------|--------------|
| All indices | ✅ | `all_indices` |
| Sparse (edge list) | ✅ | `sparse` |
| Groups | ✅ | `groups` |
| Low-Rank | ✅ | `low_rank` |
| Laplacian / Neighbor | ✅ | `laplacian` |
| None (scalar) | ✅ | `none` |
| Higher-Order Hypergraph | Optional (v2) | — |

### Aggregators

| Aggregator | Required | YAML `agg:` |
|------------|----------|-------------|
| Sum | ✅ | `sum` |
| Prod | ✅ | `prod` |
| Max | ✅ | `max` |
| Min | ✅ | `min` |
| LogSumExp | ✅ | `logsumexp` |
| LogProd | ✅ | `logprod` |
| Integral | ✅ | — |

---

*Functional Optimization: Terms contribute. Interactions select. Aggregators combine. Couplings connect. Minimize the ensemble.*
