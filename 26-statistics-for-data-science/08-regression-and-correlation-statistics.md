# Regression and Correlation Statistics

Regression is the workhorse of statistical inference in data science. This guide covers the statistical theory of simple and multiple linear regression, the classical assumptions that make OLS valid, diagnostic tools for detecting violations, and the extension to logistic regression—at the depth required for product DS and applied science interviews.

> **In plain English:** Regression fits a line (or surface) through your data and asks: "Is this slope statistically different from zero? How precisely can I estimate it? And can I trust the machinery that generated these standard errors?" The "inference" part—standard errors, t-tests, confidence intervals—only works correctly when the underlying assumptions hold.

## Table of Contents

1. [Simple and Multiple Linear Regression](#1-simple-and-multiple-linear-regression)
2. [OLS Assumptions and the Gauss-Markov Theorem](#2-ols-assumptions-and-the-gauss-markov-theorem)
3. [Interpreting Coefficients](#3-interpreting-coefficients)
4. [Inference: Standard Errors, t-tests, and Intervals](#4-inference-standard-errors-t-tests-and-intervals)
5. [R² and Adjusted R²](#5-r-and-adjusted-r)
6. [Multicollinearity and VIF](#6-multicollinearity-and-vif)
7. [Heteroscedasticity](#7-heteroscedasticity)
8. [Autocorrelation](#8-autocorrelation)
9. [Influential Points and Leverage](#9-influential-points-and-leverage)
10. [Omitted Variable Bias and Confounding](#10-omitted-variable-bias-and-confounding)
11. [Logistic Regression Statistics](#11-logistic-regression-statistics)
12. [Regularization's Effect on Inference](#12-regularizations-effect-on-inference)
13. [🎯 In the Interview](#-in-the-interview)
14. [Glossary](#glossary)
15. [References](#references)

---

## 1. Simple and Multiple Linear Regression

### 1.1 Simple Linear Regression

```
Y_i = β₀ + β₁ X_i + ε_i
```

- Y_i: observed outcome for unit i.
- X_i: observed predictor.
- β₀: intercept—expected value of Y when X = 0.
- β₁: slope—expected change in Y per unit increase in X.
- ε_i: error term (residual), capturing everything not explained by X.

**OLS estimators** minimize the sum of squared residuals Σ(Y_i − Ŷ_i)²:

```
β̂₁ = Cov(X, Y) / Var(X) = Σ(X_i − X̄)(Y_i − Ȳ) / Σ(X_i − X̄)²
β̂₀ = Ȳ − β̂₁ X̄
```

### 1.2 Multiple Linear Regression

```
Y = Xβ + ε      (matrix form)
```

where X is the (n × p) design matrix (including a column of 1s for the intercept), β is (p × 1), and ε ∼ N(0, σ²I) under classical assumptions.

OLS solution:

```
β̂ = (XᵀX)⁻¹Xᵀy
```

This exists as long as XᵀX is invertible (no perfect multicollinearity).

---

## 2. OLS Assumptions and the Gauss-Markov Theorem

The **Gauss-Markov theorem** states that under assumptions GM1–GM5, the OLS estimator β̂ is **BLUE** (Best Linear Unbiased Estimator): it has the smallest variance among all linear unbiased estimators. ([Statistics By Jim — Gauss-Markov & BLUE](https://statisticsbyjim.com/regression/gauss-markov-theorem-ols-blue/))

### The Five Core Assumptions

| # | Assumption | Violation name | Primary consequence |
|---|---|---|---|
| **GM1** | Linearity in parameters | Misspecification | Biased coefficient estimates |
| **GM2** | Random sampling / independence of errors | Autocorrelation | Underestimated SEs, inflated t-stats |
| **GM3** | No perfect multicollinearity | Singularity | (XᵀX)⁻¹ does not exist; cannot compute β̂ |
| **GM4** | Exogeneity: E[ε | X] = 0 | Endogeneity / OVB | Biased and inconsistent β̂ |
| **GM5** | Homoscedasticity: Var(ε | X) = σ² | Heteroscedasticity | OLS SEs are wrong (biased); inference invalid |

**For finite-sample t-tests and F-tests**, an additional assumption is required:

| **GM6** | Normality of errors: ε | X ∼ N(0, σ²I) | Non-normality in small samples | t-stats and p-values are approximate; CLT rescues large n |

### How to Check Each Assumption

**GM1 — Linearity:**
- Plot residuals vs. fitted values. Curvature indicates non-linearity.
- Add polynomial or interaction terms; use LOESS to explore functional form.

**GM2 — Independence:**
- Durbin-Watson test for serial correlation in time series (§8).
- For cross-sectional data: consider whether observations are clustered (family, school, firm) and use clustered SEs.

**GM3 — No perfect multicollinearity:**
- Check if any predictor is an exact linear combination of others (e.g., dummy trap: including both Male and Female dummies with an intercept).
- For near (imperfect) multicollinearity, use VIF (§6).

**GM4 — Exogeneity:**
- Not directly testable without an instrument. Assess qualitatively: is there a variable correlated with X that is omitted from the model?
- Hausman test compares OLS to IV; a significant difference suggests endogeneity.

**GM5 — Homoscedasticity:**
- Plot residuals vs. fitted values: constant spread = homoscedastic; fan shape = heteroscedastic.
- Breusch-Pagan test or White test (§7).

**GM6 — Normality of residuals:**
- Q-Q plot of residuals against theoretical normal quantiles.
- Shapiro-Wilk test (sensitive to large samples).
- With n > 100, CLT makes inference approximately valid even without normality.

---

## 3. Interpreting Coefficients

### 3.1 Raw (Unstandardized) Coefficients

> β̂_j = expected change in Y for a **one-unit increase** in X_j, **holding all other predictors constant**.

The phrase "holding all other predictors constant" is crucial and often misunderstood. It means the coefficient captures the *partial effect* of X_j after removing the linear effect of all other included regressors.

**Units matter:** If Y is revenue in dollars and X_j is age in years, β̂_j is in dollars per year. A coefficient's magnitude is only interpretable relative to its units.

### 3.2 Standardized (Beta) Coefficients

Standardize both X and Y to mean 0, SD 1 before fitting:

```
β̂_j* = β̂_j × (SD_j / SD_Y)
```

Standardized coefficients are unitless and directly comparable across predictors: β̂_j* = "how many SDs does Y move per SD increase in X_j?"

> **Why / When to use / Nuances:** Use standardized coefficients to compare the *relative importance* of predictors on different scales (e.g., age in years vs. income in thousands). Do not use them when your predictors are already on a meaningful common scale, or when you need to communicate effects in native units to business stakeholders.

### 3.3 Interaction Terms

```
Y = β₀ + β₁ X₁ + β₂ X₂ + β₃ (X₁ × X₂) + ε
```

The marginal effect of X₁ on Y is now β₁ + β₃ × X₂—it depends on the value of X₂. Never interpret β₁ alone if an interaction involving X₁ is in the model.

---

## 4. Inference: Standard Errors, t-tests, and Intervals

### 4.1 Standard Errors of OLS Coefficients

Under GM1–GM6, the variance-covariance matrix of β̂ is:

```
Var(β̂) = σ² (XᵀX)⁻¹
```

The standard error of β̂_j is the square root of the j-th diagonal element. σ² is estimated by:

```
s² = RSS / (n − p − 1) = Σ(Y_i − Ŷ_i)² / (n − p − 1)
```

where p is the number of predictors (excluding the intercept). The denominator n − p − 1 is the **degrees of freedom** for the error.

### 4.2 t-test on Individual Coefficients

```
t_j = β̂_j / SE(β̂_j)
```

Under H₀: β_j = 0, this statistic follows a t-distribution with n − p − 1 degrees of freedom.

Reject H₀ (coefficient is "statistically significant") if |t_j| > t_{α/2, n−p−1}.

With n > 100, t_{0.025, n−p−1} ≈ 1.96.

### 4.3 Confidence Intervals for Coefficients

```
95% CI: β̂_j ± t_{0.025, n−p−1} × SE(β̂_j)
```

Interpretation: In repeated sampling, 95% of such intervals would contain the true β_j.

### 4.4 Confidence vs. Prediction Intervals

| | Formula | Captures |
|---|---|---|
| **Confidence interval for E[Y\|X\*]** | Ŷ* ± t × SE_mean | Uncertainty about the *mean* response at X* |
| **Prediction interval for new Y\*** | Ŷ* ± t × SE_pred | Uncertainty about a *single new observation* at X* |

SE_pred = sqrt(SE_mean² + s²) — prediction intervals are always wider because they include the irreducible error σ².

---

## 5. R² and Adjusted R²

### 5.1 R² (Coefficient of Determination)

```
R² = 1 − RSS/TSS = SSReg/TSS
```

where TSS = Σ(Y_i − Ȳ)² (total variance), RSS = Σ(Y_i − Ŷ_i)² (residual variance).

R² measures the fraction of variance in Y explained by the model. Range: [0, 1] for models with an intercept.

**Critical flaw: R² never decreases when you add a predictor**, even if that predictor is random noise. Adding p noise variables to a model will increase R² by ≈ p/n on average. This makes R² useless for comparing models with different numbers of predictors.

### 5.2 Adjusted R²

```
Adjusted R² = 1 − (RSS / (n − p − 1)) / (TSS / (n − 1))
           = 1 − (1 − R²) × (n − 1) / (n − p − 1)
```

Adjusted R² penalizes for the number of predictors p. It *decreases* when an added predictor's contribution is smaller than chance. Use adjusted R² for model comparison.

### 5.3 When R² Misleads

- **Anscombe's Quartet:** Four datasets with identical R², means, and SDs but radically different scatterplots. Always plot your data.
- **Different targets:** R² cannot compare models with different Y variables (e.g., log(Y) vs. Y).
- **Time series:** Spurious regression can produce R² near 1 between two unrelated non-stationary series.
- **High R² ≠ good predictions:** A model can overfit (high training R², low test R²) or have large prediction intervals despite high R².

> **Why / When to use / Nuances:** In ML contexts, use cross-validated R² (or RMSE) on a held-out set. In causal inference, R² is largely irrelevant—what matters is unbiasedness of β̂, not predictive fit.

---

## 6. Multicollinearity and VIF

### 6.1 What It Is

**Perfect multicollinearity:** One predictor is an exact linear combination of others → (XᵀX)⁻¹ does not exist → OLS is undefined. Example: including both "hours_worked" and "minutes_worked = 60 × hours_worked."

**Near (imperfect) multicollinearity:** Predictors are highly but not perfectly correlated. OLS estimates exist but:
- Standard errors of collinear coefficients become large (inflated).
- Coefficients become unstable—small changes in the data produce large swings in β̂_j.
- Individual coefficient t-tests may be non-significant even though the F-test for the overall model is highly significant.

### 6.2 Variance Inflation Factor

```
VIF_j = 1 / (1 − R²_j)
```

where R²_j is the R² from regressing X_j on all other predictors.

**Interpretation:**
- VIF = 1: No correlation with other predictors.
- VIF 1–5: Moderate; generally acceptable.
- VIF > 5: High; investigate.
- VIF > 10: Severe; standard errors are inflated by more than 3× relative to orthogonal predictors.

**Caveat:** The VIF > 10 threshold is a heuristic derived under asymptotic assumptions. Research shows it produces ~40% false negatives at n = 50 and ~25% false positives at n = 5,000. ([Springer — A Caution Regarding Rules of Thumb for Variance Inflation Factors](https://link.springer.com/article/10.1007/s11135-006-9018-6))

### 6.3 Remedies

| Remedy | When to use | Tradeoff |
|---|---|---|
| Drop one of the correlated predictors | Predictors measure the same construct | Potential OVB if the dropped variable matters |
| Combine into a composite (PCA) | Large group of correlated variables | Loses individual coefficient interpretability |
| Ridge regression (L2 regularization) | All predictors theoretically relevant | Coefficients are biased; no valid p-values (§12) |
| Center and scale predictors | Interaction or polynomial terms cause collinearity | Does not reduce VIF for inherent correlations |

> **Why / When to use / Nuances:** In causal inference, you should not drop a confounder just because it is collinear—doing so introduces omitted variable bias. In prediction tasks, ridge regression handles collinearity gracefully. In inference tasks, you must choose which variable to include based on theory, not VIF alone.

---

## 7. Heteroscedasticity

### 7.1 What It Is

Heteroscedasticity means Var(ε_i | X_i) = σ_i² varies across observations, violating GM5. OLS point estimates β̂ remain unbiased (GM4 is intact), but **standard errors are wrong**, making all t-tests and F-tests invalid.

Common patterns:
- Fan shape in residual plot (variance grows with fitted values)—common in income or revenue data.
- Variance differs by group (e.g., large firms vs. small firms).

### 7.2 Detection

**Visual:** Plot residuals (or |residuals|, or residuals²) vs. fitted values. A fan shape or systematic pattern indicates heteroscedasticity.

**Formal tests:**

- **Breusch-Pagan test (1979):** Regress squared residuals on X; LM = n × R² from this auxiliary regression, distributed χ²(p) under H₀ of homoscedasticity.
- **White test (1980):** Like BP but also includes squared terms and cross-products of X, making it robust to any functional form of heteroscedasticity. White (1980): "[heteroscedasticity] leads to consistent but inefficient parameter estimates and inconsistent covariance matrix estimates." ([Ryan O'Connell Finance](https://ryanoconnellfinance.com/calculators/heteroskedasticity-test-calculator/))

### 7.3 Remedies

> **Why / When to use / Nuances — Robust SEs vs. Transform:**

| Approach | Mechanism | When to prefer |
|---|---|---|
| **Heteroscedasticity-consistent (HC) standard errors** (White/Huber-White robust SEs) | Adjust Var(β̂) = (XᵀX)⁻¹ (Xᵀ diag(ê²) X) (XᵀX)⁻¹ | When the pattern of heteroscedasticity is unknown; default in most modern econometric practice |
| **Weighted Least Squares (WLS)** | Weight observation i by 1/σ̂_i² | When you can model the variance as a function of X (e.g., variance ∝ X) |
| **Log-transform Y** | Stabilizes multiplicative variance (log-normal data) | When Y is strictly positive and skewed (e.g., revenue, prices) |
| **Generalized Least Squares (GLS)** | Full model for the error covariance structure | When the full Ω = Var(ε) is known or estimable |

**Practical recommendation:** Use HC standard errors (specifically HC1 or HC3) by default in cross-sectional data. They do not change coefficient estimates; only SEs (and therefore t-statistics and CIs) change.

---

## 8. Autocorrelation

**Definition:** Corr(ε_i, ε_j) ≠ 0 for i ≠ j. Occurs almost always in time series (yesterday's error predicts today's error) and in spatial or clustered data.

**Consequences:** Standard errors are underestimated, t-statistics are inflated, and tests have size greater than α.

### 8.1 Durbin-Watson Test

```
DW = Σ_{t=2}^{n} (ê_t − ê_{t-1})² / Σ_{t=1}^{n} ê_t²
```

- DW ≈ 2: No first-order autocorrelation.
- DW < 2 (approaching 0): Positive autocorrelation.
- DW > 2 (approaching 4): Negative autocorrelation.

DW tests only for first-order (AR(1)) autocorrelation. Use the Breusch-Godfrey test for higher-order autocorrelation.

### 8.2 Remedies

- **Clustered standard errors:** Correct SEs for within-cluster correlation without changing β̂.
- **Newey-West standard errors:** HAC (heteroscedasticity- and autocorrelation-consistent) SEs for time series.
- **Include lagged outcome as predictor:** Absorbs serial correlation into the model.
- **GLS / ARIMA models:** Model the error structure explicitly.

---

## 9. Influential Points and Leverage

### 9.1 Leverage

**Leverage** (h_ii) measures how far observation i's predictor values are from the center of the predictor space:

```
H = X(XᵀX)⁻¹Xᵀ    (hat matrix)
h_ii = i-th diagonal element of H
```

h_ii ∈ [1/n, 1]. A rule of thumb flags h_ii > 2(p+1)/n as high-leverage.

**Key distinction:** High leverage ≠ influential. A high-leverage point with a small residual may lie exactly on the regression line and exert minimal influence on β̂.

### 9.2 Cook's Distance

Cook's distance combines residual size and leverage into a single influence measure:

```
D_i = (ê_i² / (p × s²)) × (h_ii / (1 − h_ii)²)
```

Equivalently, D_i measures how much β̂ would change if observation i were deleted.

**Rule of thumb:** D_i > 4/n (or D_i > 1 for small samples) signals a potentially influential point. ([Cook's Distance — Wikipedia](https://en.wikipedia.org/wiki/Cook%27s_distance))

**Action:** Investigate influential points—do they represent data entry errors, a distinct subpopulation, or genuine outliers? Never automatically delete them; understand why they are influential.

---

## 10. Omitted Variable Bias and Confounding

### 10.1 The Formula

If the true model is Y = β₀ + β₁ X₁ + β₂ X₂ + ε but you omit X₂ and fit Y = α₀ + α₁ X₁ + u, then:

```
Bias(α̂₁) = β₂ × δ₁₂
```

where δ₁₂ = regression coefficient of X₂ on X₁ (how correlated the omitted variable is with the included variable).

**Direction of bias:**
- Omitted variable positively correlated with X₁ (δ₁₂ > 0) and positive effect (β₂ > 0) → upward bias (α̂₁ > β₁).
- Omitted variable positively correlated with X₁ and negative effect → downward bias.

### 10.2 Why This Matters for Causal Inference

OVB is the mechanism behind confounding in observational studies. If you are estimating the effect of education on income but omit ability (correlated with both education and income), your education coefficient is biased upward.

> **Why / When to use / Nuances:** Adding more control variables reduces OVB only if those variables are genuine confounders (causes of both X and Y). Adding "bad controls"—variables that are themselves caused by X (mediators)—introduces collider bias and can make estimates worse.

---

## 11. Logistic Regression Statistics

### 11.1 The Model

Logistic regression models a binary outcome P(Y = 1 | X):

```
log[p / (1 − p)] = β₀ + β₁ X₁ + ... + β_p X_p
```

The left-hand side is the **log-odds** (logit). Parameters are estimated via **Maximum Likelihood Estimation (MLE)**, not OLS—there is no closed-form solution; numerical optimization (Newton-Raphson / IRLS) is used.

### 11.2 Odds Ratios

Exponentiating a coefficient gives the **odds ratio**:

```
OR_j = exp(β̂_j)
```

**Interpretation:** For a one-unit increase in X_j, the odds of Y = 1 multiply by exp(β̂_j), holding all other variables constant.

- OR = 1: No association.
- OR > 1: X_j is associated with higher odds of Y = 1.
- OR < 1: X_j is associated with lower odds.

**Log-odds interpretation:** β̂_j is the change in the log-odds per unit increase in X_j. Less intuitive but directly linear.

**Important caveat:** Odds ratios do not equal risk ratios (relative risks) except when the outcome is rare (< ~10%). OR systematically overstates relative risk for common outcomes.

### 11.3 Inference in Logistic Regression

- **Standard errors** are derived from the inverse Fisher information matrix (observed information matrix): Var(β̂_MLE) ≈ [−∂²ℓ/∂β∂βᵀ]⁻¹.
- **Wald z-test:** z = β̂_j / SE(β̂_j), tested against N(0,1). Similar interpretation to the t-test in OLS.
- **Likelihood ratio test:** Compare log-likelihoods of the full model vs. a nested model with parameter j removed: LRT = −2[ℓ(restricted) − ℓ(full)] ~ χ²(1). More reliable than the Wald test for small samples.
- **95% CI on log-odds:** β̂_j ± 1.96 × SE; exponentiate to get CI on OR.

### 11.4 Goodness of Fit: No R²

OLS R² does not apply because logistic regression does not minimize RSS. Several **pseudo-R²** measures exist:

| Measure | Formula | Notes |
|---|---|---|
| **McFadden's** | 1 − ℓ(full) / ℓ(null) | Values 0.2–0.4 indicate good fit ([UCLA OARC](https://stats.oarc.ucla.edu/other/mult-pkg/faq/general/faq-what-are-pseudo-r-squareds/)); comparable across models with the same outcome |
| **Cox-Snell** | 1 − (L_null/L_full)^(2/n) | Cannot reach 1; often rescaled |
| **Nagelkerke** | Cox-Snell rescaled to [0,1] | Most interpretable for practitioners |
| **AUC-ROC** | Area under the ROC curve | Measures discrimination; not a variance-explained metric |

> **Why / When to use / Nuances:** McFadden's pseudo-R² is the most theoretically grounded. AUC-ROC is the most useful for evaluating predictive performance in imbalanced datasets. Do not directly compare pseudo-R² values across models with different outcomes or different sample sizes.

### 11.5 Log-Loss (Cross-Entropy)

The natural loss function for logistic regression is:

```
Log-loss = −(1/n) Σ [y_i log(p̂_i) + (1 − y_i) log(1 − p̂_i)]
```

This is exactly −ℓ(β̂)/n and measures average prediction uncertainty in bits. Lower is better.

---

## 12. Regularization's Effect on Inference

Regularization (Ridge/L2 and Lasso/L1) introduces a penalty on coefficient magnitude to reduce overfitting and handle multicollinearity. However, it fundamentally changes what the coefficients represent and invalidates classical inference.

| Property | OLS | Ridge (L2) | Lasso (L1) |
|---|---|---|---|
| Estimator | (XᵀX)⁻¹Xᵀy | (XᵀX + λI)⁻¹Xᵀy | No closed form |
| Bias | Zero (under GM1–GM4) | Introduces shrinkage bias | Introduces shrinkage bias |
| Variance | Higher (especially with multicollinearity) | Lower | Lower |
| Standard p-values | Valid (under all GM assumptions) | **Invalid — biased estimator** | **Invalid — biased + selection** |
| Handles multicollinearity | No | Yes (Ridge shrinks correlated coefficients together) | Partial (Lasso picks one of a correlated group arbitrarily) |
| Variable selection | No | No (no coefficients go to exactly zero) | Yes (some β̂ = 0) |

**Why p-values don't apply to regularized models:**
- Classical SEs assume β̂ is unbiased. Regularized estimators are biased by design.
- The penalty introduces an implicit prior (Ridge = Gaussian prior; Lasso = Laplace prior) that changes the sampling distribution.
- Post-selection inference after Lasso (e.g., "run OLS on the variables Lasso selected") is invalid without correction (see selective inference / PoSI literature).

> **Why / When to use / Nuances:** If your goal is *causal inference* or *hypothesis testing about specific coefficients*, use OLS (possibly with robust SEs) and carefully select predictors based on theory. If your goal is *prediction*, regularization is appropriate—but you lose inferential guarantees on individual coefficients. Never report "significant" p-values from a regularized model without acknowledgement that classical inference does not apply.

---

## 🎯 In the Interview

**Trap 1 — "A statistically significant coefficient means a causal effect."**
No. Statistical significance only tells you the coefficient is unlikely to be zero by chance, given the model. Causal interpretation requires a valid design (RCT, IV, DiD) and no omitted variable bias. A regression coefficient on observational data is at best an associational estimate.

**Trap 2 — "Adding more features always improves R²."**
True—R² is non-decreasing in the number of predictors. Always use adjusted R² or cross-validated metrics for model comparison.

**Trap 3 — "R² can compare models with different targets."**
False. If one model uses log(Y) and another uses Y, their R² values are not comparable (they explain variance in different quantities). Use prediction-scale RMSE or MAE instead.

**Trap 4 — "High VIF means the model is wrong."**
High VIF means certain coefficient SEs are inflated, making it hard to isolate individual effects—but it does not bias β̂. If you care about the overall model prediction, multicollinearity is less of a problem. If you care about the individual coefficient of a collinear variable, you need to address it.

**Trap 5 — "Heteroscedasticity biases the coefficients."**
No. Heteroscedasticity violates GM5 but not GM4. OLS remains unbiased; only the standard errors are wrong. Use robust SEs to fix the inference, not the point estimates.

**Trap 6 — "Logistic regression coefficients are directly comparable to linear regression coefficients."**
No. Logistic coefficients are on the log-odds scale, which is non-linear in probability. A coefficient of 0.3 means a 0.3 unit increase in log-odds, which translates to different probability changes at different baseline probability levels.

**Trap 7 — "I can interpret Lasso-selected features as causal."**
No. Lasso selects features to minimize prediction error. In the presence of multicollinearity, it picks one of a correlated group essentially arbitrarily. Post-Lasso inference requires selective inference methods.

**Common interview questions:**
- "Walk me through the OLS assumptions and what happens when each is violated."
- "What is heteroscedasticity? How do you detect and fix it?"
- "What is VIF? What does a VIF of 15 tell you?"
- "Why does R² always increase with more variables? What do you use instead?"
- "How do you interpret a logistic regression coefficient?"
- "What is omitted variable bias? Give an example."
- "What's the difference between a confidence interval and a prediction interval?"
- "Can you run p-value tests on a Ridge regression model?"

---

## Glossary

| Term | Definition |
|---|---|
| **OLS** | Ordinary Least Squares — estimator that minimizes the sum of squared residuals. |
| **BLUE** | Best Linear Unbiased Estimator — OLS under Gauss-Markov assumptions. |
| **Gauss-Markov theorem** | OLS is BLUE when GM1–GM5 hold. Normality (GM6) is additionally required for exact finite-sample t and F tests. |
| **Heteroscedasticity** | Non-constant error variance: Var(ε_i | X_i) ≠ σ². |
| **Homoscedasticity** | Constant error variance: Var(ε_i | X_i) = σ² for all i. |
| **VIF** | Variance Inflation Factor — measures how much variance of β̂_j is inflated by multicollinearity. VIF_j = 1/(1 − R²_j). |
| **Multicollinearity** | High correlation among predictors; inflates SEs and destabilizes coefficient estimates. |
| **Adjusted R²** | R² penalized for number of predictors; decreases when added variables contribute less than chance. |
| **Robust standard errors** | Heteroscedasticity-consistent (HC) SEs that remain valid under heteroscedasticity. |
| **Breusch-Pagan test** | LM test for heteroscedasticity using auxiliary regression of squared residuals on X. |
| **White test** | General test for heteroscedasticity including squared and cross-product terms; does not require normality. |
| **Durbin-Watson** | Statistic testing for first-order autocorrelation; DW ≈ 2 means no autocorrelation. |
| **Cook's distance** | Influence measure combining residual size and leverage; flags observations that substantially alter β̂ if removed. |
| **Leverage (h_ii)** | Diagonal element of the hat matrix; measures how far an observation's predictors are from the center. |
| **Omitted variable bias** | Bias in β̂_j when a variable correlated with both X_j and Y is excluded from the model. |
| **Odds ratio** | exp(β̂_j) in logistic regression; multiplicative change in odds of Y=1 per unit increase in X_j. |
| **Log-odds (logit)** | log[p/(1−p)]; the quantity that logistic regression models linearly. |
| **McFadden's pseudo-R²** | 1 − ℓ(full)/ℓ(null); goodness-of-fit measure for logistic regression. Values 0.2–0.4 indicate good fit. |
| **Prediction interval** | Interval for a single new observation; wider than CI because it includes σ². |
| **Confidence interval** | Interval for the mean response at a given X; captures uncertainty about E[Y|X]. |
| **Standardized coefficient** | Coefficient after standardizing X and Y to mean 0, SD 1; allows cross-predictor comparison. |
| **Endogeneity** | Correlation between a regressor and the error term; causes OLS to be biased and inconsistent. |

---

## References

1. Gauss-Markov Theorem and BLUE OLS Estimates — Statistics By Jim. [statisticsbyjim.com](https://statisticsbyjim.com/regression/gauss-markov-theorem-ols-blue/)
2. Key Assumptions of OLS — Albert.io Econometrics Review. [albert.io](https://www.albert.io/blog/key-assumptions-of-ols-econometrics-review/)
3. White, H. (1980). "A Heteroskedasticity-Consistent Covariance Matrix Estimator and a Direct Test for Heteroskedasticity." *Econometrica*, 48(4), 817–838.
4. Breusch, T. S., & Pagan, A. R. (1979). "A Simple Test for Heteroscedasticity and Random Coefficient Variation." *Econometrica*, 47(5), 1287–1294.
5. Cook, R. D. (1977). "Detection of Influential Observations in Linear Regression." *Technometrics*, 19(1), 15–18. [Wikipedia — Cook's distance](https://en.wikipedia.org/wiki/Cook%27s_distance)
6. O'Brien, R. M. (2007). "A Caution Regarding Rules of Thumb for Variance Inflation Factors." *Quality & Quantity*, 41, 673–690. [Springer](https://link.springer.com/article/10.1007/s11135-006-9018-6)
7. UCLA OARC. "FAQ: What Are Pseudo R-Squareds?" [stats.oarc.ucla.edu](https://stats.oarc.ucla.edu/other/mult-pkg/faq/general/faq-what-are-pseudo-r-squareds/)
8. Statlect. "Gauss-Markov Theorem." [statlect.com](https://www.statlect.com/fundamentals-of-statistics/Gauss-Markov-theorem)
9. Bookdown — A Guide on Data Analysis, Chapter 14.3: Heteroskedasticity Tests. [bookdown.org](https://bookdown.org/mike/data_analysis/heteroskedasticity-tests.html)

---

*Part of the [Statistics for Data Science](README.md) section · [Guide Home](../README.md).*
