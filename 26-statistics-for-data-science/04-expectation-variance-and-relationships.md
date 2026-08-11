# Expectation, Variance, and Relationships

Expectation and variance are the two most fundamental summary statistics of a random variable — they describe *where* probability is centered and *how spread out* it is. Beyond simple summaries, their properties (linearity, the law of total expectation, bias-variance decomposition) underpin virtually every algorithm in machine learning and statistical inference.

> **In plain English:** Expectation is the long-run average; variance measures how far outcomes typically stray from that average. Their algebra — linearity, covariance, conditional expectation — is the grammar of probabilistic reasoning.

## Table of Contents

1. [Expectation](#1-expectation)
   - [Definition](#11-definition)
   - [Linearity of Expectation](#12-linearity-of-expectation)
   - [LOTUS (Law of the Unconscious Statistician)](#13-lotus)
2. [Variance and Standard Deviation](#2-variance-and-standard-deviation)
   - [Definition and the Computational Formula](#21-definition-and-the-computational-formula)
   - [Why E[X²] ≠ E[X]²](#22-why-ex-ex)
   - [Properties of Variance](#23-properties-of-variance)
3. [Covariance and Correlation](#3-covariance-and-correlation)
   - [Covariance](#31-covariance)
   - [Correlation (Pearson)](#32-correlation-pearson)
   - [Properties](#33-properties)
4. [Variance of Sums](#4-variance-of-sums)
   - [Independent Variables](#41-independent-variables)
   - [Correlated Variables](#42-correlated-variables)
5. [Conditional Expectation](#5-conditional-expectation)
   - [Law of Total Expectation (Adam's Law)](#51-law-of-total-expectation-adams-law)
   - [Law of Total Variance (Eve's Law)](#52-law-of-total-variance-eves-law)
6. [Moments and Moment-Generating Functions](#6-moments-and-moment-generating-functions)
7. [Jensen's Inequality](#7-jensens-inequality)
8. [Chebyshev's Inequality](#8-chebyshevs-inequality)
9. [Bias-Variance Decomposition of an Estimator](#9-bias-variance-decomposition-of-an-estimator)
10. [🎯 In the Interview: Common Traps](#10-in-the-interview-common-traps)
11. [Glossary](#glossary)
12. [References](#references)

---

## 1. Expectation

### 1.1 Definition

The **expected value** (or expectation, or mean) of a random variable X is the probability-weighted average of its possible values.

**Discrete:**
```
E[X] = Σₓ x · P(X = x)
```

**Continuous:**
```
E[X] = ∫₋∞^∞ x · f(x) dx
```

where f(x) is the PDF.

**Interpretation:** If you repeat the random experiment infinitely many times and average the outcomes, the result converges to E[X] (this is the Law of Large Numbers).

**Worked example:** Roll a fair six-sided die. E[X] = (1 + 2 + 3 + 4 + 5 + 6) / 6 = 21/6 = 3.5. You never roll 3.5, but after many rolls the average approaches 3.5.

**When does E[X] not exist?** When the defining sum/integral diverges. Example: Cauchy distribution, Pareto with α ≤ 1. The St. Petersburg paradox (infinite expected payout) is a famous illustration.

---

### 1.2 Linearity of Expectation

This is the most useful property in probability:

```
E[aX + bY + c] = a·E[X] + b·E[Y] + c
```

**For any random variables X and Y** (no independence required!):
```
E[X + Y] = E[X] + E[Y]
```

**Worked example — Indicator trick:** How many heads in 10 fair coin flips?

Define Iᵢ = 1 if flip i is heads, 0 otherwise. Total heads = I₁ + I₂ + ··· + I₁₀.

E[Iᵢ] = 0.5 for each i. By linearity: E[total] = 10 × 0.5 = 5.

No need to compute the full Binomial sum! This indicator/linearity trick solves many combinatorial expectation problems elegantly.

**🎯 In the interview:** A classic problem: "Expected number of fixed points in a random permutation of n elements?" Answer = 1, regardless of n. Proof: let Iᵢ = 1 if element i is fixed. E[Iᵢ] = 1/n. By linearity: E[Σ Iᵢ] = n × (1/n) = 1.

---

### 1.3 LOTUS

**Law of the Unconscious Statistician (LOTUS):** You can compute E[g(X)] without finding the distribution of g(X) — just use the distribution of X:

**Discrete:**
```
E[g(X)] = Σₓ g(x) · P(X = x)
```

**Continuous:**
```
E[g(X)] = ∫₋∞^∞ g(x) · f(x) dx
```

**Why "unconscious"?** Students sometimes unconsciously (and incorrectly) write E[g(X)] = g(E[X]). LOTUS is the correct formula; g(E[X]) is a different (usually wrong) thing — see Jensen's inequality.

**Worked example:** X ~ Uniform(0, 1). Compute E[X²].

```
E[X²] = ∫₀¹ x² · 1 dx = [x³/3]₀¹ = 1/3
```

Note: E[X] = 1/2, so E[X]² = 1/4 ≠ 1/3 = E[X²]. This is always true unless X is deterministic.

---

## 2. Variance and Standard Deviation

### 2.1 Definition and the Computational Formula

**Definition:**
```
Var(X) = E[(X − E[X])²]
```
Variance is the expected squared deviation from the mean. It's always ≥ 0.

**Standard deviation:** σ = √Var(X) — in the same units as X, making it more interpretable.

**Computational (shortcut) formula:**
```
Var(X) = E[X²] − (E[X])²
```

**Derivation:**
```
Var(X) = E[(X − μ)²]
       = E[X² − 2μX + μ²]
       = E[X²] − 2μ·E[X] + μ²        (linearity)
       = E[X²] − 2μ² + μ²
       = E[X²] − μ²
       = E[X²] − (E[X])²
```

This formula is crucial: computing E[X²] and E[X] separately and subtracting is often easier than computing the full squared-deviation sum.

**Worked example:** X ~ Bernoulli(p). E[X] = p; E[X²] = E[X] = p (since X² = X for 0/1 variables). Var(X) = p − p² = p(1−p). ✓

---

### 2.2 Why E[X²] ≠ E[X]²

Unless X is a constant (deterministic), we always have **E[X²] > (E[X])²**, and their difference is exactly the variance:

```
E[X²] − (E[X])² = Var(X) ≥ 0
```

**Intuition:** (E[X])² is the square of the average. E[X²] is the average of squares. Squaring amplifies deviations — large outliers contribute more to E[X²] than to (E[X])².

**Why this matters in ML:** In mean squared error, the irreducible noise component is E[ε²] ≠ (E[ε])² = 0 (if noise is zero-mean). The squared expectation vanishes but the expectation of the square does not.

**Worked example:** X takes values {−1, +1} each with probability 0.5.
- E[X] = 0
- E[X²] = (−1)² × 0.5 + (1)² × 0.5 = 1
- (E[X])² = 0² = 0
- Var(X) = 1 − 0 = 1 ✓

---

### 2.3 Properties of Variance

```
Var(c) = 0                          (constants have zero variance)
Var(aX) = a²·Var(X)                 (scaling: note it's a², not a)
Var(X + c) = Var(X)                 (shifting doesn't change spread)
Var(aX + b) = a²·Var(X)
```

**Worked example:** If X ~ N(0, 1), then Y = 3X + 5 has Var(Y) = 9 × 1 = 9, so σ_Y = 3. Mean shifts, spread scales by |a|.

**🎯 In the interview:** "What is Var(2X + 3Y) when X, Y are independent?" = 4Var(X) + 9Var(Y). The cross terms vanish only when Cov(X, Y) = 0 (see Section 4).

---

## 3. Covariance and Correlation

### 3.1 Covariance

**Definition:**
```
Cov(X, Y) = E[(X − E[X])(Y − E[Y])]
```

**Computational formula:**
```
Cov(X, Y) = E[XY] − E[X]·E[Y]
```

(Source: [probabilitycourse.com](https://www.probabilitycourse.com/chapter5/5_3_1_covariance_correlation.php))

**Derivation:**
```
Cov(X, Y) = E[(X − μₓ)(Y − μᵧ)]
           = E[XY − μᵧX − μₓY + μₓμᵧ]
           = E[XY] − μᵧE[X] − μₓE[Y] + μₓμᵧ
           = E[XY] − μₓμᵧ
```

**Sign interpretation:**
- Cov > 0: X and Y tend to move together (both high or both low)
- Cov < 0: X and Y tend to move in opposite directions
- Cov = 0: no linear relationship (but see Section 3.3 — may still be dependent!)

**Special cases:**
```
Cov(X, X) = Var(X)                  (variance is covariance with itself)
Cov(X, c) = 0                       (constants have no covariance)
Cov(aX + b, cY + d) = ac·Cov(X, Y) (scale linearly)
```

**Worked example:** X = hours studied, Y = exam score. We compute from data: E[XY] = 85, E[X] = 5, E[Y] = 16. Cov(X, Y) = 85 − 5×16 = 85 − 80 = 5 > 0. More study → higher scores.

---

### 3.2 Correlation (Pearson)

**Definition (Pearson correlation coefficient):**
```
ρ(X, Y) = Corr(X, Y) = Cov(X, Y) / (σₓ · σᵧ)
```

where σₓ = √Var(X) and σᵧ = √Var(Y).

**Range:** −1 ≤ ρ ≤ 1 (always, by Cauchy-Schwarz inequality).

**Interpretation:**
- ρ = +1: Perfect positive linear relationship
- ρ = −1: Perfect negative linear relationship
- ρ = 0: No linear relationship (does NOT mean independent in general)
- |ρ| > 0.7: Strong linear association (rule of thumb)

**Why divide by standard deviations?** Covariance is in units of X × Y, making it hard to compare across different variable pairs. Correlation is dimensionless and unit-free.

> **Why / When to use Pearson vs. Covariance:** Always report correlation (not raw covariance) when comparing strength of relationships across different pairs of variables. Pearson assumes a linear relationship; for monotonic non-linear relationships, use Spearman rank correlation instead.

**Worked example (continued):** σₓ = 2, σᵧ = 3. ρ = 5 / (2 × 3) = 5/6 ≈ 0.83. Strong positive linear relationship.

**🎯 In the interview:** "Correlation only captures LINEAR association" is a critical nuance. X and Y = X² can have ρ = 0 if X is symmetric around 0, yet Y is perfectly determined by X. Always inspect scatter plots; never rely on correlation alone.

---

### 3.3 Properties

| Property | Formula |
|---|---|
| Symmetry | Cov(X, Y) = Cov(Y, X) |
| Self-covariance | Cov(X, X) = Var(X) |
| Independence implies Cov=0 | If X⊥Y then Cov(X,Y)=0 |
| Cov=0 does NOT imply independence | (except for jointly normal) |
| Bilinearity | Cov(aX+bY, Z) = a·Cov(X,Z) + b·Cov(Y,Z) |
| Cauchy-Schwarz | Cov(X,Y)² ≤ Var(X)·Var(Y) → |ρ| ≤ 1 |

**The critical asymmetry:** Independence → zero covariance, but zero covariance ↛ independence. The exception is the multivariate normal: for jointly normal variables, zero correlation *does* imply independence.

**Worked counterexample:** X ~ Uniform(−1, 1), Y = X².
- E[X] = 0, E[X³] = 0 (odd function, symmetric distribution)
- E[XY] = E[X·X²] = E[X³] = 0
- Cov(X, Y) = 0, ρ = 0
- But Y = X² is a deterministic function of X — they are maximally dependent!

---

## 4. Variance of Sums

### 4.1 Independent Variables

If X and Y are independent (or more generally, uncorrelated):
```
Var(X + Y) = Var(X) + Var(Y)
Var(X − Y) = Var(X) + Var(Y)   ← note: it's PLUS, not minus!
```

Variances add whether you add or subtract independent variables, because variance measures squared spread regardless of direction.

**Extension to n variables:**
```
Var(X₁ + X₂ + ··· + Xₙ) = Var(X₁) + Var(X₂) + ··· + Var(Xₙ)   (if all independent)
```

**Worked example:** You roll two dice: X, Y ~ Discrete Uniform(1,6), independent. Var(X) = Var(Y) = 35/12. Var(X + Y) = 35/12 + 35/12 = 35/6 ≈ 5.83.

---

### 4.2 Correlated Variables

The general formula (no independence assumed):
```
Var(X + Y) = Var(X) + Var(Y) + 2·Cov(X, Y)
```

```
Var(X − Y) = Var(X) + Var(Y) − 2·Cov(X, Y)
```

**General linear combination:**
```
Var(aX + bY) = a²·Var(X) + b²·Var(Y) + 2ab·Cov(X, Y)
```

**For n variables (general form):**
```
Var(Σᵢ aᵢXᵢ) = Σᵢ aᵢ²·Var(Xᵢ) + 2·Σᵢ<ⱼ aᵢaⱼ·Cov(Xᵢ, Xⱼ)
```

In matrix form: if **a** is a vector of coefficients and **Σ** is the covariance matrix:
```
Var(**a**ᵀ**X**) = **a**ᵀ**Σ**·**a**   (quadratic form)
```

**Portfolio risk application:** If w₁, w₂ are portfolio weights and ρ is asset correlation:
```
σ²_portfolio = w₁²σ₁² + w₂²σ₂² + 2w₁w₂ρσ₁σ₂
```
Diversification works because adding a negatively correlated asset reduces Var.

**Worked example:** Two correlated assets: σ₁ = σ₂ = 0.2, ρ = 0.5, w₁ = w₂ = 0.5.
```
σ²_portfolio = 0.25(0.04) + 0.25(0.04) + 2(0.25)(0.5)(0.04)
             = 0.01 + 0.01 + 0.01 = 0.03
σ_portfolio  = √0.03 ≈ 0.173
```
Less than either individual σ = 0.2 because of diversification.

**🎯 In the interview:** "Why does Var(X−Y) = Var(X) + Var(Y) when independent?" Intuition: subtracting something uncertain doesn't reduce your uncertainty — it doubles it. The coin flip difference: Var(H−T) = Var(H) + Var(T) since they're independent and Cov = 0.

---

## 5. Conditional Expectation

**Definition:**
```
E[X | Y = y] = Σₓ x · P(X=x | Y=y)      (discrete)
E[X | Y = y] = ∫ x · f(x|y) dx            (continuous)
```

E[X | Y] is itself a random variable — it varies as Y varies. Think of it as "the best prediction of X given you know Y."

### 5.1 Law of Total Expectation (Adam's Law)

```
E[X] = E[E[X | Y]]
```

"The expectation of the conditional expectation equals the unconditional expectation."

**Proof sketch (discrete):**
```
E[E[X|Y]] = Σᵧ E[X|Y=y] · P(Y=y)
           = Σᵧ [Σₓ x · P(X=x|Y=y)] · P(Y=y)
           = Σₓ x · Σᵧ P(X=x, Y=y)
           = Σₓ x · P(X=x) = E[X]
```

**Worked example — Mixture model:** 60% of customers are Type A (avg purchase $20) and 40% are Type B (avg purchase $50). Expected purchase = E[E[purchase | type]] = 0.6×20 + 0.4×50 = 12 + 20 = $32.

**Applied to tower property:** E[X] = E[E[X|Y]] can be applied recursively: E[X] = E[E[E[X|Y,Z]|Y]]. Useful in hierarchical models.

**🎯 In the interview:** A classic problem: "In a random graph where each edge exists independently with probability p, what is the expected number of triangles?" Use linearity of expectation and indicator variables, not conditioning — much cleaner.

---

### 5.2 Law of Total Variance (Eve's Law)

```
Var(X) = E[Var(X|Y)] + Var(E[X|Y])
```

Decomposition: Total variance = average within-group variance + between-group variance.

**Mnemonic:** "Eve's Law" = EV + VE (Expected Variance + Variance of Expectation).

**Worked example:** Exam score X. Students are either prepared (Y=1) or unprepared (Y=0). P(Y=1) = 0.7.

- E[X|Y=1] = 80, Var(X|Y=1) = 25
- E[X|Y=0] = 50, Var(X|Y=0) = 100

Within-group variance: E[Var(X|Y)] = 0.7×25 + 0.3×100 = 17.5 + 30 = 47.5

Between-group variance: Var(E[X|Y]) — first compute E[X|Y]: it equals 80 with prob 0.7 and 50 with prob 0.3.
- E[E[X|Y]] = 0.7×80 + 0.3×50 = 56 + 15 = 71
- Var(E[X|Y]) = 0.7×(80−71)² + 0.3×(50−71)² = 0.7×81 + 0.3×441 = 56.7 + 132.3 = 189

Total Var(X) = 47.5 + 189 = 236.5

**Practical use:** In hierarchical models (multilevel regression), this decomposition separates within-cluster from between-cluster variation.

---

## 6. Moments and Moment-Generating Functions

### Moments

The **n-th moment** of X is:
```
μₙ' = E[Xⁿ]
```

The **n-th central moment** (around the mean) is:
```
μₙ = E[(X − E[X])ⁿ]
```

| Moment | Name | What it measures |
|---|---|---|
| E[X] = μ₁' | Mean | Center/location |
| E[(X−μ)²] = μ₂ | Variance | Spread |
| μ₃ / σ³ | Skewness | Asymmetry |
| μ₄ / σ⁴ | Kurtosis | Tail heaviness |

**Skewness:** Positive = right tail (mean > median); Negative = left tail. Normal distribution has skewness = 0.

**Kurtosis:** Normal has kurtosis = 3 (sometimes "excess kurtosis" = kurtosis − 3 = 0 is used). Higher kurtosis → heavier tails (leptokurtic). Lower → lighter tails (platykurtic).

### Moment-Generating Functions (MGF)

**Definition:**
```
M_X(t) = E[e^(tX)]   (defined for t in a neighborhood of 0)
```

**Why useful?**
1. Differentiating recovers moments: M_X^(n)(0) = E[Xⁿ]
2. Uniqueness: if two distributions have the same MGF (in an open interval around 0), they are identical.
3. Independence: if X, Y independent, M_{X+Y}(t) = M_X(t) · M_Y(t) — convolution becomes multiplication.
4. Used to prove CLT rigorously.

**Worked example — Normal MGF:**
```
M_X(t) = exp(μt + σ²t²/2)   for X ~ N(μ, σ²)
```
M_X'(0) = μ = E[X] ✓
M_X''(0) = σ² + μ² = E[X²] ✓ → Var(X) = E[X²] − (E[X])² = σ² ✓

**Common MGFs:**

| Distribution | MGF M(t) |
|---|---|
| Bernoulli(p) | 1 − p + pe^t |
| Binomial(n,p) | (1 − p + pe^t)ⁿ |
| Poisson(λ) | exp(λ(e^t − 1)) |
| Normal(μ,σ²) | exp(μt + σ²t²/2) |
| Exponential(λ) | λ/(λ−t), t < λ |
| Gamma(α,λ) | (λ/(λ−t))^α, t < λ |

**🎯 In the interview:** "How do you show that the sum of independent Poisson(λ₁) and Poisson(λ₂) is Poisson(λ₁+λ₂)?" — Multiply MGFs: exp(λ₁(e^t−1)) · exp(λ₂(e^t−1)) = exp((λ₁+λ₂)(e^t−1)), which is the MGF of Poisson(λ₁+λ₂). Elegant and rigorous.

---

## 7. Jensen's Inequality

**Statement:** For a **convex** function g and a random variable X:
```
g(E[X]) ≤ E[g(X)]
```

For a **concave** function g:
```
g(E[X]) ≥ E[g(X)]
```

**Equality holds** if and only if X is constant (deterministic) or g is linear.

**Intuition:** A convex function curves upward. The average of the outputs (E[g(X)]) is pulled up by the curvature more than the output of the average (g(E[X])). Visually, the chord connecting two points on a convex curve lies above the curve.

**Convex functions:** x², e^x, x log x, |x|, 1/x (for x > 0), −log(x)
**Concave functions:** log(x), √x, −x², −e^x

**Applications:**

1. **E[X²] ≥ (E[X])²** — g(x) = x² is convex. Jensen gives Var(X) = E[X²] − (E[X])² ≥ 0. ✓

2. **Information theory (KL divergence non-negativity):** KL(P||Q) = E_P[log(P/Q)] = −E_P[log(Q/P)] ≥ −log(E_P[Q/P]) = −log(1) = 0. (By Jensen, since −log is convex.)

3. **EM algorithm (ELBO):** The evidence lower bound in variational inference is derived using Jensen's inequality on the log-likelihood of incomplete data.

4. **Expected utility theory:** A risk-averse agent has a concave utility function u. Jensen gives E[u(W)] ≤ u(E[W]) — the certain amount E[W] is preferred to the gamble, even at the same expected value.

5. **Geometric mean ≤ Arithmetic mean:** log is concave → E[log X] ≤ log(E[X]) → exp(E[log X]) ≤ E[X] → GM ≤ AM. ✓

**Worked example:** X ~ Uniform(0, 4). g(x) = e^x (convex).
- E[X] = 2; g(E[X]) = e² ≈ 7.39
- E[g(X)] = E[e^X] = (1/4)∫₀⁴ eˣ dx = (e⁴−1)/4 ≈ (54.6−1)/4 ≈ 13.4
- 13.4 > 7.39 ✓ Jensen confirmed.

> **Why / When to use:** Jensen is used whenever you need to bound E[g(X)] from below (convex g) or above (concave g). It appears in KL divergence proofs, variational methods, portfolio theory, and information-theoretic arguments.

**🎯 In the interview:** "Why is E[log X] ≤ log(E[X])?" — log is concave; apply Jensen. This inequality appears in proving that ML estimators are asymptotically consistent and in bounding entropy.

---

## 8. Chebyshev's Inequality

**Statement:** For any random variable X with finite mean μ and variance σ², and any k > 0:
```
P(|X − μ| ≥ k·σ) ≤ 1/k²
```

Equivalently, for any ε > 0:
```
P(|X − μ| ≥ ε) ≤ Var(X) / ε²
```

**Key property:** This bound holds for **any distribution** — no normality assumption. It is distribution-free.

**Proof (via Markov's inequality):** Markov: P(Y ≥ a) ≤ E[Y]/a for non-negative Y, a > 0. Apply to Y = (X−μ)², a = ε²: P((X−μ)² ≥ ε²) ≤ E[(X−μ)²]/ε² = Var(X)/ε².

**Comparing to the 68-95-99.7 rule (Normal):**

| k | Chebyshev bound | Normal actual |
|---|---|---|
| 1 | P(\|X−μ\| ≥ σ) ≤ 100% | 31.7% |
| 2 | P(\|X−μ\| ≥ 2σ) ≤ 25% | 4.55% |
| 3 | P(\|X−μ\| ≥ 3σ) ≤ 11.1% | 0.27% |

Chebyshev is very loose — useful for theoretical guarantees, not practical anomaly detection.

**Application — Sample size for estimation:** We want P(|X̄ − μ| ≥ ε) ≤ δ. By Chebyshev: σ²/(nε²) ≤ δ → n ≥ σ²/(δε²). This gives a distribution-free sample size guarantee.

**Worked example:** A random variable has mean 50 and standard deviation 10. What is the minimum probability that X is between 30 and 70?

k = (70−50)/10 = 2. P(|X−50| ≥ 20) ≤ 1/4. Therefore P(30 ≤ X ≤ 70) ≥ 1 − 1/4 = 75%.

> **Why / When to use:** Chebyshev gives distribution-free bounds, useful when you cannot assume normality. However, the bounds are often very loose — for normal data, use the empirical rule instead.

**🎯 In the interview:** "How many standard deviations does an observation need to be from the mean to guarantee it's an outlier with probability at most 1%?" — Chebyshev: 1/k² ≤ 0.01 → k ≥ 10. Under normality, k ≈ 2.58 suffices. Chebyshev is much more conservative.

---

## 9. Bias-Variance Decomposition of an Estimator

### 9.1 Statistical Estimator Setup

Let θ be a true unknown parameter, and θ̂ be an estimator (a function of data). We measure quality via **Mean Squared Error (MSE)**:
```
MSE(θ̂) = E[(θ̂ − θ)²]
```

### 9.2 The Decomposition

```
MSE(θ̂) = Bias(θ̂)² + Var(θ̂)
```

where:
```
Bias(θ̂) = E[θ̂] − θ
Var(θ̂)  = E[(θ̂ − E[θ̂])²]
```

**Derivation:**
```
MSE = E[(θ̂ − θ)²]
    = E[(θ̂ − E[θ̂] + E[θ̂] − θ)²]
    = E[(θ̂ − E[θ̂])²] + 2(E[θ̂]−θ)·E[θ̂ − E[θ̂]] + (E[θ̂]−θ)²
```
The middle term is zero (E[θ̂ − E[θ̂]] = 0), leaving:
```
MSE = Var(θ̂) + Bias(θ̂)²
```

(Source: [statlect.com — Mean Squared Error](https://www.statlect.com/glossary/mean-squared-error))

### 9.3 The Bias-Variance Tradeoff in ML

In supervised learning, the MSE of a model's prediction f̂(x) at a point x decomposes as:

```
E[(y − f̂(x))²] = Bias[f̂(x)]² + Var[f̂(x)] + σ²_noise
```

Where:
- **Bias²:** Systematic error from model assumptions. High in simple (underfit) models that can't capture true structure.
- **Variance:** Sensitivity to training data fluctuations. High in complex (overfit) models that memorize training data.
- **σ²_noise:** Irreducible noise in the data — no model can reduce this below the true noise floor.

**The fundamental tradeoff:**

| Model complexity | Bias | Variance | MSE |
|---|---|---|---|
| Too simple (underfit) | High | Low | High |
| Just right | Low | Low | Minimum |
| Too complex (overfit) | Low | High | High |

**Practical implications:**
- **Regularization** (L1/L2) introduces bias but reduces variance → lower MSE in high dimensions.
- **Ensemble methods** (Bagging) reduce variance without much increasing bias.
- **Boosting** reduces bias at the cost of some variance.
- **Cross-validation** estimates the combined effect; it doesn't separately measure bias and variance.

**Worked example:** Estimating μ from X₁, ..., Xₙ i.i.d. N(μ, σ²).

Estimator 1: X̄ = (1/n)Σ Xᵢ  
- Bias = E[X̄] − μ = 0 (unbiased)  
- Var(X̄) = σ²/n  
- MSE = 0 + σ²/n = σ²/n

Estimator 2: μ̂ = c·X̄ where 0 < c < 1 (shrinkage)  
- Bias = E[cX̄] − μ = cμ − μ = (c−1)μ (biased)  
- Var(cX̄) = c²σ²/n  
- MSE = (c−1)²μ² + c²σ²/n

For small σ²/n (large n), X̄ wins. For large σ²/n (small n, large noise), c < 1 can reduce MSE if (c−1)²μ² < (1−c²)σ²/n — this is the James-Stein phenomenon.

**Unbiased estimator of σ²:** Sample variance s² = Σ(Xᵢ − X̄)²/(n−1). The divisor is n−1, not n, to correct for the fact that X̄ is estimated from the same data (Bessel's correction). Using n instead gives a biased (but lower variance) estimator.

**🎯 In the interview:** "Why does regularization help?" — It adds bias (shrinks coefficients toward zero) but substantially reduces variance, giving lower MSE for finite data. "When is an unbiased estimator not optimal?" — When it has high variance; a slightly biased but much lower-variance estimator can win on MSE (bias-variance tradeoff).

---

## 10. 🎯 In the Interview: Common Traps

### Linearity of Expectation
- **Trap:** "Does E[XY] = E[X]·E[Y]?" — Only if X and Y are **independent** (or at least uncorrelated). In general, E[XY] = E[X]·E[Y] + Cov(X, Y). Interviewers love to probe this.

### Correlation vs. Independence
- **Trap:** "If Corr(X,Y) = 0, are they independent?" — **NO**, in general. Zero correlation means no **linear** relationship. Quadratic or other nonlinear dependencies can exist. Exception: jointly normal random variables.

### E[g(X)] vs. g(E[X])
- **Trap:** "What is E[1/X]?" — NOT 1/E[X]. By Jensen (1/x is convex for x>0): E[1/X] ≥ 1/E[X]. The magnitude of the difference depends on Var(X)/E[X]³ (delta method).

### Variance of Difference
- **Trap:** "Is Var(X−Y) = Var(X) − Var(Y)?" — **NO**. Var(X−Y) = Var(X) + Var(Y) − 2Cov(X,Y). Variances never subtract.

### Uncorrelated ≠ Independent (except jointly normal)
- The famous counterexample: X ~ N(0,1), Y = X² — Cov(X,Y) = 0 yet Y is fully determined by X. This fails all tests of independence except correlation.

### Conditional Expectation is a Random Variable
- **Trap:** E[X|Y] is a function of Y — it is itself random. E[E[X|Y]] = E[X] (law of total expectation). Students often confuse E[X|Y=y] (a number, fixed y) with E[X|Y] (random).

### Bias in Sample Variance
- **Trap:** Why divide by n−1, not n? Because X̄ is estimated from the same data. Using n gives Bias = −σ²/n. Using n−1 (Bessel's correction) gives an unbiased estimator.

### Chebyshev vs. Normal Tail Bounds
- Chebyshev: P(|X−μ| ≥ 3σ) ≤ 11.1%  
- Normal: P(|X−μ| ≥ 3σ) ≈ 0.27%  
- Using Chebyshev when you know the distribution is normal wastes enormous precision.

### Bias-Variance in ML
- **Trap:** "Regularization always hurts performance." — False. It introduces bias but reduces variance; for finite data, total MSE often decreases. The optimal regularization strength depends on signal-to-noise ratio.

### Jensen's Inequality Direction
- **Trap:** "For concave g, is E[g(X)] ≥ g(E[X]) or ≤?" — For concave g, E[g(X)] ≤ g(E[X]). Common example: E[log X] ≤ log(E[X]) since log is concave. This underpins AM-GM and information theory results.

---

## Glossary

| Term | Definition |
|---|---|
| Expected Value E[X] | Probability-weighted average of X; long-run mean |
| Linearity of Expectation | E[aX + bY] = aE[X] + bE[Y]; holds without independence |
| LOTUS | E[g(X)] = Σ g(x)P(X=x) or ∫ g(x)f(x)dx — no need to find distribution of g(X) |
| Variance Var(X) | E[(X−μ)²] = E[X²] − (E[X])²; always ≥ 0 |
| Standard Deviation | σ = √Var(X); same units as X |
| Covariance Cov(X,Y) | E[(X−μₓ)(Y−μᵧ)] = E[XY] − E[X]E[Y]; measures co-movement |
| Correlation ρ(X,Y) | Cov(X,Y)/(σₓσᵧ); dimensionless, in [−1,+1] |
| Law of Total Expectation | E[X] = E[E[X\|Y]] (Adam's Law) |
| Law of Total Variance | Var(X) = E[Var(X\|Y)] + Var(E[X\|Y]) (Eve's Law) |
| Moment | E[Xⁿ] (n-th raw moment); centered around mean for central moments |
| MGF | M(t) = E[e^(tX)]; M^(n)(0) = E[Xⁿ] |
| Jensen's Inequality | g convex: g(E[X]) ≤ E[g(X)]; g concave: reversed |
| Chebyshev's Inequality | P(\|X−μ\| ≥ kσ) ≤ 1/k²; distribution-free |
| Bias of Estimator | E[θ̂] − θ; systematic error |
| Variance of Estimator | E[(θ̂ − E[θ̂])²]; random fluctuation |
| MSE | Bias² + Variance; total squared error |
| Bessel's Correction | Dividing by n−1 instead of n to get unbiased sample variance |
| Bias-Variance Tradeoff | Reducing model complexity raises bias, lowers variance; optimum minimizes total MSE |
| Overdispersion | Observed variance exceeds model prediction |
| Delta Method | Approximates Var(g(X)) ≈ [g'(E[X])]² Var(X) using Taylor expansion |

---

## References

1. **DeGroot, M.H. & Schervish, M.J.** — *Probability and Statistics*, 4th ed. Addison-Wesley, 2012. (Chapters 4–5: Expectation, Variance, Covariance.)
2. **Casella, G. & Berger, R.L.** — *Statistical Inference*, 2nd ed. Duxbury, 2002. (Chapter 7: Point Estimation, bias-variance decomposition.)
3. **Yale Statistics (Pollard)** — "Chapter 4: Variances and Covariances." [http://www.stat.yale.edu/~pollard/Courses/241.fall97/Variance.pdf](http://www.stat.yale.edu/~pollard/Courses/241.fall97/Variance.pdf)
4. **probabilitycourse.com** — "Covariance and Correlation." [https://www.probabilitycourse.com/chapter5/5_3_1_covariance_correlation.php](https://www.probabilitycourse.com/chapter5/5_3_1_covariance_correlation.php)
5. **statlect.com** — "Mean Squared Error of an Estimator." [https://www.statlect.com/glossary/mean-squared-error](https://www.statlect.com/glossary/mean-squared-error)
6. **Wikipedia: Jensen's Inequality** — [https://en.wikipedia.org/wiki/Jensen%27s_inequality](https://en.wikipedia.org/wiki/Jensen%27s_inequality)
7. **MachineLearningMastery.com** — "A Gentle Introduction to Jensen's Inequality." [https://machinelearningmastery.com/a-gentle-introduction-to-jensens-inequality/](https://machinelearningmastery.com/a-gentle-introduction-to-jensens-inequality/)
8. **Bookdown: Foundations of Statistics (Neal)** — "Chapter 13: Expectation, Covariance and Correlation." [https://bookdown.org/peter_neal/math4081_notes/Correlation.html](https://bookdown.org/peter_neal/math4081_notes/Correlation.html)
9. **MIT OpenCourseWare 18.650** — Statistics for Applications (Rigollet). (Lecture notes on bias, variance, MSE.)

---

*Part of the [Statistics for Data Science](README.md) section · [Guide Home](../README.md).*
