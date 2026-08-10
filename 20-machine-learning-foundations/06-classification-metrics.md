# Classification Metrics — Precision, Recall, F1, ROC-AUC

These four metrics trip people up because they answer **different questions** about the same classifier. Accuracy alone is misleading (especially on imbalanced data), so interviewers probe whether you understand *what each metric protects against* and *when to reach for which*. This page builds the intuition from one picture and one story, with a worked example and the traps. (Companion to [Loss Functions](05-loss-functions.md) — losses are what you *train* on; these are how you *judge* the result.)

## Table of Contents

- [The Setup: Four Outcomes](#the-setup-four-outcomes)
- [Precision — "Can I Trust a Yes?"](#precision--can-i-trust-a-yes)
- [Recall — "Did I Catch Everything?"](#recall--did-i-catch-everything)
- [The Precision-Recall Trade-off](#the-precision-recall-trade-off)
- [F1 — One Balanced Number](#f1--one-balanced-number)
- [ROC-AUC — Separability Across All Thresholds](#roc-auc--separability-across-all-thresholds)
- [PR-AUC — For the Rare Class](#pr-auc--for-the-rare-class)
- [Worked Numeric Example](#worked-numeric-example)
- [When to Use Which](#when-to-use-which)
- [Interview Questions](#interview-questions)
- [Glossary](#glossary)
- [References](#references)

---

## The Setup: Four Outcomes

Every prediction from a yes/no (binary) classifier lands in one of four boxes. Anchor to a **disease test**:

|  | Actually sick (positive) | Actually healthy (negative) |
|---|---|---|
| **Test says sick** | ✅ **True Positive (TP)** — caught it | ❌ **False Positive (FP)** — false alarm |
| **Test says healthy** | ❌ **False Negative (FN)** — **missed it** | ✅ **True Negative (TN)** — correctly cleared |

This is the **confusion matrix**. The two errors are *not* equal — a false alarm (FP) and a miss (FN) have different costs — and precision vs recall are just two different ways of caring about them.

**Why not just use accuracy?** Accuracy = (TP + TN) / everything. On **imbalanced** data it's a trap: if only 1% of people are sick, a model that says "healthy" to everyone is **99% accurate** and completely useless (it never catches a sick person). That's exactly why we need precision and recall.

---

## Precision — "Can I Trust a Yes?"

> **Why (the rationale):** When a false alarm (FP) is costly — spam filter blocking legitimate email, fraud flag freezing a valid account, a recommendation showing an irrelevant result — you want to maximize the trustworthiness of every positive prediction, measured by precision.
> **When to use:** Optimize for precision when the cost of a false positive is high and you can afford to miss some real positives: spam detection, false-positive fraud blocks, content recommendation quality, ad relevance.
> **Nuances & gotchas:** Precision can be gamed by being extremely conservative (only predict "positive" when near-certain) — you get perfect precision but terrible recall and miss almost everything real. Always pair precision with recall; use F1 or the PR curve for a holistic view.

```
 Precision = TP / (TP + FP)
```

Of everything you **flagged as positive**, how many were actually positive.

- **The question:** "Of the people I told they were sick, how many really are?"
- **High precision = few false alarms.**
- **Care about it when a false positive is expensive** — marking a real email as spam (user misses it), flagging an innocent transaction, recommending a bad product, a false fraud accusation.
- **Mnemonic:** precision = **purity of your positives**.

---

## Recall — "Did I Catch Everything?"

> **Why (the rationale):** When a miss (FN) is costly — cancer undetected, fraud uncaught, a security threat not flagged — you want to maximize coverage of all real positives, measured by recall.
> **When to use:** Optimize for recall when the cost of missing a true positive is high: cancer/disease detection, fraud/security detection, critical safety defect identification, any scenario where "missing it" causes harm.
> **Nuances & gotchas:** Recall can be trivially maximized by predicting "positive" for everything — 100% recall, 0% precision. Always pair with precision. Recall is equivalent to the True Positive Rate (TPR) and Sensitivity — the same quantity used as the y-axis of the ROC curve.

```
 Recall = TP / (TP + FN)         (also called Sensitivity or True Positive Rate)
```

Of all the **actual positives**, how many did you find.

- **The question:** "Of all the truly sick people, how many did my test catch?"
- **High recall = few misses.**
- **Care about it when a false negative is expensive** — missing a cancer, missing fraud, letting a threat through airport security, missing a critical safety defect.
- **Mnemonic:** recall = **coverage of the real positives**.

> Precision looks at the **column** you predicted positive; recall looks at the **row** that is actually positive. Same TP on top, different denominator — that's the whole difference.

---

## The Precision-Recall Trade-off

A classifier outputs a **score/probability**; you pick a **threshold** above which you call it "positive." Moving that threshold trades the two metrics — picture a **fishing net**:

```
 Cast a HUGE net  →  catch every fish (high RECALL)     but also boots & junk (low PRECISION)
 Cast a TINY net  →  everything caught is a fish (high PRECISION)   but you miss lots (low RECALL)
```

- **Lower the threshold** (say "positive" more easily) → **recall ↑, precision ↓** (more catches, more false alarms).
- **Raise the threshold** (be stricter) → **precision ↑, recall ↓** (cleaner positives, more misses).

You almost always trade one for the other, so you **choose the threshold based on which error hurts more** for your problem — not a default of 0.5.

---

## F1 — One Balanced Number

> **Why (the rationale):** A single metric is needed to rank models or set an optimization objective; F1 uses the harmonic mean to balance precision and recall so that a model must be good at both — not just one — to score well.
> **When to use:** Imbalanced classes where accuracy misleads; when you need a single number to compare models or tune a threshold; classification leaderboards. Use Fβ (β>1 boosts recall weight) when misses are more costly than false alarms.
> **Nuances & gotchas:** F1 = 0 if either precision or recall is 0 — it cannot be gamed by extremes. F1 is threshold-dependent; report at the business-relevant operating threshold, not 0.5. For extreme imbalance, PR-AUC is more informative than F1 at a single threshold because it summarizes across all thresholds. Macro vs micro vs weighted F1 differ significantly on multi-class problems.

```
 F1 = 2 · (Precision · Recall) / (Precision + Recall)     ← harmonic mean
```

- **Why the harmonic (not plain) mean?** It **punishes imbalance** — you can't cheat by maxing one metric and ignoring the other. Precision 1.0 + recall 0.0 → plain average 0.5, but **F1 = 0**. F1 is high **only when precision AND recall are both high**.
- **Use F1 when** you need a single score and the classes are **imbalanced** (where accuracy lies).
- **Fβ variant:** weight recall higher (β>1, e.g. **F2** for cancer screening where misses are worse) or precision higher (β<1) when the two errors have different costs.

---

## ROC-AUC — Separability Across All Thresholds

> **Why (the rationale):** Provides a threshold-free measure of ranking quality — the probability that the model scores a random positive higher than a random negative — enabling model comparison without committing to an operating point.
> **When to use:** Comparing models during development; roughly balanced classes; when you want to understand separability before picking a threshold; binary classification baselines. AUC = 1 is perfect; 0.5 is random.
> **Nuances & gotchas:** On heavily imbalanced datasets, ROC-AUC can be misleadingly optimistic because the large TN count keeps FPR artificially low — a model can have AUC 0.95 while completely failing on the rare positive class. In that case use PR-AUC. AUC < 0.5 means the model is ranking backwards — flip predictions for a free performance improvement.

Precision, recall, and F1 all depend on **one chosen threshold**. ROC-AUC steps back and asks: **how good is the model regardless of threshold?**

- The **ROC curve** plots **True Positive Rate (recall)** on the y-axis vs **False Positive Rate** (`FP / (FP + TN)`) on the x-axis, as you sweep the threshold from strict to loose.
- **AUC = the area under that curve.** The clean intuition:

> **AUC = the probability that the model scores a random *positive* higher than a random *negative*.**

- **AUC = 1.0** → perfect separation; **0.5** → coin flip (the diagonal); **< 0.5** → worse than random (ranking backwards — flip the labels).
- **What it really measures: ranking / separability**, not a single decision point. Two models with the same accuracy can have different AUC — the higher-AUC one orders positives above negatives more reliably, giving you **better thresholds to choose from later**.

```
 TPR
 1.0 │        ┌──────  ← strong model (AUC ≈ 0.95): hugs the top-left corner
     │      ┌─┘
     │    ┌─┘   . . . . .  ← random guess (AUC = 0.5): the diagonal
     │  ┌─┘  . .
     │┌─┘ . .
 0.0 └───────────── FPR
     0            1.0
```

**Threshold-free** is the superpower: use ROC-AUC to **compare models** or **rank candidates**; then pick an operating threshold from the curve based on your precision/recall needs.

---

## PR-AUC — For the Rare Class

> **Why (the rationale):** Ignores true negatives entirely (unlike ROC) and focuses exclusively on the positive class — making it the honest metric when positives are rare and TN abundance would otherwise inflate ROC-AUC.
> **When to use:** Any severe class imbalance scenario: fraud detection, disease diagnosis, rare-event anomaly detection, defect detection. The empirical rule: if the positive class is <5% of the dataset, prefer PR-AUC over ROC-AUC.
> **Nuances & gotchas:** A random classifier on imbalanced data has a PR-AUC equal to the base positive rate (e.g., 0.01 for 1% positives) — not 0.5 like ROC-AUC. This makes PR-AUC harder to interpret as an absolute number but more honest about performance on the minority class. Always report the baseline (positive rate) alongside PR-AUC.

**The ROC-AUC trap:** on **heavily imbalanced** data, ROC-AUC can look **flatteringly high** because the huge pile of true negatives makes the False Positive Rate tiny no matter what. It hides poor performance on the rare positive class.

- **PR-AUC** = area under the **Precision-Recall curve**. It ignores true negatives entirely and focuses on the **positive class**, so it's far more honest when positives are rare (fraud, disease, defects — often <1%).
- **Rule of thumb:** balanced classes → ROC-AUC is fine; **severe imbalance → prefer PR-AUC** (and report F1/recall at your chosen threshold).

---

## Worked Numeric Example

100 people, **10 are actually sick**. Your test produces:

|  | Actually sick | Actually healthy |
|---|---|---|
| **Test: sick** | TP = 8 | FP = 12 |
| **Test: healthy** | FN = 2 | TN = 78 |

- **Accuracy** = (8 + 78) / 100 = **86%** — sounds great…
- **Precision** = 8 / (8 + 12) = 8/20 = **0.40** — …but only 40% of "sick" calls are right (lots of false alarms).
- **Recall** = 8 / (8 + 2) = 8/10 = **0.80** — it catches 80% of the truly sick.
- **F1** = 2·(0.40·0.80)/(0.40+0.80) = 0.64/1.20 = **0.53** — the balanced score exposes the weak precision that accuracy hid.

**The lesson:** 86% accuracy looked fine; precision 0.40 revealed the real problem. Always look past accuracy on imbalanced data.

---

## When to Use Which

| Metric | Answers | Reach for it when… |
|---|---|---|
| **Precision** | "Are my positive predictions trustworthy?" | false positives are costly (spam, recommendations, false accusations) |
| **Recall** | "Did I find all the real positives?" | false negatives are costly (cancer, fraud, security, safety) |
| **F1 / Fβ** | "One balanced score" | imbalanced classes; need a single number; weight an error with Fβ |
| **ROC-AUC** | "How well does it rank/separate, any threshold?" | comparing models; picking a threshold later; roughly balanced classes |
| **PR-AUC** | Same, focused on the rare class | severe class imbalance (rare positives) |
| **Accuracy** | "Overall % correct" | only when classes are **balanced** and errors are equally costly |

---

## Interview Questions

### Q: What's the difference between precision and recall in one sentence?
**Strong answer:** Precision is *of the positives I predicted, how many are correct* (trustworthiness of a "yes"); recall is *of the actual positives, how many I caught* (coverage). Same numerator (TP), different denominator — precision divides by all *predicted* positives, recall by all *actual* positives.

### Q: When do you optimize for recall over precision?
**Strong answer:** When a **false negative** is far more costly than a false alarm — cancer screening, fraud detection, security threats. I'd rather investigate some false alarms than miss a real case. I'd lower the threshold, accept lower precision, and maybe use F2 (weights recall higher). The reverse (precision-first) applies when false positives are costly, like spam filtering or recommendations.

### Q: Why not just use accuracy?
**Strong answer:** Accuracy is misleading under class imbalance. If 1% of transactions are fraud, a model that predicts "not fraud" always is 99% accurate but catches zero fraud. Precision, recall, F1, and PR-AUC focus on the positive class, which is what actually matters there.

### Q: What does an ROC-AUC of 0.5 mean? Of 0.9?
**Strong answer:** 0.5 means the model can't separate the classes at all — no better than a coin flip (the diagonal). 0.9 means that if you pick a random positive and a random negative, the model gives the positive a higher score 90% of the time. AUC measures **ranking/separability across all thresholds**, independent of any single cutoff.

### Q: Your model has 0.95 ROC-AUC but performs poorly in production on a rare event. Why?
**Strong answer:** Classic imbalance trap. With very few positives, the massive true-negative count keeps the False Positive Rate tiny, inflating ROC-AUC even when precision on the rare class is bad. I'd switch to **PR-AUC** (focuses on the positive class), inspect precision/recall at the actual operating threshold, and possibly recalibrate the threshold or use class weighting.

### Q: Why is F1 the harmonic mean and not the arithmetic mean?
**Strong answer:** The harmonic mean punishes imbalance between precision and recall, so you can't game it. Precision 1.0 with recall 0.0 gives an arithmetic mean of 0.5 but an **F1 of 0** — F1 only rewards being good at *both*, which is the whole point of a single balanced metric.

### Q: How do you choose the decision threshold?
**Strong answer:** Not by defaulting to 0.5. I look at the precision-recall (or ROC) curve and pick the operating point that matches the business cost of each error — high recall for safety-critical detection, high precision when false positives are expensive — often by optimizing Fβ or a cost-weighted objective on a validation set.

---

## Glossary

| Term | Simple explanation | Purpose |
|---|---|---|
| **Confusion matrix** | The 2×2 table of TP / FP / FN / TN counts | The foundation every classification metric is computed from |
| **True Positive (TP)** | Predicted positive and actually positive | The correct "catches" |
| **False Positive (FP)** | Predicted positive but actually negative — a false alarm | The error precision penalizes |
| **False Negative (FN)** | Predicted negative but actually positive — a miss | The error recall penalizes |
| **True Negative (TN)** | Predicted negative and actually negative | Correct rejections; can dominate and inflate accuracy/ROC-AUC |
| **Precision** | TP / (TP + FP) — purity of your positive predictions | Measures trustworthiness of a "yes"; guards against false alarms |
| **Recall / Sensitivity / TPR** | TP / (TP + FN) — coverage of actual positives | Measures how many real positives you catch; guards against misses |
| **Specificity / TNR** | TN / (TN + FP) — coverage of actual negatives | The negative-class counterpart of recall |
| **False Positive Rate (FPR)** | FP / (FP + TN) — fraction of negatives wrongly flagged | The x-axis of the ROC curve |
| **F1 score** | Harmonic mean of precision and recall | A single balanced metric for imbalanced classes |
| **Fβ score** | Weighted harmonic mean favoring recall (β>1) or precision (β<1) | Tunes the metric to the relative cost of each error |
| **Threshold** | The score cutoff above which a prediction is called positive | The dial that trades precision against recall |
| **ROC curve** | Plot of TPR vs FPR across all thresholds | Visualizes the model's ranking quality independent of threshold |
| **ROC-AUC** | Area under the ROC curve | Probability a random positive is scored above a random negative; threshold-free separability |
| **PR curve / PR-AUC** | Precision-vs-recall curve and its area | Honest performance measure for the rare positive class under imbalance |
| **Class imbalance** | One class vastly outnumbers the other | Why accuracy and ROC-AUC mislead and precision/recall/PR-AUC are needed |
| **Accuracy** | (TP + TN) / total predictions | Overall correctness; only meaningful when classes are balanced |
| **Harmonic mean** | A mean that is dominated by the smaller value | Why F1 punishes a low precision *or* a low recall |

---

## References

- [Google ML Crash Course — Classification: ROC and AUC](https://developers.google.com/machine-learning/crash-course/classification/roc-and-auc)
- [Google ML Crash Course — Precision and Recall](https://developers.google.com/machine-learning/crash-course/classification/precision-and-recall)
- [scikit-learn — Metrics and scoring (precision, recall, F1, ROC)](https://scikit-learn.org/stable/modules/model_evaluation.html)
- [The Relationship Between Precision-Recall and ROC Curves (Davis & Goadrich, ICML 2006)](https://www.biostat.wisc.edu/~page/rocpr.pdf)

---

*Previous: [Loss Functions](05-loss-functions.md) | Next: [Data Science Lifecycle](07-data-science-lifecycle.md) | Up: [Guide Home](../README.md)*
