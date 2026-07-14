# ML in Production & MLOps

Getting a model to 0.9 accuracy in a notebook is the easy 10%. The hard 90% is **keeping it working in production** — serving it fast, feeding it consistent features, catching it when it silently rots, and retraining it as the world moves. This is **MLOps**, and it's where most ML value is won or lost. This page is the production-considerations companion to the [Data Science Lifecycle](07-data-science-lifecycle.md) (which ends where this begins: deployment).

## Table of Contents

- [Why Production ML Is Different (and Hard)](#why-production-ml-is-different-and-hard)
- [Deployment Modes — Batch vs Online vs Streaming](#deployment-modes--batch-vs-online-vs-streaming)
- [Serving Mechanics](#serving-mechanics)
- [Model Packaging & Reproducible Artifacts](#model-packaging--reproducible-artifacts)
- [Latency, Scale & Cost](#latency-scale--cost)
- [Feature Stores & Training-Serving Skew](#feature-stores--training-serving-skew)
- [MLOps — CI/CD/CT and Maturity Levels](#mlops--cicdct-and-maturity-levels)
- [Versioning & Reproducibility](#versioning--reproducibility)
- [Experiment Tracking & Model Registry](#experiment-tracking--model-registry)
- [Monitoring in Production](#monitoring-in-production)
- [Drift — Data, Concept & Label](#drift--data-concept--label)
- [Retraining Strategy](#retraining-strategy)
- [Release Strategies for ML](#release-strategies-for-ml)
- [Governance & Responsible AI](#governance--responsible-ai)
- [The Failure Modes That Actually Bite](#the-failure-modes-that-actually-bite)
- [Interview Questions](#interview-questions)
- [Glossary](#glossary)
- [References](#references)

---

## Why Production ML Is Different (and Hard)

Traditional software versions **one thing: code**. ML has to version and manage **three things that change independently — code + data + model** — and, crucially, **code doesn't lose accuracy as new data arrives, but models do.** A deployment can be "green" (no errors, low latency) while silently shipping a *worse model*.

Two ideas every interviewer wants to hear:
- **CACE — "Changing Anything Changes Everything."** In ML, no feature is truly independent; change one feature, hyperparameter, or data source and effects cascade unpredictably through the whole system. (Sculley et al., *Hidden Technical Debt in ML Systems*.)
- **The ML code is a tiny box in a huge system.** The famous Sculley diagram: actual ML code is dwarfed by data collection, verification, feature extraction, serving infra, and monitoring. MLOps is about all the plumbing around the model.

Source: [Sculley et al., "Hidden Technical Debt in ML Systems" (NeurIPS 2015)](https://papers.neurips.cc/paper/5656-hidden-technical-debt-in-machine-learning-systems.pdf)

---

## Deployment Modes — Batch vs Online vs Streaming

The first production decision: *when* do predictions happen? This drives everything else (latency, infra, cost).

```
 BATCH (offline)              ONLINE (real-time)            STREAMING (event-driven)
 precompute on a schedule,    predict per request,          predict on events from a
 store results, look them up  in the response path          stream (Kafka/Flink)
 ─────────────────────────    ─────────────────────────    ─────────────────────────
 high throughput, low cost    low latency (user waiting)    seconds/ms on live events
 predictions are STALE        needs fast feature lookup     fresh streaming features
 e.g. nightly recs, risk      e.g. search rank, pricing     e.g. fraud, live anomaly
```

| | Batch | Online | Streaming |
|---|---|---|---|
| Trigger | schedule (hourly/daily) | per user request | per event |
| Optimize for | throughput + cost | **latency** | freshness on live events |
| Staleness | high (by design) | none | none |
| Use when | inputs known ahead, staleness OK | input only known at request time | must react to live events |

**Intuition:** batch is "cook the meals in advance and reheat"; online is "cook to order"; streaming is "cook as ingredients arrive on the conveyor." **Most teams evolve batch → online → streaming** as freshness needs grow, and hybrids are common (batch-train, serve online).

Sources: [Google — batch inference](https://cloud.google.com/discover/what-is-batch-inference) · [Online vs batch vs stream](https://florisalexandrou.com/blog/online-vs-batch-vs-stream-processing-in-machine-learning/)

---

## Serving Mechanics

How the model is actually exposed:

- **API:** **REST** (simple, universal, easy to debug) vs **gRPC** (HTTP/2 + binary protobuf → faster, lower overhead for internal high-throughput calls). Common pattern: REST outside, gRPC inside.
- **Model servers** (don't hand-roll): **TF Serving** (TF-native, fast gRPC, batching), **NVIDIA Triton** (multi-framework, GPU, dynamic batching), **BentoML** (easy packaging/portability), **KServe** (Kubernetes-native, scale-to-zero via Knative), **vLLM** (LLMs). *(Note: TorchServe went limited-maintenance in 2025 — de-risk for new builds.)*
- **Packaging & orchestration:** **Docker** containers (model + code + pinned deps = a reproducible artifact) run on **Kubernetes** for scaling, or **serverless** endpoints (scale-to-zero) for spiky/low traffic.
- **The managed menu (SageMaker's clean taxonomy):** **Real-time endpoint** (persistent, low latency), **Serverless** (auto scale, cold-start risk), **Async** (queued, large payloads/long jobs, scales to zero), **Batch Transform** (score a dataset, shut down).

Sources: [KServe](https://kserve.github.io/website/) · [SageMaker inference types](https://caylent.com/blog/sagemaker-inference-types)

---

## Model Packaging & Reproducible Artifacts

The model that leaves your laptop must run *identically* elsewhere. Package considerations:

| Format | Note |
|---|---|
| **pickle** | Python built-in; **executes arbitrary code on load (unsafe)** and is tightly coupled to exact library versions — breaks across environments |
| **joblib** | Like pickle but faster for large NumPy arrays; same security/version caveats |
| **ONNX** | Open, framework-agnostic standard — **best portability**; decouples from the Python runtime; the safe cross-platform choice |

**The reproducible-artifact rule:** pin **every** dependency (including transitive) to training-time versions, capture the environment, and bundle it in a **container**. A model without its exact environment is not reproducible — sklearn models saved under one version routinely fail to load under another.

Source: [scikit-learn — model persistence](https://scikit-learn.org/stable/model_persistence.html)

---

## Latency, Scale & Cost

- **Measure tail latency (p50/p95/p99), not the average** — one slow request per session ruins UX. Track it alongside throughput and hardware utilization.
- **Batching for throughput:** grouping requests (dynamic batching; *continuous* batching for LLMs) keeps hardware saturated — big throughput and cost wins — at the cost of some queueing latency. **Throughput and latency are in tension:** bigger batches = cheaper per prediction but slower per request; tune batch size + timeout to your SLA.
- **Caching** repeated predictions and feature values cuts latency and cost.
- **Hardware:** CPU for small/light models, GPU for deep nets/LLMs.
- **Compress for speed/cost:** **quantization** (FP32→INT8/INT4, 2–4× smaller with minor accuracy loss), **distillation**, **pruning** — trade a little accuracy for much cheaper, faster inference.

---

## Feature Stores & Training-Serving Skew

**The single most common reason a great offline model fails in production: training-serving skew** — features computed *differently* in training vs serving, so the model sees a distribution it never trained on. Symptom: **offline metrics improve while online metrics stall or regress.**

A **feature store** is the fix — one system that defines features once and serves them to both training and inference:

```
                    ┌──────────────── FEATURE STORE ────────────────┐
  feature logic ──► │  OFFLINE store (warehouse) → training data     │
  defined ONCE      │  ONLINE store (Redis/DynamoDB) → real-time     │  ← same logic both sides
                    └───────────────────────────────────────────────┘
```

- **Offline store** (BigQuery/Snowflake): big historical data for **training** and batch scoring.
- **Online store** (Redis/DynamoDB/Aerospike): millisecond key-value lookups for **real-time serving**.
- **Point-in-time correctness:** training rows are built with point-in-time joins so each label only sees feature values *as of that timestamp* — **prevents temporal leakage by construction**.
- **Materialization:** precompute/load features from offline → online so serving is fast and consistent.

Tools: **Feast** (open source, Linux Foundation), **Tecton** (enterprise/streaming). Sources: [Feast docs](https://docs.feast.dev/) · [Training-serving skew](https://developers.google.com/machine-learning/guides/rules-of-ml)

---

## MLOps — CI/CD/CT and Maturity Levels

MLOps = DevOps for ML, but with an extra loop because **data and models drift**. The pipeline stages:

- **CI (Continuous Integration)** — beyond code tests, also **validate data and models** (schema/distribution checks, model-quality gates, no-NaN checks).
- **CD (Continuous Delivery)** — you ship a **pipeline** (that trains and deploys a model), not just a binary.
- **CT (Continuous Training) — unique to ML, no software equivalent.** Automatically retrain as data evolves. Triggered by **new data, a schedule, or production monitoring signals** (accuracy drop, drift). *This is the key thing that has no analog in traditional CI/CD.*

**Google's maturity levels (name these):**
- **Level 0 — manual:** notebooks, hand-offs, infrequent releases, no monitoring → model decay.
- **Level 1 — pipeline automation:** automated **continuous training** with data + model validation, triggers, a feature store.
- **Level 2 — full CI/CD:** the *pipeline itself* is built, tested, and shipped automatically; DS can update pipeline code and have it auto-deploy.

Sources: [Google — MLOps: continuous delivery & automation pipelines](https://cloud.google.com/architecture/mlops-continuous-delivery-and-automation-pipelines-in-machine-learning) · [CD vs CT](https://galileo.ai/blog/cd-vs-ct-ai)

---

## Versioning & Reproducibility

**To reproduce a model you must version all of: code + data + model + config + environment.** Versioning the model but **not the data** is *false reproducibility* — the exact training snapshot is gone, so you can't recreate it.

- **Code:** git.
- **Data:** **DVC** (git-like, for datasets/pipelines), **lakeFS** (branch/commit over object storage at scale), or lakehouse table formats with time-travel (**Delta Lake, Iceberg, Hudi**).
- **Model + config + env:** the model registry + a pinned container image.

Source: [DVC](https://dvc.org/) · reproducibility ties back to [CACE](#why-production-ml-is-different-and-hard).

---

## Experiment Tracking & Model Registry

- **Experiment tracking** (**MLflow**, **Weights & Biases**): log every run's **parameters, code version, metrics, and artifacts** so results are reproducible and comparable — the backbone of iterative modeling.
- **Model registry:** a central store for the model **lifecycle and lineage** — which run produced a model, its **versions**, and stage transitions (**None → Staging → Production → Archived**; newer MLflow uses **aliases + tags**). It's the controlled gateway between "trained" and "serving."

Source: [MLflow Model Registry](https://mlflow.org/docs/latest/ml/model-registry/)

---

## Monitoring in Production

You monitor **four layers**, easiest-and-cheapest to hardest:

1. **Operational health** — latency (p50/p95/p99), throughput, error rate, uptime, resource/cost. Standard SRE.
2. **Data quality** — schema conformance, null/out-of-range/out-of-vocabulary values, cardinality, feature freshness. **Cheapest and catches most real incidents.**
3. **Drift** — feature drift, prediction drift, label drift (next section).
4. **Model performance — the hard one, because ground truth is delayed.** True accuracy needs labels, which arrive late (fraud disputes: months; clinical outcomes: years) or never.

**The delayed-ground-truth problem (interview gold):** you often can't measure accuracy live. So you:
- monitor **prediction drift** (cheap, always available) as an early warning,
- use **implicit/natural labels** (clicks, purchases, disputes) where available,
- use **label-free performance estimation** (e.g. NannyML's confidence-based estimation) to *estimate* accuracy before labels land,
- then **reconcile with realized metrics** as labels arrive.

Tools: **Evidently** (OSS), **Arize**, **WhyLabs**, **Fiddler** (+ explainability), **NannyML** (label-free), or a DIY **Evidently + Prometheus + Grafana** stack. Source: [Chip Huyen — data distribution shifts & monitoring](https://huyenchip.com/2022/02/07/data-distribution-shifts-and-monitoring.html)

---

## Drift — Data, Concept & Label

Models decay because they're **static artifacts trained on a snapshot of a moving world.** The joint distribution `P(X, Y)` drifts. Three precise types (know *which factor moves*):

| Drift | What changes | Plain-English example | Typical fix |
|---|---|---|---|
| **Data drift / covariate shift** | `P(X)` — the inputs | more over-40 users at inference than in training | retrain on recent data |
| **Concept drift** | `P(Y\|X)` — the input→output *relationship* | same apartment features, but COVID drops the price | re-engineer features / redesign / re-frame |
| **Label / prior shift** | `P(Y)` — the target base rate | fraud rate jumps 2% → 8% | recalibrate / reweight |

**Temporal patterns:** *sudden* (a competitor's price change), *gradual* (slow trend/language shift), *seasonal* (holidays — beware false alerts from comparing across seasons).

**Detection** (per-feature: reference window vs current window):
- **PSI (Population Stability Index)** — the credit-scoring standard: **<0.1 stable, 0.1–0.2 moderate, >0.2 significant drift.**
- **KS test** (continuous), **chi-square** (categorical), **KL/JS divergence** (JS is symmetric and finite — safer than KL).
- **Prediction-distribution monitoring** is the cheap first line of defense (always available, no labels needed).

Source: [Evidently — drift](https://www.evidentlyai.com/ml-in-production/data-drift) · [PSI/KS thresholds](https://www.datacamp.com/tutorial/understanding-data-drift-model-drift)

---

## Retraining Strategy

- **Scheduled** (daily/weekly/monthly) — simple and predictable, but may retrain needlessly or react too late.
- **Trigger-based** — retrain when drift breaches a threshold (e.g. PSI > 0.2) or performance drops (e.g. 5–10% accuracy drop). Efficient and responsive, but needs reliable detection.
- **Hybrid (the recommended default)** — a baseline cadence **plus** on-demand triggers.
- **Full retrain** (from scratch on a fresh window — stable, expensive) vs **incremental/online** (cheap, current, but risks catastrophic forgetting).
- **Automate it:** a CT pipeline pulls fresh data → validates → trains → evaluates against the current **champion** on a holdout → gated promotion → shadow/canary → deploy, all logged for audit.

Source: [Model drift & retraining guide](https://futureagi.com/blog/model-vs-data-drift-how-to-identify-and-handle-it/)

---

## Release Strategies for ML

**The ML twist:** unlike regular software, you're validating **model quality**, not just uptime — and quality signals (labels) are often delayed. So the rollout is staged to limit blast radius:

```
 offline eval ──► SHADOW ──► CANARY ──► A/B (guardrailed) ──► full rollout
                 (mirror     (1%→5%→    (measure BUSINESS       (champion
                  traffic,    20%…       metric w/ stats;        switched)
                  log only,   ramp if    guardrails can          + auto-rollback
                  0 user      healthy)   block even a "win")      on SLO breach
                  risk)
```

- **Shadow / dark launch** — new model scores **mirrored** live traffic but its output **isn't served**. Zero user risk; validates real behavior before exposure.
- **Canary** — route a small % of live traffic, ramp up if healthy, auto-rollback on breach.
- **A/B test** — split traffic, compare a **business/quality metric** with statistical rigor (not just latency).
- **Blue-green** — two full environments, instant cutover + instant rollback.
- **Champion / challenger** — the live model (champion) is continuously challenged; promote a challenger only when it consistently wins.
- **Guardrail metrics** — secondary metrics (latency, cost, fairness) that can **block a rollout even if the primary metric improved**.

Source: [Safe ML rollout — shadow/canary](https://www.calibreos.com/learn/mlsd-canary-deployment) · [Champion/challenger](https://www.datarobot.com/blog/introducing-mlops-champion-challenger-models/)

---

## Governance & Responsible AI

For regulated or high-impact ML, production isn't done without governance:
- **Model cards** — standardized documentation (intended use, data, metrics, limitations, fairness); increasingly auto-generated as audit artifacts.
- **Audit trails & lineage** — versioned data + code + model + config, approval gates, who-changed-what logs.
- **Continuous fairness monitoring** — bias isn't a one-time dev check; it can emerge post-deployment as inputs shift (assess multiple, sometimes-conflicting fairness definitions).
- **Explainability** — SHAP/feature-importance/counterfactuals for regulated decisions and adverse-action notices.
- **Regulation** — **EU AI Act** (risk-tiered; high-risk systems need risk management + post-market monitoring; large penalties), **GDPR** (right to explanation), and banking-style **model risk management**.
- **Human-in-the-loop** — route sampled/flagged predictions to reviewers for high-impact actions.

Sources: [Responsible AI governance (Databricks)](https://www.databricks.com/blog/responsible-ai-governance) · [EU AI Act overview](https://artificialintelligenceact.eu/)

---

## The Failure Modes That Actually Bite

Name these — they're what breaks real systems:

1. **Training-serving skew** — the silent killer: features computed differently offline vs online. Fix with a **feature store** and by **logging served features** to train on.
2. **Data-pipeline breaks are the #1 *real* cause of "drift."** A large share of dashboard "drift" is actually a bug — broken ETL, wrong null-fill, a unit/encoding change, or the wrong model version. **Before blaming the model, check the pipeline** (one practitioner estimate: ~80% of "drift" is human/pipeline error).
3. **Feedback loops** — the model changes the data it later trains on (a recommender only sees labels for what it showed; a fraud model blocks the very cases it needs to learn from) → self-reinforcing bias / model collapse. Mitigate with exploration and holdout traffic.
4. **Silent failures** — no error thrown; the model keeps serving and just degrades decisions/revenue slowly. This is *why* you monitor even when everything looks "green."

---

## Interview Questions

### Q: A model scores 0.92 offline but performs poorly in production. Why?
**Strong answer:** Usually training-serving skew — features computed differently in prod than in training, so the live distribution differs from what the model trained on (offline metrics then overstate real performance). Other causes: data/concept drift after training, an offline metric that doesn't match the business metric, label leakage inflating the offline score, or latency/SLA failures. I'd check the feature pipeline first, then validate with shadow + a business-metric A/B before trusting offline numbers.

### Q: How do you monitor a model when you can't measure its accuracy live?
**Strong answer:** Ground truth is often delayed (fraud disputes take months). So I lead with label-free signals — prediction-distribution drift and input drift (PSI/KS) as early warnings — plus implicit labels (clicks, disputes) where available, and label-free performance estimation like NannyML's CBPE to estimate accuracy before labels land. Then I reconcile with realized metrics as labels arrive.

### Q: Explain the three types of drift.
**Strong answer:** It's about which probability factor moves. Data drift / covariate shift = P(X) changes (inputs shift, relationship stable) → retrain on fresh data. Concept drift = P(Y|X) changes (same inputs, different outcome — like COVID changing prices) → may need re-engineering or re-framing. Label/prior shift = P(Y) changes (base rate moves, e.g. fraud 2%→8%). I detect them with PSI/KS on feature and prediction distributions.

### Q: How do you safely roll out a new model?
**Strong answer:** Staged, to limit blast radius: offline eval → shadow (mirror live traffic, log only, zero user risk) → canary (1%→5%→ramp) → guardrailed A/B on the business metric → full rollout with the previous version kept hot for instant auto-rollback on SLO breach. The key ML difference is I'm validating model *quality* (often via delayed labels), not just uptime.

### Q: What's unique about CI/CD for ML vs software?
**Strong answer:** Two things. CI must validate **data and models**, not just code (schema/distribution checks, quality gates). And there's **Continuous Training (CT)** — automatic retraining triggered by drift or schedule — which has no analog in software because code doesn't lose accuracy as new data arrives, but models do.

### Q: Why is a feature store useful?
**Strong answer:** It defines each feature once and serves it to both training (offline store) and real-time inference (online store), eliminating training-serving skew. It also gives point-in-time-correct training joins that prevent temporal leakage by construction, and lets teams reuse features. It's the standard fix for "great offline, bad online."

---

## Glossary

| Term | Simple explanation | Purpose |
|---|---|---|
| **MLOps** | DevOps for ML — practices to build, deploy, and operate ML reliably | Manages the code + data + model that all change independently |
| **CACE** | "Changing Anything Changes Everything" — ML features are entangled | Explains why ML systems are fragile and need heavy testing |
| **Batch / Online / Streaming inference** | Predict on a schedule / per request / per event | Chooses the latency-cost-freshness trade-off for serving |
| **Model server** | Software that hosts a model behind an API (TF Serving, Triton, KServe) | Standardized, scalable serving without hand-rolling |
| **ONNX** | Open, framework-agnostic model format | Portable, runtime-decoupled model artifacts |
| **Reproducible artifact** | Model + pinned deps + environment in a container | Runs identically everywhere; core MLOps requirement |
| **p95 / p99 latency** | The latency 95%/99% of requests come in under | Captures tail latency that averages hide |
| **Quantization** | Lower-precision weights (FP32→INT8/INT4) | 2–4× smaller/faster inference for minor accuracy loss |
| **Feature store** | System serving the same features to training and serving | Eliminates training-serving skew; point-in-time correctness |
| **Training-serving skew** | Features computed differently in training vs production | The #1 cause of good-offline-bad-online models |
| **Point-in-time correctness** | Training rows only see feature values as of their timestamp | Prevents temporal/label leakage by construction |
| **CI / CD / CT** | Integrate/test, deliver pipelines, and continuously (re)train | CT (auto-retrain) is the ML-specific piece with no software analog |
| **MLOps maturity (0/1/2)** | Manual → pipeline automation → full CI/CD | Benchmarks how automated an ML org is |
| **Model registry** | Central store of model versions, stages, and lineage | Controlled gateway from "trained" to "in production" |
| **Experiment tracking** | Logging params/metrics/artifacts per run (MLflow/W&B) | Reproducible, comparable modeling |
| **Data drift (covariate shift)** | Input distribution P(X) changes | Common decay cause; often fixed by retraining |
| **Concept drift** | The input→output relationship P(Y\|X) changes | Deeper decay; may need re-engineering or re-framing |
| **Label / prior shift** | The target base rate P(Y) changes | Recalibrate/reweight the model |
| **PSI (Population Stability Index)** | A drift score comparing two distributions (>0.2 = significant) | Standard, thresholded drift detection |
| **Delayed ground truth / label lag** | Labels arrive long after predictions (or never) | Why live accuracy is hard; drives label-free monitoring |
| **Shadow deployment** | New model scores mirrored traffic, output not served | Validate a model on real traffic at zero user risk |
| **Canary** | Route a small traffic % to the new model, then ramp | Limit blast radius during rollout |
| **Champion / challenger** | Live model challenged by candidates; promote on win | Safe, continuous model improvement |
| **Guardrail metric** | A secondary metric that can block a rollout | Prevents shipping a "win" that regresses latency/cost/fairness |
| **Model card** | Standardized documentation of a model | Governance, transparency, audit |
| **Human-in-the-loop (HITL)** | Routing flagged/sampled predictions to human review | Safety net on high-impact decisions |

---

## References

- [Sculley et al. — Hidden Technical Debt in ML Systems (NeurIPS 2015)](https://papers.neurips.cc/paper/5656-hidden-technical-debt-in-machine-learning-systems.pdf)
- [Google Cloud — MLOps: Continuous delivery and automation pipelines](https://cloud.google.com/architecture/mlops-continuous-delivery-and-automation-pipelines-in-machine-learning) · [Google — Rules of ML](https://developers.google.com/machine-learning/guides/rules-of-ml)
- [Chip Huyen — Data Distribution Shifts and Monitoring](https://huyenchip.com/2022/02/07/data-distribution-shifts-and-monitoring.html)
- [Feast — feature store docs](https://docs.feast.dev/) · [MLflow — Model Registry](https://mlflow.org/docs/latest/ml/model-registry/) · [DVC](https://dvc.org/)
- [Evidently — data drift](https://www.evidentlyai.com/ml-in-production/data-drift) · [NannyML — performance estimation](https://www.nannyml.com/) · [Safe ML rollout (shadow/canary)](https://www.calibreos.com/learn/mlsd-canary-deployment)
- [KServe](https://kserve.github.io/website/) · [SageMaker inference types](https://caylent.com/blog/sagemaker-inference-types)

---

*Previous: [Data Science Lifecycle](07-data-science-lifecycle.md) | Up: [Guide Home](../README.md)*
