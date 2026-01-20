# FEBO YAML Specification v0.1.1

> **Normative specification for FEBO problem files**

**Author**: Mark Amo-Boateng, PhD  
**Affiliation**: Xtellix, Inc.  
**Date**: January 2025

---

## 1. Overview

A FEBO file is a YAML document that specifies an optimization problem as an energy minimization ensemble.

### 1.1 Design Principles

1. **Readable**: Syntax mirrors mathematical notation
2. **Compact**: Simple things are short
3. **Explicit**: Conventions declared, not assumed
4. **Deterministic**: No inference; defaults are documented

### 1.2 File Extension

`.febo.yaml` or `.febo.yml`

---

## 2. Document Structure

```yaml
febo_version: "0.1.1"
name: <identifier>
description: <string>              # optional

conventions:                        # optional, defaults shown
  index_base: 1
  expr_lang: "febo_expr_v1"

parameters: { ... }                 # optional (model parameters)
variables: { ... }                  # required
data: { ... }                       # optional
functions: { ... }                  # optional (external function registry)
interactions: { ... }               # optional
terms: { ... }                      # optional if inline terms used
components: { ... }                 # optional if inline components used
hamiltonians: { ... }               # required
ensemble: [ ... ]                   # required

metadata: { ... }                   # optional
```

---

## 3. Header

### 3.1 febo_version (required)

```yaml
febo_version: "0.1"
```

### 3.2 name (required)

```yaml
name: portfolio_optimization
```

Pattern: `^[a-z][a-z0-9_]*$`

### 3.3 conventions (optional)

```yaml
conventions:
  index_base: 1                    # 1 (default) or 0
  expr_lang: "febo_expr_v1"        # expression language version
```

**Defaults**: `index_base: 1`, `expr_lang: "febo_expr_v1"`

---

## 4. Parameters

Parameters define symbolic values resolved at solve time. They enable reusable models at different scales.

```yaml
parameters:
  n: {type: int, default: 1000, description: "Problem dimension"}
  capacity: {type: float, description: "Plant capacity"}  # required (no default)
```

### 4.1 Structure

```yaml
parameters:
  <name>:
    type: <type>              # required
    default: <value>          # optional (if absent, parameter is required at solve time)
    description: <string>     # optional
    bounds: [lo, hi]          # optional (numeric validation)
```

### 4.2 Types

| Type | Values | Example |
|------|--------|---------|
| `int` | Integer | `1000`, `2000000000` |
| `float` | Real number | `3.14`, `1e-6` |
| `string` | Text | `"adam"`, `"sgd"` |
| `bool` | Boolean | `true`, `false` |

### 4.3 Required vs Optional

| Condition | Behavior |
|-----------|----------|
| `default` present | Parameter is **optional**; uses default if not provided |
| `default` absent | Parameter is **required**; solver errors if not provided |

### 4.4 Usage in Model

Parameters can be used anywhere a numeric or symbolic value is expected:

```yaml
parameters:
  n: {type: int, default: 1000}
  capacity: {type: float, default: 1000}
  risk_aversion: {type: float, default: 0.5}

variables:
  x: {type: continuous, shape: [n], bounds: [0, capacity]}

ensemble:
  - {use: H_return}
  - {use: H_risk, weight: risk_aversion}
```

### 4.5 Solve-Time Override

Parameters can be overridden at solve time:

```bash
# CLI
xtellix solve model.febo.yaml --param n=2000000000 --param capacity=5000

# Instance file
xtellix solve model.febo.yaml --instance params.yaml
```

Instance file format:
```yaml
# params.yaml
n: 2000000000
capacity: 5000
```

### 4.6 Examples

```yaml
# Dimension parameter with default
parameters:
  n: {type: int, default: 1000, description: "Number of variables"}

# Required parameter (no default)
parameters:
  plants: {type: int, description: "Number of plants"}

# With bounds validation
parameters:
  alpha: {type: float, default: 0.5, bounds: [0, 1], description: "Risk aversion"}

# Multiple parameters
parameters:
  plants:    {type: int, description: "Number of plants"}
  customers: {type: int, description: "Number of customers"}
  time:      {type: int, default: 52, description: "Planning horizon (weeks)"}
  capacity:  {type: float, default: 1000, description: "Plant capacity"}
```

---

## 5. Variables

```yaml
variables:
  x:
    type: continuous
    shape: [n]
    bounds: [-600, 600]
```

### 5.1 Types

| Type | Values |
|------|--------|
| `continuous` | ℝ |
| `binary` | {0, 1} |
| `integer` | ℤ |

### 5.2 Shape

Array of dimensions. Integers or symbolic names.

```yaml
shape: [100]           # fixed
shape: [n]             # symbolic
shape: [m, n, t]       # multi-dimensional
shape: [K, "*"]        # ragged (variable-length inner dimension)
```

**Ragged Shape Semantics**:

Use `"*"` to indicate variable-length dimensions (e.g., for `groups` data where each group has different size).

- `shape: [K, "*"]` means an outer array of `K` groups
- Each group is an ordered list of unique integer indices
- Validation: `len(groups) == K`
- Each element must be a valid index in the declared domain
- Duplicates within a group are invalid

**Index Bounds by `index_base`:**

| `index_base` | Valid indices for dimension `P` |
|--------------|--------------------------------|
| `1` (default) | `1..P` (inclusive) |
| `0` | `0..P-1` (inclusive) |

### 5.3 Bounds

```yaml
bounds: [0, 1]                        # numeric bounds
bounds: [-100, "inf"]                 # unbounded above
bounds: ["-inf", 100]                 # unbounded below
bounds: ["-inf", "inf"]               # fully unbounded
bounds: null                          # fully unbounded (equivalent)
bounds: {lower: 0, upper: capacity}   # reference data
```

**Special tokens**: `"inf"` and `"-inf"` (as strings) for unbounded.

---

## 6. Data (optional)

External data references.

```yaml
data:
  returns:
    source: "returns.csv"
    shape: [n]
    
  covariance:
    source: "cov.h5"
    shape: [n, n]
    format: low_rank
    rank: 50
    
  overlap_pairs:
    source: "overlap.parquet"
    shape: [E, 2]
    dtype: int32
    
  Q:
    inline: [[1, 0.5], [0.5, 2]]
```

### 6.1 Formats

| Format | Description |
|--------|-------------|
| `csv` | Comma-separated (default) |
| `json` | JSON array |
| `npy` | NumPy binary |
| `h5` | HDF5 |
| `parquet` | Apache Parquet |
| `low_rank` | Factor form Q = UᵀU |
| `sparse` | COO format |

### 6.2 Data Types

| dtype | Description |
|-------|-------------|
| `float64` | 64-bit float (default) |
| `float32` | 32-bit float |
| `int64` | 64-bit integer |
| `int32` | 32-bit integer |
| `bool` | Boolean |

### 6.3 Storage Hints (optional)

```yaml
data:
  large_matrix:
    source: "matrix.h5"
    shape: [n, n]
    storage: row_major    # or column_major
```

Storage hints help solvers optimize memory layout.

---

## 7. Functions (optional)

External function registry for `functional` terms.

```yaml
functions:
  stability_score:
    kind: python
    entrypoint: "pack_models:stability_score"
    
  contact_penalty:
    kind: wasm
    uri: "models/contact.wasm"
    
  dock_score:
    kind: shared_lib
    uri: "libdock.so"
    symbol: "compute_dock"
```

### 7.1 Function Kinds

| Kind | Description | Fields |
|------|-------------|--------|
| `python` | Python callable | `entrypoint` (module:function) |
| `wasm` | WebAssembly module | `uri` |
| `shared_lib` | Native shared library | `uri`, `symbol` |
| `http` | Remote API | `uri`, `method` |

### 7.2 Usage in Terms

```yaml
terms:
  dock: {type: functional, arity: binary, func: stability_score, args: "item[i], pose[p]"}
```

---

## 8. Interactions

Interaction structures define which index combinations terms apply to.

```yaml
interactions:
  all_i:     {type: all_indices, range: [1, n]}
  all_ij:    {type: all_indices, ranges: {i: [1, n], j: [1, n]}}
  edges:     {type: sparse, source: "graph.csv"}
  my_pairs:  {type: sparse, pairs: [[1,2], [2,3], [3,4]]}
  neighbors: {type: laplacian, dims: [100, 100]}
  by_item:   {type: groups, groups: candidates_by_item}
```

### 8.1 Interaction Types

| Type | Description | Fields |
|------|-------------|--------|
| `none` | No index expansion (scalar) | (none) |
| `all_indices` | All indices in range(s) | `range` or `ranges` |
| `sparse` | Edge/pair list | `source` or `pairs` |
| `groups` | Named groups of indices | `groups` (data ref or inline) |
| `laplacian` | Grid neighbors | `dims`, `connectivity` |
| `low_rank` | Implicit via factors | `factors`, `rank` |
| `one_hot` | Constraint groups (alias for `groups`) | `groups` |
| `index_set` | Named index set | `indices` |

### 8.2 none

Scalar interaction (no indices):

```yaml
offset: {term: one, interaction: none}
# equivalent to omitting interaction entirely
```

### 8.3 all_indices

Two forms:

```yaml
# Single index: binds default symbol `i`
all_i: {type: all_indices, range: [1, n]}

# Explicit bind
all_p: {type: all_indices, range: [1, P], bind: p}

# Multiple indices: binds named symbols (cartesian product)
all_ij: {type: all_indices, ranges: {i: [1, m], j: [1, n]}}
```

### 8.4 sparse

Three forms:

```yaml
# From data reference (matches key in data:)
overlaps: {type: sparse, source: overlap_pairs, bind: [p, q]}

# From file (does not match any data: key)
edges: {type: sparse, source: "graph.csv", bind: [i, j]}

# Inline pairs
edges: {type: sparse, pairs: [[1,2], [2,3], [1,3]], bind: [i, j]}
```

**Source Resolution:**

1. If `pairs` is present → use inline pairs
2. Else if `source` matches a key in `data:` → treat as data reference
3. Else → treat as file path/URI

### 8.5 groups

Groups of indices for constraint structures (e.g., exactly-one, at-most-one):

```yaml
# From data reference (matches key in data:)
by_item: {type: groups, groups: candidates_by_item, bind: p}

# Inline
by_bin: {type: groups, groups: [[1,2,3], [4,5,6]], bind: p}
```

**Semantics**: 

- Each group is an **ordered list of unique indices** (duplicates invalid)
- A `groups` interaction iterates over groups
- The term is evaluated once per group, using the group's list as the reduction domain
- The component aggregator then reduces across groups

**Two-level evaluation:**

```
Component with groups + agg: sum
  └── For each group g:
        └── Evaluate term (may contain sum_p over group's indices)
  └── Sum results across all groups
```

**Groups Resolution:**

1. If `groups` is an array of arrays → use inline groups
2. Else if `groups` matches a key in `data:` → treat as data reference

### 8.6 Index Binding Rules

| Interaction | Default bind | Explicit bind |
|-------------|--------------|---------------|
| `all_indices` (single) | `i` | `bind: <symbol>` |
| `all_indices` (ranges) | keys from `ranges` | — |
| `sparse` | `i`, `j` | `bind: [<s1>, <s2>]` |
| `groups` | `i` | `bind: <symbol>` |

**Rule**: If `bind` is omitted, default symbols are used. If `bind` is provided, it overrides the default.

**Normative**: For `groups` and `sparse`, `bind` SHOULD be explicitly specified to avoid symbol clashes across terms. Best practice is always explicit `bind` in production models.

**Validation**:

- `groups` data must be list-of-lists of integers
- Each integer must be within variable index bounds
- Duplicates within a group are invalid

### 8.7 one_hot

Alias for `groups`. Provided for clarity when enforcing exactly-one constraints:

```yaml
item_assignment: {type: one_hot, groups: [[1,2,3], [4,5,6]]}
```

---

## 9. Terms

Terms are atomic energy functions.

```yaml
terms:
  square: {type: analytic, arity: unary, expr: "x[i]^2"}
  cosine: {type: analytic, arity: unary, expr: "cos(x[i] / sqrt(i))"}
  covar:  {type: analytic, arity: binary, expr: "Sigma[i,j] * w[i] * w[j]"}
  one:    {type: constant, value: 1}
  dock:   {type: functional, arity: binary, expr: "dock(ligand[i], protein[j])"}
```

### 9.1 Term Types

| Type | Description |
|------|-------------|
| `analytic` | Closed-form expression |
| `functional` | Computed by external function |
| `constant` | Fixed scalar value |

### 9.2 Arity

| Arity | Index count | Example |
|-------|-------------|---------|
| `constant` | 0 | `1` |
| `unary` | 1 | `x[i]^2` |
| `binary` | 2 | `Q[i,j] * x[i] * x[j]` |
| `ternary` | 3 | `T[i,j,k] * x[i] * x[j] * x[k]` |
| `n-ary` | n | `(sum_p(y[p]) - 1)^2` |

**n-ary terms**: For `arity: n-ary`, the term expression MAY contain indexed reductions and references to multiple indices. Validity is determined by bound-symbol checks at parse time. The "inputs" are whatever the expression references from bound symbols and data.

### 9.3 Expression Syntax (febo_expr_v1)

```
# Variables and data
x[i]                    x[i,j]                  Q[i,j]

# Arithmetic
+ - * / ^               

# Functions
sqrt(x)   exp(x)   log(x)   abs(x)
sin(x)    cos(x)   tan(x)
max(a,b)  min(a,b)
sum(...)  prod(...)
sum_i(...)  sum_j(...)            # indexed reductions

# Numbers
42        3.14159       1e-6      1E+10
1_000_000               # underscores allowed (readability)

# Constraint helpers
max(0, expr)            # ReLU for inequality
(expr)^2                # penalty
```

**Note**: Expressions like `1/4000` are valid because `/` is an operator. The expression is evaluated at parse time.

### 9.4 Indexed Reductions

Indexed reductions sum or multiply over a bound index:

```
sum_p(expr)     # sum over all values of p
sum_i(expr)     # sum over all values of i
prod_j(expr)    # product over all values of j
```

**Rules:**

1. The index symbol must be **bound** by the current interaction's `bind` field (or default)
2. If the index is not bound, it's a **parse error**
3. The reduction iterates over the index's **domain** (see below)

**Reduction Domains by Interaction Type:**

| Interaction | Domain for `sum_<idx>(...)` |
|-------------|----------------------------|
| `all_indices` | The declared `range` or `ranges` |
| `groups` | The current group's index list |
| `sparse` | **Not defined** — parse error |

**Note**: For `sparse`, the bound symbols (`p`, `q`) take values from each edge pair. Reductions over these symbols are not meaningful (there's no "set of all p" — just pairs). Use `sparse` for pairwise terms only, not reductions.

**Examples:**

```yaml
# all_indices: reduction over declared range
all_p: {type: all_indices, range: [1, P], bind: p}
expr: "sum_p(cost[p] * y[p])"   # sums over p = 1..P
```

```yaml
# groups: reduction over current group's indices
by_item: {type: groups, groups: candidates_by_item, bind: p}
expr: "(sum_p(y[p]) - 1)^2"     # sums over p in current group
```

```yaml
# sparse: pairwise only, no reductions
overlaps: {type: sparse, source: overlap_pairs, bind: [p, q]}
expr: "y[p] * y[q]"             # valid: uses p,q from each pair
expr: "sum_p(y[p])"             # INVALID: p has no domain in sparse
```

---

## 10. Components

Components combine term + interaction + aggregator.

```yaml
components:
  quadratic: {term: square, interaction: all_i, agg: sum}
  product:   {term: cosine, interaction: all_i, agg: prod}
  variance:  {term: covar, interaction: all_ij, agg: sum}
  worst:     {term: violation, interaction: scenarios, agg: max}
  offset:    {term: one}
```

### 10.1 Aggregators

| Agg | Formula | Use |
|-----|---------|-----|
| `sum` | Σᵢ tᵢ | Default for indexed |
| `prod` | Πᵢ tᵢ | Multiplicative |
| `max` | maxᵢ tᵢ | Worst-case |
| `min` | minᵢ tᵢ | Best-case |
| `mean` | (1/n) Σᵢ tᵢ | Average |
| `logsumexp` | (1/β) log Σᵢ exp(β tᵢ) | Smooth max |
| `logprod` | Σᵢ log(tᵢ) | Log-likelihood |
| `identity` | t | Scalar passthrough |

### 10.2 Aggregator Defaults

| Condition | Default |
|-----------|---------|
| `interaction` omitted | `interaction: none` |
| `interaction: none` and `agg` omitted | `agg: identity` |
| `interaction` present and `agg` omitted | **Error**: `agg` required |

### 10.3 Aggregator Parameters

```yaml
components:
  soft_max:  {term: t, interaction: I, agg: {type: logsumexp, beta: 10}}
  tail_risk: {term: t, interaction: I, agg: {type: cvar, alpha: 0.95}}
```

---

## 11. Hamiltonians

Hamiltonians are weighted sums of components.

```yaml
hamiltonians:
  H_cost:
    components:
      - {use: production_cost}
      - {use: shipping_cost}
      
  H_coupling:
    couples: [production, shipping]
    components:
      - {use: interface_balance}
```

### 11.1 Hamiltonian Type Rules

**Deterministic** (no inference):

| Condition | Type |
|-----------|------|
| `couples` absent | `subsystem` |
| `couples` present | `coupling` |

The `type` field is **optional**; if provided, it must be consistent with `couples`.

### 11.2 Component Reference

```yaml
components:
  - {use: <component_id>}              # alpha defaults to 1
  - {use: <component_id>, alpha: 0.5}  # explicit alpha
  - {use: <component_id>, alpha: 1/4000}  # expression
```

`alpha` is optional (default: `1`). Can be numeric or expression.

### 11.3 Inline Components

Components can be defined inline within a Hamiltonian:

```yaml
hamiltonians:
  H_budget:
    components:
      - term: {type: analytic, arity: n-ary, expr: "(sum(w) - 1)^2"}
        alpha: 1
```

---

## 12. Ensemble

The total energy function.

```yaml
ensemble:
  - {use: H_cost}                      # weight defaults to 1
  - {use: H_risk, weight: 0.5}
  - {use: H_constraints, weight: 10000}
```

`weight` is optional (default: `1`). Can be numeric or expression.

### 12.1 Functional Weights

```yaml
ensemble:
  - {use: H_explore, weight: "temperature * exp(-iter/1000)"}
```

---

## 13. Metadata (optional)

```yaml
metadata:
  problem_type: continuous_nonconvex
  global_minimum: 0
  optimal_solution: "x = 0"
  
  difficulty:
    - "Multi-modal"
    - "Non-separable"
    
  benchmarks:
    - {n: 2e9, solver: xtellix, time: 43}
    - {n: 1e5, solver: knitro, time: 795600}
```

### 13.1 Realtime Hints (optional)

For streaming/online optimization:

```yaml
metadata:
  realtime:
    warm_start: true
    streaming_updates:
      - data: candidate_poses
      - data: overlap_pairs
      - data: cost
    update_frequency: "per_item"    # or "batch", "periodic"
    latency_target_ms: 50
```

| Field | Description |
|-------|-------------|
| `warm_start` | Solver should warm-start from previous solution |
| `streaming_updates` | Data fields that update between solves |
| `update_frequency` | How often updates arrive |
| `latency_target_ms` | Target solve time hint |

These are hints for solvers; they don't affect the mathematical model.

---

## 14. Complete Example: Griewank

```yaml
febo_version: "0.1"
name: griewank
conventions:
  index_base: 1
  expr_lang: "febo_expr_v1"

parameters:
  n: {type: int, default: 1000, description: "Problem dimension"}

variables:
  x:
    type: continuous
    shape: [n]
    bounds: [-600, 600]

interactions:
  all_i: {type: all_indices, range: [1, n]}

terms:
  square: {type: analytic, arity: unary, expr: "x[i]^2"}
  cosine: {type: analytic, arity: unary, expr: "cos(x[i] / sqrt(i))"}
  one:    {type: constant, value: 1}

components:
  quadratic: {term: square, interaction: all_i, agg: sum}
  oscill:    {term: cosine, interaction: all_i, agg: prod}
  offset:    {term: one}

hamiltonians:
  bowl:
    components:
      - {use: quadratic, alpha: 1/4000}
  oscillation:
    components:
      - {use: oscill, alpha: -1}
  constant:
    components:
      - {use: offset}

ensemble:
  - {use: bowl}
  - {use: oscillation}
  - {use: constant}
```

---

## 15. Complete Example: Portfolio

```yaml
febo_version: "0.1"
name: portfolio
conventions:
  index_base: 1

parameters:
  n: {type: int, default: 100, description: "Number of assets"}
  risk_aversion: {type: float, default: 0.5, bounds: [0, 10], description: "Risk aversion coefficient"}

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

---

## 16. Complete Example: Supply Chain with Coupling

```yaml
febo_version: "0.1"
name: supply_chain
conventions:
  index_base: 1

parameters:
  plants:    {type: int, description: "Number of plants"}
  customers: {type: int, description: "Number of customers"}
  time:      {type: int, default: 52, description: "Planning periods (weeks)"}
  capacity:  {type: float, default: 1000, description: "Plant capacity"}

variables:
  production: {type: continuous, shape: [plants, time], bounds: [0, capacity]}
  shipment:   {type: continuous, shape: [plants, customers, time], bounds: [0, "inf"]}

data:
  prod_cost: {source: "prod_cost.csv", shape: [plants]}
  ship_cost: {source: "ship_cost.csv", shape: [plants, customers]}
  demand:    {source: "demand.csv", shape: [customers, time]}

interactions:
  prod_pt:  {type: all_indices, ranges: {p: [1, plants], t: [1, time]}}
  ship_pct: {type: all_indices, ranges: {p: [1, plants], c: [1, customers], t: [1, time]}}
  cust_ct:  {type: all_indices, ranges: {c: [1, customers], t: [1, time]}}

terms:
  t_prod:      {type: analytic, arity: unary, expr: "prod_cost[p] * production[p,t]"}
  t_ship:      {type: analytic, arity: unary, expr: "ship_cost[p,c] * shipment[p,c,t]"}
  t_stockout:  {type: analytic, arity: n-ary, expr: "max(0, demand[c,t] - sum_p(shipment[p,c,t]))^2"}
  t_interface: {type: analytic, arity: n-ary, expr: "(sum_c(shipment[p,c,t]) - production[p,t])^2"}

components:
  c_prod:      {term: t_prod, interaction: prod_pt, agg: sum}
  c_ship:      {term: t_ship, interaction: ship_pct, agg: sum}
  c_stockout:  {term: t_stockout, interaction: cust_ct, agg: sum}
  c_interface: {term: t_interface, interaction: prod_pt, agg: sum}

hamiltonians:
  H_production:
    components:
      - {use: c_prod}
      
  H_distribution:
    components:
      - {use: c_ship}
      - {use: c_stockout, alpha: 100}
      
  H_coupling:
    couples: [production, shipment]
    components:
      - {use: c_interface}

ensemble:
  - {use: H_production}
  - {use: H_distribution}
  - {use: H_coupling, weight: 10000}
```

---

## 17. Validation Rules

### 17.1 Required Fields

- `febo_version`
- `name`
- `variables` (at least one)
- `hamiltonians` (at least one)
- `ensemble` (at least one entry)

### 17.2 Optional Fields

- `terms` — optional if all terms are inline
- `components` — optional if all components are inline
- `interactions` — optional if all interactions are inline or `none`

### 17.3 Reference Resolution

All references must resolve:
- `term:` → defined term or inline
- `interaction:` → defined interaction or inline
- `use:` → defined component or hamiltonian

### 17.4 Semantic Rules

- If `couples` present → hamiltonian is `coupling`
- If `couples` absent → hamiltonian is `subsystem`
- If `interaction` absent → `interaction: none`
- If `interaction: none` and `agg` absent → `agg: identity`
- If `interaction` present and `agg` absent → **Error**

---

## 18. Reserved Keywords

```
febo_version, name, description, conventions, index_base, expr_lang,
parameters, type, default, bounds,
variables, data, functions, interactions, terms, components, hamiltonians, ensemble,
metadata, shape, source, format, dtype, storage,
expr, value, arity, func, args, kind, entrypoint, uri, symbol, bind,
agg, alpha, weight, use, range, ranges, couples, inline, pairs, groups,
continuous, binary, integer, analytic, functional, constant,
unary, binary, ternary, n-ary, none, identity,
all_indices, sparse, laplacian, low_rank, one_hot, index_set,
sum, prod, max, min, mean, logsumexp, logprod, cvar,
subsystem, coupling, int, float, string, bool,
realtime, warm_start, streaming_updates, python, wasm, shared_lib, http
```

---

## Appendix A: Expression Grammar (febo_expr_v1)

```ebnf
expr       = term (('+' | '-') term)*
term       = factor (('*' | '/') factor)*
factor     = base ('^' power)?
power      = unary
unary      = '-'? base
base       = number | ref | call | '(' expr ')'
ref        = ident ('[' idx (',' idx)* ']')?
call       = ident '(' (expr (',' expr)*)? ')'
idx        = ident | number
ident      = [a-zA-Z_][a-zA-Z0-9_]*
number     = int frac? exp?
int        = [0-9]+ ('_' [0-9]+)*
frac       = '.' [0-9]+
exp        = [eE] [+-]? [0-9]+
```

**Notes**:
- Underscores in numbers for readability: `1_000_000`
- Scientific notation: `1e-6`, `3.14E+10`
- Fractions like `1/4000` are valid expressions (division operator)

---

## Appendix B: Defaults Summary

| Field | Default | Notes |
|-------|---------|-------|
| `conventions.index_base` | `1` | |
| `conventions.expr_lang` | `"febo_expr_v1"` | |
| `parameters.<p>.default` | None | Parameter required if not set |
| `data.<d>.dtype` | `float64` | |
| `interaction` (in component) | `none` | |
| `agg` (when `interaction: none`) | `identity` | |
| `alpha` (in hamiltonian component) | `1` | `{use: foo}` implies `alpha: 1` |
| `weight` (in ensemble) | `1` | `{use: H_foo}` implies `weight: 1` |
| `type` (in hamiltonian) | `subsystem` | `coupling` if `couples` present |

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 0.1.1 | 2025-01-20 | Added: `functions` registry, `groups` interaction, `bind` for index symbols, `dtype`/`storage`, ragged shapes, indexed reductions with domain rules, realtime metadata. Resolution: `source`/`groups` check `data:` keys first. |
| 0.1 | 2025-01-20 | Initial specification |
