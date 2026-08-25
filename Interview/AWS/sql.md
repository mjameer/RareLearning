# SQL Aggregates, GROUP BY & JOIN — Interview Cheat Sheet

---

## The Golden Rule

> **Any column in SELECT that is NOT inside an aggregate function MUST appear in GROUP BY.**

```sql
-- Headcount per department
SELECT d.dept_name,          -- ← not aggregated → must be in GROUP BY
       COUNT(e.id) AS headcount  -- ← aggregated → fine
FROM departments d
LEFT JOIN employees e ON d.id = e.dept_id
GROUP BY d.dept_name;
```

---

## Key Aggregates

| Function | What it does |
|---|---|
| `COUNT(col)` | Count non-NULL values |
| `COUNT(*)` | Count all rows including NULLs |
| `SUM(col)` | Total |
| `AVG(col)` | Average |
| `MAX(col)` / `MIN(col)` | Extremes |

---

## SQL Execution Order

SQL does **not** run top to bottom. Internalize this order:

```
FROM → JOIN → WHERE → GROUP BY → HAVING → SELECT → ORDER BY
```

Implication: you **cannot** use a SELECT alias in WHERE or HAVING — it doesn't exist yet.

---

## JOIN + GROUP BY + HAVING

### Pattern
```
"Find [entities] that have [aggregate condition] on [related data]"
         ↓                        ↓                    ↓
      SELECT                   HAVING               JOIN
```

### Example — Departments with more than 1 employee
```sql
SELECT d.dept_name, COUNT(e.id) AS headcount
FROM departments d
INNER JOIN employees e ON d.id = e.dept_id
GROUP BY d.dept_name
HAVING COUNT(e.id) > 1;
```

### Example — Customers with more than 3 orders
```sql
SELECT c.name, COUNT(o.id) AS order_count
FROM customers c
INNER JOIN orders o ON c.id = o.customer_id
GROUP BY c.name
HAVING COUNT(o.id) > 3;
```

### Example — Products with total sales above $10,000
```sql
SELECT p.name, SUM(o.amount) AS total_sales
FROM products p
INNER JOIN orders o ON p.id = o.product_id
GROUP BY p.name
HAVING SUM(o.amount) > 10000;
```

---

## WHERE vs HAVING

| | WHERE | HAVING |
|---|---|---|
| Filters | Raw rows (before grouping) | Aggregated groups (after grouping) |
| Can use aggregate? | ✗ No | ✓ Yes |
| Runs at step | After JOIN | After GROUP BY |

```sql
-- WRONG — headcount doesn't exist at WHERE stage
SELECT dept_name, COUNT(e.id) AS headcount
FROM departments d
LEFT JOIN employees e ON d.id = e.dept_id
WHERE COUNT(e.id) > 1        -- ✗ error
GROUP BY dept_name;

-- CORRECT
SELECT dept_name, COUNT(e.id) AS headcount
FROM departments d
LEFT JOIN employees e ON d.id = e.dept_id
GROUP BY dept_name
HAVING COUNT(e.id) > 1;      -- ✓
```

---

## Finding Duplicates

### Classic pattern — find duplicate values
```sql
SELECT email, COUNT(*) AS cnt
FROM users
GROUP BY email
HAVING COUNT(*) > 1;
```

### Get the full duplicate rows
```sql
SELECT *
FROM users
WHERE email IN (
    SELECT email
    FROM users
    GROUP BY email
    HAVING COUNT(*) > 1
);
```

### Delete duplicates, keep one
```sql
DELETE FROM users
WHERE id NOT IN (
    SELECT MIN(id)
    FROM users
    GROUP BY email
);
```

---

## COUNT(*) vs COUNT(col)

```sql
SELECT COUNT(*)     -- counts all rows including NULLs
SELECT COUNT(email) -- counts only non-NULL email values
```

For duplicate detection → use `COUNT(*)`.
For "how many have a value" → use `COUNT(col)`.

---

## Tricky SQL Interview Questions

---

### Q1 — What is the difference between DELETE, TRUNCATE, and DROP?

| Command | Removes | Rollback? | Resets identity? |
|---|---|---|---|
| `DELETE` | Rows (with WHERE filter) | ✓ Yes | ✗ No |
| `TRUNCATE` | All rows, fast | ✗ No (usually) | ✓ Yes |
| `DROP` | Entire table + structure | ✗ No | N/A |

```sql
DELETE FROM users WHERE id = 5;   -- removes one row, logged
TRUNCATE TABLE users;             -- wipes all rows instantly
DROP TABLE users;                 -- table gone entirely
```

---

### Q2 — Can you use WHERE and HAVING in the same query?

Yes. WHERE filters rows first, HAVING filters groups after.

```sql
SELECT dept_name, COUNT(e.id) AS headcount
FROM departments d
INNER JOIN employees e ON d.id = e.dept_id
WHERE e.salary > 50000          -- filter rows before grouping
GROUP BY dept_name
HAVING COUNT(e.id) > 1;         -- filter groups after
```

---

### Q3 — What does COUNT(*) vs COUNT(1) do?

Both count all rows — there is no performance difference in modern databases. `COUNT(*)` is standard and preferred.

```sql
SELECT COUNT(*)  FROM users;  -- standard
SELECT COUNT(1)  FROM users;  -- same result, older style
```

---

### Q4 — Second highest salary (no LIMIT/TOP)

```sql
SELECT MAX(salary)
FROM employees
WHERE salary < (SELECT MAX(salary) FROM employees);
```

With `LIMIT` + `OFFSET` (MySQL/Postgres):
```sql
SELECT DISTINCT salary
FROM employees
ORDER BY salary DESC
LIMIT 1 OFFSET 1;
```

#### What does OFFSET do here?

`OFFSET N` skips the first N rows from the result — the remaining rows are then handed to `LIMIT`.

**Trace with data:**

employees table:
| id | name  | salary |
|----|-------|--------|
| 1  | Alice | 100    |
| 2  | Bob   | 100    |
| 3  | Carol | 90     |
| 4  | Dave  | 75     |

**Step 1 — DISTINCT salary** (remove duplicate salaries):
```
100, 90, 75
```

**Step 2 — ORDER BY salary DESC** (sort highest to lowest):
```
Row 0 → 100   ← OFFSET 1 skips this
Row 1 → 90    ← LIMIT 1 picks this ✓
Row 2 → 75
```

**Step 3 — OFFSET 1** skips row 0 (the highest, 100).
**Step 4 — LIMIT 1** takes the next 1 row → **90**

Result: `90` — the second highest salary.

**Why DISTINCT matters:**
Without `DISTINCT`, Alice and Bob both have salary 100.
After `ORDER BY DESC` you'd get: 100, 100, 90, 75.
OFFSET 1 would skip only one 100 — and LIMIT 1 picks the second 100, not 90.
`DISTINCT` collapses duplicates first so OFFSET skips the value, not just one row.

**General formula:**
```sql
-- Nth highest salary
SELECT DISTINCT salary
FROM employees
ORDER BY salary DESC
LIMIT 1 OFFSET N-1;

-- 3rd highest → OFFSET 2
-- 5th highest → OFFSET 4
```

---

### Q5 — Find employees who earn more than their manager (self-join)

```sql
SELECT e.name AS employee, mgr.name AS manager
FROM employees e
JOIN employees mgr ON e.manager_id = mgr.id
WHERE e.salary > mgr.salary;
```

Self-join = same table aliased twice. Classic recursive relationship pattern.

---

### Q6 — What is the difference between UNION and UNION ALL?

| | UNION | UNION ALL |
|---|---|---|
| Duplicates | Removed | Kept |
| Speed | Slower (dedup step) | Faster |

```sql
SELECT name FROM employees_us
UNION
SELECT name FROM employees_uk;      -- deduplicates

SELECT name FROM employees_us
UNION ALL
SELECT name FROM employees_uk;      -- keeps all rows
```

---

### Q7 — NULL trap: what does this return?

```sql
SELECT * FROM employees WHERE dept_id = NULL;
```

**Returns nothing.** NULL comparisons with `=` always evaluate to UNKNOWN.

Correct way:
```sql
SELECT * FROM employees WHERE dept_id IS NULL;
```

---

### Q8 — Rows present in one table but not another (anti-join)

```sql
-- Employees with no department
SELECT e.name
FROM employees e
LEFT JOIN departments d ON e.dept_id = d.id
WHERE d.id IS NULL;
```

Alternative with NOT EXISTS:
```sql
SELECT e.name
FROM employees e
WHERE NOT EXISTS (
    SELECT 1 FROM departments d WHERE d.id = e.dept_id
);
```

---

### Q9 — Running total (window function)

```sql
SELECT name, salary,
       SUM(salary) OVER (ORDER BY id) AS running_total
FROM employees;
```

#### The problem with GROUP BY for this

If you use `SUM` with `GROUP BY`, it collapses all rows into one total — you lose individual rows.
A running total needs **every row kept**, but each row shows the cumulative sum up to that point.
That's exactly what a **window function** solves.

#### What OVER does

`OVER` tells SQL: *"don't collapse rows — instead, for each row, compute this aggregate across a sliding window of rows."*

```
SUM(salary) OVER (ORDER BY id)
                  ↑
                  defines the window: "all rows from the start up to the current row, ordered by id"
```

No `GROUP BY` needed. All rows stay. Each row gets its own computed value.

#### Full trace with data

employees table:
| id | name  | salary |
|----|-------|--------|
| 1  | Alice | 100    |
| 2  | Bob   | 150    |
| 3  | Carol | 200    |
| 4  | Dave  | 50     |

Window for each row (ordered by id, cumulative from top):
```
Row id=1: window = [100]            → SUM = 100
Row id=2: window = [100, 150]       → SUM = 250
Row id=3: window = [100, 150, 200]  → SUM = 450
Row id=4: window = [100, 150, 200, 50] → SUM = 500
```

Result:
| name  | salary | running_total |
|-------|--------|---------------|
| Alice | 100    | 100           |
| Bob   | 150    | 250           |
| Carol | 200    | 450           |
| Dave  | 50     | 500           |

#### Regular SUM vs Window SUM — side by side

```sql
-- Regular SUM with GROUP BY → collapses to 1 row
SELECT SUM(salary) AS total FROM employees;
-- Result: 500  (one row, all individual rows gone)

-- Window SUM → all rows kept, running total per row
SELECT name, salary,
       SUM(salary) OVER (ORDER BY id) AS running_total
FROM employees;
-- Result: 4 rows, each with its own cumulative total
```

#### Partitioned running total (per department)

```sql
SELECT name, dept_id, salary,
       SUM(salary) OVER (PARTITION BY dept_id ORDER BY id) AS dept_running_total
FROM employees;
```

`PARTITION BY dept_id` resets the running total for each department — like running totals per group, without collapsing rows.

#### Key mental model

```
GROUP BY  → collapse rows into groups, one output row per group
OVER      → keep all rows, compute across a sliding window per row
```

Any aggregate (`SUM`, `AVG`, `COUNT`, `MAX`, `MIN`) can become a window function by adding `OVER (...)`.

---

### Q10 — RANK vs DENSE_RANK vs ROW_NUMBER

```sql
SELECT name, salary,
    ROW_NUMBER()  OVER (ORDER BY salary DESC) AS row_num,
    RANK()        OVER (ORDER BY salary DESC) AS rnk,
    DENSE_RANK()  OVER (ORDER BY salary DESC) AS dense_rnk
FROM employees;
```

Given salaries: 100, 100, 90:

| name | salary | ROW_NUMBER | RANK | DENSE_RANK |
|---|---|---|---|---|
| Alice | 100 | 1 | 1 | 1 |
| Bob | 100 | 2 | 1 | 1 |
| Carol | 90 | 3 | 3 | 2 |

- `ROW_NUMBER` — always unique, no ties
- `RANK` — ties get same rank, next rank skips (1,1,3)
- `DENSE_RANK` — ties get same rank, no skip (1,1,2)

---

## Quick Reference Card

```
Duplicate values        → GROUP BY col HAVING COUNT(*) > 1
Full duplicate rows     → WHERE col IN (subquery above)
Anti-join               → LEFT JOIN + WHERE right.id IS NULL
Self-join               → same table aliased twice
Second highest          → MAX(salary) WHERE salary < MAX(salary)
Filter before GROUP BY  → WHERE
Filter after GROUP BY   → HAVING
Running total           → SUM() OVER (ORDER BY ...)
Ranking with ties       → RANK() or DENSE_RANK()
```
