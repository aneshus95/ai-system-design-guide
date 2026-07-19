# Data Science Coding & SQL — Practice Questions

A practice bank for **Python (data-science / pandas)** and **SQL** coding interviews. Attempt each question before opening the solution. Assume `import pandas as pd`, `import numpy as np`. For SQL, examples use Postgres-flavored syntax.

## Table of Contents

- [Sample Data — Run This First](#sample-data--run-this-first)
- [Python — Fundamentals with a DS Flavor](#python--fundamentals-with-a-ds-flavor)
- [Python — Pandas Data Manipulation](#python--pandas-data-manipulation)
- [Python — NumPy & Vectorization](#python--numpy--vectorization)
- [Python — Implement DS Functions from Scratch](#python--implement-ds-functions-from-scratch)
- [Python — Harder & Real-Company Patterns](#python--harder--real-company-patterns)
- [Python — Common DS Patterns (by Category)](#python--common-ds-patterns-by-category)
- [Python — Efficiency: Alternative Solutions](#python--efficiency-alternative-solutions)
- [Pandas `groupby` — Full Reference](#pandas-groupby--full-reference)
- [SQL — The 7 Recurring Patterns (~90% of Interviews)](#sql--the-7-recurring-patterns-90-of-interviews)
- [SQL — Core (SELECT / Filter / Aggregate)](#sql--core-select--filter--aggregate)
- [SQL — Joins](#sql--joins)
- [SQL — Subqueries](#sql--subqueries)
- [SQL — Window Functions](#sql--window-functions)
- [SQL — Dates & Advanced Patterns](#sql--dates--advanced-patterns)
- [SQL — Gaps & Islands and Sessionization](#sql--gaps--islands-and-sessionization)
- [SQL — Cohort & Retention](#sql--cohort--retention)
- [SQL — Recursive CTE & Hierarchy](#sql--recursive-cte--hierarchy)
- [SQL — Efficient Alternatives & Rewrites](#sql--efficient-alternatives--rewrites)
- [How to Approach These in the Interview](#how-to-approach-these-in-the-interview)
- [References](#references)

---

## Sample Data — Run This First

The same small dataset works for both the Python and SQL questions. It's deliberately seeded with edge cases: **salary ties** (for `DENSE_RANK`), **NULL ages** (median-fill), **a customer with no orders** (anti-join), **three orders on one day** (daily-sum), **consecutive login dates** (streaks), a **>30-min gap** (sessionization), and **price-change runs**.

### Python / pandas setup

```python
import pandas as pd, numpy as np

employees = pd.DataFrame({
    'id':[1,2,3,4,5,6,7,8,9,10],
    'name':['Alice','Bob','Carol','Dave','Eve','Frank','Grace','Heidi','Ivan','Judy'],
    'dept':['Eng','Eng','Eng','Eng','Sales','Sales','Sales','Sales','Sales','Sales'],
    'salary':[120,100,100,90,95,95,80,70,60,150],          # ties: Bob/Carol=100, Eve/Frank=95
    'hire_date':pd.to_datetime(['2020-01-15','2019-03-01','2021-06-20','2022-02-10','2018-07-30',
                                '2020-11-05','2021-01-25','2023-04-18','2019-09-09','2017-05-05']),
    'manager_id':[10,1,1,1,10,5,5,5,5,None],               # Judy (10) = CEO, no manager
    'age':[34,45,None,29,52,None,38,41,33,50]})            # NULLs → group median-fill (P8)

customers = pd.DataFrame({
    'id':[1,2,3,4,5],
    'name':['ACME','Globex','Initech','Umbrella','Stark'],
    'country':['US','US','IN','UK','US']})                 # Umbrella (4) has NO orders

orders = pd.DataFrame({
    'id':[101,102,103,104,105,106,107,108],
    'customer_id':[1,1,2,3,3,1,2,5],
    'amount':[250,120,300,80,220,90,150,400],
    'order_date':pd.to_datetime(['2019-02-05','2019-02-05','2019-03-10','2019-02-20',
                                 '2019-04-01','2019-02-05','2019-03-11','2019-04-15']),
    'status':['completed','completed','pending','completed','completed','refunded','completed','pending']})
# customer 1 has 3 orders on 2019-02-05 (250+120+90=460) → beats Stark's single 400 (P23)

logins = pd.DataFrame({
    'user_id':[1,1,1,1,1,2,2],
    'login_date':pd.to_datetime(['2024-01-01','2024-01-02','2024-01-03','2024-01-03',  # dup
                                 '2024-01-05','2024-01-01','2024-01-02'])})             # u1 streak 1-3, u2 1-2

events = pd.DataFrame({
    'user_id':[1,1,1,1,2,2],
    'event':['view','click','view','click','view','view'],
    'event_time':pd.to_datetime(['2024-01-01 10:00','2024-01-01 10:10','2024-01-01 10:50',
                                 '2024-01-01 11:00','2024-01-01 09:00','2024-01-01 09:20'])})  # 40-min gap → new session

price_history = pd.DataFrame({
    'product_id':[1,1,1,1,1],
    'day':pd.to_datetime(['2024-01-01','2024-01-02','2024-01-03','2024-01-04','2024-01-05']),
    'price':[10,10,12,12,10]})                             # runs: 10,10 | 12,12 | 10

activity = pd.DataFrame({
    'user_id':[1,1,1,2,2,3],
    'activity_date':pd.to_datetime(['2024-01-01','2024-01-02','2024-02-15',
                                    '2024-01-01','2024-03-01','2024-02-01'])})           # cohorts + day-1 retention

users = pd.DataFrame({
    'id':[1,2,3,4],
    'email':['a@x.com','b@x.com','a@x.com','b@x.com'],
    'signup_date':pd.to_datetime(['2024-01-01','2024-01-02','2024-02-01','2024-01-15'])})

trades = pd.DataFrame({'symbol':['AAPL','AAPL','MSFT'],
    'time':pd.to_datetime(['2024-01-01 09:31','2024-01-01 09:35','2024-01-01 09:32'])})
quotes = pd.DataFrame({'symbol':['AAPL','AAPL','MSFT','MSFT'],
    'time':pd.to_datetime(['2024-01-01 09:30','2024-01-01 09:34','2024-01-01 09:30','2024-01-01 09:33']),
    'price':[187.0,187.5,410.0,410.2]})

region_sales = pd.DataFrame({'region':['North','North','South','South','East','East'],
    'quarter':['Q1','Q2','Q1','Q2','Q1','Q2'], 'revenue':[100,120,90,110,70,80]})
monthly_revenue = pd.DataFrame({'month':pd.to_datetime(['2024-01-01','2024-02-01','2024-03-01','2024-04-01']),
    'revenue':[1000,1100,990,1200]})
daily_sales = pd.DataFrame({'day':pd.date_range('2024-01-01', periods=10),
    'sales':[100,120,110,130,90,140,150,160,120,130]})

# Convenience frames matching a few questions' shorthand column names:
sales = orders.rename(columns={'customer_id':'customer','amount':'order_amount'})   # P6
web_events = events.rename(columns={'event_time':'timestamp'})                       # P11
signups = users[['email','signup_date']]                                            # P12
tags_df = pd.DataFrame({'user':[1,2], 'tags':['python, sql, ml', 'sql, viz']})       # P27
```

> **Which frame does each `df` mean?** P6→`sales` · P7/P8/P25→`employees` · P9→`orders`+`customers` · P10/S13→`daily_sales` · P11→`web_events` · P12→`signups`/`users` · P13→`region_sales` · P14/S12→`monthly_revenue` · P23/P28→`orders` · P26→`trades`/`quotes` · P27→`tags_df` · P29/S19→`logins`.

### SQL setup (Postgres-flavored; runs on [db-fiddle](https://www.db-fiddle.com/) / [sqliteonline](https://sqliteonline.com/))

```sql
CREATE TABLE employees (id INT, name TEXT, dept TEXT, salary INT, hire_date DATE, manager_id INT, age INT);
INSERT INTO employees VALUES
 (1,'Alice','Eng',120,'2020-01-15',10,34),(2,'Bob','Eng',100,'2019-03-01',1,45),
 (3,'Carol','Eng',100,'2021-06-20',1,NULL),(4,'Dave','Eng',90,'2022-02-10',1,29),
 (5,'Eve','Sales',95,'2018-07-30',10,52),(6,'Frank','Sales',95,'2020-11-05',5,NULL),
 (7,'Grace','Sales',80,'2021-01-25',5,38),(8,'Heidi','Sales',70,'2023-04-18',5,41),
 (9,'Ivan','Sales',60,'2019-09-09',5,33),(10,'Judy','Sales',150,'2017-05-05',NULL,50);

CREATE TABLE customers (id INT, name TEXT, country TEXT);
INSERT INTO customers VALUES (1,'ACME','US'),(2,'Globex','US'),(3,'Initech','IN'),(4,'Umbrella','UK'),(5,'Stark','US');

CREATE TABLE orders (id INT, customer_id INT, amount INT, order_date DATE, status TEXT);
INSERT INTO orders VALUES
 (101,1,250,'2019-02-05','completed'),(102,1,120,'2019-02-05','completed'),
 (103,2,300,'2019-03-10','pending'),(104,3,80,'2019-02-20','completed'),
 (105,3,220,'2019-04-01','completed'),(106,1,90,'2019-02-05','refunded'),
 (107,2,150,'2019-03-11','completed'),(108,5,400,'2019-04-15','pending');

CREATE TABLE logins (user_id INT, login_date DATE);
INSERT INTO logins VALUES (1,'2024-01-01'),(1,'2024-01-02'),(1,'2024-01-03'),(1,'2024-01-03'),
 (1,'2024-01-05'),(2,'2024-01-01'),(2,'2024-01-02');

CREATE TABLE events (user_id INT, event TEXT, event_time TIMESTAMP);
INSERT INTO events VALUES (1,'view','2024-01-01 10:00'),(1,'click','2024-01-01 10:10'),
 (1,'view','2024-01-01 10:50'),(1,'click','2024-01-01 11:00'),
 (2,'view','2024-01-01 09:00'),(2,'view','2024-01-01 09:20');

CREATE TABLE price_history (product_id INT, day DATE, price INT);
INSERT INTO price_history VALUES (1,'2024-01-01',10),(1,'2024-01-02',10),(1,'2024-01-03',12),
 (1,'2024-01-04',12),(1,'2024-01-05',10);

CREATE TABLE activity (user_id INT, activity_date DATE);
INSERT INTO activity VALUES (1,'2024-01-01'),(1,'2024-01-02'),(1,'2024-02-15'),
 (2,'2024-01-01'),(2,'2024-03-01'),(3,'2024-02-01');

CREATE TABLE users (id INT, email TEXT, signup_date DATE);
INSERT INTO users VALUES (1,'a@x.com','2024-01-01'),(2,'b@x.com','2024-01-02'),
 (3,'a@x.com','2024-02-01'),(4,'b@x.com','2024-01-15');

CREATE TABLE daily_sales (day DATE, sales INT);
INSERT INTO daily_sales VALUES ('2024-01-01',100),('2024-01-02',120),('2024-01-03',110),
 ('2024-01-04',130),('2024-01-05',90),('2024-01-06',140),('2024-01-07',150),('2024-01-08',160);

CREATE TABLE monthly_revenue (month DATE, revenue INT);
INSERT INTO monthly_revenue VALUES ('2024-01-01',1000),('2024-02-01',1100),('2024-03-01',990),('2024-04-01',1200);
```

> **Where to run it:** Python — any notebook/Colab. SQL — paste into [db-fiddle.com](https://www.db-fiddle.com/) (pick PostgreSQL) or [sqliteonline.com](https://sqliteonline.com/); for SQLite, swap `INTERVAL '30 minutes'` for `julianday()` date math and `DATE_TRUNC` for `strftime`.

---

## Python — Fundamentals with a DS Flavor

**P1. Count word frequencies in a plain Python list and return the top 3 most common words, from most to least frequent.** You have a plain Python list of strings (e.g., `['apple', 'banana', 'apple', 'cherry', 'apple', 'banana']`). *Example:* `['apple'×3, 'banana'×2, 'cherry'×1]` → `[('apple',3), ('banana',2), ('cherry',1)]`.
<details><summary>Solution</summary>

```python
from collections import Counter
def top3(words):
    return Counter(words).most_common(3)     # [('a', 5), ('b', 3), ('c', 2)]
```
`Counter` is the idiomatic tool; `most_common(k)` returns the k highest by count.
</details>

**P2. Return elements from list A that are NOT in list B, preserving their original left-to-right order.** You have two plain Python lists. *Example:* `a=[1,2,3,4]`, `b=[2,4]` → `[1,3]`. (Hint: converting `b` to a set before checking makes each lookup O(1) instead of O(n).)
<details><summary>Solution</summary>

```python
def diff(a, b):
    bset = set(b)                 # O(1) lookups
    return [x for x in a if x not in bset]
```
Convert `b` to a set first — using `x in b` on a list is O(n) per check → O(n²) total.
</details>

**P3. Flatten a list-of-lists into one list, remove duplicates, and keep the first occurrence of each value.** You have a nested list like `[[1,2,2],[3,1],[4]]`. *Example:* `[[1,2,2],[3,1],[4]]` → `[1,2,3,4]` (second `2` and second `1` are dropped).
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

**P4. From a dict of `{student: [scores]}`, return a new dict with only the students whose score average is ≥ 60.** You have a dict like `{'Alice': [70,80,90], 'Bob': [40,50,55]}`. *Example:* Alice avg=80 (keep), Bob avg=48.3 (drop) → `{'Alice': 80.0}`.
<details><summary>Solution</summary>

```python
def passing(d):
    return {s: sum(v)/len(v) for s, v in d.items() if sum(v)/len(v) >= 60}
```
Comprehension with a filter; guard against empty lists in real code.
</details>

**P5. Write a generator that yields a rolling average of the last `k` numbers, starting once at least `k` numbers have been seen.** Given a stream of numbers and a window size `k`. *Example:* stream `[1,2,3,4,5]`, `k=3` → yields `2.0` (avg of 1,2,3), `3.0` (avg of 2,3,4), `4.0` (avg of 3,4,5).
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

**P6. Compute total and average order amount per customer, sorted by total descending.** DataFrame `sales` with columns `customer` and `order_amount`. *Example:* customer ACME with orders 250, 120, 90 → total=460, avg≈153.
<details><summary>Solution</summary>

```python
(df.groupby('customer')['order_amount']
   .agg(total='sum', avg='mean')
   .sort_values('total', ascending=False)
   .reset_index())
```
Named aggregation (`agg(total='sum', ...)`) gives clean column names in one pass.
</details>

**P7. Return the top 2 highest-paid employees within each department, keeping ties.** DataFrame `employees` with columns `name`, `dept`, and `salary`. *Example:* Eng has Alice(120), Bob(100), Carol(100), Dave(90) → keep Alice and Bob (or Carol — ties are fine).
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

**P8. Fill missing ages with the median age of each person's own department, not the overall median.** DataFrame `employees` with columns `dept` and `age` (some `age` values are `NaN`). *Example:* Eng ages 34, 45, NaN, 29 → median of known Eng ages is 34, so the NaN becomes 34.
<details><summary>Solution</summary>

```python
df['age'] = df.groupby('dept')['age'].transform(lambda s: s.fillna(s.median()))
```
`transform` returns a series aligned to the original index — key for group-wise fills. (Doing `.median()` alone would collapse groups.)
</details>

**P9. Left-join orders to customers so that every order is kept even if no matching customer exists, with NaN for missing customer columns.** DataFrame `orders` (columns `id`, `customer_id`, `amount`, `order_date`, `status`) merged with `customers` (columns `id`, `name`, `country`) on `customer_id`. *Example:* if an order references a deleted customer, it still shows up with a null name.
<details><summary>Solution</summary>

```python
merged = orders.merge(customers, on='customer_id', how='left')
```
`how='left'` keeps every order; unmatched customers get NaN name. (`how='inner'` would silently drop orders.)
</details>

**P10. Add a 7-day rolling average column (mean of each row and the 6 preceding rows) to a daily sales DataFrame; first 6 rows get NaN.** DataFrame `daily_sales` with columns `day` (date) and `sales` (daily sales figure). *Example:* day 7's rolling average = mean of days 1–7; the first 6 rows get NaN (not enough history yet).
<details><summary>Solution</summary>

```python
df = df.sort_values('date')
df['ma7'] = df['sales'].rolling(window=7).mean()
```
Sort by date first. `rolling(7).mean()` gives NaN for the first 6 rows (not enough history) — use `min_periods=1` if you want partial windows.
</details>

**P11. Count how many events each user triggered per calendar day by extracting the date from the timestamp and grouping by (user_id, day).** DataFrame `web_events` with columns `user_id`, `event` (e.g., 'view', 'click'), and `timestamp` (full datetime). *Example:* user 1 has 2 events at 10:00 and 10:10 on 2024-01-01 → count = 2 for that user-day.
<details><summary>Solution</summary>

```python
df['day'] = pd.to_datetime(df['timestamp']).dt.date
counts = df.groupby(['user_id', 'day']).size().reset_index(name='event_count')
```
`.size()` counts rows per group; `dt.date` buckets timestamps to day.
</details>

**P12. Deduplicate a users DataFrame by email address, keeping the row with the most recent signup date.** DataFrame `users` (or `signups`) with columns `email` and `signup_date`. *Example:* `a@x.com` signed up on 2024-01-01 and again on 2024-02-01 → keep the 2024-02-01 row.
<details><summary>Solution</summary>

```python
(df.sort_values('signup_date')
   .drop_duplicates(subset='email', keep='last'))
```
Sort ascending, keep `last` = most recent. (Mirrors the SQL `ROW_NUMBER()` dedup pattern.)
</details>

**P13. Pivot a long-format sales table so each quarter becomes its own column, one row per region.** DataFrame `region_sales` with columns `region`, `quarter` (values: 'Q1', 'Q2'), and `revenue`. *Example:* North/Q1=100, North/Q2=120 → a single row `North | 100 | 120`.
<details><summary>Solution</summary>

```python
df.pivot_table(index='region', columns='quarter', values='revenue', aggfunc='sum').reset_index()
```
`pivot_table` (not `pivot`) handles duplicate index/column pairs by aggregating.
</details>

**P14. Add a month-over-month % revenue change column to a monthly revenue DataFrame.** DataFrame `monthly_revenue` with columns `month` (date, one row per month) and `revenue`. *Example:* Jan=1000, Feb=1100 → Feb's change = +10.0%; Mar=990 → Mar's change = −10.0%.
<details><summary>Solution</summary>

```python
df = df.sort_values('month')
df['mom_pct'] = df['revenue'].pct_change() * 100
```
`pct_change()` = (current − previous) / previous; equivalent to SQL `LAG`.
</details>

---

## Python — NumPy & Vectorization

**P15. Replace all negative values in a NumPy array with 0 using a single vectorized call — no Python loop.** You have a 1-D NumPy array like `[-3, 1, -5, 4, 0]`. *Example:* `[-3, 1, -5, 4, 0]` → `[0, 1, 0, 4, 0]`.
<details><summary>Solution</summary>

```python
arr = np.where(arr < 0, 0, arr)      # or: arr[arr < 0] = 0
```
`np.where(cond, a, b)` is the vectorized if/else; boolean masking also works in place.
</details>

**P16. Z-score standardize every column of a 2-D NumPy array so each column has mean 0 and std 1, without loops.** You have a 2-D array where rows are samples and columns are features. *Example:* column `[10, 20, 30]` → `[-1.0, 0.0, 1.0]`.
<details><summary>Solution</summary>

```python
z = (arr - arr.mean(axis=0)) / arr.std(axis=0)
```
`axis=0` computes per-column stats; **broadcasting** subtracts/divides row-wise automatically.
</details>

**P17. Compute the Euclidean distance √(Σ(aᵢ−bᵢ)²) between two NumPy vectors without writing a loop.** Given two 1-D arrays `a` and `b` of the same length. *Example:* `a=[0,0]`, `b=[3,4]` → distance = 5.0.
<details><summary>Solution</summary>

```python
dist = np.sqrt(np.sum((a - b) ** 2))   # or np.linalg.norm(a - b)
```
`np.linalg.norm(a - b)` is the idiomatic one-liner.
</details>

---

## Python — Implement DS Functions from Scratch

**P18. Implement precision, recall, and F1 score from scratch using two binary label lists, handling zero-denominator cases.** You have two Python lists of 0s and 1s: `y_true` (actual labels) and `y_pred` (model predictions). *Example:* `y_true=[1,0,1,1]`, `y_pred=[1,1,1,0]` → TP=2, FP=1, FN=1 → P=0.67, R=0.67, F1=0.67.
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

**P19. Implement a reproducible train/test split without sklearn: shuffle indices, slice the last `test_size` fraction as the test set, accept a random seed.** Given arrays `X` (features) and `y` (labels) and a `test_size` fraction (e.g., 0.2). *Example:* 100 samples → 80 train, 20 test.
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

**P20. Compute Root Mean Squared Error (RMSE) = √(mean of (actual − predicted)²) from scratch.** Given `y_true` and `y_pred` (lists or arrays of numbers). *Example:* `y_true=[3,5]`, `y_pred=[2,4]` → errors are −1, −1 → RMSE = 1.0.
<details><summary>Solution</summary>

```python
def rmse(y_true, y_pred):
    y_true, y_pred = np.array(y_true), np.array(y_pred)
    return np.sqrt(np.mean((y_true - y_pred) ** 2))
```
</details>

**P21. Remove outliers from a numeric list using the 1.5×IQR fence rule: drop any value below Q1−1.5×IQR or above Q3+1.5×IQR.** Given a list of numbers. *Example:* `[10,12,11,100,13]` → 100 is far above the upper fence and gets dropped.
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

**P22. Compute cosine similarity = dot(a,b) / (‖a‖ × ‖b‖) between two numeric vectors from scratch; result ranges from −1 to 1.** Given two lists or arrays `a` and `b`. *Example:* `a=[1,0]`, `b=[1,0]` → similarity = 1.0; `a=[1,0]`, `b=[0,1]` → similarity = 0.0 (orthogonal).
<details><summary>Solution</summary>

```python
def cosine(a, b):
    a, b = np.array(a), np.array(b)
    return np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b))
```
Guard against zero-norm vectors in production.
</details>

---

## Python — Harder & Real-Company Patterns

**P23. (Shopify/Amazon) Find which customer spent the most on a single calendar day and what date that was, returning ties.** DataFrame `orders` with columns `customer_id`, `order_date`, and `amount`. *Example:* ACME placed 3 orders on 2019-02-05 totalling 460; Stark placed one order on 2019-04-15 for 400 → ACME wins with 460 on 2019-02-05.
<details><summary>Solution</summary>

```python
daily = df.groupby(['customer_id', 'order_date'], as_index=False)['order_cost'].sum()
daily[daily['order_cost'] == daily['order_cost'].max()]
```
Aggregate to daily totals first, then filter to the max. (`==max()` returns ties; `.nlargest(1)` would drop them.)
</details>

**P24. Add a `band` column to orders — 'low' (amount < 50), 'mid' (50–199), 'high' (≥ 200) — using vectorized logic, no `.apply`.** DataFrame `orders` with column `amount`. *Example:* amounts 30, 120, 300 → 'low', 'mid', 'high'.
<details><summary>Solution + alternatives</summary>

```python
# BEST — vectorized, handles many conditions cleanly:
df['band'] = np.select(
    [df.amount < 50, df.amount < 200],
    ['low', 'mid'],
    default='high')

# Also fine for 2 buckets: np.where(df.amount < 50, 'low', 'high')
# AVOID (slow, row-by-row): df['band'] = df.amount.apply(lambda x: 'low' if x < 50 else ...)
```
`np.select` is the idiomatic multi-branch vectorized if/else — far faster than `.apply` on big data.
</details>

**P25. Add a column showing each employee's salary as a percentage of their department's total salary, per group without a separate merge.** DataFrame `employees` with columns `dept` and `salary`. *Example:* Eng total=410; Alice earns 120 → her share = 29.3%.
<details><summary>Solution</summary>

```python
df['pct_of_dept'] = df['salary'] / df.groupby('dept')['salary'].transform('sum') * 100
```
`transform('sum')` broadcasts the group total back to every row — no merge needed.
</details>

**P26. As-of join: for each trade, attach the most recent quote price for the same symbol at or before the trade's timestamp.** DataFrame `trades` (columns `symbol`, `time`) and `quotes` (columns `symbol`, `time`, `price`). *Example:* AAPL trade at 09:31 → matches the 09:30 quote at $187.0, not the 09:34 quote.
<details><summary>Solution</summary>

```python
trades = trades.sort_values('time'); quotes = quotes.sort_values('time')
pd.merge_asof(trades, quotes, on='time', by='symbol', direction='backward')
```
`merge_asof` is the specialized tool for "nearest earlier match" joins (finance, event enrichment) — both frames must be sorted on the key.
</details>

**P27. Split a comma-separated tags string and explode it so each tag gets its own row paired with its user.** DataFrame `tags_df` with columns `user` and `tags`, where `tags` is a single string of comma-separated values like `'python, sql, ml'`. *Example:* user 1 with `'python, sql, ml'` → 3 rows: (1,'python'), (1,'sql'), (1,'ml').
<details><summary>Solution</summary>

```python
df.assign(tag=df['tags'].str.split(',')).explode('tag')
# strip whitespace: .assign(tag=lambda d: d['tag'].str.strip())
```
`str.split` → list column → `explode` fans it to one row per element (the pandas "unnest").
</details>

**P28. Aggregate total spend per customer and assign each customer a 0–100 percentile rank showing what fraction of other customers they outspend.** DataFrame `orders` with columns `customer_id` and `amount`. *Example:* if ACME spends more than 75% of customers, their percentile is 75.0.
<details><summary>Solution</summary>

```python
spend = df.groupby('customer_id')['amount'].sum()
pct = spend.rank(pct=True) * 100          # 0–100 percentile
```
`rank(pct=True)` gives the percentile directly; `method='dense'`/`'min'` controls tie behavior.
</details>

**P29. Find the length of every consecutive-day login streak per user: deduplicate same-day logins first, then identify runs of back-to-back calendar days.** DataFrame `logins` with columns `user_id` and `login_date`. *Example:* user 1 logged in on Jan 1, 2, 3 (streak=3), skipped Jan 4, then Jan 5 (streak=1).
<details><summary>Solution (the pandas gaps-and-islands trick)</summary>

```python
df = df.drop_duplicates(['user','date']).sort_values(['user','date'])
df['date'] = pd.to_datetime(df['date'])
# A streak = consecutive days ⇒ (date - rank_days) is constant within a streak:
grp = df.groupby('user')
df['island'] = (df['date'] - pd.to_timedelta(grp.cumcount(), unit='D'))
streaks = df.groupby(['user','island']).size()   # length of each streak
```
Subtracting a running counter of days from the date makes consecutive dates map to the **same constant** → group by it. This is the pandas mirror of the SQL gaps-and-islands pattern below.
</details>

---

## Python — Common DS Patterns (by Category)

Interviewers think in patterns. Drill these buckets — **missing data, cleaning, reshaping, time series, filtering, feature engineering** — and most pandas questions become recognizable.

### A. Missing Data

**P32. Fill missing values: numeric columns with the column mean, categorical columns with the most-frequent (mode) value.** DataFrame `employees` with numeric column `age` (some NaN) and categorical column `dept` (some NaN).
<details><summary>Solution</summary>

```python
df['age']  = df['age'].fillna(df['age'].mean())            # numeric → mean/median
df['dept'] = df['dept'].fillna(df['dept'].mode()[0])       # categorical → most frequent
```
`.mode()` returns a Series (can be multi-modal) → take `[0]`. Median is safer than mean for skewed data.
</details>

**P33. Fill gaps in a time-series column: forward-fill the last known value, then back-fill any remaining leading NaNs.** DataFrame with columns `date` and `value` (some `value` entries are NaN). *Example:* `[NaN, 10, NaN, 20]` → `[10, 10, 20, 20]`.
<details><summary>Solution</summary>

```python
df = df.sort_values('date')
df['value'] = df['value'].ffill().bfill()     # ffill carries last obs forward; bfill fills any leading NaNs
```
</details>

**P34. Fill missing values in a numeric time-series column by linear interpolation — draw a straight line between known points, not repeat the last value.** DataFrame with columns `date` and `value` (some NaN). *Example:* `[10, NaN, 20]` → `[10, 15.0, 20]`.
<details><summary>Solution + when to prefer it</summary>

```python
df['value'] = df['value'].interpolate(method='linear')
```
**`fillna` vs `interpolate`:** `fillna`/`ffill` repeats a constant/last value; **`interpolate` estimates from the trend** (draws a line between known points) — better for continuous/time-series data.
</details>

### B. Duplicates & Cleaning

**P35. Count exact duplicate rows, drop them, and also show dedup by `email` key keeping the last row.** DataFrame `users` with columns `email` and `signup_date`.
<details><summary>Solution</summary>

```python
df.duplicated().sum()                          # how many dup rows
df.drop_duplicates()                           # drop exact-dup rows
df.drop_duplicates(subset='email', keep='last')  # dedup by a key, keep the last
```
</details>

**P36. Clean a messy string column and safely cast other columns to numeric and datetime types: trim/lowercase `name`, coerce bad `amount` strings to NaN, parse `date`, and convert `dept` to category dtype.** A raw DataFrame has a `name` column with extra whitespace and mixed casing, an `amount` column stored as strings (some invalid entries), a `date` column as strings, and a `dept` column with repeated categories.
<details><summary>Solution</summary>

```python
df['name']  = df['name'].str.strip().str.lower()          # trim + lowercase
df['amount'] = pd.to_numeric(df['amount'], errors='coerce')  # bad values → NaN, not crash
df['date']   = pd.to_datetime(df['date'], errors='coerce')
df['dept']   = df['dept'].astype('category')              # memory + speed for repeated categories
```
`errors='coerce'` turns unparseable values into NaN instead of raising — key for real-world dirty data.
</details>

**P37. Add a boolean `is_outlier` column flagging rows where `|z-score| > 3` (more than 3 standard deviations from the mean).** DataFrame `orders` with column `amount`. *Example:* if mean=190 and std=110, an amount of 700 has z≈4.6 and would be flagged.
<details><summary>Solution</summary>

```python
z = (df['amount'] - df['amount'].mean()) / df['amount'].std()
df['is_outlier'] = z.abs() > 3
```
(IQR version is in [P21](#python--implement-ds-functions-from-scratch).)
</details>

### C. Reshaping

**P38. Reshape a wide table (one column per quarter) into a long table (one row per quarter) — i.e. unpivot it.** DataFrame `region_sales` currently has columns `region`, `Q1`, `Q2`, `Q3`. *Example:* `North | 100 | 120 | 90` (wide) → 3 rows: (North, Q1, 100), (North, Q2, 120), (North, Q3, 90) (long).
<details><summary>Solution</summary>

```python
pd.melt(df, id_vars='region', value_vars=['Q1','Q2','Q3'],
        var_name='quarter', value_name='revenue')
```
`melt` = unpivot; `pivot_table` = the reverse (long → wide, see [P13](#python--pandas-data-manipulation)).
</details>

**P39. Build a frequency cross-tabulation of `status` × `customer_id`, with statuses as rows, customer IDs as columns, and order counts as cell values.** DataFrame `orders` with columns `status` (e.g., 'completed', 'pending', 'refunded') and `customer_id`.
<details><summary>Solution</summary>

```python
pd.crosstab(orders['status'], orders['customer_id'])   # counts of status × customer
# normalize='index' for row-wise proportions
```
</details>

### D. Time Series & Datetime

**P40. Extract five calendar components from a datetime column: `year`, `month`, `dow` (0=Monday, 6=Sunday), `hour`, and `is_weekend` (True if Saturday or Sunday).** DataFrame `web_events` with column `event_time` (full timestamp).
<details><summary>Solution</summary>

```python
ts = pd.to_datetime(df['event_time'])
df['year']  = ts.dt.year
df['month'] = ts.dt.month
df['dow']   = ts.dt.dayofweek        # 0=Mon … 6=Sun
df['hour']  = ts.dt.hour
df['is_weekend'] = ts.dt.dayofweek >= 5
```
Datetime decomposition captures seasonality (day-of-week, month) — a top time-series feature-engineering move.
</details>

**P41. Aggregate a daily sales DataFrame into weekly totals.** DataFrame `daily_sales` with columns `day` (date) and `sales` (integer). *Example:* days Mon–Sun with sales [100,120,110,130,90,140,150] → one row with total 840.
<details><summary>Solution</summary>

```python
weekly = (daily_sales.set_index('day')['sales']
                     .resample('W').sum())      # 'W' week, 'M' month, 'D' day, 'H' hour
```
`resample` = groupby on a time frequency; the index must be datetime.
</details>

**P42. Add `prev_amount` (previous order amount) and `delta` (current minus previous) as per-customer lag features; first order per customer gets NaN.** DataFrame `orders` with columns `customer_id`, `order_date`, and `amount`. *Example:* ACME's orders 250 → 120 → 90 give `delta` values NaN, −130, −30.
<details><summary>Solution</summary>

```python
g = orders.sort_values('order_date').groupby('customer_id')['amount']
orders['prev_amount'] = g.shift(1)          # previous order's amount (SQL LAG)
orders['delta']       = g.diff()            # change vs previous
```
`shift`/`diff` inside a groupby stay within each group — no leakage across customers.
</details>

**P43. Add a 3-day rolling mean (`ma3`) and rolling std (`std3`) to the daily sales DataFrame, using `min_periods=1` for partial windows.** DataFrame `daily_sales` with columns `day` and `sales`. *Example:* sales [100,120,110,...] → day 3 ma3 = (100+120+110)/3 ≈ 110.
<details><summary>Solution</summary>

```python
daily_sales = daily_sales.sort_values('day')
daily_sales['ma3']  = daily_sales['sales'].rolling(3, min_periods=1).mean()
daily_sales['std3'] = daily_sales['sales'].rolling(3).std()
```
`min_periods=1` returns partial-window values instead of NaN for the first rows.
</details>

### E. Filtering & Selection

**P44. Filter `employees` to keep only rows where `dept` is Eng or Sales AND `salary` is between 90 and 120 inclusive — show both boolean-mask and `query()` style.** DataFrame `employees` with columns `dept` and `salary`. *Example:* Alice (Eng, 120) passes both conditions; Dave (Eng, 90) passes; Eve (Sales, 95) passes; Bob (Eng, 100) passes; Judy (Sales, 150) fails the salary range.
<details><summary>Solution</summary>

```python
df[df['dept'].isin(['Eng','Sales']) & df['salary'].between(90, 120)]
# readable alternative:
df.query("dept in ['Eng','Sales'] and 90 <= salary <= 120")
```
`isin` for membership, `between` for ranges; `query` is often cleaner for compound conditions.
</details>

**P45. Demonstrate `.loc` (label/boolean, select Eng rows with name+salary columns) vs `.iloc` (position, select first 3 rows and columns 0 & 2) on `employees`, and explain when to use each.** DataFrame `employees` with columns `name`, `dept`, `salary`, etc.
<details><summary>Solution</summary>

```python
df.loc[df['dept'] == 'Eng', ['name','salary']]   # by LABEL / boolean mask + column names
df.iloc[0:3, [0, 2]]                             # by POSITION (rows 0–2, cols 0 & 2)
```
`loc` = labels/conditions; `iloc` = integer positions. Mixing them up is a classic bug.
</details>

**P46. Select the 3 highest-amount and 3 lowest-amount orders using the most efficient built-in methods (not sort + head).** DataFrame `orders` with column `amount`. *Example:* amounts [400, 300, 250, 220, 150, 120, 90, 80] → top-3: 400, 300, 250; bottom-3: 80, 90, 120.
<details><summary>Solution</summary>

```python
orders.nlargest(3, 'amount')      # faster than sort_values().head(3) — partial sort
orders.nsmallest(3, 'amount')
```
</details>

### F. Feature Engineering

**P47. Bin employee salaries two ways: `salary_q` (4 equal-frequency quartiles via `qcut`) and `salary_band` (fixed cuts: 0–80 'low', 81–110 'mid', 111+ 'high' via `cut`).** DataFrame `employees` with column `salary`. *Example:* salary 95 → Q2 (roughly), 'mid' band.
<details><summary>Solution</summary>

```python
df['salary_q']    = pd.qcut(df['salary'], q=4, labels=['Q1','Q2','Q3','Q4'])   # equal-frequency
df['salary_band'] = pd.cut(df['salary'], bins=[0,80,110,999],
                           labels=['low','mid','high'])                        # fixed edges
```
`qcut` = equal-sized buckets (quantiles); `cut` = fixed cut points.
</details>

**P48. One-hot encode the `dept` column into 0/1 dummy columns, dropping one dummy to avoid the dummy-variable trap (perfect multicollinearity).** DataFrame `employees` with a `dept` column whose values are 'Eng' or 'Sales'. *Example:* Eng rows → `dept_Sales=0`; Sales rows → `dept_Sales=1`.
<details><summary>Solution</summary>

```python
pd.get_dummies(df, columns=['dept'], drop_first=True)   # drop_first avoids the dummy-variable trap
```
</details>

**P49. Normalize the `salary` column two ways: `salary_minmax` (min-max scaled to 0–1) and `salary_z` (z-score with mean 0 and std 1).** DataFrame `employees` with column `salary`. *Example:* salaries range 60–150 → the minimum (60) gets 0.0 and the maximum (150) gets 1.0 under min-max.
<details><summary>Solution</summary>

```python
s = df['salary']
df['salary_minmax'] = (s - s.min()) / (s.max() - s.min())     # → [0,1]
df['salary_z']      = (s - s.mean()) / s.std()                # mean 0, std 1
```
</details>

### G. Vectorization (the red-flag pattern)

**P50. Create a `flag` column ('big' if amount > 200 AND status='completed', else 'small') using a vectorized approach; also show the slow `apply(axis=1)` version and explain why it's an interview red flag.** DataFrame `orders` with columns `amount` and `status`.
<details><summary>Solution</summary>

```python
# ❌ slow — row-by-row Python:
df['flag'] = df.apply(lambda r: 'big' if r['amount'] > 200 and r['status']=='completed' else 'small', axis=1)

# ✅ vectorized — boolean logic on whole columns:
df['flag'] = np.where((df['amount'] > 200) & (df['status']=='completed'), 'big', 'small')
```
`apply(axis=1)` runs a Python function per row — a common interview red flag. Combine column conditions with `&`/`|` (parenthesize each!) and `np.where`/`np.select`.
</details>

Source: [Top pandas interview patterns (DataCamp)](https://www.datacamp.com/blog/top-python-pandas-interview-questions-and-answers) · [Time-series feature engineering](https://machinelearningmastery.com/7-pandas-tricks-for-time-series-feature-engineering/)

---

## Python — Efficiency: Alternative Solutions

Interviewers flag explicit loops where vectorization is expected. Know the fast alternative for each common task:

| Task | ❌ Slow | ✅ Efficient |
|---|---|---|
| Conditional column | `.apply(lambda row: ...)` | `np.where` / `np.select` (vectorized) |
| Lookup/enrich by key | Python `for` + dict | `df['x'].map(mapping)` or `merge` |
| Top-N rows | `.sort_values().head(n)` | `.nlargest(n, col)` (partial sort, faster) |
| Group-wise value on each row | `merge` a groupby back | `groupby(...).transform(...)` |
| Filter | boolean mask on huge frame | `df.query("amount > 100")` (readable, can be faster) |
| Row-wise math | iterate rows | column arithmetic (`df.a * df.b`) — C-speed |
| Repeated string categories | `object` dtype | `astype('category')` (memory + speed) |
| Count per group | `groupby().apply(len)` | `groupby().size()` or `value_counts()` |
| Existence check in list | `x in a_list` (O(n)) | `x in a_set` (O(1)) |

**P30. Sum a numeric DataFrame column three ways — (1) Python `for` loop, (2) `.apply(lambda x: x).sum()`, (3) built-in vectorized `.sum()` — and explain why each differs in speed.** DataFrame `orders` with column `amount`.
<details><summary>Solution</summary>

```python
# ❌ Python loop over rows — slowest:
total = 0
for v in df['amount']: total += v
# ⚠️ .apply — still Python-level:
df['amount'].apply(lambda x: x).sum()
# ✅ vectorized — runs in C:
df['amount'].sum()
```
Rule of thumb: if you're writing a `for` loop over a DataFrame, there's almost always a vectorized alternative. Reach for iteration only for genuinely sequential logic (and even then prefer `itertuples()` over `iterrows()`).
</details>

**P31. Replace abbreviated country codes with full names using a mapping dict in a single vectorized operation — no loop.** DataFrame `customers` with column `country` (values like `'US'`, `'IN'`, `'UK'`). *Example:* `'US'` → `'United States'`; any code not in the dict → NaN.
<details><summary>Solution</summary>

```python
df['country'] = df['code'].map({'US':'United States','IN':'India'})   # vectorized lookup
# .replace(mapping) also works; .map is faster and returns NaN for misses
```
</details>

---

## Pandas `groupby` — Full Reference

`groupby` is the workhorse of pandas interviews. Understand it through **one mental model + three operation types** (examples run on the [sample data](#sample-data--run-this-first)).

**The mental model — split → apply → combine:**
```
employees.groupby('dept')      # SPLIT rows into one bucket per dept (lazy — shows nothing yet)
          .agg / transform / filter   # APPLY a function to each bucket
                              # COMBINE results back together
```
`df.groupby('dept')` alone does nothing visible — what you do *next* decides the output shape. The three fundamentally different things:

### 1. Aggregate — many rows → **one row per group** (output *shrinks*)
```python
employees.groupby('dept')['salary'].mean()               # one number per dept

# multiple named aggregations (clean column names):
employees.groupby('dept').agg(
    headcount=('name', 'count'),
    avg_salary=('salary', 'mean'),
    max_salary=('salary', 'max'))

# different funcs per column:
employees.groupby('dept').agg({'salary': ['mean', 'max'], 'age': 'median'})

# custom:
employees.groupby('dept')['salary'].agg(lambda s: s.max() - s.min())   # salary spread
```
Aggregators: `sum, mean, median, min, max, count, size, std, var, nunique, first, last`.

### 2. Transform — returns the **same shape** (one value per *original* row)
The group result is broadcast back to every row → ideal for new columns.
```python
employees['dept_avg'] = employees.groupby('dept')['salary'].transform('mean')          # stat per row
employees['age'] = employees.groupby('dept')['age'].transform(lambda s: s.fillna(s.median()))  # group fill
employees['pct'] = employees['salary'] / employees.groupby('dept')['salary'].transform('sum') * 100
g = employees.groupby('dept')['salary']
employees['z'] = (employees['salary'] - g.transform('mean')) / g.transform('std')        # group z-score
```

### 3. Filter — keep or drop **whole groups** (pandas' `HAVING`)
```python
employees.groupby('dept').filter(lambda gp: len(gp) > 5)                  # depts with >5 employees
employees.groupby('dept').filter(lambda gp: gp['salary'].mean() > 90)
```

### 4. Apply — the do-anything escape hatch (most flexible, slowest)
```python
employees.groupby('dept').apply(lambda gp: gp.nlargest(2, 'salary'))      # top 2 rows per group
```

### 5. Select specific rows per group
```python
employees.sort_values('salary', ascending=False).groupby('dept').head(2)  # top 2 per dept
employees.groupby('dept').tail(1)          # last row per group
employees.groupby('dept').nth(0)           # the 0th row per group
employees.groupby('dept')['salary'].idxmax()   # index of max-salary row per dept
```

### 6. Cumulative & ranking within groups
```python
orders.sort_values('order_date').groupby('customer_id')['amount'].cumsum()   # running total per customer
employees.groupby('dept')['salary'].rank(ascending=False)                    # rank within dept
employees.groupby('dept').cumcount()                                         # 0,1,2… position in group
```

### 7. Utility & inspection
```python
employees.groupby('dept').size()               # rows per group (counts NaN)
employees.groupby('dept').count()              # non-null count per column
employees.groupby('dept')['name'].nunique()
employees.groupby('dept').get_group('Eng')     # pull out one group as a DataFrame
for name, gp in employees.groupby('dept'):     # iterate groups
    print(name, len(gp))
orders.groupby(['customer_id', 'status'])['amount'].sum()   # group by multiple keys
```

### Cheat sheet

| You want… | Method | Output shape |
|---|---|---|
| One summary per group | **`.agg` / `.mean()` etc.** | **shrinks** (1 row/group) |
| A group stat beside each row | **`.transform`** | **same** as input |
| Keep/drop whole groups | **`.filter`** | subset of rows |
| Arbitrary per-group logic | **`.apply`** | anything (slowest) |
| First/last/N-th rows per group | `.head` / `.tail` / `.nth` | subset |
| Running total / rank in group | `.cumsum` / `.rank` / `.cumcount` | same as input |
| Counts | `.size` (incl. NaN) / `.count` (non-null) | 1/group |

> **The mental hook:** *`agg` shrinks, `transform` keeps shape, `filter` selects groups, `apply` does anything.* ~90% of interview `groupby` questions are one of the first three. Tip: add `.reset_index()` after an aggregation to turn the group keys back into normal columns.

---

## SQL — The 7 Recurring Patterns (~90% of Interviews)

Most SQL interview questions are one of these shapes — recognize the pattern and the query writes itself:

| # | Pattern | Signature tool | Example below |
|---|---|---|---|
| 1 | **Filter & aggregate** | `GROUP BY` + `HAVING` | S1, S2 |
| 2 | **Conditional joins** | `LEFT JOIN` + `IS NULL` (anti-join) | S3, S5 |
| 3 | **Top-N per group** | `ROW_NUMBER() OVER (PARTITION BY…)` | S10 |
| 4 | **Period-over-period** | `LAG` / `LEAD` | S12 |
| 5 | **Gaps & islands / sessionization** | `LAG` + running `SUM` of a flag | S19–S21 |
| 6 | **Dedup keeping latest** | `ROW_NUMBER()` + filter `= 1` | S14 |
| 7 | **Hierarchy / manager chain** | recursive CTE / self-join | S22–S23 |

Source: [The 7 SQL interview patterns](https://datavidhya.com/blog/sql-data-engineering-interview-questions/)

---

## SQL — Core (SELECT / Filter / Aggregate)

*Tables:* `employees(id, name, dept, salary, hire_date, manager_id)`, `orders(id, customer_id, amount, order_date, status)`, `customers(id, name, country)`.

**S1. Average salary per department, keeping only departments with more than 5 employees, sorted highest average first.** Table `employees(name, dept, salary)`. *Example:* if Sales has 6 people and Eng has 4, only Sales appears (Eng is dropped).
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

**S2. For each order status, return the status label, count of orders, and total revenue.** Table `orders(id, customer_id, amount, order_date, status)`. *Example:* 'completed' might have 5 orders totalling $860.
<details><summary>Solution</summary>

```sql
SELECT status, COUNT(*) AS n_orders, SUM(amount) AS revenue
FROM orders
GROUP BY status;
```
</details>

**S3. Find all customers who have never placed a single order (anti-join pattern).** Tables `customers(id, name, country)` and `orders(id, customer_id, amount, ...)`. *Example:* Umbrella (id=4) has no orders and should appear; ACME (id=1) has orders and should not.
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

**S4. Inner-join each order to its customer to show order ID, amount, customer name, and country — unmatched orders are silently dropped.** Tables `orders(id, customer_id, amount, order_date, status)` and `customers(id, name, country)`. *Example:* all 8 orders match a customer in our dataset, so all 8 rows appear.
<details><summary>Solution</summary>

```sql
SELECT o.id, o.amount, c.name, c.country
FROM orders o
JOIN customers c ON c.id = o.customer_id;   -- INNER JOIN
```
</details>

**S5. Compute total revenue per country, so countries with no orders show 0 rather than being missing.** Tables `customers(id, name, country)` and `orders(id, customer_id, amount, ...)`. *Example:* UK has only Umbrella who placed no orders → revenue = 0.
<details><summary>Solution</summary>

```sql
SELECT c.country, COALESCE(SUM(o.amount), 0) AS revenue
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.id
GROUP BY c.country;
```
`LEFT JOIN` keeps all countries; `COALESCE` turns NULL sums (no orders) into 0.
</details>

**S6. Self-join `employees` to show each employee paired with their manager's name; use a left join so the CEO (no manager) still appears with NULL.** Table `employees(id, name, dept, salary, hire_date, manager_id)`. *Example:* Alice (manager_id=10) → manager = Judy.
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

**S7. Return all employees whose salary is above the overall company average.** Table `employees(name, salary)`. *Example:* average salary ≈ 96; Alice (120), Bob (100), Carol (100), and Judy (150) all qualify.
<details><summary>Solution</summary>

```sql
SELECT name, salary
FROM employees
WHERE salary > (SELECT AVG(salary) FROM employees);
```
Scalar subquery — runs once.
</details>

**S8. Return employees who earn above their own department's average — first with a correlated subquery (slow, reruns per row), then with a faster window-function rewrite (computes once per dept).** Table `employees(name, dept, salary)`. *Example:* Eng avg=102.5 → Alice (120) qualifies; Sales avg=91.7 → Judy (150) and Eve (95) qualify.
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

**S9. Find the 2nd-highest *distinct* salary — if several people tie for the top salary, they still only count as the 1st.** Table `employees(salary)`. *Example:* salaries 150, 120, 100, 100, 95, 95, 90, 80, 70, 60 → rank 1 = 150, rank 2 = 120 (not 100, because 120 is the second unique value).
<details><summary>Solution</summary>

```sql
WITH r AS (SELECT salary, DENSE_RANK() OVER (ORDER BY salary DESC) AS rk FROM employees)
SELECT DISTINCT salary FROM r WHERE rk = 2;
```
`DENSE_RANK` (no gaps) is correct for "Nth highest distinct value"; `ROW_NUMBER` would miss ties.
</details>

**S10. Return the top 3 highest-paid employees within each department (with 2 departments you get up to 6 rows).** Table `employees(name, dept, salary)`. *Example:* Eng (4 people): Alice(120), Bob(100), Carol(100) → all three qualify; Sales (6 people): Judy(150), Eve(95), Frank(95) → those three qualify.
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

**S11. Compute a running (cumulative) total of order revenue, ordered by date.** Table (or derived table) `daily_orders(order_date, amount)`. *Example:* amounts 250, 120, 300 on consecutive dates → running totals 250, 370, 670.
<details><summary>Solution</summary>

```sql
SELECT order_date,
       SUM(amount) OVER (ORDER BY order_date) AS running_total
FROM daily_orders;
```
Adding `ORDER BY` inside `OVER` makes each row sum itself + all prior rows.
</details>

**S12. Add a column showing each month's revenue change vs the previous month; the first row is NULL (no prior month).** Table `monthly_revenue(month, revenue)`. *Example:* Jan=1000, Feb=1100 → Feb change = +100; Mar=990 → Mar change = −110.
<details><summary>Solution</summary>

```sql
SELECT month, revenue,
       revenue - LAG(revenue) OVER (ORDER BY month) AS mom_change
FROM monthly_revenue;
```
`LAG` pulls the previous row's value; the first row is NULL (no prior month).
</details>

**S13. Compute a 3-day moving average of sales (each day averaged with the two days before it).** Table `daily_sales(day, sales)`. *Example:* sales [100, 120, 110, 130] → day 3 ma3 = (100+120+110)/3 ≈ 110; day 4 ma3 = (120+110+130)/3 ≈ 120.
<details><summary>Solution</summary>

```sql
SELECT day, sales,
       AVG(sales) OVER (ORDER BY day ROWS BETWEEN 2 PRECEDING AND CURRENT ROW) AS ma3
FROM daily_sales;
```
`ROWS BETWEEN 2 PRECEDING AND CURRENT ROW` = a 3-row sliding window.
</details>

**S14. Delete duplicate user rows (same email), keeping only the one with the smallest `id`.** Table `users(id, email, signup_date)`. *Example:* `a@x.com` appears at id=1 and id=3 → delete id=3.
<details><summary>Solution</summary>

```sql
WITH d AS (SELECT *, ROW_NUMBER() OVER (PARTITION BY email ORDER BY id) AS rn FROM users)
DELETE FROM users WHERE id IN (SELECT id FROM d WHERE rn > 1);
```
</details>

---

## SQL — Dates & Advanced Patterns

**S15. Count how many orders each customer placed in the most recent 30 days, writing the date filter in a sargable way so an index can be used.** Table `orders(customer_id, order_date, amount, status)`.
<details><summary>Solution</summary>

```sql
SELECT customer_id, COUNT(*) AS recent_orders
FROM orders
WHERE order_date >= CURRENT_DATE - INTERVAL '30 days'
GROUP BY customer_id;
```
Keep the predicate **sargable** — filter the raw `order_date` column, don't wrap it in a function.
</details>

**S16. Compute Daily Active Users (DAU = count of distinct users per calendar day) for the last 7 days.** Table `events(user_id, event, event_time)`. *Example:* if both user 1 and user 2 fired events on 2024-01-01 → DAU = 2.
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

**S17. For each calendar month, compute the percentage of orders whose status is 'completed' (completed count / total × 100).** Table `orders(order_date, status)`. *Example:* Feb has 3 orders — 2 completed, 1 refunded → 66.7% completed.
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

**S18. Find the first order date (acquisition date) for each customer — the foundation for cohort retention analysis.** Table `orders(customer_id, order_date)`. *Example:* ACME (id=1) first ordered on 2019-02-05; Globex (id=2) on 2019-03-10.
<details><summary>Solution</summary>

```sql
SELECT customer_id, MIN(order_date) AS first_order
FROM orders
GROUP BY customer_id;
```
Foundation for cohort/retention analysis.
</details>

---

## SQL — Gaps & Islands and Sessionization

The highest-value advanced pattern: an **island** is a contiguous run of rows; a **gap** is the break between runs. Same shape powers login streaks, sessionization, consecutive purchases, and unchanged-price periods.

**S19. Find each user's longest streak of consecutive login days (deduplicate same-day logins first).** Table `logins(user_id, login_date)`. *Example:* user 1 logged in Jan 1, 2, 3 → streak of 3; then Jan 5 alone → streak of 1.
<details><summary>Solution (the row_number trick)</summary>

```sql
WITH grp AS (
  SELECT user_id, login_date,
         login_date - (ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY login_date))::int AS island
  FROM logins
)
SELECT user_id, MIN(login_date) AS streak_start, MAX(login_date) AS streak_end,
       COUNT(*) AS streak_len
FROM grp
GROUP BY user_id, island
ORDER BY streak_len DESC;
```
**Why it works:** for consecutive dates, `date − row_number` stays **constant** (both increase by 1 each row), so it's a stable island id. A gap bumps it. Group by it to collapse each run. Take the max `streak_len` per user for the longest streak.
</details>

**S20. Assign a session number to each event, where a gap of more than 30 minutes between a user's consecutive events starts a new session.** Table `events(user_id, event, event_time)`. *Example:* user 1's events at 10:00, 10:10, 10:50, 11:00 — the 40-min gap before 10:50 starts session 2.
<details><summary>Solution (flag-then-cumulative-sum — the key composite pattern)</summary>

```sql
WITH flagged AS (
  SELECT user_id, event_time,
         CASE WHEN event_time - LAG(event_time) OVER (PARTITION BY user_id ORDER BY event_time)
                   > INTERVAL '30 minutes'
              OR LAG(event_time) OVER (PARTITION BY user_id ORDER BY event_time) IS NULL
              THEN 1 ELSE 0 END AS is_new_session
  FROM events
)
SELECT user_id, event_time,
       SUM(is_new_session) OVER (PARTITION BY user_id ORDER BY event_time) AS session_id
FROM flagged;
```
Two passes: (1) `LAG` the previous event time and **flag** a new session when the gap exceeds the timeout (or is the first event); (2) a **running SUM of the flag** turns those 0/1 marks into incrementing session numbers. This flag-then-cumsum shape is the most important composite pattern in analytics SQL.
</details>

**S21. Find each contiguous period where a product's price stayed the same, and return that period's start and end date.** Table `price_history(product_id, day, price)`. *Example:* prices [10, 10, 12, 12, 10] → three periods: 10 (Jan 1–2), 12 (Jan 3–4), 10 (Jan 5).
<details><summary>Solution</summary>

```sql
WITH marked AS (
  SELECT *, CASE WHEN price = LAG(price) OVER (PARTITION BY product_id ORDER BY day)
                 THEN 0 ELSE 1 END AS chg
  FROM price_history
), grp AS (
  SELECT *, SUM(chg) OVER (PARTITION BY product_id ORDER BY day) AS island
  FROM marked
)
SELECT product_id, price, MIN(day) AS from_day, MAX(day) AS to_day
FROM grp GROUP BY product_id, price, island;
```
Same flag-then-cumsum: flag when the price changes, cumulative-sum to id each unchanged run.
</details>

Source: [Gaps & islands pattern](https://datavidhya.com/learn/sql/interview-patterns/gaps-and-islands/)

---

## SQL — Cohort & Retention

**S22. Build a monthly cohort retention table: for each signup-month cohort, count distinct active users at each subsequent month offset.** Table `activity(user_id, activity_month)`. *Example:* users who first appeared in Jan 2024 — how many were also active in Feb (offset=1), Mar (offset=2), etc.?
<details><summary>Solution</summary>

```sql
WITH cohort AS (                                   -- each user's first month
  SELECT user_id, MIN(activity_month) AS cohort_month
  FROM activity GROUP BY user_id
),
joined AS (
  SELECT c.cohort_month,
         (EXTRACT(YEAR FROM a.activity_month)*12 + EXTRACT(MONTH FROM a.activity_month))
       - (EXTRACT(YEAR FROM c.cohort_month )*12 + EXTRACT(MONTH FROM c.cohort_month )) AS month_offset,
         a.user_id
  FROM activity a JOIN cohort c ON c.user_id = a.user_id
)
SELECT cohort_month, month_offset, COUNT(DISTINCT user_id) AS active_users
FROM joined
GROUP BY cohort_month, month_offset
ORDER BY cohort_month, month_offset;
```
Cohort = each user's first-activity month; `month_offset` = months since signup; count distinct actives per (cohort, offset). Pivoting `month_offset` into columns gives the classic retention triangle.
</details>

**S23. Compute Day-1 retention: what percentage of users had at least one activity event the day after their first-ever activity day?** Table `activity(user_id, activity_date)`. *Example:* 3 users; only user 1 was active on day 1 → day-1 retention = 33.3%.
<details><summary>Solution</summary>

```sql
WITH first_day AS (
  SELECT user_id, MIN(activity_date) AS d0 FROM activity GROUP BY user_id
)
SELECT AVG((a.user_id IS NOT NULL)::int) * 100 AS day1_retention_pct
FROM first_day f
LEFT JOIN activity a
  ON a.user_id = f.user_id AND a.activity_date = f.d0 + INTERVAL '1 day';
```
Left-join each user's day-0 to activity on day-0+1; the average of the match flag is the retention rate.
</details>

Source: [SQL cohort & retention questions](https://letsdatascience.com/blog/sql-cohort-retention-interview-questions)

---

## SQL — Recursive CTE & Hierarchy

**S24. Use a recursive CTE to find every employee in the full reporting subtree under a given manager (all levels deep), tracking depth as level.** Table `employees(id, name, manager_id)`. *Example:* starting at Judy (id=10) returns all 9 other employees across multiple levels.
<details><summary>Solution</summary>

```sql
WITH RECURSIVE subtree AS (
  SELECT id, name, manager_id, 1 AS level
  FROM employees WHERE id = :manager_id          -- anchor: the manager
  UNION ALL
  SELECT e.id, e.name, e.manager_id, s.level + 1
  FROM employees e JOIN subtree s ON e.manager_id = s.id   -- recurse down
)
SELECT * FROM subtree;
```
Anchor row = the starting manager; the recursive part repeatedly joins direct reports until none remain. `level` gives the depth in the org chart.
</details>

---

## SQL — Efficient Alternatives & Rewrites

Interviewers love "can you write that a better way?" Know these rewrites:

| Slower / risky | Faster / safer | Why |
|---|---|---|
| Correlated subquery (`WHERE x > (SELECT AVG…WHERE dept=e.dept)`) | Window: `AVG(x) OVER (PARTITION BY dept)` | Correlated subquery re-runs per row; window computes once |
| `WHERE id NOT IN (SELECT …)` | `NOT EXISTS` or `LEFT JOIN … IS NULL` | `NOT IN` returns **nothing** if the subquery has a NULL |
| `SELECT DISTINCT` to dedup groups | `GROUP BY` | Clearer intent; often a better plan |
| Self-join to get previous row | `LAG()` window function | One pass vs an n×n-ish join |
| `WHERE YEAR(date) = 2025` | `date >= '2025-01-01' AND date < '2026-01-01'` | Sargable — lets the index work |
| `UNION` (dedups) | `UNION ALL` when duplicates impossible | Skips the dedup sort |
| `COUNT(DISTINCT big_col)` everywhere | approx (`APPROX_COUNT_DISTINCT`) if exactness not needed | Much cheaper at scale |
| `SELECT *` | select only needed columns | Less I/O; enables covering indexes |

**S25. Rewrite "employees above their department's average salary" using `AVG(salary) OVER (PARTITION BY dept)` instead of a correlated subquery, and explain why it's faster.** Table `employees(name, dept, salary)`. *Example:* Eng avg≈102.5 → Alice (120) passes; Sales avg≈91.7 → Judy (150) and Eve (95) pass.
<details><summary>Solution</summary>

```sql
SELECT name, dept, salary FROM (
  SELECT *, AVG(salary) OVER (PARTITION BY dept) AS dept_avg
  FROM employees
) t
WHERE salary > dept_avg;
```
The window computes each dept average **once**; the correlated-subquery version re-executes the average for every row.
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
- [StrataScratch — Python/Pandas DS interview questions](https://www.stratascratch.com/blog/python-pandas-interview-questions-for-data-science) · [DataLemur — SQL interview questions](https://datalemur.com/) · [LeetCode Database (Top SQL 50)](https://leetcode.com/studyplan/top-sql-50/)
- [The 7 recurring SQL interview patterns](https://datavidhya.com/blog/sql-data-engineering-interview-questions/) · [Gaps & islands](https://datavidhya.com/learn/sql/interview-patterns/gaps-and-islands/) · [Cohort & retention questions](https://letsdatascience.com/blog/sql-cohort-retention-interview-questions)
- [DataCamp — Top 30 pandas interview questions](https://www.datacamp.com/blog/top-python-pandas-interview-questions-and-answers) · [Top 99 SQL interview questions](https://www.datacamp.com/blog/top-sql-interview-questions-and-answers-for-beginners-and-intermediate-practitioners)
- [pandas documentation](https://pandas.pydata.org/docs/) · [PostgreSQL — SQL language](https://www.postgresql.org/docs/current/sql.html)

---

## Glossary

| Term | Simple explanation | Purpose |
|---|---|---|
| **DataFrame** | A pandas 2-D table of rows and columns | The core structure for tabular data work |
| **Series** | A single pandas column (1-D labeled array) | Result of selecting one column; input to many ops |
| **`groupby`** | Split rows into groups by a key, then aggregate/transform each group | Per-category summaries (avg per dept, count per user) |
| **`agg`** | Reduce each group to one value per column (sum, mean, etc.) | Produce a summary table, one row per group |
| **`transform`** | Return a value for every original row, aligned to the group | Fill/normalize within groups without collapsing rows |
| **`apply`** | Run a custom function over a group, column, or row | Flexible catch-all when agg/transform don't fit |
| **`merge` / join** | Combine two DataFrames on matching keys | Bring related data together (like SQL JOIN) |
| **`merge_asof`** | Join on nearest key rather than exact match | Time-series joins (latest price before a trade) |
| **`pivot` / `pivot_table`** | Reshape long data into a wide cross-tab | Rows×columns summaries |
| **`melt`** | Reshape wide data into long (unpivot) | Tidy format for analysis/plotting |
| **Vectorization** | Operating on whole arrays at once instead of Python loops | Faster, cleaner pandas/numpy code |
| **Window function** | SQL calc across a set of rows related to the current row, without collapsing them | Running totals, rankings, per-group comparisons |
| **`PARTITION BY`** | Splits window-function rows into groups | Restart the window per category |
| **`ROW_NUMBER` / `RANK` / `DENSE_RANK`** | Assign position numbers within a partition | Top-N per group, deduplication |
| **CTE (`WITH`)** | A named temporary result you can reference in a query | Break complex SQL into readable steps |
| **Recursive CTE** | A CTE that references itself | Hierarchies, sequences, gaps-and-islands |
| **`HAVING`** | Filter *after* grouping/aggregation | Keep groups meeting a condition (count ≥ 5) |
| **Gaps and islands** | Pattern for finding consecutive runs (or breaks) in ordered data | Sessionization, streak detection |
| **Cohort / retention** | Group users by start period and track behavior over time | Measure repeat engagement |
| **Sargable** | A predicate that can use an index | Faster queries; avoid wrapping indexed columns in functions |
| **Execution order** | The logical order SQL runs (FROM→WHERE→GROUP BY→HAVING→SELECT→ORDER BY) | Explains why aliases/aggregates behave as they do |

---

*Previous: [OOP in Python](02-oop-in-python.md) | Up: [Guide Home](../README.md)*
