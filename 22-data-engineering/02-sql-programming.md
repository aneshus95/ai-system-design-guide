# SQL Programming

SQL is the single most-tested practical skill in Data Science and Data Engineering interviews — you will be asked to *write* queries live. This page teaches SQL the way that actually makes it click: not as syntax to memorize, but as a **declarative language with a fixed execution order**. Once you understand *the order the database runs your clauses in*, everything else — why `WHERE` can't use aggregates, why window functions can't go in `WHERE`, why `GROUP BY` collapses rows — stops being arbitrary rules and becomes obvious.

## Table of Contents

- [What SQL Is (Declarative)](#what-sql-is-declarative)
- [The Key Intuition: Query Execution Order](#the-key-intuition-query-execution-order)
- [SELECT, WHERE, ORDER BY, LIMIT](#select-where-order-by-limit)
- [Aggregations, GROUP BY & HAVING](#aggregations-group-by--having)
- [JOINs](#joins)
- [Subqueries](#subqueries)
- [CTEs (WITH)](#ctes-with)
- [Window Functions (the Big One)](#window-functions-the-big-one)
- [Set Operations](#set-operations)
- [CASE & NULL Handling](#case--null-handling)
- [Indexes & Performance](#indexes--performance)
- [Classic Interview Patterns](#classic-interview-patterns)
- [Interview Questions](#interview-questions)
- [Glossary](#glossary)
- [References](#references)

---

## What SQL Is (Declarative)

SQL is **declarative**: you describe *what* you want, not *how* to get it. You write "give me total sales per region, only regions over $1M, sorted high to low," and the database's **query planner** figures out the actual steps (which index to use, join order, etc.). Contrast with Python, where you write the loop yourself.

That's the mental shift: **you specify the result; the engine decides the algorithm.** Which is exactly why understanding the *logical execution order* matters more than any single keyword.

---

## The Key Intuition: Query Execution Order

You *write* a query in this order:
```sql
SELECT ... FROM ... WHERE ... GROUP BY ... HAVING ... ORDER BY ... LIMIT ...
```
But the database *runs* it in a **different** order — and this single fact explains almost every "why doesn't this work?" in SQL:

```
 1. FROM / JOIN   → grab the tables and combine them          (get the raw rows)
 2. WHERE         → filter individual rows                     (before grouping!)
 3. GROUP BY      → collapse rows into groups
 4. HAVING        → filter the groups                          (after grouping!)
 5. SELECT        → pick/compute the output columns
    (window functions run here)
 6. ORDER BY      → sort the result
 7. LIMIT         → keep the top N
```

**Everything falls out of this order:**
- **Why can't `WHERE` use `COUNT(*)`?** Because `WHERE` (step 2) runs *before* `GROUP BY` (step 3) — the groups don't exist yet. Use `HAVING` (step 4) for that.
- **Why can't you use a `SELECT` alias in `WHERE`?** Because `SELECT` (step 5) runs *after* `WHERE`. The alias doesn't exist yet.
- **But you *can* use a `SELECT` alias in `ORDER BY`** — because `ORDER BY` (step 6) runs *after* `SELECT`.
- **Why can't a window function go in `WHERE`?** Window functions run in step 5 (`SELECT`), after filtering — so to filter on one, you wrap the query in a CTE/subquery and filter on the *outer* level.

> **Memorize this order and SQL stops being a bag of exceptions.** `WHERE` filters rows *before* grouping; `HAVING` filters groups *after*.

---

## SELECT, WHERE, ORDER BY, LIMIT

The everyday query skeleton:

```sql
SELECT name, salary                 -- which columns to return
FROM   employees                    -- from which table
WHERE  department = 'Sales'         -- keep only matching rows
  AND  salary > 50000
ORDER BY salary DESC                -- sort (DESC = high→low)
LIMIT 10;                           -- top 10 only
```

- **`WHERE`** filters individual rows with conditions (`=`, `<>`, `>`, `IN`, `BETWEEN`, `LIKE 'A%'` for pattern match, `IS NULL`).
- **`DISTINCT`** removes duplicate rows: `SELECT DISTINCT department FROM employees`.
- **`ORDER BY col DESC, col2 ASC`** — sort by multiple keys.
- **`LIMIT n` / `OFFSET`** — pagination (top-N, skip-N).

---

## Aggregations, GROUP BY & HAVING

**Aggregate functions** collapse many rows into one number: `COUNT`, `SUM`, `AVG`, `MIN`, `MAX`.

**`GROUP BY`** splits rows into buckets and applies the aggregate **per bucket**:

```sql
SELECT   department, COUNT(*) AS headcount, AVG(salary) AS avg_salary
FROM     employees
WHERE    active = true          -- (row filter, BEFORE grouping)
GROUP BY department
HAVING   COUNT(*) > 5           -- (group filter, AFTER grouping)
ORDER BY avg_salary DESC;
```

**The interview classic — `WHERE` vs `HAVING`:**
- **`WHERE`** filters **rows** *before* they're grouped (can't see aggregates).
- **`HAVING`** filters **groups** *after* aggregation (can use `COUNT`, `SUM`, etc.).

> Rule: every column in `SELECT` must either be **in the `GROUP BY`** or **inside an aggregate**. (You can't select a raw column that isn't grouped — which row's value would it pick?)

---

## JOINs

A JOIN combines rows from two tables on a matching key. The intuition is **set overlap** — which rows to keep when the key does/doesn't match:

```
 INNER JOIN         LEFT JOIN            FULL OUTER JOIN
  A ∩ B              all A + matches      everything
  ( only rows        of B (NULLs where    (A and B, NULLs
   that match )       no match )           where no match )

   A   B              A   B                 A   B
  [■■■]              [■■■■]                [■■■■■]
   only               all-of-A              all of both
   overlap            + overlap
```

| Join | Keeps | Use when |
|---|---|---|
| **INNER** | only rows matching in **both** tables | you need a match on both sides |
| **LEFT** | **all** left rows + matches from right (NULL if none) | "all customers, even those with no orders" |
| **RIGHT** | all right rows + matches from left | rarely used (just flip to LEFT) |
| **FULL OUTER** | all rows from both, NULLs where no match | reconcile two sources |
| **CROSS** | every combination (Cartesian product) | generate all pairs (careful — explodes) |
| **SELF** | a table joined to itself | hierarchies (employee → manager) |

```sql
SELECT c.name, o.order_id, o.amount
FROM   customers c
LEFT JOIN orders o ON o.customer_id = c.id     -- all customers, even orderless ones
WHERE  o.amount > 100 OR o.amount IS NULL;
```

> **Gotcha:** putting a right-table filter in `WHERE` on a `LEFT JOIN` silently turns it into an INNER join (it drops the NULL rows). Put the condition in the `ON` clause instead if you want to keep unmatched left rows.

---

## Subqueries

A query nested inside another. Three flavors:

- **Scalar subquery** — returns one value, used inline:
  ```sql
  SELECT name, salary FROM employees
  WHERE salary > (SELECT AVG(salary) FROM employees);   -- above-average earners
  ```
- **`IN` subquery** — returns a list to match against:
  ```sql
  SELECT * FROM orders WHERE customer_id IN (SELECT id FROM customers WHERE country = 'US');
  ```
- **Correlated subquery** — the inner query references the outer row, so it re-runs **per row** (powerful but can be slow):
  ```sql
  SELECT e.name FROM employees e
  WHERE e.salary > (SELECT AVG(salary) FROM employees WHERE department = e.department);
  --                                                                    ^ references outer row
  ```

> **Correlated vs non-correlated:** non-correlated runs once; correlated runs once *per outer row*. Interviewers probe this because correlated subqueries are a common performance trap (often rewritable as a JOIN or window function).

---

## CTEs (WITH)

A **Common Table Expression** is a named temporary result you define up front with `WITH`, then reference like a table. It's the readability workhorse — break a complex query into named steps instead of nesting subqueries five deep:

```sql
WITH dept_avg AS (
    SELECT department, AVG(salary) AS avg_sal
    FROM employees
    GROUP BY department
)
SELECT e.name, e.salary, d.avg_sal
FROM   employees e
JOIN   dept_avg d ON d.department = e.department
WHERE  e.salary > d.avg_sal;      -- earns above their dept average
```

- **Why CTEs:** readable named steps, avoid repeating a subquery, and enable **recursion**.
- **Recursive CTEs** walk hierarchies/graphs (org charts, category trees, "all reports under this manager"):
  ```sql
  WITH RECURSIVE reports AS (
      SELECT id, name, manager_id FROM employees WHERE id = 1   -- anchor: the top boss
      UNION ALL
      SELECT e.id, e.name, e.manager_id
      FROM employees e JOIN reports r ON e.manager_id = r.id    -- recurse down
  )
  SELECT * FROM reports;
  ```

---

## Window Functions (the Big One)

**The single highest-value advanced topic** — CTEs + window functions appear in ~40% of hard SQL interview questions. A window function computes across a set of rows **related to the current row** — *without collapsing them* (unlike `GROUP BY`). You keep every row *and* get an aggregate/rank alongside it.

```sql
SELECT
  name, department, salary,
  RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS dept_rank,
  AVG(salary) OVER (PARTITION BY department)                AS dept_avg,
  SUM(salary) OVER (ORDER BY hire_date)                     AS running_total
FROM employees;
```

**The anatomy: `FUNCTION() OVER (PARTITION BY ... ORDER BY ...)`**
- **`PARTITION BY`** = "reset the calculation for each group" (like GROUP BY, but rows aren't collapsed). Omit it → the window is the whole table.
- **`ORDER BY`** (inside `OVER`) = order within the partition — needed for ranking and running totals.

**The workhorse functions:**
| Function | What it does | Classic use |
|---|---|---|
| **`ROW_NUMBER()`** | 1,2,3… unique per row | deduplication, top-N per group |
| **`RANK()`** | ranking with **gaps** on ties (1,2,2,4) | leaderboards |
| **`DENSE_RANK()`** | ranking **no gaps** on ties (1,2,2,3) | "Nth highest salary" |
| **`LAG()` / `LEAD()`** | value from the previous / next row | period-over-period change |
| **`SUM/AVG() OVER (ORDER BY ...)`** | running total / moving average | cumulative metrics, trends |
| **`NTILE(n)`** | split rows into n buckets | quartiles, deciles |

> **Why window functions can't go in `WHERE`:** they run at the `SELECT` step (after filtering). To filter on a rank ("keep only rank ≤ 3"), wrap it in a **CTE** and filter on the outer query — this is *the* most common hard-question pattern (top-N per group).

```sql
WITH ranked AS (
  SELECT *, ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC) AS rn
  FROM employees
)
SELECT * FROM ranked WHERE rn <= 3;    -- top 3 earners per department
```

---

## Set Operations

Stack the results of two queries (they must have matching columns):

- **`UNION`** — combine and **remove duplicates**.
- **`UNION ALL`** — combine and **keep duplicates** (faster — no dedup pass; use it when you know there are none).
- **`INTERSECT`** — rows in **both**.
- **`EXCEPT`** (`MINUS` in Oracle) — rows in the first **but not** the second.

```sql
SELECT id FROM active_users
EXCEPT
SELECT id FROM banned_users;     -- active users who aren't banned
```

---

## CASE & NULL Handling

**`CASE`** is SQL's if/else — huge for feature engineering and pivoting:

```sql
SELECT name,
  CASE WHEN salary > 100000 THEN 'High'
       WHEN salary > 50000  THEN 'Mid'
       ELSE 'Low' END AS band,
  -- pivot: turn rows into columns with CASE inside an aggregate
  SUM(CASE WHEN region = 'US' THEN amount ELSE 0 END) AS us_sales
FROM employees GROUP BY name;
```

**NULL is the classic trap** — NULL means *unknown*, so:
- `NULL = NULL` is **not true** (it's unknown) — use `IS NULL` / `IS NOT NULL`, never `= NULL`.
- Any arithmetic with NULL → NULL (`5 + NULL = NULL`).
- **`COUNT(*)`** counts all rows; **`COUNT(col)`** skips NULLs — a favorite gotcha.
- **`COALESCE(a, b, c)`** returns the first non-NULL — the standard "default value" tool.

---

## Indexes & Performance

An **index** is a lookup structure (usually a B-tree) that lets the database find rows without scanning the whole table — like a book's index instead of reading every page. Interview-relevant points:

- Index the columns you **filter (`WHERE`), join (`ON`), and sort (`ORDER BY`)** on.
- Indexes **speed reads but slow writes** (every insert must update the index) and cost storage — don't index everything.
- **`EXPLAIN` / `EXPLAIN ANALYZE`** shows the query plan — how to tell a **full table scan** (slow) from an **index scan** (fast).
- Common wins: avoid `SELECT *`, filter early, replace correlated subqueries with joins/window functions, avoid functions on indexed columns in `WHERE` (`WHERE YEAR(date) = 2025` can't use the index — use a range instead).

---

## Classic Interview Patterns

**Nth-highest value** (2nd highest salary) — `DENSE_RANK` handles ties correctly:
```sql
WITH r AS (SELECT salary, DENSE_RANK() OVER (ORDER BY salary DESC) AS rk FROM employees)
SELECT DISTINCT salary FROM r WHERE rk = 2;
```

**Top-N per group** (top 3 per department) — window + CTE (shown above).

**Remove duplicates, keep one:**
```sql
WITH d AS (SELECT *, ROW_NUMBER() OVER (PARTITION BY email ORDER BY id) AS rn FROM users)
DELETE FROM users WHERE id IN (SELECT id FROM d WHERE rn > 1);
```

**Running total / month-over-month** — `SUM() OVER (ORDER BY ...)` and `LAG()`.

**Find gaps / consecutive streaks** — `ROW_NUMBER()` minus a date (the "islands and gaps" pattern).

---

## Interview Questions

### Q: What's the difference between `WHERE` and `HAVING`?
**Strong answer:** `WHERE` filters individual **rows before** grouping and can't see aggregates; `HAVING` filters **groups after** aggregation and can use `COUNT`/`SUM`/etc. It falls straight out of execution order — `WHERE` (step 2) runs before `GROUP BY` (step 3), `HAVING` (step 4) runs after.

### Q: Explain the logical execution order of a SELECT query.
**Strong answer:** FROM/JOIN → WHERE → GROUP BY → HAVING → SELECT (window functions) → ORDER BY → LIMIT. It's different from the written order, and it explains why aliases from SELECT work in ORDER BY but not WHERE, and why window functions can't appear in WHERE.

### Q: INNER vs LEFT JOIN?
**Strong answer:** INNER returns only rows that match in both tables; LEFT returns all rows from the left table plus matches from the right (NULLs where there's no match). Use LEFT when you need to keep unmatched left rows — e.g. "all customers, including those with zero orders." Watch the gotcha: a right-table condition in `WHERE` turns a LEFT JOIN back into an INNER JOIN.

### Q: When do you use a window function instead of GROUP BY?
**Strong answer:** When you need an aggregate or rank **alongside each row without collapsing rows**. GROUP BY returns one row per group; a window function keeps all rows and adds the computed column (rank, running total, dept average). Ranking, running totals, and period-over-period comparisons all need windows.

### Q: Why can't you filter on a window function in WHERE, and how do you do it?
**Strong answer:** Window functions execute at the SELECT step, after WHERE. So you compute the window value in a CTE (or subquery) and filter on the outer query — the standard top-N-per-group pattern with `ROW_NUMBER()`.

### Q: `COUNT(*)` vs `COUNT(column)`?
**Strong answer:** `COUNT(*)` counts all rows; `COUNT(column)` counts only rows where that column is **non-NULL**. Different answers whenever the column has NULLs — a common bug source.

### Q: How would you find the 2nd highest salary (handling ties)?
**Strong answer:** `DENSE_RANK() OVER (ORDER BY salary DESC)` in a CTE, then select where rank = 2. DENSE_RANK (no gaps) is correct for "Nth highest distinct value"; `ROW_NUMBER` would miss ties and `RANK` leaves gaps.

### Q: How do you optimize a slow query?
**Strong answer:** Run `EXPLAIN ANALYZE` to find full table scans; add indexes on filtered/joined/sorted columns; avoid `SELECT *`; filter early; rewrite correlated subqueries as joins or window functions; and avoid wrapping indexed columns in functions in the `WHERE` clause (it defeats the index).

---

## Glossary

| Term | Simple explanation | Purpose |
|---|---|---|
| **Declarative** | You describe the result, not the steps to get it | The engine's planner chooses the algorithm for you |
| **Execution order** | The logical order clauses run: FROM→WHERE→GROUP BY→HAVING→SELECT→ORDER BY→LIMIT | Explains most SQL "why doesn't this work" rules |
| **SELECT** | Chooses which columns/expressions to return | The output projection of a query |
| **WHERE** | Filters individual rows before grouping | Removes rows that don't meet a condition |
| **GROUP BY** | Collapses rows into groups for aggregation | Compute per-category aggregates |
| **HAVING** | Filters groups after aggregation | Filter on aggregate results (COUNT, SUM…) |
| **Aggregate function** | COUNT/SUM/AVG/MIN/MAX — collapse many rows to one value | Summary statistics per group or table |
| **JOIN** | Combine rows from two tables on a key | Bring related data together |
| **INNER / LEFT / FULL JOIN** | Keep matches only / all-left+matches / all rows | Control which unmatched rows survive |
| **Subquery** | A query nested inside another | Reuse a result inline (scalar, IN, correlated) |
| **Correlated subquery** | Inner query that references the outer row (re-runs per row) | Row-relative comparisons; a performance watch-point |
| **CTE (WITH)** | A named temporary result you reference like a table | Readable named steps; enables recursion |
| **Recursive CTE** | A CTE that references itself | Traverse hierarchies/graphs (org charts, trees) |
| **Window function** | Aggregate/rank over related rows without collapsing them | Rankings, running totals, period-over-period |
| **PARTITION BY** | Resets a window calculation per group | Per-group ranks/aggregates while keeping rows |
| **ROW_NUMBER / RANK / DENSE_RANK** | Row numbering / ranking with gaps / ranking without gaps | Top-N, dedup, Nth-highest |
| **LAG / LEAD** | Value from the previous / next row | Compare a row to its neighbor over time |
| **UNION / UNION ALL** | Stack two result sets, dedup / keep duplicates | Combine queries with matching columns |
| **CASE** | SQL's if/else expression | Conditional logic, feature bucketing, pivoting |
| **NULL** | Unknown/missing value | Needs IS NULL and COALESCE; breaks `=` comparisons |
| **COALESCE** | Returns the first non-NULL argument | Provide default values for missing data |
| **Index** | A lookup structure (B-tree) over a column | Find rows fast without a full table scan |
| **Full table scan** | Reading every row because no index applies | The slow path an index avoids |
| **EXPLAIN / query plan** | Shows how the DB will execute a query | Diagnose and optimize slow queries |

---

## References

- [PostgreSQL — SQL Language documentation](https://www.postgresql.org/docs/current/sql.html) · [Window Functions tutorial](https://www.postgresql.org/docs/current/tutorial-window.html)
- [Mode — SQL Tutorial (Basic → Advanced)](https://mode.com/sql-tutorial/)
- [60 SQL Interview Questions, Beginner to Advanced (Dataquest, 2026)](https://www.dataquest.io/blog/sql-interview-questions-from-beginner-to-advanced/)
- [Top 99 SQL Interview Questions (DataCamp, 2026)](https://www.datacamp.com/blog/top-sql-interview-questions-and-answers-for-beginners-and-intermediate-practitioners)

---

*Previous: [Apache Spark Architecture](01-apache-spark-architecture.md) | Up: [Guide Home](../README.md)*
