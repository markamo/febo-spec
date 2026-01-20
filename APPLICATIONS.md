# Functional Optimization Part 2: Real-World Formulations

> **From Mathematical Building Blocks to Practical Specifications**

**Author**: Mark Amo-Boateng, PhD  
**Affiliation**: Xtellix, Inc.  
**Version**: 0.1.1  
**Date**: January 2025

---

## Introduction

Part 1 established the FEBO hierarchy:

```
Level 1: Terms              → Basic unit functions
Level 2: Primitives         → Interaction Structures + Aggregators (orthogonal)
Level 3: Components         → Aggregator(Terms over Interaction)
Level 4: Hamiltonians       → Subsystem H (single system) or Coupling H (multi-system)
Level 5: Ensemble           → Weighted Hamiltonians → Minimize
```

This document demonstrates how real-world optimization problems—many considered intractable at scale—naturally decompose into this structure. For each problem, we show:

1. **Mathematical Building Blocks**: Terms, Interaction Structures, Aggregators, Components, Hamiltonians
2. **FEBO Specification**: The complete, auditable problem specification
3. **Auditability**: What you can inspect at any solution
4. **Scale Challenge**: Why incumbents fail and what FEBO enables

---

## Problem 0: Griewank Benchmark (Reference Example)

The Griewank function is a standard optimization benchmark that perfectly illustrates FEBO's structure, including multiple aggregator types.

### The Problem

$$f(x) = 1 + \frac{1}{4000}\sum_{i=1}^{n} x_i^2 - \prod_{i=1}^{n} \cos\left(\frac{x_i}{\sqrt{i}}\right)$$

**Characteristics:**
- Separable quadratic component
- Global multiplicative coupling
- Multi-modal (many local minima)
- Global minimum: $f(0) = 0$ at $x^* = \mathbf{0}$

### Mathematical Building Blocks

**Level 1: Terms**

| Term ID | Type | Arity | Expression | Description |
|---------|------|-------|------------|-------------|
| $t_{\text{square}}$ | Static | Unary | $x_i^2$ | Quadratic contribution |
| $t_{\text{cosine}}$ | Functional | Unary | $\cos(x_i / \sqrt{i})$ | Oscillatory term |
| $t_{\text{const}}$ | Static | Constant | $1$ | Offset |

**Level 2a: Interaction Structures**

| ID | Type | Description |
|----|------|-------------|
| $\mathcal{I}_{\text{all}}$ | All indices | $\{i : i \in 1..n\}$ |

**Level 2b: Aggregators**

| ID | Type | Formula |
|----|------|---------|
| Sum | Additive | $\sum_i$ |
| Prod | Multiplicative | $\prod_i$ |

**Level 3: Components**

| Component | Term | Interaction | Aggregator | Result |
|-----------|------|-------------|------------|--------|
| $C_{\text{quadratic}}$ | $t_{\text{square}}$ | $\mathcal{I}_{\text{all}}$ | Sum | $\sum_{i=1}^{n} x_i^2$ |
| $C_{\text{product}}$ | $t_{\text{cosine}}$ | $\mathcal{I}_{\text{all}}$ | **Prod** | $\prod_{i=1}^{n} \cos(x_i/\sqrt{i})$ |
| $C_{\text{constant}}$ | $t_{\text{const}}$ | — | — | $1$ |

**Level 4: Hamiltonians**

| Hamiltonian | Type | Expression | Weight $\alpha$ |
|-------------|------|------------|-----------------|
| $H_{\text{bowl}}$ | Subsystem | $\text{Sum}_i(x_i^2)$ | $1/4000$ |
| $H_{\text{oscillation}}$ | Subsystem | $\text{Prod}_i(\cos(x_i/\sqrt{i}))$ | $-1$ |
| $H_{\text{offset}}$ | Constant | $1$ | $1$ |

*Note: No Coupling Hamiltonian needed—this is a single-system problem where all variables interact within one domain.*

**Level 5: Ensemble**

$$H_{\text{total}}(x) = \frac{1}{4000}\sum_{i=1}^{n} x_i^2 - \prod_{i=1}^{n} \cos\frac{x_i}{\sqrt{i}} + 1$$

### FEBO Specification

```yaml
febo_version: "0.1.1"
name: griewank_benchmark

# ============================================================
# VARIABLES
# ============================================================
variables:
  x:
    type: continuous
    shape: [n]
    bounds: [-600, 600]

# ============================================================
# LEVEL 1: TERMS
# ============================================================
terms:
  t_square:
    type: analytic
    arity: unary
    expr: "x[i]^2"
    
  t_cosine:
    type: functional
    arity: unary
    expr: "cos(x[i] / sqrt(i))"
    
  t_const:
    type: constant
    value: 1

# ============================================================
# LEVEL 2: INTERACTION STRUCTURES
# ============================================================
interactions:
  all_i:
    type: all_indices
    range: [1, n]
    bind: i

# ============================================================
# LEVEL 3: COMPONENTS
# ============================================================
components:
  c_quadratic:
    term: t_square
    interaction: all_i
    agg: sum
    
  c_product:
    term: t_cosine
    interaction: all_i
    agg: prod        # KEY: non-Sum aggregator
    
  c_constant:
    term: t_const

# ============================================================
# LEVEL 4: HAMILTONIANS
# ============================================================
hamiltonians:
  H_bowl:
    description: "Quadratic bowl pulling toward origin"
    components:
      - {use: c_quadratic, alpha: 0.00025}
    
  H_oscillation:
    description: "Oscillatory landscape creating local minima"
    components:
      - {use: c_product, alpha: -1.0}
    
  H_offset:
    description: "Constant offset"
    components:
      - {use: c_constant, alpha: 1.0}

# ============================================================
# LEVEL 5: ENSEMBLE
# ============================================================
ensemble:
  - {use: H_bowl, weight: 1.0}
  - {use: H_oscillation, weight: 1.0}
  - {use: H_offset, weight: 1.0}
```

### Auditability at Solution

**At global optimum $x^* = \mathbf{0}$:**

```
H_total(x*) = 0.000
├── H_bowl:        (1/4000) × Sum(0²) = 0.000
├── H_oscillation: (-1) × Prod(cos(0)) = -1.000
└── H_offset:      1.000
                   ─────────────────────────────
Total:             0.000 + (-1.000) + 1.000 = 0.000 ✓
```

**At suboptimal point $x = (1, 1, ..., 1)$:**

```
H_total(x) > 0  ✗ SUBOPTIMAL

Breakdown:
├── H_bowl:        (1/4000) × n = 0.00025n   ← Variables not at origin
├── H_oscillation: (-1) × 0.37 ≈ -0.37       ← Coupling degraded
└── H_offset:      1.000

Diagnosis: H_bowl > 0 indicates variables displaced from origin
Action: Move x toward zero
```

### Scale Comparison

| Solver | Variables | Time | $H_{\text{total}}$ | Auditable |
|--------|-----------|------|-------------------|-----------|
| Knitro | 100,000 | 9.2 days | 298 (local minimum) | ❌ |
| **Xtellix** | **2,000,000,000** | **43 sec** | **≈ 0** (global) | **✅** |

**Combined advantage: 370,000,000×** (20,000× more variables, 18,500× faster)

---

## Problem 1: Fusion Plasma Control

### The Problem

Control a tokamak fusion reactor plasma in real-time by optimizing:
- Magnetic coil currents (confinement)
- Heating power distribution (temperature)  
- Plasma shape and position (stability)
- Density profile (fusion rate)

**Why It's Hard:**
- Multi-physics coupling (MHD + transport + heating)
- Real-time constraint (<1ms control loop)
- Safety-critical (avoid disruptions)
- Worst-case stability (any unstable mode = failure)

### Mathematical Building Blocks

**Level 1: Terms**

| Term | Type | Expression | Physics |
|------|------|------------|---------|
| $t_{\text{growth}}$ | Functional | $\gamma_k(I, n, T)$ | MHD growth rate for mode $k$ |
| $t_{\text{confinement}}$ | Functional | $-\tau_E(n, T, P)$ | Negative confinement time |
| $t_{\text{power}}$ | Static | $(P_{\text{in}} - P_{\text{loss}})^2$ | Power balance deviation |
| $t_{\text{stress}}$ | Functional | $\sigma(I) / \sigma_{\text{max}}$ | Coil stress ratio |
| $t_{\text{safety}}$ | Functional | $[\max(0, q_{\text{min}} - q(r))]^2$ | Safety factor violation |

**Level 2a: Interaction Structures**

| ID | Type | Variables | Description |
|----|------|-----------|-------------|
| $\mathcal{I}_{\text{modes}}$ | Index set | $\{k : k \in \text{MHD modes}\}$ | All instability modes |
| $\mathcal{I}_{\text{coils}}$ | Index set | $\{c : c \in \text{coils}\}$ | All magnetic coils |
| $\mathcal{I}_{\text{radial}}$ | Grid | $\{r : r \in [0, a]\}$ | Radial profile points |

**Level 2b: Aggregators**

| Usage | Aggregator | Why |
|-------|------------|-----|
| MHD stability | **Max** | Worst-case mode determines stability |
| Power/constraints | Sum | Additive penalties |
| Safety profile | Sum | Integrated violation |

**Level 3: Components**

| Component | Term | Interaction | Aggregator | Meaning |
|-----------|------|-------------|------------|---------|
| $C_{\text{MHD}}$ | $t_{\text{growth}}$ | $\mathcal{I}_{\text{modes}}$ | **Max** | Worst MHD mode |
| $C_{\text{perf}}$ | $t_{\text{confinement}}$ | — | — | Confinement time |
| $C_{\text{balance}}$ | $t_{\text{power}}$ | — | Sum | Power balance |
| $C_{\text{coils}}$ | $t_{\text{stress}}$ | $\mathcal{I}_{\text{coils}}$ | Sum | Total coil stress |
| $C_{\text{safety}}$ | $t_{\text{safety}}$ | $\mathcal{I}_{\text{radial}}$ | Sum | Safety factor integral |

**Level 4: Hamiltonians**

| Hamiltonian | Type | Components | Description |
|-------------|------|------------|-------------|
| $H_{\text{MHD}}$ | Subsystem (physics) | $C_{\text{MHD}}$ | MHD stability |
| $H_{\text{performance}}$ | Subsystem (objective) | $C_{\text{perf}}$ | Maximize confinement |
| $H_{\text{power}}$ | **Coupling** | $C_{\text{balance}}$ | Links heating to transport |
| $H_{\text{engineering}}$ | Subsystem (constraint) | $C_{\text{coils}}$ | Hardware limits |
| $H_{\text{safety}}$ | Subsystem (constraint) | $C_{\text{safety}}$ | Safety margins |

*Note: $H_{\text{power}}$ is a Coupling Hamiltonian because power balance connects heating subsystem to transport subsystem.*

**Level 5: Ensemble**

$$H_{\text{total}} = w_1 H_{\text{MHD}} + w_2 H_{\text{performance}} + w_3 H_{\text{power}} + w_4 H_{\text{engineering}} + w_5 H_{\text{safety}}$$

### FEBO Specification

```yaml
febo_version: "0.1.1"
name: fusion_plasma_control

variables:
  coil_currents:
    type: continuous
    shape: [18]
    bounds: [-50000, 50000]    # Amps
    
  heating_power:
    type: continuous
    shape: [8]
    bounds: [0, 20e6]          # Watts
    
  density_profile:
    type: continuous
    shape: [100]               # Radial points
    bounds: [1e18, 1e21]       # particles/m³
    
  temperature_profile:
    type: continuous
    shape: [100]
    bounds: [100, 50000]       # eV

terms:
  t_growth:
    type: functional
    arity: n-ary
    expr: "mhd_growth_rate(k, coil_currents, density_profile, temperature_profile)"
    
  t_confinement:
    type: functional
    expr: "-confinement_time(density_profile, temperature_profile, heating_power)"
    
  t_power_balance:
    type: functional
    expr: "(P_input - P_loss - P_radiation)^2"
    
  t_coil_stress:
    type: functional
    expr: "stress(coil_currents[c]) / max_stress[c]"
    
  t_safety_factor:
    type: functional
    expr: "max(0, q_min - safety_factor(r))^2"

interactions:
  mhd_modes:
    type: all_indices
    range: [1, 10]
    bind: k
    
  coil_set:
    type: all_indices
    range: [1, 18]
    bind: c
    
  radial_grid:
    type: all_indices
    range: [1, 100]
    bind: r

components:
  c_mhd:
    term: t_growth
    interaction: mhd_modes
    agg: max           # WORST-CASE mode
    
  c_confinement:
    term: t_confinement
    
  c_power:
    term: t_power_balance
    agg: sum
    
  c_coils:
    term: t_coil_stress
    interaction: coil_set
    agg: sum
    
  c_safety:
    term: t_safety_factor
    interaction: radial_grid
    agg: sum

hamiltonians:
  H_MHD:
    type: subsystem
    description: "MHD stability - worst mode must be stable"
    components:
      - {use: c_mhd, alpha: 1.0}
    subsystem_vars: [coil_currents, density_profile, temperature_profile]
      
  H_performance:
    type: subsystem
    description: "Maximize energy confinement time"
    components:
      - {use: c_confinement, alpha: 1.0}
    subsystem_vars: [density_profile, temperature_profile, heating_power]
      
  H_power:
    type: coupling              # COUPLING: links heating to transport
    description: "Power balance between subsystems"
    components:
      - {use: c_power, alpha: 1.0}
    couples: [heating_power, density_profile, temperature_profile]
      
  H_engineering:
    type: subsystem
    description: "Hardware constraints"
    components:
      - {use: c_coils, alpha: 1.0}
    subsystem_vars: [coil_currents]
      
  H_safety:
    type: subsystem
    description: "Safety factor profile"
    components:
      - {use: c_safety, alpha: 1.0}

ensemble:
  - {use: H_MHD, weight: 1000.0}        # Critical: stability
  - {use: H_performance, weight: 1.0}
  - {use: H_power, weight: 100.0}       # Coupling
  - {use: H_engineering, weight: 10.0}
  - {use: H_safety, weight: 100.0}

constraints:
  time_budget: 0.001  # 1ms max solve time for real-time control
```

### Auditability Example

**Scenario: Disruption occurs during experiment**

```
H_total = 847.3  ✗ HIGH ENERGY (unstable state)

Breakdown:
├── H_MHD: 523.0         ✗ INSTABILITY DETECTED
│   └── Max over modes:
│       ├── mode (2,1): γ = 0.02  stable ✓
│       ├── mode (3,2): γ = 0.523 ✗ UNSTABLE (growth rate > 0)
│       └── mode (1,1): γ = -0.1  stable ✓
│
├── H_performance: -1.2  ✓ (τ_E = 1.2 sec)
├── H_power: 0.3         ✓ (near balance)
├── H_engineering: 4.2   ✓ (coils within limits)
└── H_safety: 320.0      ✗ SAFETY FACTOR VIOLATED
    └── Violated at r/a = 0.95: q = 1.8 < q_min = 2.0

Diagnosis: Mode (3,2) went unstable while safety factor dropped
Root cause: Density gradient too steep at edge
Action: Reduce edge fueling, increase outer coil current
```

### Scale Comparison

| Approach | Response Time | Can Prevent Disruption? |
|----------|---------------|------------------------|
| Manual operator | Seconds | ❌ Too slow |
| PID control | 10ms | ⚠️ Sometimes |
| Model predictive | 100ms | ⚠️ Often |
| **FEBO + Xtellix** | **<1ms** | **✅ Yes** |

---

## Problem 2: Power Grid Optimization

### The Problem

Optimize a continental power grid for the next 24-168 hours:
- Unit commitment (which generators on/off)
- Economic dispatch (how much power each)
- Transmission flow (respecting line limits)
- Storage scheduling (batteries, pumped hydro)
- Reserve allocation (handle contingencies)

**Why It's Hard:**
- Mixed integer (on/off) + continuous (power levels)
- N-1 contingency (survive any single failure)
- Non-convex (startup costs, transmission losses)
- Massive scale (10,000+ buses, 50,000+ lines, 168 hours)

### Mathematical Building Blocks

**Level 1: Terms**

| Term | Type | Expression | Description |
|------|------|------------|-------------|
| $t_{\text{gen}}$ | Static | $c_g \cdot p_g$ | Generation cost |
| $t_{\text{startup}}$ | Static | $S_g \cdot u_g$ | Startup cost |
| $t_{\text{balance}}$ | Static | $(\sum_g p_g - \sum_d D_d - \text{losses})^2$ | Power balance |
| $t_{\text{flow}}$ | Functional | $[\max(0, |f_\ell| - F_\ell^{\max})]^2$ | Line flow violation |
| $t_{\text{contingency}}$ | Functional | $\text{violation}_c$ | Post-contingency violation |
| $t_{\text{commit}}$ | Static | $(u_g - u_g^2)^2$ | Binary enforcement |

**Level 2a: Interaction Structures**

| ID | Type | Description |
|----|------|-------------|
| $\mathcal{I}_{\text{gen}}$ | Index set | All generators |
| $\mathcal{I}_{\text{lines}}$ | Index set | All transmission lines |
| $\mathcal{I}_{\text{buses}}$ | Index set | All buses/nodes |
| $\mathcal{I}_{\text{time}}$ | Index set | Time periods |
| $\mathcal{I}_{\text{contingencies}}$ | Index set | N-1 failure scenarios |

**Level 2b: Aggregators**

| Usage | Aggregator | Why |
|-------|------------|-----|
| Costs | Sum | Additive |
| N-1 security | **Max** | Worst contingency determines security |
| Constraints | Sum | Penalty aggregation |

**Level 3: Components**

| Component | Aggregator | Meaning |
|-----------|------------|---------|
| $C_{\text{cost}}$ | Sum | Total generation cost |
| $C_{\text{startup}}$ | Sum | Total startup costs |
| $C_{\text{balance}}$ | Sum | Power balance at all buses |
| $C_{\text{thermal}}$ | Sum | Line limit violations |
| $C_{\text{N-1}}$ | **Max** | Worst contingency violation |
| $C_{\text{binary}}$ | Sum | Integrality enforcement |

**Level 4: Hamiltonians**

| Hamiltonian | Type | Description |
|-------------|------|-------------|
| $H_{\text{cost}}$ | Subsystem | Economic objective |
| $H_{\text{physics}}$ | **Coupling** | Links generation to load (power balance) |
| $H_{\text{storage}}$ | Subsystem | Battery/hydro dynamics |
| $H_{\text{generators}}$ | Subsystem | Ramp rates, min up/down |
| $H_{\text{security}}$ | **Coupling** | N-1 contingency (links base case to failure scenarios) |
| $H_{\text{integrality}}$ | Subsystem | Binary enforcement |

**Level 5: Ensemble**

$$H_{\text{total}} = H_{\text{cost}} + H_{\text{physics}} + H_{\text{storage}} + H_{\text{generators}} + H_{\text{security}} + H_{\text{integrality}}$$

### FEBO Specification

```yaml
febo_version: "0.1.1"
name: power_grid_optimization

variables:
  commitment:      # u[g,t] ∈ {0,1}
    type: binary
    shape: [500, 168]    # 500 generators, 168 hours
    
  generation:      # p[g,t] ∈ [0, Pmax]
    type: continuous
    shape: [500, 168]
    
  voltage:         # V[b,t]
    type: continuous
    shape: [10000, 168]  # 10000 buses
    bounds: [0.95, 1.05]
    
  phase_angle:     # θ[b,t]
    type: continuous
    shape: [10000, 168]
    
  storage_state:   # s[b,t]
    type: continuous
    shape: [200, 168]    # 200 storage units

terms:
  t_gen_cost:
    expr: "cost[g] * generation[g,t]"
    
  t_startup_cost:
    expr: "startup[g] * max(0, commitment[g,t] - commitment[g,t-1])"
    
  t_power_balance:
    expr: "(sum_g(generation[g,t]) - sum_d(demand[d,t]) - losses[t])^2"
    
  t_line_violation:
    expr: "max(0, abs(flow[l,t]) - capacity[l])^2"
    
  t_contingency_violation:
    type: functional
    expr: "max_violation_after_contingency(c, t)"
    
  t_ramp:
    expr: "max(0, abs(generation[g,t] - generation[g,t-1]) - ramp_rate[g])^2"
    
  t_binary:
    expr: "(commitment[g,t] * (1 - commitment[g,t]))^2"

interactions:
  all_generators:
    type: all_indices
    range: [1, 500]
    bind: g
    
  all_lines:
    type: all_indices
    range: [1, 50000]
    bind: l
    
  all_buses:
    type: all_indices
    range: [1, 10000]
    bind: b
    
  all_time:
    type: all_indices
    range: [1, 168]
    bind: t
    
  contingencies:
    type: all_indices
    range: [1, 50000]
    bind: c

components:
  c_gen_cost:
    term: t_gen_cost
    interaction: [all_generators, all_time]
    agg: sum
    
  c_startup:
    term: t_startup_cost
    interaction: [all_generators, all_time]
    agg: sum
    
  c_balance:
    term: t_power_balance
    interaction: [all_buses, all_time]
    agg: sum
    
  c_thermal:
    term: t_line_violation
    interaction: [all_lines, all_time]
    agg: sum
    
  c_n1_security:
    term: t_contingency_violation
    interaction: [contingencies, all_time]
    agg: max         # WORST-CASE contingency
    
  c_ramp:
    term: t_ramp
    interaction: [all_generators, all_time]
    agg: sum
    
  c_binary:
    term: t_binary
    interaction: [all_generators, all_time]
    agg: sum

hamiltonians:
  H_cost:
    type: subsystem
    description: "Economic cost (objective)"
    components:
      - {use: c_gen_cost, alpha: 1.0}
      - {use: c_startup, alpha: 1.0}
      
  H_physics:
    type: coupling
    description: "Power balance links generation to demand"
    components:
      - {use: c_balance, alpha: 1.0}
      - {use: c_thermal, alpha: 1.0}
      
  H_generators:
    type: subsystem
    description: "Generator operational constraints"
    components:
      - {use: c_ramp, alpha: 1.0}
      
  H_security:
    type: coupling
    description: "N-1 contingency security (worst-case failure)"
    components:
      - {use: c_n1_security, alpha: 1.0}
      
  H_integrality:
    type: subsystem
    description: "Binary enforcement for commitment"
    components:
      - {use: c_binary, alpha: 1.0}

ensemble:
  - {use: H_cost, weight: 1.0}
  - {use: H_physics, weight: 10000.0}
  - {use: H_generators, weight: 1000.0}
  - {use: H_security, weight: 10000.0}
  - {use: H_integrality, weight: 100000.0}
```

### Auditability Example

**Scenario: Price spike investigation**

```
H_total = 2,847,523.00  (daily cost)

Cost Breakdown:
├── H_cost: 2,834,100
│   ├── c_gen_cost: $2,521,300
│   │   ├── Coal units: $892,000 (35%)
│   │   ├── Gas CC: $1,203,000 (48%)  ← Main driver
│   │   ├── Gas CT: $412,000 (16%)
│   │   └── Renewables: $14,300 (1%)
│   └── c_startup: $312,800
│       └── 47 startups, avg $6,655 each
│
├── H_physics: 0.0  ✓ (all balanced)
├── H_generators: 0.0  ✓ (ramps satisfied)
├── H_security: 13,423  ⚠️ BINDING
│   └── Limiting contingency: Line L-2847 failure
│       Post-contingency: Line L-1923 at 99.2% capacity
│
└── H_integrality: 0.0  ✓ (all binary)

Diagnosis: N-1 security constraint forcing expensive gas
Recommendation: Add 200MW transmission capacity on L-1923
Savings estimate: $127,000/day
```

### Scale Comparison

| Approach | Scale | Time | Gap |
|----------|-------|------|-----|
| CPLEX/Gurobi | 1,000 buses | Hours | 2-5% |
| Decomposition | 5,000 buses | Hours | Heuristic |
| **FEBO + Xtellix** | **10,000+ buses** | **Minutes** | **<0.1%** |

---

## Problem 3: Drug Discovery Pipeline

### The Problem

Optimize a drug candidate simultaneously for:
- Efficacy (binds target strongly)
- Safety (doesn't bind off-targets, non-toxic)
- Druggability (can be manufactured, delivered)
- Novelty (patentable, not prior art)

**Why It's Hard:**
- Vast search space (~10^60 drug-like molecules)
- Multi-objective (efficacy vs safety tradeoffs)
- Expensive evaluations (docking, ADMET prediction)
- Worst-case toxicity (any toxic endpoint = failure)

### Mathematical Building Blocks

**Level 1: Terms**

| Term | Type | Expression | Description |
|------|------|------------|-------------|
| $t_{\text{bind}}$ | Functional | $E_{\text{dock}}(m, \text{target})$ | Binding energy (lower = better) |
| $t_{\text{off}}$ | Functional | $E_{\text{dock}}(m, \text{off}_j)$ | Off-target binding |
| $t_{\text{tox}}$ | Functional | $\text{toxicity}_j(m)$ | Toxicity endpoint $j$ |
| $t_{\text{lipinski}}$ | Functional | $\text{violation}(m, \text{rule}_k)$ | Lipinski rule violation |
| $t_{\text{synth}}$ | Functional | $\text{SA\_score}(m)$ | Synthetic accessibility |
| $t_{\text{novel}}$ | Functional | $1 - \text{similarity}(m, \text{prior\_art})$ | Novelty score |

**Level 2a: Interaction Structures**

| ID | Type | Description |
|----|------|-------------|
| $\mathcal{I}_{\text{off-targets}}$ | Index set | All off-target proteins |
| $\mathcal{I}_{\text{tox-endpoints}}$ | Index set | All toxicity endpoints |
| $\mathcal{I}_{\text{lipinski}}$ | Index set | Lipinski rules (5) |

**Level 2b: Aggregators**

| Usage | Aggregator | Why |
|-------|------------|-----|
| Binding affinity | **Min** | Best docking pose |
| Off-target binding | **Max** | Worst off-target determines selectivity |
| Toxicity | **Max** | Any toxic endpoint = failure |
| Druggability | Sum | Additive violations |

**Level 3: Components**

| Component | Aggregator | Meaning |
|-----------|------------|---------|
| $C_{\text{efficacy}}$ | **Min** | Best binding pose energy |
| $C_{\text{selectivity}}$ | **Max** | Worst off-target binding |
| $C_{\text{toxicity}}$ | **Max** | Worst toxicity endpoint |
| $C_{\text{druggability}}$ | Sum | Total Lipinski violations |
| $C_{\text{practical}}$ | Sum | Synthesis + novelty |

**Level 4: Hamiltonians**

| Hamiltonian | Type | Description |
|-------------|------|-------------|
| $H_{\text{efficacy}}$ | Subsystem | Binding to target |
| $H_{\text{safety}}$ | Subsystem | Toxicity + off-target |
| $H_{\text{druggability}}$ | Subsystem | ADMET properties |
| $H_{\text{practical}}$ | Subsystem | Synthesis + IP |

*Note: No Coupling Hamiltonian—these are different evaluation criteria on the same molecule, not different subsystems.*

**Level 5: Ensemble**

$$H_{\text{total}} = w_1 H_{\text{efficacy}} + w_2 H_{\text{safety}} + w_3 H_{\text{druggability}} + w_4 H_{\text{practical}}$$

### FEBO Specification

```yaml
febo_version: "0.1.1"
name: drug_discovery_pipeline

variables:
  molecule:
    type: molecular_structure    # Special type for molecules
    representation: SMILES
    constraints:
      - max_atoms: 50
      - max_rings: 5
      
  conformation:
    type: continuous
    shape: [50, 3]              # 3D coordinates
    
  binding_pose:
    type: continuous
    shape: [6]                  # translation + rotation

terms:
  t_binding:
    type: functional
    expr: "docking_score(molecule, conformation, target_protein)"
    model: "autodock_vina"
    
  t_off_target:
    type: functional
    expr: "docking_score(molecule, conformation, off_target[j])"
    
  t_toxicity:
    type: functional
    expr: "tox_model(molecule, endpoint[j])"
    model: "tox21_ensemble"
    
  t_lipinski:
    type: functional
    expr: "lipinski_violation(molecule, rule[k])"
    
  t_admet:
    type: functional
    expr: "admet_deviation(molecule, property[p])"
    
  t_synthetic:
    type: functional
    expr: "sa_score(molecule)"
    
  t_novelty:
    type: functional
    expr: "1 - max_tanimoto(molecule, prior_art_db)"

interactions:
  off_targets:
    type: all_indices
    range: [1, 50]
    bind: j
    
  tox_endpoints:
    type: all_indices
    range: [1, 12]
    bind: j
    
  lipinski_rules:
    type: all_indices
    range: [1, 5]
    bind: k

components:
  c_binding:
    term: t_binding
    agg: min              # Best pose
    
  c_selectivity:
    term: t_off_target
    interaction: off_targets
    agg: max              # Worst off-target
    
  c_toxicity:
    term: t_toxicity
    interaction: tox_endpoints
    agg: max              # Worst toxicity
    
  c_lipinski:
    term: t_lipinski
    interaction: lipinski_rules
    agg: sum
    
  c_admet:
    term: t_admet
    agg: sum
    
  c_synthesis:
    term: t_synthetic
    
  c_novelty:
    term: t_novelty

hamiltonians:
  H_efficacy:
    description: "Drug-target interaction"
    components:
      - {use: c_binding, alpha: 1.0}
      - {use: c_selectivity, alpha: -0.5}  # Penalize off-target
      
  H_safety:
    description: "Safety profile"
    components:
      - {use: c_toxicity, alpha: 10.0}  # High weight on safety
      
  H_druggability:
    description: "Drug-likeness"
    components:
      - {use: c_lipinski, alpha: 1.0}
      - {use: c_admet, alpha: 1.0}
      
  H_practical:
    description: "Practical considerations"
    components:
      - {use: c_synthesis, alpha: 0.5}
      - {use: c_novelty, alpha: -1.0}   # Reward novelty

ensemble:
  - {use: H_efficacy, weight: 1.0}
  - {use: H_safety, weight: 2.0}
  - {use: H_druggability, weight: 0.5}
  - {use: H_practical, weight: 0.3}
```

### Auditability Example

**Scenario: Why was candidate MOL-4721 rejected?**

```
H_total(MOL-4721) = 8.34  ✗ REJECTED (threshold: 2.0)

Breakdown:
├── H_efficacy: -2.1  ✓ GOOD
│   ├── c_binding: -8.4 kcal/mol (strong binder) ✓
│   └── c_selectivity: 1.2 (some off-target, acceptable)
│
├── H_safety: 6.7  ✗ FAILED
│   └── c_toxicity: Max = 0.67
│       ├── hepatotox: 0.67  ✗ EXCEEDS 0.3 THRESHOLD
│       ├── cardiotox: 0.12  ✓
│       ├── nephrotox: 0.08  ✓
│       └── mutagenicity: 0.03  ✓
│
├── H_druggability: 2.1  ⚠️ MARGINAL
│   ├── c_lipinski: 1 violation (MW = 523 > 500)
│   └── c_admet: solubility borderline
│
└── H_practical: 1.6  ✓ OK
    ├── c_synthesis: SA = 3.2 (moderate)
    └── c_novelty: 0.73 (sufficiently novel)

Diagnosis: Hepatotoxicity from metabolic liability
Root cause: Aromatic amine at position 4
Recommendation: Replace with amide, re-evaluate
```

### Scale Comparison

| Approach | Molecules/Day | Multi-Objective? | Explainable? |
|----------|--------------|------------------|--------------|
| Traditional HTS | 100,000 | ❌ Sequential | ❌ |
| ML screening | 1,000,000 | ⚠️ Separate models | ⚠️ |
| **FEBO + Xtellix** | **10,000,000+** | **✅ Simultaneous** | **✅ Full audit** |

---

## Problem 4: Global Supply Chain

### The Problem

Optimize a multi-echelon global supply chain:
- Raw materials → Components → Products → Distribution → Customers
- Multiple suppliers, plants, warehouses, markets
- Uncertainty in demand and supply
- Service level requirements

**Why It's Hard:**
- Multi-echelon coupling (decisions cascade)
- Mixed integer (facility selection, lot sizing)
- Stochastic demand
- Massive scale (millions of SKU-location-time combinations)

### Mathematical Building Blocks

**Level 1: Terms**

| Term | Type | Expression | Description |
|------|------|------------|-------------|
| $t_{\text{prod}}$ | Static | $c_{fp} \cdot x_{fpt}$ | Production cost |
| $t_{\text{hold}}$ | Static | $h_{wp} \cdot I_{wpt}$ | Holding cost |
| $t_{\text{trans}}$ | Static | $d_{ij} \cdot y_{ijpt}$ | Transportation cost |
| $t_{\text{stockout}}$ | Static | $s_p \cdot [\max(0, D - I)]^2$ | Stockout penalty |
| $t_{\text{balance}}$ | Static | $(\text{in} - \text{out} - \Delta I)^2$ | Inventory balance |
| $t_{\text{link}}$ | Static | $(\text{output}_{\text{tier1}} - \text{input}_{\text{tier2}})^2$ | Echelon coupling |

**Level 2a: Interaction Structures**

| ID | Type | Description |
|----|------|-------------|
| $\mathcal{I}_{\text{facilities}}$ | Index set | All facilities |
| $\mathcal{I}_{\text{products}}$ | Index set | All SKUs |
| $\mathcal{I}_{\text{time}}$ | Index set | Planning periods |
| $\mathcal{I}_{\text{lanes}}$ | Sparse | Valid transportation lanes |

**Level 2b: Aggregators**

| Usage | Aggregator | Why |
|-------|------------|-----|
| All costs | Sum | Additive |
| Service level | Sum | Aggregate penalties |

**Level 3: Components**

| Component | Aggregator | Meaning |
|-----------|------------|---------|
| $C_{\text{production}}$ | Sum | Total production cost |
| $C_{\text{inventory}}$ | Sum | Total holding cost |
| $C_{\text{transport}}$ | Sum | Total transportation cost |
| $C_{\text{service}}$ | Sum | Total stockout penalty |
| $C_{\text{balance}}$ | Sum | Inventory balance |
| $C_{\text{echelon}}$ | Sum | Tier coupling |

**Level 4: Hamiltonians**

| Hamiltonian | Type | Description |
|-------------|------|-------------|
| $H_{\text{cost}}$ | Subsystem | Operating costs |
| $H_{\text{service}}$ | Subsystem | Customer service |
| $H_{\text{physics}}$ | Subsystem | Inventory balance |
| $H_{\text{raw-comp}}$ | **Coupling** | Raw materials → Components |
| $H_{\text{comp-prod}}$ | **Coupling** | Components → Products |
| $H_{\text{prod-dist}}$ | **Coupling** | Products → Distribution |

*The Coupling Hamiltonians are critical: they link the echelons so optimization considers end-to-end effects.*

**Level 5: Ensemble**

$$H_{\text{total}} = H_{\text{cost}} + H_{\text{service}} + H_{\text{physics}} + H_{\text{raw-comp}} + H_{\text{comp-prod}} + H_{\text{prod-dist}}$$

### FEBO Specification

```yaml
febo_version: "0.1.1"
name: global_supply_chain

variables:
  production:        # x[f,p,t]
    type: continuous
    shape: [50, 1000, 52]    # 50 plants, 1000 products, 52 weeks
    
  inventory:         # I[w,p,t]
    type: continuous
    shape: [200, 1000, 52]   # 200 warehouses
    
  shipment:          # y[i,j,p,t]
    type: continuous
    shape: [10000, 52]       # 10000 lanes × 52 weeks (sparse)
    
  fulfillment:       # z[c,p,t]
    type: continuous
    shape: [500, 1000, 52]   # 500 customers

terms:
  t_production_cost:
    expr: "unit_cost[f,p] * production[f,p,t]"
    
  t_holding_cost:
    expr: "holding_cost[w,p] * inventory[w,p,t]"
    
  t_transport_cost:
    expr: "lane_cost[i,j] * shipment[i,j,p,t]"
    
  t_stockout:
    expr: "stockout_penalty[c,p] * max(0, demand[c,p,t] - fulfillment[c,p,t])^2"
    
  t_inventory_balance:
    expr: "(inventory[w,p,t] - inventory[w,p,t-1] - inflow + outflow)^2"
    
  t_echelon_link:
    expr: "(output[tier1,p,t] - input[tier2,p,t])^2"

interactions:
  all_production:
    type: all_indices
    ranges: {f: [1, 50], p: [1, 1000], t: [1, 52]}
    
  all_inventory:
    type: all_indices
    ranges: {w: [1, 200], p: [1, 1000], t: [1, 52]}
    
  transport_lanes:
    type: sparse
    source: "valid_lanes.csv"
    bind: [i, j]
    
  customer_demand:
    type: all_indices
    ranges: {c: [1, 500], p: [1, 1000], t: [1, 52]}

components:
  c_prod_cost:
    term: t_production_cost
    interaction: all_production
    agg: sum
    
  c_hold_cost:
    term: t_holding_cost
    interaction: all_inventory
    agg: sum
    
  c_trans_cost:
    term: t_transport_cost
    interaction: transport_lanes
    agg: sum
    
  c_service:
    term: t_stockout
    interaction: customer_demand
    agg: sum
    
  c_balance:
    term: t_inventory_balance
    interaction: all_inventory
    agg: sum
    
  c_echelon_12:
    term: t_echelon_link
    interaction: {tier1: raw_suppliers, tier2: component_plants}
    agg: sum
    
  c_echelon_23:
    term: t_echelon_link
    interaction: {tier1: component_plants, tier2: assembly_plants}
    agg: sum
    
  c_echelon_34:
    term: t_echelon_link
    interaction: {tier1: assembly_plants, tier2: distribution_centers}
    agg: sum

hamiltonians:
  H_cost:
    type: subsystem
    description: "Total operating cost"
    components:
      - {use: c_prod_cost, alpha: 1.0}
      - {use: c_hold_cost, alpha: 1.0}
      - {use: c_trans_cost, alpha: 1.0}
      
  H_service:
    type: subsystem
    description: "Customer service level"
    components:
      - {use: c_service, alpha: 1.0}
      
  H_physics:
    type: subsystem
    description: "Inventory balance constraints"
    components:
      - {use: c_balance, alpha: 1.0}
      
  H_raw_component:
    type: coupling
    description: "Raw → Component echelon coupling"
    components:
      - {use: c_echelon_12, alpha: 1.0}
    couples: [raw_materials, components]
      
  H_component_product:
    type: coupling
    description: "Component → Product echelon coupling"
    components:
      - {use: c_echelon_23, alpha: 1.0}
    couples: [components, products]
      
  H_product_distribution:
    type: coupling
    description: "Product → Distribution echelon coupling"
    components:
      - {use: c_echelon_34, alpha: 1.0}
    couples: [products, distribution]

ensemble:
  - {use: H_cost, weight: 1.0}
  - {use: H_service, weight: 10.0}
  - {use: H_physics, weight: 10000.0}
  - {use: H_raw_component, weight: 10000.0}
  - {use: H_component_product, weight: 10000.0}
  - {use: H_product_distribution, weight: 10000.0}
```

### Auditability Example

**Scenario: Why is product P-4521 frequently out of stock?**

```
Tracing stockout for P-4521 at DC-Chicago, Week 23:

H_service contribution: $47,200 stockout penalty

Backward trace through Coupling Hamiltonians:

H_product_distribution (Products → Distribution):
├── Assembly plant AP-Detroit → DC-Chicago
└── Shipment: 0 units (no supply available)  ✗

H_component_product (Components → Products):
├── AP-Detroit needed component C-892
└── Inventory: 0 (stockout at component level)  ✗

H_raw_component (Raw → Components):
├── Component plant CP-Mexico needed raw material R-17
└── Supplier S-China: 3-week delay (port congestion)  ← ROOT CAUSE

Diagnosis: Port congestion in China → R-17 delay → C-892 stockout → P-4521 unavailable
Recommendation: 
  1. Air-freight R-17 ($12,000) vs stockout cost ($47,200)
  2. Add safety stock for C-892 at CP-Mexico
  3. Qualify backup supplier for R-17
```

### Scale Comparison

| Approach | Scale | Echelons | Time |
|----------|-------|----------|------|
| SAP APO | 10K SKU-locations | 2 (sequential) | Hours |
| Llamasoft | 100K SKU-locations | 3 (heuristic) | Hours |
| **FEBO + Xtellix** | **1M+ SKU-locations** | **4+ (simultaneous)** | **Minutes** |

---

## Problem 5: Autonomous Fleet Management

### The Problem

Manage a fleet of autonomous vehicles in real-time:
- Assign vehicles to ride requests
- Route vehicles efficiently
- Schedule charging/maintenance
- Balance supply and demand across zones

**Why It's Hard:**
- Real-time (<100ms decisions)
- Stochastic demand
- Battery constraints (continuous resource)
- Multi-objective (service, efficiency, safety)

### Mathematical Building Blocks

**Level 1: Terms**

| Term | Type | Expression | Description |
|------|------|------------|-------------|
| $t_{\text{wait}}$ | Static | $(t_{\text{pickup}} - t_{\text{request}})^2$ | Wait time squared |
| $t_{\text{distance}}$ | Static | $d_{vr}$ | Distance vehicle to request |
| $t_{\text{battery}}$ | Functional | $[\max(0, B_{\text{min}} - B_{vt})]^2$ | Battery constraint |
| $t_{\text{collision}}$ | Functional | $\text{risk}(v_1, v_2, t)$ | Collision risk |
| $t_{\text{assign}}$ | Static | $(x_{vr} - x_{vr}^2)$ | Binary assignment |

**Level 2a: Interaction Structures**

| ID | Type | Description |
|----|------|-------------|
| $\mathcal{I}_{\text{vehicles}}$ | Index set | All vehicles |
| $\mathcal{I}_{\text{requests}}$ | Index set | Active ride requests |
| $\mathcal{I}_{\text{pairs}}$ | Sparse | Vehicle pairs for collision |
| $\mathcal{I}_{\text{time}}$ | Index set | Time steps |

**Level 2b: Aggregators**

| Usage | Aggregator | Why |
|-------|------------|-----|
| Service quality | Sum | Total wait time |
| Battery constraint | **Min** | Worst battery level |
| Safety | **Max** | Worst collision risk |

**Level 3: Components**

| Component | Aggregator | Meaning |
|-----------|------------|---------|
| $C_{\text{wait}}$ | Sum | Total wait time |
| $C_{\text{distance}}$ | Sum | Total distance |
| $C_{\text{battery}}$ | **Min** | Minimum battery level |
| $C_{\text{safety}}$ | **Max** | Maximum collision risk |
| $C_{\text{binary}}$ | Sum | Assignment integrality |

**Level 4: Hamiltonians**

| Hamiltonian | Type | Description |
|-------------|------|-------------|
| $H_{\text{service}}$ | Subsystem | Customer experience |
| $H_{\text{efficiency}}$ | Subsystem | Fleet efficiency |
| $H_{\text{battery}}$ | Subsystem | Energy management |
| $H_{\text{safety}}$ | **Coupling** | Vehicle-vehicle collision avoidance |
| $H_{\text{integrality}}$ | Subsystem | Assignment constraints |

*$H_{\text{safety}}$ is a Coupling Hamiltonian because it involves interactions between different vehicles (subsystems).*

### FEBO Specification

```yaml
febo_version: "0.1.1"
name: autonomous_fleet_management

variables:
  assignment:        # x[v,r] ∈ {0,1}
    type: binary
    shape: [1000, 500]    # 1000 vehicles, 500 active requests
    
  pickup_time:       # t[v,r]
    type: continuous
    shape: [1000, 500]
    
  route:             # path[v,t]
    type: continuous
    shape: [1000, 100, 2]  # 1000 vehicles, 100 time steps, (x,y)
    
  charging:          # c[v,s,t]
    type: binary
    shape: [1000, 50, 24]  # vehicles × stations × hours
    
  battery:           # B[v,t]
    type: continuous
    shape: [1000, 100]
    bounds: [0, 100]

terms:
  t_wait_time:
    expr: "(pickup_time[v,r] - request_time[r])^2"
    
  t_trip_distance:
    expr: "distance(vehicle_pos[v], pickup_loc[r]) + trip_distance[r]"
    
  t_battery_violation:
    expr: "max(0, B_min - battery[v,t])^2"
    
  t_collision_risk:
    type: functional
    expr: "collision_probability(route[v1,t], route[v2,t])"
    
  t_double_assignment:
    expr: "(sum_v(assignment[v,r]) - 1)^2"
    
  t_binary_enforce:
    expr: "assignment[v,r] * (1 - assignment[v,r])"

interactions:
  all_assignments:
    type: all_indices
    ranges: {v: [1, 1000], r: [1, 500]}
    
  all_vehicles_time:
    type: all_indices
    ranges: {v: [1, 1000], t: [1, 100]}
    
  vehicle_pairs:
    type: sparse
    source: "nearby_pairs.csv"
    bind: [v1, v2]
    
  all_requests:
    type: all_indices
    range: [1, 500]
    bind: r

components:
  c_wait:
    term: t_wait_time
    interaction: all_assignments
    agg: sum
    
  c_distance:
    term: t_trip_distance
    interaction: all_assignments
    agg: sum
    
  c_battery:
    term: t_battery_violation
    interaction: all_vehicles_time
    agg: min            # WORST battery level
    
  c_collision:
    term: t_collision_risk
    interaction: vehicle_pairs
    agg: max            # WORST collision risk
    
  c_single_assign:
    term: t_double_assignment
    interaction: all_requests
    agg: sum
    
  c_binary:
    term: t_binary_enforce
    interaction: all_assignments
    agg: sum

hamiltonians:
  H_service:
    type: subsystem
    description: "Customer wait time"
    components:
      - {use: c_wait, alpha: 1.0}
      
  H_efficiency:
    type: subsystem
    description: "Fleet efficiency (minimize distance)"
    components:
      - {use: c_distance, alpha: 1.0}
      
  H_battery:
    type: subsystem
    description: "Battery constraints"
    components:
      - {use: c_battery, alpha: 1.0}
      
  H_safety:
    type: coupling
    description: "Inter-vehicle collision avoidance"
    components:
      - {use: c_collision, alpha: 1.0}
    couples: [vehicle_1, vehicle_2]
      
  H_feasibility:
    type: subsystem
    description: "Assignment constraints"
    components:
      - {use: c_single_assign, alpha: 1.0}
      - {use: c_binary, alpha: 1.0}

ensemble:
  - {use: H_service, weight: 1.0}
  - {use: H_efficiency, weight: 0.1}
  - {use: H_battery, weight: 100.0}
  - {use: H_safety, weight: 10000.0}
  - {use: H_feasibility, weight: 100000.0}

constraints:
  max_solve_time: 0.1  # 100ms for real-time
```

### Auditability Example

**Scenario: Why was vehicle V-0847 assigned to request R-4521 instead of closer V-0312?**

```
Assignment decision for R-4521:

Candidate vehicles:
├── V-0312: 0.3 km away
├── V-0847: 1.2 km away (SELECTED)
└── V-0923: 0.8 km away

Energy decomposition for each assignment:

Option: Assign V-0312
├── H_service: 0.09  ✓ (short wait)
├── H_efficiency: 0.3 ✓
├── H_battery: 847.0 ✗ FAILED
│   └── V-0312 battery: 8% → Would drop to 2% < 5% minimum
├── H_safety: 0.0  ✓
└── H_total: 847.4  ✗

Option: Assign V-0847 (SELECTED)
├── H_service: 1.44  (longer wait, acceptable)
├── H_efficiency: 1.2
├── H_battery: 0.0  ✓ (V-0847 battery: 72%)
├── H_safety: 0.0  ✓
└── H_total: 2.64  ✓ LOWEST

Option: Assign V-0923
├── H_service: 0.64
├── H_efficiency: 0.8
├── H_battery: 0.0  ✓
├── H_safety: 234.0 ✗ COLLISION RISK
│   └── V-0923 route conflicts with V-0156 at intersection
└── H_total: 235.4  ✗

Decision: V-0847 selected due to battery constraint on V-0312 and collision risk for V-0923
```

### Scale Comparison

| Approach | Fleet Size | Decision Time | Optimal? |
|----------|------------|---------------|----------|
| Greedy | 10,000 | 10ms | ❌ |
| Hungarian | 1,000 | 100ms | ⚠️ Local |
| **FEBO + Xtellix** | **10,000+** | **<100ms** | **✅ Global** |

---

## Problem 6: Large-Scale Portfolio Optimization

### The Problem

Optimize a portfolio with:
- 10,000+ assets
- Multiple risk measures (variance, CVaR, drawdown)
- Transaction costs (nonlinear)
- Constraints (sectors, factors, ESG)
- Cardinality (limit number of positions)

**Why It's Hard:**
- Non-convex (cardinality, transaction costs)
- Multiple risk measures (variance + CVaR + drawdown)
- Large-scale covariance (10,000 × 10,000)
- Worst-case drawdown across scenarios

### Mathematical Building Blocks

**Level 1: Terms**

| Term | Type | Expression | Description |
|------|------|------------|-------------|
| $t_{\text{return}}$ | Static | $\mu_i w_i$ | Expected return |
| $t_{\text{var}}$ | Static | $\Sigma_{ij} w_i w_j$ | Covariance contribution |
| $t_{\text{cvar}}$ | Functional | $\text{CVaR}_\alpha(w)$ | Conditional VaR |
| $t_{\text{drawdown}}$ | Functional | $\text{drawdown}_t(w)$ | Drawdown at time $t$ |
| $t_{\text{cost}}$ | Functional | $\text{cost}(w - w_0)$ | Transaction cost |
| $t_{\text{position}}$ | Static | $\mathbf{1}[w_i > 0]$ | Position indicator |

**Level 2a: Interaction Structures**

| ID | Type | Description |
|----|------|-------------|
| $\mathcal{I}_{\text{assets}}$ | Index set | All assets |
| $\mathcal{I}_{\text{pairs}}$ | Dense | Asset pairs for covariance |
| $\mathcal{I}_{\text{scenarios}}$ | Index set | Historical scenarios |
| $\mathcal{I}_{\text{sectors}}$ | Index set | Sector groupings |
| $\mathcal{I}_{\text{factors}}$ | Index set | Risk factors |

**Level 2b: Aggregators**

| Usage | Aggregator | Why |
|-------|------------|-----|
| Return | Sum | Additive |
| Variance | Sum | Quadratic form |
| CVaR | Mean (tail) | Expected tail loss |
| Drawdown | **Max** | Worst drawdown determines pain |
| Constraints | Sum | Penalty aggregation |

**Level 3: Components**

| Component | Aggregator | Meaning |
|-----------|------------|---------|
| $C_{\text{return}}$ | Sum | Expected return |
| $C_{\text{variance}}$ | Sum | Portfolio variance |
| $C_{\text{cvar}}$ | Mean | Conditional VaR |
| $C_{\text{drawdown}}$ | **Max** | Maximum drawdown |
| $C_{\text{cost}}$ | Sum | Total transaction cost |
| $C_{\text{cardinality}}$ | Sum | Position count |

**Level 4: Hamiltonians**

| Hamiltonian | Type | Description |
|-------------|------|-------------|
| $H_{\text{return}}$ | Subsystem | Return objective |
| $H_{\text{risk}}$ | Subsystem | Risk measures |
| $H_{\text{cost}}$ | Subsystem | Trading costs |
| $H_{\text{budget}}$ | Subsystem | Fully invested |
| $H_{\text{sectors}}$ | Subsystem | Sector limits |
| $H_{\text{factors}}$ | **Coupling** | Factor exposure links assets |
| $H_{\text{esg}}$ | Subsystem | ESG constraints |
| $H_{\text{cardinality}}$ | Subsystem | Position limits |

*$H_{\text{factors}}$ is a Coupling Hamiltonian because factor exposures create correlations across assets (linking their "subsystems").*

### FEBO Specification

```yaml
febo_version: "0.1.1"
name: portfolio_optimization

variables:
  weights:           # w[i]
    type: continuous
    shape: [10000]
    bounds: [-0.1, 0.1]    # Allow short up to 10%
    
  trades:            # Δw[i]
    type: continuous
    shape: [10000]
    
  active:            # a[i] ∈ {0,1}
    type: binary
    shape: [10000]

data:
  expected_returns:
    shape: [10000]
    source: "returns.csv"
    
  covariance:
    shape: [10000, 10000]
    source: "covariance.h5"
    format: low_rank        # Store as U'U, rank 100
    
  factor_loadings:
    shape: [10000, 50]
    source: "factors.csv"
    
  scenarios:
    shape: [10000, 1000]    # 1000 historical scenarios
    source: "scenarios.h5"

terms:
  t_return:
    expr: "expected_returns[i] * weights[i]"
    
  t_variance:
    expr: "covariance[i,j] * weights[i] * weights[j]"
    
  t_scenario_return:
    expr: "scenarios[i,s] * weights[i]"
    
  t_drawdown:
    type: functional
    expr: "drawdown(cumulative_return[t])"
    
  t_transaction:
    type: functional
    expr: "transaction_cost(trades[i])"  # Nonlinear
    
  t_budget:
    expr: "(sum(weights) - 1)^2"
    
  t_sector_exposure:
    expr: "max(0, sum(weights[i] for i in sector[s]) - limit[s])^2"
    
  t_factor_exposure:
    expr: "(sum(factor_loadings[i,f] * weights[i]) - target[f])^2"
    
  t_esg_score:
    expr: "max(0, min_esg - sum(esg[i] * weights[i]))^2"
    
  t_cardinality:
    expr: "max(0, sum(active) - max_positions)^2"
    
  t_active_link:
    expr: "(weights[i]^2 - active[i] * weights[i]^2)^2"  # active=0 → weights=0

interactions:
  all_assets:
    type: all_indices
    range: [1, 10000]
    bind: i
    
  asset_pairs:
    type: all_indices
    ranges: {i: [1, 10000], j: [1, 10000]}
    # Low-rank storage handled by data section
    
  scenarios:
    type: all_indices
    range: [1, 1000]
    bind: s
    
  time_periods:
    type: all_indices
    range: [1, 252]
    bind: t
    
  sectors:
    type: all_indices
    range: [1, 11]
    bind: s
    
  factors:
    type: all_indices
    range: [1, 50]
    bind: f

components:
  c_return:
    term: t_return
    interaction: all_assets
    agg: sum
    
  c_variance:
    term: t_variance
    interaction: asset_pairs
    agg: sum
    
  c_cvar:
    term: t_scenario_return
    interaction: [all_assets, scenarios]
    agg: cvar_95      # Bottom 5% mean
    
  c_drawdown:
    term: t_drawdown
    interaction: time_periods
    agg: max          # WORST drawdown
    
  c_cost:
    term: t_transaction
    interaction: all_assets
    agg: sum
    
  c_budget:
    term: t_budget
    
  c_sectors:
    term: t_sector_exposure
    interaction: sectors
    agg: sum
    
  c_factors:
    term: t_factor_exposure
    interaction: factors
    agg: sum
    
  c_esg:
    term: t_esg_score
    
  c_cardinality:
    term: t_cardinality
    
  c_active:
    term: t_active_link
    interaction: all_assets
    agg: sum

hamiltonians:
  H_return:
    type: subsystem
    description: "Expected return (maximize → negative weight)"
    components:
      - {use: c_return, alpha: -1.0}
      
  H_risk:
    type: subsystem
    description: "Risk measures"
    components:
      - {use: c_variance, alpha: 1.0}
      - {use: c_cvar, alpha: 0.5}
      - {use: c_drawdown, alpha: 0.3}
      
  H_cost:
    type: subsystem
    description: "Transaction costs"
    components:
      - {use: c_cost, alpha: 1.0}
      
  H_budget:
    type: subsystem
    description: "Fully invested constraint"
    components:
      - {use: c_budget, alpha: 1.0}
      
  H_sectors:
    type: subsystem
    description: "Sector exposure limits"
    components:
      - {use: c_sectors, alpha: 1.0}
      
  H_factors:
    type: coupling
    description: "Factor exposure targets"
    components:
      - {use: c_factors, alpha: 1.0}
    couples: [assets_via_factors]
      
  H_esg:
    type: subsystem
    description: "ESG compliance"
    components:
      - {use: c_esg, alpha: 1.0}
      
  H_cardinality:
    type: subsystem
    description: "Position count limits"
    components:
      - {use: c_cardinality, alpha: 1.0}
      - {use: c_active, alpha: 1.0}

ensemble:
  - {use: H_return, weight: 1.0}
  - {use: H_risk, weight: 0.5}        # Risk aversion
  - {use: H_cost, weight: 0.1}
  - {use: H_budget, weight: 10000.0}
  - {use: H_sectors, weight: 1000.0}
  - {use: H_factors, weight: 100.0}
  - {use: H_esg, weight: 1000.0}
  - {use: H_cardinality, weight: 100.0}
```

### Auditability Example

**Scenario: Monthly portfolio review - why did we underperform?**

```
Portfolio Analysis for Q4 2024:

H_total decomposition at current weights:

├── H_return: -0.0847        [Expected: 8.47%/year]
│   └── Top contributors:
│       ├── AAPL: 0.8%
│       ├── MSFT: 0.7%
│       └── NVDA: 0.6%
│
├── H_risk: 0.0234           [Total risk measure]
│   ├── c_variance: 0.0156   [Vol: 12.5%]
│   │   └── Factor attribution:
│   │       ├── Market beta: 68%
│   │       ├── Size: 12%
│   │       ├── Value: 8%
│   │       └── Idiosyncratic: 12%
│   │
│   ├── c_cvar: 0.0045       [CVaR 95%: -2.1%]
│   │   └── Worst scenarios: Oct 2008, Mar 2020, Sep 2022
│   │
│   └── c_drawdown: 0.0033   [Max DD: 18%]
│       └── Binding period: Aug-Oct 2024  ← ACTUAL DRAWDOWN
│
├── H_cost: 0.0012           [Transaction costs: 12bps]
├── H_budget: 0.0000         ✓ Fully invested
├── H_sectors: 0.0000        ✓ Within limits
├── H_factors: 0.0023        ⚠️ Slightly off factor targets
│   └── Value exposure: -0.15 vs target 0.0
└── H_cardinality: 0.0000    ✓ 127 positions < 150 limit

Diagnosis: 
- Underperformance driven by c_drawdown binding in Q4
- Value underweight hurt during rotation
- Max drawdown term correctly flagged Oct 2024 as painful period

Recommendation:
- Increase value exposure (+0.15)
- Consider reducing max drawdown weight if tracking error acceptable
```

### Scale Comparison

| Approach | Assets | Risk Measures | Time |
|----------|--------|---------------|------|
| CVXPY | 1,000 | Variance only | Seconds |
| Gurobi | 3,000 | Variance + cardinality | Minutes |
| **FEBO + Xtellix** | **10,000+** | **Var + CVaR + MaxDD** | **Seconds** |

---

## Summary: Patterns Across Problems

### Aggregator Usage

| Aggregator | When to Use | Examples |
|------------|-------------|----------|
| **Sum** | Additive costs, penalties | Most objectives, constraints |
| **Max** | Worst-case, safety-critical | MHD stability, N-1 security, max drawdown, collision risk |
| **Min** | Resource floors | Battery levels, minimum binding affinity |
| **Prod** | Multiplicative coupling | Griewank, reliability |
| **Mean/CVaR** | Stochastic, tail risk | CVaR, scenario averaging |

### Coupling Hamiltonian Patterns

| Pattern | Example | What It Couples |
|---------|---------|-----------------|
| Multi-physics | Fusion | MHD ↔ Transport ↔ Heating |
| Multi-echelon | Supply chain | Raw → Component → Product → Distribution |
| Multi-system | Power grid | Generation ↔ Transmission ↔ Load |
| Safety coupling | Fleet | Vehicle ↔ Vehicle (collision) |
| Factor coupling | Portfolio | Asset ↔ Asset (via factors) |

### Auditability Value

| Use Case | Question Answered |
|----------|-------------------|
| **Debugging** | Why did the optimization fail? Which H violated? |
| **Explanation** | Why was this decision made? Energy decomposition. |
| **Root cause** | Trace through Coupling H to find upstream cause |
| **Compliance** | Auditable record for regulators |
| **Tuning** | Which H is binding? Adjust weights. |

---

## Appendix: FEBO v1 Primitives Summary

### Terms

| Type | Arity | Examples |
|------|-------|----------|
| Static | Constant, Unary, Binary | $c$, $x_i^2$, $Q_{ij} x_i x_j$ |
| Functional | Any | $\text{neural\_net}(x)$, $\text{simulate}(x)$ |

### Interaction Structures

| Type | Description |
|------|-------------|
| All indices | $\{i : i \in 1..n\}$ |
| Sparse | Edge list, graph |
| Dense | All pairs |
| Low-rank | Implicit via factorization |
| Constraint-induced | One-hot groups, flow, capacity |

### Aggregators

| Aggregator | Formula | Use |
|------------|---------|-----|
| Sum | $\sum_i t_i$ | Default, additive |
| Prod | $\prod_i t_i$ | Multiplicative |
| Max | $\max_i t_i$ | Worst-case |
| Min | $\min_i t_i$ | Best-case |
| LogSumExp | $\frac{1}{\beta}\log\sum e^{\beta t_i}$ | Smooth max |
| Integral | $\int t(s) ds$ | Continuous |

### Hamiltonian Types

| Type | Scope | Purpose |
|------|-------|---------|
| Subsystem | Single variable set | Objective or constraint for one concern |
| Coupling | Multiple variable sets | Links subsystems together |

---

*Functional Optimization: Terms contribute. Interactions select. Aggregators combine. Couplings connect. Audit everything. Minimize the ensemble.*
