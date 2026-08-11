# 05 · Sampling, the Central Limit Theorem, and Estimation

A data scientist who cannot reason about *how data were collected*, *what the sampling distribution of their statistic looks like*, and *how to extract parameters from a model* will misinterpret almost every result they produce. This page builds the chain from raw data collection all the way through point and interval estimation, covering the theorems and pitfalls most likely to surface in technical interviews.

> **In plain English:** You can never measure an entire population, so you draw a sample. The Central Limit Theorem (CLT) then tells you that the average of your sample will be approximately normally distributed no matter what shape the original data have — as long as your sample is large enough. That guarantee is what lets you build confidence intervals, run hypothesis tests, and trust MLE estimates in practice.

---

## Table of Contents

1. [Sampling Fundamentals](#1-sampling-fundamentals)
2. [Sampling Distributions and Standard Error](#2-sampling-distributions-and-standard-error)
3. [Law of Large Numbers](#3-law-of-large-numbers)
4. [Central Limit Theorem](#4-central-limit-theorem)
5. [Point Estimation](#5-point-estimation)
   - 5.1 [Desirable Properties of Estimators](#51-desirable-properties-of-estimators)
   - 5.2 [Method of Moments (MoM)](#52-method-of-moments-mom)
   - 5.3 [Maximum Likelihood Estimation (MLE)](#53-maximum-likelihood-estimation-mle)
   - 5.4 [Maximum A Posteriori (MAP) Estimation](#54-maximum-a-posteriori-map-estimation)
6. [Interval Estimation — Confidence Intervals](#6-interval-estimation--confidence-intervals)
7. [Bias–Variance Decomposition of Estimators](#7-biasvariance-decomposition-of-estimators)
8. [Glossary](#glossary)
9. [References](#references)

---

## 1. Sampling Fundamentals

### 1.1 Population vs. Sample

| Concept | Symbol | Definition |
|---|---|---|
| **Population** | N | The complete set of all units of interest (often infinite or impractical to measure fully). |
| **Sample** | n | A subset drawn from the population used to make inferences. |
| **Parameter** | θ (Greek) | A fixed, usually unknown, numerical property of the population (e.g., population mean μ). |
| **Statistic** | θ̂ (hat) | A function of the sample data used to estimate a parameter (e.g., sample mean x̄). |

The goal of **statistical inference** is to use a statistic computed from n observations to say something reliable about the parameter θ that governs the whole population.

---

### 1.2 Sampling Methods

#### Simple Random Sampling (SRS)
Every individual in the population has an **equal and independent** probability of selection. Requires a complete sampling frame (list of all units).

- **Pros:** Unbiased, mathematically tractable, gold standard for inference.
- **Cons:** Expensive and impractical for large dispersed populations; rare subgroups may be missed by chance.

#### Stratified Sampling
Divide the population into non-overlapping **strata** (subgroups sharing a characteristic), then draw SRS within each stratum. Sample size from each stratum is proportional (proportionate stratified) or optimally allocated (Neyman allocation).

- **Pros:** Guarantees representation of each stratum; **reduces sampling variance** compared to SRS — particularly valuable when strata differ substantially on the outcome.
- **Cons:** Requires prior knowledge of stratum membership; costly to administer.

> **Why / When to use:**  
> Use stratified sampling when you know the population has meaningful subgroups (e.g., age bands, geographic regions) and you need estimates for each subgroup *or* when a subgroup is rare and SRS would under-represent it. Stratified sampling always has variance ≤ SRS variance (equality only when strata are identical).

#### Cluster Sampling
Divide the population into clusters (ideally each cluster mirrors the whole population), randomly select entire clusters, and measure every unit inside the selected clusters.

- **Pros:** Dramatically reduces cost when units are geographically dispersed (e.g., schools in a country).
- **Cons:** Units within a cluster tend to be correlated → **design effect** inflates variance relative to SRS. Cluster sampling is generally *less* statistically efficient than SRS.

> **Why / When to use:**  
> Cluster vs. stratified is a common interview question. In stratified sampling you sample *from every stratum*; in cluster sampling you sample *only selected clusters* (the rest get zero representation). Cluster trading lower cost for higher variance; stratified trading more upfront structure for lower variance.

#### Systematic Sampling
Order the population list and select every k-th element (k = N/n), starting from a random start in [1, k].

- **Pros:** Simple to execute; approximately as efficient as SRS when the list is randomly ordered.
- **Cons:** If the list has a periodic pattern with period k, systematic sampling can be catastrophically biased (e.g., sampling only Monday shifts from a weekly roster).

#### Convenience Sampling (Non-Probability)
Sample whoever is easy to reach (e.g., students in a classroom, users who voluntarily fill a survey).

- **Pros:** Fast, cheap.
- **Cons:** High risk of systematic bias; **cannot support valid statistical inference** about the broader population. Should be called out in every analysis.

---

### 1.3 Sampling Bias Types

| Bias Type | Definition | Classic Example |
|---|---|---|
| **Selection bias** | Systematic difference between who gets included vs. excluded from the sample. | Polling only landline telephone owners in 1948 excluded younger, more urban voters → wrong prediction. |
| **Survivorship bias** | Analyzing only "survivors" (units still present at measurement time), ignoring those that dropped out. | Studying only companies that are still operating inflates estimated success rates. |
| **Non-response bias** | Those who respond differ systematically from those who don't. | Health survey respondents may be healthier than non-respondents. |
| **Undercoverage bias** | Some population segments have zero or near-zero probability of selection. | Online survey covers only internet users; excludes elderly or rural populations without internet access. |

> **🎯 In the interview:**  
> Interviewers love "what can go wrong with this data" questions. Mention: (1) *which* bias type is plausible, (2) what direction it pushes estimates, (3) what you would do to mitigate it (e.g., post-stratification weighting for non-response, multiple frames for undercoverage).

---

## 2. Sampling Distributions and Standard Error

### 2.1 Sampling Distribution

If you were to repeat your sampling procedure many times and compute a statistic (say x̄) each time, the resulting distribution of those statistic values is the **sampling distribution** of that statistic.

Key insight: a sampling distribution is **not** the distribution of the data. It is the distribution of a *function of the data* (the statistic) across hypothetical repeated samples.

### 2.2 Standard Error vs. Standard Deviation

| Quantity | Symbol | Measures |
|---|---|---|
| **Standard deviation** (population) | σ | Spread of *individual observations* around μ. |
| **Standard deviation** (sample) | s | Estimate of σ from data; spread of *individual observations* around x̄. |
| **Standard error of the mean** | SE = σ/√n | Spread of *sample means* (x̄ values) around μ across repeated samples. |

$$\text{SE}(\bar{X}) = \frac{\sigma}{\sqrt{n}}$$

When σ is unknown (almost always), replace it with the sample standard deviation s:

$$\widehat{\text{SE}}(\bar{X}) = \frac{s}{\sqrt{n}}$$

**Intuition:** Averaging over more data points cancels noise; the mean becomes more precise at rate √n (not n).

> **🎯 In the interview — classic trap:**  
> "SE shrinks as 1/√n, not 1/n. To halve the SE you need to *quadruple* the sample size."  
> Also: SE is a property of the *estimator*, not the data. A large SE means the estimator is unstable across samples, not that the data are noisy (though those are often related).

---

## 3. Law of Large Numbers

### 3.1 Statement

Let X₁, X₂, … be i.i.d. random variables with finite mean μ = E[X].

**Weak LLN (Khinchin):** The sample mean x̄ₙ converges *in probability* to μ as n → ∞:

$$\bar{X}_n \xrightarrow{p} \mu$$

That is, for any ε > 0: P(|x̄ₙ − μ| > ε) → 0 as n → ∞.

**Strong LLN (Kolmogorov):** Under the same conditions (finite mean suffices for weak; strong additionally needs finite variance or just finite first moment in some formulations), x̄ₙ converges *almost surely* to μ:

$$\bar{X}_n \xrightarrow{a.s.} \mu$$

Almost sure convergence is stronger: the event {x̄ₙ → μ} has probability 1 (the exceptional paths where it doesn't converge form a set of measure zero).

### 3.2 Intuition

Roll a fair die. On any single roll the result varies. Average 10 rolls and you get something near 3.5. Average 10,000 rolls and you're essentially *at* 3.5. The LLN is the formal guarantee behind this everyday experience.

### 3.3 What LLN Does NOT Say

- It does **not** say early outcomes are "corrected" by later ones (the gambler's fallacy).
- It does **not** guarantee the distribution looks normal (that's the CLT, below).
- It does **not** apply when the mean is undefined (e.g., Cauchy distribution has no mean → the sample mean wanders forever).

> **🎯 In the interview:**  
> The LLN is often conflated with the CLT. LLN → *where* the mean lands (converges to μ). CLT → *what shape* the distribution of that mean has (approximately normal). Both are needed to do inference.

---

## 4. Central Limit Theorem

### 4.1 Statement

Let X₁, …, Xₙ be i.i.d. with mean μ and **finite variance** σ². Define the standardized sample mean:

$$Z_n = \frac{\bar{X}_n - \mu}{\sigma / \sqrt{n}}$$

Then as n → ∞:

$$Z_n \xrightarrow{d} \mathcal{N}(0, 1)$$

Equivalently, for any fixed n:

$$\bar{X}_n \stackrel{\text{approx.}}{\sim} \mathcal{N}\!\left(\mu,\; \frac{\sigma^2}{n}\right)$$

This holds regardless of the shape of the population distribution — as long as the conditions below are met.

### 4.2 Conditions (and When the CLT Fails)

| Condition | What happens when violated |
|---|---|
| **Independent observations** | CLT still holds with weaker dependence (e.g., stationary mixing processes), but proof is more involved. If observations are strongly dependent (e.g., time series with long memory), the standard CLT fails and the rate may be slower. |
| **Finite variance (σ² < ∞)** | **Heavy-tailed distributions** (e.g., Pareto with shape α ≤ 2, Cauchy) have infinite variance → sample mean does not converge to Normal. The appropriate limit theorem involves **stable distributions** (Lévy, Cauchy). This matters for financial returns and network degree distributions. |
| **Identically distributed (i.i.d.)** | Lindeberg–Feller CLT generalizes to non-identical distributions as long as no single observation dominates (Lindeberg condition). |

### 4.3 The n ≥ 30 Rule of Thumb — and Its Caveats

The often-cited "n ≥ 30" heuristic is a historical convenience, **not a statistical law**. As noted in research by Tim Hesterberg at Google ([research.google.com](https://research.google.com/pubs/archive/34906.pdf)), the n ≥ 30 rule is neither necessary nor sufficient:

- A **nearly symmetric, thin-tailed** distribution may be well-approximated with n = 10.
- A **strongly right-skewed** distribution (e.g., income) may need n = 100–500 before the normal approximation is reliable.
- A **heavy-tailed** distribution with finite variance (e.g., t(3)) needs n much larger than 30.
- A distribution with **infinite variance** (Cauchy) never converges to Normal.

Practical guidance: plot the data, assess skewness and tailweight, and use **bootstrap confidence intervals** when normality of the sampling distribution is in doubt.

### 4.4 Worked Intuition Example

Population: a Bernoulli(p = 0.1) — very skewed (90% zeros, 10% ones).

| n | Shape of x̄ₙ distribution |
|---|---|
| 5 | Very skewed; looks nothing like Normal |
| 30 | Rough bell shape but noticeably right-skewed |
| 100 | Good Normal approximation |
| 1000 | Excellent Normal approximation |

The CLT ensures the *mean* approaches Normal, even though individual observations are 0 or 1. This is what lets us use z-tests for proportions (with appropriate n).

> **🎯 In the interview — the single most important CLT point:**  
> "The CLT is about the sampling distribution of the **mean** (or sum), *not* about the distribution of the raw data. Even after collecting a million data points, a skewed population distribution is still skewed. The mean of those million points is, however, very accurately approximated as Normal — which is what lets us build CIs and run tests."

---

## 5. Point Estimation

A **point estimator** θ̂ is a function of the data X₁, …, Xₙ that produces a single value intended to approximate the true parameter θ. The realized value from a specific dataset is called the **estimate**.

### 5.1 Desirable Properties of Estimators

#### Unbiasedness
$$\text{Bias}(\hat{\theta}) = \mathbb{E}[\hat{\theta}] - \theta = 0$$

The sample mean x̄ is unbiased for μ. The sample variance s² = Σ(xᵢ − x̄)²/(n−1) is unbiased for σ² (dividing by n gives a biased estimator; the 1/(n−1) Bessel correction removes the bias).

#### Consistency
θ̂ₙ converges in probability to θ as n → ∞:

$$\hat{\theta}_n \xrightarrow{p} \theta$$

A biased estimator can still be consistent if the bias vanishes as n grows.

#### Efficiency
Among all unbiased estimators, the one with the **smallest variance** is the most efficient. The **Cramér–Rao Lower Bound (CRLB)** provides a lower bound on the variance of any unbiased estimator:

$$\text{Var}(\hat{\theta}) \geq \frac{1}{I(\theta)}$$

where I(θ) is the **Fisher information**. An estimator achieving the CRLB is called **efficient** (or UMVUE — uniformly minimum variance unbiased estimator).

#### Sufficiency
A statistic T(X) is **sufficient** for θ if the conditional distribution of the data given T does not depend on θ. Sufficiency means T captures all the information in the data about θ — nothing more is lost. By the Rao–Blackwell theorem, any unbiased estimator can be improved (variance reduced) by conditioning on a sufficient statistic.

---

### 5.2 Method of Moments (MoM)

**Procedure:** Set the first k *population moments* equal to the corresponding *sample moments* and solve for the k unknown parameters. (Karl Pearson introduced this in 1894 — [Wikipedia](https://en.wikipedia.org/wiki/Method_of_moments_(statistics)).)

$$\mathbb{E}[X^r] = \frac{1}{n}\sum_{i=1}^n x_i^r, \quad r = 1, 2, \ldots, k$$

**Example — Normal(μ, σ²):**

- 1st moment: E[X] = μ → set x̄ = μ̂_MoM → **μ̂ = x̄**
- 2nd central moment: Var(X) = σ² → set sample variance (biased, 1/n) = σ̂² → **σ̂² = (1/n)Σ(xᵢ − x̄)²**

(Note MoM gives biased σ̂²; MLE gives the same biased estimate in this case.)

**Properties of MoM:**
- Consistent and asymptotically normal.
- Often easy to compute in closed form.
- Can be inefficient (higher variance than MLE in many cases).
- Can yield estimates outside valid parameter ranges (e.g., negative variance estimates in rare edge cases).

> **Why / When to use MoM vs. MLE:**  
> MoM is preferred when the likelihood is hard to write or optimize (e.g., mixture models for initialization). MLE is preferred when efficiency matters and the likelihood is tractable — MLE achieves the CRLB asymptotically (see §5.3).

---

### 5.3 Maximum Likelihood Estimation (MLE)

#### Core Idea
Find the parameter value θ that makes the observed data *most probable*. Given i.i.d. data X₁ = x₁, …, Xₙ = xₙ:

$$\mathcal{L}(\theta) = \prod_{i=1}^n f(x_i \mid \theta)$$

Because products of small probabilities underflow, maximize the **log-likelihood** instead (equivalent, since log is monotone):

$$\ell(\theta) = \log \mathcal{L}(\theta) = \sum_{i=1}^n \log f(x_i \mid \theta)$$

$$\hat{\theta}_{\text{MLE}} = \arg\max_\theta\; \ell(\theta)$$

Solve by setting the score equation to zero: ∂ℓ/∂θ = 0.

---

#### Worked Example 1: Bernoulli(p)

Data: n binary observations, of which k are 1s.

$$\ell(p) = k \log p + (n - k) \log(1 - p)$$

$$\frac{d\ell}{dp} = \frac{k}{p} - \frac{n-k}{1-p} = 0$$

Solving: **p̂_MLE = k/n** — the empirical proportion. Intuitive and unbiased.

---

#### Worked Example 2: Normal(μ, σ²), both unknown

$$\ell(\mu, \sigma^2) = -\frac{n}{2}\log(2\pi\sigma^2) - \frac{1}{2\sigma^2}\sum_{i=1}^n (x_i - \mu)^2$$

Setting ∂ℓ/∂μ = 0: **μ̂_MLE = x̄**

Setting ∂ℓ/∂σ² = 0: **σ̂²_MLE = (1/n) Σ(xᵢ − x̄)²** (biased — note n, not n−1).

This is the one well-known case where MLE gives a biased estimator; the UMVUE for σ² uses n−1.

---

#### Key Properties of MLE

**1. Invariance:** If θ̂ is the MLE of θ, then for any function g, the MLE of g(θ) is g(θ̂).  
*Example:* MLE of σ (the standard deviation) is √(σ̂²_MLE) — just plug in. You do not need to redo the optimization.  
Source: [Academia.edu — Invariance Properties of MLEs](https://www.academia.edu/24458382/Invariance_Properties_of_Maximum_Likelihood_Estimators).

**2. Consistency:** θ̂_MLE → θ in probability as n → ∞ (under regularity conditions).

**3. Asymptotic Normality (Fisher):** Under regularity conditions:

$$\sqrt{n}\,(\hat{\theta}_{\text{MLE}} - \theta) \xrightarrow{d} \mathcal{N}\!\left(0,\; [I(\theta)]^{-1}\right)$$

where I(θ) is the Fisher information. Equivalently, the MLE is asymptotically efficient — it achieves the CRLB as n → ∞.  
Source: [Gregory Gundersen — Asymptotic Normality of MLE](https://gregorygundersen.com/blog/2019/11/28/asymptotic-normality-mle/).

**4. Asymptotic Efficiency:** Among all consistent estimators, MLE has the smallest asymptotic variance.

**Caveats:**
- MLE can overfit with small samples (no regularization).
- Requires the model to be correctly specified; MLE under misspecification converges to the "closest" parameter under KL divergence.
- For finite mixtures, the likelihood can be unbounded (degenerate solutions) — need constraints or regularization.

> **🎯 In the interview:**  
> "MLE of a function of θ is that function of θ̂ (invariance). This means MLE of the odds ratio p/(1−p) when p̂ = 0.3 is 0.3/0.7 = 3/7 directly."  
> Also: "MLE of σ² in a normal model divides by n (biased), but we usually report s² (divides by n−1) for unbiasedness."

---

### 5.4 Maximum A Posteriori (MAP) Estimation

MAP extends MLE by incorporating a **prior distribution** π(θ) on the parameter. By Bayes' theorem:

$$\text{posterior} \propto \text{likelihood} \times \text{prior}$$
$$p(\theta \mid \mathbf{x}) \propto \mathcal{L}(\theta) \cdot \pi(\theta)$$

MAP selects the **mode of the posterior**:

$$\hat{\theta}_{\text{MAP}} = \arg\max_\theta \left[\ell(\theta) + \log \pi(\theta)\right]$$

#### Connection to Regularization

| Prior on θ | MAP optimization | ML equivalent |
|---|---|---|
| **Gaussian prior** N(0, τ²) | Adds −θ²/(2τ²) to log-likelihood | **L2 / Ridge regularization** (penalize large weights) |
| **Laplace prior** Laplace(0, b) | Adds −|θ|/b to log-likelihood | **L1 / Lasso regularization** (induces sparsity) |

The regularization coefficient λ in ridge regression equals σ²/τ² — it encodes how strongly you believe weights should be near zero.  
Source: [Towards Data Science — A Bayesian Take on Regularization](https://towardsdatascience.com/a-bayesian-take-on-model-regularization-9356116b6457/).

#### MAP vs. MLE vs. MoM

| Property | MoM | MLE | MAP |
|---|---|---|---|
| Uses prior? | No | No | Yes |
| Bayesian? | No | No | Yes (mode of posterior) |
| Efficient asymptotically? | Sometimes | Yes (achieves CRLB) | Yes (prior vanishes as n→∞) |
| Equivalent to? | Matching moments | — | MLE + regularization |
| Good small-n? | Moderate | Can overfit | Yes, if prior is sensible |

> **Why / When to use MAP vs. MLE:**  
> When you have small data and a reasonable prior (e.g., you know model weights should be near zero), MAP prevents overfitting. As n → ∞, the prior is overwhelmed by data and MAP → MLE. MAP is still a point estimate — for full uncertainty quantification you want the full posterior (Bayesian inference), not just its mode.

> **🎯 In the interview:**  
> "L2-regularized logistic regression is doing MAP inference with a Gaussian prior on the weights. L1-regularized logistic regression is doing MAP with a Laplace prior. The sparsity from Lasso arises because the Laplace prior has a sharp peak at zero — many weights are MAP-estimated to exactly zero."

---

## 6. Interval Estimation — Confidence Intervals

### 6.1 What Is a Confidence Interval?

A **confidence interval (CI)** at level (1−α)×100% is a procedure that, if repeated many times on fresh samples, would produce an interval containing the true parameter θ at least (1−α)×100% of the time.

Formally, for random intervals [L(X), U(X)]:

$$P\left(L(\mathbf{X}) \leq \theta \leq U(\mathbf{X})\right) = 1 - \alpha$$

### 6.2 The Critical Misinterpretation

> **❌ WRONG:** "There is a 95% probability that the true parameter lies in [2.1, 4.7]."  
> **✓ CORRECT:** "If we were to repeat this sampling procedure many times, 95% of the resulting intervals would contain the true parameter."

The distinction: in frequentist statistics, θ is **fixed** (not random). Once you compute a specific interval [2.1, 4.7], it either contains θ or it doesn't — no probability is involved for that realized interval. The "95%" characterizes the long-run behavior of the *procedure*, not the probability of this specific interval.  
Source: [PSU STAT 200 — Interpreting Confidence Intervals](https://online.stat.psu.edu/stat200/lesson/4/4.2/4.2.1).

> **🎯 In the interview:**  
> This is one of the most commonly tested statistical misconceptions. Be explicit: "A 95% CI means the *process* captures θ 95% of the time — not that *this* specific interval has a 95% chance of containing θ. Under Bayesian inference, the analog — a 95% **credible interval** — does allow you to say 'there is a 95% posterior probability that θ is in this interval'."

### 6.3 CI for a Population Mean

#### Case 1: Population σ known → Z-interval

$$\bar{x} \pm z_{\alpha/2} \cdot \frac{\sigma}{\sqrt{n}}$$

Common critical values: z₀.₀₅ = 1.645 (90%), **z₀.₀₂₅ = 1.96** (95%), z₀.₀₀₅ = 2.576 (99%).

#### Case 2: Population σ unknown → t-interval

$$\bar{x} \pm t_{\alpha/2,\, n-1} \cdot \frac{s}{\sqrt{n}}$$

The t-distribution with (n−1) degrees of freedom has heavier tails than Normal, reflecting the extra uncertainty from estimating σ. As n → ∞, t → Z.

> **Why / When to use z vs. t:**  
> - If σ is **known** (rare outside physics/quality control) or n is very large (n > 100 as a rule of thumb): use z.  
> - If σ is **unknown** and n is small: **always use t** — it is conservative in exactly the right direction (wider intervals = better coverage).  
> - The t-interval is robust to mild non-normality of the data when n ≥ 30, thanks to the CLT; it is not robust with heavy tails and very small n.

### 6.4 CI for a Proportion

For a binary outcome with observed proportion p̂ = k/n, the **Wald interval** is:

$$\hat{p} \pm z_{\alpha/2} \sqrt{\frac{\hat{p}(1-\hat{p})}{n}}$$

**Caveats with Wald:** Poor coverage when p̂ is near 0 or 1, or when n is small. Better alternatives:
- **Wilson score interval** (recommended by most statisticians for p close to 0 or 1).
- **Agresti–Coull** ("add 2 successes and 2 failures" before Wald formula) — easy to compute, good coverage.
- **Clopper–Pearson (exact)** — conservative but exact.

### 6.5 Margin of Error and Width

The **margin of error (MoE)** is the half-width of a CI:

$$\text{MoE} = z_{\alpha/2} \cdot \frac{\sigma}{\sqrt{n}}$$

**Effect on CI width:**

| Change | Effect on Width |
|---|---|
| Increase confidence level (e.g., 95% → 99%) | **Wider** (larger critical value) |
| Increase n | **Narrower** (SE shrinks as 1/√n) |
| Higher population variability (larger σ) | **Wider** |

To achieve a target MoE = E:

$$n = \left(\frac{z_{\alpha/2} \cdot \sigma}{E}\right)^2$$

For proportions (worst case p = 0.5): n = z²α/2 / (4E²). This is how sample sizes are determined before a study.

### 6.6 Bootstrap Confidence Intervals

When the sampling distribution of a statistic is not well-approximated by Normal (small n, complex statistic like a median or correlation), the **bootstrap** provides a non-parametric CI:

1. Resample n observations with replacement from your data B times (B = 1000–10000).
2. Compute the statistic on each resample → B bootstrap estimates.
3. Use the 2.5th and 97.5th percentiles of those estimates as the 95% bootstrap percentile CI.

The bootstrap is model-free and requires only that your sample is representative of the population.

---

## 7. Bias–Variance Decomposition of Estimators

For any estimator θ̂ of θ, the **Mean Squared Error (MSE)** decomposes as:

$$\text{MSE}(\hat{\theta}) = \mathbb{E}\left[(\hat{\theta} - \theta)^2\right] = \text{Bias}(\hat{\theta})^2 + \text{Var}(\hat{\theta})$$

where:
- **Bias(θ̂) = E[θ̂] − θ** (systematic error: on average, how far off are we?)
- **Var(θ̂) = E[(θ̂ − E[θ̂])²]** (random error: how much does the estimator fluctuate?)

Source: [Stanford Encyclopedia of Philosophy — Bias–Variance Decomposition](https://plato.stanford.edu/entries/bounded-rationality/bias-variance-decomp.html).

### Implications

- An unbiased estimator minimizes MSE only if it also has minimum variance — unbiasedness is not sufficient.
- A **biased but low-variance** estimator can have smaller MSE than an unbiased high-variance one. This is the justification for ridge regression: introducing bias by shrinking coefficients often more than compensates with reduced variance, particularly when predictors are correlated.
- **Consistency** ≠ unbiasedness. An estimator can be biased at every finite n yet have MSE → 0 (i.e., converge to θ).

### Mini-Example: Comparing Two Estimators of σ²

| Estimator | Formula | Bias | Variance | MSE (Normal model) |
|---|---|---|---|---|
| s² (UMVUE) | Σ(xᵢ−x̄)²/(n−1) | 0 | 2σ⁴/(n−1) | 2σ⁴/(n−1) |
| σ̂²_MLE | Σ(xᵢ−x̄)²/n | −σ²/n | 2σ⁴(n−1)/n² | σ⁴(2n−1)/n² |

For small n, MLE has smaller MSE than s² despite being biased — the variance reduction outweighs the bias penalty.

> **🎯 In the interview:**  
> "MSE = Bias² + Variance is the fundamental identity. This is why regularization (which introduces bias) can improve prediction: if it reduces variance enough, total MSE falls. The same decomposition explains why cross-validation is needed — training error only measures bias, not variance."

---

## Glossary

| Term | Definition |
|---|---|
| **Population** | The entire set of units about which inference is desired. |
| **Sample** | A subset of the population from which data are collected. |
| **Parameter (θ)** | A fixed unknown numerical property of the population. |
| **Statistic (θ̂)** | A function of sample data used to estimate a parameter. |
| **Sampling distribution** | Distribution of a statistic across all possible samples of size n. |
| **Standard error (SE)** | Standard deviation of the sampling distribution of a statistic; SE(x̄) = σ/√n. |
| **Law of Large Numbers (LLN)** | The sample mean converges to the population mean as n → ∞. |
| **Central Limit Theorem (CLT)** | The standardized sample mean converges in distribution to N(0,1) as n → ∞, given finite variance. |
| **Unbiased estimator** | E[θ̂] = θ; no systematic error on average. |
| **Consistent estimator** | θ̂ → θ in probability as n → ∞. |
| **Efficient estimator** | Unbiased estimator achieving the Cramér–Rao Lower Bound. |
| **Sufficient statistic** | Captures all information in the data about the parameter. |
| **Likelihood L(θ)** | Probability of observed data as a function of the parameter θ. |
| **Log-likelihood ℓ(θ)** | Natural log of the likelihood; maximized in practice for numerical stability. |
| **MLE** | Parameter value maximizing the likelihood of observed data. |
| **MAP** | Parameter value maximizing the posterior; MLE + log prior. |
| **Fisher information I(θ)** | Expected squared score; measures how much data can tell us about θ. |
| **Confidence interval (CI)** | An interval procedure such that (1−α)×100% of realized intervals cover θ. |
| **Margin of error** | Half-width of a CI = z_{α/2} × SE. |
| **MSE** | Mean Squared Error = Bias² + Variance. |
| **Bootstrap CI** | Non-parametric CI based on resampling with replacement. |
| **Survivorship bias** | Analyzing only entities that "survived" selection, ignoring those lost. |
| **Design effect** | Ratio of actual variance under a sampling design to variance under SRS. |

---

## References

1. **Hesterberg, T. (Google Research).** *"It's Time To Retire the n ≥ 30 Rule."* [research.google.com/pubs/archive/34906.pdf](https://research.google.com/pubs/archive/34906.pdf) — Critical analysis of the n ≥ 30 rule of thumb.

2. **PSU STAT 200.** *"Interpreting Confidence Intervals."* [online.stat.psu.edu/stat200/lesson/4/4.2/4.2.1](https://online.stat.psu.edu/stat200/lesson/4/4.2/4.2.1) — Authoritative source on the frequentist CI interpretation.

3. **Gundersen, G.** *"Asymptotic Normality of MLE."* [gregorygundersen.com/blog/2019/11/28/asymptotic-normality-mle/](https://gregorygundersen.com/blog/2019/11/28/asymptotic-normality-mle/) — Derivation of MLE asymptotic normality.

4. **Duke University SAMSI.** *"Likelihood and Maximum Likelihood Estimation."* [stat.duke.edu/~sayan/SAMSI/lec/411notes03.pdf](https://www2.stat.duke.edu/~sayan/SAMSI/lec/411notes03.pdf) — Lecture notes covering MLE properties.

5. **Wikipedia — Method of Moments (Statistics).** [en.wikipedia.org/wiki/Method_of_moments_(statistics)](https://en.wikipedia.org/wiki/Method_of_moments_(statistics)) — Definition and history (Karl Pearson, 1894).

6. **Towards Data Science — Bayesian Regularization.** *"A Bayesian Take on Model Regularization."* [towardsdatascience.com/a-bayesian-take-on-model-regularization-9356116b6457](https://towardsdatascience.com/a-bayesian-take-on-model-regularization-9356116b6457/) — MAP ↔ L1/L2 connection.

7. **Stanford Encyclopedia of Philosophy.** *"Bias–Variance Decomposition of MSE."* [plato.stanford.edu/entries/bounded-rationality/bias-variance-decomp.html](https://plato.stanford.edu/entries/bounded-rationality/bias-variance-decomp.html)

8. **Scribbr — Sampling Methods.** [scribbr.com/methodology/sampling-methods/](https://www.scribbr.com/methodology/sampling-methods/) — Accessible reference on sampling design types and tradeoffs.

9. **Statistics LibreTexts — Data Sampling.** [stats.libretexts.org/Courses/Queensborough_Community_College/MA336:_Statistics/01:_Introduction_to_Statistical_Studies/1.10:_Data_Sampling_and_Variation_in_Data_and_Sampling](https://stats.libretexts.org/Courses/Queensborough_Community_College/MA336:_Statistics/01:_Introduction_to_Statistical_Studies/1.10:_Data_Sampling_and_Variation_in_Data_and_Sampling) — Introductory reference on probability sampling designs.

10. **Econometrics Blog.** *"Thirty Isn't the Magic Number."* [econometrics.blog/post/thirty-isn-t-the-magic-number/](https://www.econometrics.blog/post/thirty-isn-t-the-magic-number/) — Critique of the n ≥ 30 heuristic with simulation evidence.

---

*Part of the [Statistics for Data Science](README.md) section · [Guide Home](../README.md).*
