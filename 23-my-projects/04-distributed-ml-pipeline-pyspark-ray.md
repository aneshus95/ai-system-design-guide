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

**Why Spark here:** Spark excels at **data parallelism** — the same operation across every row of a massive dataset — which is exactly ETL, feature engineering, and preprocessing at scale. PySpark DataFrames + **lazy evaluation** (compute triggered only on an action) let the optimizer plan efficient distributed execution over data that won't fit on one machine.

**What the features looked like:** joins of transactions with issuer/network/BIN attributes; **windowed/time-series aggregations** (e.g., rolling per-issuer and per-network approval rates); label construction (approved vs declined). A common production pattern is **Spark → Delta Lake** (versioned/ACID feature storage) → **MLflow** tracking, in a Bronze→Silver→Gold medallion layout.

> For the mechanics of *why* Spark is fast (driver/executors, lazy DAG, shuffles), see [22-data-engineering — Apache Spark Architecture](../22-data-engineering/01-apache-spark-architecture.md).

Sources: [Databricks — When to use Spark vs Ray](https://docs.databricks.com/aws/en/machine-learning/ray/spark-ray-overview) · [Databricks — feature engineering with Spark/Delta/MLflow](https://dev.to/jubinsoni/azure-databricks-for-feature-engineering-at-scale-with-apache-spark-delta-lake-and-mlflow-3k4n)

---

## Deep Dive 2 — Ray for Distributed Training (and Why Both)

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

The routing decision must happen **inside the authorization flow**, in real time — so features have to be available at millisecond latency and be **identical** to those used in training.

- A **feature store** keeps an **offline store** (history for training) and a low-latency **online store** (real-time serving), populated by **materialization jobs**.
- **Training–serving skew / parity:** one source of truth for feature definitions + identical transformation logic offline and online → the model sees the same features in prod as in training.
- **Point-in-time correctness:** offline training sets are built with point-in-time joins to avoid **label leakage**; the online store serves the latest feature vector by entity key (card/BIN/merchant) in **single-digit milliseconds**.

**Mapping:** Spark computes offline features → materialized into a KV online store (Redis/Aerospike-class) → at auth time, look up by entity, run low-latency inference (e.g., Ray Serve or a compiled model), emit the routing decision.

Sources: [Feast docs](https://docs.feast.dev/) · [Feast — solving training-serving skew](https://medium.com/@scoopnisker/solving-the-training-serving-skew-problem-with-feast-feature-store-3719b47e23a2)

---

## Deep Dive 5 — Modeling Details (Imbalance, Labels, Leakage)

- **Class imbalance:** declines are the minority class. Prefer **class weights / cost-sensitive learning** (XGBoost/LightGBM `scale_pos_weight`) over blind resampling; evaluate with **precision, recall, F1, PR-AUC** — **not accuracy**, which is misleading under imbalance.
- **Label:** binary approval (approved vs declined) per candidate route, or an approval probability used to rank routes.
- **Leakage:** use only features available **at authorization time** (nothing post-outcome).
- **Selection / feedback bias:** you only observe the outcome of the route you *actually took* — a **counterfactual / off-policy** problem. Naively training on logged routes bakes in the old policy's bias; addressing it (off-policy evaluation, exploration) is a strong senior-level talking point.

Sources: [Analytics Vidhya — class imbalance techniques](https://www.analyticsvidhya.com/blog/2020/07/10-techniques-to-deal-with-class-imbalance-in-machine-learning/)

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
