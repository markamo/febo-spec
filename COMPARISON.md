# FEBO vs Traditional Optimization Formats: A Side-by-Side Comparison

This document compares how the same optimization problems are formulated in FEBO versus traditional frameworks.

---

## Problem 1: Portfolio Optimization

**Mathematical formulation:**
```
maximize:   μᵀw                    (expected return)
minimize:   wᵀΣw                   (variance)
subject to: Σᵢ wᵢ = 1              (fully invested)
            wᵢ ≥ 0                 (long only)
```

---

### FEBO

```yaml
febo_version: "0.1"
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
  H_risk:   {components: [{use: total_risk, alpha: 1}]}
  H_budget: {components: [{use: budget_pen, alpha: 1}]}

ensemble:
  - {use: H_return, weight: 1}
  - {use: H_risk, weight: risk_aversion}
  - {use: H_budget, weight: 10000}
```

**Lines: 32** | **Readable: ✅** | **Auditable: ✅** | **Scalable: ✅**

---

### Pyomo

```python
from pyomo.environ import *
import numpy as np

# Load data
mu = np.loadtxt("returns.csv")
Sigma = np.loadtxt("covariance.csv", delimiter=",")
n = len(mu)
risk_aversion = 0.5

# Create model
model = ConcreteModel()

# Variables
model.w = Var(range(n), bounds=(0, 1))

# Objective: maximize return - risk_aversion * variance
def objective_rule(m):
    return_term = sum(mu[i] * m.w[i] for i in range(n))
    risk_term = sum(Sigma[i,j] * m.w[i] * m.w[j] 
                    for i in range(n) for j in range(n))
    return return_term - risk_aversion * risk_term

model.obj = Objective(rule=objective_rule, sense=maximize)

# Constraint: fully invested
def budget_rule(m):
    return sum(m.w[i] for i in range(n)) == 1

model.budget = Constraint(rule=budget_rule)

# Solve
solver = SolverFactory('ipopt')
results = solver.solve(model)

# Extract solution
weights = [model.w[i].value for i in range(n)]
```

**Lines: 32** | **Readable: ⚠️** | **Auditable: ❌** | **Scalable: ⚠️**

---

### AMPL

```ampl
# portfolio.mod

param n;
param mu{1..n};
param Sigma{1..n, 1..n};
param risk_aversion default 0.5;

var w{1..n} >= 0, <= 1;

maximize objective:
    sum{i in 1..n} mu[i] * w[i] 
    - risk_aversion * sum{i in 1..n, j in 1..n} Sigma[i,j] * w[i] * w[j];

subject to budget:
    sum{i in 1..n} w[i] = 1;
```

```ampl
# portfolio.dat
param n := 100;
param mu := include returns.csv;
param Sigma := include covariance.csv;
```

**Lines: 15** | **Readable: ✅** | **Auditable: ❌** | **Scalable: ⚠️**

---

### Gurobi Python

```python
import gurobipy as gp
from gurobipy import GRB
import numpy as np

# Load data
mu = np.loadtxt("returns.csv")
Sigma = np.loadtxt("covariance.csv", delimiter=",")
n = len(mu)
risk_aversion = 0.5

# Create model
model = gp.Model("portfolio")

# Variables
w = model.addVars(n, lb=0, ub=1, name="w")

# Objective
return_expr = gp.quicksum(mu[i] * w[i] for i in range(n))
risk_expr = gp.quicksum(Sigma[i,j] * w[i] * w[j] 
                        for i in range(n) for j in range(n))
model.setObjective(return_expr - risk_aversion * risk_expr, GRB.MAXIMIZE)

# Constraint
model.addConstr(gp.quicksum(w[i] for i in range(n)) == 1, "budget")

# Solve
model.optimize()

# Extract solution
weights = [w[i].X for i in range(n)]
```

**Lines: 27** | **Readable: ⚠️** | **Auditable: ❌** | **Scalable: ✅**

---

### CVXPY

```python
import cvxpy as cp
import numpy as np

# Load data
mu = np.loadtxt("returns.csv")
Sigma = np.loadtxt("covariance.csv", delimiter=",")
n = len(mu)
risk_aversion = 0.5

# Variables
w = cp.Variable(n)

# Objective
return_term = mu @ w
risk_term = cp.quad_form(w, Sigma)
objective = cp.Maximize(return_term - risk_aversion * risk_term)

# Constraints
constraints = [
    cp.sum(w) == 1,
    w >= 0,
    w <= 1
]

# Solve
problem = cp.Problem(objective, constraints)
problem.solve()

# Extract solution
weights = w.value
```

**Lines: 26** | **Readable: ✅** | **Auditable: ❌** | **Scalable: ⚠️** (convex only)

---

### MPS Format

```
NAME          PORTFOLIO
ROWS
 N  OBJ
 E  BUDGET
COLUMNS
    W0001     OBJ       -0.05234    BUDGET    1.0
    W0002     OBJ       -0.04123    BUDGET    1.0
    W0003     OBJ       -0.06234    BUDGET    1.0
    ... (repeat for all n variables)
RHS
    RHS1      BUDGET    1.0
BOUNDS
 UP BND1      W0001     1.0
 UP BND1      W0002     1.0
    ... (repeat for all n variables)
QUADOBJ
    W0001     W0001     0.00234
    W0001     W0002     0.00045
    W0002     W0002     0.00312
    ... (n² entries for covariance)
ENDATA
```

**Lines: ~n² + 4n** | **Readable: ❌** | **Auditable: ❌** | **Scalable: ✅**

---

### Portfolio Comparison Summary

| Aspect | FEBO | Pyomo | AMPL | Gurobi | CVXPY | MPS |
|--------|------|-------|------|--------|-------|-----|
| **Lines of code** | 32 | 32 | 15 | 27 | 26 | ~n² |
| **Human readable** | ✅ | ⚠️ | ✅ | ⚠️ | ✅ | ❌ |
| **Structure explicit** | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| **Multi-objective clear** | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| **Auditability built-in** | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| **Parameterized** | ✅ | ⚠️ | ✅ | ⚠️ | ⚠️ | ❌ |
| **Solver agnostic** | ✅ | ✅ | ✅ | ❌ | ✅ | ✅ |

---

## Problem 2: Griewank Function (Nonlinear, Non-convex)

**Mathematical formulation:**
```
minimize: f(x) = 1 + (1/4000) Σᵢ xᵢ² - Πᵢ cos(xᵢ/√i)

where x ∈ [-600, 600]ⁿ
```

This problem has:
- **Sum** aggregator (quadratic term)
- **Product** aggregator (cosine term)
- Highly non-convex (many local minima)

---

### FEBO

```yaml
febo_version: "0.1"
name: griewank

parameters:
  n: {type: int, default: 1000}

variables:
  x: {type: continuous, shape: [n], bounds: [-600, 600]}

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
  bowl:        {components: [{use: quadratic, alpha: 1/4000}]}
  oscillation: {components: [{use: oscill, alpha: -1}]}
  constant:    {components: [{use: offset, alpha: 1}]}

ensemble:
  - {use: bowl, weight: 1}
  - {use: oscillation, weight: 1}
  - {use: constant, weight: 1}
```

**Lines: 28** | **Prod aggregator: ✅ Native** | **Structure: ✅ Explicit**

---

### Pyomo

```python
from pyomo.environ import *
import math

n = 1000

model = ConcreteModel()
model.I = RangeSet(1, n)
model.x = Var(model.I, bounds=(-600, 600))

def griewank_rule(m):
    sum_term = sum(m.x[i]**2 for i in m.I) / 4000
    # PROBLEM: Pyomo doesn't have native product aggregator
    # Must use log transform: prod(cos) = exp(sum(log(cos)))
    prod_term = exp(sum(log(cos(m.x[i] / math.sqrt(i))) for i in m.I))
    return 1 + sum_term - prod_term

model.obj = Objective(rule=griewank_rule, sense=minimize)

solver = SolverFactory('ipopt')
results = solver.solve(model)
```

**Lines: 17** | **Prod aggregator: ❌ Requires log transform** | **Structure: ❌ Implicit**

**Problem:** The log transform `exp(sum(log(cos(...))))` introduces numerical issues when `cos(x) ≤ 0`.

---

### Knitro (Python)

```python
from knitro import *
import math

n = 1000

# Callback for objective
def eval_fc(kc, cb, evalRequest, evalResult, userParams):
    x = evalRequest.x
    
    sum_term = sum(x[i]**2 for i in range(n)) / 4000
    
    prod_term = 1.0
    for i in range(n):
        prod_term *= math.cos(x[i] / math.sqrt(i + 1))
    
    evalResult.obj = 1 + sum_term - prod_term
    return 0

# Create problem
kc = KN_new()
KN_add_vars(kc, n)

for i in range(n):
    KN_set_var_lobnds(kc, i, -600)
    KN_set_var_upbnds(kc, i, 600)

# Set callback
cb = KN_add_eval_callback(kc, True, None, eval_fc)

# Solve
KN_solve(kc)
x = KN_get_var_primal_values_all(kc)
KN_free(kc)
```

**Lines: 30** | **Prod aggregator: ✅ Manual loop** | **Structure: ❌ Callback-based**

---

### AMPL

```ampl
param n := 1000;
var x{1..n} >= -600, <= 600;

minimize griewank:
    1 + (1/4000) * sum{i in 1..n} x[i]^2 
    - prod{i in 1..n} cos(x[i] / sqrt(i));
```

**Lines: 5** | **Prod aggregator: ✅ Native** | **Structure: ❌ Flat**

AMPL is concise but:
- No explicit structure (what's the bowl vs oscillation?)
- No auditability (why did it fail at a local minimum?)
- No parameterization for scale testing

---

### Griewank Comparison Summary

| Aspect | FEBO | Pyomo | Knitro | AMPL |
|--------|------|-------|--------|------|
| **Prod aggregator** | ✅ Native | ❌ Log transform | ⚠️ Manual | ✅ Native |
| **Structure visible** | ✅ bowl + oscillation | ❌ | ❌ | ❌ |
| **Auditable** | ✅ H_bowl, H_oscillation | ❌ | ❌ | ❌ |
| **Numerical stability** | ✅ | ❌ (log of negative) | ✅ | ✅ |
| **Scale to 2B vars** | ✅ | ❌ | ❌ | ❌ |

---

## Problem 3: Robust Optimization (Minimax)

**Mathematical formulation:**
```
minimize: max_{s ∈ Scenarios} f(x, s)
```

This requires a **Max** aggregator over scenarios.

---

### FEBO

```yaml
febo_version: "0.1"
name: robust

parameters:
  n: {type: int, default: 100}
  S: {type: int, default: 1000}

variables:
  x: {type: continuous, shape: [n], bounds: [-10, 10]}

data:
  scenarios: {source: "scenarios.csv", shape: [S, n]}

interactions:
  all_s: {type: all_indices, range: [1, S]}

terms:
  loss: {type: analytic, arity: n-ary, expr: "sum_i((x[i] - scenarios[s,i])^2)"}

components:
  worst_case: {term: loss, interaction: all_s, agg: max}

hamiltonians:
  H_robust: {components: [{use: worst_case, alpha: 1}]}

ensemble:
  - {use: H_robust, weight: 1}
```

**Lines: 22** | **Max aggregator: ✅ Native** | **Clear worst-case structure: ✅**

---

### Pyomo (Epigraph Reformulation)

```python
from pyomo.environ import *
import numpy as np

scenarios = np.loadtxt("scenarios.csv", delimiter=",")
S, n = scenarios.shape

model = ConcreteModel()
model.x = Var(range(n), bounds=(-10, 10))
model.t = Var()  # epigraph variable

# Objective: minimize the epigraph variable
model.obj = Objective(expr=model.t, sense=minimize)

# Constraints: t >= f(x, s) for each scenario
def scenario_constraint(m, s):
    return m.t >= sum((m.x[i] - scenarios[s, i])**2 for i in range(n))

model.scenario_cons = Constraint(range(S), rule=scenario_constraint)

solver = SolverFactory('ipopt')
results = solver.solve(model)
```

**Lines: 20** | **Max aggregator: ❌ Requires epigraph reformulation** | **Extra variable: t**

---

### Gurobi (Epigraph)

```python
import gurobipy as gp
from gurobipy import GRB
import numpy as np

scenarios = np.loadtxt("scenarios.csv", delimiter=",")
S, n = scenarios.shape

model = gp.Model("robust")

x = model.addVars(n, lb=-10, ub=10, name="x")
t = model.addVar(name="t")  # epigraph

model.setObjective(t, GRB.MINIMIZE)

# t >= loss(s) for each scenario
for s in range(S):
    loss = gp.quicksum((x[i] - scenarios[s, i])**2 for i in range(n))
    model.addConstr(t >= loss, f"scenario_{s}")

model.optimize()
```

**Lines: 19** | **Max aggregator: ❌ Epigraph + S constraints** | **Constraint explosion: ⚠️**

---

### AMPL (Epigraph)

```ampl
param n;
param S;
param scenarios{1..S, 1..n};

var x{1..n} >= -10, <= 10;
var t;

minimize worst_case: t;

subject to scenario_bound{s in 1..S}:
    t >= sum{i in 1..n} (x[i] - scenarios[s,i])^2;
```

**Lines: 10** | **Max aggregator: ❌ Requires reformulation** | **S extra constraints**

---

### Robust Comparison Summary

| Aspect | FEBO | Pyomo | Gurobi | AMPL |
|--------|------|-------|--------|------|
| **Max aggregator** | ✅ Native | ❌ Epigraph | ❌ Epigraph | ❌ Epigraph |
| **Extra variables** | 0 | 1 | 1 | 1 |
| **Extra constraints** | 0 | S | S | S |
| **Scales to 1M scenarios** | ✅ | ❌ | ❌ | ❌ |
| **Structure clear** | ✅ worst_case | ❌ | ❌ | ❌ |

---

## Problem 4: Supply Chain (Multi-System Coupling)

**Multiple subsystems that must be optimized jointly:**
- Production system
- Distribution system  
- Interface coupling

---

### FEBO

```yaml
febo_version: "0.1"
name: supply_chain

parameters:
  plants:    {type: int}
  customers: {type: int}
  time:      {type: int, default: 52}

variables:
  production: {type: continuous, shape: [plants, time], bounds: [0, 1000]}
  shipment:   {type: continuous, shape: [plants, customers, time], bounds: [0, "inf"]}

hamiltonians:
  H_production:
    components:
      - {use: c_prod_cost, alpha: 1}
      
  H_distribution:
    components:
      - {use: c_ship_cost, alpha: 1}
      - {use: c_stockout, alpha: 100}
      
  H_coupling:
    couples: [production, shipment]
    components:
      - {use: c_interface, alpha: 1}

ensemble:
  - {use: H_production, weight: 1}
  - {use: H_distribution, weight: 1}
  - {use: H_coupling, weight: 10000}
```

**Structure: ✅ Explicit subsystems + coupling** | **Auditability: ✅ Which system failed?**

---

### Pyomo

```python
model = ConcreteModel()

# Variables
model.production = Var(plants, time, bounds=(0, 1000))
model.shipment = Var(plants, customers, time, bounds=(0, None))

# Single monolithic objective
def objective(m):
    prod_cost = sum(cost[p] * m.production[p,t] for p in plants for t in time)
    ship_cost = sum(dist[p,c] * m.shipment[p,c,t] for p in plants for c in customers for t in time)
    stockout = sum(max(0, demand[c,t] - sum(m.shipment[p,c,t] for p in plants))**2 
                   for c in customers for t in time)
    interface = sum((sum(m.shipment[p,c,t] for c in customers) - m.production[p,t])**2
                    for p in plants for t in time)
    return prod_cost + ship_cost + 100*stockout + 10000*interface

model.obj = Objective(rule=objective)
```

**Structure: ❌ Monolithic blob** | **Auditability: ❌ Which term caused failure?**

---

### Supply Chain Comparison

| Aspect | FEBO | Pyomo/Gurobi/AMPL |
|--------|------|-------------------|
| **Subsystem structure** | ✅ Explicit H_production, H_distribution | ❌ Mixed in one objective |
| **Coupling explicit** | ✅ H_coupling with `couples:` | ❌ Just another term |
| **Diagnosis** | ✅ "H_coupling violated" | ❌ "Infeasible" |
| **Reusable subsystems** | ✅ Compose Hamiltonians | ❌ Rewrite from scratch |

---

## Overall Comparison Matrix

| Feature | FEBO | Pyomo | AMPL | Gurobi | CVXPY | MPS |
|---------|------|-------|------|--------|-------|-----|
| **Declarative** | ✅ | ⚠️ | ✅ | ❌ | ✅ | ❌ |
| **Human readable** | ✅ | ⚠️ | ✅ | ⚠️ | ✅ | ❌ |
| **Sum aggregator** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| **Prod aggregator** | ✅ | ❌ | ✅ | ❌ | ❌ | ❌ |
| **Max aggregator** | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| **Multi-objective** | ✅ Native | ⚠️ Manual | ⚠️ Manual | ⚠️ Manual | ⚠️ | ❌ |
| **Subsystem structure** | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| **Coupling explicit** | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| **Auditability** | ✅ Built-in | ❌ | ❌ | ❌ | ❌ | ❌ |
| **Parameterized** | ✅ | ⚠️ | ✅ | ⚠️ | ⚠️ | ❌ |
| **Solver agnostic** | ✅ | ✅ | ✅ | ❌ | ✅ | ✅ |
| **Nonlinear** | ✅ | ✅ | ✅ | ⚠️ | ❌ | ❌ |
| **Non-convex** | ✅ | ✅ | ✅ | ⚠️ | ❌ | ❌ |
| **Billion-scale** | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |

---

## Key Differentiators

### 1. Aggregators Beyond Sum

| Problem Type | Traditional | FEBO |
|--------------|-------------|------|
| Worst-case (minimax) | Epigraph reformulation + S constraints | `agg: max` |
| Multiplicative coupling | Log transform (numerically unstable) | `agg: prod` |
| CVaR (tail risk) | Auxiliary variables + constraints | `agg: {type: cvar, alpha: 0.95}` |

### 2. Explicit Structure

**Traditional**: One flat objective function
```python
obj = cost1 + cost2 + penalty1 + penalty2 + coupling
```

**FEBO**: Hierarchical, named components
```yaml
hamiltonians:
  H_cost: ...
  H_penalty: ...
  H_coupling: ...
```

### 3. Auditability

**Traditional**: "Solver returned infeasible"

**FEBO**:
```
H_total = 847.3 ✗
├── H_production: 12.3 ✓
├── H_distribution: 5.2 ✓
└── H_coupling: 829.8 ✗ ← Interface violated
    └── Plant P-3 → Customer C-15: mismatch of 230 units
```

### 4. Scale

| Framework | Practical Limit | FEBO + Xtellix |
|-----------|-----------------|----------------|
| Pyomo | ~100K variables | - |
| Gurobi | ~10M variables (LP), ~100K (MIQP) | - |
| AMPL | ~1M variables | - |
| **FEBO** | - | **2B+ variables** |

---

## When to Use What

| Use Case | Best Choice |
|----------|-------------|
| Quick LP/QP prototype | CVXPY |
| Commercial MIP | Gurobi |
| Academic optimization | Pyomo |
| Algebraic modeling | AMPL |
| **Large-scale, non-convex, auditable** | **FEBO** |
| **Multi-system with coupling** | **FEBO** |
| **Worst-case / robust** | **FEBO** |
| **Production at scale** | **FEBO** |

---

## Conclusion

FEBO is not "better" at everything—for a quick 100-variable LP, CVXPY is simpler.

**FEBO excels when:**

1. **Aggregators matter**: Max, Prod, CVaR are native, not reformulated
2. **Structure matters**: Subsystems and coupling are explicit
3. **Auditability matters**: Know why the solution is what it is
4. **Scale matters**: Billions of variables, not thousands
5. **Reuse matters**: Compose Hamiltonians, don't rewrite

**The analogy holds:**

| Domain | Simple Tool | Scaling Tool |
|--------|-------------|--------------|
| Web | HTML | React |
| Data | CSV | Parquet |
| ML | Scikit-learn | PyTorch |
| **Optimization** | **Pyomo/CVXPY** | **FEBO** |
