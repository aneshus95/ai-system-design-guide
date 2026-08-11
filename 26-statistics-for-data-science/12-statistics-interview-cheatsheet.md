# Statistics Interview Cheatsheet

A rapid-review reference for data science interviews. Scan before a technical screen; use the "If you see X → answer Y" table to anchor fast retrieval.

> **In plain English:** This page is your pre-interview flash card deck — not a textbook. Everything is compressed for speed. Expand on any topic using the detailed pages in this section.

---

## 1. If You See X → Use / Answer Y

*(≥ 25 rows; covers test selection, definitions, and common concept questions)*

| Scenario / Prompt (X) | Answer / Action (Y) |
|----------------------|---------------------|
| **Compare means of 2 independent groups (σ unknown, continuous)** | **Welch's two-sample t-test** (default; robust to unequal variances). Use Student's only if equal variances are confidently confirmed. |
| **Compare means of 3+ independent groups** | **One-way ANOVA**, then post-hoc (Tukey's HSD for equal n/variance; Games-Howell otherwise). *Never run multiple t-tests — inflates Type I error.* |
| **Before/after measure on same subjects** | **Paired t-test** (= one-sample t on differences d_i). Non-normal differences → Wilcoxon signed-rank. |
| **Two categorical variables — are they associated?** | **Chi-square test of independence**. Check: all expected frequencies ≥ 5. |
| **One categorical variable vs a hypothesized distribution** | **Chi-square goodness-of-fit test**. |
| **Two continuous variables — linear association?** | **Pearson correlation test** (t = r√(n−2)/√(1−r²)). Non-normal or ordinal → Spearman's ρ. |
| **Non-normal data or small n, two independent groups** | **Mann-Whitney U test** (non-parametric, rank-based). |
| **Non-normal data, two paired measurements** | **Wilcoxon signed-rank test**. |
| **Non-normal data, 3+ independent groups** | **Kruskal-Wallis H test**. |
| **One proportion vs a known value** | **z-test for proportions**; verify np₀ ≥ 5 and n(1−p₀) ≥ 5. |
| **Two proportions (e.g., A/B test CTR)** | **z-test for two proportions** or chi-square 2×2 contingency. |
| **Large n (≥ 30), σ known** | **z-test**. In practice, t-test with large n converges to same result. |
| **p-value definition asked** | Probability of observing data this extreme (or more) *assuming H₀ is true*. NOT the probability H₀ is true. (ASA 2016) |
| **"Is p < 0.05 enough to conclude the effect is important?"** | **No.** p conflates effect size with sample size. Report effect size (Cohen's d, η², r). Large n → tiny effect can be "significant." |
| **"Failing to reject H₀ means H₀ is true?"** | **No.** It means insufficient evidence against H₀. May simply lack power. |
| **Type I vs Type II error** | **Type I (α):** false positive — reject H₀ when it's true. **Type II (β):** false negative — fail to reject when H₁ is true. |
| **How to increase statistical power?** | Increase n; increase effect size; relax α (cautiously); reduce variance; use one-tailed if direction justified. |
| **When does the CLT apply?** | When n is large enough (rule of thumb: n ≥ 30), the sampling distribution of the mean is approximately Normal regardless of the population distribution. Skewed distributions need larger n. |
| **Mean vs median — which to report?** | **Mean:** symmetric distributions, no heavy outliers. **Median:** skewed data, ordinal, heavy-tailed, or when outliers dominate (e.g., income, house prices). |
| **Correlation vs causation** | Correlation measures linear co-movement (−1 to +1). Causation requires: controlled assignment (randomization or IV), temporal precedence, no confounding. |
| **MLE vs MAP** | **MLE:** maximize P(data\|θ) — no prior. **MAP:** maximize P(data\|θ)·P(θ) — incorporates prior; = MLE + regularization. MAP → MLE as n → ∞. |
| **Frequentist vs Bayesian** | **Frequentist:** parameters are fixed unknowns; probability = long-run frequency. **Bayesian:** parameters have distributions; update prior with data to get posterior. No single right answer — context and prior availability matter. |
| **Bootstrap — when to use?** | When the sampling distribution of a statistic is unknown or hard to derive analytically (e.g., median, trimmed mean, complex estimator); small n; constructing non-parametric CIs. |
| **A/B test sample size drivers** | Baseline rate/mean, minimum detectable effect (MDE), desired power (1−β), significance level α, and variance. Smaller MDE or lower α → larger n needed. |
| **Running 20 tests simultaneously (multiple comparisons)** | Apply **Bonferroni correction** (α/m) for strong control, or **Benjamini-Hochberg** (controls FDR) when many comparisons and some true effects expected. |
| **Normality check method** | Shapiro-Wilk (n < 50); Q-Q plot (any n, especially large); histogram. Note: at large n, trivial departures are flagged — use judgment. |
| **Equal variance check** | Levene's test. But: just use Welch's t-test by default — it doesn't require this assumption. |
| **Confidence interval interpretation** | If you repeated sampling many times, (1−α)×100% of CIs constructed this way would contain the true parameter. It does NOT mean 95% probability the parameter is in *this* interval. |
| **Duality of CI and hypothesis test** | Reject H₀: θ = θ₀ at level α ⟺ θ₀ lies outside the (1−α) CI. They are mathematically equivalent. |
| **Effect size for two means** | **Cohen's d** = (μ₁−μ₂)/s_pooled. Small: 0.2, Medium: 0.5, Large: 0.8. |
| **Effect size for ANOVA** | **η²** = SS_between/SS_total (biased). **ω²** is less biased. Small: 0.01, Medium: 0.06, Large: 0.14. |
| **When to prefer non-parametric tests** | Small n + non-normality; ordinal data; severe outliers; when median is the right estimand. *They are less powerful than parametric tests when assumptions hold.* |

---

## 2. Distributions at a Glance

| Distribution | Parameters | Mean | Variance | When it appears |
|-------------|-----------|------|----------|----------------|
| **Normal N(μ, σ²)** | μ, σ² | μ | σ² | Sample means (CLT), many natural phenomena |
| **Standard Normal N(0,1)** | — | 0 | 1 | z-scores, z-test statistic |
| **t(df)** | degrees of freedom | 0 (df>1) | df/(df−2) (df>2) | t-tests; heavier tails than Normal; → Normal as df→∞ |
| **F(d₁, d₂)** | two dfs | d₂/(d₂−2) | — | ANOVA F-statistic; ratio of two χ² variables |
| **χ²(df)** | df | df | 2·df | Goodness-of-fit, independence tests; sum of squared standard normals |
| **Binomial(n, p)** | n, p | np | np(1−p) | Count of successes in n trials; A/B test conversions |
| **Poisson(λ)** | λ | λ | λ | Count of rare events per interval; mean = variance |
| **Uniform(a, b)** | a, b | (a+b)/2 | (b−a)²/12 | Random sampling; prior in Bayesian with no information |
| **Bernoulli(p)** | p | p | p(1−p) | Single binary trial |
| **Exponential(λ)** | λ | 1/λ | 1/λ² | Time between Poisson events; memoryless |
| **Beta(α, β)** | α, β | α/(α+β) | — | Bayesian prior/posterior for proportions; bounded [0,1] |
| **Gamma(α, β)** | shape α, rate β | α/β | α/β² | Waiting times; conjugate prior for Poisson rate |
| **Log-Normal** | μ, σ² of log(X) | e^(μ+σ²/2) | — | Multiplicative processes; income, stock prices |
| **Geometric(p)** | p | 1/p | (1−p)/p² | Number of trials until first success |

---

## 3. Common Interview Questions with Model Answers

**Q1: What is a p-value?**
The p-value is the probability of observing a test statistic at least as extreme as the one computed, *assuming the null hypothesis is true*. It is not the probability that H₀ is true, and it does not measure effect size or practical importance. (ASA 2016 Statement)

**Q2: What is the difference between Type I and Type II errors?**
A Type I error (false positive, rate = α) occurs when you reject a true null hypothesis. A Type II error (false negative, rate = β) occurs when you fail to reject a false null. Power = 1 − β is the probability of correctly detecting a true effect. Reducing α increases β (and decreases power) — there is an inherent trade-off.

**Q3: Why use ANOVA instead of running multiple t-tests?**
With k groups, there are k(k−1)/2 pairwise t-tests. At α = 0.05 each, the family-wise error rate (P(at least one false positive)) grows rapidly — for 5 groups: ≈ 40%. ANOVA tests all groups simultaneously under one α, controlling the overall error rate. If ANOVA is significant, post-hoc tests (Tukey, Bonferroni) then identify specific pairs.

**Q4: What is statistical power, and how do you increase it?**
Power (1 − β) is the probability of rejecting H₀ when H₁ is true. You increase it by: (1) increasing sample size n, (2) targeting a larger effect size (e.g., stronger treatment), (3) reducing measurement variance, (4) increasing α (at the cost of more Type I errors), or (5) using a one-tailed test when the direction is pre-specified.

**Q5: What is the Central Limit Theorem and when does it apply?**
The CLT states that the sampling distribution of the sample mean converges to Normal as n increases, regardless of the population distribution. A common rule of thumb is n ≥ 30, but skewed or heavy-tailed populations require larger n. The CLT justifies using z- and t-tests on non-normal data with sufficient sample sizes.

**Q6: What is the difference between correlation and causation?**
Correlation quantifies the linear relationship between two variables (Pearson r ∈ [−1, 1]). Causation means changes in one variable directly produce changes in another. Establishing causation requires: random assignment (or a valid instrumental variable), temporal ordering (cause precedes effect), and ruling out confounders. Observational correlation alone never implies causation.

**Q7: Explain the confidence interval interpretation.**
A 95% CI is constructed such that, across many hypothetical repetitions of the study, 95% of the resulting intervals would contain the true population parameter. It is NOT a statement that "there is a 95% probability the parameter is in this particular interval" — the parameter is fixed; the interval is random.

**Q8: What is the relationship between a confidence interval and a hypothesis test?**
They are dual: a two-sided α-level hypothesis test rejects H₀: θ = θ₀ if and only if θ₀ falls outside the corresponding (1−α) confidence interval. CIs are generally more informative — they convey direction, magnitude, and precision, not just a binary reject/fail-to-reject decision.

**Q9: When would you use a non-parametric test?**
When data are ordinal (e.g., Likert scales), when the normality assumption is violated and n is too small for CLT to rescue, or when heavy outliers would distort mean-based statistics. Specifically: Mann-Whitney U instead of two-sample t; Wilcoxon signed-rank instead of paired t; Kruskal-Wallis instead of one-way ANOVA; Spearman ρ instead of Pearson r.

**Q10: What is MLE vs MAP estimation?**
Maximum Likelihood Estimation (MLE) finds θ that maximizes P(data | θ) — it uses only the data, no prior. Maximum A Posteriori (MAP) maximizes P(data | θ) · P(θ) — it incorporates a prior distribution over θ. MAP with a Gaussian prior is equivalent to L2-regularized regression; MAP → MLE as sample size grows (prior gets overwhelmed by data).

**Q11: Frequentist vs Bayesian statistics — core difference?**
Frequentists treat parameters as fixed unknowns and define probability as long-run frequency of events. Bayesians treat parameters as random variables with prior distributions, updated via Bayes' theorem to produce a posterior. Bayesian methods naturally quantify uncertainty over parameters and incorporate domain knowledge; frequentist methods are more standardized for hypothesis testing and require no prior specification.

**Q12: What is bootstrap resampling and when do you use it?**
Bootstrap resampling repeatedly samples with replacement from the observed data to empirically approximate the sampling distribution of any statistic. Use it when: the statistic has no closed-form sampling distribution (median, trimmed mean, AUC), sample size is small, or you need non-parametric confidence intervals. It is computationally intensive but broadly applicable.

**Q13: What drives sample size in an A/B test?**
Five factors: (1) baseline metric value, (2) minimum detectable effect (MDE) — smaller MDE needs more data, (3) desired power (1−β, typically 0.80), (4) significance level α (typically 0.05), and (5) variance of the metric. The classic formula for two proportions: n ≈ 2(z_{α/2} + z_β)² · p̄(1−p̄) / (MDE)².

**Q14: What is multiple testing correction?**
When you run m hypothesis tests simultaneously, the probability of at least one false positive grows. Corrections: **Bonferroni** (use α/m for each test — strict, controls family-wise error), or **Benjamini-Hochberg** (controls false discovery rate FDR — less conservative, preferred when many tests and some true effects are expected, e.g., genomics, A/A tests).

**Q15: What is Cohen's d and how do you interpret it?**
Cohen's d = (μ₁ − μ₂) / s_pooled — a standardized, unit-free measure of mean difference. Conventions: 0.2 = small, 0.5 = medium, 0.8 = large (Jacob Cohen, 1988). Always report alongside p-values because p alone conflates effect size with sample size.

**Q16: What is the difference between one-tailed and two-tailed tests?**
A two-tailed test checks for any difference (H₁: μ ≠ μ₀); a one-tailed test checks for a difference in a specific direction (H₁: μ > μ₀ or μ < μ₀). One-tailed tests are more powerful in the predicted direction but completely miss effects in the opposite direction. The direction must be pre-specified before data collection — choosing it after seeing the data is a form of p-hacking.

**Q17: Explain the bias-variance trade-off.**
Model error decomposes as: Total Error = Bias² + Variance + Irreducible noise. High bias (underfitting): model misses true patterns. High variance (overfitting): model fits noise, poor generalization. Regularization, cross-validation, and increasing training data help manage this trade-off.

**Q18: What is the chi-square test of independence, and what are its assumptions?**
It tests whether two categorical variables are associated in a contingency table, by comparing observed cell frequencies to those expected under independence. Assumptions: (1) independent observations, (2) expected frequency ≥ 5 in every cell (≥ 1 minimum, with ≤ 20% cells below 5 for larger tables). It does not measure strength of association — use Cramér's V for that.

**Q19: When is the paired t-test appropriate, and why is it more powerful?**
Use the paired t-test when the same subjects (or matched pairs) provide both measurements (e.g., before/after treatment). It is more powerful than the two-sample t-test because pairing removes between-subject variability — the SE is computed on differences dᵢ = x₁ᵢ − x₂ᵢ, which have smaller variance than unpaired observations when subjects are positively correlated.

**Q20: What is the duality of confidence intervals and hypothesis tests?**
A (1−α) CI and a two-sided α-level test are mathematically equivalent: the test rejects H₀: θ = θ₀ if and only if θ₀ lies outside the CI. Both are derived by inverting the same pivotal inequality. CIs are generally preferred in reporting because they communicate direction and magnitude, not just a yes/no decision.

---

## 4. Top Formulas to Memorize

```
# ───────── Test Statistics ─────────────────────────────────────────────────

z = (x̄ − μ₀) / (σ / √n)                         # one-sample z-test

t = (x̄ − μ₀) / (s / √n),  df = n − 1             # one-sample t-test

t = (x̄₁ − x̄₂) / √(s₁²/n₁ + s₂²/n₂)             # Welch's t-test (df via Satterthwaite)

t = d̄ / (s_d / √n),  df = n − 1                   # paired t-test

F = MS_between / MS_within                          # ANOVA F-statistic

χ² = Σ (O_i − E_i)² / E_i                          # chi-square

t = r√(n−2) / √(1−r²),  df = n − 2                # Pearson correlation test

# ───────── Effect Sizes ────────────────────────────────────────────────────

Cohen's d  = (μ₁ − μ₂) / s_pooled                  # two-group standardized diff

s_pooled   = √[((n₁−1)s₁² + (n₂−1)s₂²) / (n₁+n₂−2)]

η²         = SS_between / SS_total                  # ANOVA effect size

Cramér's V = √(χ² / (n · min(r−1, c−1)))           # categorical association

# ───────── Power / Sample Size ─────────────────────────────────────────────

Power      = 1 − β                                  # P(reject H₀ | H₁ true)

n (two proportions) ≈ 2(z_{α/2} + z_β)² · p̄(1−p̄) / MDE²

# ───────── Bayes ───────────────────────────────────────────────────────────

P(θ|data) ∝ P(data|θ) · P(θ)                       # posterior ∝ likelihood × prior

# ───────── Duality ─────────────────────────────────────────────────────────

Reject H₀: θ=θ₀ at α  ⟺  θ₀ ∉ (1−α) CI

# ───────── Confidence Interval (mean, σ unknown) ───────────────────────────

CI = x̄ ± t_{α/2, n−1} · (s / √n)

# ───────── Family-wise Error ───────────────────────────────────────────────

FWER ≈ 1 − (1−α)^m                                  # m independent tests
Bonferroni: use α* = α / m per test

# ───────── Pooled Proportion (two-proportion z-test) ───────────────────────

p̂ = (x₁ + x₂) / (n₁ + n₂)
z = (p̂₁ − p̂₂) / √(p̂(1−p̂)(1/n₁ + 1/n₂))
```

---

## 5. Common Traps

| Trap | What goes wrong | Correct approach |
|------|----------------|-----------------|
| **"p < 0.05 → effect is real and important"** | Significance ≠ practical significance. Large n makes tiny effects significant. | Always report effect size (d, η², r) alongside p. |
| **"p = 0.07 → no effect"** | Absence of evidence ≠ evidence of absence. You may just be underpowered. | Report power or CI; distinguish "no evidence of effect" from "evidence of no effect." |
| **Multiple t-tests instead of ANOVA** | Family-wise error rate explodes. | Use ANOVA; then Tukey/Bonferroni post-hoc. |
| **Misinterpreting the CI** | "95% probability the parameter is in this interval" — wrong. | The interval is random; the parameter is fixed. Correct: 95% of such intervals contain the true value. |
| **Choosing one-tailed test after seeing data** | Effectively doubles the Type I error rate. | Pre-register the direction before data collection. |
| **Applying paired t-test to independent data (or vice versa)** | Wrong SE, wrong df, wrong inference. | Confirm whether samples share subjects. |
| **Ignoring outliers in t-test** | A single extreme point inflates s, shrinks t, inflates p — may miss real effect. | Check; consider trimmed mean, Winsorization, or non-parametric test. |
| **Confusing correlation with causation** | Observational r does not establish directionality or ruling out confounders. | Use RCT, IV, or causal graph methods to approach causation. |
| **Forgetting expected-frequency assumption in chi-square** | Test is invalid if cells have E < 5. | Pool rare categories or use Fisher's Exact Test (2×2 with small n). |
| **Using mean to summarize skewed data** | Mean pulled by outliers; misleads. | Report median (and IQR); use median-based tests. |
| **Stopping an A/B test early because p < 0.05** | "Peeking problem" — sequential testing inflates α substantially. | Pre-specify n; use sequential testing methods (O'Brien-Fleming, alpha spending) if early stopping needed. |
| **Ignoring variance heterogeneity in ANOVA** | Standard F-test is biased when variances differ across groups. | Use Welch's ANOVA; use Games-Howell post-hoc. |
| **MLE with small n for complex models** | MLE overfits; no regularization. | Use MAP (prior = regularization), cross-validate, or use penalized likelihood. |

---

## References

1. Wasserstein, R. L., & Lazar, N. A. (2016). *The ASA Statement on p-Values.* The American Statistician. [https://www.tandfonline.com/doi/full/10.1080/00031305.2016.1154108](https://www.tandfonline.com/doi/full/10.1080/00031305.2016.1154108)
2. American Statistical Association (2016). *P-value Statement* (PDF). [https://www.amstat.org/asa/files/pdfs/p-valuestatement.pdf](https://www.amstat.org/asa/files/pdfs/p-valuestatement.pdf)
3. Delacre, M., Lakens, D., & Leys, C. (2017). *Why Psychologists Should by Default Use Welch's t-test.* IRSP. [https://rips-irsp.com/articles/10.5334/irsp.82](https://rips-irsp.com/articles/10.5334/irsp.82)
4. Sullivan, G. M., & Feinn, R. (2012). *Using Effect Size.* Journal of Graduate Medical Education. [https://pmc.ncbi.nlm.nih.gov/articles/PMC3444174/](https://pmc.ncbi.nlm.nih.gov/articles/PMC3444174/)
5. Kim, J., & Park, S. (2019). *Type I and Type II Errors and Statistical Power.* StatPearls/NCBI. [https://www.ncbi.nlm.nih.gov/books/NBK557530/](https://www.ncbi.nlm.nih.gov/books/NBK557530/)
6. Cohen, J. (1988). *Statistical Power Analysis for the Behavioral Sciences* (2nd ed.). [Referenced via statisticsbyjim.com](https://statisticsbyjim.com/basics/cohens-d/)
7. Scribbr. *Chi-Square Tests.* [https://www.scribbr.com/statistics/chi-square-tests/](https://www.scribbr.com/statistics/chi-square-tests/)
8. UC Berkeley Stat 159. *Duality: Confidence Sets and Hypothesis Tests.* [https://ucb-stat-159-s21.github.io/site/Notes/duality.html](https://ucb-stat-159-s21.github.io/site/Notes/duality.html)

---

*Part of the [Statistics for Data Science](README.md) section · [Guide Home](../README.md).*
