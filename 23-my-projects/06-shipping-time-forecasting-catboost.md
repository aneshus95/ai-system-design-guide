# Order Shipping-Time Forecasting (EDA + CatBoost)

> **My project.** Forecasted order shipping time with **EDA + CatBoost regression**, reaching **70%+ accuracy within a ±3-day tolerance** — giving customers accurate delivery expectations instead of a static carrier estimate.

## Table of Contents

- [The Narrative](#the-narrative)
- [What I Built — Methodology](#what-i-built--methodology)
- [Deep Dive 1 — EDA for a Forecasting Problem](#deep-dive-1--eda-for-a-forecasting-problem)
- [Deep Dive 2 — CatBoost (and Why It Fits)](#deep-dive-2--catboost-and-why-it-fits)
- [Deep Dive 3 — The "±3-Day Accuracy" Metric](#deep-dive-3--the-3-day-accuracy-metric)
- [Deep Dive 4 — Validating a Time-Based Forecast (No Leakage)](#deep-dive-4--validating-a-time-based-forecast-no-leakage)
- [Interview Q&A](#interview-qa)
- [Honest Caveats](#honest-caveats)
- [References](#references)

---

## The Narrative

**Situation.** Customers saw a carrier's static estimate — "dispatch date + average transit time" — which ignores current network conditions, route-specific history, carrier/depot performance, and seasonality. Vague or wrong delivery dates drive cart abandonment, "where is my order?" support tickets, and lost repeat business.

**Task.** Predict the **actual** shipping/delivery time per order from historical data, accurately enough to set a trustworthy customer-facing promise.

**Action.** I ran thorough **EDA** to understand the (right-skewed) delivery-time distribution, outliers, and the features that actually move it (distance/zone, carrier, weekday, order size, seasonality), engineered those features, and trained a **CatBoost regressor** — chosen for its native categorical handling and strong, low-tuning defaults on tabular data. I evaluated with a business-facing **±3-day tolerance accuracy** alongside MAE/RMSE.

**Result.** **70%+ of orders predicted within ±3 days** of the true delivery time — a materially better, customer-communicable promise than the carrier's static ETA.

---

## What I Built — Methodology

```
 historical orders
      │
      ▼
 ┌───────────────┐   distributions · outliers · missing · correlations ·
 │      EDA      │   seasonality · feature↔target relationships
 └──────┬────────┘
        ▼
 ┌───────────────┐   distance/zone · weekday/month · carrier/depot ·
 │  FEATURE ENG  │   order size/weight · service level · holiday flags
 └──────┬────────┘
        ▼
 ┌───────────────┐   native categorical handling (ordered target stats),
 │   CatBoost    │   ordered boosting, symmetric trees
 │   regressor   │
 └──────┬────────┘
        ▼
 evaluate: ±3-day accuracy (business KPI)  +  MAE / RMSE (engineering)
        │  time-based (walk-forward) validation → no leakage
        ▼
 customer-facing delivery estimate
```

---

## Deep Dive 1 — EDA for a Forecasting Problem

EDA is where the modeling decisions are actually made. For shipping-time, the steps that mattered:

- **Target distribution** — delivery time is typically **right-skewed** (a long tail of slow deliveries). Check mean/median, spread, skewness → this motivates a possible **log-transform** of the target and a preference for outlier-robust metrics.
- **Outliers** — box-plots + rules (1.5×IQR, z-score, 3×std). A parcel stuck in customs for 60 days distorts regression relationships; decide cap vs separate handling.
- **Missing data** — quantify per column; impute (median/tree-based) or drop.
- **Correlation / relationships** — correlation matrix + grouped box-plots of shipping time by carrier, zone, weekday, order size to find real predictors and spot multicollinearity.
- **Temporal patterns / seasonality** — peak-season/holiday slowdowns, weekday-vs-weekend dispatch. Critical: it dictates a **time-based validation** strategy (Deep Dive 4).
- **Feature engineering** — origin→destination **distance/zone**, **weekday/month/holiday flags**, **carrier/depot**, **order size/weight/item count**, **service level**.

Sources: [What is EDA — GeeksforGeeks](https://www.geeksforgeeks.org/data-analysis/what-is-exploratory-data-analysis/) · [Predictive parcel delivery — ShippyPro](https://www.shippypro.com/blog/en/predictive-parcel-delivery-data-driven-delivery-dates-ecommerce)

---

## Deep Dive 2 — CatBoost (and Why It Fits)

CatBoost ("Categorical Boosting") is **gradient boosting on decision trees** — an ensemble where each new tree fits the residual errors of the current ensemble. Its two signature innovations solve a subtle leakage problem, and both are exactly why it suited this data.

**1. Ordered boosting (fights prediction shift / target leakage).** In standard GBDT, the gradient for each example is estimated by a model that was *itself trained on that example* — a subtle target leakage that biases residuals and causes **prediction shift** (train/test distribution mismatch). Ordered boosting fixes it with a permutation scheme: each example's residual is computed by a model trained **only on the examples before it** in a random permutation, so a point never influences its own gradient.

**2. Native categorical handling via ordered target statistics (vs one-hot).** High-cardinality categoricals (carrier, depot, destination zone) would explode under one-hot. CatBoost replaces a category with a **target statistic** (roughly its target mean) — but computed using **only the rows before the current one** in a permutation, with a smoothing prior for rare categories, so no row sees its own target. Low-cardinality features fall back to one-hot below `one_hot_max_size`, and it can auto-generate feature crosses.

**3. Symmetric (oblivious) trees.** Every node at a given depth splits on the **same feature + threshold**. This acts as strong built-in **regularization** (resists overfitting with little tuning) and enables **very fast, branch-free inference** — useful for serving ETAs at low latency.

**Why CatBoost here vs XGBoost/LightGBM:** GBDTs are state-of-the-art on heterogeneous tabular data. CatBoost's edge is **best-in-class categorical handling with no manual encoding** (and no leakage), plus strong out-of-the-box defaults. LightGBM often trains fastest on huge data; XGBoost is mature but historically needed manual categorical encoding.

Sources: [CatBoost: unbiased boosting with categorical features (arXiv 1706.09516)](https://arxiv.org/abs/1706.09516) · [Why CatBoost grows symmetric trees — Manokhin](https://valeman.substack.com/p/why-catboost-grows-symmetric-trees) · [CatBoost categorical encoding — GeeksforGeeks](https://www.geeksforgeeks.org/machine-learning/catboosts-categorical-encoding-one-hot-vs-target-encoding/)

---

## Deep Dive 3 — The "±3-Day Accuracy" Metric

**What it is:** the fraction of predictions where **|predicted − actual| ≤ 3 days**. So "70%+ within ±3 days" = at least 70% of orders landed within 3 days of the true delivery time. It converts a regression problem into a **tolerance/threshold accuracy** — it's a *business KPI*, not a native regression metric.

**Why use it:** customers experience delivery as "did it arrive close to what was promised?" — a hit/miss within a window that maps directly to an SLA/promise, and it's trivial to communicate ("7 in 10 orders within 3 days of our estimate").

**Its blind spots (say this before you're asked):**
- **Insensitive to magnitude past the cutoff** — a 4-day miss and a 40-day miss both count as "wrong," hiding the tail.
- **Insensitive within the window** — 0 days off and 2.9 days off both count as "right."

So I paired it with the real regression metrics:

| Metric | What it tells you | Note |
|---|---|---|
| **±3-day accuracy** | Customer-facing promise hit-rate | Business KPI; blind to magnitude |
| **MAE** | Typical error in days | **Robust to outliers** (linear penalty) |
| **RMSE** | Surfaces costly large misses | Penalizes big errors more |
| **R²** | Variance explained | Overall fit |

Sources: [Regression metrics — GeeksforGeeks](https://www.geeksforgeeks.org/machine-learning/regression-metrics/) · [Which regression metric to use — Karabulut](https://medium.com/@SelinKarabulut/which-regression-model-performance-metrics-to-use-r%C2%B2-rmse-mae-mape-9433ec0d0df4)

---

## Deep Dive 4 — Validating a Time-Based Forecast (No Leakage)

The single biggest correctness risk in a delivery forecast is **temporal leakage**.

**Never random-shuffle CV.** A random split puts *future* orders in train and *past* orders in test, leaking future information and giving optimistically inflated scores. Instead use a **chronological / walk-forward (forward-chaining) split** — train on the earlier period, validate on the later, expanding the window (e.g., scikit-learn `TimeSeriesSplit`).

```
 time ─────────────────────────────────────────►
 [ train ][ val ]
 [ train      ][ val ]
 [ train             ][ val ]     ← expanding window; val always AFTER train
```

Also ensure **every engineered feature is computable at prediction time** (nothing derived from post-delivery info). A red flag for hidden leakage: suspiciously stable CV scores across folds.

Sources: [Cross-validation with time series — CodeCut](https://codecut.ai/cross-validation-with-time-series/)

---

## Interview Q&A

**Q: Why CatBoost over XGBoost?**
Two reasons that fit this data: (1) **native categorical handling** — carrier/depot/zone are high-cardinality, and CatBoost encodes them with **ordered target statistics** instead of hand-built one-hot/target encoders, which also **avoids the leakage** naive target encoding causes; (2) **ordered boosting + symmetric trees** give well-regularized results with minimal tuning and fast, low-latency inference. XGBoost historically needs manual categorical encoding and more tuning to match it.

**Q: How do you handle categorical features?**
Pass them to CatBoost as categorical columns. Internally each row's category is encoded from the target mean of only the rows *before it* in a random permutation, with a smoothing prior for rare categories — so a row never sees its own target. Low-cardinality features fall back to one-hot.

**Q: How did you validate to avoid leakage?**
No random shuffle — a chronological walk-forward split (`TimeSeriesSplit`), training on earlier orders and validating on later ones, and I confirmed every feature is available at prediction time. Random CV would leak future info and inflate scores.

**Q: Why ±3-day accuracy, not just MAE?**
It's the business KPI — it maps to the customer promise and is easy to communicate. But it's blind to error magnitude past the threshold, so I report it alongside MAE (typical error) and RMSE (surfaces costly large misses).

**Q: Delivery times are skewed — how did you handle that?**
Confirmed the right skew in EDA (histogram/skewness). Options I used/considered: log-transform the target so the model optimizes a more symmetric scale (then exponentiate back), cap/treat extreme outliers (customs holds, lost parcels) separately, and prefer MAE (outlier-robust) over squared-error framing when the tail is heavy.

---

## Honest Caveats

- **"±N-day accuracy" is a business-metric framing** (thresholded absolute error), not a textbook-standard metric — defend it as a KPI paired with MAE/RMSE.
- **Load-bearing CatBoost claims** (ordered boosting, ordered target statistics, symmetric trees) are directly from the paper; specific cross-library benchmark ranks are source-dependent — cite as illustrative.
- Vendor ETA-accuracy figures (from delivery SaaS blogs) are marketing — don't present as guaranteed.

---

## References

- [CatBoost: unbiased boosting with categorical features (arXiv 1706.09516)](https://arxiv.org/abs/1706.09516)
- [CatBoost categorical encoding — GeeksforGeeks](https://www.geeksforgeeks.org/machine-learning/catboosts-categorical-encoding-one-hot-vs-target-encoding/) · [Why symmetric trees — Manokhin](https://valeman.substack.com/p/why-catboost-grows-symmetric-trees)
- [Regression metrics — GeeksforGeeks](https://www.geeksforgeeks.org/machine-learning/regression-metrics/)
- [Cross-validation with time series — CodeCut](https://codecut.ai/cross-validation-with-time-series/)
- [Predictive parcel delivery — ShippyPro](https://www.shippypro.com/blog/en/predictive-parcel-delivery-data-driven-delivery-dates-ecommerce)

---

## Glossary

| Term | Simple explanation | Purpose |
|---|---|---|
| **EDA (Exploratory Data Analysis)** | Systematic inspection of a dataset through summary statistics, distributions, and visualizations before modeling | Drives feature engineering and modeling decisions with evidence rather than guesses |
| **CatBoost** | An open-source gradient-boosted decision tree library from Yandex with native categorical handling and ordered boosting | State-of-the-art on heterogeneous tabular data; removes need for manual categorical encoding |
| **Gradient Boosting** | An ensemble method that trains trees sequentially, each fitting the residual errors of the previous ensemble | Achieves high accuracy on tabular data by combining many weak learners |
| **Ordered Boosting** | CatBoost's variant of gradient boosting where each sample's gradient is computed using only earlier samples in a random permutation | Eliminates prediction shift — a subtle target leakage present in standard gradient boosting |
| **Ordered Target Statistics** | CatBoost's method of encoding a categorical feature as the target mean of only the rows preceding it in a random permutation, with a smoothing prior | Encodes high-cardinality categoricals without leaking the current row's target |
| **Symmetric (Oblivious) Trees** | Decision trees where every node at the same depth uses the same feature and threshold for splitting | Acts as built-in regularization; enables fast branch-free inference |
| **Right-Skewed Distribution** | A distribution with a long tail on the right side — the mean is higher than the median | Indicates outliers (e.g., lost parcels) that can distort regression; motivates log-transform |
| **Log-Transform** | Applying the natural logarithm to the target variable before training | Compresses the right tail so the model optimizes a more symmetric scale |
| **Outlier** | A data point far from the bulk of the distribution | Can distort regression relationships; must be capped, removed, or modeled separately |
| **Temporal Leakage** | Using future data during training because of an improper random split | Inflates evaluation scores; predictions in production will be worse than measured |
| **Walk-Forward (Chronological) Cross-Validation** | A CV strategy where training always precedes validation in time, with an expanding window | Simulates real deployment where future orders must be predicted from past data only |
| **TimeSeriesSplit** | Scikit-learn's implementation of walk-forward cross-validation | Prevents temporal leakage by ensuring validation always follows training chronologically |
| **±3-Day Accuracy** | The fraction of predictions where the absolute error is at most 3 days | A business-facing KPI that maps directly to the customer delivery promise |
| **MAE (Mean Absolute Error)** | Average of absolute prediction errors; each error counts equally | Robust to extreme outliers; reports typical error in the original units (days) |
| **RMSE (Root Mean Squared Error)** | Square root of the average squared error; large errors are penalized more | Surfaces costly large misses (lost parcels, extreme delays) more visibly than MAE |
| **R² (Coefficient of Determination)** | Proportion of target variance explained by the model (1 = perfect, 0 = mean-only baseline) | Measures overall model fit on the held-out set |
| **Feature Engineering** | Creating new input columns (distance/zone, weekday, holiday flags, etc.) from raw data | Provides the model with domain knowledge it cannot learn from raw timestamps alone |
| **High-Cardinality Categorical** | A categorical feature with many possible values (e.g., carrier codes, postal zones) | One-hot encoding would explode dimensionality; CatBoost handles these natively |
| **Multicollinearity** | When two or more features are highly correlated with each other | Can destabilize coefficient-based models; identified during EDA via correlation matrix |
| **IQR (Interquartile Range)** | The range between the 25th and 75th percentiles of a distribution | Used in box-plot outlier rules (e.g., flag values beyond 1.5 × IQR from the quartiles) |

---

*Previous: [Graph RAG over BIAN](05-graph-rag-over-bian.md) | Next: [PO Extraction + BERT Classifier](07-po-extraction-and-bert-classifier.md) | Up: [Guide Home](../README.md)*
