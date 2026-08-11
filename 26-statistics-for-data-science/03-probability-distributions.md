# Probability Distributions

A probability distribution is a mathematical function that describes the likelihood of all possible outcomes of a random variable. Mastering distributions is essential for data science: every statistical model, every Bayesian prior, every hypothesis test, and every generative model is built on them.

> **In plain English:** A distribution is just the recipe that tells you how probability "weight" is spread across possible values. Know the recipe, know the model.

## Table of Contents

1. [Fundamentals: Discrete vs. Continuous](#1-fundamentals-discrete-vs-continuous)
2. [Discrete Distributions](#2-discrete-distributions)
   - [Bernoulli](#21-bernoulli)
   - [Binomial](#22-binomial)
   - [Poisson (+ Poisson Approximation of Binomial)](#23-poisson)
   - [Geometric](#24-geometric)
   - [Negative Binomial](#25-negative-binomial)
   - [Hypergeometric](#26-hypergeometric)
   - [Discrete Uniform](#27-discrete-uniform)
   - [Multinomial](#28-multinomial)
3. [Continuous Distributions](#3-continuous-distributions)
   - [Continuous Uniform](#31-continuous-uniform)
   - [Normal / Gaussian](#32-normalgaussian)
   - [Exponential](#33-exponential)
   - [Gamma](#34-gamma)
   - [Beta](#35-beta)
   - [Log-Normal](#36-log-normal)
   - [Student's t](#37-students-t)
   - [Chi-Square](#38-chi-square)
   - [F Distribution](#39-f-distribution)
   - [Weibull](#310-weibull)
   - [Pareto](#311-pareto)
4. [Relationships Between Distributions](#4-relationships-between-distributions)
5. [Decision Table: What to Model → Which Distribution](#5-decision-table)
6. [🎯 In the Interview: Common Traps](#6-in-the-interview-common-traps)
7. [Glossary](#glossary)
8. [References](#references)

---

## 1. Fundamentals: Discrete vs. Continuous

| Property | Discrete | Continuous |
|---|---|---|
| Random variable values | Countable (integers) | Uncountable (real intervals) |
| Probability function | PMF: P(X = x) | PDF: f(x), where P(a ≤ X ≤ b) = ∫f(x)dx |
| Sum/integral | Σ P(X = x) = 1 | ∫ f(x) dx = 1 |
| P(X = exact value) | Can be > 0 | Always 0 (use intervals) |

**CDF** (cumulative distribution function) F(x) = P(X ≤ x) applies to both and is always non-decreasing, right-continuous, with F(−∞) = 0 and F(+∞) = 1.

---

## 2. Discrete Distributions

### 2.1 Bernoulli

**What it models:** A single binary trial with probability p of "success" (1) and 1−p of "failure" (0).

**Parameter:** p ∈ (0, 1)

**PMF:**
```
P(X = x) = p^x · (1−p)^(1−x),   x ∈ {0, 1}
```
Equivalently: P(X=1) = p, P(X=0) = 1−p.

**Mean:** E[X] = p

**Variance:** Var(X) = p(1−p)

**Key properties:**
- Maximum variance at p = 0.5 (most uncertain).
- Building block for Binomial (sum of n Bernoulli trials).

**Real example:** Whether a single email is spam (1) or not (0).

> **Why / When to use:** Use Bernoulli for any single yes/no event. If you repeat it n times independently, escalate to Binomial.

**🎯 In the interview:** Interviewers sometimes ask "what is the variance when p=0?" — it's 0 (no randomness). Also: Bernoulli is a special case of Binomial with n=1.

---

### 2.2 Binomial

**What it models:** The number of successes in n independent Bernoulli(p) trials.

**Parameters:** n (number of trials, positive integer), p ∈ (0, 1)

**PMF:**
```
P(X = k) = C(n, k) · p^k · (1−p)^(n−k),   k = 0, 1, ..., n
```
where C(n, k) = n! / (k!(n−k)!) is the binomial coefficient.

**Mean:** E[X] = np

**Variance:** Var(X) = np(1−p)

**Key properties:**
- Sum of n independent Bernoulli(p) RVs.
- Symmetric when p = 0.5; right-skewed when p is small, left-skewed when p is large.
- As n → ∞ with p fixed, Binomial(n, p) → Normal(np, np(1−p)) by CLT.
- As n → ∞, p → 0 with np = λ fixed, Binomial(n, p) → Poisson(λ).

**Real example:** Number of defective items in a batch of 100, where each has a 2% defect rate: X ~ Binomial(100, 0.02).

> **Why / When to use:** Use Binomial when: (1) fixed number of trials n, (2) each trial is independent, (3) constant probability p per trial, (4) two outcomes only. Violating any of these invalidates the model.

**Worked mini-example:** You flip a fair coin 10 times. P(exactly 3 heads) = C(10,3) · 0.5³ · 0.5⁷ = 120 · (1/128) · (1/128) ≈ 0.117.

**🎯 In the interview:** Common trap — "can we add Binomials?" Yes, if same p: Bin(n₁,p) + Bin(n₂,p) = Bin(n₁+n₂, p). If different p, you cannot simply add.

---

### 2.3 Poisson

**What it models:** The number of events occurring in a fixed interval of time or space, given events occur at constant average rate λ and independently of each other.

**Parameter:** λ > 0 (rate = mean number of events per interval)

**PMF:**
```
P(X = k) = (e^(−λ) · λ^k) / k!,   k = 0, 1, 2, ...
```

**Mean:** E[X] = λ

**Variance:** Var(X) = λ  ← mean equals variance (key diagnostic!)

**Key properties:**
- Support is all non-negative integers (no upper bound).
- Sum of independent Poisson(λ₁) + Poisson(λ₂) = Poisson(λ₁+λ₂).
- For large λ, Poisson(λ) ≈ Normal(λ, λ) by CLT.

**Real example:** Number of customer support tickets per hour; number of typos per page; radioactive decay counts per second.

#### Poisson Approximation of Binomial

When n is large and p is small such that np = λ is moderate, Binomial(n, p) ≈ Poisson(λ = np).

**Rule of thumb:** Use Poisson approximation when n ≥ 20 and p ≤ 0.05; the approximation is excellent when n ≥ 100 and np ≤ 10. (Source: [Oxford Emory Math Center](https://mathcenter.oxford.emory.edu/site/math117/connectingPoissonAndBinomial/))

**Why it works:** As n → ∞, p → 0 with np = λ fixed:
```
C(n,k) · p^k · (1−p)^(n−k)  →  e^(−λ) · λ^k / k!
```

**Worked mini-example:** A factory produces 1,000 parts per day; each part has 0.001 probability of being defective. λ = 1000 × 0.001 = 1. P(0 defects) ≈ e⁻¹ ≈ 0.368.

> **Why / When to use Poisson vs Binomial:** Use Poisson when n is large and p is tiny (rare events). Use Binomial when n is small/fixed and p is not tiny.

**🎯 In the interview:** The most common trap: "Does Poisson assume independence?" YES — events must occur independently at a constant rate. Clustered events (e.g., earthquakes triggering aftershocks) violate this. Another trap: mean = variance in Poisson; if your data's variance >> mean, consider Negative Binomial (overdispersion).

---

### 2.4 Geometric

**What it models:** The number of trials until the first success in a sequence of independent Bernoulli(p) trials.

**Parameter:** p ∈ (0, 1)

**PMF (number of trials version):**
```
P(X = k) = (1−p)^(k−1) · p,   k = 1, 2, 3, ...
```
(Alternative "number of failures before first success": P(X = k) = (1−p)^k · p, k = 0, 1, 2, ...)

**Mean:** E[X] = 1/p

**Variance:** Var(X) = (1−p) / p²

**Key property — Memorylessness:**
```
P(X > m + n | X > m) = P(X > n)
```
Past failures give no information about future trials. This is the discrete analog of the exponential distribution's memorylessness.

**Real example:** Number of cold calls until a sale; number of rolls of a die until a "6" appears (p = 1/6, expected rolls = 6).

**🎯 In the interview:** Geometric is the only discrete memoryless distribution. Interviewers often ask you to derive this property or connect it to the exponential.

---

### 2.5 Negative Binomial

**What it models:** The number of trials until the r-th success in independent Bernoulli(p) trials. Generalizes Geometric (r=1).

**Parameters:** r > 0 (number of successes required), p ∈ (0,1)

**PMF (number of trials):**
```
P(X = k) = C(k−1, r−1) · p^r · (1−p)^(k−r),   k = r, r+1, ...
```

**Mean:** E[X] = r/p

**Variance:** Var(X) = r(1−p) / p²

**Key property:** Variance > Mean (overdispersion). This makes it ideal for count data where Poisson's mean=variance assumption fails.

**Real example:** Number of insurance claims per policyholder (overdispersed count); number of A/B test visitors until r conversions.

> **Why / When to use:** When modeling count data with variance > mean (overdispersion), use Negative Binomial instead of Poisson.

**🎯 In the interview:** "Why not just use Poisson for count data?" — Poisson forces Var = Mean. Real count data is often overdispersed; Negative Binomial has an extra parameter r to model that extra variation.

---

### 2.6 Hypergeometric

**What it models:** The number of successes in n draws WITHOUT replacement from a finite population of size N containing K successes.

**Parameters:** N (population size), K (number of successes in population), n (draws)

**PMF:**
```
P(X = k) = [C(K, k) · C(N−K, n−k)] / C(N, n),   k = max(0, n+K−N) to min(n, K)
```

**Mean:** E[X] = nK/N

**Variance:** Var(X) = n · (K/N) · (1 − K/N) · (N−n)/(N−1)

Note: The factor (N−n)/(N−1) is the **finite population correction** — variance is smaller than Binomial because sampling without replacement reduces uncertainty.

**Real example:** Drawing 5 cards from a 52-card deck; number of defectives in a quality control sample; number of women on a randomly selected committee.

> **Why / When to use:** Use Hypergeometric when sampling without replacement from a finite population. As N → ∞ or n/N → 0, it converges to Binomial(n, K/N).

**🎯 In the interview:** The key difference from Binomial is the without-replacement aspect. If sampling with replacement, use Binomial.

---

### 2.7 Discrete Uniform

**What it models:** Each of n equally likely outcomes.

**PMF:**
```
P(X = k) = 1/n,   k ∈ {a, a+1, ..., b},   n = b − a + 1
```

**Mean:** E[X] = (a + b) / 2

**Variance:** Var(X) = (n² − 1) / 12 = [(b−a+1)² − 1] / 12

**Real example:** Fair die roll (a=1, b=6); random integer selection; shuffled card index.

---

### 2.8 Multinomial

**What it models:** Generalization of Binomial to k > 2 categories. Counts the number of outcomes in each category across n trials, where each trial independently falls in category i with probability pᵢ.

**Parameters:** n (trials), p₁, p₂, ..., pₖ with Σpᵢ = 1

**PMF:**
```
P(X₁=x₁, X₂=x₂, ..., Xₖ=xₖ) = n! / (x₁! x₂! ··· xₖ!) · p₁^x₁ · p₂^x₂ ··· pₖ^xₖ
```
where x₁ + x₂ + ··· + xₖ = n.

**Marginals:** Each Xᵢ ~ Binomial(n, pᵢ)

**Mean:** E[Xᵢ] = npᵢ

**Variance:** Var(Xᵢ) = npᵢ(1−pᵢ)

**Covariance:** Cov(Xᵢ, Xⱼ) = −npᵢpⱼ  (negative because categories compete)

**Real example:** Document topic modeling (word counts per topic); multi-class classification outcomes; election vote counts across k candidates.

> **Why / When to use:** Use Multinomial whenever you have n trials, k > 2 outcomes, and constant probabilities. It underlies naive Bayes text classification and Dirichlet-Multinomial topic models.

---

## 3. Continuous Distributions

### 3.1 Continuous Uniform

**What it models:** All values in an interval [a, b] are equally likely.

**PDF:**
```
f(x) = 1/(b−a),   a ≤ x ≤ b
```

**Mean:** E[X] = (a + b) / 2

**Variance:** Var(X) = (b − a)² / 12

**Real example:** Random number generation; arrival time of a bus known only to come within a 10-minute window; quantization error in ADC.

---

### 3.2 Normal / Gaussian

**What it models:** The bell-shaped symmetric distribution arising from the sum of many independent, identically distributed random variables (via CLT). The most important distribution in statistics.

**Parameters:** μ (mean), σ² (variance), σ > 0

**PDF:**
```
f(x) = (1 / (σ√(2π))) · exp(−(x−μ)² / (2σ²)),   x ∈ (−∞, +∞)
```

**Mean:** E[X] = μ

**Variance:** Var(X) = σ²

**Standard Normal:** Z = (X − μ)/σ ~ N(0, 1). Standardization allows use of a single lookup table.

#### The 68-95-99.7 Rule (Empirical Rule)

| Interval | Probability |
|---|---|
| μ ± 1σ | ≈ 68.27% |
| μ ± 2σ | ≈ 95.45% |
| μ ± 3σ | ≈ 99.73% |

This is critical for anomaly detection: an observation beyond ±3σ is "rare" under a normal model.

#### Why It's Everywhere: Central Limit Theorem (CLT)

If X₁, X₂, ..., Xₙ are i.i.d. with mean μ and variance σ², then:
```
(X̄ − μ) / (σ/√n)  →  N(0, 1)  as n → ∞
```
This means sample means (and sums) approach normality regardless of the original distribution, justifying normal-based inference for large samples.

**Real example:** IQ scores, measurement errors, height of adults, returns of diversified portfolios.

**Worked mini-example:** Heights of adult men are N(μ=70 inches, σ=3 inches). P(height > 76) = P(Z > (76−70)/3) = P(Z > 2) ≈ 1 − 0.9772 = 2.28%.

> **Why / When to use:** Normal is appropriate when: data is continuous, symmetric, and unimodal; you have large samples (CLT); you're computing confidence intervals or t-tests. Caution: real data often has heavier tails (use t-distribution for small n), or is skewed (use log-normal).

**🎯 In the interview:** "Why is normal distribution everywhere?" — CLT. But also: "When does CLT fail?" — when underlying distributions have infinite variance (heavy tails, e.g., Pareto) or when n is small. Also: "What is the sum of two independent normals?" — Normal with summed means and summed variances.

---

### 3.3 Exponential

**What it models:** The waiting time between events in a Poisson process (events occur at constant rate λ > 0).

**Parameter:** λ > 0 (rate), often also written with θ = 1/λ (mean)

**PDF:**
```
f(x) = λ · e^(−λx),   x ≥ 0
```

**CDF:**
```
F(x) = 1 − e^(−λx),   x ≥ 0
```

**Mean:** E[X] = 1/λ

**Variance:** Var(X) = 1/λ²

**Key property — Memorylessness:**
```
P(X > s + t | X > s) = P(X > t)   for all s, t ≥ 0
```
The age of a component has no effect on its remaining lifetime. This is the only memoryless continuous distribution.

**Real example:** Time between customer arrivals at a bank; lifetime of a light bulb under constant failure rate; time between server requests.

**Worked mini-example:** A server receives requests at rate λ = 5/minute. Mean time between requests = 1/5 = 0.2 minutes = 12 seconds. P(wait > 30s = 0.5 min) = e^(−5·0.5) = e^(−2.5) ≈ 0.082.

> **Why / When to use:** Exponential is correct only when the failure/arrival rate is constant (no aging or wear-out). If failure rate increases with time (wear-out), use Weibull.

**🎯 In the interview:** "What makes Exponential special?" — memorylessness. "What if failure rate isn't constant?" — Weibull generalizes it with a shape parameter. Geometric is its discrete analog.

---

### 3.4 Gamma

**What it models:** The waiting time until the α-th event in a Poisson process. Generalization of Exponential (α=1).

**Parameters:** α > 0 (shape), β > 0 (rate; alternatively θ = 1/β is the scale)

**PDF:**
```
f(x) = (β^α / Γ(α)) · x^(α−1) · e^(−βx),   x > 0
```
where Γ(α) = (α−1)! for integer α (the Gamma function generalizes factorial).

**Mean:** E[X] = α/β

**Variance:** Var(X) = α/β²

**Special cases:**
- α = 1: Exponential(β)
- α = n/2, β = 1/2: Chi-Square(n)

**Real example:** Total time to process α sequential Poisson-distributed steps; insurance claim total amounts; rainfall accumulation.

**Worked mini-example:** A loan application requires 3 independent review steps, each taking Exp(λ=2/hour). Total time ~ Gamma(α=3, β=2). E[total] = 3/2 = 1.5 hours.

---

### 3.5 Beta

**What it models:** A probability — the Beta distribution is defined on [0, 1] and is ideal for modeling unknown probabilities or proportions.

**Parameters:** α > 0 (shape), β > 0 (shape)

**PDF:**
```
f(x) = x^(α−1) · (1−x)^(β−1) / B(α, β),   0 < x < 1
```
where B(α, β) = Γ(α)Γ(β)/Γ(α+β) is the beta function.

**Mean:** E[X] = α / (α + β)

**Variance:** Var(X) = αβ / [(α+β)²(α+β+1)]

**Shape intuition:**
- α = β = 1: Uniform[0,1]
- α = β > 1: Symmetric bell centered at 0.5
- α > β: Skewed toward 1
- α < β: Skewed toward 0
- α < 1 and β < 1: U-shaped (bimodal at extremes)

#### Conjugate Prior Intuition

Beta(α, β) is the **conjugate prior** for the Binomial likelihood. If your prior on success probability p is Beta(α, β), and you observe x successes in n trials, the posterior is:
```
p | data ~ Beta(α + x, β + n − x)
```
Interpret α as "prior pseudo-successes" and β as "prior pseudo-failures." This is a closed-form Bayesian update. (Source: [MIT 18.05 Conjugate Priors](https://math.mit.edu/~dav/05.dir/class15-prep.pdf))

**Real example:** Estimating click-through rate for an ad; estimating the probability that a new drug works; A/B test conversion rate posterior.

**Worked mini-example:** Prior: Beta(2, 2) (mild belief p ≈ 0.5). Observe 8 successes, 2 failures. Posterior: Beta(2+8, 2+2) = Beta(10, 4). Posterior mean = 10/14 ≈ 0.714.

> **Why / When to use:** Use Beta as a prior whenever you're estimating a probability or proportion. Its flexibility (U-shaped, bell, skewed) covers many belief states.

**🎯 In the interview:** "What does the Beta conjugate prior mean?" — The posterior is in the same family as the prior, giving a closed-form update (no MCMC needed). "What happens as α and β grow?" — the distribution concentrates (variance shrinks), reflecting stronger prior beliefs.

---

### 3.6 Log-Normal

**What it models:** A variable X whose logarithm is normally distributed: ln(X) ~ N(μ, σ²). Always positive, right-skewed.

**Parameters:** μ (mean of log), σ (std of log)

**PDF:**
```
f(x) = (1 / (xσ√(2π))) · exp(−(ln x − μ)² / (2σ²)),   x > 0
```

**Mean:** E[X] = exp(μ + σ²/2)

**Variance:** Var(X) = [exp(σ²) − 1] · exp(2μ + σ²)

**Key property:** Arises as the product of many independent positive random variables (multiplicative CLT analog).

**Real example:** Stock prices, income distributions, city populations, latency in distributed systems, claim sizes in insurance.

> **Why / When to use:** Whenever you expect multiplicative growth or a quantity that is always positive and right-skewed. If you take the log of your data and it looks normal, use log-normal.

**🎯 In the interview:** "How do you check if data is log-normal?" — Take log and check for normality via QQ-plot or Shapiro-Wilk test. Common confusion: E[X] ≠ exp(μ); the correct formula includes the +σ²/2 correction term.

---

### 3.7 Student's t

**What it models:** The distribution of the t-statistic when estimating the mean of a normally distributed population with unknown variance and small sample size.

**Parameter:** ν > 0 (degrees of freedom)

**PDF:**
```
f(t) = Γ((ν+1)/2) / (√(νπ) · Γ(ν/2)) · (1 + t²/ν)^(−(ν+1)/2)
```

**Mean:** E[X] = 0  (for ν > 1)

**Variance:** Var(X) = ν/(ν−2)  (for ν > 2)

**Key properties:**
- Symmetric around 0, bell-shaped, but with **heavier tails** than Normal.
- As ν → ∞, t(ν) → N(0,1).
- For small ν (e.g., ν=1): Cauchy distribution (undefined mean!).
- For ν ≥ 30, practically indistinguishable from Normal in most tests.

**Real example:** Two-sample t-test comparing means; any inference about a population mean when σ is estimated from small samples; Bayesian robust regression.

**Worked mini-example:** Estimating mean of n=10 measurements: use t(9) distribution for 95% CI, not Normal, because σ is unknown. The t critical value (α=0.05, df=9) is 2.262 vs. 1.96 for Normal — the heavier tails demand wider confidence intervals.

> **Why / When to use:** Always use t instead of Normal when (1) population variance is unknown AND (2) sample size is small (n < 30 as a rule of thumb). With large n, use Normal or t (same result).

**🎯 In the interview:** "Why does the t-distribution have heavier tails?" — Because we're estimating σ from data, adding extra uncertainty. The smaller the sample, the heavier the tails (smaller ν). Common trap: assuming normality when using t-tests — the underlying data should be approximately normal (or n large enough for CLT).

---

### 3.8 Chi-Square

**What it models:** The sum of squares of ν independent standard normal variables. Fundamental to variance estimation and goodness-of-fit tests.

**Parameter:** ν > 0 (degrees of freedom)

**Construction:**
```
If Z₁, ..., Zν ~ i.i.d. N(0,1), then X = Z₁² + ··· + Zν² ~ χ²(ν)
```

**PDF:** Special case of Gamma: χ²(ν) = Gamma(α=ν/2, β=1/2)

**Mean:** E[X] = ν

**Variance:** Var(X) = 2ν

**Key properties:**
- Always non-negative; right-skewed; skewness decreases with ν.
- As ν → ∞, χ²(ν) → N(ν, 2ν) by CLT.
- (n−1)S²/σ² ~ χ²(n−1) where S² is the sample variance — this is how we do variance tests.

**Real example:** Pearson's chi-square goodness-of-fit test; chi-square test of independence in contingency tables; likelihood ratio test statistic.

**Worked mini-example:** Testing if a die is fair: compute Σ(O−E)²/E over 6 faces. Under H₀, this statistic ~ χ²(5). Critical value at α=0.05 is 11.07.

**🎯 In the interview:** "What degrees of freedom does the chi-square test of independence use?" — (rows−1)(cols−1). A common trap is using the wrong df.

---

### 3.9 F Distribution

**What it models:** The ratio of two independent chi-square variables divided by their degrees of freedom. Used to compare variances and in ANOVA.

**Parameters:** d₁, d₂ (numerator and denominator degrees of freedom)

**Construction:**
```
If X ~ χ²(d₁) and Y ~ χ²(d₂), independent, then F = (X/d₁) / (Y/d₂) ~ F(d₁, d₂)
```

**Mean:** E[X] = d₂/(d₂−2)  (for d₂ > 2)

**Variance:** Var(X) = 2d₂²(d₁+d₂−2) / [d₁(d₂−2)²(d₂−4)]  (for d₂ > 4)

**Key properties:**
- Always non-negative; right-skewed.
- F(1, ν) = t(ν)² — squaring a t-statistic gives an F with 1 numerator df.
- F(d₁, d₂) = 1 / F(d₂, d₁) — reciprocal relationship.

**Real example:** ANOVA F-test (is there any group mean difference?); comparing model fits (F-test for regression); Levene's test for equality of variances.

**🎯 In the interview:** "What is the relationship between t and F?" — F(1, ν) = t(ν)². An F-test in ANOVA with two groups is equivalent to a two-sample t-test.

---

### 3.10 Weibull

**What it models:** Generalizes the Exponential for time-to-failure, allowing the failure rate to increase, decrease, or stay constant over time.

**Parameters:** λ > 0 (scale), k > 0 (shape)

**PDF:**
```
f(x) = (k/λ) · (x/λ)^(k−1) · exp(−(x/λ)^k),   x ≥ 0
```

**Mean:** E[X] = λ · Γ(1 + 1/k)

**Variance:** Var(X) = λ² · [Γ(1 + 2/k) − (Γ(1 + 1/k))²]

**Shape parameter k:**
- k < 1: Decreasing failure rate (infant mortality / early burnout)
- k = 1: Constant failure rate → reduces to Exponential(1/λ)
- k > 1: Increasing failure rate (wear-out / aging)
- k ≈ 3.5: Approximates Normal distribution

**Real example:** Product reliability engineering; wind speed distribution; time to cancer recurrence.

> **Why / When to use:** When failure rates change over time — the "bathtub curve" in reliability combines three Weibulls: infant mortality (k<1), useful life (k=1), wear-out (k>1).

**🎯 In the interview:** "When does Weibull reduce to Exponential?" — When k=1. "Why is Weibull more realistic for product lifetimes?" — Products degrade; failure rate is not constant.

---

### 3.11 Pareto

**What it models:** A heavy-tailed distribution following a power law. The Pareto principle ("80-20 rule") emerges from it.

**Parameters:** α > 0 (shape / tail index), xₘ > 0 (scale / minimum value)

**PDF:**
```
f(x) = α · xₘ^α / x^(α+1),   x ≥ xₘ
```

**CDF:**
```
F(x) = 1 − (xₘ/x)^α,   x ≥ xₘ
```

**Mean:** E[X] = α·xₘ/(α−1)  (for α > 1; undefined for α ≤ 1)

**Variance:** Var(X) = xₘ²·α / [(α−1)²(α−2)]  (for α > 2; undefined for α ≤ 2)

**Power law / tail behavior:** P(X > x) ~ x^(−α). The smaller α, the heavier the tail.

**Key properties:**
- Log-log plot of the CDF is linear — a diagnostic for power laws.
- For α ≤ 1: undefined mean (infinite expected wealth!).
- For 1 < α ≤ 2: finite mean but infinite variance.

**Real example:** Wealth distribution, city populations, internet traffic (file sizes, degree distribution of social networks), book sales (long tail).

> **Why / When to use:** Use Pareto / power law when you expect extreme events to dominate (top 20% account for 80% of outcomes). Never use Normal for such data — you'll vastly underestimate tail risk.

**🎯 In the interview:** "What is a power law?" — P(X > x) ∝ x^(−α). Log-log scale is linear. Key diagnostic: if the top 1% of users account for ~50% of revenue, you likely have a power law. CLT fails for Pareto with α ≤ 2 because variance is infinite.

---

## 4. Relationships Between Distributions

```
Bernoulli(p)
    │
    │  n i.i.d. trials
    ▼
Binomial(n, p) ──────────────────────── n large, p fixed ──────► Normal(np, np(1−p))  [CLT]
    │
    │  n→∞, p→0, np=λ
    ▼
Poisson(λ) ──────────────────────────── λ large ──────────────► Normal(λ, λ)  [CLT]

Exponential(λ)  [α=1 special case]
    │
    │  sum of α i.i.d. exponentials
    ▼
Gamma(α, λ)
    │
    │  α=ν/2, λ=1/2
    ▼
Chi-Square(ν)
    │          Z²₁ + ··· + Z²ν (Z~N(0,1))
    │
    ▼
F(d₁, d₂) = [χ²(d₁)/d₁] / [χ²(d₂)/d₂]

Student's t(ν) = Z / √(χ²(ν)/ν)    where Z~N(0,1) independent of χ²(ν)
t(ν)² = F(1, ν)

Beta(α, β) is conjugate prior for Binomial
Dirichlet is conjugate prior for Multinomial  [multivariate generalization of Beta]

Geometric(p) = Discrete analog of Exponential
Negative Binomial(r, p) = Discrete analog of Gamma

Log-Normal: if ln(X) ~ Normal, then X ~ Log-Normal
```

---

## 5. Decision Table

| What you are modeling | Distribution to use |
|---|---|
| Single yes/no trial | Bernoulli(p) |
| Number of successes in n fixed trials | Binomial(n, p) |
| Number of rare events in fixed time/space | Poisson(λ) |
| Trials until first success | Geometric(p) |
| Trials until r-th success | Negative Binomial(r, p) |
| Successes in draw without replacement | Hypergeometric(N, K, n) |
| Counts in k categories across n trials | Multinomial(n, p₁, ..., pₖ) |
| Continuous value, equally likely in range | Uniform(a, b) |
| Symmetric, unimodal continuous, large n | Normal(μ, σ²) |
| Waiting time, constant rate, memoryless | Exponential(λ) |
| Waiting time for α-th event | Gamma(α, λ) |
| Unknown probability or proportion | Beta(α, β) |
| Positive, multiplicative, right-skewed | Log-Normal(μ, σ) |
| Mean estimate, unknown σ, small n | Student's t(ν) |
| Sum of squared standard normals | Chi-Square(ν) |
| Ratio of variances or ANOVA | F(d₁, d₂) |
| Time-to-failure, variable failure rate | Weibull(λ, k) |
| Heavy-tailed, power law, 80-20 | Pareto(α, xₘ) |
| Overdispersed count data (Var >> Mean) | Negative Binomial(r, p) |

---

## 6. 🎯 In the Interview: Common Traps

1. **Poisson independence assumption:** Poisson requires events at a *constant rate* and *independently*. Clustered events (aftershocks, viral spread) violate this.

2. **Binomial independence:** Each trial must be independent. Sampling without replacement → Hypergeometric, not Binomial.

3. **When to use t vs. Normal:** With unknown σ and small n, always t. With large n, t ≈ Normal anyway.

4. **Log-Normal mean:** E[X] = exp(μ + σ²/2), NOT exp(μ). Failing to add the σ²/2 term is a common error.

5. **Pareto and undefined moments:** For α ≤ 1, the mean is undefined. For α ≤ 2, the variance is undefined. CLT does not apply to sums of Pareto(α≤2) random variables.

6. **Beta with α < 1, β < 1:** The distribution is U-shaped (bimodal at 0 and 1) — representing a prior belief that the probability is near 0 or near 1, not the middle.

7. **Variance of Poisson = Mean:** If you see Var >> Mean in count data, do not use Poisson — use Negative Binomial (overdispersion).

8. **Chi-square degrees of freedom:** In the goodness-of-fit test with k categories, df = k−1 (not k). In a test of independence in a r×c table, df = (r−1)(c−1).

9. **F-test and t-test relationship:** F(1, ν) = t(ν)². A two-sample t-test is a special case of one-way ANOVA.

10. **Exponential vs Weibull:** Exponential assumes constant failure rate (memoryless). Real products wear out — use Weibull with k > 1.

---

## Glossary

| Term | Definition |
|---|---|
| PMF (Probability Mass Function) | P(X=x) for discrete RVs; gives exact probabilities |
| PDF (Probability Density Function) | f(x) for continuous RVs; P(a≤X≤b) = ∫f(x)dx |
| CDF (Cumulative Distribution Function) | F(x) = P(X ≤ x); defined for all distributions |
| Support | The set of values where the PMF/PDF is non-zero |
| Parameter | A fixed constant that defines a specific distribution |
| Conjugate Prior | A prior whose posterior is in the same distribution family |
| Memorylessness | P(X > s+t \| X > s) = P(X > t); property of Geometric and Exponential |
| Overdispersion | When observed variance exceeds what a model predicts (e.g., Var > Mean for count data) |
| Power Law | P(X > x) ~ x^(−α); heavy-tailed behavior |
| Degrees of Freedom | Parameter(s) controlling shape of t, χ², F distributions; typically n−k for k estimated params |
| Moment | E[Xⁿ] — the n-th moment of a distribution |
| MGF (Moment Generating Function) | M(t) = E[e^(tX)]; encodes all moments |
| Finite Population Correction | Factor (N−n)/(N−1) reducing variance in Hypergeometric sampling |

---

## References

1. **DeGroot, M.H. & Schervish, M.J.** — *Probability and Statistics*, 4th ed. Addison-Wesley, 2012. (Canonical graduate-level reference for all distributions and their derivations.)
2. **Casella, G. & Berger, R.L.** — *Statistical Inference*, 2nd ed. Duxbury, 2002. (Distributions, sufficiency, exponential families.)
3. **Oxford Emory Math Center** — "The Connection Between the Poisson and Binomial Distributions." [https://mathcenter.oxford.emory.edu/site/math117/connectingPoissonAndBinomial/](https://mathcenter.oxford.emory.edu/site/math117/connectingPoissonAndBinomial/)
4. **MIT 18.05** — "Conjugate Priors: Beta and Normal." [https://math.mit.edu/~dav/05.dir/class15-prep.pdf](https://math.mit.edu/~dav/05.dir/class15-prep.pdf)
5. **Yale Statistics (Pollard)** — "Chapter 8: Poisson Approximations." [http://www.stat.yale.edu/~pollard/Courses/241.fall97/Poisson.pdf](http://www.stat.yale.edu/~pollard/Courses/241.fall97/Poisson.pdf)
6. **NIST/SEMATECH e-Handbook of Statistical Methods** — [https://www.itl.nist.gov/div898/handbook/](https://www.itl.nist.gov/div898/handbook/) (Authoritative formulas for all standard distributions.)
7. **Wikipedia: List of probability distributions** — [https://en.wikipedia.org/wiki/List_of_probability_distributions](https://en.wikipedia.org/wiki/List_of_probability_distributions) (Cross-check for formulas.)

---

*Part of the [Statistics for Data Science](README.md) section · [Guide Home](../README.md).*
