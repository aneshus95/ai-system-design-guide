# 🧠 SQL Pattern Master Notes

> Goal: Become highly proficient at SQL interview problems and LeetCode SQL by learning to **recognize query patterns, reason about rows and groups, and derive queries** instead of memorizing solutions.

---

# 0. 🧭 Universal SQL Problem-Solving Framework

Use this for **every** SQL problem.

## 1. INPUT

Identify:
- Tables
- Columns
- Data types
- Primary keys
- Foreign keys
- Relationships

## 2. OUTPUT

Ask:
- What columns must be returned?
- One row per what?
- Is aggregation required?
- Does ordering matter?
- Are duplicates allowed?

## 3. GRAIN

This is one of the most important SQL questions:

> **What does one output row represent?**

Examples:
- One row per employee
- One row per department
- One row per customer
- One row per customer per month
- One row per product

If you don't know the grain, the query can easily be wrong.

## 4. FILTER

Ask:

> Which rows should exist before aggregation?

Usually:

```sql
WHERE ...
```

## 5. RELATIONSHIPS

Ask:

> Do I need information from another table?

If yes:

```sql
JOIN
```

## 6. GROUP

Ask:

> Do multiple rows need to become one row?

If yes:

```sql
GROUP BY
```

## 7. AGGREGATE

Ask:

> Do I need COUNT, SUM, AVG, MIN, MAX, etc.?

## 8. WINDOW

Ask:

> Do I need calculations across related rows while keeping the original rows?

Use:

```sql
OVER(...)
```

## 9. ORDER

Ask:

> Do I need ranking, latest/earliest, running totals, or sorted output?

## 10. EDGE CASES

Check:
- NULL
- Duplicate rows
- Ties
- No matching rows
- Zero counts
- Multiple records on same date
- Customers with no transactions
- Employees without managers
- Division by zero

---

# 1. 🗺️ SQL Pattern Recognition Cheat Sheet

| Problem clue | SQL pattern |
|---|---|
| Filter rows | `WHERE` |
| Remove duplicates | `DISTINCT` |
| Combine tables | `JOIN` |
| Keep unmatched rows | `LEFT JOIN` |
| Count rows | `COUNT(*)` |
| Count unique values | `COUNT(DISTINCT x)` |
| Group by category | `GROUP BY` |
| Filter groups | `HAVING` |
| Sort output | `ORDER BY` |
| Top N rows | `ORDER BY + LIMIT` |
| Nth highest | `DENSE_RANK` / subquery |
| Rank rows | `RANK`, `DENSE_RANK`, `ROW_NUMBER` |
| Top row per group | Window function |
| Latest row per entity | `ROW_NUMBER()` |
| Previous/next row | `LAG` / `LEAD` |
| Running total | `SUM() OVER` |
| Moving average | Window frame |
| Compare current vs previous | `LAG` |
| Consecutive dates | `LAG` + date difference |
| Gaps and islands | Window functions |
| Conditional count | `SUM(CASE WHEN...)` |
| Conditional aggregation | `SUM/COUNT(CASE...)` |
| Pivot-like result | Conditional aggregation |
| Customers with no records | `LEFT JOIN ... IS NULL` |
| Existence | `EXISTS` |
| Non-existence | `NOT EXISTS` |
| Compare against aggregate | Subquery / CTE |
| Reuse intermediate result | CTE |
| Recursive hierarchy | Recursive CTE |
| Self relationship | Self JOIN |
| Duplicate records | `GROUP BY ... HAVING COUNT(*) > 1` |
| Date filtering | Date functions / ranges |
| String pattern | `LIKE`, string functions |
| Null replacement | `COALESCE` |
| Null comparison | `IS NULL` / `IS NOT NULL` |
| Set comparison | `UNION`, `INTERSECT`, `EXCEPT` |
| Relational division | `GROUP BY/HAVING` or `NOT EXISTS` |

---

# 2. 🔎 SELECT + WHERE

## 🧠 Mental Model

> "Start with the rows I care about, then choose what I want to see."

## 🔎 Recognize

- Find employees with salary > X
- Customers from a country
- Orders after a date
- Filter records by condition

## 🧩 Generic Template

```sql
SELECT
    column1,
    column2
FROM table_name
WHERE condition;
```

## 👀 Visual

```text
Table
 ↓
Filter rows
 ↓
Remaining rows
 ↓
Selected columns
```

## 💡 Example

```sql
SELECT name, salary
FROM employees
WHERE salary > 100000;
```

## 🔒 Invariant

Every returned row satisfies the `WHERE` condition.

## ⚠️ Common Mistake

`WHERE` filters individual rows.

It cannot directly filter an aggregate like:

```sql
COUNT(*)
```

For aggregate filtering, use `HAVING`.

---

# 3. 🎯 DISTINCT

## 🧠 Mental Model

> "I only care about unique combinations of the selected columns."

## 🧩 Generic Template

```sql
SELECT DISTINCT column
FROM table;
```

Multiple columns:

```sql
SELECT DISTINCT department, location
FROM employees;
```

This returns unique **combinations**, not independently unique values.

## 🔎 Recognize

Words like:
- Unique
- Different
- Distinct
- How many different X?

## 💡 Example

```sql
SELECT COUNT(DISTINCT customer_id)
FROM orders;
```

## ⚠️ Common Mistake

These are different:

```sql
COUNT(*)
```

vs.

```sql
COUNT(DISTINCT customer_id)
```

---

# 4. 🔗 INNER JOIN

## 🧠 Mental Model

> "Match rows from two tables where the relationship is satisfied."

## 🔎 Recognize

- Employee + department
- Order + customer
- Product + category
- Need columns from another table

## 🧩 Generic Template

```sql
SELECT
    a.column,
    b.column
FROM table_a a
JOIN table_b b
    ON a.key = b.key;
```

## 👀 Visual

```text
Table A          Table B

 A1 ───────────── B1
 A2 ───────────── B2

Only matching rows survive.
```

## 💡 Example

```sql
SELECT e.name, d.department_name
FROM employees e
JOIN departments d
    ON e.department_id = d.id;
```

## 🔒 Invariant

Every returned row has a matching row satisfying the `ON` condition.

## ⚠️ Critical Question

Before joining, ask:

> **What is the relationship cardinality?**

Is it:
- One-to-one?
- One-to-many?
- Many-to-many?

A one-to-many join can multiply rows.

---

# 5. 👈 LEFT JOIN

## 🧠 Mental Model

> "Keep every row from the left table, even if there is no match."

## 🔎 Recognize

- Customers with no orders
- Employees without projects
- Products never sold
- Include entities with zero activity

## 🧩 Generic Template

```sql
SELECT
    a.*,
    b.column
FROM table_a a
LEFT JOIN table_b b
    ON a.id = b.a_id;
```

## 👀 Visual

```text
LEFT TABLE              RIGHT TABLE

A ──────────────── B
C ──────────────── D
E ──────────────── NULL
```

The unmatched left row remains.

## 💡 Example

Customers who never ordered:

```sql
SELECT c.customer_id
FROM customers c
LEFT JOIN orders o
    ON c.customer_id = o.customer_id
WHERE o.customer_id IS NULL;
```

## 🔒 Invariant

Every row from the left table remains represented.

## ⚠️ Common Trap

This can accidentally turn a `LEFT JOIN` into an effective inner join:

```sql
FROM customers c
LEFT JOIN orders o
    ON c.id = o.customer_id
WHERE o.status = 'completed'
```

The `WHERE` condition removes NULL-extended rows.

Sometimes the condition belongs in the `ON` clause instead.

---

# 6. 🧮 GROUP BY + Aggregation

## 🧠 Mental Model

> "Collapse multiple rows into one result row per group."

## 🔎 Recognize

- Total sales per customer
- Average salary per department
- Number of orders per customer
- Maximum score per student

## 🧩 Generic Template

```sql
SELECT
    group_column,
    AGGREGATE(value)
FROM table
GROUP BY group_column;
```

Common aggregates:

```sql
COUNT(*)
COUNT(column)
COUNT(DISTINCT column)
SUM(column)
AVG(column)
MIN(column)
MAX(column)
```

## 👀 Visual

```text
Raw rows

A
A
A
B
B
C

GROUP BY

A → one row
B → one row
C → one row
```

## 💡 Example

```sql
SELECT customer_id, SUM(amount) AS total_spent
FROM orders
GROUP BY customer_id;
```

## 🔒 Invariant

There is exactly one output row per grouping key.

## ⚠️ Critical Question

Before writing `GROUP BY`, say:

> "One output row represents ______."

---

# 7. 🚦 HAVING

## 🧠 Mental Model

> "Filter groups after aggregation."

## WHERE vs HAVING

```text
WHERE
  ↓
Filter individual rows
  ↓
GROUP BY
  ↓
Aggregate
  ↓
HAVING
  ↓
Filter groups
```

## 🧩 Generic Template

```sql
SELECT
    customer_id,
    COUNT(*) AS order_count
FROM orders
GROUP BY customer_id
HAVING COUNT(*) >= 5;
```

## 🔎 Recognize

- Customers with more than 5 orders
- Departments with average salary > X
- Products sold at least N times

## 🔒 Invariant

Every output group satisfies the aggregate condition.

---

# 8. 🏆 ORDER BY + LIMIT

## 🧠 Mental Model

> "Sort the result according to my desired priority, then keep the required number of rows."

## 🧩 Generic Template

```sql
SELECT *
FROM table
ORDER BY column DESC
LIMIT 10;
```

## 🔎 Recognize

- Highest salary
- Lowest price
- Top 5 customers
- Latest 10 orders

## 💡 Example

```sql
SELECT *
FROM employees
ORDER BY salary DESC
LIMIT 3;
```

## ⚠️ Common Trap

If ties matter, simple `LIMIT` may not solve the actual question.

Example:

> "Third highest salary"

If multiple employees share the same salary, you may need a ranking concept.

---

# 9. 🥇 Ranking — ROW_NUMBER, RANK, DENSE_RANK

## 🧠 Mental Model

> "Assign an ordered position to rows, optionally within groups."

## 🔎 Recognize

- Rank employees
- Top N per department
- Second highest salary
- Latest record per customer
- Rank scores

## 🧩 Generic Template

```sql
SELECT
    *,
    ROW_NUMBER() OVER (
        ORDER BY salary DESC
    ) AS rn
FROM employees;
```

## ROW_NUMBER

Always produces unique positions:

```text
salary   row_number
100      1
100      2
90       3
```

## RANK

Ties share rank, with gaps:

```text
salary   rank
100      1
100      1
90       3
```

## DENSE_RANK

Ties share rank, without gaps:

```text
salary   dense_rank
100      1
100      1
90       2
```

## 🔒 Key Question

Ask:

> **What should happen when values tie?**

That determines the ranking function.

---

# 10. 🏅 Top N Per Group

## 🧠 Mental Model

> "Rank rows inside each group, then filter the rank."

## 🔎 Recognize

- Top 3 salaries per department
- Most expensive product per category
- Latest 2 orders per customer

## 🧩 Generic Template

```sql
WITH ranked AS (
    SELECT
        *,
        ROW_NUMBER() OVER (
            PARTITION BY group_column
            ORDER BY value DESC
        ) AS rn
    FROM table_name
)

SELECT *
FROM ranked
WHERE rn <= 3;
```

## 👀 Visual

```text
Department A
  rank 1
  rank 2
  rank 3
  rank 4

Department B
  rank 1
  rank 2
  rank 3
```

`PARTITION BY` creates an independent ranking universe for each group.

## 🔒 Invariant

The ranking resets for every partition.

---

# 11. 🪟 Window Functions

## 🧠 Mental Model

> "Calculate across related rows without collapsing them."

This is the key difference:

```text
GROUP BY
→ collapses rows

WINDOW FUNCTION
→ keeps rows
```

## 🧩 Generic Template

```sql
FUNCTION(column) OVER (
    PARTITION BY group_column
    ORDER BY sort_column
)
```

Examples:

```sql
SUM(amount) OVER (...)
AVG(amount) OVER (...)
ROW_NUMBER() OVER (...)
RANK() OVER (...)
LAG(value) OVER (...)
LEAD(value) OVER (...)
```

## 👀 Visual

```text
Rows remain:

A  10
A  20
A  30

Window calculation:
A total = 60

Output:

A 10 60
A 20 60
A 30 60
```

## 🔒 Invariant

The calculation can see the defined window, but the original row grain remains.

---

# 12. ⬅️ LAG / ➡️ LEAD

## 🧠 Mental Model

> "Look at another row relative to the current row."

## 🔎 Recognize

- Compare today vs yesterday
- Previous transaction
- Next event
- Change from previous value
- Consecutive records

## 🧩 Generic Template

```sql
LAG(value) OVER (
    PARTITION BY entity_id
    ORDER BY date
)
```

Example:

```sql
SELECT
    customer_id,
    date,
    amount,
    LAG(amount) OVER (
        PARTITION BY customer_id
        ORDER BY date
    ) AS previous_amount
FROM transactions;
```

## 👀 Visual

```text
Date       Amount     Previous
Jan 1       100        NULL
Jan 2       130        100
Jan 3       120        130
```

## 🔒 Invariant

Rows are ordered according to the specified window ordering before relative comparison.

---

# 13. 📈 Running Total

## 🧠 Mental Model

> "Each row should know the cumulative value up to that point."

## 🧩 Generic Template

```sql
SUM(amount) OVER (
    PARTITION BY customer_id
    ORDER BY date
    ROWS BETWEEN UNBOUNDED PRECEDING
         AND CURRENT ROW
) AS running_total
```

## 👀 Visual

```text
Amount       Running Total

10               10
20               30
15               45
25               70
```

## 🔎 Recognize

- Cumulative sales
- Account balance
- Running count
- Cumulative points

## 🔒 Invariant

The running value represents all rows from the beginning of the partition through the current row.

---

# 14. 📊 Moving Average

## 🧠 Mental Model

> "Calculate an aggregate over a moving window of neighboring rows."

## 🧩 Generic Template

```sql
AVG(value) OVER (
    ORDER BY date
    ROWS BETWEEN 2 PRECEDING
         AND CURRENT ROW
)
```

This is a 3-row moving average.

## 👀 Visual

```text
10  20  30  40  50
└──────┘
window

    └──────┘
    window
```

## 🔎 Recognize

- Rolling average
- Last 7 days
- Previous N records
- Moving statistics

## ⚠️ Important

Know whether the problem means:
- Previous N rows
- Previous N days
- Calendar time interval

These are not always the same.

---

# 15. 🧩 Conditional Aggregation

## 🧠 Mental Model

> "Count or sum only the rows satisfying a condition."

## 🧩 Generic Template

```sql
SUM(
    CASE
        WHEN condition THEN 1
        ELSE 0
    END
)
```

Or:

```sql
COUNT(
    CASE
        WHEN condition THEN 1
    END
)
```

## 💡 Example

```sql
SELECT
    department,
    SUM(CASE WHEN salary >= 100000 THEN 1 ELSE 0 END) AS high_earners,
    SUM(CASE WHEN salary < 100000 THEN 1 ELSE 0 END) AS other_earners
FROM employees
GROUP BY department;
```

## 🔎 Recognize

- Count active/inactive
- Count successful/failed
- Pivot-like questions
- Multiple categories in one row

## 🔒 Invariant

Each row contributes only to the aggregate(s) whose condition it satisfies.

---

# 16. ❓ CASE WHEN

## 🧠 Mental Model

> "Transform a value based on conditions."

## 🧩 Generic Template

```sql
CASE
    WHEN condition1 THEN result1
    WHEN condition2 THEN result2
    ELSE result3
END
```

## 💡 Example

```sql
SELECT
    name,
    CASE
        WHEN salary >= 150000 THEN 'High'
        WHEN salary >= 100000 THEN 'Medium'
        ELSE 'Low'
    END AS salary_band
FROM employees;
```

## 🔎 Recognize

- Categorization
- Conditional labels
- Bucketing
- Conditional calculations

---

# 17. 🔍 EXISTS

## 🧠 Mental Model

> "I don't need the matching rows. I only need to know whether at least one exists."

## 🧩 Generic Template

```sql
SELECT *
FROM customers c
WHERE EXISTS (
    SELECT 1
    FROM orders o
    WHERE o.customer_id = c.customer_id
);
```

## 🔎 Recognize

Words like:
- Has at least one
- There exists
- Customers who have...
- Products that have...

## 👀 Visual

```text
Customer
   │
   ├── order? YES → keep
   └── order? NO  → discard
```

## 🔒 Invariant

The outer row survives if the subquery finds at least one matching row.

## ⚠️ Advantage

`EXISTS` expresses existence directly and avoids accidentally multiplying outer rows through a one-to-many join.

---

# 18. 🚫 NOT EXISTS

## 🧠 Mental Model

> "Keep the row only if no matching row exists."

## 🧩 Generic Template

```sql
SELECT *
FROM customers c
WHERE NOT EXISTS (
    SELECT 1
    FROM orders o
    WHERE o.customer_id = c.customer_id
);
```

## 🔎 Recognize

- Never
- Has no
- Customers without orders
- Products never purchased
- Employees with no projects

## 🔒 Invariant

No matching row exists for the outer entity.

---

# 19. 🕳️ LEFT JOIN + IS NULL

## 🧠 Mental Model

> "Keep the left row when the right-side match doesn't exist."

## 🧩 Generic Template

```sql
SELECT a.*
FROM table_a a
LEFT JOIN table_b b
    ON a.id = b.a_id
WHERE b.a_id IS NULL;
```

## 🔎 Recognize

Same family as:

```sql
NOT EXISTS
```

## ⚠️ Important

`NOT EXISTS` and `LEFT JOIN ... IS NULL` can express similar anti-join logic, but understand the join key and NULL behavior before choosing one.

---

# 20. 🧱 Common Table Expressions — CTE

## 🧠 Mental Model

> "Break a complicated query into named intermediate steps."

## 🧩 Generic Template

```sql
WITH step1 AS (
    SELECT ...
),

step2 AS (
    SELECT ...
    FROM step1
)

SELECT *
FROM step2;
```

## 🔎 Recognize

- Multi-step logic
- Need to reuse an intermediate result
- Ranking then filtering
- Aggregating then joining
- Query is becoming difficult to reason about

## 👀 Visual

```text
Raw table
   ↓
CTE 1
   ↓
CTE 2
   ↓
Final query
```

## 🔒 Invariant

Each CTE should have a clear meaning and known row grain.

## ⚠️ Pro Habit

Before writing each CTE, state:

> "One row in this CTE represents ______."

---

# 21. 🔁 Self JOIN

## 🧠 Mental Model

> "The same table contains two roles, so I join the table to itself."

## 🔎 Recognize

- Employee → manager
- Friend relationships
- Parent → child
- Compare rows within same table
- Employees earning more than managers

## 🧩 Generic Template

```sql
SELECT
    e.name,
    m.name AS manager_name
FROM employees e
JOIN employees m
    ON e.manager_id = m.employee_id;
```

## 👀 Visual

```text
employees

Employee
   │
   └──── manager_id ────→ Employee
```

The aliases represent different roles.

---

# 22. 🧮 Subquery

## 🧠 Mental Model

> "Solve a smaller query first, then use its result in the outer query."

## 🔎 Recognize

- Compare against an aggregate
- Need a derived value
- Need a temporary result
- Nested condition

## 🧩 Scalar Subquery

```sql
SELECT name
FROM employees
WHERE salary > (
    SELECT AVG(salary)
    FROM employees
);
```

## 👀 Visual

```text
Inner query
    ↓
Produces value
    ↓
Outer query uses it
```

## ⚠️ Common Mistake

Know whether the subquery returns:
- One value
- One column with many rows
- Multiple columns/rows

That determines whether you can use:

```sql
=
IN
EXISTS
JOIN
```

---

# 23. 📦 UNION vs UNION ALL

## 🧠 Mental Model

> "Combine the results of two compatible queries vertically."

## UNION

Removes duplicates:

```sql
SELECT city FROM customers
UNION
SELECT city FROM suppliers;
```

## UNION ALL

Keeps duplicates:

```sql
SELECT city FROM customers
UNION ALL
SELECT city FROM suppliers;
```

## 👀 Visual

```text
Query A
  ↓
  A
  B

Query B
  ↓
  B
  C

UNION
A
B
C

UNION ALL
A
B
B
C
```

## 🔒 Rule

Both queries need compatible:
- Number of columns
- Data types

---

# 24. 📅 Date Filtering

## 🧠 Mental Model

> "Convert the natural-language time requirement into an exact range."

## 🔎 Recognize

- Last 30 days
- Orders in 2025
- Same month
- Consecutive dates
- Latest transaction
- Monthly sales

## 🧩 Generic Range Pattern

For a timestamp column:

```sql
WHERE timestamp_col >= '2025-01-01'
  AND timestamp_col <  '2026-01-01'
```

## ⚠️ Important

For timestamps, half-open intervals are often safer:

```text
>= start
< next_boundary
```

instead of trying to use:

```text
23:59:59
```

## 🔎 Date Pattern Questions

Ask:

> Is the problem based on calendar dates, timestamps, or row ordering?

These can require different solutions.

---

# 25. 🧮 Date Difference

## 🧠 Mental Model

> "Compare dates to detect duration, gaps, or consecutive activity."

SQL syntax differs by database.

Examples may use:

```sql
DATEDIFF(...)
```

or:

```sql
date1 - date2
```

or:

```sql
DATE_PART(...)
```

## 🔎 Recognize

- Consecutive days
- Days between events
- Time since previous event
- Retention windows

## 🧩 General Pattern

```sql
current_date - previous_date
```

The exact function depends on the SQL dialect.

---

# 26. 🔥 Consecutive Rows / Consecutive Dates

## 🧠 Mental Model

> "Compare each row with the previous row to determine whether the sequence continues."

## 🔎 Recognize

- Consecutive days
- Streaks
- Consecutive logins
- Winning streak
- Repeated activity

## 🧩 Generic Strategy

1. Partition by entity.
2. Order by date.
3. Use `LAG`.
4. Compare current and previous date.
5. Identify breaks.
6. Group the resulting islands.

Example:

```sql
WITH ordered AS (
    SELECT
        user_id,
        activity_date,
        LAG(activity_date) OVER (
            PARTITION BY user_id
            ORDER BY activity_date
        ) AS previous_date
    FROM activity
)

SELECT *
FROM ordered;
```

## 🔒 Invariant

Each row knows whether it continues the previous sequence.

---

# 27. 🏝️ Gaps and Islands

## 🧠 Mental Model

> "Turn a sequence into groups of consecutive rows."

This is one of the most important advanced SQL patterns.

## 🔎 Recognize

- Consecutive dates
- Streaks
- Group consecutive records
- Sessions
- Continuous ranges

## 🧩 Common Strategy

Create a group identifier using row number/date arithmetic.

For consecutive dates:

```sql
ROW_NUMBER() OVER (
    PARTITION BY user_id
    ORDER BY activity_date
)
```

Then combine it with date information to identify an island.

Another strategy:

```text
Current row
    ↓
Compare with previous row
    ↓
Is there a gap?
    ↓
If yes → start new group
```

## 👀 Visual

```text
Jan 1
Jan 2
Jan 3      ← Island 1

Jan 7
Jan 8      ← Island 2

Jan 20     ← Island 3
```

## 🔒 Invariant

Every row receives the identifier of the consecutive sequence it belongs to.

---

# 28. 🧑‍🤝‍🧑 Relational Division

## 🧠 Mental Model

> "Find entities that satisfy ALL required conditions."

## 🔎 Recognize

- Customers who bought every product
- Students who completed every required course
- Employees with all required skills

## 🧩 GROUP BY Strategy

Conceptually:

```sql
SELECT customer_id
FROM purchases
WHERE product_id IN (...)
GROUP BY customer_id
HAVING COUNT(DISTINCT product_id) = required_count;
```

## 🧩 NOT EXISTS Strategy

Ask:

> "Does there exist a required item that this entity does NOT have?"

If no such item exists, the entity qualifies.

## 🔒 Invariant

The entity has satisfied every required condition.

---

# 29. 🧹 Duplicate Detection

## 🧠 Mental Model

> "Group by the columns that define uniqueness, then find groups occurring more than once."

## 🧩 Generic Template

```sql
SELECT
    column1,
    column2,
    COUNT(*) AS cnt
FROM table_name
GROUP BY
    column1,
    column2
HAVING COUNT(*) > 1;
```

## 🔎 Recognize

- Duplicate emails
- Duplicate records
- Repeated combinations
- Find duplicate values

## 🔒 Invariant

Every returned group appears more than once.

---

# 30. 🧮 NULL Handling

## 🧠 Mental Model

> "NULL means unknown/missing, not zero and not an ordinary value."

## ❌ Wrong

```sql
WHERE column = NULL
```

## ✅ Correct

```sql
WHERE column IS NULL
```

or:

```sql
WHERE column IS NOT NULL
```

## COALESCE

Replace NULL with a fallback:

```sql
COALESCE(column, 0)
```

Example:

```sql
SELECT
    customer_id,
    COALESCE(total_spent, 0)
FROM ...
```

## 🔎 Recognize

- Missing values
- No matching records
- Zero activity
- Optional fields

## ⚠️ Critical

SQL uses three-valued logic:

```text
TRUE
FALSE
UNKNOWN
```

NULL comparisons often produce `UNKNOWN`.

---

# 31. 🔄 COALESCE

## 🧠 Mental Model

> "Use the first non-NULL value."

## 🧩 Template

```sql
COALESCE(value1, value2, value3)
```

Example:

```sql
COALESCE(phone, email, 'No contact')
```

## 🔎 Recognize

- Replace missing values
- Default values
- Fallback columns
- LEFT JOIN with missing matches

---

# 32. 🔢 COUNT(*) vs COUNT(column)

## COUNT(*)

Counts rows:

```sql
COUNT(*)
```

NULL does not matter.

## COUNT(column)

Counts non-NULL values:

```sql
COUNT(column)
```

## COUNT(DISTINCT column)

Counts unique non-NULL values:

```sql
COUNT(DISTINCT column)
```

## 👀 Example

```text
value
-----
10
20
NULL
20
```

Results:

```text
COUNT(*)              = 4
COUNT(value)          = 3
COUNT(DISTINCT value) = 2
```

This distinction appears constantly in SQL interviews.

---

# 33. 🎭 SQL Execution Order

A powerful mental model:

```text
FROM
  ↓
JOIN
  ↓
WHERE
  ↓
GROUP BY
  ↓
HAVING
  ↓
SELECT
  ↓
DISTINCT
  ↓
ORDER BY
  ↓
LIMIT
```

This is the **logical** processing order, not necessarily the physical execution plan used by the database engine.

## Why It Matters

It explains why:

```sql
WHERE COUNT(*) > 5
```

doesn't work.

The aggregate doesn't exist yet at the logical `WHERE` stage.

Use:

```sql
HAVING COUNT(*) > 5
```

---

# 34. 🪜 SQL Query Construction Order

When writing a query, think in this order:

## Step 1 — Define the grain

> One output row = ______.

## Step 2 — FROM

Which table is the base?

## Step 3 — JOIN

What other information is required?

## Step 4 — WHERE

Which raw rows should survive?

## Step 5 — GROUP BY

Do I need to collapse rows?

## Step 6 — Aggregation

What should I calculate?

## Step 7 — HAVING

Which groups survive?

## Step 8 — Window functions

Do I need calculations across rows without collapsing them?

## Step 9 — SELECT

What should I return?

## Step 10 — ORDER BY / LIMIT

How should the final result be presented?

---

# 35. 🧠 GROUP BY vs Window Functions

This distinction is essential.

## GROUP BY

```sql
SELECT
    department,
    AVG(salary)
FROM employees
GROUP BY department;
```

Output:

```text
department | avg_salary
-----------|-----------
A          | 100000
B          | 120000
```

Rows are collapsed.

## Window Function

```sql
SELECT
    name,
    department,
    salary,
    AVG(salary) OVER (
        PARTITION BY department
    ) AS dept_avg
FROM employees;
```

Output:

```text
name | dept | salary | dept_avg
-----|------|--------|---------
A    | X    | 100    | 120
B    | X    | 140    | 120
C    | X    | 120    | 120
```

Rows remain.

## 🧠 Rule

> **Need fewer rows? → GROUP BY**

> **Need the original rows plus context? → Window Function**

---

# 36. 🧠 JOIN vs EXISTS

## JOIN

Use when you need columns from the other table.

```sql
SELECT c.name, o.amount
FROM customers c
JOIN orders o
    ON c.id = o.customer_id;
```

## EXISTS

Use when you only care whether a match exists.

```sql
SELECT c.name
FROM customers c
WHERE EXISTS (
    SELECT 1
    FROM orders o
    WHERE o.customer_id = c.id
);
```

## 🔑 Question

> Do I need data from the matching table, or only proof that a match exists?

---

# 37. 🧠 WHERE vs HAVING

## WHERE

Filters rows:

```sql
WHERE salary > 100000
```

## HAVING

Filters groups:

```sql
HAVING AVG(salary) > 100000
```

## Visual

```text
Rows
 ↓
WHERE
 ↓
Rows
 ↓
GROUP BY
 ↓
Groups
 ↓
HAVING
 ↓
Groups
```

---

# 38. 🧠 RANK vs DENSE_RANK vs ROW_NUMBER

| Function | Ties | Gaps |
|---|---|---|
| `ROW_NUMBER()` | Different numbers | No gaps |
| `RANK()` | Same rank | Yes |
| `DENSE_RANK()` | Same rank | No |

Example:

```text
Scores: 100, 100, 90
```

```text
ROW_NUMBER:   1, 2, 3
RANK:         1, 1, 3
DENSE_RANK:   1, 1, 2
```

## 🔑 Recognition

If the question says:

> "Nth highest distinct value"

Think:

```sql
DENSE_RANK()
```

---

# 39. 🏗️ Advanced Pattern — Latest Row Per Entity

## 🧠 Mental Model

> "Rank each entity's rows from newest to oldest, then keep rank 1."

## 🧩 Template

```sql
WITH ranked AS (
    SELECT
        *,
        ROW_NUMBER() OVER (
            PARTITION BY customer_id
            ORDER BY created_at DESC
        ) AS rn
    FROM orders
)

SELECT *
FROM ranked
WHERE rn = 1;
```

## 🔎 Recognize

- Latest order per customer
- Most recent status
- Current record
- Last transaction

## 🔒 Invariant

`rn = 1` represents the newest row within each entity.

---

# 40. 🏗️ Advanced Pattern — Compare to Previous Row

## 🧩 Template

```sql
WITH ordered AS (
    SELECT
        *,
        LAG(value) OVER (
            PARTITION BY entity_id
            ORDER BY event_time
        ) AS previous_value
    FROM events
)

SELECT
    *,
    value - previous_value AS change
FROM ordered;
```

## 🔎 Recognize

- Growth
- Decline
- Difference
- Previous transaction
- Day-over-day change

---

# 41. 🏗️ Advanced Pattern — Conditional Existence

## 🧠 Mental Model

> "Keep an entity if it satisfies one condition and does not violate another."

## Example Structure

```sql
SELECT c.customer_id
FROM customers c
WHERE EXISTS (
    SELECT 1
    FROM orders o
    WHERE o.customer_id = c.customer_id
      AND o.status = 'completed'
)
AND NOT EXISTS (
    SELECT 1
    FROM orders o
    WHERE o.customer_id = c.customer_id
      AND o.status = 'cancelled'
);
```

## 🔎 Recognize

- Has A but not B
- Completed but never cancelled
- Bought X but not Y

---

# 42. 🏗️ Advanced Pattern — Self Comparison

## 🧠 Mental Model

> "Compare two rows from the same table by giving the table two aliases."

## Example

```sql
SELECT
    e.name
FROM employees e
JOIN employees m
    ON e.manager_id = m.employee_id
WHERE e.salary > m.salary;
```

## 🔎 Recognize

- Employee vs manager
- Current vs previous record
- Same-table relationships
- Pairs within one table

---

# 43. 🌲 Recursive CTE

## 🧠 Mental Model

> "Repeatedly follow a relationship until there are no more descendants."

## 🔎 Recognize

- Organizational hierarchy
- Parent-child tree
- Categories
- Folder structures
- Ancestors/descendants
- Graph-like recursive relationships

## 🧩 Generic Structure

```sql
WITH RECURSIVE hierarchy AS (

    -- Anchor
    SELECT ...
    FROM table
    WHERE root_condition

    UNION ALL

    -- Recursive step
    SELECT ...
    FROM table t
    JOIN hierarchy h
        ON t.parent_id = h.id
)

SELECT *
FROM hierarchy;
```

## 👀 Visual

```text
CEO
├── Director
│   ├── Manager
│   └── Manager
└── Director
    └── Manager
```

## 🔒 Invariant

Each recursive iteration expands the hierarchy by one level or according to the recursive relationship.

---

# 44. 🚨 Common SQL Mistakes

## Mistake 1 — Wrong grain

You return multiple rows when the question expects one row per customer.

### Fix

Ask:

> One row represents what?

---

## Mistake 2 — Accidental row multiplication

Joining a one-to-many table can multiply rows.

```text
Customer
  ↓
3 orders
```

A join can produce 3 customer rows.

### Fix

Understand cardinality before aggregating.

---

## Mistake 3 — WHERE instead of HAVING

Wrong:

```sql
WHERE COUNT(*) > 3
```

Correct:

```sql
HAVING COUNT(*) > 3
```

---

## Mistake 4 — Incorrect NULL comparison

Wrong:

```sql
WHERE x = NULL
```

Correct:

```sql
WHERE x IS NULL
```

---

## Mistake 5 — COUNT(column) vs COUNT(*)

Remember:

```text
COUNT(*) → rows
COUNT(x) → non-NULL x
COUNT(DISTINCT x) → unique non-NULL x
```

---

## Mistake 6 — LEFT JOIN accidentally becoming INNER JOIN

Be careful with conditions on the right table in `WHERE`.

---

## Mistake 7 — Using LIMIT for an Nth distinct value

If ties matter, consider:

```sql
DENSE_RANK()
```

---

## Mistake 8 — Forgetting tie behavior

Ask:

> What happens if two rows have the same value?

---

## Mistake 9 — Filtering too late

Filtering before an expensive aggregation/join can often reduce the data being processed.

But always prioritize correctness first; optimization depends on the database and execution plan.

---

## Mistake 10 — Assuming SQL dialects are identical

LeetCode may use a particular SQL dialect/version.

Functions for:
- dates
- string operations
- formatting
- intervals

can differ between MySQL, PostgreSQL, SQL Server, Oracle, etc.

---

# 45. ⚡ SQL Complexity Thinking

SQL doesn't have a single simple Big-O model like a typical array algorithm.

Think about:

- Number of rows
- Join cardinality
- Index availability
- Sorts
- Grouping
- Window operations
- Aggregation
- Query plan

## Conceptual expensive operations

Often pay attention to:

```text
Large JOINs
Sorting
GROUP BY
DISTINCT
Window functions
Correlated subqueries
```

## ⚠️ Important

Don't assume:

> "CTE = slower"

or:

> "JOIN = slower than EXISTS"

The optimizer and database engine matter.

For real systems, inspect:

```text
EXPLAIN
EXPLAIN ANALYZE
```

when available.

---

# 46. 🧠 SQL Index Mental Model

## 🧠 Mental Model

> "An index is an auxiliary structure that can help the database find rows without scanning the entire table."

Common candidates:
- Join keys
- Frequently filtered columns
- Frequently sorted columns
- Unique keys

But indexes have costs:
- Storage
- Insert/update/delete overhead
- Maintenance
- Possible poor selectivity

## 🔑 Interview Question

If asked:

> "How would you optimize this query?"

Think:

```text
1. Check execution plan
2. Check indexes
3. Check join cardinality
4. Filter early when appropriate
5. Avoid unnecessary columns/rows
6. Avoid accidental duplication
7. Consider aggregation strategy
```

---

# 47. 🧭 SQL Pattern Decision Tree

```text
START
  │
  ▼
Need to filter rows?
  │
  ├── YES → WHERE
  │
  ▼
Need another table?
  │
  ├── YES → JOIN
  │
  ▼
Need unmatched left rows?
  │
  ├── YES → LEFT JOIN
  │
  ▼
Need one row per group?
  │
  ├── YES → GROUP BY
  │
  ▼
Need aggregate?
  │
  ├── YES → COUNT/SUM/AVG/MIN/MAX
  │
  ▼
Need filter on aggregate?
  │
  ├── YES → HAVING
  │
  ▼
Need ranking?
  │
  ├── YES → RANK / DENSE_RANK / ROW_NUMBER
  │
  ▼
Need previous/next row?
  │
  ├── YES → LAG / LEAD
  │
  ▼
Need running calculation?
  │
  ├── YES → Window Function
  │
  ▼
Need existence?
  │
  ├── YES → EXISTS
  │
  ▼
Need absence?
  │
  ├── YES → NOT EXISTS / LEFT JOIN IS NULL
  │
  ▼
Need repeated hierarchy?
  │
  ├── YES → Recursive CTE
  │
  ▼
Need consecutive sequences?
  │
  ├── YES → LAG + Gaps & Islands
  │
  ▼
Need complex intermediate steps?
  │
  └── YES → CTE
```

---

# 48. 🧠 SQL Master Mental Models

## Model 1 — Grain

> **One output row represents ______.**

This should become automatic.

## Model 2 — Row vs Group

Ask:

> Am I filtering individual rows or groups?

```text
Row → WHERE
Group → HAVING
```

## Model 3 — Collapse vs Preserve

Ask:

> Do I want to collapse rows?

```text
YES → GROUP BY
NO  → Window Function
```

## Model 4 — Match vs Existence

Ask:

> Do I need columns from the other table?

```text
YES → JOIN
NO  → EXISTS
```

## Model 5 — Current vs Neighbor

Ask:

> Do I need another row relative to this row?

```text
Previous → LAG
Next     → LEAD
```

## Model 6 — Rank Within Group

Ask:

> Does ranking restart for every category?

```sql
PARTITION BY category
```

## Model 7 — Sequence

Ask:

> Are rows consecutive?

Think:

```text
LAG
↓
Detect breaks
↓
Create groups
↓
Aggregate islands
```

---

# 49. 📝 SQL Problem Review Template

After solving every problem, record:

## Problem

**Name:**  
**Difficulty:**  
**Pattern:**  

## 1. What is the table grain?

## 2. What should one output row represent?

## 3. What is the base table?

## 4. What joins are required?

## 5. What rows need to be filtered?

## 6. Do I need GROUP BY?

## 7. What aggregation is required?

## 8. Do I need a window function?

## 9. Do ties matter?

## 10. How should NULL behave?

## 11. What was my first approach?

## 12. What was wrong/slow about it?

## 13. What pattern solved it?

## 14. What is the invariant?

## 15. What edge cases exist?

## 16. How could this query be optimized?

## 17. Can I reproduce the query without looking?

---

# 50. 🏆 SQL Interview Pattern Progression

## Phase 1 — Foundations

1. SELECT
2. WHERE
3. DISTINCT
4. ORDER BY
5. LIMIT
6. CASE WHEN
7. NULL
8. COALESCE

## Phase 2 — Aggregation

9. GROUP BY
10. COUNT
11. SUM
12. AVG
13. MIN/MAX
14. HAVING
15. Conditional aggregation

## Phase 3 — Relationships

16. INNER JOIN
17. LEFT JOIN
18. Self JOIN
19. EXISTS
20. NOT EXISTS
21. Subqueries

## Phase 4 — Window Functions

22. ROW_NUMBER
23. RANK
24. DENSE_RANK
25. LAG
26. LEAD
27. Running totals
28. Moving averages
29. Top N per group

## Phase 5 — Advanced Patterns

30. CTEs
31. Latest row per entity
32. Consecutive dates
33. Gaps and islands
34. Relational division
35. Recursive CTE
36. Complex multi-step transformations

## Phase 6 — Performance

37. Indexes
38. Execution plans
39. Join cardinality
40. Query optimization
41. Partitioning concepts
42. Avoiding unnecessary scans/sorts

---

# 51. 🎯 Final SQL Mental Model

When you get a SQL problem, don't immediately start writing SQL.

Ask:

> **What does one output row represent?**

Then:

> **Where does that row come from?**

Then:

> **Which rows should survive?**

Then:

> **Do I need to combine tables?**

Then:

> **Do I need to collapse rows into groups?**

Then:

> **Do I need context from neighboring/related rows without collapsing them?**

Then choose:

```text
WHERE
JOIN
GROUP BY
HAVING
WINDOW
EXISTS
CTE
```

Finally ask:

> **What happens with duplicates, NULLs, ties, and missing relationships?**

That is the foundation for becoming strong at SQL interviews and SQL LeetCode problems.
