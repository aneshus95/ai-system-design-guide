# ML System Design (Applied ML)

ML system design interviews ask you to design a complete machine-learning solution end-to-end — from raw data to production monitoring — for a realistic business problem. There is no single right answer; you are graded on a **structured, trade-off-driven approach** that shows you can build ML systems that actually work in production. This chapter covers non-LLM "classical" ML scenarios and complements the LLM/RAG system-design content elsewhere in this guide.

## Table of Contents

- [The 6-Step Framework](#the-6-step-framework)
- [Recommendation System](#recommendation-system)
- [News / Feed Ranking](#news--feed-ranking)
- [Fraud / Anomaly Detection](#fraud--anomaly-detection)
- [Ad CTR Prediction](#ad-ctr-prediction)
- [Customer Churn Prediction](#customer-churn-prediction)
- [Demand / Sales Forecasting](#demand--sales-forecasting)
- [Search Ranking & Relevance (LTR)](#search-ranking--relevance-ltr)
- [ETA Prediction](#eta-prediction)
- [Credit Scoring](#credit-scoring)
- [Online vs Batch Serving, Feature Stores, Drift](#online-vs-batch-serving-feature-stores-and-drift)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## The 6-Step Framework

Use this as your skeleton for **any** ML system design question. Spend roughly equal time on each step; the details change per scenario, but the skeleton stays constant.

| Step | What to cover | Time (45-min interview) |
|------|---------------|------------------------|
| 1. Clarify & Frame | Business goal → ML objective, constraints, scale, latency SLA | ~5 min |
| 2. Data | Sources, labeling, features, pipeline, leakage risks | ~8 min |
| 3. Model | Baseline first, then candidates, justify each choice | ~8 min |
| 4. Evaluation | Offline metrics + online A/B, proxy vs. true metric gap | ~7 min |
| 5. Deployment | Serving mode, scaling, monitoring, rollback | ~7 min |
| 6. Iteration | Feedback loops, retraining cadence, drift handling | ~5 min |

### ASCII Framework Flow

```
┌───────────────────────────────────────────────────────────────┐
│  1. CLARIFY & FRAME                                           │
│     Business goal → ML objective → constraints (latency/cost) │
└──────────────────────────┬────────────────────────────────────┘
                           │
                           ▼
┌───────────────────────────────────────────────────────────────┐
│  2. DATA                                                      │
│     Sources → Labels → Features → Pipeline → Leakage check   │
└──────────────────────────┬────────────────────────────────────┘
                           │
                           ▼
┌───────────────────────────────────────────────────────────────┐
│  3. MODEL                                                     │
│     Baseline (heuristic/LR) → Candidates → Justify choice    │
└──────────────────────────┬────────────────────────────────────┘
                           │
                           ▼
┌───────────────────────────────────────────────────────────────┐
│  4. EVALUATION                                                │
│     Offline metrics (AUC, NDCG, MAE) + Online A/B test       │
└──────────────────────────┬────────────────────────────────────┘
                           │
                           ▼
┌───────────────────────────────────────────────────────────────┐
│  5. DEPLOYMENT                                                │
│     Serving (batch/online) → Scale → Canary → Monitoring     │
└──────────────────────────┬────────────────────────────────────┘
                           │
                           ▼
┌───────────────────────────────────────────────────────────────┐
│  6. ITERATION                                                 │
│     Feedback loops → Retraining → Drift detection → v2       │
└───────────────────────────────────────────────────────────────┘
```

### Step 1 — Clarify & Frame

Before touching model architecture, nail down:

- **Business goal**: "Increase 7-day retention by X%", "reduce fraud losses by $Y"
- **ML objective**: What proxy label maps to that goal? (click ≠ satisfaction)
- **Constraints**: Latency SLA (real-time = <150 ms, batch = minutes/hours), throughput (QPS), budget
- **Scale**: How many users, items, events per day?
- **Success definition**: How will you know v1 is good enough to ship?

### Step 2 — Data

- **Sources**: User logs, item metadata, third-party enrichment, labels (explicit vs. implicit)
- **Labeling**: Positive labels (click, purchase, conversion); negative sampling strategy matters
- **Features**: User features, item features, context features, interaction features
- **Pipeline**: Ingestion → cleaning → transformation → feature store → training dataset
- **Leakage check**: Would any feature contain future information at training time?

### Step 3 — Model

Always propose a **baseline** (rule-based or logistic regression) before complex models. Then justify upgrades:

| Model class | When to reach for it |
|-------------|----------------------|
| Logistic regression | High sparsity (CTR), interpretability needed, very low latency |
| Gradient boosted trees (XGBoost/LightGBM) | Tabular data, feature interactions, churn/fraud/forecasting |
| Two-tower neural network | Large-scale retrieval, embedding-based similarity |
| Factorization machines / DeepFM | CTR with sparse categorical features |
| GBDT + Neural re-ranker | Two-stage ranking (retrieve then rank) |
| Time-series models (ARIMA, LightGBM, DeepAR) | Demand forecasting |

### Step 4 — Evaluation

**Offline metrics** (measured on held-out test set, split by time, not random):

| Task type | Primary offline metric | Secondary |
|-----------|----------------------|-----------|
| Binary classification | AUC-ROC, PR-AUC | F1, calibration |
| Ranking | NDCG@k, MAP, MRR | Precision@k |
| Regression / forecasting | MAE, RMSE, MAPE | Quantile loss |
| Anomaly detection | Precision@k, recall at fixed FPR | F1 |

**Online metrics**: Always A/B test before full rollout. Track the true business KPI (revenue, retention) not just the proxy metric.

### Step 5 — Deployment

- **Serving mode**: Batch (precompute, store in cache) vs. online (real-time inference)
- **Latency budget**: Feature fetch (1–5 ms) + model inference (10–30 ms) + network (~15 ms) = headroom to ~150 ms
- **Scaling**: Containerized model servers (Triton, TorchServe), autoscaling, model quantization
- **Canary rollout**: 1% → 5% → 20% → 100%; automatic rollback on metric degradation
- **Monitoring**: Prediction distribution, input feature drift, model accuracy vs. ground truth

### Step 6 — Iteration

- **Feedback loops**: Log predictions + outcomes; use to build next training set
- **Retraining cadence**: Daily/weekly for most; streaming/near-real-time for fraud
- **Drift handling**: Data drift (input distribution shift), concept drift (relationship between features and label shifts)
- **Bias and fairness**: Monitor performance disaggregated by user segments

---

## Recommendation System

**Prompt**: "Design a recommendation system for a video/music streaming platform."

**Plain English**: You can't score all N million items for every user on every page load — too slow. So you do it in two stages: first *retrieve* a small candidate set (~1000), then *rank* just those carefully.

### Architecture (Retrieve → Rank)

```
All Items (10M+)
      │
      ▼
┌─────────────────────────┐
│  CANDIDATE RETRIEVAL    │  ← collaborative filtering, two-tower embedding ANN
│  ~1000 candidates       │    (recall-focused, fast)
└──────────┬──────────────┘
           │
           ▼
┌─────────────────────────┐
│  RANKING MODEL          │  ← GBDT or DeepFM, rich features
│  Top-K scored items     │    (precision-focused)
└──────────┬──────────────┘
           │
           ▼
┌─────────────────────────┐
│  BUSINESS RULES         │  ← diversity, freshness, content policy
│  Final ranked list      │
└─────────────────────────┘
```

**Data**: Implicit feedback (plays, likes, skips, watch time) > explicit ratings. Cold start for new users: use popularity or onboarding survey.

**Features**: User embedding, item embedding, user-item interaction (recency, frequency), context (time of day, device).

**Model**: Two-tower retrieval network for candidate generation; GBDT or DeepFM for ranking. Start with matrix factorization as baseline.

**Offline metrics**: Recall@k for retrieval; NDCG@k, Precision@k for ranking. **Online**: stream time, click-through rate, 7-day retention.

**Serving**: Precompute user/item embeddings nightly (batch); ANN index (FAISS/ScaNN) for retrieval; ranking model called online for top-1000 candidates; cache final ranked list in Redis (~50 ms end-to-end).

**Iteration**: Log impressions and plays; retrain embeddings weekly; near-real-time feature updates for session context.

---

## News / Feed Ranking

**Prompt**: "Design a ranking system for a social media feed."

**Plain English**: Users don't see every post from everyone they follow — there are too many. The system predicts which posts a user would most want to see *right now*.

```
User's Network Posts (thousands)
         │
         ▼
  ┌──────────────────┐
  │ CANDIDATE POOL   │  ~500 posts from follows + sponsored + trending
  └──────┬───────────┘
         │
         ▼
  ┌──────────────────┐
  │  RANKER          │  predict P(like), P(comment), P(share), P(time-spent)
  └──────┬───────────┘
         │
         ▼
  ┌──────────────────┐
  │ DIVERSITY FILTER │  dedupe, content policy, ad injection
  └──────────────────┘
```

**Data**: Engagement signals (likes, comments, shares, time spent, hides). Label: weighted engagement score. Handle position bias (posts shown higher get more clicks regardless of quality) via inverse propensity scoring.

**Model**: GBDT on user/post/author/context features; optionally a neural re-ranker. Key features: author affinity score, post age decay, predicted engagement per content type.

**Offline metrics**: NDCG weighted by engagement type. **Online**: per-session engagement rate, unfollow rate, DAU.

**Serving**: ~100 ms budget. Features from online store (Redis) + model inference.

**Iteration**: Feedback loop is implicit (engagement). Watch for filter bubble / echo chamber effects; add diversity term to ranking objective.

---

## Fraud / Anomaly Detection

**Prompt**: "Design a real-time fraud detection system for a payment platform."

**Plain English**: Fraud is rare (<0.5% of transactions) and constantly evolving. You need a hybrid: rules catch known patterns instantly, ML catches unknown patterns, and humans review edge cases.

```
Transaction Event
      │
      ▼
┌─────────────────────────────────────────┐
│  FEATURE EXTRACTION (real-time + batch) │
│  velocity features (tx/hour), geo,      │
│  merchant category, device fingerprint  │
└──────────────┬──────────────────────────┘
               │
               ▼
  ┌────────────────────────┐
  │  RULE ENGINE           │  known fraud patterns → instant block
  └──────────┬─────────────┘
             │ (unknown patterns)
             ▼
  ┌────────────────────────┐
  │  ML RISK SCORER        │  outputs P(fraud); threshold → approve/review/decline
  └──────────┬─────────────┘
             │ (borderline)
             ▼
  ┌────────────────────────┐
  │  HUMAN REVIEW QUEUE    │  analysts label → feeds retraining
  └────────────────────────┘
```

**Data**: Heavily imbalanced (fraud ~0.1–0.5%); use oversampling (SMOTE) or cost-sensitive learning. Label delay is real — fraud confirmed days later. Split by time for evaluation.

**Features**: Velocity (count of transactions in last 1h/24h/7d), geo velocity, device fingerprint match, merchant category mismatch, time since last transaction.

**Model**: GBDT (XGBoost/LightGBM) is the industry standard for structured transaction data. Optionally add GNN for social graph / account linkage signals.

**Offline metrics**: Precision@k, Recall at fixed FPR, PR-AUC (use PR over ROC for imbalanced). **Online**: fraud loss rate, false positive rate (friction on legit users), chargeback rate.

**Serving**: Must be <200 ms. Precompute user-level aggregate features in streaming pipeline (Flink/Kafka Streams); merge with real-time transaction features at inference.

**Iteration**: Adversarial — fraudsters adapt. Monitor for new fraud patterns. Retrain frequently (daily or triggered). Champion/challenger model deployment.

---

## Ad CTR Prediction

**Prompt**: "Design a click-through rate prediction system for a search/social ad platform."

**Plain English**: For each ad auction, you need to estimate P(user clicks this ad) within ~150 ms across billions of daily requests. The model output feeds directly into the auction — a 0.1% AUC improvement can mean millions in revenue.

```
Ad Request (user, query, context)
         │
         ▼
  ┌──────────────────┐
  │  AD RETRIEVAL    │  candidate ads from index (targeting match)
  └──────┬───────────┘
         │
         ▼
  ┌──────────────────┐
  │  CTR MODEL       │  P(click | user, ad, context)
  └──────┬───────────┘
         │
         ▼
  ┌──────────────────┐
  │  AUCTION ENGINE  │  rank by eCPM = P(click) × bid; apply floor price
  └──────────────────┘
```

**Data**: 1–2% CTR means heavy class imbalance; negative downsample (keep 10–20% of negatives), but correct probabilities with calibration. Train/val split must be by time. Logging bias: only shown ads get labels.

**Features**: User features (demographics, interest embedding, historical CTR), ad features (ad embedding, creative type, historical CTR), context features (query, device, time), user×ad cross features.

**Model**: Logistic regression on hashed features is the proven baseline (sparse, fast). Upgrade to Factorization Machines or DeepFM for feature interaction modelling. **Calibration is critical**: if model predicts 5% CTR, real CTR should be ~5% — use Platt scaling or isotonic regression.

**Offline metrics**: AUC, log-loss, calibration error. **Online**: CTR, RPM (revenue per thousand impressions), advertiser ROI.

**Serving**: Sub-100 ms requirement. Use model quantization; keep feature lookup (Redis) and inference in same DC. Serve with ONNX Runtime or TensorRT.

**Iteration**: Retrain daily on recent data (recent click patterns matter more than older ones). Explore/exploit via epsilon-greedy or UCB for new ads with no history.

---

## Customer Churn Prediction

**Prompt**: "Design an ML system to predict which customers will cancel their subscription in the next 30 days."

**Plain English**: Churn prediction is binary classification over a prediction window. You score users periodically and trigger retention campaigns for high-risk users. The business impact is measured by incremental retention (users who *would have* churned but didn't, due to intervention).

```
Raw Events (logins, feature usage, support tickets, billing)
         │
         ▼  (batch ETL, nightly)
  ┌─────────────────────────┐
  │  FEATURE ENGINEERING    │
  │  RFM, engagement trend, │
  │  tenure, support count  │
  └──────────┬──────────────┘
             │
             ▼
  ┌─────────────────────────┐
  │  CHURN SCORER (GBDT)    │  output: P(churn in 30d)
  └──────────┬──────────────┘
             │
             ▼
  ┌─────────────────────────┐
  │  INTERVENTION ENGINE    │  email / discount / CSM outreach per tier
  └─────────────────────────┘
```

**Data**: Label = cancelled within next 30 days. Observation window = past 90 days of activity. Hold out most recent cohort for test.

**Features**: Recency (days since last login), Frequency (sessions per week, trend), Monetary (subscription tier), support tickets filed, feature adoption breadth, contract length, NPS score if available.

**Model**: GBDT (XGBoost/LightGBM) — tabular data, interpretable via SHAP, handles mixed types. Logistic regression as baseline.

**Offline metrics**: PR-AUC, F1 at business-chosen threshold. **Online**: intervention acceptance rate, 60-day retention lift vs. control group (A/B test).

**Serving**: Batch scoring is fine (run nightly, write scores to CRM). No real-time latency requirement.

**Iteration**: Causal inference matters — use uplift modelling (treatment effect) so you only reach out to customers who *respond* to interventions, not those who would have stayed anyway.

---

## Demand / Sales Forecasting

**Prompt**: "Design a demand forecasting system for a retail chain predicting daily sales per SKU per store."

**Plain English**: You have thousands (or millions) of time series. You need to predict horizon T days ahead with uncertainty estimates so inventory teams can order the right stock.

```
Historical Sales (per SKU per store)
+ External Signals (weather, holidays, promotions, econ)
         │
         ▼
  ┌─────────────────────────────────┐
  │  FEATURE ENGINEERING            │
  │  lag features (t-7, t-14, t-28) │
  │  rolling stats, seasonality     │
  │  one-hot holidays, promo flags  │
  └──────────────┬──────────────────┘
                 │
                 ▼
  ┌─────────────────────────────────┐
  │  GLOBAL MODEL (LightGBM/DeepAR) │  one model fits all SKUs
  └──────────────┬──────────────────┘
                 │
                 ▼
  ┌─────────────────────────────────┐
  │  FORECAST OUTPUT                │
  │  point forecast + P10/P90 CI    │
  └─────────────────────────────────┘
```

**Data**: Historical daily sales per (store, SKU). External: promotions calendar, holidays, weather. Handle stockouts (zero sales ≠ zero demand).

**Model**: LightGBM on engineered lag/rolling features scales to millions of series efficiently. Deep alternatives: Amazon DeepAR (global RNN) or Temporal Fusion Transformer (TFT) for multivariate signals. Start with seasonal naïve baseline.

**Offline metrics**: MAE, RMSE, MAPE, Weighted Absolute Percentage Error (WAPE). Use **backtest** (rolling-origin cross-validation) — never random splits for time series.

**Serving**: Batch (daily or weekly). Write forecasts to a data warehouse or demand-planning tool.

**Iteration**: Track forecast vs. actual. Retrain monthly or when MAPE degrades. Add new signals (pricing, competitor data) incrementally.

---

## Search Ranking & Relevance (LTR)

**Prompt**: "Design a search ranking system for an e-commerce product catalog."

**Plain English**: When a user types "running shoes size 10", you need to show the most relevant, purchasable, high-quality results first. This is a two-stage retrieve → rank problem, where ranking is framed as learning-to-rank (LTR).

```
Query: "running shoes size 10"
         │
         ▼
  ┌──────────────────────────────┐
  │  QUERY UNDERSTANDING         │
  │  spell correct, parse intent │
  │  entity extraction           │
  └──────────────┬───────────────┘
                 │
                 ▼
  ┌──────────────────────────────┐
  │  CANDIDATE RETRIEVAL (~1000) │
  │  BM25 lexical match +        │
  │  Dense vector (ANN) semantic │
  └──────────────┬───────────────┘
                 │
                 ▼
  ┌──────────────────────────────┐
  │  LTR RANKER                  │
  │  GBDT or neural (LambdaMART) │
  │  features: BM25 score,       │
  │  CTR, conversion, freshness  │
  └──────────────┬───────────────┘
                 │
                 ▼
  ┌──────────────────────────────┐
  │  BUSINESS RULES              │
  │  out-of-stock demotion,      │
  │  ad injection, diversity     │
  └──────────────────────────────┘
```

**Data**: Click logs with position. **Position bias**: higher positions get more clicks. Correct via click models or inverse propensity weighting. Relevance judgements (human labels) for evaluation.

**Model**: LambdaMART (GBDT-based LTR) is the industry standard. Features: query-doc relevance score (BM25, embedding cosine), product popularity, historical CTR, conversion rate, recency.

**Offline metrics**: NDCG@10, MRR. **Online**: CTR, add-to-cart rate, revenue per search.

**Serving**: ~100 ms budget. Retrieval from inverted index (Elasticsearch) + ANN index (FAISS) in parallel; rank with LTR model.

**Iteration**: Continuously log search sessions; retrain LTR weekly. Watch for query distribution shift (new trending queries the model hasn't seen).

---

## ETA Prediction

**Prompt**: "Design an ETA (estimated time of arrival) prediction system for a ride-sharing or food delivery platform."

**Plain English**: ETA must be fast, accurate, and account for real-time conditions. Overestimating frustrates users; underestimating causes late deliveries and bad reviews.

```
Origin + Destination
         │
         ▼
  ┌────────────────────────────┐
  │  ROUTE GENERATION          │  top-K candidate routes (graph search)
  └──────────────┬─────────────┘
                 │
                 ▼
  ┌────────────────────────────┐
  │  BASE ETA ESTIMATION       │  distance / historical speed per segment
  └──────────────┬─────────────┘
                 │
                 ▼
  ┌────────────────────────────┐
  │  ML ADJUSTMENT LAYER       │
  │  features: real-time       │
  │  traffic, weather, time,   │
  │  driver behavior history   │
  └──────────────┬─────────────┘
                 │
                 ▼
             ETA + CI
```

**Data**: Historical trip logs with actual arrival times. Real-time traffic from GPS pings, weather API. Segment-level speed data.

**Model**: GBDT on engineered route features (distance, segment speeds, turn counts, time of day). Deep alternative: graph neural network on road segments. Start with historical median speed baseline.

**Offline metrics**: MAE, RMSE on held-out trips; P90 error (tail accuracy matters for SLA). **Online**: user rating correlation with ETA accuracy, cancellation rate.

**Serving**: Real-time, <500 ms. Cache segment-level speed features; update every few minutes from GPS stream.

**Iteration**: Retrain daily. Special handling for weather events, local events (stadium games). Monitor by city, time-of-day, distance bucket.

---

## Credit Scoring

**Prompt**: "Design an ML system for credit risk scoring at a lending company."

**Plain English**: You're predicting probability of default (PD) over a 12-month window. This is a high-stakes, regulated domain — interpretability, fairness, and calibration are non-negotiable alongside accuracy.

```
Applicant Data
(bureau, bank statements, application form)
         │
         ▼
  ┌──────────────────────────────┐
  │  FEATURE ENGINEERING         │
  │  repayment history, DTI,     │
  │  credit utilization, tenure  │
  └──────────────┬───────────────┘
                 │
                 ▼
  ┌──────────────────────────────┐
  │  RISK MODEL (LR or GBDT)     │
  │  output: P(default in 12mo)  │
  └──────────────┬───────────────┘
                 │
                 ▼
  ┌──────────────────────────────┐
  │  SCORECARD / DECISION        │
  │  approve / decline / pricing │
  │  + adverse action reason     │
  └──────────────────────────────┘
```

**Data**: Credit bureau data, bank statements, repayment history. Label: 90+ day default within 12 months. Handle survivorship bias (only approved applicants have outcome labels — use reject inference carefully).

**Model**: Logistic regression with Weight-of-Evidence (WoE) features is the regulatory baseline (interpretable, auditable). GBDT improves Gini significantly; requires SHAP for explanations. **Avoid protected attributes** (race, gender) to comply with fair lending laws.

**Offline metrics**: Gini coefficient (= 2×AUC−1), KS statistic, calibration. **Online**: default rate by score band vs. forecast.

**Serving**: Batch (origination scoring) or real-time (instant credit decisions). Explainability is required — output top 3 reasons for decline.

**Iteration**: Population Stability Index (PSI) to detect applicant distribution shift. Retrain when PSI > 0.2. Re-validate for fairness at each retraining cycle.

---

## Online vs Batch Serving, Feature Stores, and Drift

### Online vs Batch

| Dimension | Batch | Online (Real-time) |
|-----------|-------|-------------------|
| Latency | Minutes–hours | <100–500 ms |
| Use cases | Churn, forecasting, credit | Fraud, CTR, ETA |
| Complexity | Low | High |
| Freshness | Hours | Seconds |
| Infrastructure | Spark/Airflow + DB | Kafka + feature store + model server |

### Feature Store

A feature store prevents **training/serving skew** — the #1 silent killer of production ML systems.

```
             ┌─────────────────────────────┐
             │        FEATURE STORE        │
             │                             │
  Batch ETL ─┤ Offline store (S3/BigQuery) │─ Training jobs
             │                             │
  Streaming ─┤ Online store (Redis/DynamoDB)│─ Real-time inference
             │                             │
             └─────────────────────────────┘
                    ↑ same feature logic ↑
                    (no skew between train/serve)
```

Key properties: shared transformation logic, schema versioning, point-in-time correct joins (no future leakage), feature lineage tracking.

### Training/Serving Skew

Occurs when features computed at training time differ from features computed at inference. Examples:
- Different aggregation windows (train: 7-day avg, serve: 6-day avg due to lag)
- Different null handling
- Normalization computed on full dataset but applied differently at serve time

**Prevention**: Centralize transformation logic in the feature store; test feature parity with shadow mode before deploying.

### Drift & Monitoring

```
Production Monitoring Stack
─────────────────────────────────────────────
Input features   → PSI / KS test vs. training baseline
Model outputs    → prediction distribution shift
Business KPIs    → alert if CTR / fraud rate / NDCG degrades
Ground truth     → compare model predictions to outcomes
                   (when labels arrive with delay)
─────────────────────────────────────────────
```

| Drift type | What shifts | Detection |
|------------|-------------|-----------|
| Data drift | Input feature distribution | PSI, KS test |
| Concept drift | P(y\|x) relationship | Monitor accuracy over time |
| Label drift | Target distribution | Monitor outcome rates |
| System drift | Pipeline bugs, schema changes | Data validation (Great Expectations) |

**Retraining triggers**: scheduled (nightly/weekly), performance-based (metric drops below threshold), or event-based (major product change).

---

## Interview Questions

### Q: How would you design a recommendation system for a new platform with no user history (cold start)?

**Strong answer:**
Cold start is a spectrum — new user, new item, or both. For new users, fall back to popularity-based recommendations segmented by coarse attributes (geo, device, referral source) gathered at onboarding. Collect implicit signals (clicks, skips) quickly and transition to personalized models after ~5 interactions. For new items, use content-based features (title embeddings, genre, metadata) to place the item in embedding space without historical data. The long-term fix is to invest in an exploration strategy (epsilon-greedy or Thompson sampling) that deliberately shows new items to a fraction of users to gather signal quickly.

### Q: What is training/serving skew and how do you prevent it?

**Strong answer:**
Training/serving skew occurs when the feature computation logic differs between the training pipeline and the inference pipeline, causing the model to see a different distribution at serve time than what it was trained on. Common causes: different aggregation windows, different null-filling strategies, or normalization constants computed on a different dataset slice. Prevention requires centralising all feature transformation logic in a **feature store** so both training jobs and the inference API draw from the same code path. Shadow deployment — running the new inference pipeline alongside the old one and comparing feature values before go-live — is a strong operational safeguard.

### Q: Fraud labels often arrive days after the transaction. How does this affect model training and evaluation?

**Strong answer:**
Label delay means that recent transactions appear negative (no fraud confirmed yet) even if they are actually fraud in progress — this is called "censored labels." If you naïvely train on the latest data, the model learns that recent transactions are safe, which is exactly backwards. The fix is to use a **label delay cutoff**: only use transactions that are old enough for labels to have matured (e.g., 14+ days old). For evaluation, similarly hold out a time window that is fully labelled. Additionally, implement a feedback ingestion pipeline that updates labels retroactively and triggers retraining once confirmed fraud arrives.

### Q: CTR is 1–2%. How do you handle class imbalance in a click prediction model?

**Strong answer:**
Negative downsampling is the standard approach: keep 100% of positive (clicked) examples and randomly sample 10–20% of negatives to rebalance the dataset. This speeds up training and improves recall, but it distorts the output probability: if you downsample negatives by 10×, raw model scores will be inflated. **Recalibration** is therefore mandatory — apply Platt scaling or isotonic regression on a held-out set to restore accurate probability estimates, since these probabilities feed directly into eCPM auction calculations. Evaluation should use log-loss and calibration curves, not just AUC.

### Q: How do you evaluate a ranking system offline when click data has position bias?

**Strong answer:**
Higher-ranked positions receive more clicks regardless of quality, so naive click rates are biased estimators of relevance. Correct for this using **inverse propensity scoring (IPS)**: weight each training example by 1/P(shown at that position), where position propensities are estimated from randomization experiments (randomly shuffling results for a small traffic slice). For evaluation, use **unbiased NDCG** via IPS weights rather than raw NDCG. Additionally, maintain a set of human relevance judgements for a fixed query sample so you have a ground-truth offline evaluation set that is completely free of position bias.

### Q: When should you use a two-stage retrieve-then-rank architecture versus a single-stage model?

**Strong answer:**
Single-stage works when the item corpus is small enough to score everything in time (e.g., <10,000 items). Once the corpus grows to millions, scoring every item per request exceeds latency budgets, so you split into retrieval (fast, high recall, lower precision — approximate nearest-neighbour on embeddings) followed by ranking (slower, high precision on the small candidate set). The boundary is roughly when you cannot score all items within your latency SLA on available compute. The tradeoff: two-stage introduces recall errors at retrieval that no ranker can recover, so retrieval recall@K must be monitored as a primary metric.

### Q: How would you monitor an ML model in production and know when to retrain?

**Strong answer:**
Layer monitoring across three levels. First, **input health**: track feature distributions using PSI or KS tests against training baseline — alert when PSI > 0.2 for key features. Second, **output health**: track prediction score distributions; a sudden shift (e.g., average predicted fraud probability drops from 2% to 0.1%) often signals a data pipeline bug. Third, **outcome health**: when ground-truth labels are available (even with delay), compare model predictions against actuals and track rolling accuracy. Retraining can be scheduled (nightly/weekly), threshold-triggered (accuracy drops 5% below baseline), or event-triggered (a major product change invalidates historical patterns). Always maintain a **champion/challenger** setup so you can safely promote a retrained model via A/B test.

### Q: What is the difference between data drift and concept drift? Why does it matter?

**Strong answer:**
**Data drift** means the input feature distribution P(X) has shifted — for example, a sudden influx of mobile users changes the device-type distribution the model was trained on. **Concept drift** means the relationship between features and the target P(Y|X) has shifted — for example, the fraud patterns that used to predict fraud no longer do, because fraudsters have changed tactics. Data drift is detectable without labels (compare input distributions); concept drift requires observing outcomes. Both matter because a model that was accurate when trained can silently degrade in production. Data drift is the early warning signal; concept drift confirms the model needs retraining. In fraud especially, concept drift is adversarial and rapid.

### Q: How would you design a churn prediction system such that the business interventions are actually effective?

**Strong answer:**
Simple churn scoring (predict who will churn, then contact them all) fails because some churners will stay regardless of intervention and some will churn regardless — you waste budget on both. The right frame is **uplift modelling** (causal ML): model the *incremental* probability of retention *caused by* the intervention, not just P(churn). Train four-quadrant uplift models using historical A/B experiment data to identify "persuadables" — users who respond positively to outreach. Target only persuadables with your highest-cost interventions (discounts, CSM calls); use cheaper channels for sure churners. Validate with held-out A/B tests measuring incremental retention, not raw retention rate.

### Q: How does learning-to-rank (LTR) differ from standard binary classification, and which LTR approach would you recommend?

**Strong answer:**
Standard binary classification optimises per-document relevance independently, ignoring the ordering relationship between documents. LTR optimises a ranking quality metric directly. There are three LTR paradigms: **pointwise** (treat each doc independently — simplest, misses relative order), **pairwise** (learn that doc A should rank above doc B — RankNet), and **listwise** (optimise list-level metrics directly — LambdaMART/LambdaRank optimises NDCG). For production search and recommendation I recommend **LambdaMART** (GBDT with LambdaRank loss): it combines the interpretability and efficiency of GBDT with direct NDCG optimisation, and is the proven industry standard at Google, Microsoft, and e-commerce platforms. Layer a neural re-ranker on top once you need deep feature interactions.

### Q: Describe how you would set up an A/B test for a new ML model, including what metrics to track.

**Strong answer:**
First, split traffic randomly into control (existing model) and treatment (new model) groups — typically starting at 1% treatment. Randomise at the user level (not request level) to avoid within-user interference. Define primary and guardrail metrics before the experiment: primary might be CTR or revenue per user; guardrail metrics are things you must not harm (page load time, unsubscribe rate). Run the experiment long enough to achieve statistical significance — typically 1–2 weeks minimum to capture weekly seasonality. Use sequential testing or pre-register a fixed sample size to avoid peeking bias. Report both statistical significance (p-value) and practical significance (effect size, confidence interval). A model with better offline AUC can still lose an A/B test — always trust online metrics over offline proxies.

### Q: How would you handle a model that performs well in offline evaluation but poorly in production?

**Strong answer:**
This is one of the most common production ML failure modes. Start by checking for **distribution shift**: compare production input features against the training distribution. Next, check for **label leakage** in the training pipeline — a feature that contains future information inflates offline metrics artificially. Check for **feedback loop issues**: if the model's own past predictions influenced what data was collected (e.e., only showing high-scoring items means you never observe low-scoring item outcomes), the training data is biased. Also verify that the **feature computation is identical** between training and serving (no training/serving skew). Finally, check the time split in offline evaluation — random splits leak future information for time-series-ordered data, inflating metrics. Fix the evaluation methodology first, then diagnose the model.

---

## References

- Exponent — [Machine Learning System Design Interview Guide (2026)](https://www.tryexponent.com/blog/machine-learning-system-design-interview-guide)
- Alirezadir — [ML System Design Framework (GitHub)](https://github.com/alirezadir/machine-learning-interviews/blob/main/src/MLSD/ml-system-design.md)
- System Design Handbook — [ML System Design Interview](https://www.systemdesignhandbook.com/guides/machine-learning-system-design-interview/)
- Analytics Vidhya — [System Design for ML Interviews: 10 Real Problems Walked Through (2026)](https://www.analyticsvidhya.com/blog/2026/06/system-design-for-ml-interviews-real-problems-solved/)
- System Design Handbook — [Fraud Detection System Design (2026)](https://www.systemdesignhandbook.com/guides/fraud-detection-system-design/)
- IGotAnOffer — [ML System Design Interview: Examples, Answers, Prep](https://igotanoffer.com/en/advice/machine-learning-system-design-interview)

---

---

## Glossary

| Term | Simple explanation | Purpose |
|---|---|---|
| **ML Objective** | The specific, measurable prediction task the model solves, mapped from a business goal | Translates vague goals like "increase retention" into concrete labels and loss functions |
| **Latency SLA** | The maximum acceptable response time for a system, such as under 150 ms | Sets the hard engineering constraint that determines whether batch or online serving is feasible |
| **QPS (Queries Per Second)** | The number of prediction requests a system must handle every second | Drives infrastructure decisions about model size, serving hardware, and caching |
| **Feature Store** | A centralised system that stores, versions, and serves pre-computed features to both training pipelines and inference endpoints | Prevents training/serving skew by ensuring both sides use identical transformation logic |
| **Training/Serving Skew** | A mismatch between how features are computed at training time versus at inference time | The most common silent cause of models that perform well offline but poorly in production |
| **Data Leakage** | Any feature that contains information about the future or about the target variable that would not be available at inference | Causes inflated offline metrics that do not hold on real-world data |
| **Cold Start Problem** | The challenge of making predictions for new users or new items that have no historical interaction data | Addressed with popularity-based fallbacks, content features, or onboarding surveys |
| **Candidate Retrieval** | The first stage of a two-stage ranking system that quickly narrows millions of items to a few thousand candidates | Optimises for recall rather than precision; must be fast enough to run on every request |
| **Ranking Model** | The second stage of a two-stage system that scores a small candidate set with rich features | Optimises for precision; can afford higher computational cost since only a small set is scored |
| **Two-Tower Network** | A neural architecture with one tower for users and one for items, both mapping to a shared embedding space | Enables approximate nearest-neighbour retrieval at scale by pre-computing item embeddings |
| **Collaborative Filtering** | A recommendation approach that predicts a user's preferences based on the preferences of similar users | Captures latent taste patterns without needing explicit item content descriptions |
| **Matrix Factorisation** | A collaborative filtering technique that decomposes a user-item interaction matrix into low-rank user and item embeddings | The classic baseline for recommendation systems; simple and interpretable |
| **ANN (Approximate Nearest Neighbour)** | An algorithm that finds vectors close to a query vector without scanning every item in the index | Enables fast embedding-based retrieval over millions or billions of items |
| **FAISS** | A library from Meta for fast approximate nearest-neighbour search on dense vectors | The most widely used tool for embedding retrieval in recommendation and search systems |
| **Implicit Feedback** | User behaviour signals like clicks, plays, or dwell time that reveal preferences without explicit ratings | More abundant than explicit ratings but noisier; requires careful negative sampling |
| **NDCG (Normalised Discounted Cumulative Gain)** | A ranking metric that rewards placing relevant items higher in the list | Primary offline metric for ranking systems; accounts for both relevance and position |
| **MAP (Mean Average Precision)** | The mean of the average precision scores across all queries in a ranking evaluation | Summarises ranking quality across many queries; sensitive to both precision and recall |
| **MRR (Mean Reciprocal Rank)** | The mean of one-over-the-rank of the first relevant result across all queries | Used when only the position of the first relevant result matters |
| **CTR (Click-Through Rate)** | The fraction of shown items or ads that were clicked | A primary online metric for ranking, recommendation, and ad systems |
| **eCPM (Effective Cost Per Mille)** | The expected revenue per thousand ad impressions, computed as P(click) × bid × 1000 | The quantity an ad auction maximises to decide which ad to show |
| **Factorisation Machine (FM)** | A model that efficiently learns pairwise feature interactions through low-rank embedding matrices | Effective for sparse categorical features in CTR and recommendation tasks |
| **DeepFM** | A model combining a factorisation machine and a deep neural network for joint low-order and high-order feature interaction learning | Improves CTR prediction by capturing both simple and complex feature interactions |
| **LambdaMART** | A gradient boosted tree model trained with a ranking-specific loss function that directly optimises NDCG | The industry-standard learning-to-rank algorithm for search and recommendation systems |
| **LTR (Learning to Rank)** | A family of supervised ML methods that train on relative orderings rather than absolute labels | Frames search ranking as a ML problem; sub-types include pointwise, pairwise, and listwise |
| **BM25** | A classical text-relevance scoring function based on term frequency and inverse document frequency | Fast lexical retrieval baseline; combined with dense retrieval in modern search systems |
| **Position Bias** | The tendency for items shown in higher positions to receive more clicks regardless of actual relevance | Must be corrected via inverse propensity scoring or randomisation experiments for fair ranking training |
| **Inverse Propensity Scoring (IPS)** | Weighting each training example by the inverse probability it was shown at that position | Produces an unbiased estimate of item relevance from position-biased click data |
| **Uplift Modelling** | Estimating the incremental effect of an intervention on a specific individual rather than predicting the outcome directly | Targets only users who respond to outreach, improving ROI of marketing or retention campaigns |
| **RFM Features** | Recency, Frequency, and Monetary features derived from transaction history | Classic customer behaviour features for churn prediction and customer segmentation |
| **SHAP (SHapley Additive exPlanations)** | A method that fairly distributes the model's prediction among input features using game theory | Provides consistent, model-agnostic feature importance and individual prediction explanations |
| **MAPE (Mean Absolute Percentage Error)** | The average percentage error between predicted and actual values | Interpretable forecasting metric; undefined when actuals are zero |
| **WAPE (Weighted Absolute Percentage Error)** | A variant of MAPE weighted by actual values to handle zero-sales periods | More robust than MAPE for intermittent demand forecasting |
| **DeepAR** | Amazon's RNN-based probabilistic forecasting model that trains a global model across many related time series | Produces calibrated prediction intervals and handles cold-start for new series |
| **Temporal Fusion Transformer (TFT)** | A Transformer-based forecasting model that handles multivariate time series with interpretable attention | Captures complex temporal patterns and external signals for demand forecasting |
| **Data Drift** | A shift in the distribution of input features over time relative to the training distribution | Detected by comparing feature distributions with PSI or KS tests; an early warning before accuracy degrades |
| **Concept Drift** | A shift in the relationship between features and the target variable over time | Requires retraining even if input distributions look normal; common in fraud and financial markets |
| **PSI (Population Stability Index)** | A metric quantifying the shift in a feature's distribution between two time periods | PSI > 0.2 conventionally triggers model review or retraining |
| **Canary Deployment** | Rolling a new model out to a small fraction of traffic before full rollout | Limits the blast radius of model regressions; enables data-driven promotion decisions |
| **Champion/Challenger** | Running a new model in shadow or low-traffic mode alongside the current production model | Allows safe comparison without full risk; the challenger replaces champion only after winning an A/B test |
| **Platt Scaling** | Fitting a logistic regression on top of raw model scores to produce calibrated probabilities | Required after negative downsampling in CTR models to restore accurate probability outputs |
| **Weight of Evidence (WoE)** | A feature transformation that encodes categories by their log-odds ratio relative to the target | Makes logistic regression work well on categorical features; widely used in credit scoring |
| **Gini Coefficient (Credit)** | A ranking metric for credit models equal to 2 × AUC − 1 | Industry-standard measure of a credit model's ability to separate good and bad borrowers |
| **Reject Inference** | Statistical techniques to estimate the credit outcomes of applicants who were rejected and have no observed labels | Corrects the survivorship bias inherent in credit datasets where only approved loans have outcomes |
| **Feedback Loop** | A cycle where model predictions influence future training data, potentially amplifying biases | Must be monitored to prevent the model from reinforcing its own errors over time |

*Previous: [Statistics and Probability](03-statistics-and-probability.md)*
