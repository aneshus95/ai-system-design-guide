# Hypothesis Testing

Hypothesis testing is a formal procedure for deciding whether sample data provide enough evidence to reject a claim about a population parameter. It structures the question — "could this pattern have arisen by chance?" — into a decision rule with quantified error rates.

> **In plain English:** You start by assuming nothing interesting is going on (the null hypothesis). You collect data, measure how surprising those data are *if the null were true*, and decide whether the surprise is large enough to declare the null implausible.

---

## Table of Contents

1. [The Framework](#1-the-framework)
2. [Test Statistic & Sampling Distribution Under H₀](#2-test-statistic--sampling-distribution-under-h)
3. [The p-Value — Correct Definition & Misinterpretations](#3-the-p-value--correct-definition--misinterpretations)
4. [Significance Level α](#4-significance-level-α)
5. [Type I and Type II Errors](#5-type-i-and-type-ii-errors)
6. [Statistical Power (1 − β)](#6-statistical-power-1--β)
7. [Effect Size vs Statistical Significance](#7-effect-size-vs-statistical-significance)
8. [Confidence Intervals vs Hypothesis Tests — The Duality](#8-confidence-intervals-vs-hypothesis-tests--the-duality)
9. [Major Tests: Assumptions, Formulas, When to Use](#9-major-tests-assumptions-formulas-when-to-use)
   - 9.1 z-Test
   - 9.2 One-Sample t-Test
   - 9.3 Two-Sample (Independent) t-Test & Welch's t-Test
   - 9.4 Paired t-Test
   - 9.5 One-Way ANOVA & F-Test
   - 9.6 Two-Way ANOVA
   - 9.7 Post-Hoc Tests
   - 9.8 Chi-Square Tests
   - 9.9 Pearson Correlation Test
10. [Non-Parametric Alternatives](#10-non-parametric-alternatives)
11. [Checking Assumptions](#11-checking-assumptions)
12. [Master Decision Table](#12-master-decision-table)
13. [🎯 In the Interview](#-in-the-interview)
14. [Glossary](#glossary)
15. [References](#references)

---

## 1. The Framework

### Null Hypothesis (H₀) and Alternative Hypothesis (H₁ / Hₐ)

| Element | Meaning |
|---------|---------|
| **H₀** | The "nothing is happening" claim; the one you attempt to *disprove*. E.g., μ = 0, μ₁ = μ₂, there is no association. |
| **H₁** | The claim you'd accept if H₀ is implausible given the data. |

You never "prove" H₁; you merely gather evidence against H₀.

### One-Tailed vs Two-Tailed Tests

| Type | H₁ form | When to use |
|------|----------|-------------|
| **Two-tailed** | μ ≠ μ₀ | Direction of effect unknown; most common default. |
| **Right-tailed** | μ > μ₀ | You predict the parameter is *larger*. |
| **Left-tailed** | μ < μ₀ | You predict the parameter is *smaller*. |

> **Why / When to use / Nuances:** Prefer two-tailed unless the direction is pre-specified by domain theory before data collection. A one-tailed test is more powerful in the predicted direction but completely misses effects in the opposite direction, and choosing the tail *after* seeing the data inflates the Type I error rate.

**Mini-example:** An e-commerce team runs an A/B test on button color. They don't know whether the new color will help or hurt conversions → two-tailed (H₁: conversion rate_new ≠ conversion rate_old). If they had strong prior evidence the change could only improve clicks, a one-tailed test might be argued, but two-tailed is safer.

---

## 2. Test Statistic & Sampling Distribution Under H₀

A **test statistic** compresses the data into a single number measuring how far the observed result is from what H₀ predicts, scaled by sampling variability:

```
test statistic = (observed estimate − null value) / standard error of the estimate
```

Under H₀, the test statistic follows a known reference distribution (z, t, F, χ²). The tail probability of the observed statistic under that distribution is the p-value.

**Why scaling by SE matters:** A mean difference of 5 points is hugely significant if SE = 0.1 (t ≈ 50) and totally unremarkable if SE = 10 (t = 0.5).

---

## 3. The p-Value — Correct Definition & Misinterpretations

### Correct Definition

> The **p-value** is the probability, under the assumed statistical model (including H₀), of obtaining a test statistic at least as extreme as the one actually observed. ([ASA Statement, 2016](https://www.amstat.org/asa/files/pdfs/p-valuestatement.pdf))

Formally: **p = P(T ≥ t_obs | H₀ true)** for a one-tailed test; **p = P(|T| ≥ |t_obs| | H₀ true)** for two-tailed.

### Common Misinterpretations (Memorize These)

| Myth | Truth |
|------|-------|
| "p = 0.03 means there is a 3% chance H₀ is true." | p is about the data given H₀; it says nothing about P(H₀ is true). That requires a prior (Bayesian territory). |
| "p < 0.05 means the result is important or the effect is large." | p conflates effect size with sample size. A tiny, practically irrelevant effect can yield p < 0.0001 with large n. |
| "p = 0.06 means there is no effect." | Failing to reject H₀ does not prove H₀. It may just reflect insufficient power. |
| "p is the probability the result occurred by chance." | p assumes H₀ is true; it is not a probability over outcomes in an unconditional sense. |

The ASA (2016) explicitly states: *"a p-value does not measure the size of an effect or the importance of a result."* ([source](https://www.tandfonline.com/doi/full/10.1080/00031305.2016.1154108))

---

## 4. Significance Level α

**α** (alpha) is the pre-specified maximum acceptable probability of a **Type I error** (false positive). Common choices:

| α | Usage context |
|---|---------------|
| 0.05 | Social science, A/B testing (conventional default) |
| 0.01 | Medical, stricter evidence threshold |
| 0.001 | Physics, genomics (genome-wide significance ≈ 5×10⁻⁸) |

The decision rule: **reject H₀ if p < α**.

α must be set *before* data collection to be valid. Choosing α after seeing the p-value is "p-hacking."

---

## 5. Type I and Type II Errors

|  | H₀ is actually TRUE | H₀ is actually FALSE |
|--|---------------------|----------------------|
| **Reject H₀** | **Type I error (α)** — false positive | Correct — true positive (power = 1 − β) |
| **Fail to reject H₀** | Correct — true negative | **Type II error (β)** — false negative |

- **Type I error rate = α** (controlled directly by your threshold).
- **Type II error rate = β** (depends on effect size, n, variance, α).
- Lowering α (e.g., 0.05 → 0.01) reduces Type I errors but *increases* β (fewer true effects detected).

**Mini-example (medical testing):** H₀ = drug has no effect.
- Type I: declaring the drug works when it doesn't → wasteful, possibly harmful.
- Type II: missing a drug that actually works → also harmful.
The acceptable ratio of these depends on domain costs (pre-set by study design, not chosen post-hoc).

---

## 6. Statistical Power (1 − β)

**Power** = P(reject H₀ | H₁ is true) = probability of detecting a real effect.

The conventional minimum target is **power ≥ 0.80** (i.e., β ≤ 0.20). ([StatPearls, NCBI](https://www.ncbi.nlm.nih.gov/books/NBK557530/))

### What Drives Power?

| Driver | Direction | Intuition |
|--------|-----------|-----------|
| **Effect size ↑** | Power ↑ | Larger signal is easier to detect. |
| **Sample size n ↑** | Power ↑ | More data → smaller SE → test statistic grows. |
| **α ↑** (less strict) | Power ↑ | Widening the rejection zone catches more true effects (but also more false positives). |
| **Population variance ↓** | Power ↑ | Less noise → cleaner signal. |
| **One-tailed vs two-tailed** | One-tailed ↑ | Concentrates rejection in predicted direction. |

**Power analysis** (a priori): given a target power (0.80), α, and expected effect size, solve for the required n. Done in software (G*Power, R's `pwr` package, Python `statsmodels`).

**Mini-example:** A/B test on click-through rate. Current CTR = 5%, expected lift = 1pp (absolute). With α = 0.05, power = 0.80, you need roughly 4,400 users per arm. If you only run 500 per arm, power drops to ~0.25 — you'll miss the effect 75% of the time even if it's real.

---

## 7. Effect Size vs Statistical Significance

Statistical significance tells you whether an effect is *detectable*. Effect size tells you whether it is *meaningful*.

### Cohen's d (for means comparison)

```
d = (μ₁ − μ₂) / s_pooled

where s_pooled = sqrt[((n₁−1)s₁² + (n₂−1)s₂²) / (n₁+n₂−2)]
```

Cohen's conventional benchmarks ([Cohen 1988](https://statisticsbyjim.com/basics/cohens-d/)):

| d | Interpretation |
|---|----------------|
| 0.2 | Small |
| 0.5 | Medium |
| 0.8 | Large |

### Other Effect Size Measures

| Measure | Used for | Formula / Note |
|---------|----------|----------------|
| **Cohen's d** | Two-group mean difference | Pooled-SD scaled |
| **η² (eta-squared)** | ANOVA | SS_between / SS_total |
| **ω² (omega-squared)** | ANOVA (less biased than η²) | Corrected for sampling bias |
| **r (Pearson r)** | Correlation | −1 to +1; r² = variance explained |
| **Cramér's V** | Chi-square (categorical) | √(χ²/(n·min(r−1, c−1))) |
| **Odds Ratio / Risk Ratio** | Binary outcomes, medical | Ratio of odds or probabilities |

> **Why / When to use / Nuances:** Always report effect sizes alongside p-values. A large n can make a d = 0.02 "statistically significant" while being practically negligible. Conversely, d = 1.2 can be non-significant with n = 10. The effect size is the quantity that matters for decisions. ([PMC article on effect size](https://pmc.ncbi.nlm.nih.gov/articles/PMC3444174/))

**Mini-example:** An e-learning platform improves test scores by 0.4 points on a 100-point scale (d = 0.02). With 50,000 students, p < 0.001. The result is statistically significant but educationally irrelevant.

---

## 8. Confidence Intervals vs Hypothesis Tests — The Duality

A **95% confidence interval** for a parameter θ contains all values θ₀ that would *not* be rejected by a two-tailed α = 0.05 test. This is the **duality theorem**:

> **Reject H₀: θ = θ₀ at level α  ⟺  θ₀ lies outside the (1 − α) CI.**

([UC Berkeley duality notes](https://ucb-stat-159-s21.github.io/site/Notes/duality.html))

**Practical advantages of CIs over p-values alone:**

1. **Direction and magnitude** visible at a glance (CI: "effect is 2–8 units"; p-value: "p = 0.01" — which direction?).
2. **Practical significance** assessable: does the CI exclude the *minimum practically meaningful* effect?
3. Easier to combine across studies (meta-analysis uses CIs).
4. **Avoids the binary trap:** a CI entirely above zero is more informative than just "p < 0.05."

**Warning:** A 95% CI does *not* mean "there is a 95% probability that θ lies in this interval." The parameter is fixed; the interval is random. The correct interpretation: if you repeated the study many times, 95% of the constructed intervals would contain the true θ.

---

## 9. Major Tests: Assumptions, Formulas, When to Use

### 9.1 z-Test

**When:** Population variance σ² is *known* (rare in practice), **or** n is large enough (n ≥ 30 by CLT convention) that s ≈ σ. Testing a proportion with large n also uses z.

**Test statistic:**
```
z = (x̄ − μ₀) / (σ / √n)     [one-sample, known σ]
z = (p̂ − p₀) / √(p₀(1−p₀)/n)  [proportion test]
```

**Assumptions:**
- Random sampling / independence.
- σ known, or n large (CLT applies).
- For proportion: np₀ ≥ 5 and n(1−p₀) ≥ 5.

**Distribution under H₀:** Standard Normal N(0,1).

> **Why / When to use:** In practice the t-test subsumes the z-test (t → z as n → ∞). Use z explicitly when testing proportions or when σ is truly known (e.g., quality-control processes with historical variance).

---

### 9.2 One-Sample t-Test

**When:** Comparing a sample mean to a known/hypothesized value; σ unknown; n can be small if data are approximately normal.

**Test statistic:**
```
t = (x̄ − μ₀) / (s / √n)      df = n − 1
```

**Assumptions:**
- Independence of observations.
- Approximately normal distribution **or** n ≥ 30 (CLT).
- No severe outliers (can distort s).

**Mini-example:** A coffee shop claims its espresso shots are 30 mL on average. A quality auditor measures 20 shots: x̄ = 28 mL, s = 3 mL. t = (28 − 30)/(3/√20) = −2.98, df = 19. Compare to t-critical at α = 0.05 two-tailed ≈ ±2.09 → reject H₀.

---

### 9.3 Two-Sample (Independent) t-Test & Welch's t-Test

**When:** Comparing means of two independent groups.

**Student's t (equal variances assumed):**
```
t = (x̄₁ − x̄₂) / (s_p · √(1/n₁ + 1/n₂))

s_p = √[((n₁−1)s₁² + (n₂−1)s₂²) / (n₁+n₂−2)]
df = n₁ + n₂ − 2
```

**Welch's t (unequal variances — preferred default):**
```
t = (x̄₁ − x̄₂) / √(s₁²/n₁ + s₂²/n₂)
df = Welch-Satterthwaite approximation (non-integer)
```

> **Why / When to use:** Welch's t-test is safer as a default. A 2017 paper in the *International Review of Social Psychology* argues researchers should "by default use Welch's t-test instead of Student's t-test" because Welch's performs equivalently when variances are equal, and much better when they are not. ([Delacre et al., 2017](https://rips-irsp.com/articles/10.5334/irsp.82)) Using Levene's test first to *decide* which t to use is itself problematic (low power of Levene's at small n).

**Assumptions (both):** Independence within and between groups; approximate normality or large n; for Student's only: equal variances (homoscedasticity).

---

### 9.4 Paired t-Test

**When:** Two measurements on the *same subjects* (before/after, matched pairs). Equivalently, a one-sample t-test on the differences dᵢ = x₁ᵢ − x₂ᵢ.

```
t = d̄ / (s_d / √n)      df = n − 1
```

**Assumptions:** Pairs are independent of each other; differences are approximately normally distributed.

**Mini-example:** Blood pressure measured before and after a drug in 15 patients. Because the same patient provides both readings, pairing removes between-subject variability → more powerful than an independent two-sample test.

> **Why / When to use:** Never apply an independent t-test to paired data — you ignore the correlation structure and lose power (or get wrong results if the correlation is negative).

---

### 9.5 One-Way ANOVA & F-Test

**When:** Comparing means across **k ≥ 3 independent groups** on one factor.

**Why not just run multiple t-tests?** With k groups there are k(k−1)/2 pairwise comparisons. At α = 0.05 each, family-wise error rate explodes: P(at least one Type I error) ≈ 1 − 0.95^C. For k = 5, that's ≈ 40%. ANOVA controls the family-wise error at α.

**F-statistic:**
```
F = MS_between / MS_within = (SS_between / df_between) / (SS_within / df_within)

df_between = k − 1
df_within  = N − k    (N = total observations)
```

**Assumptions:**
- Independence of observations.
- Approximate normality within each group (robust for large n by CLT).
- **Homogeneity of variance** across groups (check with Levene's test).

**Null hypothesis:** All group means are equal (μ₁ = μ₂ = … = μₖ).

**A significant F-test only tells you *some* means differ — not which pairs.**

**Mini-example:** Three diet plans, 30 participants each. ANOVA tests whether mean weight loss differs across the three diets. F(2, 87) = 5.3, p = 0.007 → reject H₀. Post-hoc required to identify which diets differ.

---

### 9.6 Two-Way ANOVA

**When:** Two categorical independent variables (factors). Tests:
1. Main effect of Factor A.
2. Main effect of Factor B.
3. **Interaction effect** (A × B): whether the effect of A depends on the level of B.

**Interaction is often the most interesting result** — e.g., Drug A works better than Drug B, but only for women.

**Additional assumptions:** Balanced design (equal cell sizes) preferred; same normality and homoscedasticity requirements.

---

### 9.7 Post-Hoc Tests

Run **after** a significant ANOVA omnibus F-test to identify *which* specific group pairs differ.

| Post-hoc test | When to prefer |
|---------------|---------------|
| **Tukey's HSD** | All pairwise comparisons, equal n, homogeneous variance — controls family-wise error rate efficiently. |
| **Bonferroni correction** | Few planned comparisons; divides α by number of tests (conservative, simple). |
| **Scheffé** | All possible contrasts (not just pairwise); most conservative. |
| **Games-Howell** | Unequal variances and/or unequal sample sizes. |
| **Dunnett** | Comparing each group to a single control (not all pairs). |

---

### 9.8 Chi-Square Tests

#### Chi-Square Goodness of Fit

**When:** One categorical variable; test whether observed frequencies match a hypothesized distribution.

```
χ² = Σ [(O_i − E_i)² / E_i]      df = k − 1
```

**Assumptions:**
- Observations independent.
- Expected frequency ≥ 5 in each cell (for 2 categories); for 3+ categories: all E_i ≥ 1 and ≤ 20% of cells have E_i < 5. ([Scribbr](https://www.scribbr.com/statistics/chi-square-tests/))

**Mini-example:** A die is rolled 120 times. Each face should appear 20 times (E = 20). Observed: {18, 22, 14, 26, 20, 20}. χ²(5) = 3.6, p = 0.61 → fail to reject — no evidence the die is unfair.

#### Chi-Square Test of Independence

**When:** Two categorical variables in a contingency table; test whether they are associated.

```
χ² = Σ Σ [(O_ij − E_ij)² / E_ij]      df = (r − 1)(c − 1)
E_ij = (row_i total × col_j total) / grand total
```

Same expected-frequency assumptions apply. Does **not** give direction or strength of association (use Cramér's V for effect size).

---

### 9.9 Pearson Correlation Test

**When:** Testing whether two continuous variables have a linear association (H₀: ρ = 0).

**Test statistic:**
```
t = r · √(n − 2) / √(1 − r²)      df = n − 2
```

**Assumptions:** Both variables approximately bivariate normal; no severe outliers; observations independent; relationship is linear.

**Mini-example:** r = 0.45, n = 30. t = 0.45·√28/√(1−0.2025) = 0.45·5.29/0.893 ≈ 2.67, df = 28. p ≈ 0.013 → statistically significant linear association.

> **Why / When to use:** Pearson r measures *linear* association only. Two variables can have strong non-linear association and r ≈ 0. Always plot a scatterplot first. For non-linear or non-normal data, use Spearman's ρ.

---

## 10. Non-Parametric Alternatives

Non-parametric tests make fewer distributional assumptions (typically: independence + ordinal/continuous scale). They operate on **ranks** rather than raw values.

| Parametric test | Non-parametric alternative | When to prefer non-parametric |
|-----------------|---------------------------|-------------------------------|
| One-sample t-test | **Wilcoxon signed-rank test** | Non-normal data, small n, ordinal outcome |
| Two-sample t-test | **Mann-Whitney U test** | Non-normal distributions, ordinal data, outliers |
| Paired t-test | **Wilcoxon signed-rank test** | Non-normal differences, ordinal scale |
| One-way ANOVA | **Kruskal-Wallis H test** | Non-normal groups, ordinal outcome, k ≥ 3 groups |
| Sign test (crude) | **Sign test** | Very small n; only direction of difference is reliable |
| Pearson correlation | **Spearman's ρ** | Non-linear monotonic association, ordinal, outliers |

> **Why / When to use / Nuances:** Non-parametric tests are *not* assumption-free — they still require independence. They test slightly different things (ranks, medians in some setups) vs parametric tests (means). When parametric assumptions hold, parametric tests are more powerful. With n ≥ 30 per group, the CLT often makes t-tests robust enough that non-parametric alternatives offer little advantage. ([Mann-Whitney U vs t-test comparison](https://statmate.org/blog/t-test-vs-mann-whitney))

**Mann-Whitney U interpretation note:** The test technically compares the probability P(X > Y) rather than medians per se; medians can be equal while the test rejects, if the distribution shapes differ.

**Mini-example:** Customer satisfaction scores (1–5 Likert) for two service types, n₁ = 12, n₂ = 14. Skewed, small n, ordinal → Mann-Whitney U preferred over two-sample t-test.

---

## 11. Checking Assumptions

### Normality

| Method | Type | Notes |
|--------|------|-------|
| **Shapiro-Wilk test** | Formal | Best for n < 50; H₀ = data are normal; sensitive to large n (trivially rejects with big samples) |
| **Kolmogorov-Smirnov (Lilliefors)** | Formal | Less powerful than Shapiro-Wilk for detecting non-normality |
| **Q-Q plot** | Graphical | Points should fall near the diagonal; practical and intuitive; preferred for large n |
| **Histogram / density plot** | Graphical | Quick visual check for severe skew or multimodality |

**Important:** With n ≥ 30–50 per group, the CLT makes most parametric tests robust to modest departures from normality. Normality tests have low power for small n (when it matters most) and trivially reject for large n (when it matters least).

### Equal Variances (Homoscedasticity)

| Method | Notes |
|--------|-------|
| **Levene's test** | H₀ = variances equal; robust to non-normality. A non-significant result does NOT prove equal variances — consider Welch's t as default anyway. |
| **Bartlett's test** | More powerful but sensitive to non-normality. |
| **Ratio of variances / boxplots** | Quick graphical check; if max/min variance ratio > 4–5, concern warranted. |

---

## 12. Master Decision Table

| Scenario | Outcome variable type | Groups / structure | Recommended test |
|----------|-----------------------|--------------------|-----------------|
| One sample vs known mean, σ known | Continuous | 1 sample | z-test |
| One sample vs known mean, σ unknown | Continuous | 1 sample | One-sample t-test |
| Compare two independent group means, n large or normal, variances equal | Continuous | 2 independent | Student's two-sample t-test |
| Compare two independent group means (default / unequal variance) | Continuous | 2 independent | **Welch's t-test** |
| Compare two related/paired measurements | Continuous | 2 paired | Paired t-test |
| Compare two independent groups, non-normal or ordinal | Ordinal/continuous | 2 independent | Mann-Whitney U |
| Compare two related groups, non-normal or ordinal | Ordinal/continuous | 2 paired | Wilcoxon signed-rank |
| Compare 3+ independent group means, normal, equal variance | Continuous | k ≥ 3 independent | One-way ANOVA + post-hoc |
| Compare 3+ independent group means, non-normal or ordinal | Ordinal/continuous | k ≥ 3 independent | Kruskal-Wallis |
| Compare 3+ groups with 2 factors; check interaction | Continuous | k×j factorial | Two-way ANOVA |
| Test proportion vs known value | Binary/proportion | 1 sample | z-test for proportions |
| Compare two proportions | Binary | 2 independent | z-test for two proportions / chi-square |
| Categorical variable fits hypothesized distribution | Categorical | 1 variable | Chi-square goodness of fit |
| Association between two categorical variables | Categorical | 2 variables | Chi-square test of independence |
| Linear association between two continuous variables (normal) | Continuous | 2 variables | Pearson correlation test |
| Monotonic association, non-normal or ordinal | Ordinal/continuous | 2 variables | Spearman's ρ |
| Before/after on same subjects, normal differences | Continuous | Paired | Paired t-test |
| Before/after on same subjects, non-normal differences | Ordinal/continuous | Paired | Wilcoxon signed-rank |
| Compare variances of two groups | Continuous | 2 groups | Levene's / F-test for equality of variances |
| ANOVA significant: which specific pairs differ? | Continuous | k ≥ 3 | Tukey's HSD / Bonferroni / Games-Howell |
| Test if all distributions identical across many groups (non-parametric) | Ordinal | k ≥ 3 | Kruskal-Wallis |
| Only direction (sign) of difference is reliable | Any | Paired | Sign test |

---

## 🎯 In the Interview

### Key Traps and How to Avoid Them

**1. "p < 0.05 means the result is important."**
WRONG. p-value conflates effect size with sample size. Always report effect size (Cohen's d, η², r) alongside p. With n = 100,000, even d = 0.01 gives p < 0.001.

**2. "p = 0.06 means there's no effect."**
WRONG. Failing to reject H₀ ≠ proving H₀. You may simply have insufficient power. The ASA (2016) explicitly warns against equating "p > 0.05" with "no effect."

**3. "The 95% CI means there's a 95% probability the true parameter is in that interval."**
WRONG. The parameter is fixed; the interval is random. Correct: 95% of intervals constructed this way from repeated samples would contain the true value.

**4. "I ran 10 t-tests across 10 metrics so I'd know if any changed."**
WRONG. This inflates family-wise Type I error. With 10 tests at α = 0.05, P(at least one false positive) ≈ 40%. Use ANOVA, or apply Bonferroni/Benjamini-Hochberg correction.

**5. "ANOVA significant → I know which groups differ."**
WRONG. ANOVA only tests the omnibus hypothesis. You need post-hoc tests (Tukey, Bonferroni, etc.) to identify specific pairs.

**6. "I should run Levene's test first, then decide between Student's and Welch's."**
DEBATABLE / risky. Levene's has low power at small n. Modern recommendation: **use Welch's by default**. It matches Student's when variances are equal and beats it when they're not.

**7. "Non-parametric tests are always safer."**
WRONG. They are less powerful when parametric assumptions hold, and they test slightly different hypotheses (ranks, not means).

**8. Correlation = causation.**
WRONG. Correlation measures linear co-movement; causation requires controlled assignment, temporal precedence, and ruling out confounders.

### Likely Interview Questions

- *"What is a p-value?"* → Give the ASA definition: probability of data at least this extreme if H₀ were true. Then proactively mention what it is NOT.
- *"When would you use a t-test vs z-test?"* → σ known + large n → z; σ unknown or small n → t.
- *"Why use ANOVA instead of multiple t-tests?"* → Family-wise error inflation; ANOVA controls it.
- *"What is statistical power and how do you increase it?"* → Increase n, increase effect size, increase α, reduce variance, one-tailed (if justified).
- *"When do you use non-parametric tests?"* → Non-normal + small n, ordinal data, severe outliers, or when median (not mean) is the right estimand.

---

## Glossary

| Term | Definition |
|------|-----------|
| **Null Hypothesis (H₀)** | The default claim of no effect or no difference. |
| **Alternative Hypothesis (H₁)** | The claim accepted if H₀ is rejected. |
| **p-value** | P(data this extreme or more | H₀ true). NOT P(H₀ true). |
| **α (alpha)** | Pre-specified Type I error threshold. |
| **β (beta)** | Type II error rate; P(fail to reject | H₁ true). |
| **Power (1−β)** | P(correctly reject H₀ | H₁ true). |
| **Effect size** | Standardized magnitude of difference or association (d, η², r, V). |
| **Test statistic** | Numerical summary of data relative to H₀ (t, z, F, χ²). |
| **Degrees of freedom** | Number of independent pieces of information; parameter of the reference distribution. |
| **Family-wise error rate** | Probability of at least one false positive across multiple tests. |
| **Bonferroni correction** | Divide α by number of tests; simple but conservative. |
| **Cohen's d** | Standardized mean difference: (μ₁−μ₂)/s_pooled. |
| **Confidence interval** | Interval estimate capturing true parameter with (1−α)×100% frequency across repeated samples. |
| **Homoscedasticity** | Equal variances across groups. |
| **CLT** | Central Limit Theorem: sample means → Normal for large n regardless of population distribution. |

---

## References

1. Wasserstein, R. L., & Lazar, N. A. (2016). *The ASA Statement on p-Values: Context, Process, and Purpose.* The American Statistician. [https://www.tandfonline.com/doi/full/10.1080/00031305.2016.1154108](https://www.tandfonline.com/doi/full/10.1080/00031305.2016.1154108)
2. American Statistical Association (2016). *Statement on Statistical Significance and P-Values* (PDF). [https://www.amstat.org/asa/files/pdfs/p-valuestatement.pdf](https://www.amstat.org/asa/files/pdfs/p-valuestatement.pdf)
3. Delacre, M., Lakens, D., & Leys, C. (2017). *Why Psychologists Should by Default Use Welch's t-test Instead of Student's t-test.* International Review of Social Psychology. [https://rips-irsp.com/articles/10.5334/irsp.82](https://rips-irsp.com/articles/10.5334/irsp.82)
4. Sullivan, G. M., & Feinn, R. (2012). *Using Effect Size—or Why the P Value Is Not Enough.* Journal of Graduate Medical Education. [https://pmc.ncbi.nlm.nih.gov/articles/PMC3444174/](https://pmc.ncbi.nlm.nih.gov/articles/PMC3444174/)
5. Kim, J., & Park, S. (2019). *Type I and Type II Errors and Statistical Power.* StatPearls. [https://www.ncbi.nlm.nih.gov/books/NBK557530/](https://www.ncbi.nlm.nih.gov/books/NBK557530/)
6. Cohen, J. (1988). *Statistical Power Analysis for the Behavioral Sciences* (2nd ed.). Erlbaum. Referenced via [https://statisticsbyjim.com/basics/cohens-d/](https://statisticsbyjim.com/basics/cohens-d/)
7. UC Berkeley Stat 159 (2021). *Duality between confidence sets and hypothesis tests.* [https://ucb-stat-159-s21.github.io/site/Notes/duality.html](https://ucb-stat-159-s21.github.io/site/Notes/duality.html)
8. Scribbr. *Chi-Square Tests.* [https://www.scribbr.com/statistics/chi-square-tests/](https://www.scribbr.com/statistics/chi-square-tests/)
9. Statistics Solutions. *Mann-Whitney U Test.* [https://www.statisticssolutions.com/free-resources/directory-of-statistical-analyses/mann-whitney-u-test/](https://www.statisticssolutions.com/free-resources/directory-of-statistical-analyses/mann-whitney-u-test/)

---

*Part of the [Statistics for Data Science](README.md) section · [Guide Home](../README.md).*
