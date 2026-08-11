# 09 — Bayesian Statistics

Bayesian statistics is a framework for updating beliefs in light of evidence. Rather than treating model parameters as fixed but unknown constants (the frequentist view), the Bayesian framework treats parameters as **random variables with probability distributions** that encode uncertainty. Every inference begins with a prior belief, gets updated by observed data through the likelihood, and produces a posterior distribution that is the complete summary of what is known after seeing the data.

> **In plain English:** Bayesian reasoning is exactly what humans do intuitively. Before you flip a coin you have a belief about its fairness; after 10 flips you update that belief; after 1000 flips you are very confident. Bayesian statistics just makes that updating process mathematically precise.

## Table of Contents

1. [Frequentist vs Bayesian Mindset](#1-frequentist-vs-bayesian-mindset)
2. [The Three Pillars: Prior, Likelihood, Posterior](#2-the-three-pillars-prior-likelihood-posterior)
3. [Types of Priors](#3-types-of-priors)
4. [Conjugate Priors](#4-conjugate-priors)
5. [Worked Example: Beta-Binomial (Conversion Rate)](#5-worked-example-beta-binomial-conversion-rate)
6. [Credible Interval vs Confidence Interval](#6-credible-interval-vs-confidence-interval)
7. [Posterior Predictive Distribution](#7-posterior-predictive-distribution)
8. [Bayes Factors and Bayesian Hypothesis Testing](#8-bayes-factors-and-bayesian-hypothesis-testing)
9. [MAP vs Full Posterior](#9-map-vs-full-posterior)
10. [When Bayesian Shines — and Its Costs](#10-when-bayesian-shines--and-its-costs)
11. [MCMC and Conjugacy at Intuition Level](#11-mcmc-and-conjugacy-at-intuition-level)
12. [🎯 In the Interview](#12--in-the-interview)
13. [Glossary](#glossary)
14. [References](#references)

---

## 1. Frequentist vs Bayesian Mindset

| Dimension | Frequentist | Bayesian |
|---|---|---|
| What is a parameter? | Fixed but unknown constant | Random variable with a distribution |
| What is probability? | Long-run frequency of events | Degree of belief (subjective or objective) |
| Data | Random (different each experiment) | Fixed (what we observed) |
| Parameters | Fixed (no distribution) | Random (have a distribution) |
| Output | Point estimate + confidence interval | Full posterior distribution |
| Prior information | Ignored (or informal) | Explicitly encoded as a prior |
| Sequential updating | Requires re-running the test | Natural: today's posterior = tomorrow's prior |

**Frequentist example:** "The true conversion rate θ is fixed. I ran an experiment and got a 95% confidence interval of [0.04, 0.08]." This interval either contains θ or it does not; we cannot say θ has a 95% probability of being in it.

**Bayesian example:** "My prior belief is that θ ~ Beta(2, 20). After observing 8 conversions out of 100 visits, the posterior is Beta(10, 112). The 95% credible interval is [0.04, 0.13]. There is a 95% probability that θ lies in that interval — given my prior and the data."

> **Why this distinction matters in interviews:** The most common trap is saying a confidence interval means "there is a 95% chance the parameter is in this range." That is the Bayesian credible interval interpretation, not the frequentist confidence interval interpretation.

---

## 2. The Three Pillars: Prior, Likelihood, Posterior

Bayes' theorem applied to inference:

```
P(θ | data) = P(data | θ) × P(θ) / P(data)
```

Or in proportional form (dropping the normalising constant P(data)):

```
posterior ∝ likelihood × prior
```

- **Prior P(θ):** Your belief about the parameter *before* seeing data. Can come from domain expertise, previous experiments, or theoretical considerations.
- **Likelihood P(data | θ):** How probable is the observed data for each possible value of θ? This is the standard frequentist building block (e.g. binomial likelihood, Gaussian likelihood).
- **Posterior P(θ | data):** The updated belief after combining prior and data. This is the complete inferential output in Bayesian analysis. ([CMU lecture notes on Bayesian inference](https://www.stat.cmu.edu/~larry/=sml/Bayes.pdf))

**P(data) (marginal likelihood / evidence):** The normalising constant that ensures the posterior integrates to 1. It is often intractable and is the main source of computational difficulty in Bayesian analysis. In conjugate models and simple cases it can be computed analytically; otherwise we use MCMC or variational inference.

---

## 3. Types of Priors

| Prior Type | Description | Example | When to Use |
|---|---|---|---|
| **Informative** | Strongly constrains parameter to a specific region | Beta(50, 50) for a coin you believe is fair | Strong domain knowledge or previous experiments |
| **Weakly informative** | Gently regularises but doesn't dominate the data | Normal(0, 1) for regression coefficients (Stan defaults) | Default in modern Bayesian analysis; prevents pathological posteriors |
| **Uninformative (flat)** | Assigns equal probability to all values | Uniform(0, 1) for a probability | When you genuinely have no prior knowledge; can lead to improper posteriors |
| **Jeffreys prior** | Invariant to reparameterisation; derived from the Fisher information matrix | Beta(0.5, 0.5) for a binomial proportion | When you want objective/reference priors that do not arbitrarily favour one parameterisation |

**Jeffreys prior formula:**  
π(θ) ∝ √det(I(θ)), where I(θ) is the Fisher information matrix.

For a Bernoulli/binomial proportion p, Jeffreys prior is Beta(1/2, 1/2) — a U-shaped distribution that puts more weight near 0 and 1 than the flat Uniform(0,1) prior.

> **Prior sensitivity:** Informative priors can dominate in small-sample regimes. Always run a **sensitivity analysis**: re-run your model with several plausible priors to check that posterior conclusions are robust to prior choice. If they are not, your data are too sparse to be conclusive.

---

## 4. Conjugate Priors

A prior P(θ) is **conjugate** to a likelihood P(data | θ) if the resulting posterior P(θ | data) belongs to the **same family** as the prior. This is a mathematical convenience: it yields closed-form posteriors without numerical integration.

### Key Conjugate Pairs

| Likelihood | Conjugate Prior | Posterior | Parameter Updated |
|---|---|---|---|
| Binomial / Bernoulli | Beta(α, β) | Beta(α + successes, β + failures) | Probability p |
| Poisson | Gamma(α, β) | Gamma(α + Σxᵢ, β + n) | Rate λ |
| Normal (known σ²) | Normal(μ₀, σ₀²) | Normal(weighted average, reduced variance) | Mean μ |
| Multinomial | Dirichlet(α) | Dirichlet(α + counts) | Category probabilities |
| Exponential | Gamma(α, β) | Gamma(α + n, β + Σxᵢ) | Rate λ |

**Why conjugacy matters:**
1. Closed-form posterior — no MCMC needed.
2. Parameters of the prior have natural interpretations as **pseudo-observations** (e.g. α and β in the Beta prior represent prior successes and failures).
3. Enables sequential/online updating: new data simply updates the hyperparameters.

Source: [STAT 535 Conjugate Priors, University of South Carolina](https://people.stat.sc.edu/hitchcock/stat535slides5BRBhandout.pdf)

---

## 5. Worked Example: Beta-Binomial (Conversion Rate)

**Scenario:** You are running an e-commerce site. You want to estimate the conversion rate θ (proportion of visitors who make a purchase). You have a weak prior belief that conversion rates for this type of site run around 5%.

### Step 1 — Choose Prior

Encode "about 5% with moderate uncertainty" as:

```
θ ~ Beta(α=2, β=38)
```

- Prior mean = α / (α + β) = 2/40 = 0.05
- Prior effective sample size = α + β = 40 (equivalent to seeing 2 conversions in 40 visits)

### Step 2 — Collect Data

You observe **n = 200 visits** with **k = 14 conversions**.

### Step 3 — Compute Posterior

By the Beta-Binomial conjugacy:

```
θ | data ~ Beta(α + k, β + n − k)
         = Beta(2 + 14, 38 + 200 − 14)
         = Beta(16, 224)
```

### Step 4 — Summarise Posterior

| Quantity | Value |
|---|---|
| Posterior mean | 16 / 240 ≈ 0.067 (6.7%) |
| Posterior mode (MAP) | (16 − 1) / (240 − 2) ≈ 0.063 (6.3%) |
| 95% credible interval | approx [0.039, 0.103] |

### Step 5 — Interpret

The posterior has been pulled from the prior mean (5%) toward the data (14/200 = 7%), with the pull proportional to the relative information content. The prior contributed 40 pseudo-observations vs 200 real observations, so the data dominates (~83% weight) but the prior still has an effect.

**Sequential updating:** If next week you get 50 more visits with 3 conversions:

```
New posterior = Beta(16 + 3, 224 + 47) = Beta(19, 271)
```

No need to re-run from scratch — this is one of Bayesian inference's biggest practical advantages.

---

## 6. Credible Interval vs Confidence Interval

This is the single most important distinction in Bayesian vs frequentist inference for interviews.

| | Credible Interval (Bayesian) | Confidence Interval (Frequentist) |
|---|---|---|
| **What it says** | "Given the data and prior, there is X% posterior probability that θ lies in this interval." | "If we repeated this experiment many times, X% of such intervals would contain the true θ." |
| **About the parameter** | The parameter is treated as random; the interval has a probability statement about it directly | The parameter is fixed; the interval is random |
| **Interpretation of a specific interval** | **Yes:** "There is a 95% probability θ is in [a, b]" (given the prior and data) | **No:** "The probability θ is in [0.04, 0.08] is either 0 or 1; we just don't know which" |
| **Requires a prior** | Yes | No |
| **Depends on stopping rule** | No (likelihood principle) | Yes (affects p-values and CIs) |

> **Interview trap:** "A 95% confidence interval means there is a 95% chance the true parameter lies inside it." This is **false** under frequentist theory. It is the correct interpretation for a Bayesian credible interval. Examiners frequently ask candidates to articulate this distinction precisely.

**Types of credible intervals:**
- **Equal-tailed interval (ETI):** cuts 2.5% from each tail. Analogous to a standard CI.
- **Highest Posterior Density interval (HPD):** the shortest interval containing X% of the posterior mass. Preferred when the posterior is skewed.

---

## 7. Posterior Predictive Distribution

The posterior predictive distribution answers: **"What values of new data ỹ do I expect, integrating over my uncertainty in θ?"**

```
P(ỹ | data) = ∫ P(ỹ | θ) × P(θ | data) dθ
```

It is a weighted average of the likelihood across all plausible parameter values (weighted by the posterior). This is more honest than plugging in a single point estimate because it propagates parameter uncertainty into predictions.

**Example (Beta-Binomial continued):** After observing the data above, the posterior predictive probability of getting exactly k conversions out of the next m visitors averages the binomial likelihood over the Beta(16, 224) posterior. This yields the Beta-Binomial distribution.

**Use in model checking:** Draw samples from the posterior predictive and compare to the observed data. Large discrepancies suggest model misfit (posterior predictive checks / PPC).

---

## 8. Bayes Factors and Bayesian Hypothesis Testing

### Bayes Factor Definition

The **Bayes factor (BF₁₀)** is the ratio of the marginal likelihoods of two competing models (or hypotheses):

```
BF₁₀ = P(data | H₁) / P(data | H₀)
```

It quantifies **how much the data update the odds in favour of H₁ over H₀**.

### Relationship to Posterior Odds

```
Posterior odds = Prior odds × Bayes factor
P(H₁ | data) / P(H₀ | data) = [P(H₁) / P(H₀)] × BF₁₀
```

Source: [Bayesian Statistics the Fun Way — Chapter 16](https://bookdown.org/pbaumgartner/bayesian-fun/16-bayes-factor-posterior-odds.html)

### Jeffreys' Interpretation Scale

| BF₁₀ | Evidence for H₁ |
|---|---|
| 1 – 3 | Anecdotal |
| 3 – 10 | Moderate |
| 10 – 30 | Strong |
| 30 – 100 | Very strong |
| > 100 | Decisive |

### Advantages over p-values

| | p-value | Bayes Factor |
|---|---|---|
| Quantifies evidence for H₀? | No (can't "accept" H₀) | Yes (BF < 1 supports H₀) |
| Affected by stopping rule? | Yes | No (satisfies likelihood principle) |
| Interpretable as odds update? | No | Yes |
| Requires prior on parameters? | No | Yes (but often default/uninformative priors are used) |

> **Nuance:** Computing Bayes factors requires the marginal likelihood P(data | Hᵢ) = ∫ P(data | θ, Hᵢ) P(θ | Hᵢ) dθ, which is often intractable. Approximations include the Bayesian Information Criterion (BIC) and bridge sampling.

---

## 9. MAP vs Full Posterior

| Method | Definition | Pros | Cons |
|---|---|---|---|
| **MLE** | argmax P(data \| θ) | Simple; no prior needed | Overfits with small data; no uncertainty |
| **MAP** (Maximum A Posteriori) | argmax P(θ \| data) = argmax P(data \| θ) × P(θ) | Adds regularisation (prior acts as L2 for Gaussian prior); single point estimate | Ignores posterior shape; still loses uncertainty quantification; mode ≠ mean for skewed posteriors |
| **Full Posterior** | Entire distribution P(θ \| data) | Complete uncertainty quantification; credible intervals; posterior predictive; model comparison | Computationally expensive; requires MCMC or VI |

**MAP and regularisation:** With a Gaussian prior N(0, σ²) on θ, MAP estimation is equivalent to ridge regression (L2 regularisation). With a Laplace prior, MAP is equivalent to LASSO. This connection unifies frequentist regularisation and Bayesian inference.

> **When to use MAP:** When you need a point estimate, computational cost is a constraint, and the posterior is roughly symmetric/unimodal. For asymmetric posteriors or when uncertainty propagation matters, use the full posterior.

---

## 10. When Bayesian Shines — and Its Costs

### When Bayesian is the Right Choice

| Scenario | Reason |
|---|---|
| **Small data** | Prior regularises the posterior, preventing overfitting |
| **Domain expertise available** | Prior formally encodes known constraints |
| **Sequential updating / online learning** | Today's posterior becomes tomorrow's prior; no batch re-processing |
| **A/B testing** | Can compute P(θ_A > θ_B) directly; can stop early with proper Bayesian stopping rules; no multiple-comparison issues from peeking |
| **Hierarchical / multi-level models** | Priors on group-level parameters naturally share information (partial pooling) |
| **Full uncertainty propagation** | Posterior predictive integrates over parameter uncertainty |
| **Model comparison** | Bayes factors and WAIC/LOO-CV are principled model selection tools |

### Costs and Limitations

| Cost | Detail |
|---|---|
| **Prior sensitivity** | With small data, results can be dominated by the prior. Requires sensitivity analysis. |
| **Computational cost** | Full posterior inference requires MCMC (often slow) or variational inference (faster but approximate). |
| **Subjectivity** | Critics argue the prior injects subjective belief. (Counter: priors can be weakly informative or Jeffreys.) |
| **Intractable marginal likelihood** | Model comparison requires P(data \| H), which is typically intractable. |
| **Expertise required** | Requires statistical expertise to choose priors and diagnose MCMC convergence. |

> **Bayesian A/B testing advantage:** Unlike frequentist testing, you can answer "What is the probability that variant B is better than variant A?" directly, and you can peek at results without inflating Type I error (as long as the decision rule is Bayesian, e.g. decision at posterior probability threshold).

---

## 11. MCMC and Conjugacy at Intuition Level

### Conjugacy (recap)

When the prior and likelihood are conjugate, the posterior is analytically available. The hyperparameters of the prior simply get updated with sufficient statistics from the data. No numerical methods needed. This is computationally free but only works for specific prior-likelihood combinations.

### When Conjugacy Fails

Most real models (hierarchical models, logistic regression, neural networks, complex generative models) have posteriors that cannot be written in closed form. We need numerical methods.

### Markov Chain Monte Carlo (MCMC) — Intuition

**Goal:** Draw samples θ⁽¹⁾, θ⁽²⁾, ..., θ⁽ᴺ⁾ from the posterior P(θ | data) without knowing its normalising constant.

**How it works (Metropolis-Hastings):**
1. Start at some θ⁽ᵗ⁾.
2. Propose a new value θ* from a proposal distribution q(θ* | θ⁽ᵗ⁾).
3. Compute acceptance ratio: α = [P(data | θ*) P(θ*)] / [P(data | θ⁽ᵗ⁾) P(θ⁽ᵗ⁾)].
4. Accept θ* with probability min(1, α); else stay at θ⁽ᵗ⁾.
5. After a burn-in period, the chain's samples are approximately distributed as the posterior.

**Modern MCMC:** Hamiltonian Monte Carlo (HMC), as implemented in Stan and PyMC, uses gradient information to propose efficient moves. No-U-Turn Sampler (NUTS) is the standard in practice.

**Variational Inference (VI):** Instead of sampling, approximate the posterior with a simpler distribution (e.g. a factored Gaussian) and minimise KL divergence. Much faster than MCMC but less accurate for multi-modal or correlated posteriors.

| Method | Speed | Accuracy | Scalability |
|---|---|---|---|
| Conjugate analytical | Instant | Exact (for conjugate models) | Limited to conjugate families |
| MCMC (NUTS/HMC) | Slow | Gold standard | Moderate (millions of params is hard) |
| Variational Inference | Fast | Approximate | High (used in deep Bayesian models) |

---

## 12. 🎯 In the Interview

### Common Traps

**Trap 1 — Confidence interval interpretation:**
> "A 95% confidence interval means there is a 95% probability the true parameter is in this range."

**Correct answer:** This is the **credible interval** interpretation. A 95% confidence interval means: if you repeated the experiment infinitely many times and computed a CI each time, 95% of those intervals would contain the true parameter. The specific interval you computed either contains the true value or it doesn't — probability is 0 or 1, we just don't know which.

**Trap 2 — Prior vs posterior:**
> "In Bayesian inference, you set a prior and that's your answer."

**Correct answer:** The prior is just the starting point. The posterior = prior updated by the likelihood. With enough data, the posterior converges to the MLE regardless of a reasonable prior (Bernstein-von Mises theorem).

**Trap 3 — Bayes factor interpretation:**
> "A Bayes factor of 10 means H₁ is 10 times more probable than H₀."

**Correct answer:** BF₁₀ = 10 means the data are 10 times more likely under H₁ than H₀ (i.e. it updates the prior odds by a factor of 10). To get posterior odds you still need to multiply by prior odds.

**Trap 4 — MAP vs MLE:**
> "MAP and MLE give the same estimate."

**Correct answer:** They coincide only with a flat (uniform) prior. With an informative prior, MAP is regularised toward the prior mode. MAP with a Gaussian prior = ridge regression; MAP with a Laplace prior = LASSO.

**Trap 5 — When to use Bayesian:**
> "Bayesian methods are always better."

**Correct answer:** Bayesian methods excel with small data, sequential updating, and when priors are available. They are computationally expensive, sensitive to prior choice, and require expertise. Frequentist methods are simpler, scalable, and prior-free — better when data are abundant and the inference task is standard.

### Key Formulas to Know

```
posterior ∝ likelihood × prior

Posterior odds = Prior odds × Bayes Factor

Beta-Binomial: Beta(α, β) → Beta(α + k, β + n − k)

MAP with Gaussian prior ↔ Ridge regression
MAP with Laplace prior  ↔ LASSO
```

---

## Glossary

| Term | Definition |
|---|---|
| **Prior** | Distribution encoding beliefs about a parameter before observing data |
| **Likelihood** | P(data \| θ) — how probable the data are for each parameter value |
| **Posterior** | P(θ \| data) — updated belief after combining prior and likelihood via Bayes' theorem |
| **Conjugate prior** | A prior that yields a posterior in the same distributional family |
| **Credible interval** | Bayesian interval with a direct probability statement about the parameter |
| **Confidence interval** | Frequentist interval with a long-run coverage guarantee |
| **MAP** | Maximum A Posteriori — the mode of the posterior distribution |
| **Bayes factor** | Ratio of marginal likelihoods; quantifies evidence for one hypothesis over another |
| **Posterior predictive** | Distribution of new data integrating over posterior uncertainty in parameters |
| **MCMC** | Family of algorithms for sampling from intractable posteriors |
| **HMC / NUTS** | Gradient-based MCMC samplers used in Stan and PyMC |
| **Jeffreys prior** | Objective prior invariant to reparameterisation; proportional to √det(Fisher information) |
| **Marginal likelihood** | P(data \| H) = ∫ P(data \| θ) P(θ) dθ; normalising constant; used in Bayes factors |

---

## References

1. [CMU — Bayes and Statistical Machine Learning (Larry Wasserman)](https://www.stat.cmu.edu/~larry/=sml/Bayes.pdf)
2. [An Introduction to Bayesian Thinking — StatsWithR](https://statswithr.github.io/book/bayesian-inference.html)
3. [STAT 535: Conjugate Priors — University of South Carolina](https://people.stat.sc.edu/hitchcock/stat535slides5BRBhandout.pdf)
4. [Bayesian Statistics the Fun Way — Chapter 16: Bayes Factors](https://bookdown.org/pbaumgartner/bayesian-fun/16-bayes-factor-posterior-odds.html)
5. [Bayesian Statistics: A Practical Introduction 2026 — Quantt](https://www.quantt.co.uk/resources/bayesian-statistics-guide)
6. [Conjugate Prior Overview — ScienceDirect](https://www.sciencedirect.com/topics/computer-science/conjugate-prior)

---

*Part of the [Statistics for Data Science](README.md) section · [Guide Home](../README.md).*
