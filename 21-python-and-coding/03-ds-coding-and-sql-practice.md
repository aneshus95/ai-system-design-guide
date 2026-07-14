# Data Science Coding & SQL — Practice Questions

A practice bank for **Python (data-science / pandas)** and **SQL** coding interviews. Attempt each question before opening the solution. Assume `import pandas as pd`, `import numpy as np`. For SQL, examples use Postgres-flavored syntax.

## Table of Contents

- [Python — Fundamentals with a DS Flavor](#python--fundamentals-with-a-ds-flavor)
- [Python — Pandas Data Manipulation](#python--pandas-data-manipulation)
- [Python — NumPy & Vectorization](#python--numpy--vectorization)
- [Python — Implement DS Functions from Scratch](#python--implement-ds-functions-from-scratch)
- [SQL — Core (SELECT / Filter / Aggregate)](#sql--core-select--filter--aggregate)
- [SQL — Joins](#sql--joins)
- [SQL — Subqueries](#sql--subqueries)
- [SQL — Window Functions](#sql--window-functions)
- [SQL — Dates & Advanced Patterns](#sql--dates--advanced-patterns)
- [How to Approach These in the Interview](#how-to-approach-these-in-the-interview)
- [References](#references)

---

## Python — Fundamentals with a DS Flavor

**P1. Count word frequencies in a list and return the top 3.**
<details><summary>Solution</summary>

```python
from collections import Counter
def top3(words):
    return Counter(words).most_common(3)     # [('a', 5), ('b', 3), ('c', 2)]
```
`Counter` is the idiomatic tool; `most_common(k)` returns the k highest by count.
</details>

**P2. Given two lists, return elements in the first not in the second (set difference), preserving order.**
<details><summary>Solution</summary>

```python
def diff(a, b):
    bset = set(b)                 # O(1) lookups
    return [x for x in a if x not in bset]
```
Convert `b` to a set first — using `x in b` on a list is O(n) per check → O(n²) total.
</details>

**P3. Flatten a list of lists and dedupe, keeping first-seen order.**
<details><summary>Solution</summary>

```python
def flatten_unique(lol):
    seen, out = set(), []
    for sub in lol:
        for x in sub:
            if x not in seen:
                seen.add(x); out.append(x)
    return out
```
`dict.fromkeys` trick also works: `list(dict.fromkeys(x for sub in lol for x in sub))`.
</details>

**P4. Given a dict of `{student: [scores]}`, return `{student: average}` for students whose average ≥ 60.**
<details><summary>Solution</summary>

```python
def passing(d):
    return {s: sum(v)/len(v) for s, v in d.items() if sum(v)/len(v) >= 60}
```
Comprehension with a filter; guard against empty lists in real code.
</details>

**P5. Write a generator that yields a moving average of window k over a stream of numbers.**
<details><summary>Solution</summary>

```python
from collections import deque
def moving_avg(stream, k):
    win = deque(maxlen=k)
    for x in stream:
        win.append(x)
        if len(win) == k:
            yield sum(win) / k
```
`deque(maxlen=k)` auto-evicts the oldest — clean O(1) sliding window.
</details>

---

## Python — Pandas Data Manipulation

*(Assume a DataFrame `df` with columns as described in each question.)*

**P6. `df[customer, order_amount]` — total and average order amount per customer, sorted by total desc.**
<details><summary>Solution</summary>

```python
(df.groupby('customer')['order_amount']
   .agg(total='sum', avg='mean')
   .sort_values('total', ascending=False)
   .reset_index())
```
Named aggregation (`agg(total='sum', ...)`) gives clean column names in one pass.
</details>

**P7. Find the top 2 highest-paid employees **per department** (`df[emp, dept, salary]`).**
<details><summary>Solution</summary>

```python
(df.sort_values('salary', ascending=False)
   .groupby('dept')
   .head(2))
# or with rank:
df[df.groupby('dept')['salary'].rank(method='first', ascending=False) <= 2]
```
`groupby(...).head(n)` after sorting is the cleanest top-N-per-group.
</details>

**P8. Fill missing values in `age` with the **median age of that person's `dept`**.**
<details><summary>Solution</summary>

```python
df['age'] = df.groupby('dept')['age'].transform(lambda s: s.fillna(s.median()))
```
`transform` returns a series aligned to the original index — key for group-wise fills. (Doing `.median()` alone would collapse groups.)
</details>

**P9. Merge `orders[order_id, customer_id, amount]` with `customers[customer_id, name]`, keeping **all** orders even if the customer is missing.**
<details><summary>Solution</summary>

```python
merged = orders.merge(customers, on='customer_id', how='left')
```
`how='left'` keeps every order; unmatched customers get NaN name. (`how='inner'` would silently drop orders.)
</details>

**P10. From `df[date, sales]` (daily), compute a 7-day rolling average of sales.**
<details><summary>Solution</summary>

```python
df = df.sort_values('date')
df['ma7'] = df['sales'].rolling(window=7).mean()
```
Sort by date first. `rolling(7).mean()` gives NaN for the first 6 rows (not enough history) — use `min_periods=1` if you want partial windows.
</details>

**P11. Given `df[user_id, event, timestamp]`, count events per user per day.**
<details><summary>Solution</summary>

```python
df['day'] = pd.to_datetime(df['timestamp']).dt.date
counts = df.groupby(['user_id', 'day']).size().reset_index(name='event_count')
```
`.size()` counts rows per group; `dt.date` buckets timestamps to day.
</details>

**P12. Remove duplicate rows by `email`, keeping the row with the **most recent** `signup_date`.**
<details><summary>Solution</summary>

```python
(df.sort_values('signup_date')
   .drop_duplicates(subset='email', keep='last'))
```
Sort ascending, keep `last` = most recent. (Mirrors the SQL `ROW_NUMBER()` dedup pattern.)
</details>

**P13. Pivot `df[region, quarter, revenue]` so quarters become columns.**
<details><summary>Solution</summary>

```python
df.pivot_table(index='region', columns='quarter', values='revenue', aggfunc='sum').reset_index()
```
`pivot_table` (not `pivot`) handles duplicate index/column pairs by aggregating.
</details>

**P14. Compute month-over-month % change in `revenue` from `df[month, revenue]` (one row per month).**
<details><summary>Solution</summary>

```python
df = df.sort_values('month')
df['mom_pct'] = df['revenue'].pct_change() * 100
```
`pct_change()` = (current − previous) / previous; equivalent to SQL `LAG`.
</details>

---

## Python — NumPy & Vectorization

**P15. Given a NumPy array, replace all negative values with 0 (vectorized, no loop).**
<details><summary>Solution</summary>

```python
arr = np.where(arr < 0, 0, arr)      # or: arr[arr < 0] = 0
```
`np.where(cond, a, b)` is the vectorized if/else; boolean masking also works in place.
</details>

**P16. Standardize (z-score) each column of a 2-D array.**
<details><summary>Solution</summary>

```python
z = (arr - arr.mean(axis=0)) / arr.std(axis=0)
```
`axis=0` computes per-column stats; **broadcasting** subtracts/divides row-wise automatically.
</details>

**P17. Compute the Euclidean distance between two vectors (no loop).**
<details><summary>Solution</summary>

```python
dist = np.sqrt(np.sum((a - b) ** 2))   # or np.linalg.norm(a - b)
```
`np.linalg.norm(a - b)` is the idiomatic one-liner.
</details>

---

## Python — Implement DS Functions from Scratch

**P18. Implement precision, recall, and F1 from lists of true and predicted labels (binary, 0/1).**
<details><summary>Solution</summary>

```python
def prf1(y_true, y_pred):
    tp = sum(t == 1 and p == 1 for t, p in zip(y_true, y_pred))
    fp = sum(t == 0 and p == 1 for t, p in zip(y_true, y_pred))
    fn = sum(t == 1 and p == 0 for t, p in zip(y_true, y_pred))
    precision = tp / (tp + fp) if tp + fp else 0.0
    recall    = tp / (tp + fn) if tp + fn else 0.0
    f1 = 2 * precision * recall / (precision + recall) if precision + recall else 0.0
    return precision, recall, f1
```
Guard every denominator against zero. (See [Classification Metrics](../20-machine-learning-foundations/06-classification-metrics.md).)
</details>

**P19. Implement a train/test split (shuffle + split) without sklearn.**
<details><summary>Solution</summary>

```python
def train_test_split(X, y, test_size=0.2, seed=42):
    rng = np.random.default_rng(seed)
    idx = rng.permutation(len(X))
    cut = int(len(X) * (1 - test_size))
    tr, te = idx[:cut], idx[cut:]
    return X[tr], X[te], y[tr], y[te]
```
Shuffle indices, then slice — seed for reproducibility. (For classification you'd stratify.)
</details>

**P20. Compute RMSE between predictions and actuals.**
<details><summary>Solution</summary>

```python
def rmse(y_true, y_pred):
    y_true, y_pred = np.array(y_true), np.array(y_pred)
    return np.sqrt(np.mean((y_true - y_pred) ** 2))
```
</details>

**P21. Remove outliers from a list using the 1.5×IQR rule.**
<details><summary>Solution</summary>

```python
def drop_outliers(x):
    x = np.array(x)
    q1, q3 = np.percentile(x, [25, 75])
    iqr = q3 - q1
    lo, hi = q1 - 1.5 * iqr, q3 + 1.5 * iqr
    return x[(x >= lo) & (x <= hi)]
```
</details>

**P22. Cosine similarity between two vectors.**
<details><summary>Solution</summary>

```python
def cosine(a, b):
    a, b = np.array(a), np.array(b)
    return np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b))
```
Guard against zero-norm vectors in production.
</details>

---

## SQL — Core (SELECT / Filter / Aggregate)

*Tables:* `employees(id, name, dept, salary, hire_date, manager_id)`, `orders(id, customer_id, amount, order_date, status)`, `customers(id, name, country)`.

**S1. Average salary per department, only departments with more than 5 employees, highest first.**
<details><summary>Solution</summary>

```sql
SELECT dept, AVG(salary) AS avg_salary
FROM employees
GROUP BY dept
HAVING COUNT(*) > 5
ORDER BY avg_salary DESC;
```
`WHERE` filters rows before grouping; `HAVING` filters groups after. (See [SQL execution order](../22-data-engineering/02-sql-programming.md#the-key-intuition-query-execution-order).)
</details>

**S2. Count orders and total revenue per status.**
<details><summary>Solution</summary>

```sql
SELECT status, COUNT(*) AS n_orders, SUM(amount) AS revenue
FROM orders
GROUP BY status;
```
</details>

**S3. Customers who have never placed an order.**
<details><summary>Solution</summary>

```sql
SELECT c.id, c.name
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.id
WHERE o.id IS NULL;             -- anti-join
-- or: WHERE NOT EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id)
```
`LEFT JOIN ... WHERE right IS NULL` is the classic anti-join; `NOT EXISTS` is the NULL-safe alternative to `NOT IN`.
</details>

---

## SQL — Joins

**S4. For each order, show customer name and country (drop orders with no matching customer).**
<details><summary>Solution</summary>

```sql
SELECT o.id, o.amount, c.name, c.country
FROM orders o
JOIN customers c ON c.id = o.customer_id;   -- INNER JOIN
```
</details>

**S5. Total revenue per country (include countries with zero orders as 0).**
<details><summary>Solution</summary>

```sql
SELECT c.country, COALESCE(SUM(o.amount), 0) AS revenue
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.id
GROUP BY c.country;
```
`LEFT JOIN` keeps all countries; `COALESCE` turns NULL sums (no orders) into 0.
</details>

**S6. Self-join: list each employee with their manager's name.**
<details><summary>Solution</summary>

```sql
SELECT e.name AS employee, m.name AS manager
FROM employees e
LEFT JOIN employees m ON m.id = e.manager_id;
```
`LEFT JOIN` so the CEO (no manager) still appears with NULL.
</details>

---

## SQL — Subqueries

**S7. Employees earning above the company-wide average salary.**
<details><summary>Solution</summary>

```sql
SELECT name, salary
FROM employees
WHERE salary > (SELECT AVG(salary) FROM employees);
```
Scalar subquery — runs once.
</details>

**S8. Employees earning above **their own department's** average (correlated subquery).**
<details><summary>Solution</summary>

```sql
SELECT e.name, e.dept, e.salary
FROM employees e
WHERE e.salary > (SELECT AVG(salary) FROM employees WHERE dept = e.dept);
-- Better (avoids per-row re-run):
SELECT name, dept, salary FROM (
  SELECT *, AVG(salary) OVER (PARTITION BY dept) AS dept_avg FROM employees
) t WHERE salary > dept_avg;
```
The window-function version is usually faster than the correlated subquery.
</details>

---

## SQL — Window Functions

**S9. 2nd-highest salary (handling ties correctly).**
<details><summary>Solution</summary>

```sql
WITH r AS (SELECT salary, DENSE_RANK() OVER (ORDER BY salary DESC) AS rk FROM employees)
SELECT DISTINCT salary FROM r WHERE rk = 2;
```
`DENSE_RANK` (no gaps) is correct for "Nth highest distinct value"; `ROW_NUMBER` would miss ties.
</details>

**S10. Top 3 highest-paid employees per department.**
<details><summary>Solution</summary>

```sql
WITH ranked AS (
  SELECT *, ROW_NUMBER() OVER (PARTITION BY dept ORDER BY salary DESC) AS rn
  FROM employees
)
SELECT * FROM ranked WHERE rn <= 3;
```
Window functions can't go in `WHERE` — compute in a CTE, filter outside. (The canonical top-N-per-group pattern.)
</details>

**S11. Running total of daily revenue over time.**
<details><summary>Solution</summary>

```sql
SELECT order_date,
       SUM(amount) OVER (ORDER BY order_date) AS running_total
FROM daily_orders;
```
Adding `ORDER BY` inside `OVER` makes each row sum itself + all prior rows.
</details>

**S12. Month-over-month revenue change using LAG.**
<details><summary>Solution</summary>

```sql
SELECT month, revenue,
       revenue - LAG(revenue) OVER (ORDER BY month) AS mom_change
FROM monthly_revenue;
```
`LAG` pulls the previous row's value; the first row is NULL (no prior month).
</details>

**S13. 3-day moving average of sales (window frame).**
<details><summary>Solution</summary>

```sql
SELECT day, sales,
       AVG(sales) OVER (ORDER BY day ROWS BETWEEN 2 PRECEDING AND CURRENT ROW) AS ma3
FROM daily_sales;
```
`ROWS BETWEEN 2 PRECEDING AND CURRENT ROW` = a 3-row sliding window.
</details>

**S14. Remove duplicate users by email, keeping the earliest `id`.**
<details><summary>Solution</summary>

```sql
WITH d AS (SELECT *, ROW_NUMBER() OVER (PARTITION BY email ORDER BY id) AS rn FROM users)
DELETE FROM users WHERE id IN (SELECT id FROM d WHERE rn > 1);
```
</details>

---

## SQL — Dates & Advanced Patterns

**S15. Number of orders in the last 30 days per customer.**
<details><summary>Solution</summary>

```sql
SELECT customer_id, COUNT(*) AS recent_orders
FROM orders
WHERE order_date >= CURRENT_DATE - INTERVAL '30 days'
GROUP BY customer_id;
```
Keep the predicate **sargable** — filter the raw `order_date` column, don't wrap it in a function.
</details>

**S16. Daily active users (DAU) for the last 7 days.**
<details><summary>Solution</summary>

```sql
SELECT DATE(event_time) AS day, COUNT(DISTINCT user_id) AS dau
FROM events
WHERE event_time >= CURRENT_DATE - INTERVAL '7 days'
GROUP BY DATE(event_time)
ORDER BY day;
```
`COUNT(DISTINCT user_id)` — distinct users, not events.
</details>

**S17. Percentage of orders that are 'completed' per month.**
<details><summary>Solution</summary>

```sql
SELECT DATE_TRUNC('month', order_date) AS month,
       AVG((status = 'completed')::int) * 100 AS pct_completed
FROM orders
GROUP BY 1;
-- portable version:
-- SUM(CASE WHEN status='completed' THEN 1 ELSE 0 END)*100.0 / COUNT(*)
```
`AVG(boolean::int)` is a neat rate trick; the `CASE` version is portable across engines.
</details>

**S18. First order date per customer (cohort starter).**
<details><summary>Solution</summary>

```sql
SELECT customer_id, MIN(order_date) AS first_order
FROM orders
GROUP BY customer_id;
```
Foundation for cohort/retention analysis.
</details>

---

## How to Approach These in the Interview

- **Think aloud and clarify first** — ask about nulls, duplicates, ties, data types, and expected output shape before coding. Interviewers score your process, not just the answer.
- **State the plan, then write it.** For SQL: "I'll group by X, aggregate Y, then filter groups with HAVING." For pandas: name the operation (groupby-transform, merge how=left, etc.).
- **Watch the classic traps:** `WHERE` vs `HAVING`; `INNER` silently dropping rows (use `LEFT` when you need all); ties in ranking (`DENSE_RANK` vs `ROW_NUMBER`); `COUNT(*)` vs `COUNT(col)`; window functions can't sit in `WHERE`; pandas `transform` vs `agg`; sargable date filters.
- **Say the complexity** where relevant (set lookup O(1) vs list O(n); why you sorted first).
- **Test on a tiny example** — walk 2–3 rows through your logic out loud.

---

## References

- [SQL Programming (this guide)](../22-data-engineering/02-sql-programming.md) · [Python Core Concepts](01-python-core-concepts.md) · [Classification Metrics](../20-machine-learning-foundations/06-classification-metrics.md)
- [StrataScratch — DS SQL & Python questions](https://www.stratascratch.com/) · [DataLemur — SQL interview questions](https://datalemur.com/) · [LeetCode Database](https://leetcode.com/studyplan/top-sql-50/)
- [pandas documentation](https://pandas.pydata.org/docs/) · [PostgreSQL — SQL language](https://www.postgresql.org/docs/current/sql.html)

---

*Previous: [OOP in Python](02-oop-in-python.md) | Up: [Guide Home](../README.md)*
