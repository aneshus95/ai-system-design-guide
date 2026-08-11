# 10 — Resampling, Bootstrap, and Monte Carlo

When the assumptions underlying parametric tests (normality, known variance, closed-form sampling distributions) are uncertain or violated, **resampling methods** let the data speak for themselves. They generate the reference distribution empirically — by repeatedly drawing from the observed sample or from a null model — rather than relying on mathematical approximations. These methods are foundational for modern data science: bootstrap underpins most ML uncertainty estimates, permutation tests power A/B test analyses, and Monte Carlo simulation is the engine behind Bayesian computation, risk models, and games.

> **In plain English:** Instead of looking up a t-table and hoping your data are normal, resampling methods say: "I'll simulate what the world looks like by shuffling or resampling my data thousands of times, and build the reference distribution myself."

## Table of Contents

1. [Why Resampling?](#1-why-resampling)
2. [The Bootstrap](#2-the-bootstrap)
3. [Bootstrap Confidence Intervals](#3-bootstrap-confidence-intervals)
4. [When Bootstrap Fails](#4-when-bootstrap-fails)
5. [The Jackknife](#5-the-jackknife)
6. [Permutation / Randomisation Tests](#6-permutation--randomisation-tests)
7. [Bootstrap vs Permutation Tests](#7-bootstrap-vs-permutation-tests)
8. [Cross-Validation as Resampling](#8-cross-validation-as-resampling)
9. [Monte Carlo Methods](#9-monte-carlo-methods)
10. [🎯 In the Interview](#10--in-the-interview)
11. [Glossary](#glossary)
12. [References](#references)

---

## 1. Why Resampling?

Parametric inference works beautifully when distributional assumptions hold. In practice:
- Your sample may be small (n < 30).
- The statistic of interest (e.g. the median, a correlation, an AUC, the maximum) has no simple closed-form sampling distribution.
- Your data may be heavy-tailed, skewed, or censored.
- The Central Limit Theorem may not have kicked in yet.

**Resampling** creates new samples from the one observed sample to estimate the sampling distribution of any statistic empirically — no distributional assumptions required. ([Wikipedia — Resampling (statistics)](https://en.wikipedia.org/wiki/Resampling_(statistics)))

The three core families:

| Method | Null hypothesis? | Sampling mechanism | Main use |
|---|---|---|---|
| Bootstrap | No (estimates variability) | Resample with replacement from observed data | SEs, CIs, bias estimation |
| Jackknife | No | Leave-one-out systematic subsamples | Bias/variance of estimators |
| Permutation test | Yes (breaks group label) | Shuffle labels among observations | Hypothesis testing |

---

## 2. The Bootstrap

Introduced by Bradley Efron in 1979, the bootstrap estimates the sampling distribution of any statistic T(X) by treating the empirical distribution of the sample as a stand-in for the true population distribution.

### Procedure (Nonparametric Bootstrap)

1. From observed sample X = {x₁, ..., xₙ}, draw a **bootstrap sample** X* of size n **with replacement**.
2. Compute the statistic of interest: T* = T(X*).
3. Repeat steps 1–2 **B times** (typically B = 1,000–10,000).
4. The empirical distribution of {T*₁, T*₂, ..., T*_B} approximates the sampling distribution of T.

### Bootstrap Standard Error

```
SE_boot = std({T*₁, ..., T*_B})
```

This is consistent for most statistics (under regularity conditions) and requires no formula for the SE of T.

### Worked Example: Median Income

You have n = 50 salary observations. The median is $72,000. What is the SE of this estimate?

- Draw B = 5,000 bootstrap samples of size 50 with replacement.
- Compute the median of each → {m₁*, m₂*, ..., m₅₀₀₀*}.
- SE_boot = standard deviation of those 5,000 medians ≈ $3,200.
- 95% CI via percentile method: [2.5th percentile of medians, 97.5th percentile] ≈ [$65,900, $78,400].

Source: [The Statistical Bootstrap and Other Resampling Methods — Burns Statistics](https://www.burns-stat.com/documents/tutorials/the-statistical-bootstrap-and-other-resampling-methods-2/)

---

## 3. Bootstrap Confidence Intervals

Several methods exist for constructing CIs from bootstrap samples, with different accuracy properties.

### Method 1 — Percentile Interval

```
CI = [T*_(α/2), T*_(1-α/2)]
```

The simplest approach: use the 2.5th and 97.5th percentiles of the bootstrap distribution directly. Assumes the bootstrap distribution is an unbiased, symmetric reflection of the sampling distribution. Works well for symmetric statistics.

### Method 2 — Basic (Reverse Percentile) Interval

```
CI = [2×T_obs − T*_(1-α/2), 2×T_obs − T*_(α/2)]
```

Corrects for the direction of the bias. Slightly more accurate than the simple percentile method.

### Method 3 — BCa (Bias-Corrected and Accelerated)

The **BCa interval** is the gold standard. It corrects for both:
- **Bias (z₀):** whether the bootstrap distribution is centred on the observed statistic.
- **Acceleration (a):** skewness of the bootstrap distribution (estimated via jackknife).

It achieves second-order accuracy — error O(n⁻¹) vs O(n⁻¹/²) for the percentile method.

Implementation note: z₀ = Φ⁻¹(proportion of bootstrap values < T_obs). The acceleration a is typically estimated from jackknife replicates.

Source: [The Bias-Corrected and Accelerated Bootstrap Interval — SAS DO Loop](https://blogs.sas.com/content/iml/2017/07/12/bootstrap-bca-interval.html)

### Comparison

| Method | Accuracy | Complexity | Notes |
|---|---|---|---|
| Percentile | O(n⁻¹/²) | Simple | Adequate for symmetric, unbiased statistics |
| Basic (reverse) | O(n⁻¹/²) | Simple | Better direction correction |
| BCa | O(n⁻¹) | Moderate | Best in practice; handles skew and bias |
| Studentised (bootstrap-t) | O(n⁻²) | Complex | Highest accuracy but needs nested bootstrap for SE |

> **When to prefer BCa:** Whenever the statistic is skewed, the sample is moderate-sized, or precision matters (e.g. reporting CIs for published results). BCa is the default in most statistical software (R `boot` package, Python `scipy.stats.bootstrap`).

---

## 4. When Bootstrap Fails

The bootstrap is remarkably general but has known failure modes:

| Failure Mode | Why It Fails | What to Do Instead |
|---|---|---|
| **Extreme statistics (min, max, extreme quantiles)** | The empirical distribution puts zero mass beyond the observed range; bootstrap cannot extrapolate | Parametric modelling of extremes (EVT); smoothed bootstrap |
| **Very small n** | With n < ~15, the empirical CDF is a poor approximation of the population CDF | Parametric bootstrap if distributional form is known; exact methods |
| **Heavy-tailed distributions** | When the population variance is infinite, bootstrap SEs are inconsistent | Subsampling (draw m << n without replacement); trimmed statistics |
| **Dependent data (time series)** | IID resampling breaks temporal structure | Block bootstrap (draw consecutive blocks); stationary bootstrap; circular block bootstrap |
| **Highly structured data (hierarchical)** | Resampling units without respecting nesting structure | Cluster bootstrap (resample clusters, not observations) |
| **Sparse categorical data** | Some categories may vanish in bootstrap samples | Stratified bootstrap |

> **Key interview insight:** Bootstrap fails for the **maximum** (and minimum). The bootstrap distribution of the max is always ≤ the observed max, so it systematically underestimates the sampling variability of the max. This is a classic gotcha.

---

## 5. The Jackknife

The **jackknife** is an older resampling method (Quenouille 1949, Tukey 1958) that generates n leave-one-out samples.

### Procedure

1. For i = 1 to n, compute T₍₋ᵢ₎ = T applied to the data with observation i removed.
2. Jackknife SE: SE_jack = √[(n-1)/n × Σᵢ(T₍₋ᵢ₎ − T̄_jack)²], where T̄_jack = mean of all n leave-one-out estimates.
3. Jackknife bias estimate: bias_jack = (n−1)(T̄_jack − T_obs).

### Relationship to Bootstrap

- Jackknife is a **linear approximation** to the bootstrap — it explores local perturbations of the data.
- Bootstrap is generally more accurate for nonlinear statistics and non-smooth estimators.
- Jackknife fails for statistics that are not smooth functions of the data (e.g. the median for small samples, the max).
- Jackknife is used internally to estimate the acceleration parameter a in BCa intervals.

### When to Use Jackknife

- Legacy software contexts.
- Estimating the acceleration in BCa.
- When bootstrap replicates are expensive to compute and n is moderate.

---

## 6. Permutation / Randomisation Tests

A **permutation test** constructs the exact null distribution by enumerating (or randomly sampling) all possible reorderings of the data under the null hypothesis.

### Core Idea

Under H₀ (e.g. two groups have identical distributions), group labels are **exchangeable** — randomly shuffling which observations belong to group A vs group B should produce statistics similar to what we observed. If the observed statistic is extreme relative to the permutation distribution, we reject H₀.

### Procedure

1. Compute the observed test statistic T_obs (e.g. difference in group means).
2. Repeat B times:
   a. Randomly permute the group labels.
   b. Recompute the test statistic T*_b.
3. p-value = proportion of T*_b ≥ T_obs (for a one-sided test).

### Worked Example: A/B Test

You test two landing page variants. Conversion counts:
- Group A (n=200): 16 conversions → rate 8.0%
- Group B (n=200): 24 conversions → rate 12.0%
- Observed difference: 4.0 percentage points

Permutation test:
- Combine all 400 observations (32 conversions, 368 non-conversions).
- Randomly split into two groups of 200, ten thousand times.
- Compute the difference in rates each time.
- p-value = fraction of permutations with |difference| ≥ 4.0%.

If that fraction is 0.04, you reject H₀ at the 5% level. **No assumptions about normality or equal variance required.**

### When Permutation Tests Excel

| Situation | Why Permutation Wins |
|---|---|
| Small samples | Exact test; no CLT needed |
| Non-normal data | Assumption-free |
| Complex test statistics | Works for any statistic (AUC, correlation, F-statistic) |
| A/B testing with binary outcomes | Exact null vs approximate chi-squared |
| When parametric assumptions are dubious | Direct answer from data |

> **Exact vs approximate permutation tests:** With n observations split into two groups of n/2, there are C(n, n/2) possible permutations. For n=20 that is ~184,756 — enumerable. For larger n, random sampling of B permutations (Monte Carlo permutation test) gives a precise enough p-value.

---

## 7. Bootstrap vs Permutation Tests

> **When to use which:**

| Dimension | Bootstrap | Permutation Test |
|---|---|---|
| **Goal** | Estimate variability / build CIs | Hypothesis testing under a specific null |
| **Null hypothesis** | Not required | Required (defines what to permute) |
| **Sampling** | With replacement from original data | Without replacement, shuffling labels |
| **Output** | SE, CI, bias | p-value |
| **Assumptions** | IID observations (roughly) | Exchangeability under H₀ |
| **Works for** | Any statistic; CIs for complex estimators | Comparing groups; testing independence |

**Rule of thumb:** Use permutation tests when you want a p-value for a comparison. Use bootstrap when you want confidence intervals or uncertainty estimates for an estimator.

---

## 8. Cross-Validation as Resampling

Cross-validation is a resampling technique for estimating **out-of-sample predictive performance** — the expected error on data the model has not seen.

### k-Fold Cross-Validation

1. Randomly partition data into k folds of equal size.
2. For each fold i:
   - Train on the other k−1 folds.
   - Evaluate on fold i.
3. Average the k evaluation scores.

**Why k=5 or k=10?** These values balance bias (more folds → less bias, as training set is larger) and variance (more folds → more variance in fold-level estimates, and more compute). k=10 is the empirical default.

### Leave-One-Out CV (LOOCV)

k = n. Each observation is held out once. Low bias, high variance, computationally expensive (though some models like ridge regression have closed-form LOOCV). Approximately equivalent to the jackknife for many purposes.

### Stratified k-Fold

Ensures each fold has approximately the same class distribution as the full dataset. **Always use this for classification problems with class imbalance.**

### Nested Cross-Validation

Used when both model selection (hyperparameter tuning) and performance estimation are needed on the same dataset.

```
Outer loop (performance estimation):
  For each outer fold:
    Inner loop (hyperparameter selection):
      For each inner fold:
        Train / validate
    Select best hyperparameters using inner CV
    Evaluate on outer test fold with best hyperparameters
Average outer fold scores = unbiased performance estimate
```

Without the outer loop, selecting hyperparameters and reporting CV score on the same splits produces optimistically biased estimates.

### Pitfalls

| Pitfall | Description | Fix |
|---|---|---|
| **Data leakage** | Preprocessing (scaling, feature selection, SMOTE) applied before splitting | Apply all preprocessing inside the CV loop, fitted only on training folds |
| **Non-IID / time series** | Random shuffling breaks temporal dependencies | Use **time-series CV** (expanding or sliding window); always train on past, test on future |
| **Dependent observations** | Spatial data, repeated measures, clustered data | **Group k-fold**: never split a group across train and test |
| **Blocked CV for time series** | Standard k-fold leaks future into past | Use `TimeSeriesSplit` (sklearn) or blocked CV |
| **Small dataset with many classes** | Folds may miss rare classes | Stratified k-fold or LOOCV |

> **Leakage is the most common CV pitfall in interviews.** The canonical example: "I scaled all my features and then did cross-validation. Test accuracy is 97%." The scaler was fit on the whole dataset including the test fold, leaking information. Correct approach: fit the scaler only on the training fold inside each CV iteration.

---

## 9. Monte Carlo Methods

**Monte Carlo (MC) methods** estimate quantities that are hard to compute analytically by drawing **random samples** and computing empirical averages.

### Core Principle

If you want to estimate E[f(X)] where X ~ P(X), and the integral ∫ f(x) P(x) dx is intractable:

```
E[f(X)] ≈ (1/B) × Σᵢ f(Xᵢ),   where Xᵢ ~ P(X)
```

By the law of large numbers, this converges to the true expectation as B → ∞.

### Worked Example: Estimating π

Draw n random points (x, y) uniformly in the unit square [0,1]². Count how many fall inside the unit circle (x² + y² ≤ 1). The fraction ≈ π/4, so π ≈ 4 × (count inside circle) / n.

With n=10,000, you typically get π accurate to about 2 decimal places.

### Variance of MC Estimates

```
Var[MC estimate] = Var[f(X)] / B
SE[MC estimate] = std[f(X)] / √B
```

To halve the error, quadruple the number of samples. Convergence is O(B⁻¹/²) — slow but independent of problem dimension (no curse of dimensionality for MC).

### Importance Sampling — Intuition

When the region of interest (e.g. rare events) has very low probability under P(X), naive MC is inefficient — most samples contribute nothing useful. **Importance sampling** draws from an alternative distribution Q(X) that over-represents the important region:

```
E_P[f(X)] = E_Q[f(X) × P(X)/Q(X)]
           ≈ (1/B) Σᵢ f(Xᵢ) × w(Xᵢ),   where Xᵢ ~ Q, w = P/Q
```

The weights w(Xᵢ) = P(Xᵢ)/Q(Xᵢ) correct for the fact that we're sampling from the wrong distribution. Well-chosen Q can dramatically reduce variance.

**Example:** Estimating the probability of a 5-sigma event by sampling from N(5, 1) instead of N(0, 1), then reweighting.

### Monte Carlo in Practice

| Application | How MC is Used |
|---|---|
| **Risk / finance** | Simulate thousands of asset price paths; estimate portfolio VaR |
| **Bayesian MCMC** | Sample from intractable posteriors |
| **Permutation tests** | Random permutations when full enumeration is impossible |
| **Cross-validation** | Monte Carlo CV: random train/test splits (repeated random subsampling) |
| **Hyperparameter search** | Random search ≈ MC sampling of hyperparameter space |
| **Numerical integration** | When the integrand is complex or high-dimensional |
| **Game AI / operations research** | Monte Carlo Tree Search (MCTS) |

### Monte Carlo CV (vs k-Fold)

**Monte Carlo CV** (also called repeated random subsampling): randomly split the data into train/test many times, rather than the structured k-fold approach. Advantages: test sets can overlap; number of splits is decoupled from data size. Disadvantage: some observations may never appear in a test set. k-fold CV guarantees each observation is tested exactly once per repeat.

---

## 10. 🎯 In the Interview

### Common Traps

**Trap 1 — Bootstrap for the maximum:**
> "I'd use bootstrap to get a CI for the maximum value in my dataset."

**Correct answer:** Bootstrap fails for the maximum (and minimum). The bootstrap distribution of the max is bounded above by the observed max, so it systematically underestimates the variability. Use parametric extreme value theory or subsampling instead.

**Trap 2 — Scaling before CV:**
> "I standardise all features and then do 10-fold CV."

**Correct answer:** Data leakage. The scaler has seen all data including what will become the test fold. Fit the scaler inside the loop, on training folds only. This applies to any preprocessing: imputation, feature selection, SMOTE oversampling, PCA.

**Trap 3 — Permutation test exchangeability:**
> "I can use a permutation test for any dataset."

**Correct answer:** Permutation tests require exchangeability under H₀. For paired data, permute within pairs, not globally. For hierarchical / clustered data, permute at the cluster level. For time series, temporal permutation breaks the dependence structure and produces a null distribution that is too conservative (if the data are positively autocorrelated, random permutations are less extreme than the actual null).

**Trap 4 — Bootstrap vs permutation confusion:**
> "I use bootstrap to compute my p-value."

**Correct answer:** Bootstrap estimates variability (SEs and CIs) but does not directly test a null hypothesis. Permutation tests generate the null distribution by shuffling under H₀ and produce a p-value. (You can also derive a p-value from a bootstrap CI by checking whether 0 is inside the CI, but this is indirect and less powerful than a direct permutation test.)

**Trap 5 — Time-series cross-validation:**
> "I'll use standard 5-fold CV for my stock price prediction model."

**Correct answer:** Standard k-fold randomly shuffles, which leaks future data into training. Use `TimeSeriesSplit`: always train on past observations and test on strictly future observations. Walk-forward or expanding-window CV.

### Quick Reference: Which Method for Which Goal?

| Goal | Method |
|---|---|
| SE of a complex estimator | Bootstrap |
| CI for a complex statistic | Bootstrap (BCa preferred) |
| p-value for group comparison | Permutation test |
| Model generalisation error | k-fold CV (stratified for classification) |
| Hyperparameter selection + performance estimate | Nested CV |
| Estimating a probability / integral | Monte Carlo simulation |
| Rare event probability | Importance sampling MC |

---

## Glossary

| Term | Definition |
|---|---|
| **Bootstrap** | Resampling with replacement from the observed data to estimate the sampling distribution |
| **Bootstrap SE** | Standard deviation of the bootstrap distribution of a statistic |
| **BCa interval** | Bias-corrected and accelerated bootstrap CI; second-order accurate |
| **Jackknife** | Leave-one-out resampling; linear approximation to the bootstrap |
| **Permutation test** | Hypothesis test that generates the null distribution by permuting labels |
| **Exchangeability** | Property that the joint distribution is unchanged by permuting observations — required for permutation tests |
| **k-fold CV** | Data split into k folds; each used once as test set |
| **LOOCV** | Leave-one-out CV; k = n |
| **Stratified k-fold** | k-fold preserving class distribution in each fold |
| **Nested CV** | Outer loop for performance estimation; inner loop for hyperparameter selection |
| **Data leakage** | Information from the test set contaminating the training process |
| **Monte Carlo** | Estimating quantities by random simulation |
| **Importance sampling** | MC variant that samples from a proposal distribution and reweights |
| **Block bootstrap** | Bootstrap for dependent/time-series data; resamples blocks of consecutive observations |

---

## References

1. [Resampling (statistics) — Wikipedia](https://en.wikipedia.org/wiki/Resampling_(statistics))
2. [The Statistical Bootstrap and Other Resampling Methods — Burns Statistics](https://www.burns-stat.com/documents/tutorials/the-statistical-bootstrap-and-other-resampling-methods-2/)
3. [The Bias-Corrected and Accelerated Bootstrap Interval — SAS IML Blog](https://blogs.sas.com/content/iml/2017/07/12/bootstrap-bca-interval.html)
4. [Resampling and Monte Carlo Simulations — Duke Computational Statistics](https://people.duke.edu/~ccc14/sta-663-2016/15B_ResamplingAndSimulation.html)
5. [How Do Bootstrap and Permutation Tests Work? — ResearchGate](https://www.researchgate.net/publication/38348895_How_do_Bootstrap_and_permutation_tests_work)
6. [K-fold and Monte Carlo Cross-Validation vs Bootstrap — NIRPY Research](https://nirpyresearch.com/kfold-montecarlo-cross-validation-bootstrap-primer/)
7. [Lecture 24: Bootstrap Confidence Intervals — UW Madison](https://pages.stat.wisc.edu/~shao/stat710/stat710-24.pdf)

---

*Part of the [Statistics for Data Science](README.md) section · [Guide Home](../README.md).*
