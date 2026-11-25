# Factory Staff Assignment

A small mixed-integer linear programming (MILP) model, solved with **Gurobi** in Python, that builds a weekly roster for a factory running **3 shifts a day, 7 days a week** with a pool of **20 workers**. The model assigns workers to day–shift slots so that every shift's demand is met exactly while respecting basic labor rules.

## Repository contents

| File | Description |
| --- | --- |
| [staff_assignment.ipynb](staff_assignment.ipynb) | Notebook with input data, model, solve, and CSV export |
| [staff_assignment.csv](staff_assignment.csv) | Solved roster: one row per worker per day (`Staff, Day, Shift, Status`) |

## 1. Problem

Shift demand (number of workers needed):

| Day | Morning | Afternoon | Evening | Total |
| --- | :---: | :---: | :---: | :---: |
| Monday | 6 | 4 | 3 | 13 |
| Tuesday | 5 | 5 | 5 | 15 |
| Wednesday | 6 | 4 | 4 | 14 |
| Thursday | 5 | 5 | 5 | 15 |
| Friday | 5 | 5 | 3 | 13 |
| Saturday | 6 | 5 | 4 | 15 |
| Sunday | 5 | 5 | 4 | 14 |
| **Week** | | | | **99** |

Labor rules for each worker:

- At most one shift per day.
- At least 2 days off per week (i.e. at most 5 working days).
- No evening shift on day $j$ followed by a morning shift on day $j+1$.

## 2. Mathematical formulation

**Sets**

- Workers $I = \{1,\dots,20\}$, days $J = \{1,\dots,7\}$, shifts $K = \{1,2,3\}$ (morning, afternoon, evening).

**Parameters**

- $d_{jk}$: required number of workers on day $j$, shift $k$.

**Decision variables**

- $x_{ijk} \in \{0,1\}$: 1 if worker $i$ works day $j$, shift $k$.

**Objective**

$$\min \sum_{i \in I} \sum_{j \in J} \sum_{k \in K} x_{ijk}$$

**Constraints**

| Rule | Constraint |
| --- | --- |
| Demand met exactly | $\sum_{i \in I} x_{ijk} = d_{jk} \quad \forall j, k$ |
| One shift per day | $\sum_{k \in K} x_{ijk} \le 1 \quad \forall i, j$ |
| ≤ 5 working days per week | $\sum_{j \in J}\sum_{k \in K} x_{ijk} \le 5 \quad \forall i$ |
| No evening → next morning | $x_{ij3} + x_{i,j+1,1} \le 1 \quad \forall i,\ j = 1,\dots,6$ |

Because each worker works at most one shift per day, the weekly limit on assignments is equivalent to "at most 5 working days", i.e. at least 2 days off.

## 3. Implementation

The notebook follows four steps:

1. **Input data** – sizes `I = 20`, `J = 7`, `K = 3` and the `demand[j][k]` matrix.
2. **Model** – `x = model.addVars(I, J, K, vtype=GRB.BINARY)`, the objective, and the four constraint families above (indices are 0-based in code, so evening = `2`, morning = `0`).
3. **Solve** – `model.optimize()`.
4. **Export** – if optimal, every (worker, day) pair is written to `staff_assignment.csv` with either the assigned shift or `N/A / Leave`.

## 4. Results

- Model size: 420 binary variables, 301 constraints.
- Gurobi finds an optimal solution at the root (gap 0%) in a fraction of a second.
- Total assignments: **99**. Worker 1 works 4 days; the other 19 workers work 5 days each.

## 5. Notes on the model

- **The objective is effectively constant.** The demand constraints are equalities, so the sum of all $x_{ijk}$ is always $\sum d_{jk} = 99$. Every feasible roster is "optimal", and the model is really a feasibility problem. To make the optimization meaningful, you could e.g. change demand to $\ge d_{jk}$ (allowing overstaffing) and minimize total assignments, or add costs/preferences such as penalizing evening shifts, balancing workload, or rewarding consecutive days off.
- **Capacity is tight.** 20 workers × 5 days = 100 possible assignments against a demand of 99, so there is only one spare slot. Higher demand or fewer workers quickly makes the model infeasible.
- **Single-week horizon.** The rest rule does not wrap from Sunday evening to Monday morning. For a repeating weekly roster, add $x_{i,7,3} + x_{i,1,1} \le 1$.
- **No fairness rules.** The model does not control how evening shifts or weekends are spread among workers, or whether days off are consecutive.

## 6. Running

```bash
pip install gurobipy pandas
```

Open `staff_assignment.ipynb` (originally run on Google Colab) and run all cells. The pip-installed Gurobi comes with a restricted size-limited license, which is enough for this model.
