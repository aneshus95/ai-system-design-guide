# Distributed ML Pipeline (PySpark + Ray) — Transaction-Routing Model

> **My project.** Built a distributed ML pipeline — **PySpark** for feature engineering at scale, **Ray** for distributed training/tuning — to train a **transaction-routing model** for a fintech client that **optimizes card-network approval rate at real-time scale**.

## Table of Contents

- [The Narrative](#the-narrative)
- [What I Built — Architecture](#what-i-built--architecture)
- [Deep Dive 1 — PySpark for Feature Engineering at Scale](#deep-dive-1--pyspark-for-feature-engineering-at-scale)
- [Deep Dive 2 — Ray for Distributed Training (and Why Both)](#deep-dive-2--ray-for-distributed-training-and-why-both)
- [Deep Dive 3 — Transaction Routing & Approval Rate](#deep-dive-3--transaction-routing--approval-rate)
- [Deep Dive 4 — Real-Time Serving & Feature Stores](#deep-dive-4--real-time-serving--feature-stores)
- [Deep Dive 5 — Modeling Details (Imbalance, Labels, Leakage)](#deep-dive-5--modeling-details-imbalance-labels-leakage)
- [Interview Q&A](#interview-qa)
- [Honest Caveats](#honest-caveats)
- [References](#references)

---

## The Narrative

**Situation.** When a card payment comes in, it can often be routed across more than one card network, and the choice affects whether the issuer **approves or declines** it. Static, rule-based routing leaves approvals (and revenue) on the table — even a fraction of a percent of extra approvals is large money at scale. The training data was billions of historical transactions — far past a single machine.

**Task.** Learn, from history, **which route maximizes the probability of approval** for each transaction, and serve that decision **inside the authorization flow** (real time).

**Action.** I split the pipeline by the tool each part is best at: **PySpark** for distributed ETL and feature engineering over the full transaction history (joins, rolling per-issuer/per-network approval-rate windows, label construction), then handed the prepared dataset to **Ray** (Ray Data → Ray Train) for **distributed model training** and **Ray Tune** for hyperparameter search. Features were materialized to a **low-latency online store** so the model could score at transaction time with the *same* features it trained on.

**Result.** An ML-driven routing model that predicts per-network approval probability and routes to maximize expected approval at real-time latency — replacing static rules with learned patterns across issuer, BIN, amount, geography, and time.

---

## What I Built — Architecture

```
 billions of historical transactions
        │
        ▼
 ┌───────────────────────────┐
 │ PYSPARK  (data parallel)  │  clean · join · rolling approval-rate windows
 │  ETL + feature engineering│  · label = approved/declined · write to Delta
 └─────────────┬─────────────┘
               │  from_spark / RayDP  (hand-off)
               ▼
 ┌───────────────────────────┐
 │ RAY  (task/compute parallel)                    │
 │  Ray Data → Ray Train (distributed training)     │
 │  Ray Tune (hyperparameter sweep)                 │
 └─────────────┬─────────────┘
               │  materialize features
               ▼
 ┌───────────────────────────┐        ┌──────────────────────────┐
 │ OFFLINE store (Delta)     │  ───►  │ ONLINE store (KV, ms reads)│
 │ history for training      │        │ real-time feature lookups  │
 └───────────────────────────┘        └────────────┬─────────────┘
                                                    ▼
                                   auth-time inference → routing decision
```

---

## Deep Dive 1 — PySpark for Feature Engineering at Scale

> **Why (the rationale):** Billions of transaction rows don't fit on a single machine for ETL. Spark's distributed DataFrame model — lazy evaluation, query optimizer, data parallelism across executors — handles joins, windowed aggregations, and label construction at this scale without custom distributed coding. The Delta Lake output adds ACID transactions and time-travel for safe point-in-time training sets.
> **When to use:** When the feature engineering dataset exceeds single-machine memory (typically tens to hundreds of GB+), or when joins between large tables (transactions × issuer attributes) are required. For smaller datasets, pandas or DuckDB is simpler and faster.
> **Nuances & gotchas:** Wide window aggregations (rolling per-issuer approval rates over large lookbacks) can cause expensive shuffles and skew if some issuers have orders of magnitude more transactions. Lazy evaluation means bugs surface at action time, not at transformation definition — debugging requires careful inspection of the execution plan (`df.explain()`). Spark's JVM overhead makes it poorly suited for complex Python-native ML logic (use Ray/PyTorch for that).

**Why Spark here:** Spark excels at **data parallelism** — the same operation across every row of a massive dataset — which is exactly ETL, feature engineering, and preprocessing at scale. PySpark DataFrames + **lazy evaluation** (compute triggered only on an action) let the optimizer plan efficient distributed execution over data that won't fit on one machine.

**What the features looked like:** joins of transactions with issuer/network/BIN attributes; **windowed/time-series aggregations** (e.g., rolling per-issuer and per-network approval rates); label construction (approved vs declined). A common production pattern is **Spark → Delta Lake** (versioned/ACID feature storage) → **MLflow** tracking, in a Bronze→Silver→Gold medallion layout.

> For the mechanics of *why* Spark is fast (driver/executors, lazy DAG, shuffles), see [22-data-engineering — Apache Spark Architecture](../22-data-engineering/01-apache-spark-architecture.md).

Sources: [Databricks — When to use Spark vs Ray](https://docs.databricks.com/aws/en/machine-learning/ray/spark-ray-overview) · [Databricks — feature engineering with Spark/Delta/MLflow](https://dev.to/jubinsoni/azure-databricks-for-feature-engineering-at-scale-with-apache-spark-delta-lake-and-mlflow-3k4n)

---

## Deep Dive 2 — Ray for Distributed Training (and Why Both)

> **Why (the rationale):** Spark is optimized for *data* parallelism (same operation across every row); it's not designed for the *compute* parallelism of distributed model training, where workers need to exchange gradients and share model state. Ray's task/actor model, Ray Train's distributed trainer wrappers, and Ray Tune's parallel trial execution are each optimized for this compute-bound workload. Using both keeps each framework in its lane.
> **When to use:** Use Spark alone if ETL and lightweight ML (Spark MLlib) are sufficient. Use Ray alone if data fits in memory and training is the bottleneck. Use both when the dataset requires distributed ETL at Spark scale *and* training requires distributed compute at Ray scale — the canonical "big data meets big model" pattern.
> **Nuances & gotchas:** The Spark→Ray data hand-off (via `from_spark` / RayDP) is a potential bottleneck and memory pressure point — if the serialized dataset is very large, transferring it between frameworks can dominate pipeline runtime. RayDP (Spark on Ray) simplifies co-location but adds operational complexity. Ray Tune's distributed search requires careful resource allocation to avoid worker contention on shared GPU nodes.

**Ray** scales Python/ML from a laptop to a cluster. Its libraries compose end-to-end:
- **Ray Data** — distributed/streaming data processing that feeds training.
- **Ray Train** — distributed training/fine-tuning, wrapping PyTorch/XGBoost/etc. under one Trainer API.
- **Ray Tune** — distributed hyperparameter search with efficient search algorithms.
- **Ray Serve** — model serving.

**Why both Spark *and* Ray (the core "why" question):** they're optimized for different parallelism.
- **Spark = data parallelism** → ETL, feature engineering, MLlib.
- **Ray = task/compute parallelism** → distributed training, hyperparameter search, RL, simulation — where Spark is less optimized.

**How they connect:** the recommended "data-first" pattern is Spark for retrieval/preprocessing, then hand off to Ray for compute-heavy training via `from_spark` (Spark → Ray Data in memory), writing results back to Delta afterward. **RayDP** ("Spark on Ray") can run PySpark inside a Ray cluster so Spark's output feeds training in the same app.

**Defensible one-liner:** *"Spark does distributed ETL/feature engineering; I pass the prepared dataset to Ray (Ray Data → Ray Train) for distributed training and Ray Tune for the sweep. Ray handles the compute-bound training that Spark isn't optimized for."*

Sources: [Ray docs — Train/Tune/Data](https://docs.ray.io/en/latest/index.html) · [Databricks — Combine Ray and Spark](https://docs.databricks.com/aws/en/machine-learning/ray/connect-spark-ray) · [RayDP](https://docs.ray.io/en/latest/ray-more-libs/raydp.html)

---

## Deep Dive 3 — Transaction Routing & Approval Rate

**Background (why routing exists):** the **Durbin Amendment** (Dodd-Frank, 2010) requires **debit** transactions to be routable over **at least two unaffiliated networks** — which is what makes merchant network choice ("routing") possible; a 2023 rule extended this routing choice to **card-not-present (online) debit**. **Least-cost routing (LCR)** picks the cheapest eligible network per transaction.

**What "card-network approval rate" means:** the **authorization rate** = the % of submitted transactions the issuer/network approves. Even a **0.5% lift** is millions in recovered revenue at scale; online transactions run ~10% lower than in-person due to more conservative issuer fraud logic. The most common decline is the generic **"Do not honor" (05)**.

**How ML optimizes it:** train on historical transactions to **predict the probability a given network/route will approve a specific transaction**, learning patterns across issuer, BIN, region, amount band, and time-of-day — then route to the network with the highest expected approval (optionally under a cost/LCR constraint → a **multi-objective** framing is the strongest answer). Industry reports 2–5% acceptance lift vs static rules (e.g., Stripe Adaptive Acceptance, Adyen Uplift). Complementary levers: **network tokenization** and **account updater** keep credentials current and raise approval odds.

Sources: [Congress.gov CRS — debit interchange & routing](https://www.congress.gov/crs-product/R41913) · [Federal Register — debit routing 2023](https://www.federalregister.gov/documents/2023/11/14/2023-24034/debit-card-interchange-fees-and-routing) · [Stripe — optimizing authorization rates](https://stripe.com/guides/optimizing-authorization-rates)

---

## Deep Dive 4 — Real-Time Serving & Feature Stores

> **Why (the rationale):** The authorization flow is latency-critical (typically <100 ms end-to-end). Computing features on-the-fly from raw transaction history at inference time is infeasible at that latency. A feature store solves this by pre-materializing features offline and serving them from a low-latency KV store, while enforcing the same feature transformation logic in both environments — eliminating training–serving skew.
> **When to use:** Feature stores are necessary when: (a) inference latency requirements are tight (ms-level), (b) features require expensive aggregation over history (rolling windows), and (c) the same features must be consistent between training and serving. Simpler point-in-time features computed cheaply at inference time may not need a store.
> **Nuances & gotchas:** The materialization job introduces freshness lag — features in the online store reflect the state as of the last materialization, not real-time. For high-velocity signals (per-card approval patterns changing by the hour), this staleness matters and requires streaming materialization. Point-in-time joins for training are subtle to implement correctly; a bug here causes label leakage that inflates offline metrics but fails in production.

The routing decision must happen **inside the authorization flow**, in real time — so features have to be available at millisecond latency and be **identical** to those used in training.

- A **feature store** keeps an **offline store** (history for training) and a low-latency **online store** (real-time serving), populated by **materialization jobs**.
- **Training–serving skew / parity:** one source of truth for feature definitions + identical transformation logic offline and online → the model sees the same features in prod as in training.
- **Point-in-time correctness:** offline training sets are built with point-in-time joins to avoid **label leakage**; the online store serves the latest feature vector by entity key (card/BIN/merchant) in **single-digit milliseconds**.

**Mapping:** Spark computes offline features → materialized into a KV online store (Redis/Aerospike-class) → at auth time, look up by entity, run low-latency inference (e.g., Ray Serve or a compiled model), emit the routing decision.

Sources: [Feast docs](https://docs.feast.dev/) · [Feast — solving training-serving skew](https://medium.com/@scoopnisker/solving-the-training-serving-skew-problem-with-feast-feature-store-3719b47e23a2)

---

## Deep Dive 5 — Modeling Details (Imbalance, Labels, Leakage)

> **Why (the rationale):** Class imbalance, leakage, and counterfactual bias are the three failure modes specific to this problem. Accuracy is misleading under imbalance (the majority "approved" class dominates). Leakage from features available only post-authorization inflates offline metrics but is unavailable in production. Counterfactual bias means the model learns the *old policy's* routing choices, not the optimal routing — the core off-policy problem.
> **When to use:** Cost-sensitive learning (class weights) is the first-line response to class imbalance when the minority class matters but isn't extremely rare; SMOTE/resampling adds risk of overfitting to synthetic points. Off-policy evaluation is necessary whenever training data is generated by a prior policy and you want to evaluate a new one without running it live.
> **Nuances & gotchas:** The counterfactual/selection bias problem is hard to fully solve without online experimentation (routing some fraction of transactions to non-greedy routes to gather counterfactual labels). Naive cost-sensitive learning sets `scale_pos_weight` based on the training set ratio, but the right weight is also a function of business costs (decline costs vs false-positive costs), which are rarely symmetric.

- **Class imbalance:** declines are the minority class. Prefer **class weights / cost-sensitive learning** (XGBoost/LightGBM `scale_pos_weight`) over blind resampling; evaluate with **precision, recall, F1, PR-AUC** — **not accuracy**, which is misleading under imbalance.
- **Label:** binary approval (approved vs declined) per candidate route, or an approval probability used to rank routes.
- **Leakage:** use only features available **at authorization time** (nothing post-outcome).
- **Selection / feedback bias:** you only observe the outcome of the route you *actually took* — a **counterfactual / off-policy** problem. Naively training on logged routes bakes in the old policy's bias; addressing it (off-policy evaluation, exploration) is a strong senior-level talking point.

Sources: [Analytics Vidhya — class imbalance techniques](https://www.analyticsvidhya.com/blog/2020/07/10-techniques-to-deal-with-class-imbalance-in-machine-learning/)

---

## Key Decisions & Tradeoffs

| Decision | Options considered | Why chosen | Tradeoff accepted |
|---|---|---|---|
| **PySpark for feature engineering + Ray for training vs one framework** | Spark MLlib end-to-end, Ray Data + Ray Train end-to-end, Dask | Spark excels at data parallelism (ETL, windowed joins over billions of rows); Ray excels at compute parallelism (distributed training, hyperparameter sweeps) — each framework stays in its lane | Spark→Ray data hand-off (via `from_spark`/RayDP) is a serialization bottleneck; operational complexity of running and coordinating two distributed systems |
| **Delta Lake as the offline feature store vs raw Parquet** | Raw Parquet on S3/ADLS, Iceberg, Hive tables | Delta adds ACID transactions, schema enforcement, and time-travel (point-in-time snapshots) needed for correct point-in-time training set construction without label leakage | Slight write overhead vs raw Parquet; requires Delta-compatible compute (Spark, Databricks); Delta log can grow large and needs periodic vacuuming |
| **Class weights / cost-sensitive learning vs SMOTE resampling** | SMOTE, random oversampling, undersampling, ensemble methods | Class weights (XGBoost `scale_pos_weight`) are lower variance, don't generate synthetic samples, and are supported natively in Ray Train's distributed trainers | The right weight value is ambiguous when business costs of false positives and false negatives are asymmetric; doesn't synthesize hard minority examples at the decision boundary |
| **Approval-probability regression vs binary classification** | Binary classifier (approved/not), multi-class (per network), probability regression | Predicting approval probability per network allows routing to the maximum-expected-approval network under an optional LCR cost constraint — enables multi-objective optimization rather than a hard argmax | Calibration of probability outputs is critical and adds evaluation complexity; a well-calibrated classifier is harder to build than a ranked binary one |
| **Online store for real-time feature serving vs compute-at-inference** | Compute features on-the-fly from raw logs at auth time, columnar cache | Real-time feature computation over rolling window aggregations (per-issuer/per-BIN approval rates) is infeasible in <100 ms; pre-materialized KV lookup is single-digit-ms | Materialization lag — online store reflects state as of the last materialization, not real time; high-velocity signals go stale; streaming materialization needed for very fresh features |
| **Ray Tune for hyperparameter search vs grid search** | Sequential grid search, random search, Optuna | Ray Tune distributes trials in parallel across the cluster using efficient search algorithms (ASHA early stopping); finds better hyperparameters faster than sequential search at the same compute budget | Requires a Ray cluster sized for concurrent trials; ASHA's early stopping can prematurely kill promising long-training-curve models |

---

## Production Issues & Fixes

*Representative issues for this system class — mapped to what I actually hit; tailor to your own incidents.*

**1. Data skew on high-volume issuer BINs causing Spark shuffle OOM and stragglers**
- **Symptom:** The rolling per-issuer approval-rate window aggregation job consistently failed or ran 10× slower than expected; the job tracker showed one executor stuck at 99% while others finished.
- **Root cause:** A handful of top-10 issuer BINs accounted for ~40% of all transactions. When Spark shuffled by `issuer_bin` for the window aggregation, all rows for the hot BINs landed on single executors — causing OOM kills and straggler tasks.
- **Fix:** Salted the shuffle key (appended a random bucket suffix 0–9 to `issuer_bin` before the aggregation, then summed the buckets); configured `spark.sql.adaptive.skewJoin.enabled=true` (Spark 3.x AQE) to auto-detect and split skewed partitions.
- **Prevention/monitoring:** Add a partition skew check (`df.groupBy(partition_key).count()` top-10) before heavy shuffles; alert when the max-to-median partition size ratio exceeds 5×; set executor memory headroom to 2× the expected max partition.

**2. Training–serving skew: feature logic diverged between PySpark pipeline and online serving code**
- **Symptom:** The model's offline PR-AUC was 0.82; production approval-lift was near zero — routing decisions were essentially random relative to baseline.
- **Root cause:** The PySpark feature pipeline computed rolling approval rates over a 7-day trailing window; the online serving code recomputed a simpler 24-hour window (an unreviewed shortcut made during a deadline). The model had never seen the 24-hour features during training.
- **Fix:** Refactored feature logic into a shared Python library imported by both the Spark job and the online serving code; added a feature-parity integration test that runs both paths on the same sample and asserts output agreement within tolerance.
- **Prevention/monitoring:** Run the parity test in CI on every merge; add a live monitoring job that computes feature values for a sample of auth-time requests both online and offline and alerts on mean absolute deviation >1% for rolling-window features.

**3. Spot-instance interruptions mid-Ray-Tune trial causing full sweep restart**
- **Symptom:** The hyperparameter sweep on spot GPU instances was interrupted 3 times in one week, each time losing hours of trial results and restarting from scratch.
- **Root cause:** Ray Tune was not configured with fault-tolerant checkpointing per trial; spot reclamations killed the trial workers, and Ray re-queued them with no saved state.
- **Fix:** Enabled `ray.tune.RunConfig(storage_path=..., checkpoint_config=CheckpointConfig(checkpoint_frequency=1))` so each trial checkpoints after every epoch; surviving trials resume from their last checkpoint on restart. Mixed spot and on-demand (10% on-demand) to keep the Ray head node stable.
- **Prevention/monitoring:** Track spot interruption rate per worker; alert when >20% of trial workers are reclaimed in a 30-minute window; log checkpoint save/restore events per trial.

**4. Label leakage from a feature that used post-authorization information**
- **Symptom:** Offline PR-AUC looked excellent (0.91); production approval lift was far below the expected 2–3%, and the model heavily over-predicted approval for one network.
- **Root cause:** A "network response code frequency" feature was computed over a trailing window that included the current transaction's response code — information only available after the authorization decision (the outcome being predicted). The model learned to exploit this signal directly.
- **Fix:** Strictly enforced point-in-time correctness in the Delta Lake join: each training row is joined to feature values as of `auth_timestamp - 1 second`; added a leakage audit that computes feature-outcome correlations on a hold-out set and flags any feature with correlation >0.9 with the label.
- **Prevention/monitoring:** Run the leakage audit as a CI step after any feature addition; require all new features to include a timestamp provenance tag in the feature catalog; track feature–label correlation in offline experiments before promoting to production.

**5. Class imbalance — model predicting "approved" for nearly all transactions**
- **Symptom:** After deployment, the model's routing was almost identical to the baseline (always pick the historically highest-approval network); per-network probability scores clustered near 0.85–0.90 for all routes with little differentiation.
- **Root cause:** Declines were ~8% of transactions; the model trained without class weighting and found it could achieve >90% accuracy by predicting approval for everything. The decision boundary never learned to distinguish routes for the minority decline class.
- **Fix:** Set `scale_pos_weight` = (# approved) / (# declined) ≈ 11.5 for XGBoost; re-evaluated using PR-AUC and F1 on the minority class rather than accuracy; confirmed per-network differentiation in probability calibration plots before re-deploying.
- **Prevention/monitoring:** Add PR-AUC and minority-class F1 as gating metrics for model promotion; alert when the model's per-network probability spread (max − min) averages below 0.05 on the eval set (indicates failure to differentiate routes).

**6. Feature freshness lag — online store staleness causing incorrect routing during issuer incidents**
- **Symptom:** During a 4-hour outage by a major issuer, the model continued routing significant volume to that issuer's network because its online-store approval rate feature still reflected yesterday's (pre-outage) high approval rate.
- **Root cause:** The materialization job ran every 6 hours; the feature values in the online store were up to 6 hours stale. The model had no signal that the issuer was currently experiencing a high decline rate.
- **Fix:** Switched the rolling per-issuer approval-rate feature to a streaming materialization pipeline (Kafka → Flink → online store), reducing freshness lag to <5 minutes for the most time-sensitive features; kept the batch pipeline for slower-moving BIN-level features.
- **Prevention/monitoring:** Track online-store feature age per entity key; alert when any high-velocity feature is >15 minutes stale; add a "last materialized" timestamp to the serving payload and log it alongside each routing decision.

---

## Interview Q&A

**Q: Why both Spark and Ray — isn't one enough?**
Spark is optimized for data-parallel ETL/feature engineering; Ray for task/compute-parallel work (distributed training, tuning) where Spark is weaker. I use Spark to prepare features at scale and hand off to Ray Data → Ray Train for training and Ray Tune for the sweep, crossing via `from_spark`/RayDP.

**Q: How do you handle class imbalance in approval prediction?**
Declines are the minority. I use class weights / cost-sensitive learning (faster and less overfit-prone than resampling) and evaluate with PR-AUC, precision, recall, F1 — never accuracy.

**Q: How do you serve at real-time latency?**
Precompute features offline in Spark, materialize to a low-latency online store for single-digit-ms lookups, and score inside the auth flow. The feature store guarantees offline/online parity.

**Q: How do you avoid training–serving skew?**
Single source of truth for feature definitions, identical transformation logic offline and online, point-in-time-correct joins for training (no leakage), and the online store materialized from the same pipeline.

**Q: How does the model actually improve approval rate?**
It predicts per-network approval probability from historical patterns (issuer, BIN, amount, geography, time) and routes to the highest expected approval, optionally under a cost/LCR constraint. Industry lift is ~2–5% over static rules.

**Q: What's the trickiest ML pitfall here?**
The counterfactual/selection bias — you only see the outcome of the route you chose. Training naively on logged data inherits the old policy's bias; I'd account for it with off-policy evaluation and some exploration in routing.

---

## Honest Caveats

- **No public source ties PySpark+Ray specifically to card-approval routing** — that's this project's synthesis; the building blocks (Spark ETL, Ray training, feature stores, ML routing) are each well-sourced.
- **Vendor uplift numbers (2–5%, 6%, etc.) are marketing/third-party** — cite as "reported," not guaranteed.
- **Durbin routing (two-network, LCR) applies to debit;** credit-card routing choice is more limited — confirm debit vs credit scope. Be ready to state whether the model optimizes approval only, or approval subject to a cost constraint (multi-objective).

---

## References

- [Ray — docs (Train / Tune / Data)](https://docs.ray.io/en/latest/index.html) · [RayDP (Spark on Ray)](https://docs.ray.io/en/latest/ray-more-libs/raydp.html)
- [Databricks — When to use Spark vs Ray](https://docs.databricks.com/aws/en/machine-learning/ray/spark-ray-overview) · [Combine Ray and Spark](https://docs.databricks.com/aws/en/machine-learning/ray/connect-spark-ray)
- [Congress.gov CRS — Regulation of Debit Interchange Fees (Durbin)](https://www.congress.gov/crs-product/R41913) · [Federal Register — Debit routing 2023](https://www.federalregister.gov/documents/2023/11/14/2023-24034/debit-card-interchange-fees-and-routing)
- [Stripe — Optimizing authorization rates](https://stripe.com/guides/optimizing-authorization-rates)
- [Feast — feature store docs](https://docs.feast.dev/)

---

## Glossary

| Term | Simple explanation | Purpose |
|---|---|---|
| **PySpark** | Python API for Apache Spark — a distributed data processing engine that splits work across many machines | Handles feature engineering over billions of rows that won't fit on one machine |
| **Ray** | A Python framework for distributing compute-heavy tasks (training, tuning, simulation) across a cluster | Scales model training and hyperparameter search beyond what a single machine can do |
| **Ray Data** | Ray's distributed data-loading and preprocessing library that feeds Ray Train pipelines | Bridges from raw data or Spark output to the training loop without a memory bottleneck |
| **Ray Train** | Ray's distributed training library wrapping PyTorch, XGBoost, and other frameworks | Runs the same training job across multiple GPUs or machines in parallel |
| **Ray Tune** | Ray's distributed hyperparameter search library with efficient search algorithms | Finds the best model configuration faster than sequential grid search |
| **Ray Serve** | Ray's model-serving layer for deploying trained models as scalable REST endpoints | Serves the trained routing model at real-time latency inside the auth flow |
| **RayDP** | A library that runs PySpark inside a Ray cluster so Spark output feeds directly into Ray training | Eliminates data-hand-off overhead when running both Spark and Ray together |
| **Data Parallelism** | Running the same operation on every row of a dataset in parallel across nodes | Spark's strength; ideal for ETL and feature engineering over large datasets |
| **Task/Compute Parallelism** | Distributing compute-heavy tasks (model training iterations, trials) across workers | Ray's strength; ideal for distributed training and hyperparameter sweeps |
| **ETL (Extract, Transform, Load)** | The pipeline that reads raw data, cleans and transforms it, and writes it to a target store | The first stage of any ML pipeline; produces the feature dataset for training |
| **Lazy Evaluation** | Deferring computation until an action is explicitly triggered | Allows Spark to optimize the full computation plan before running any work |
| **Delta Lake** | An open-source storage layer on top of Parquet files that adds ACID transactions and versioning | Stores features reliably with schema enforcement and time-travel for point-in-time training |
| **Feature Store** | A system with an offline store (for training) and an online store (for serving) sharing the same feature definitions | Eliminates training–serving skew by enforcing identical feature logic in both environments |
| **Training–Serving Skew** | A mismatch between feature values computed offline during training and those computed online at inference | Causes the model to perform worse in production than in evaluation |
| **Point-in-Time Join** | Joining features to labels using only data that was available before the label's timestamp | Prevents leakage of future information into the training set |
| **Online Store** | A low-latency key-value store (e.g., Redis) that serves pre-materialized feature vectors at inference time | Enables single-digit millisecond feature lookups inside the authorization flow |
| **Transaction Routing** | Choosing which card network to route a payment through at authorization time | Optimizing routing for approval rate recovers revenue lost to unnecessary declines |
| **Authorization Rate** | The percentage of submitted transactions that the issuer or network approves | The primary business metric; even a 0.5% lift is significant at scale |
| **Least-Cost Routing (LCR)** | Routing a payment to the cheapest eligible network | A constraint that may conflict with approval-rate maximization; motivates multi-objective framing |
| **Durbin Amendment** | A 2010 U.S. law (Dodd-Frank) requiring debit transactions to be routable over at least two unaffiliated networks | The legal basis for merchant network choice in debit routing |
| **Class Imbalance** | When one target class (e.g., declined transactions) is much rarer than the other | Naive accuracy metrics are misleading; requires cost-sensitive learning or PR-AUC |
| **PR-AUC (Precision-Recall AUC)** | Area under the precision-recall curve; more informative than ROC-AUC under class imbalance | Measures model quality for the minority class without being inflated by easy negatives |
| **Counterfactual / Off-Policy Bias** | The problem that you only observe outcomes for the routes you actually chose, not for alternatives | Training on logged data bakes in the old policy's biases; requires off-policy evaluation to correct |
| **BIN (Bank Identification Number)** | The first digits of a card number identifying the issuing bank and card type | A key feature for routing models; different issuers have different approval patterns by network |

---

*Previous: [LangGraph Coding Agent](03-langgraph-coding-agent-with-rag.md) | Next: [Graph RAG over BIAN](05-graph-rag-over-bian.md) | Up: [Guide Home](../README.md)*
