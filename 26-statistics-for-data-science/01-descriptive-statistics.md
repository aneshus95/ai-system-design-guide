# 01 — Descriptive Statistics

Descriptive statistics **summarise and communicate the essential features of a dataset** without fitting a model or making inferences about a population. Every data-science interview begins here; interviewers use these topics to check whether a candidate truly understands their data before reaching for algorithms.

> **In plain English:** Before you model anything, you need to "look at your data." Descriptive statistics are the vocabulary you use to describe what you see — how spread out is it, where is the centre, is it lopsided, are there weird points? Getting these wrong taints every downstream analysis.

---

## Table of Contents

1. [Population vs Sample — and Bessel's Correction](#1-population-vs-sample--and-bessels-correction)
2. [Measures of Central Tendency](#2-measures-of-central-tendency)
3. [Measures of Spread](#3-measures-of-spread)
4. [Percentiles, Quantiles, Z-Scores, and Standardisation](#4-percentiles-quantiles-z-scores-and-standardisation)
5. [Shape — Skewness and Kurtosis](#5-shape--skewness-and-kurtosis)
6. [Outlier Detection](#6-outlier-detection)
7. [Covariance and Correlation](#7-covariance-and-correlation)
8. [Visual Summaries](#8-visual-summaries)
9. [Glossary](#glossary)
10. [References](#references)

---

## 1. Population vs Sample — and Bessel's Correction

| Concept | Symbol | Description |
|---|---|---|
| **Population** | N | Every possible observation of interest |
| **Sample** | n | A subset drawn from the population |
| **Population mean** | μ | `μ = (1/N) Σ xᵢ` |
| **Sample mean** | x̄ | `x̄ = (1/n) Σ xᵢ` |
| **Population variance** | σ² | `σ² = (1/N) Σ (xᵢ − μ)²` |
| **Sample variance** | s² | `s² = (1/(n−1)) Σ (xᵢ − x̄)²` |

### Why divide by n−1? (Bessel's Correction)

When you estimate variance from a sample you use x̄ (itself estimated from the data), which sits closer to the sample points than the true μ does. This **underestimates** the squared deviations. Dividing by `n − 1` instead of `n` corrects this bias, making `s²` an unbiased estimator of σ² (NIST/SEMATECH e-Handbook §6.3.3.1).

```
s² = (1/(n−1)) Σ (xᵢ − x̄)²
```

> **Note:** `s` (sample standard deviation) is **not** itself an unbiased estimator of σ; it is merely consistent. The correction for `s` would require a factor `c₄` (complex for small n). In practice s is universally used.

**Mini-example:**  
Data: {2, 4, 4, 4, 5, 5, 7, 9}  
x̄ = 40/8 = 5  
Σ(xᵢ − x̄)² = 9+1+1+1+0+0+4+16 = 32  
Population variance σ² = 32/8 = 4  
Sample variance s² = 32/7 ≈ 4.57

### 🎯 In the interview

- "When should you use n vs n−1?" → **Always n−1 for a sample; n only when you have the full population.**
- Forgetting Bessel's correction in a coding question is a common trap. In NumPy, `np.var(x)` divides by n; `np.var(x, ddof=1)` divides by n−1.

---

## 2. Measures of Central Tendency

### 2.1 Arithmetic Mean

```
x̄ = (1/n) Σᵢ xᵢ
```

- Sensitive to outliers (a single extreme value pulls x̄ hard).
- Only meaningful for **interval/ratio data** — do not average ordinal categories.

### 2.2 Median

Middle value (or mean of two middle values) after sorting.

- Robust to outliers.
- Appropriate for **skewed distributions** (income, house prices).
- 50th percentile.

### 2.3 Mode

Most frequently occurring value. Can be multimodal. Suitable for **categorical/nominal** data.

### 2.4 Weighted Mean

```
x̄_w = Σ wᵢ xᵢ / Σ wᵢ
```

Use when observations have unequal importance (survey strata weights, GPA with credit-hour weights).

### 2.5 Trimmed (Truncated) Mean

Drop the top and bottom `α%` of sorted values, then compute the mean of the remainder.

```
x̄_trim(α) = mean of middle (1−2α) fraction of sorted data
```

Balances robustness (like median) with efficiency (like mean). Used in Olympic scoring.

### 2.6 Geometric Mean

```
GM = (x₁ · x₂ · … · xₙ)^(1/n)  =  exp( (1/n) Σ ln xᵢ )
```

- Only valid for **positive values**.
- Correct average for **multiplicative processes** (compound growth rates, ratios, log-normally distributed data).
- Example: If a stock returns +50%, −50%, the arithmetic mean is 0% but the geometric mean is √(1.5 × 0.5) − 1 ≈ −13.4%, which reflects actual terminal wealth.

### 2.7 Harmonic Mean

```
HM = n / Σ (1/xᵢ)
```

- Valid for **positive values** only.
- Correct average when the quantity is a **rate** with a fixed denominator (e.g., speed when travelling equal distances at different speeds).
- Harmonic mean ≤ Geometric mean ≤ Arithmetic mean (AM-GM-HM inequality).
- F1-score in ML is the harmonic mean of precision and recall.

**Mini-example (when means diverge):**  
A car travels 60 km at 30 km/h and 60 km at 60 km/h.  
Arithmetic mean speed = 45 km/h → **wrong** (total distance = 120 km, total time = 2+1 = 3 h, actual average = 40 km/h).  
Harmonic mean = 2/(1/30 + 1/60) = 2/(0.0333+0.0167) = 40 km/h ✓

> **Why / When to use / Nuances:**
> - Normal, symmetric, no outliers → **arithmetic mean** is efficient and unbiased.
> - Skewed (income, house prices, response times) → **median**; mean is misleading.
> - Ratios, growth rates, log-scale data → **geometric mean**.
> - Rates with fixed denominator → **harmonic mean**.
> - Categorical → **mode**.
> - Robust version of mean with few outliers → **trimmed mean**.

### 🎯 In the interview

- "Median household income vs mean household income — which is reported and why?" → Median; income is right-skewed so mean is pulled up by billionaires.
- "The mean and median of a dataset are very different — what does that tell you?" → Likely skewed distribution or outliers.

---

## 3. Measures of Spread

### 3.1 Range

```
Range = x_max − x_min
```

Extremely sensitive to a single outlier. Rarely used alone.

### 3.2 Variance and Standard Deviation

```
σ² = (1/N) Σ (xᵢ − μ)²          (population)
s²  = (1/(n−1)) Σ (xᵢ − x̄)²    (sample)
σ   = √σ²      s = √s²
```

- Variance is in **squared units**; std is in the same units as the data.
- Std is the "typical distance from the mean."

### 3.3 Interquartile Range (IQR)

```
IQR = Q3 − Q1   (75th percentile − 25th percentile)
```

Robust spread measure: contains the middle 50% of data. Not affected by extreme tails.

### 3.4 Mean Absolute Deviation (MAD)

```
MAD_mean = (1/n) Σ |xᵢ − x̄|
```

Robust alternative to std; uses absolute values instead of squares so outliers have less influence.

### 3.5 Median Absolute Deviation

A robust scale estimate:

```
MAD_median = median( |xᵢ − median(x)| )
```

For a normal distribution, `σ ≈ 1.4826 × MAD_median` (Leys et al., 2013).

### 3.6 Coefficient of Variation (CV)

```
CV = (s / x̄) × 100%
```

- Unitless relative dispersion.
- Use to compare spread across datasets with different units or different means (e.g., comparing salary variability across countries).
- Only meaningful when x̄ > 0 and the data is ratio-scale.

**Mini-example:**  
Dataset A: heights in cm — mean 170, std 8, CV = 4.7%  
Dataset B: salaries in USD — mean 60,000, std 15,000, CV = 25%  
Salaries are relatively far more spread out, even though the absolute std is huge.

> **Why / When to use / Nuances:**
> - Symmetric, no outliers → **std / variance** are the standard choice.
> - Outliers present or skewed → **IQR** or **MAD_median** are more robust.
> - Comparing spread across different scales → **CV**.
> - Variance is preferred algebraically (it decomposes additively for independent variables); std is preferred for interpretation (same units).

### 🎯 In the interview

- "A feature has std=0. What does that mean?" → Constant feature; drop it (zero variance means no information).
- "Why does ML regularisation penalise weights, not variances?" → Weights live in the model space; regularisation prevents overfitting, not high spread of features.
- "std is very high — does that mean the model is wrong?" → No, high variance is a property of the data, not the model.

---

## 4. Percentiles, Quantiles, Z-Scores, and Standardisation

### 4.1 Percentiles and Quantiles

The **p-th percentile** is the value below which p% of observations fall.

```
Quartiles: Q1=P25, Q2=P50 (median), Q3=P75
Quintiles: P20, P40, P60, P80
Deciles:   P10, P20, …, P90
```

**Computing percentiles (linear interpolation method — used by NumPy, pandas):**

Sort data. The rank of the p-th percentile is `L = (p/100) × (n−1)` (0-indexed). Interpolate between `x[floor(L)]` and `x[ceil(L)]`.

**Mini-example:**  
Data (sorted): {3, 7, 8, 11, 15}  
P25: L = 0.25 × 4 = 1.0 → Q1 = x[1] = 7  
P75: L = 0.75 × 4 = 3.0 → Q3 = x[3] = 11

### 4.2 Z-Score (Standard Score)

```
z = (x − μ) / σ
```

Expresses how many standard deviations x lies from the mean. For a Normal distribution:

| Range | % of data |
|---|---|
| μ ± 1σ | ~68.27% |
| μ ± 2σ | ~95.45% |
| μ ± 3σ | ~99.73% |

(These are exact for N(0,1); real data only approximates this — Empirical Rule / 68-95-99.7 rule.)

### 4.3 Standardisation (Z-score Normalisation)

```
x_std = (x − x̄) / s
```

Transforms a feature to mean 0, std 1. This is **not** the same as normalisation (min-max scaling to [0,1]).

**When to standardise:**
- Algorithms sensitive to scale: logistic regression, SVM, PCA, KNN, gradient descent.
- When comparing features on different scales.

**When NOT to standardise:**
- Tree-based models (decision trees, random forests, gradient boosting) — scale invariant.
- When the original scale is meaningful and interpretable.

> **Why / When to use / Nuances:**
> - Standardisation preserves the shape of the distribution; it shifts and scales but does not make data "normal."
> - Min-max scaling is preferred when you need bounded output (e.g., neural network inputs); z-score when the algorithm assumes zero-mean, unit-variance inputs.

### 🎯 In the interview

- "A z-score of 3.5 — is it an outlier?" → By the z-score rule (|z| > 3), yes — but only if data is approximately normal.
- "Does standardisation make your data Gaussian?" → **No.** Shape is unchanged; only mean and scale shift.

---

## 5. Shape — Skewness and Kurtosis

### 5.1 Skewness

Measures asymmetry of the distribution around its mean.

```
skewness = E[(X − μ)³] / σ³
         = (1/n) Σ ((xᵢ − x̄)/s)³    (sample, simplified)
```

| Value | Interpretation |
|---|---|
| skewness > 0 | Right-skewed (positive): long right tail; mean > median |
| skewness = 0 | Symmetric |
| skewness < 0 | Left-skewed (negative): long left tail; mean < median |

**Thumb rule:** |skewness| > 1 is substantially skewed; |skewness| > 2 is highly skewed.

**Examples:** Income (right-skewed), exam scores near a ceiling (left-skewed), temperature (often near-symmetric).

### 5.2 Kurtosis

Measures the "tailedness" (and implicit peakedness) relative to a normal distribution.

```
kurtosis = E[(X − μ)⁴] / σ⁴
```

**Excess kurtosis** (most software default) = kurtosis − 3.

| Excess kurtosis | Type | Description |
|---|---|---|
| = 0 | Mesokurtic | Normal distribution |
| > 0 | Leptokurtic | Heavier tails, sharper peak (e.g., Student's t, financial returns) |
| < 0 | Platykurtic | Lighter tails, flatter peak (e.g., uniform distribution) |

**Financial data is typically leptokurtic** — more extreme events than a normal model predicts (fat tails). This is critical for risk management.

**Mini-example:**  
Normal data: skewness ≈ 0, excess kurtosis ≈ 0  
S&P 500 daily returns: slight negative skew, excess kurtosis ≈ 5–8 (fat tails)

> **Why / When to use / Nuances:**
> - Many statistical tests assume normality; check skewness and excess kurtosis first.
> - High kurtosis (fat tails) means outliers are more common than expected — important for fraud detection and anomaly detection.
> - Kurtosis does NOT measure "peakedness" alone — it primarily captures tail weight (DeCarlo, 1997).

### 🎯 In the interview

- "The distribution of your target variable is right-skewed. How does this affect modelling?" → Consider log-transform; report median + IQR not mean; evaluate model on median absolute error, not MSE.
- "What is excess kurtosis of 0?" → The distribution has tails as heavy as a normal distribution.

---

## 6. Outlier Detection

### 6.1 IQR Rule (Tukey's Fences)

```
Lower fence = Q1 − 1.5 × IQR
Upper fence = Q3 + 1.5 × IQR
```

Points outside these fences are flagged. For "extreme" outliers, use the factor 3.0 instead of 1.5. This is the rule used by boxplots (Tukey, 1977).

### 6.2 Z-Score Rule

Flag points with |z| > 3 (or sometimes 2.5 or 3.5 depending on strictness).

```
z = (xᵢ − x̄) / s
```

**Problem:** The mean and std are themselves sensitive to outliers, so a masking effect can hide extreme values.

### 6.3 Modified Z-Score (Iglewicz & Hoaglin, 1993)

Uses the robust MAD_median instead:

```
M_i = 0.6745 × (xᵢ − median(x)) / MAD_median
```

Flag when |M_i| > 3.5. The constant 0.6745 makes M consistent with z-scores under normality (0.6745 ≈ Φ⁻¹(0.75)).

**Mini-example:**  
Data: {2, 3, 3, 4, 4, 4, 5, 5, 100}  
Median = 4, MAD_median = 1  
M for 100 = 0.6745 × (100−4)/1 = 64.7 → clearly flagged.  
Regular z-score would be distorted because x̄ ≈ 14.4, s ≈ 32.

> **Why / When to use / Nuances:**
> - IQR rule: fast, interpretable, robust — the default for exploratory analysis and boxplots.
> - Z-score rule: only reliable when data is approximately normal with no extreme outliers.
> - Modified Z-score: best when the dataset may already contain outliers (they don't inflate the estimator).
> - Domain knowledge always overrides statistical rules — a 7-foot basketball player is not a data error.

### 🎯 In the interview

- "How do you detect outliers?" → Mention at least two methods, state assumptions, and emphasise domain knowledge.
- "Should you always remove outliers?" → **No.** Sometimes they are the signal (fraud, rare disease, mechanical failure). Decision depends on why they exist.

---

## 7. Covariance and Correlation

### 7.1 Covariance

```
Cov(X,Y) = (1/(n−1)) Σ (xᵢ − x̄)(yᵢ − ȳ)   (sample)
```

- Positive: X and Y tend to move together.
- Negative: X and Y tend to move in opposite directions.
- Zero: no **linear** relationship (but nonlinear could exist).
- Units are the product of the units of X and Y — hard to interpret in absolute terms.

### 7.2 Pearson Correlation

```
r = Cov(X,Y) / (s_X · s_Y)
```

- Ranges from −1 to +1.
- Measures **linear** association.
- Assumes both variables are continuous and the relationship is linear; sensitive to outliers.
- A single outlier can drive r from 0 to ±0.9.

### 7.3 Spearman Rank Correlation

```
ρ = Pearson r computed on the ranks of X and Y
```

- Measures **monotonic** (not necessarily linear) association.
- Robust to outliers and non-normal distributions.
- Appropriate for ordinal data or when Pearson assumptions are violated.
- Range: −1 to +1.

### 7.4 Kendall's τ (Tau)

```
τ = (concordant pairs − discordant pairs) / (n(n−1)/2)
```

- A pair (i,j) is concordant if the rankings agree; discordant if they disagree.
- More robust than Spearman for small samples or when there are many ties.
- Slower to compute (O(n²) naïve vs O(n log n) for Spearman).
- Interpretation: τ = 0.6 means 60% more concordant pairs than discordant.

**Mini-example:**  
X = [1,2,3,4,5], Y = [2,1,4,3,6]  
Pearson r = 0.943 (linear, near-perfect)  
Spearman ρ = 0.900 (monotonic, slightly lower due to rank swaps)  
Kendall τ ≈ 0.800

> **Why / When to use / Nuances:**
> - Pearson: best when both variables are continuous, bivariate-normal, relationship is linear.
> - Spearman: ordinal data, non-linear monotonic relationships, outliers present.
> - Kendall: small samples, tied ranks, when a probabilistic interpretation (concordance) is useful.
> - All three can be zero even when there is a strong nonlinear relationship (e.g., Y = X²).

### 7.5 Correlation is Not Causation

A correlation between X and Y can arise because:
1. X causes Y
2. Y causes X
3. A confounding variable Z causes both X and Y
4. Pure coincidence (spurious correlation)

**Classic example:** Ice cream sales and drowning deaths are positively correlated. Confounder: hot weather increases both. Banning ice cream will not prevent drownings.

To establish causation, you need a **randomised controlled experiment** or careful causal inference (Pearl's do-calculus, instrumental variables, etc.).

### 🎯 In the interview

- "Correlation between two features is 0.95 — should you drop one?" → Multicollinearity matters for some models (logistic regression, linear regression) but not tree models. Check VIF. Don't blindly drop.
- "A feature has near-zero Pearson correlation with the target but your model finds it important. Why?" → Nonlinear relationship — Pearson only captures linear association.
- "Correlation dropped after adding more data. What happened?" → New data may have changed the distribution; there may have been selection bias in the original sample.

---

## 8. Visual Summaries

### 8.1 Histogram

Groups data into equal-width bins and plots count/frequency/density.

**What it reveals:** Shape (skewness, modality), spread, approximate range, where data is dense.

**Nuances:**
- Bin width matters — too narrow is noisy (every point is a spike), too wide hides structure. Sturges' Rule: `k = ⌈log₂(n) + 1⌉`; Freedman-Diaconis: `bin_width = 2 × IQR × n^(−1/3)`.
- A histogram shows frequency within bins, not individual points — values within a bin are invisible.

### 8.2 Boxplot (Box-and-Whisker Plot)

Displays: Q1, median (Q2), Q3 as the box; whiskers extend to the last data point within 1.5×IQR; beyond that, points are plotted individually as outlier dots.

**What it reveals:** Median, IQR, symmetry, tail length, potential outliers.

**Nuances:**
- Hides multimodality — a bimodal distribution can look symmetric in a boxplot.
- Better for comparing multiple groups side-by-side than for a single distribution.
- Violin plot = boxplot + KDE overlay, revealing multimodality.

### 8.3 Empirical Cumulative Distribution Function (ECDF)

```
ECDF(x) = (number of observations ≤ x) / n
```

Plots the fraction of data ≤ x against x.

**What it reveals:** Full shape without binning; percentiles directly readable; comparison of two distributions (stochastic dominance); no bin-width artefacts.

**Preferred by many statisticians** over histograms for exploratory analysis (Harrell, 2015).

### 8.4 QQ-Plot (Quantile-Quantile Plot)

Plots the quantiles of your data against the theoretical quantiles of a reference distribution (usually Normal).

- Points lie on a straight 45° line → data follows the reference distribution.
- S-shaped curve → distribution has lighter tails than reference.
- Bow-shaped curve → distribution has heavier tails (leptokurtic).
- Points deviate at top and bottom → skewness present.

**What it reveals:** Distributional assumptions, tail behaviour, outliers in the tails.

**Mini-example:**  
A QQ-plot of model residuals: if points curve upward at the right, residuals are right-skewed — the normality assumption for OLS inference is violated.

### 🎯 In the interview

- "How do you check if data is normally distributed?" → QQ-plot visually + Shapiro-Wilk or Kolmogorov-Smirnov test formally (but note: for large n, any test will reject; rely on QQ-plot and domain knowledge).
- "A boxplot shows no outliers but the histogram is bimodal — why?" → Bimodality is hidden in boxplots. Always use multiple plots.
- "What does it mean if your ECDF and a theoretical CDF diverge badly in the tails?" → Your data has different tail behaviour — extreme events are more (or less) likely than the model assumes.

---

## Glossary

| Term | Simple explanation | Purpose |
|---|---|---|
| **Mean** | Sum divided by count | Measure of centre for symmetric data |
| **Median** | Middle value after sorting | Robust centre for skewed data |
| **Mode** | Most frequent value | Centre for categorical or discrete data |
| **Geometric mean** | n-th root of the product of values | Averaging multiplicative processes or growth rates |
| **Harmonic mean** | n divided by sum of reciprocals | Averaging rates with a fixed denominator |
| **Variance** | Average squared deviation from mean | Measure of spread (squared units) |
| **Standard deviation** | Square root of variance | Spread in original units |
| **IQR** | Q3 − Q1 | Robust spread; middle 50% of data |
| **MAD** | Median of absolute deviations from median | Most robust spread measure |
| **CV** | Std divided by mean (%) | Relative spread; compare across scales |
| **Percentile** | Value below which p% of data falls | Rank-based position |
| **Z-score** | (x − mean) / std | How many SDs from the mean |
| **Skewness** | Normalised 3rd central moment | Direction and degree of asymmetry |
| **Kurtosis** | Normalised 4th central moment | Tailedness relative to normal |
| **Pearson r** | Cov(X,Y)/(σ_X σ_Y) | Linear association between two variables |
| **Spearman ρ** | Pearson r on ranks | Monotonic association; robust to outliers |
| **Kendall τ** | (Concordant − discordant) / total pairs | Probability of concordance; robust with ties |
| **Bessel's correction** | Divide by n−1 not n in sample variance | Makes sample variance an unbiased estimator of σ² |
| **IQR outlier rule** | Points outside Q1−1.5×IQR or Q3+1.5×IQR | Tukey's standard outlier flag |
| **ECDF** | Fraction of data ≤ x | Full distributional picture without binning |
| **QQ-plot** | Sample quantiles vs theoretical quantiles | Check distributional assumptions |

---

## References

1. NIST/SEMATECH e-Handbook of Statistical Methods, §6.3 — *Measures of Shape* and §6.3.3 — *Bessel's Correction*. https://www.itl.nist.gov/div898/handbook/
2. Tukey, J. W. (1977). *Exploratory Data Analysis*. Addison-Wesley. — Original source of the 1.5×IQR rule and boxplot.
3. Iglewicz, B. & Hoaglin, D. (1993). *How to Detect and Handle Outliers*. ASQC Quality Press. — Modified Z-score definition.
4. DeCarlo, L. T. (1997). "On the meaning and use of kurtosis." *Psychological Methods, 2*(3), 292–307. https://doi.org/10.1037/1082-989X.2.3.292
5. Leys, C. et al. (2013). "Detecting outliers: Do not use standard deviation around the mean, use absolute deviation around the median." *Journal of Experimental Social Psychology, 49*(4), 764–766.
6. Freedman, D. & Diaconis, P. (1981). "On the histogram as a density estimator." *Z. Wahrscheinlichkeitstheorie*, 57, 453–476.
7. Harrell, F. E. (2015). *Regression Modeling Strategies* (2nd ed.). Springer. — Advocates ECDF over histograms.
8. Ang, A. & Chen, J. (2002). "Asymmetric correlations of equity portfolios." *Journal of Financial Economics, 63*(3), 443–494. — Real-world correlation pitfalls.

---

*Part of the [Statistics for Data Science](README.md) section · [Guide Home](../README.md).*
