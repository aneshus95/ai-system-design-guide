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
- [Window Functions — Made Visual](#window-functions--made-visual)
- [Window Frames (Moving Windows)](#window-frames-moving-windows)
- [Set Operations](#set-operations)
- [CASE & NULL Handling](#case--null-handling)
- [Advanced SQL Topics](#advanced-sql-topics)
- [Query Optimization Deep Dive](#query-optimization-deep-dive)
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

## Window Functions — Made Visual

Window functions are the highest-value advanced topic (~40% of hard SQL questions), and also the most-misunderstood. Let's build the picture from actual rows.

### The one idea: keep every row, add a computed column

**`GROUP BY` collapses rows. A window function does NOT.** That's the whole difference. Watch what each does to the same table:

```
 SOURCE TABLE                     GROUP BY dept              WINDOW: AVG() OVER (PARTITION BY dept)
 name   dept   salary            dept   avg_salary          name   dept   salary  dept_avg
 Alice  Eng    120        ─┐     Eng    106.7        ─┐      Alice  Eng    120     106.7   ─┐
 Bob    Eng    100         ├──►  Sales   85           │      Bob    Eng    100     106.7    │  5 rows
 Carol  Eng    100         │     (2 rows — names      │      Carol  Eng    100     106.7    │  KEPT — every
 Dave   Sales  90          │      are GONE)           ┘      Dave   Sales   90      85      │  row survives,
 Eve    Sales  80         ─┘                                 Eve    Sales   80      85      ─┘  avg attached
```

**GROUP BY** answers "one number per group" and throws the rows away. **The window function** answers "attach the group's number to *every row*." You keep Alice, Bob, Carol *and* each learns their dept average sits next to them. That's why you use windows when you need per-row detail **and** an aggregate together.

### The mental model: a spotlight that slides down the table

For **each row**, the database:
1. **draws a "window"** — the set of *related* rows (defined by `PARTITION BY`),
2. **computes** the function over that window,
3. **writes the result next to the current row**,
4. moves to the next row and repeats.

```
  FUNCTION() OVER ( PARTITION BY <which rows are "related"> ORDER BY <order inside the window> )
                    └── draws the window ──┘                └── needed for rank / running totals ──┘
```

- **`PARTITION BY dept`** = "the window for a row is only the rows with the same dept." Omit it → the window is the *whole table*.
- **`ORDER BY`** *inside* `OVER(...)` = the order the spotlight walks the window in — required for ranking and running totals (there's no "previous row" without an order).

### Ranking, visualized (the three ranks)

`ORDER BY salary DESC`, partitioned by dept — watch how ties are handled differently:

```
 Eng partition (sorted desc):     ROW_NUMBER  RANK  DENSE_RANK
   Alice   120                        1         1        1
   Bob     100   ◄─ tie               2         2        2
   Carol   100   ◄─ tie               3         2        2
   (next distinct value)            ...         4        3
                                      │         │        │
                     always unique ──┘  gaps ──┘  no gaps┘
```
- **`ROW_NUMBER()`** — always 1,2,3, even on ties (arbitrary tiebreak). *Use for dedup / top-N.*
- **`RANK()`** — ties share a rank, then it **skips** (1,2,2,**4**). *Use for leaderboards.*
- **`DENSE_RANK()`** — ties share a rank, **no skip** (1,2,2,**3**). *Use for "Nth highest distinct value."*

### Running total, visualized (add `ORDER BY`, no partition)

`SUM(salary) OVER (ORDER BY hire_date)` — each row's window is *itself + everything before it*, so the window **grows** as you go down:

```
 hire_date  salary   window (rows summed)          running_total
 Jan        120      [120]                          120
 Feb        100      [120,100]                      220
 Mar        100      [120,100,100]                  320
 Apr         90      [120,100,100,90]               410   ← each row sees all rows up to itself
```

### The workhorse functions

| Function | What it does | Classic use |
|---|---|---|
| **`ROW_NUMBER()`** | 1,2,3… unique per row | deduplication, top-N per group |
| **`RANK()`** | ranking with **gaps** on ties (1,2,2,4) | leaderboards |
| **`DENSE_RANK()`** | ranking **no gaps** on ties (1,2,2,3) | "Nth highest salary" |
| **`LAG(col)` / `LEAD(col)`** | value from the previous / next row | period-over-period change |
| **`SUM/AVG() OVER (ORDER BY…)`** | running total / moving average | cumulative metrics, trends |
| **`FIRST_VALUE/LAST_VALUE`** | first/last value in the window | "compare each row to the group's best" |
| **`NTILE(n)`** | split rows into n equal buckets | quartiles, deciles |

### Why windows can't go in `WHERE` (and the fix)

Window functions run at the **`SELECT`** step — *after* `WHERE`. So the rank doesn't exist yet when `WHERE` runs. To filter on it, compute it in a **CTE**, then filter on the outer query. This is *the* canonical hard-question pattern — **top-N per group**:

```sql
WITH ranked AS (
  SELECT *, ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC) AS rn
  FROM employees
)
SELECT * FROM ranked WHERE rn <= 3;    -- top 3 earners per department
```

---

## Window Frames (Moving Windows)

By default, `... OVER (ORDER BY x)` gives a **growing** window (everything up to the current row — a running total). A **frame clause** lets you make the window a **fixed-size sliding window** instead — essential for moving averages.

```
 ROWS BETWEEN 2 PRECEDING AND CURRENT ROW     →  a 3-row window that SLIDES down:

 row1  ┌───┐                     frame = {row1}                (not enough preceding yet)
 row2  │win│                     frame = {row1, row2}
 row3  └───┘ ◄current            frame = {row1, row2, row3}    ← 3-row average here
 row4      ┌───┐                 frame = {row2, row3, row4}    ← window slid down one
 row5      │win│ ◄current        frame = {row3, row4, row5}
           └───┘
```

```sql
-- 3-day moving average of sales
SELECT day, sales,
  AVG(sales) OVER (ORDER BY day ROWS BETWEEN 2 PRECEDING AND CURRENT ROW) AS moving_avg_3d
FROM daily_sales;
```

**Frame vocabulary:**
- **`ROWS`** — count by physical rows (exact: "2 rows back"). Most common.
- **`RANGE`** — count by *value* (all rows with an `ORDER BY` value within a range; treats ties as one). The default when you add `ORDER BY` is `RANGE UNBOUNDED PRECEDING → CURRENT ROW`.
- **Bounds:** `UNBOUNDED PRECEDING` (start of partition), `N PRECEDING`, `CURRENT ROW`, `N FOLLOWING`, `UNBOUNDED FOLLOWING` (end of partition).
- **Whole partition:** `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING`.

> **Gotcha:** a running total *seems* to work without a frame because adding `ORDER BY` silently applies the default `RANGE ... CURRENT ROW` frame. But `RANGE` lumps tied `ORDER BY` values into the same frame — if two rows share the same date, they'll get the *same* running total. Use `ROWS` when you want strict row-by-row behavior.

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

## Advanced SQL Topics

Beyond the core, these show up in senior DS/DE interviews.

**`EXISTS` vs `IN` (and `NOT IN`'s NULL trap).** `WHERE EXISTS (subquery)` returns true as soon as the subquery finds *one* matching row — it **short-circuits**, so it's usually faster than `IN` for large/correlated subqueries. The big trap: **`NOT IN` silently returns nothing if the subquery contains a NULL** (because "x not in (1, NULL)" evaluates to unknown). Prefer `NOT EXISTS` for anti-joins.
```sql
SELECT * FROM customers c
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id);   -- customers with orders
```

**`LATERAL` join (a.k.a. `CROSS APPLY`).** A join where the right side can **reference each left row** — lets you run a subquery *per row* (e.g. "the 3 most recent orders for each customer"). It's the clean way to do top-N-per-group inline.
```sql
SELECT c.name, o.*
FROM customers c
CROSS JOIN LATERAL (
  SELECT * FROM orders o WHERE o.customer_id = c.id ORDER BY o.date DESC LIMIT 3
) o;
```

**Pivot / unpivot.** Turn rows into columns (pivot) with `CASE` inside aggregates (shown above), or use native `PIVOT` in some engines; unpivot turns columns back into rows with `UNION ALL` or `UNNEST`.

**Date/time & string functions.** `DATE_TRUNC('month', ts)` (bucket by month), `EXTRACT(YEAR FROM ts)`, date arithmetic (`ts + INTERVAL '7 days'`), `DATEDIFF`; strings: `CONCAT`, `SUBSTRING`, `TRIM`, `LOWER`, `REPLACE`, `LIKE`/`ILIKE`, regex (`~`). Time-bucketing is bread-and-butter for analytics.

**JSON / semi-structured.** Modern SQL queries JSON columns: `data->>'field'` (Postgres), `JSON_VALUE`/`JSON_EXTRACT` — useful when events/logs live in one column.

**Transactions & ACID.** A **transaction** groups statements so they succeed or fail together (`BEGIN … COMMIT` / `ROLLBACK`). **ACID** = **A**tomicity (all-or-nothing), **C**onsistency (valid states only), **I**solation (concurrent txns don't corrupt each other), **D**urability (committed = survives a crash).

**Isolation levels** (the concurrency trade-off — higher = safer but slower):

| Level | Prevents | Allows |
|---|---|---|
| **Read Uncommitted** | — | dirty reads (see uncommitted data) |
| **Read Committed** | dirty reads | non-repeatable reads |
| **Repeatable Read** | non-repeatable reads | phantom rows |
| **Serializable** | everything (acts as if txns ran one-at-a-time) | lowest concurrency |

**`MERGE` / `UPSERT`.** "Insert if new, update if exists" in one statement — Postgres `INSERT ... ON CONFLICT DO UPDATE`, standard `MERGE`. Core for idempotent pipelines (see [Spark/data-eng ingestion](../06-retrieval-systems/15-data-engineering-for-ai.md)).

**Views vs materialized views.** A **view** is a saved query (re-runs every time — always fresh, no storage). A **materialized view** stores the *result* (fast reads, but must be `REFRESH`ed — stale until then). Trade freshness for speed.

**Partitioning.** Split one huge table into physical chunks by a key (usually date). Queries that filter on that key hit only the relevant partitions (**partition pruning**) — the SQL-table analog of sharding, and how warehouses stay fast on billions of rows.

---

## Query Optimization Deep Dive

The mental model: **SQL is declarative, so a query optimizer/planner turns your query into an execution plan.** Optimization is about helping the planner pick a *fast* plan — mostly by using indexes well and avoiding full scans of huge tables.

### Indexes — the foundation

An **index** is a sorted lookup structure (usually a **B-tree**) — like a book's index, so the DB finds rows without reading every page (a **full table scan**).
- Index the columns you **filter (`WHERE`)**, **join (`ON`)**, and **sort (`ORDER BY`)** on.
- Indexes **speed reads but slow writes** (every insert/update maintains them) and cost storage — don't index everything.
- **Composite index order matters:** an index on `(a, b)` helps `WHERE a = ?` and `WHERE a = ? AND b = ?`, but **not** `WHERE b = ?` alone (leftmost-prefix rule) — like a phone book sorted by last-then-first name.
- **Covering index:** if an index contains *every* column a query needs (filter + join + `SELECT`), the DB answers from the index alone and never touches the table — an **index-only scan**, the fastest path.

### Sargability — the #1 rule for `WHERE`

A predicate is **sargable** ("Search ARGument-able") if it lets the DB use an index seek. **The killer: wrapping the indexed column in a function or math makes it non-sargable** — the index is useless.

```sql
WHERE YEAR(order_date) = 2025        -- ❌ non-sargable: function on the column → full scan
WHERE order_date >= '2025-01-01'     -- ✅ sargable: a range on the raw column → index seek
  AND order_date <  '2026-01-01'

WHERE status = 'active'              -- ✅ sargable
WHERE UPPER(status) = 'ACTIVE'       -- ❌ non-sargable (unless you index the expression)
WHERE price * 1.1 > 100              -- ❌ move the math to the other side: price > 100/1.1
```

### Join algorithms (what the planner chooses)

Interviewers love this — it explains *why* a join is slow:

| Algorithm | How it works | Best when |
|---|---|---|
| **Nested Loop** | for each row of the small table, look up matches in the other (ideally via index) | one side is small / has an index |
| **Hash Join** | build a hash table on the smaller side, probe it with the larger | large, unsorted tables, equality joins |
| **Merge Join** | sort both sides, walk them in parallel like a zipper | both inputs already sorted / on sorted indexes |

*Intuition:* nested loop = "look each one up"; hash = "build a dictionary, then check"; merge = "zip two sorted lists." If a join is slow, `EXPLAIN` often shows a nested loop over a **big** table with no index — add the index.

### Reading the plan (`EXPLAIN`)

`EXPLAIN` shows the plan; `EXPLAIN ANALYZE` actually runs it and shows real timings/row counts. What to look for:
- **`Seq Scan` / full table scan** on a big table in a `WHERE`/`JOIN` → usually a missing or unusable (non-sargable) index.
- **Row-estimate vs actual mismatch** → stale statistics; run `ANALYZE` so the planner has accurate data.
- **The expensive node** → the plan is a tree; find the node eating the time and fix *that*.

### Common optimizations (the checklist)

- **Avoid `SELECT *`** — fetch only needed columns (enables covering indexes, less I/O).
- **Filter early / push predicates down** — reduce rows before joins and aggregations.
- **Keep predicates sargable** (no functions on indexed columns).
- **Replace correlated subqueries** with joins or window functions (they re-run per row).
- **`UNION ALL` over `UNION`** when duplicates are impossible (skips the dedup sort).
- **`EXISTS`/`NOT EXISTS` over `IN`/`NOT IN`** for large or NULL-prone subqueries.
- **Batch, don't loop** — one set-based query beats N per-row queries (the app-side "N+1" trap).
- **Keep statistics fresh** and consider **partitioning** for very large tables.

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

### Q: What does "sargable" mean and why does it matter?
**Strong answer:** A predicate is sargable if the DB can use an index seek for it. Wrapping the indexed column in a function (`WHERE YEAR(date) = 2025`) makes it non-sargable → full scan. Rewriting it as a range on the raw column (`date >= '2025-01-01' AND date < '2026-01-01'`) keeps it sargable so the index works. It's the single most common reason an "indexed" query is still slow.

### Q: What's a covering index?
**Strong answer:** An index that contains *every* column a query needs — its filter, join, and SELECT columns. The DB answers from the index alone (index-only scan) without touching the table, which is the fastest read path. It's why avoiding `SELECT *` helps: fewer columns → easier to cover.

### Q: Explain the three join algorithms.
**Strong answer:** Nested loop — for each row of the small side, look up matches in the other (great with an index, one side small). Hash join — build a hash table on the smaller side and probe with the larger (best for big unsorted equality joins). Merge join — sort both sides and walk them in parallel like a zipper (best when inputs are already sorted). The planner picks based on size, indexes, and sortedness; a nested loop over a big unindexed table is the classic slow plan.

### Q: `EXISTS` vs `IN` — when and why?
**Strong answer:** `EXISTS` short-circuits (stops at the first match) so it's usually better for large or correlated subqueries; `IN` is fine for small static lists. Critically, **`NOT IN` breaks with NULLs** in the subquery (returns no rows), so use `NOT EXISTS` for anti-joins.

### Q: What are transaction isolation levels?
**Strong answer:** They trade concurrency for correctness. Read Committed (default in most DBs) prevents dirty reads; Repeatable Read also prevents non-repeatable reads; Serializable behaves as if transactions ran one at a time (safest, slowest). Higher isolation prevents more anomalies (dirty/non-repeatable reads, phantoms) at the cost of throughput.

### Q: What's the difference between a view and a materialized view?
**Strong answer:** A view is a saved query that re-runs every time (always fresh, no storage). A materialized view stores the result (fast reads, less compute) but goes stale until you `REFRESH` it. You pick based on freshness-vs-speed — materialized views suit expensive aggregations queried far more often than the data changes.

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
| **Window frame** | The subset of the partition a window function computes over (e.g. `ROWS BETWEEN 2 PRECEDING AND CURRENT ROW`) | Fixed-size sliding windows — moving averages |
| **ROWS vs RANGE** | Frame by physical row count vs by ORDER-BY value | ROWS = strict row window; RANGE lumps ties together |
| **EXISTS / NOT EXISTS** | Tests whether a subquery returns any row (short-circuits) | Efficient existence checks and NULL-safe anti-joins |
| **LATERAL / CROSS APPLY** | A join whose right side references each left row | Per-row subqueries, e.g. top-N per group inline |
| **Transaction** | A group of statements that commit or roll back together | All-or-nothing changes (the T in ACID) |
| **ACID** | Atomicity, Consistency, Isolation, Durability | The guarantees that make a DB reliable under failure/concurrency |
| **Isolation level** | How much concurrent transactions can affect each other | Trade concurrency for correctness (Read Committed → Serializable) |
| **UPSERT / MERGE** | Insert-if-new, update-if-exists in one statement | Idempotent writes in data pipelines |
| **View** | A saved query that re-runs each time | Reusable logic, always fresh, no storage |
| **Materialized view** | A stored (cached) query result | Fast reads on expensive aggregations; needs REFRESH |
| **Partitioning** | Splitting a big table into chunks by a key | Partition pruning — scan only relevant chunks |
| **Index** | A sorted lookup structure (B-tree) over columns | Find rows fast without a full table scan |
| **Composite index** | An index on multiple columns, order matters | Serves leftmost-prefix filters (`(a,b)` helps `a`, not `b` alone) |
| **Covering index** | An index holding all columns a query needs | Index-only scan — answer without touching the table |
| **Sargable** | A predicate an index seek can use (no function on the column) | The #1 rule for making WHERE clauses use indexes |
| **Query planner / optimizer** | The engine that turns your SQL into an execution plan | Chooses join algorithms, indexes, and order for you |
| **Join algorithms** | Nested loop / hash / merge | How the planner physically executes a join; explains join speed |
| **Full table scan (Seq Scan)** | Reading every row because no index applies | The slow path indexes and sargable predicates avoid |
| **EXPLAIN / query plan** | Shows how the DB will execute a query (`ANALYZE` runs it) | Diagnose and optimize slow queries |
| **Statistics (ANALYZE)** | The DB's row-count/distribution estimates | Keep fresh so the planner picks good plans |

---

## References

- [PostgreSQL — SQL Language documentation](https://www.postgresql.org/docs/current/sql.html) · [Window Functions tutorial](https://www.postgresql.org/docs/current/tutorial-window.html) · [Using EXPLAIN](https://www.postgresql.org/docs/current/using-explain.html)
- [Use The Index, Luke — SQL indexing & tuning](https://use-the-index-luke.com/) · [What is Sargability? — Baeldung](https://www.baeldung.com/sql/sargability)
- [PostgreSQL Join Optimization: Nested Loop, Hash, Merge](https://dev.to/philip_mcclarence_2ef9475/postgresql-join-optimization-nested-loop-hash-and-merge-1cn9) · [Window frame clause (ROWS/RANGE/GROUPS) — jOOQ](https://www.jooq.org/doc/latest/manual/sql-building/column-expressions/window-functions/window-frame/)
- [Mode — SQL Tutorial (Basic → Advanced)](https://mode.com/sql-tutorial/)
- [60 SQL Interview Questions, Beginner to Advanced (Dataquest, 2026)](https://www.dataquest.io/blog/sql-interview-questions-from-beginner-to-advanced/) · [Top 99 SQL Interview Questions (DataCamp, 2026)](https://www.datacamp.com/blog/top-sql-interview-questions-and-answers-for-beginners-and-intermediate-practitioners)

---

*Previous: [Apache Spark Architecture](01-apache-spark-architecture.md) | Up: [Guide Home](../README.md)*
