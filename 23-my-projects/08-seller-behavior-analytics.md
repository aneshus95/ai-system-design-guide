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

> **Why (the rationale):** Clustering uncovers natural behavioral personas without imposing predefined categories. Pre-labeling sellers as "top" vs "low" by quota attainment alone misses the nuance that sellers with similar attainment can reach it via very different behavioral patterns — clustering surfaces those patterns, which is what coaching needs.
> **When to use:** Clustering is appropriate for initial exploratory segmentation when the right groupings aren't known in advance, the feature space is multi-dimensional, and the goal is to surface interpretable personas for stakeholders. If you already know the groups (top/low by attainment), clustering is unnecessary — just compare those groups directly.
> **Nuances & gotchas:** k-means is sensitive to scale (a feature in thousands dominates over one in single digits), initialization (k-means++ helps but doesn't eliminate local minima), and the choice of K (elbow method is a heuristic, not a deterministic answer). Clusters are *descriptive* — they don't imply the personas *cause* the performance difference; that requires the hypothesis testing and experimental design in later deep dives. Cluster labels ("top performer persona") must be validated against actual attainment, not assumed from centroid values.

**k-means** partitions sellers into K groups by minimizing within-cluster sum-of-squares (inertia): assign each point to the nearest centroid → recompute centroids → repeat. It assumes roughly spherical, similar-variance clusters and is sensitive to initialization (`k-means++`) and feature scale.

- **Feature scaling is critical** — k-means uses Euclidean distance and can't rescale per axis, so a feature on a larger numeric scale dominates. **Standardize (z-score) before clustering.**
- **Choosing K:** **elbow method** (inertia vs K, pick the bend — favors compactness) + **silhouette score** (`(b−a)/max(a,b)`, range −1..1 — favors separation). Use both.
- **Alternatives:** hierarchical/agglomerative (no preset K, dendrogram), DBSCAN (arbitrary shapes + outliers as noise), GMM (soft/probabilistic membership, per-axis covariance so less scale-sensitive).
- **Interpreting clusters:** profile each cluster by its centroid in *original units* and overlay outcome metrics (mean revenue attainment) to label personas.

> **Key caveat:** clustering is *unsupervised* and descriptive — the "top vs low performer" split is a hypothesis to **confirm** with a statistical test (Deep Dive 2), not assume.

Sources: [scikit-learn — Clustering](https://scikit-learn.org/stable/modules/clustering.html) · [Silhouette analysis — scikit-learn](https://scikit-learn.org/stable/auto_examples/cluster/plot_kmeans_silhouette_analysis.html)

---

## Deep Dive 2 — Hypothesis Testing (What Really Separates Top Performers)

> **Why (the rationale):** Without hypothesis testing, any observed behavioral difference could be sampling noise — especially when you're comparing the tails of a performance distribution. A t-test with multiple-comparison correction separates real signals from false discoveries when many behaviors are tested simultaneously. Effect size (Cohen's d) guards against the trap of "statistically significant but practically trivial" with large samples.
> **When to use:** Two-sample t-tests for comparing continuous behavioral metrics (call frequency, deal size, discount rate) between two groups with adequate sample size. Mann-Whitney U test if the distributions are heavily non-normal. Multiple-comparison correction (Bonferroni or FDR) whenever more than a handful of hypotheses are tested simultaneously.
> **Nuances & gotchas:** The p-value alone is not sufficient — a very large sample makes trivially small differences "significant" (e.g., top performers make 0.1 more calls per week on average). Always pair p-value with Cohen's d to confirm practical significance. Pre-registering hypotheses before looking at data prevents HARKing (Hypothesizing After Results are Known). Welch's t-test should be the default over Student's when group sizes or variances differ — which is typical when comparing top vs low performers.

To confirm "top performers do behavior X more" isn't just noise:

- **Null vs alternative:** H₀ = top and low performers do X at the same rate; H₁ = top performers do X more.
- **Two-sample t-test** compares the means of the two groups → a **p-value** = probability of a result this extreme *if H₀ were true* (not the probability H₀ is true). If p < α (commonly 0.05), reject H₀. Use **Welch's** t-test when variances/sizes differ.
- **Type I / II errors:** Type I = false positive (reject a true H₀, prob = α); Type II = false negative (miss a real effect, prob = β); **power = 1 − β**.
- **Multiple-comparison correction (the trap):** testing many behaviors inflates false positives — at 20 tests, P(≥1 false positive) > 64%. Apply **Bonferroni** (test each at α/m — conservative) or the less-conservative **FDR / Benjamini-Hochberg**.
- **Effect size (Cohen's d):** report *alongside* p-values — with large samples, trivially small differences become "significant," so effect size tells you whether the behavior gap is **practically meaningful**.

Sources: [Type I/II errors — Statistics By Jim](https://statisticsbyjim.com/hypothesis-testing/types-errors-hypothesis-testing/) · [p-hacking & multiple comparisons — MetricGate](https://metricgate.com/blogs/p-hacking-statistics/)

---

## Deep Dive 3 — Predictive Modeling & SHAP

> **Why (the rationale):** A predictive model surfaces the joint importance of behaviors while controlling for other factors — something pairwise t-tests miss (a behavior may look important in isolation but be redundant given another correlated behavior). SHAP provides model-agnostic, game-theoretically grounded attribution with direction (does more of X raise or lower attainment?) and local explanations (per-seller insights for personalized coaching), making the model's output actionable rather than a black box.
> **When to use:** SHAP + tree ensemble is the right pattern when you want both prediction accuracy and feature-level explanations on tabular data. Use `TreeExplainer` (fast, exact) for tree models. For regression tasks where interpretability is paramount and accuracy is secondary, linear models with regularization can be more directly interpretable.
> **Nuances & gotchas:** SHAP importance is associational — a behavior's high SHAP value means it's correlated with high attainment in this dataset, not that it causally drives attainment. Correlated features split SHAP value between them, making importance rankings unstable when features are collinear. Filtering to *controllable* behaviors is a judgment call that requires domain knowledge — the model doesn't know what a manager can actually change. Global SHAP rankings may not be valid for individual sellers with atypical profiles.

Predict revenue attainment (regression) — or top-performer status (classification) — from behavioral features, using a tree ensemble (random forest / gradient boosting), then explain it:

- **SHAP (SHapley Additive exPlanations)** attributes each prediction's deviation from the average prediction to each feature, via game-theoretic Shapley values — giving both **magnitude and direction** (does more of behavior X push predicted attainment up or down?) and both **global** (overall drivers) and **local** (per-seller) explanations. `TreeExplainer` is fast/exact for tree models.
- Rank behaviors by SHAP → identify the top drivers → keep the **controllable** ones → translate into coaching plays.

> **Critical caveat:** feature importance is **associational, not causal**. Correlated features shadow each other, and a high-importance behavior may not be causally controllable. Confirm actionability, and validate the highest-value plays causally via the experiment in Deep Dive 4.

Sources: [SHAP — Lundberg & Lee (arXiv 1705.07874)](https://arxiv.org/pdf/1705.07874) · [Feature-importance caveats — ScienceDirect](https://www.sciencedirect.com/science/article/pii/S0020025522001268)

---

## Deep Dive 4 — Measuring the 10% Lift (the Hard Part)

> **Why (the rationale):** Pre/post comparison is intuitive but confounded by regression to the mean (RTM) — selecting the lowest performers and re-measuring them will show apparent improvement even with no coaching, because extreme scores contain luck/noise that averages out. A control group of equally-low uncoached performers quantifies how much of the improvement is RTM + time trend, letting you isolate the coaching-specific increment. DiD removes stable group differences *and* shared time trends simultaneously.
> **When to use:** Randomized treatment/control is the gold standard whenever you can randomly assign coaching to a subset of low performers. DiD is the right quasi-experimental fallback when randomization isn't possible but you have pre-treatment data for both groups and can verify the parallel-trends assumption. Pre/post alone should be reported with explicit RTM caveats.
> **Nuances & gotchas:** DiD's parallel-trends assumption is untestable in the post-period — you can only check that trends were parallel *before* the intervention. Quota changes, territory realignments, or product launches that affect only one group violate the assumption. Spillover effects (uncoached sellers adopting coached behaviors informally) can dilute the control group, understating the true lift.

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

## Key Decisions & Tradeoffs

| Decision | Options considered | Why chosen | Tradeoff accepted |
|---|---|---|---|
| **k-means clustering vs supervised segmentation by quota attainment** | Hard split top/low by quota attainment percentile; k-means on behavioral features; GMM soft clustering | Splitting by quota alone forces a binary that misses the behavioral nuance — sellers with similar attainment can reach it via very different patterns; k-means surfaces those patterns unsupervised, which is what coaching needs | k-means assigns hard cluster membership, masking sellers on the boundary; clusters are descriptive, not causal — top-performer label must be validated against actual attainment separately |
| **k-means vs DBSCAN vs GMM** | k-means; DBSCAN; GMM | k-means is interpretable (centroids in original units) and fast; stakeholders could inspect and name personas from centroid values; DBSCAN's cluster count and density parameters are harder to explain | k-means assumes spherical, roughly equal-variance clusters; if behavioral space is elongated or uneven, k-means may split or merge natural groups incorrectly |
| **Welch's t-test + Bonferroni vs FDR correction vs no correction** | No multiple-comparison correction (each test at α=0.05); Bonferroni; Benjamini-Hochberg FDR | Testing ~20 behavioral metrics simultaneously would inflate false discoveries to >64% without correction; Bonferroni is conservative but interpretable and defensible in a stakeholder presentation; FDR is an alternative for higher power | Bonferroni is overly conservative — it may miss real behavioral signals (Type II errors) when many behaviors are tested; FDR controls false-discovery rate at the cost of occasional false positives |
| **SHAP for feature importance vs coefficient-based (linear model) vs permutation importance** | Linear regression coefficients; random forest permutation importance; SHAP (TreeExplainer) | SHAP provides both magnitude and direction per feature and per seller (local explanations useful for individualized coaching); permutation importance gives global ranks without direction; linear model assumes additive effects | SHAP importance is associational — high SHAP doesn't imply controllability; correlated behaviors split SHAP value, making rankings unstable with collinear features |
| **Treatment/control design vs pre/post only for measuring lift** | Pre/post single-group comparison; matched control group; DiD | Pre/post alone is confounded by regression to the mean (selecting lowest performers then re-measuring will show apparent improvement even without coaching); a comparison group lets both groups regress equally, isolating the real coaching effect | Full randomization of who gets coached is organizationally difficult; matched control assumes comparability of treated and control groups, which must be checked |
| **Filter to controllable behaviors before coaching translation** | Use all SHAP top-k behaviors regardless of controllability; domain-expert filter; sales leadership review | A behavior may be statistically important but not actionable (e.g., territory quality, inherited account size); filtering to behaviors a manager can actually change ensures coaching is actionable rather than observational | Filtering on controllability is a judgment call — some behaviors are partly controllable; the boundary is subjective and must be negotiated with sales leadership |

---

## Production Issues & Fixes

*Representative issues for this system class — mapped to what I actually hit; tailor to your own incidents.*

- **Symptom:** Re-running k-means clustering on the same data with a different random seed produces materially different cluster assignments for ~20% of sellers, making persona labels unstable across runs. **Root cause:** k-means++ initialization still depends on the random seed, and the behavioral feature space has several clusters of similar density with sellers near the boundaries being assigned to different clusters depending on initialization. **Fix:** Ran k-means 20 times with different seeds and selected the run with the lowest inertia as the canonical solution; also computed per-seller cluster assignment stability score (fraction of runs assigning them to the same cluster) and flagged unstable sellers for manual review. **Prevention:** Tracked inertia variance across seeds at each analysis refresh; if variance exceeded 10% of the canonical inertia, triggered a re-review of K and feature scaling.

- **Symptom:** Clusters identified in Q1 become semantically misaligned by Q3 — the "high-activity closer" cluster's centroid shifts significantly, and some sellers previously labeled as top performers now fall into the "low-cadence laggard" cluster. **Root cause:** The seller population and behavior distributions are non-stationary — new product launches changed activity norms, territory realignments shifted deal sizes, and new hires diluted the behavior distributions within each cluster. **Fix:** Added a quarterly cluster refresh with explicit drift detection: computed centroid distance between consecutive quarterly runs and flagged clusters whose centroids moved more than 1 standard deviation. **Prevention:** Set a quarterly rerun schedule; added a cluster drift metric to the analytics dashboard; notified sales leadership before updating persona labels.

- **Symptom:** A post-analysis review finds that 7 out of 18 behavioral hypotheses initially flagged as "significant" (p < 0.05) don't hold when Bonferroni correction is applied — the original deck overstated the number of confirmed differentiators. **Root cause:** Multiple-comparison correction was applied after the initial stakeholder presentation (not before), and the first round used uncorrected p-values. **Fix:** Reran all 18 tests with Bonferroni-corrected threshold (α/18 ≈ 0.003), updated the findings to 5 confirmed behaviors, and re-briefed leadership on the corrected set. **Prevention:** Pre-registered all hypotheses and the correction method in a written analysis plan before touching the data; all significance claims in the deck must reference the corrected threshold.

- **Symptom:** Coaching program shows +10% lift in revenue attainment for the coached group vs comparison — but the coached group also had a new sales manager installed at the start of the period. **Root cause:** A confounder (manager change) co-occurred with the coaching intervention, making it impossible to attribute the lift solely to the coached behaviors. The DiD parallel-trends assumption is violated because the treatment group had a structural change the control group didn't. **Fix:** Acknowledged the confound in reporting; added a robustness check by analyzing attainment sub-groups that had the same manager pre- and post-intervention; the lift held for the stable-manager subset but was smaller. **Prevention:** Documented pre-intervention group characteristics (manager, territory, quota, tenure) and checked for balance before assigning treatment; any structural change during the measurement window is flagged in the analysis log.

- **Symptom:** The predictive model's SHAP top-3 behaviors shift noticeably between quarterly model refreshes — behaviors ranked 1–3 in Q1 drop to 4–7 in Q3, confusing the coaching playbook. **Root cause:** Two of the top-3 behaviors were highly correlated (call frequency and email touchpoint count); SHAP splits value between correlated features unpredictably, and small changes in the feature distribution across quarters shifted the split. **Fix:** Before finalizing the SHAP ranking, computed a feature correlation matrix and grouped correlated behaviors (>0.7 Pearson); reported them as a cluster of related drivers rather than separate ranked items. **Prevention:** Added a correlation-aware SHAP stability check at each quarterly refresh; if a top-10 behavior's rank shifted by more than 3 positions, investigated the correlation group before updating coaching materials.

- **Symptom:** The churn-risk predictive model (predicting which sellers miss quota next quarter) shows artificially high accuracy in CV but substantially worse precision in production. **Root cause:** The training dataset included seller activity features that were computed using the full quarter's data — e.g., "total calls this quarter" at training time included the end-of-quarter burst that happened *after* the prediction would need to be made mid-quarter. This is feature leakage: the model saw information not available at the prediction point. **Fix:** Re-engineered features to use only activity data available at prediction time (e.g., first 6 weeks of the quarter only); re-evaluated CV on the temporally-correct features, which showed lower but honest accuracy. **Prevention:** Added a feature-cutoff documentation step: every feature in the predictive model must document the data recency it uses and is reviewed against the prediction point before training.

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

## Glossary

| Term | Simple explanation | Purpose |
|---|---|---|
| **Clustering** | An unsupervised algorithm that groups data points into clusters based on similarity | Groups sellers into behavioral personas without requiring pre-labeled performance data |
| **k-Means** | A clustering algorithm that partitions N points into K clusters by minimizing within-cluster sum-of-squares | Standard baseline for behavioral segmentation; fast and interpretable |
| **k-Means++** | An improved k-means initialization that spreads starting centroids apart | Reduces the chance of bad local minima and produces more stable clusters |
| **Inertia (Within-Cluster SS)** | Sum of squared distances from each point to its cluster centroid | The objective k-means minimizes; lower means tighter clusters |
| **Elbow Method** | Plotting inertia vs. K and choosing K at the "elbow" where improvement flattens | A heuristic for selecting the number of clusters |
| **Silhouette Score** | A per-point measure `(b−a)/max(a,b)` where a = mean intra-cluster distance and b = mean nearest-cluster distance | Measures how well separated clusters are; ranges −1 (bad) to +1 (excellent) |
| **DBSCAN** | A density-based clustering algorithm that groups points in dense regions and marks sparse points as outliers | Finds arbitrarily shaped clusters without a preset K |
| **GMM (Gaussian Mixture Model)** | A probabilistic clustering model that assigns soft (probabilistic) membership to each cluster | Less scale-sensitive than k-means; models per-cluster covariance |
| **Hypothesis Testing** | A statistical procedure to decide whether an observed difference between groups is likely real or due to chance | Confirms that top-performer behaviors are genuinely different, not just noise |
| **Null Hypothesis (H₀)** | The default assumption that there is no difference between groups | Rejected when the p-value falls below the significance threshold |
| **p-Value** | The probability of observing a result as extreme as the data if H₀ were true | Not the probability H₀ is false; only meaningful relative to a pre-set α |
| **Two-Sample t-Test** | A test comparing the means of two independent groups | Tests whether top and low performers differ significantly on a behavioral metric |
| **Welch's t-Test** | A variant of the t-test that does not assume equal variances in the two groups | Preferred when group sizes or variances differ |
| **Type I Error (α)** | Rejecting a true H₀ — a false positive | Controlled by setting α (commonly 0.05); inflated by testing many behaviors |
| **Type II Error (β)** | Failing to reject a false H₀ — a false negative | Reduced by increasing sample size; power = 1−β |
| **Multiple-Comparison Correction** | Adjusting significance thresholds when running many simultaneous tests | Prevents false discoveries from accumulating across many behavior comparisons |
| **Bonferroni Correction** | Testing each hypothesis at α/m (where m = number of tests) | Conservative but simple; controls family-wise error rate |
| **FDR / Benjamini-Hochberg** | A less conservative correction that controls the expected fraction of false discoveries | Preferred when power is important and some false positives are acceptable |
| **Cohen's d** | A standardized effect size measuring how many standard deviations apart two group means are | Distinguishes practically meaningful differences from trivially small ones that are statistically significant |
| **SHAP (SHapley Additive exPlanations)** | A game-theoretic framework that assigns each feature a contribution to each individual prediction | Gives magnitude and direction of feature influence; enables both global and per-seller explanations |
| **TreeExplainer** | SHAP's fast, exact algorithm for tree-based models (random forest, gradient boosting) | Computes exact Shapley values efficiently without sampling |
| **Feature Importance** | A ranking of input features by how much they affect model predictions | Points to candidate coaching behaviors; must be verified as controllable and causal |
| **Revenue Attainment** | Actual revenue achieved divided by quota, expressed as a percentage | The primary outcome metric for measuring coaching effectiveness |
| **Regression to the Mean (RTM)** | The statistical tendency for extreme scores to move closer to the average on re-measurement | Makes low performers appear to improve even without any intervention; requires a control group |
| **Difference-in-Differences (DiD)** | A quasi-experimental design that compares the pre-to-post change in a treatment group to the same change in a control group | Removes shared time trends and RTM by measuring the treatment-minus-control delta |
| **Parallel-Trends Assumption** | The DiD requirement that treatment and control groups would have followed the same trend without the intervention | The key condition that must be checked before interpreting a DiD result causally |

---

*Previous: [PO Extraction + BERT Classifier](07-po-extraction-and-bert-classifier.md) | Up: [Guide Home](../README.md)*
