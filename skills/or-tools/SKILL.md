---
name: or-tools
description: "Use when the user asks about Google OR-Tools, CP-SAT, MIP/linear models, routing, scheduling, or hard vs soft constraint modeling in Python."
---

# OR-Tools AI Skill

This skill helps the AI assistant provide knowledgeable, concise, and accurate guidance when the user is working with **Google OR-Tools** (operations research / optimization) in this repository.

## When to Use This Skill

- The user is writing or debugging code that uses OR-Tools (CP-SAT, linear solver, routing, etc.).
- The user is designing optimization models (variables, constraints, objective functions) or needs advice on choosing solvers.
- The user asks for help installing, importing, or using the OR-Tools Python API.

## How to Respond (Guidelines)

- Favor concrete code examples in Python using `ortools` (e.g., `ortools.sat.python.cp_model`, `ortools.linear_solver`).
- Keep answers focused on the request; avoid unrelated optimization libraries unless explicitly asked.
- When suggesting improvements, keep changes minimal and safe for the user’s existing code.
- If the user asks about performance or scaling, mention modeling practices (e.g., reduce variable count, use implied constraints, choose appropriate solver).
- Prefer CP-SAT (`cp_model`) for mixed Boolean/integer logic and rich combinatorial constraints.
- Explain constraint intent before code when the model is non-trivial.
- Build constraints sequentially, one after the other, in clearly separated blocks.
- Keep each constraint block independent in structure, but allow all blocks to reuse the same shared variables and input data.
- Distinguish clearly:
    - Hard constraints: must always hold (`model.Add(...)`).
    - Soft constraints: may be violated with a penalty variable in the objective.

## Constraint Sequencing Pattern

Use this order when generating or refactoring models:

1. Create shared data and decision variables.
2. Add constraint block `C1`.
3. Add constraint block `C2`.
4. Continue with `C3`, `C4`, ... each in its own separated section/comment.
5. Add objective only after constraints are defined.
6. Solve and inspect status/results.

Each block should have:
- A clear name/comment.
- Exactly one modeling intent.
- Reuse of existing model data/variables instead of redefining them.

## Modeling Checklist

- Define decision variables with clear domains.
- Add hard feasibility constraints first.
- Add constraints in sequence and keep each constraint in its own separated block.
- Add soft constraints by introducing violation/slack variables.
- Build a weighted objective that reflects business priorities.
- Validate solver status (`OPTIMAL` or `FEASIBLE`) before reading values.
- For debugging infeasibility, temporarily remove soft penalties and test hard constraints incrementally.

## Examples

### Simple CP-SAT Example
```python
from ortools.sat.python import cp_model

model = cp_model.CpModel()
x = model.NewIntVar(0, 10, 'x')
y = model.NewIntVar(0, 10, 'y')
model.Add(x + y <= 10)
model.Maximize(x + 2 * y)

solver = cp_model.CpSolver()
status = solver.Solve(model)
if status == cp_model.OPTIMAL:
    print(solver.Value(x), solver.Value(y))
```

### Linear Solving Example
```python
from ortools.linear_solver import pywraplp

solver = pywraplp.Solver.CreateSolver('CBC')
x = solver.NumVar(0, 10, 'x')
y = solver.NumVar(0, 10, 'y')
solver.Add(x + y <= 10)
solver.Maximize(x + 2 * y)
result_status = solver.Solve()
```

### Hard Constraints Example (Must Hold)
```python
from ortools.sat.python import cp_model

model = cp_model.CpModel()

# Staff assignment over 7 days.
days = range(7)
alice = {d: model.NewBoolVar(f"alice_d{d}") for d in days}
bob = {d: model.NewBoolVar(f"bob_d{d}") for d in days}

# C1 (hard): exactly one worker per day.
for d in days:
    model.Add(alice[d] + bob[d] == 1)

# C2 (hard): Alice works at most 4 days.
model.Add(sum(alice[d] for d in days) <= 4)

# C3 (hard): Bob cannot work day 0.
model.Add(bob[0] == 0)

# Any objective works; this one just balances load toward Bob.
model.Maximize(sum(bob[d] for d in days))

solver = cp_model.CpSolver()
status = solver.Solve(model)
if status in (cp_model.OPTIMAL, cp_model.FEASIBLE):
    print("Alice:", [solver.Value(alice[d]) for d in days])
    print("Bob:", [solver.Value(bob[d]) for d in days])
```

### Soft Constraints Example (Penalized Violations)
```python
from ortools.sat.python import cp_model

model = cp_model.CpModel()

days = range(7)
alice = {d: model.NewBoolVar(f"alice_d{d}") for d in days}
bob = {d: model.NewBoolVar(f"bob_d{d}") for d in days}

# C1 (hard): exactly one worker per day.
for d in days:
    model.Add(alice[d] + bob[d] == 1)

# C2 (soft): Alice should work at most 3 days.
# Violation is allowed via alice_excess and penalized in objective.
alice_total = sum(alice[d] for d in days)
alice_excess = model.NewIntVar(0, 7, "alice_excess")
model.Add(alice_excess >= alice_total - 3)
model.Add(alice_excess >= 0)

# C3 (soft): avoid Bob on weekend (days 5, 6).
bob_weekend = model.NewIntVar(0, 2, "bob_weekend")
model.Add(bob_weekend == bob[5] + bob[6])

# Minimize weighted penalties. Lower is better.
model.Minimize(10 * alice_excess + 3 * bob_weekend)

solver = cp_model.CpSolver()
status = solver.Solve(model)
if status in (cp_model.OPTIMAL, cp_model.FEASIBLE):
    print("Objective:", solver.ObjectiveValue())
    print("alice_excess:", solver.Value(alice_excess))
    print("bob_weekend:", solver.Value(bob_weekend))
```
