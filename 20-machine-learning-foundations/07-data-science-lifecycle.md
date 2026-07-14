# The Data Science Lifecycle — End-to-End Flow of Any ML Problem

Interviewers love the question *"walk me through how you'd approach a data science problem end to end."* The mistake is to jump straight to "train a model." A strong answer starts at the **business problem** and ends at a **deployed, monitored system that loops back to improve** — with modeling as just one stage in the middle. This page walks that full flow intuitively, stage by stage. (Production/MLOps — deployment, serving, monitoring, retraining — get their own deep dive in [ML in Production](08-ml-in-production-and-mlops.md).)

## Table of Contents

- [The Big Idea: It's a Loop, Not a Line](#the-big-idea-its-a-loop-not-a-line)
- [Stage 1 — Problem Framing](#stage-1--problem-framing)
- [Stage 2 — Data Collection & Understanding](#stage-2--data-collection--understanding)
- [Stage 3 — Exploratory Data Analysis (EDA)](#stage-3--exploratory-data-analysis-eda)
- [Stage 4 — Data Prep & Feature Engineering](#stage-4--data-prep--feature-engineering)
- [Stage 5 — Modeling](#stage-5--modeling)
- [Stage 6 — Evaluation](#stage-6--evaluation)
- [Stage 7 & 8 — Deployment & Monitoring](#stage-7--8--deployment--monitoring)
- [The Three Things That Kill Projects](#the-three-things-that-kill-projects)
- [Interview Questions](#interview-questions)
- [Glossary](#glossary)
- [References](#references)

---

## The Big Idea: It's a Loop, Not a Line

Every framework — **CRISP-DM**, Microsoft's **TDSP**, Google's ML lifecycle, Chip Huyen's "iterative process" — draws the same thing: **a circle**. You don't march through once; you **loop back** whenever the data says your question was wrong, the model underperforms, or production drifts.

```
        ┌──────────────────────────────────────────────────────────┐
        │                                                          ▼
 (1) FRAME ──► (2) DATA ──► (3) EDA ──► (4) FEATURES ──► (5) MODEL ──► (6) EVALUATE
    business      collect      explore     clean/build     baseline      metrics tied
    problem →     · label ·    · outliers   features ·      → tune ·      to business ·
    ML task       split ·      · missing ·  NO LEAKAGE      CV · track    slice · fairness
      ▲           leakage?     hypotheses                                     │
      │                                                                      │ good enough?
      │  loop back if the numbers say the framing/data/features were wrong   │  no ──┘
      │                                                                      │  yes
      └──────────── (8) MONITOR ◄──── (7) DEPLOY ◄────────────────────────────┘
                    drift · decay              serve · A/B · rollback
                    → retrain (loops to Stage 2)
```

The three canonical frameworks map onto each other:

| | CRISP-DM | TDSP (Microsoft) | Google / Huyen |
|---|---|---|---|
| Stages | Business → Data Understanding → Data Prep → Modeling → Evaluation → Deployment | Business → Data Acquisition → Modeling → Deployment → **Customer Acceptance** | Scoping → Data → Modeling → Deployment → Monitoring → (loop) |
| Adds | the classic reference | team roles, git structure, an acceptance gate | explicit "is ML even the right tool?" + monitoring/governance |

Sources: [CRISP-DM](https://www.datascience-pm.com/crisp-dm-2/) · [Microsoft TDSP](https://learn.microsoft.com/en-us/azure/architecture/data-science-process/lifecycle) · [Google — ML project phases](https://developers.google.com/machine-learning/managing-ml-projects/phases)

---

## Stage 1 — Problem Framing

**The intuition:** before *any* data, turn a fuzzy business goal into a crisp, measurable ML problem — and check ML is even the right hammer.

- **Is ML the right tool at all?** If a rule, heuristic, or SQL query solves it, **don't use ML**. ML earns its cost only when the pattern is complex, changing, and hard to hand-code. ([Google — ML framing](https://developers.google.com/machine-learning/problem-framing/ml-framing))
- **Business KPI ≠ ML metric — the #1 framing distinction.** The *business metric* is revenue / churn % / conversion; the *ML metric* is accuracy / F1 / RMSE. Projects fail by optimizing the ML metric while the business needle doesn't move (a recommender going 85%→92% accuracy with **zero revenue lift** is useless). **Always connect the two, and re-check at Stage 6.** Define success on three levels: business ("churn −5%"), ML ("F1 > 0.8"), and economic ("positive ROI net of infra cost").
- **Frame the task type:** classification (predict the decision when thresholds are fixed), regression (predict the number when thresholds are dynamic — but beware, a regressor doesn't know your product thresholds), ranking (search/recs), or forecasting (time series).
- **Define the target/label carefully.** A *direct* label is ideal. A *proxy* label (when you can't measure the true outcome) is risky — **no proxy is perfect**. Classic trap: using "video shared" as a proxy for "video is good" fails, because people share videos they dislike.
- **Pin the success threshold and a measurement horizon** up front, so "did it work?" isn't argued after the fact.

> **One-liner:** *Frame the business outcome first, decide if ML is warranted, choose the task type, define a trustworthy label, and tie the ML metric to a business metric — that link is what most failed projects are missing.*

---

## Stage 2 — Data Collection & Understanding

**The intuition:** get the data that can actually answer the question — and set up the splits so you don't fool yourself later.

- **Sources & labeling** — identify sources, then get labels (direct, human annotation, or weak/programmatic labeling). Labeling cost is often the real feasibility constraint.
- **Data quality first pass** — missing values, duplicates, type/unit mismatches, encodings.
- **Splitting — the part that separates pros from amateurs:**
  - Hold the **test set untouched** until final selection — no test data may touch preprocessing, feature selection, tuning, or training.
  - **Time-ordered data → temporal split** (train on past, test on future) to simulate deployment. A random split leaks the future.
  - **Correlated rows (same user/patient) → grouped split**, or the same entity appears in train *and* test.
- **Data leakage — the recurring killer** (watch for it here *and* in Stage 4):
  - **Target leakage** — a feature encodes the answer / isn't available at prediction time.
  - **Temporal leakage** — a future value used as a predictor.
  - **Preprocessing leakage** — computing scaling/encoding stats on the *full* dataset before splitting (a peek at the test distribution).
  - **Symptom:** suspiciously great validation scores that collapse in production.

Sources: [Data leakage (scikit-learn common pitfalls)](https://scikit-learn.org/stable/common_pitfalls.html) · [Train/val/test & leakage](https://gtracademy.org/train-validation-test-splits-and-data-leakage-in-practice/)

---

## Stage 3 — Exploratory Data Analysis (EDA)

**The intuition:** *look at the data before you model it* — build a feel for shape, gaps, and relationships, and let it generate hypotheses.

- **Distributions** — histograms/box plots for skew, spread, outliers (delivery times, prices are usually right-skewed → maybe log-transform).
- **Outliers** — detect (IQR, z-score, box plots) but **treat as potential signal, not automatically errors**.
- **Missingness** — diagnose the *pattern* ([MCAR/MAR/MNAR](01-classical-ml-algorithms.md)) before choosing to drop vs impute.
- **Correlations & target relationships** — heatmaps and grouped stats to find real predictors and multicollinearity.
- **Document every assumption and insight** — EDA is iterative and feeds feature ideas.

> See the dedicated visuals in [Classical ML — EDA](01-classical-ml-algorithms.md) and [Statistics & Probability](03-statistics-and-probability.md).

---

## Stage 4 — Data Prep & Feature Engineering

**The intuition:** models are only as good as their features — and **this stage eats the most time** (surveys range 15–80%; ~45% is the most defensible figure — it's consistently the single largest, least-loved phase).

- **Operations:** cleaning, imputation, encoding (one-hot/target/ordinal), scaling, feature creation, feature selection, dimensionality reduction.
- **THE leakage rule (memorize it):** **fit every transformer on TRAIN only, then transform val/test.** Imputers, scalers, encoders, and feature selectors all have separate `fit`/`transform`. Fitting a scaler on the whole dataframe in pandas → leakage.
- **Use pipelines** (`Pipeline` + `ColumnTransformer`) so preprocessing is bundled with the model and fit only on each training fold — `GridSearchCV` over a pipeline is the correct, leak-free way to tune.
- **Feature engineering is where domain knowledge pays off** — the right engineered feature (ratio, time-since-event, aggregate) often beats a fancier model.

> **Watch for training-serving skew:** the exact transforms used in training must run identically at serving time — the #1 reason a good offline model fails in prod. A [feature store](08-ml-in-production-and-mlops.md#feature-stores--training-serving-skew) solves this.

---

## Stage 5 — Modeling

**The intuition:** **baseline first, always** — then earn every bit of added complexity.

- **Start with a dumb baseline** (`DummyClassifier` = majority class; or a simple logistic/linear model). It (a) sets the bar a complex model must beat, (b) proves the end-to-end pipeline works, (c) reveals whether complexity actually helps. A fancy model that barely beats "predict the mean" is a red flag.
- **Cross-validation:** K-Fold (5/10); **Stratified** K-Fold for imbalanced classes; **temporal/grouped** CV when data is time-ordered/grouped; **nested CV** when you tune *and* estimate performance (avoids optimistic bias).
- **Hyperparameter tuning:** grid (exhaustive), random (cheaper, often as good), or **Bayesian optimization** (efficient for big spaces) — always *inside* the CV/pipeline.
- **Track experiments** (params, metrics, artifacts — MLflow/W&B) so runs are reproducible and comparable.

> **The classic mistake:** tuning on the test set, or CV that ignores time/group structure → inflated scores that die in production.

---

## Stage 6 — Evaluation

**The intuition:** a single accuracy number is almost never the answer — evaluate against the **business goal**, and look *under* the aggregate.

- **Tie the metric to the cost of each error.** Choose precision/recall/F1/ROC-AUC/PR-AUC based on whether false positives or false negatives hurt more (see [Classification Metrics](06-classification-metrics.md)).
- **Slice, don't just aggregate.** Overall accuracy can hide catastrophic failure on a minority segment — always break metrics down by subgroup / protected attribute.
- **Error analysis** — look at *where and why* it's wrong (worst slices, systematic patterns); this drives the loop back to data/features.
- **Fairness** — measure disparities (demographic parity, equal opportunity, equalized odds); note these definitions can *conflict* — a model fair by one can be unfair by another.
- **Threshold selection is its own decision** — don't default to 0.5; pick the operating point from the business cost of each error.
- **The "good enough?" gate** — does it beat the baseline **and** meet the Stage-1 success criteria? If not → **loop back** to features, data, or even the problem framing.

---

## Stage 7 & 8 — Deployment & Monitoring

**The intuition:** shipping the model is the *start*, not the finish — a static model faces a moving world, so you must watch it and retrain.

- **Deployment** — operationalize the model *and its data pipeline* (batch / online / streaming), roll out safely (shadow → canary → A/B), and keep rollback ready.
- **Monitoring** — track health (latency/errors), data quality, **drift** (data vs concept), and model performance (hard, because ground-truth labels are delayed).
- **Retraining** — trigger on schedule and/or on drift/performance drop; validate against the current champion; redeploy. This closes the loop back to Stage 2.

> This is a whole discipline (MLOps) — covered in depth in **[ML in Production](08-ml-in-production-and-mlops.md)**: serving modes, feature stores, CI/CD/CT, drift detection, release strategies, governance, and failure modes.

---

## The Three Things That Kill Projects

Across every stage, three failure patterns recur — name these in an interview:

1. **Business KPI ≠ ML metric** (Stage 1 & 6) — optimizing accuracy while the business needle doesn't move.
2. **Data leakage** (Stages 2, 4, 5) — the unifying defense is *fit on train only, inside a pipeline, with a split that respects time and groups.*
3. **Treating deployment as the finish line** (Stage 7–8) — no monitoring → silent decay as the world drifts.

**The elevator answer to "how do you approach a DS problem?":**
> *"I frame the business problem and define success first — including whether ML is even warranted and how the ML metric ties to a business metric. Then I get and understand the data, set up leak-free splits that respect time and groups, do EDA, engineer features inside a pipeline fit only on training data, build a dumb baseline before anything complex, cross-validate and tune, and evaluate against the business goal with subgroup slicing and the right threshold. If it clears the bar I deploy it safely — shadow then canary then A/B — and monitor for drift so I can retrain. And I treat the whole thing as a loop, not a line."*

---

## Interview Questions

### Q: Walk me through the end-to-end lifecycle of a data science project.
**Strong answer:** Frame the business problem and success criteria (and check ML is warranted, tying the ML metric to a business KPI) → collect/understand data with leak-free, time/group-aware splits → EDA → clean and engineer features inside a pipeline fit only on train → baseline model, then cross-validate and tune → evaluate against the business goal with subgroup slicing and threshold selection → deploy safely (shadow/canary/A-B) → monitor for drift and retrain. It's a loop — I name where I'd loop back (e.g., evaluation revealing the framing was wrong).

### Q: What's the difference between a business metric and an ML metric, and why does it matter?
**Strong answer:** The business metric is the outcome the company cares about (revenue, churn, conversion); the ML metric is a model-quality proxy (F1, AUC, RMSE). They can diverge — a model can improve AUC while revenue is flat. The most common cause of "successful models that fail" is optimizing the ML metric without confirming it moves the business metric, so I connect them at framing and re-validate with an online A/B test.

### Q: How do you prevent data leakage?
**Strong answer:** Split before touching the data; fit all transformers (imputers, scalers, encoders, feature selectors) on the training fold only and apply to test — done cleanly with a scikit-learn Pipeline inside cross-validation. Respect structure: temporal splits for time-ordered data, grouped splits for correlated rows. And be alert to target leakage — features that encode the answer or aren't available at prediction time. The tell is validation scores that look too good and collapse in production.

### Q: Why build a baseline model?
**Strong answer:** It sets the bar any complex model must beat, verifies the end-to-end pipeline (data → model → metric) actually works, and reveals whether added complexity pays off. A model that barely beats "predict the majority class" is telling you the features or framing are the problem, not the algorithm.

### Q: Roughly how much time goes into each stage?
**Strong answer:** Data understanding and prep dominate — commonly cited as the largest share (surveys range 15–80%; ~45% is defensible). Modeling is a smaller slice than beginners expect, and deployment + monitoring is ongoing, not one-and-done. The honest framing: most of the work is data, and the model is the easy part.

---

## Glossary

| Term | Simple explanation | Purpose |
|---|---|---|
| **CRISP-DM** | The classic 6-phase data-mining process (business → data → prep → model → eval → deploy) | The canonical, tool-agnostic lifecycle reference |
| **TDSP** | Microsoft's Team Data Science Process (adds roles + a customer-acceptance gate) | Agile, team-oriented version of the lifecycle |
| **Problem framing** | Turning a business goal into a measurable ML task with a defined label | Prevents solving the wrong problem |
| **Business metric vs ML metric** | The outcome the business cares about vs a model-quality proxy | Their disconnect is the top cause of failed projects |
| **Proxy label** | A stand-in target when the true outcome can't be measured | Enables ML but introduces measurement error — no proxy is perfect |
| **Data leakage** | Information from outside the training data (future/target/test) sneaking into features | Causes inflated offline scores that collapse in production |
| **Temporal split** | Train on the past, test on the future | Simulates deployment for time-ordered data; prevents future leakage |
| **Grouped split** | Keep all rows of one entity (user/patient) on the same side of the split | Prevents the same entity leaking across train/test |
| **EDA** | Exploratory data analysis — summarize and visualize before modeling | Understand shape/gaps/relationships; generate hypotheses |
| **Feature engineering** | Creating/transforming inputs the model learns from | Often the biggest accuracy lever; where domain knowledge pays |
| **Pipeline** | A bundled sequence of preprocessing + model fit together | Ensures transformers fit on train-only (leak-free) and reproducibly |
| **Baseline model** | The simplest possible model (majority class / mean / logistic) | Sets the bar and verifies the end-to-end pipeline works |
| **Cross-validation** | Rotating train/validation folds to estimate performance | Robust performance estimate; tune without overfitting the test set |
| **Nested CV** | Outer loop for evaluation, inner loop for tuning | Unbiased performance estimate when you also tune hyperparameters |
| **Error analysis** | Studying where/why the model is wrong, not just the aggregate | Drives targeted fixes and the loop back to data/features |
| **Model drift / decay** | Performance dropping over time as the world changes | Why deployment isn't the finish line; triggers retraining |

---

## References

- [CRISP-DM overview](https://www.datascience-pm.com/crisp-dm-2/) · [CRISP-ML(Q) (arXiv 2003.05155)](https://arxiv.org/pdf/2003.05155)
- [Microsoft TDSP lifecycle](https://learn.microsoft.com/en-us/azure/architecture/data-science-process/lifecycle) · [Google — ML project phases](https://developers.google.com/machine-learning/managing-ml-projects/phases) · [Google — ML problem framing](https://developers.google.com/machine-learning/problem-framing/ml-framing)
- [scikit-learn — Common pitfalls & data leakage](https://scikit-learn.org/stable/common_pitfalls.html) · [Data preparation without leakage](https://machinelearningmastery.com/data-preparation-without-data-leakage/)
- [Chip Huyen — Designing ML Systems (iterative process)](https://huyenchip.com/books/) · [Google — Rules of ML](https://developers.google.com/machine-learning/guides/rules-of-ml)

---

*Previous: [Classification Metrics](06-classification-metrics.md) | Next: [ML in Production & MLOps](08-ml-in-production-and-mlops.md) | Up: [Guide Home](../README.md)*
