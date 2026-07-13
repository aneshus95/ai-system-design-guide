# Seller Behavior Analytics — Clustering, Hypothesis Testing, Predictive Modeling

> **My project.** Led a team of three to surface top-performing seller behaviors via **clustering, hypothesis testing, and predictive modeling**, then coached low performers on the high-impact behaviors — lifting their revenue attainment by **10%+**.

## Table of Contents

- [The Narrative](#the-narrative)
- [What I Built — Methodology](#what-i-built--methodology)
- [Deep Dive 1 — Clustering for Behavioral Segmentation](#deep-dive-1--clustering-for-behavioral-segmentation)
- [Deep Dive 2 — Hypothesis Testing (What Really Separates Top Performers)](#deep-dive-2--hypothesis-testing-what-really-separates-top-performers)
- [Deep Dive 3 — Predictive Modeling & SHAP](#deep-dive-3--predictive-modeling--shap)
- [Deep Dive 4 — Measuring the 10% Lift (the Hard Part)](#deep-dive-4--measuring-the-10-lift-the-hard-part)
- [Interview Q&A](#interview-qa)
- [Honest Caveats](#honest-caveats)
- [References](#references)

---

## The Narrative

**Situation.** Sales leadership had a hunch that top sellers "just do things differently," but no data-backed view of *which* behaviors actually drive performance — so coaching was generic and low performers stayed stuck.

**Task.** Identify the behaviors that genuinely separate high from low performers, and turn that into targeted coaching that measurably moves the needle.

**Action.** Leading a team of three, I split the work into three rigor-linked workstreams: **clustering** to surface behavioral personas, **hypothesis testing** to confirm which behaviors *significantly* differ between top and low performers (not just noise), and **predictive modeling with SHAP** to rank which controllable behaviors drive revenue attainment. We then translated the top drivers into concrete coaching plays and measured the result against a comparison group.

**Result.** Coached low performers lifted revenue attainment by **10%+** relative to comparison — attributable to the intervention rather than chance.

---

## What I Built — Methodology

```
 seller behavioral data (activity, cadence, discounting, pipeline hygiene, ...)
        │
        ├─►  CLUSTERING ─────────► personas: high-activity closers vs low-cadence laggards
        │
        ├─►  HYPOTHESIS TESTING ─► which behaviors DIFFER significantly (top vs low)?
        │                          t-tests + effect size + multiple-comparison correction
        │
        └─►  PREDICTIVE MODEL ───► SHAP: which CONTROLLABLE behaviors drive attainment?
                        │
                        ▼
             coaching plays for low performers
                        │
                        ▼
             measure lift vs comparison group  →  +10% revenue attainment
```

---

## Deep Dive 1 — Clustering for Behavioral Segmentation

**k-means** partitions sellers into K groups by minimizing within-cluster sum-of-squares (inertia): assign each point to the nearest centroid → recompute centroids → repeat. It assumes roughly spherical, similar-variance clusters and is sensitive to initialization (`k-means++`) and feature scale.

- **Feature scaling is critical** — k-means uses Euclidean distance and can't rescale per axis, so a feature on a larger numeric scale dominates. **Standardize (z-score) before clustering.**
- **Choosing K:** **elbow method** (inertia vs K, pick the bend — favors compactness) + **silhouette score** (`(b−a)/max(a,b)`, range −1..1 — favors separation). Use both.
- **Alternatives:** hierarchical/agglomerative (no preset K, dendrogram), DBSCAN (arbitrary shapes + outliers as noise), GMM (soft/probabilistic membership, per-axis covariance so less scale-sensitive).
- **Interpreting clusters:** profile each cluster by its centroid in *original units* and overlay outcome metrics (mean revenue attainment) to label personas.

> **Key caveat:** clustering is *unsupervised* and descriptive — the "top vs low performer" split is a hypothesis to **confirm** with a statistical test (Deep Dive 2), not assume.

Sources: [scikit-learn — Clustering](https://scikit-learn.org/stable/modules/clustering.html) · [Silhouette analysis — scikit-learn](https://scikit-learn.org/stable/auto_examples/cluster/plot_kmeans_silhouette_analysis.html)

---

## Deep Dive 2 — Hypothesis Testing (What Really Separates Top Performers)

To confirm "top performers do behavior X more" isn't just noise:

- **Null vs alternative:** H₀ = top and low performers do X at the same rate; H₁ = top performers do X more.
- **Two-sample t-test** compares the means of the two groups → a **p-value** = probability of a result this extreme *if H₀ were true* (not the probability H₀ is true). If p < α (commonly 0.05), reject H₀. Use **Welch's** t-test when variances/sizes differ.
- **Type I / II errors:** Type I = false positive (reject a true H₀, prob = α); Type II = false negative (miss a real effect, prob = β); **power = 1 − β**.
- **Multiple-comparison correction (the trap):** testing many behaviors inflates false positives — at 20 tests, P(≥1 false positive) > 64%. Apply **Bonferroni** (test each at α/m — conservative) or the less-conservative **FDR / Benjamini-Hochberg**.
- **Effect size (Cohen's d):** report *alongside* p-values — with large samples, trivially small differences become "significant," so effect size tells you whether the behavior gap is **practically meaningful**.

Sources: [Type I/II errors — Statistics By Jim](https://statisticsbyjim.com/hypothesis-testing/types-errors-hypothesis-testing/) · [p-hacking & multiple comparisons — MetricGate](https://metricgate.com/blogs/p-hacking-statistics/)

---

## Deep Dive 3 — Predictive Modeling & SHAP

Predict revenue attainment (regression) — or top-performer status (classification) — from behavioral features, using a tree ensemble (random forest / gradient boosting), then explain it:

- **SHAP (SHapley Additive exPlanations)** attributes each prediction's deviation from the average prediction to each feature, via game-theoretic Shapley values — giving both **magnitude and direction** (does more of behavior X push predicted attainment up or down?) and both **global** (overall drivers) and **local** (per-seller) explanations. `TreeExplainer` is fast/exact for tree models.
- Rank behaviors by SHAP → identify the top drivers → keep the **controllable** ones → translate into coaching plays.

> **Critical caveat:** feature importance is **associational, not causal**. Correlated features shadow each other, and a high-importance behavior may not be causally controllable. Confirm actionability, and validate the highest-value plays causally via the experiment in Deep Dive 4.

Sources: [SHAP — Lundberg & Lee (arXiv 1705.07874)](https://arxiv.org/pdf/1705.07874) · [Feature-importance caveats — ScienceDirect](https://www.sciencedirect.com/science/article/pii/S0020025522001268)

---

## Deep Dive 4 — Measuring the 10% Lift (the Hard Part)

This is what an interviewer will press hardest on, because it's the easiest place to fool yourself.

**The core threat — regression to the mean (RTM).** If you select the *lowest* performers and re-measure them, they'll appear to improve **even with no coaching**, because extreme scores contain luck/measurement error that averages out next period. This is the single biggest trap when "coaching low performers."

**Also:** confounders — seasonality, product launches, territory changes, quota resets.

**Credible designs (weak → strong):**

| Design | What it controls | Verdict |
|---|---|---|
| **Pre/post single group** | nothing — RTM + time trends confound it | Weakest; avoid as sole evidence |
| **Treatment vs control** (randomize *among* low performers) | both groups regress equally → difference isolates the true effect | **Clean fix for RTM** |
| **Difference-in-Differences** | (post−pre)ₜᵣₑₐₜ − (post−pre)_control removes stable traits + shared time trends | Strong quasi-experiment; needs **parallel-trends** assumption |

**Defensible claim:** *the 10%+ lift is the coached low-performers' attainment gain relative to a comparable uncoached comparison group (randomized or via DiD), which nets out regression to the mean and shared time trends.* Report the lift with statistical significance and adequate power. If only pre/post was available, say so — RTM can't be fully ruled out.

**Revenue attainment** = actual revenue ÷ target (quota), as a %. E.g., $90k against a $100k quota = 90%.

Sources: [Regression toward the mean — Wikipedia](https://en.wikipedia.org/wiki/Regression_toward_the_mean) · [Difference-in-Differences — World Bank DIME](https://dimewiki.worldbank.org/Difference-in-Differences) · [Quota attainment — Wall Street Prep](https://www.wallstreetprep.com/knowledge/quota-attainment/)

---

## Interview Q&A

**Q: How did you validate the clusters?**
Standardized features first (k-means is scale-sensitive), chose K via elbow + silhouette, confirmed stability across seeds/resamples, profiled centroids in original units with outcome overlays, and sanity-checked personas with sales stakeholders. Silhouette gave separation quality; business interpretability was the final gate.

**Q: How did you avoid p-hacking across many behaviors?**
Pre-specified the hypotheses, applied Bonferroni (or FDR) for multiple comparisons, and reported effect sizes (Cohen's d) — so "significant" also meant practically meaningful, not just a small p from a large sample.

**Q: How did you avoid regression to the mean when coaching the lowest performers?**
Used a comparison/control group of equally-low performers; both regress equally, so the treatment-minus-control difference is the real effect. Where randomization wasn't possible, difference-in-differences under a checked parallel-trends assumption.

**Q: How did you attribute the 10% lift to coaching vs other factors?**
Treatment/control (or DiD) nets out shared time trends, seasonality, and RTM; the lift is the incremental gain over the comparison group, tested for significance and adequate power.

**Q: How did model findings become actionable behaviors?**
Ranked behaviors by SHAP (magnitude + direction), filtered to *controllable* ones, and turned the top drivers into concrete coaching plays — acknowledging importance is associational, so we validated the highest-value plays through the controlled coaching experiment.

**Q: How did you lead the team of three?**
Split the workstreams (clustering / hypothesis testing / predictive modeling), set the analysis plan up front (pre-registered hypotheses to prevent p-hacking), ran review checkpoints, and owned the experimental-design rigor and stakeholder communication.

---

## Honest Caveats

- **The causal-strength of the 10% depends on the design** — with a randomized control or DiD you can claim causality; with only pre/post, RTM is a genuine limitation, so be honest about which you had.
- **Cluster labels (top vs low) must be validated** against an actual performance metric, not assumed from the clustering.
- **Feature importance is associational** — coaching should target *controllable* drivers, validated experimentally.
- **Confirm the exact "revenue attainment" denominator** (quota vs target vs forecast) and the t-test variant (Student vs Welch).

---

## References

- [scikit-learn — Clustering](https://scikit-learn.org/stable/modules/clustering.html) · [Silhouette analysis](https://scikit-learn.org/stable/auto_examples/cluster/plot_kmeans_silhouette_analysis.html)
- [Type I/II errors — Statistics By Jim](https://statisticsbyjim.com/hypothesis-testing/types-errors-hypothesis-testing/) · [p-hacking & multiple comparisons — MetricGate](https://metricgate.com/blogs/p-hacking-statistics/)
- [SHAP — Lundberg & Lee (arXiv 1705.07874)](https://arxiv.org/pdf/1705.07874)
- [Regression toward the mean — Wikipedia](https://en.wikipedia.org/wiki/Regression_toward_the_mean) · [Difference-in-Differences — World Bank DIME](https://dimewiki.worldbank.org/Difference-in-Differences)
- [Quota attainment — Wall Street Prep](https://www.wallstreetprep.com/knowledge/quota-attainment/)

---

*Previous: [PO Extraction + BERT Classifier](07-po-extraction-and-bert-classifier.md) | Up: [Guide Home](../README.md)*
