# Data Science Coding & SQL — Practice Questions

A practice bank for **Python (data-science / pandas)** and **SQL** coding interviews. Attempt each question before opening the solution. Assume `import pandas as pd`, `import numpy as np`. For SQL, examples use Postgres-flavored syntax.

## Table of Contents

- [Sample Data — Run This First](#sample-data--run-this-first)
- [Python — Fundamentals with a DS Flavor](#python--fundamentals-with-a-ds-flavor)
- [Python — Pandas Data Manipulation](#python--pandas-data-manipulation)
- [Python — NumPy & Vectorization](#python--numpy--vectorization)
- [Python — Implement DS Functions from Scratch](#python--implement-ds-functions-from-scratch)
- [Python — Harder & Real-Company Patterns](#python--harder--real-company-patterns)
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

## Python — Harder & Real-Company Patterns

**P23. (Shopify/Amazon) From `orders[customer_id, order_date, order_cost]`, find the customer(s) with the highest *daily* total order cost (sum per customer per day), and the date.**
<details><summary>Solution</summary>

```python
daily = df.groupby(['customer_id', 'order_date'], as_index=False)['order_cost'].sum()
daily[daily['order_cost'] == daily['order_cost'].max()]
```
Aggregate to daily totals first, then filter to the max. (`==max()` returns ties; `.nlargest(1)` would drop them.)
</details>

**P24. Multi-condition bucketing: label `amount` as 'low'/'mid'/'high' — efficiently, no `apply`.**
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

**P25. Add a column with each row's value as a % of its group total (`df[dept, salary]`).**
<details><summary>Solution</summary>

```python
df['pct_of_dept'] = df['salary'] / df.groupby('dept')['salary'].transform('sum') * 100
```
`transform('sum')` broadcasts the group total back to every row — no merge needed.
</details>

**P26. Time-based join: for each `trade[symbol, time]`, attach the most recent `quote[symbol, time, price]` at or before the trade (as-of join).**
<details><summary>Solution</summary>

```python
trades = trades.sort_values('time'); quotes = quotes.sort_values('time')
pd.merge_asof(trades, quotes, on='time', by='symbol', direction='backward')
```
`merge_asof` is the specialized tool for "nearest earlier match" joins (finance, event enrichment) — both frames must be sorted on the key.
</details>

**P27. One row per `(user, tag)` from `df[user, tags]` where `tags` is a comma-separated string.**
<details><summary>Solution</summary>

```python
df.assign(tag=df['tags'].str.split(',')).explode('tag')
# strip whitespace: .assign(tag=lambda d: d['tag'].str.strip())
```
`str.split` → list column → `explode` fans it to one row per element (the pandas "unnest").
</details>

**P28. Rank customers by total spend and return their percentile.**
<details><summary>Solution</summary>

```python
spend = df.groupby('customer_id')['amount'].sum()
pct = spend.rank(pct=True) * 100          # 0–100 percentile
```
`rank(pct=True)` gives the percentile directly; `method='dense'`/`'min'` controls tie behavior.
</details>

**P29. Count consecutive-day login streaks per user from `logins[user, date]` (distinct dates).**
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

**P30. Sum a numeric column — three ways, fastest last.**
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

**P31. Replace values via a mapping dict — efficiently.**
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

## SQL — Gaps & Islands and Sessionization

The highest-value advanced pattern: an **island** is a contiguous run of rows; a **gap** is the break between runs. Same shape powers login streaks, sessionization, consecutive purchases, and unchanged-price periods.

**S19. Longest streak of consecutive login days per user (`logins[user_id, login_date]`, distinct).**
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

**S20. Sessionize a clickstream: assign a session id where a >30-minute gap starts a new session (`events[user_id, event_time]`).**
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

**S21. Periods where a product's price stayed unchanged (`price_history[product_id, day, price]`).**
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

**S22. Monthly signup cohorts and how many users return in each subsequent month (`activity[user_id, activity_month]`).**
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

**S23. Day-1 retention: % of users active the day after their signup day.**
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

**S24. All employees under a given manager (the full reporting subtree).**
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

**S25. Rewrite "employees above their dept average" without a correlated subquery.**
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

*Previous: [OOP in Python](02-oop-in-python.md) | Up: [Guide Home](../README.md)*
