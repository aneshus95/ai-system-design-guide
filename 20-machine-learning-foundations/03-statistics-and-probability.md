# Statistics and Probability

Statistics and probability form the bedrock of every data science role: you cannot run an A/B test, build a regression model, or evaluate a classifier without them. Interviewers at product and research companies treat this as a mandatory axis — weak answers here are a quick filter-out regardless of your ML or coding skills.

## Table of Contents

- [Descriptive Statistics](#descriptive-statistics)
- [Probability Basics](#probability-basics)
- [Bayes' Theorem](#bayes-theorem)
- [Common Distributions](#common-distributions)
- [Central Limit Theorem](#central-limit-theorem)
- [Law of Large Numbers](#law-of-large-numbers)
- [Sampling and Bias](#sampling-and-bias)
- [Hypothesis Testing](#hypothesis-testing)
- [Confidence Intervals](#confidence-intervals)
- [Common Statistical Tests](#common-statistical-tests)
- [A/B Testing](#ab-testing)
- [Correlation vs Causation](#correlation-vs-causation)
- [Covariance vs Correlation](#covariance-vs-correlation)
- [MLE vs MAP](#mle-vs-map)
- [Bayesian vs Frequentist](#bayesian-vs-frequentist)
- [Bootstrapping and Resampling](#bootstrapping-and-resampling)
- [Standard Error](#standard-error)
- [Multicollinearity and VIF](#multicollinearity-and-vif)
- [R² vs Adjusted R²](#r-vs-adjusted-r)
- [P-Hacking](#p-hacking)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## Descriptive Statistics

Descriptive statistics **summarize** a dataset so you can grasp its shape before modelling.

### Measures of Central Tendency

| Measure | Formula / Definition | Best Used When |
|---------|---------------------|----------------|
| **Mean** | Sum / count | Distribution is roughly symmetric |
| **Median** | Middle value (sorted) | Outliers present (income, house prices) |
| **Mode** | Most frequent value | Categorical data, multi-modal distributions |

**In plain English:** If 9 people earn $30k and 1 person earns $1M, the mean ($127k) is misleading. The median ($30k) tells the real story. Always report both.

### Measures of Spread

- **Variance** σ² = E[(X − μ)²] — average squared deviation from the mean. Squaring magnifies outliers.
- **Standard deviation** σ = √σ² — same units as the data, more interpretable.
- **IQR** = Q3 − Q1 — robust to outliers; used in box plots.

### Shape: Skewness and Kurtosis

- **Skewness** measures asymmetry. Positive skew → long right tail (income). Negative skew → long left tail (test scores near 100%).
- **Kurtosis** measures tail heaviness. High kurtosis ("leptokurtic") means fat tails — extreme events are more common than a normal distribution would predict.

```
Positive skew            Negative skew
   |                          |
   |  *                    *  |
   | ***                  *** |
   |*****  ->        <- *****
   +--------              --------+
  mode median mean    mean median mode
```

---

## Probability Basics

### Conditional Probability

P(A | B) = P(A ∩ B) / P(B)

"The probability of A given we already know B occurred." Shrinks the sample space.

### Independence

A and B are independent if P(A ∩ B) = P(A) × P(B). Knowing B gives no information about A.

### Expected Value

E[X] = Σ xᵢ P(xᵢ) for discrete, ∫ x f(x) dx for continuous.

**Worked Example 1 — Dice Roll:**

Roll a fair six-sided die. What is E[X]?

E[X] = 1(1/6) + 2(1/6) + 3(1/6) + 4(1/6) + 5(1/6) + 6(1/6)
     = 21/6 = **3.5**

**Worked Example 2 — Coin Flip Game:**

You win $4 on heads and lose $1 on tails (fair coin). Should you play?

E[gain] = 0.5 × ($4) + 0.5 × (−$1) = $2 − $0.50 = **$1.50 per flip → Yes, play.**

---

## Bayes' Theorem

**Formula:**

P(H | E) = [ P(E | H) × P(H) ] / P(E)

- **Prior** P(H): belief before seeing evidence.
- **Likelihood** P(E | H): how probable the evidence is if H is true.
- **Posterior** P(H | E): updated belief after evidence.

### Worked Numeric Example — Medical Test

A disease affects **1% of the population**. A test is **99% sensitive** (true positive rate) and **95% specific** (true negative rate, so 5% false positive rate).

You test positive. What is the probability you actually have the disease?

```
               Has Disease (1%)    No Disease (99%)
               ┌──────────────┐    ┌──────────────┐
Test Positive  │  TP = 0.99%  │    │  FP = 4.95%  │
Test Negative  │  FN = 0.01%  │    │  TN = 94.05% │
               └──────────────┘    └──────────────┘
```

P(disease | positive) = 0.99% / (0.99% + 4.95%) = **≈ 16.7%**

**In plain English:** Even with a very accurate test, a rare disease means most positive results are false positives. This is the base-rate fallacy — always check the prior.

---

## Common Distributions

| Distribution | Type | PMF/PDF Key | Mean | Variance | Typical Use |
|---|---|---|---|---|---|
| **Bernoulli(p)** | Discrete | P(X=1)=p | p | p(1−p) | Single binary outcome |
| **Binomial(n,p)** | Discrete | C(n,k)pᵏ(1−p)ⁿ⁻ᵏ | np | np(1−p) | k successes in n trials |
| **Poisson(λ)** | Discrete | e⁻λ λᵏ/k! | λ | λ | Events per unit time/area |
| **Normal(μ,σ²)** | Continuous | Bell curve | μ | σ² | Heights, errors, CLT limit |
| **Uniform(a,b)** | Continuous | 1/(b−a) | (a+b)/2 | (b−a)²/12 | Random seed generation |
| **Exponential(λ)** | Continuous | λe⁻λx | 1/λ | 1/λ² | Time between Poisson events |

**In plain English:**
- If you're counting **rare events** (server crashes per hour), reach for Poisson.
- If you're modelling **wait time until the next event**, use Exponential (and note: it has the memoryless property — past waiting time doesn't affect future waiting time).
- Binomial is the sum of n independent Bernoulli trials.

---

## Central Limit Theorem

**Statement:** Given a population with mean μ and finite variance σ², the distribution of the **sample mean** X̄ from samples of size n approaches Normal(μ, σ²/n) as n → ∞, **regardless of the population's shape**.

```
Population dist.        n=5             n=30           n=100
(Skewed right)
   *                  **               ****            *****
  ***        →       ****    →        ******   →     *******
 *****              ******           ********        *******
****** ...        ********          *********       ********
                  (still skewed)   (less skewed)   (≈ normal!)
```

**Why it matters for interviews:**
1. Justifies using z-tests/t-tests on non-normal data when n is large.
2. Underlies confidence intervals and p-values.
3. Rule of thumb: n ≥ 30 is usually sufficient for the CLT to kick in (more if distribution is very skewed).

**Standard error of the mean:** SE = σ / √n. As sample size increases, estimates become more precise.

---

## Law of Large Numbers

As the number of trials n → ∞, the sample mean X̄ converges to the true population mean μ.

**CLT vs LLN:**
- LLN: the sample mean **converges to** μ (a number).
- CLT: the **distribution of** the sample mean converges to a normal (a shape).

---

## Sampling and Bias

### Sampling Methods

| Method | How | When to Use |
|--------|-----|-------------|
| **Simple random** | Every unit equally likely | Homogeneous population |
| **Stratified** | Sample within subgroups proportionally | Heterogeneous population; preserve subgroup representation |
| **Cluster** | Randomly pick groups, sample all within | Geographically spread, cost-constrained |
| **Systematic** | Every kth element | Large ordered lists |

### Key Biases

- **Selection bias:** The sample is not representative — e.g., only surveying users who opened your email.
- **Survivorship bias:** You only see units that "survived" some filter (e.g., studying successful startups and ignoring the failed ones).
- **Confirmation bias:** Collecting data that confirms a pre-existing hypothesis.
- **Non-response bias:** Respondents differ systematically from non-respondents.

**In plain English:** Most real-world ML datasets have selection bias baked in. Always ask: "Who is missing from this data and why?"

---

## Hypothesis Testing

### The Framework

1. State **H₀** (null): the default assumption (e.g., "the new feature has no effect").
2. State **H₁** (alternative): what you want to prove.
3. Choose significance level **α** (typically 0.05).
4. Compute the **test statistic** and **p-value**.
5. Decision: if p ≤ α, reject H₀; else, fail to reject H₀.

### P-Value

The p-value is the probability of observing data at least as extreme as the actual data, **assuming H₀ is true**.

```
         H₀ distribution
              ___
            /   \
           /     \         ← if your test statistic
          /       \           lands in this tail...
_________/         \__________ p-value is the shaded area
                    |█████|
                    ← extreme →
```

**Common misconception:** p-value is NOT the probability that H₀ is true, and NOT the probability of a false positive. It is a conditional probability under H₀.

### Type I and Type II Errors

```
                      GROUND TRUTH
                   H₀ True    H₀ False
              ┌────────────┬────────────┐
Reject H₀    │  Type I ❌  │ Correct ✓  │
(Test says    │  (α = FPR)  │   (Power)  │
 positive)    ├────────────┼────────────┤
Fail to       │ Correct ✓  │  Type II ❌ │
reject H₀     │   (TNR)    │  (β = FNR)  │
              └────────────┴────────────┘
```

- **Type I error (α):** False positive — rejecting a true H₀. "We launched a feature that did nothing."
- **Type II error (β):** False negative — failing to reject a false H₀. "We killed a feature that actually worked."
- **Statistical Power = 1 − β:** The probability of correctly detecting a real effect. Increased by larger sample size or larger effect size.

### Significance Level α

The threshold you set before the test. α = 0.05 means you accept a 5% chance of a Type I error. Reducing α reduces Type I errors but increases Type II errors (lower power).

---

## Confidence Intervals

A 95% CI does **not** mean "there's a 95% chance the true parameter is in this interval." The parameter is fixed; the interval is random.

**Correct interpretation:** If you ran this experiment 100 times and built a CI each time, approximately 95 of those intervals would contain the true parameter.

**Formula (large n, known σ):**

CI = X̄ ± z*(σ/√n)

For 95% CI: z* = 1.96.

**In plain English:** A CI is a measure of precision. Wide CI → small sample, high variance. Narrow CI → confident estimate.

---

## Common Statistical Tests

| Test | When to Use | Assumptions |
|------|-------------|-------------|
| **z-test** | Compare means, large n (≥30), σ known | Normal or large n |
| **t-test (1-sample)** | Compare sample mean to known value, small n | Normal population or n ≥ 30 |
| **t-test (2-sample)** | Compare means of two independent groups | Approximate normality, similar variance |
| **Paired t-test** | Compare two related groups (before/after) | Differences are normally distributed |
| **Chi-square test** | Association between categorical variables | Expected cell counts ≥ 5 |
| **ANOVA** | Compare means across 3+ groups | Normality, equal variances, independence |
| **Mann-Whitney U** | Non-parametric alternative to 2-sample t-test | Ordinal data or non-normal small samples |

**Choosing a test — quick heuristic:**
- 1 group vs known value → 1-sample t-test
- 2 independent groups → 2-sample t-test
- 2 paired groups → paired t-test
- 3+ groups → ANOVA
- Categorical association → chi-square

---

## A/B Testing

A/B testing is structured hypothesis testing applied to product decisions. It is the single most frequently tested experimentation topic in DS interviews.

### Design Flow

```
       ┌──────────────────────────────────────────┐
       │ 1. Define success metric (primary KPI)   │
       │ 2. State H₀ / H₁                         │
       │ 3. Choose α and desired power (1−β)       │
       │ 4. Calculate required sample size n       │
       │ 5. Randomly split users → control/treat  │
       │ 6. Run until n reached (no peeking!)      │
       │ 7. Compute p-value; make decision         │
       └──────────────────────────────────────────┘
```

### Sample Size Formula

n per group ≈ 2 × (z_α/2 + z_β)² × σ² / δ²

where δ is the minimum detectable effect (MDE). Larger MDE → smaller n needed.

### Key Pitfalls

| Pitfall | What It Is | Fix |
|---------|------------|-----|
| **Peeking** | Checking results early and stopping if p < 0.05 | Commit to sample size upfront; use sequential testing |
| **Multiple comparisons** | Testing 20 metrics inflates false positives | Bonferroni correction (α/m) or Benjamini-Hochberg FDR |
| **Novelty effect** | Users behave differently just because something is new | Run test long enough to outlast novelty period |
| **Sample ratio mismatch (SRM)** | Assignment split ≠ planned split (e.g., 52/48 instead of 50/50) | Investigate logging/bucketing bugs before trusting results |
| **Network effects** | Control and treat groups contaminate each other | Use cluster-level randomization |
| **Simpson's paradox** | Aggregated trend reverses when data is segmented | Always stratify and segment analysis |

**In plain English:** Peeking is the most common exam question. Each additional look at the data is an extra chance to see a spurious p < 0.05 by chance. With daily checks over 14 days, your effective false positive rate can exceed 25% even when α = 0.05.

---

## Correlation vs Causation

**Correlation** measures linear association between two variables. **Causation** means X directly produces a change in Y.

Spurious correlations are everywhere: ice cream sales correlate with drowning rates (common cause: summer). To establish causation, you need:
1. Correlation
2. Temporal precedence (X before Y)
3. Elimination of confounders

**Methods for causal inference:** randomized controlled trials (A/B tests), instrumental variables, difference-in-differences, regression discontinuity.

---

## Covariance vs Correlation

**Covariance:** Cov(X,Y) = E[(X−μX)(Y−μY)] — measures joint variability. Units are product of X and Y units (hard to interpret).

**Pearson Correlation:** ρ = Cov(X,Y) / (σX × σY) — normalized to [−1, 1]. Unit-free and interpretable.

- ρ = 1: perfect positive linear relationship
- ρ = 0: no linear relationship (may still have nonlinear relationship)
- ρ = −1: perfect negative linear relationship

**Spearman correlation** uses ranks instead of raw values — robust to outliers and captures monotonic (not just linear) relationships.

---

## MLE vs MAP

Both are methods for estimating model parameters.

| | MLE | MAP |
|--|-----|-----|
| **Maximizes** | Likelihood P(data \| θ) | Posterior P(θ \| data) ∝ P(data\|θ) × P(θ) |
| **Prior** | None (implicit uniform) | Explicit prior over θ |
| **Regularization** | None | Gaussian prior → L2 (Ridge); Laplace prior → L1 (Lasso) |
| **Overfitting** | More prone with small n | Less prone (prior acts as regularizer) |
| **Output** | Point estimate | Point estimate (mode of posterior) |

**In plain English:**
- MLE: "What parameter value makes the observed data most likely?"
- MAP: "What parameter value is most likely given both the data and my prior belief?"
- MAP = MLE when the prior is uniform.
- Full Bayesian inference goes further: instead of a point estimate, it gives you the entire posterior distribution over θ.

---

## Bayesian vs Frequentist

| | Frequentist | Bayesian |
|--|-------------|----------|
| **Probability** | Long-run frequency of events | Degree of belief / uncertainty |
| **Parameters** | Fixed, unknown constants | Random variables with distributions |
| **Data** | Random (repeated sampling) | Fixed (what you observed) |
| **Output** | p-values, confidence intervals | Posterior distributions, credible intervals |
| **Prior** | Not used | Explicitly incorporated |
| **Strength** | Objective, reproducible | Incorporates domain knowledge; natural uncertainty quantification |

**Interview angle:** In industry, frequentist methods dominate A/B testing for simplicity. Bayesian methods are used when you have strong priors, small samples, or need full uncertainty quantification (e.g., multi-armed bandits, Bayesian optimization).

---

## Bootstrapping and Resampling

**Bootstrapping** estimates the sampling distribution of a statistic by repeatedly resampling **with replacement** from the original data.

**Algorithm:**
1. Draw B samples of size n (with replacement) from your dataset.
2. Compute your statistic (mean, median, regression coefficient) on each sample.
3. Use the distribution of those B statistics to estimate SE and build CIs.

**Why it matters:** Works when you don't know the population distribution. No analytical formula needed. Great for complex statistics (e.g., median, ratio, AUC).

**Bootstrap 95% CI:** Take the 2.5th and 97.5th percentile of the bootstrap distribution.

---

## Standard Error

SE = σ / √n

Standard error is the standard deviation of the **sampling distribution of the mean** — it quantifies how much X̄ varies from sample to sample. As n grows, SE shrinks (estimates become more precise). Do not confuse SE with population standard deviation σ.

---

## Multicollinearity and VIF

**Multicollinearity:** When two or more predictor variables in a regression model are highly correlated. This makes coefficient estimates unstable and hard to interpret (standard errors inflate).

**Variance Inflation Factor (VIF):**

VIF_j = 1 / (1 − R²_j)

where R²_j is the R² from regressing predictor j on all other predictors.

| VIF | Interpretation |
|-----|----------------|
| 1 | No multicollinearity |
| 1–5 | Moderate, usually acceptable |
| > 5 | Concerning |
| > 10 | Severe, action needed |

**Fixes:** Drop one of the correlated predictors; PCA to decorrelate; Ridge regression.

---

## R² vs Adjusted R²

**R²** = 1 − SS_res / SS_tot. Measures fraction of variance explained. Always increases (or stays the same) when you add predictors — even useless ones.

**Adjusted R²** = 1 − (1 − R²)(n − 1)/(n − k − 1)

Penalizes for number of predictors k. Can decrease if a new predictor adds less than expected by chance. **Use adjusted R² for model comparison.**

---

## P-Hacking

P-hacking (or data dredging) is running many tests and selectively reporting only the ones with p < 0.05. With 20 independent tests at α = 0.05, you expect **1 false positive** purely by chance.

**Signs of p-hacking:**
- Testing many subgroups and reporting only significant ones.
- Stopping data collection as soon as p < 0.05 (peeking).
- Trying multiple transformations until significance is achieved.

**Defenses:** Pre-registration (state hypotheses before data collection); Bonferroni correction; separating exploratory from confirmatory analysis; reporting all tests run.

---

## Interview Questions

### Q: What is a p-value and what does it NOT tell you?
**Strong answer:**
The p-value is the probability of observing data at least as extreme as what we saw, assuming the null hypothesis is true. It does NOT tell you the probability that the null is true, the probability your result is a false positive, or the practical significance of the effect. A tiny p-value with a huge sample can correspond to a negligibly small effect size that has no business impact.

### Q: Explain the Central Limit Theorem and why it matters in practice.
**Strong answer:**
The CLT states that the sampling distribution of the sample mean approaches a normal distribution as n increases, regardless of the population's shape (provided finite variance). In practice this means we can apply z-tests and t-tests to non-normal data as long as samples are large enough (n ≥ 30 as a rule of thumb). It also underpins confidence interval construction and the validity of many hypothesis tests.

### Q: What is the difference between Type I and Type II errors? Which is worse?
**Strong answer:**
A Type I error is rejecting a true null (false positive, probability = α). A Type II error is failing to reject a false null (false negative, probability = β). Which is worse depends on context: in medical testing, a Type II error (missing a real disease) may be more dangerous; in spam filtering, a Type I error (blocking a real email) may be more costly. We control the trade-off by adjusting α and sample size.

### Q: How do you design an A/B test from scratch?
**Strong answer:**
First define a single primary metric and write the null/alternative hypotheses. Set α (typically 0.05) and desired power (0.8 or 0.9). Define the minimum detectable effect based on business needs. Use a sample size calculator to determine n per variant. Randomly assign users, run the test until the planned sample is reached — no early stopping. Validate the assignment ratio (check for SRM), then compute the p-value and confidence interval. Report effect size alongside statistical significance.

### Q: What is Bayes' theorem? Give a practical example.
**Strong answer:**
Bayes' theorem states P(H|E) = P(E|H) × P(H) / P(E). It lets us update a prior belief with new evidence to get a posterior. Classic example: a disease with 1% prevalence, a test that is 99% sensitive and 95% specific. A positive test result only gives you a ~17% chance of actually having the disease — because the base rate is so low that false positives dominate. This base-rate fallacy is tested frequently.

### Q: What is the difference between MLE and MAP estimation?
**Strong answer:**
MLE maximizes P(data | θ) — it picks the parameter that makes the observed data most likely, with no prior assumption. MAP maximizes P(data | θ) × P(θ) — it adds a prior over θ, which acts as regularization. A Gaussian prior on coefficients corresponds to L2 (Ridge) regularization; a Laplace prior corresponds to L1 (Lasso). MAP reduces to MLE when the prior is uniform. Full Bayesian inference returns the entire posterior distribution rather than a point estimate.

### Q: What is the difference between correlation and causation? How do you establish causation?
**Strong answer:**
Correlation means two variables move together; causation means one variable directly produces a change in the other. Correlation can be spurious (ice cream sales and drowning rates both rise in summer — caused by heat). To establish causation you need temporal precedence, correlation, and elimination of confounders. The gold standard is a randomized controlled trial. Observational approaches include instrumental variables, difference-in-differences, and regression discontinuity.

### Q: What is a confidence interval and how do you interpret it correctly?
**Strong answer:**
A 95% CI does not mean "there's a 95% probability the parameter is in this interval." The parameter is fixed; the interval is random. Correct interpretation: if we repeated this experiment 100 times and built a CI each time, about 95 of those intervals would contain the true parameter. In practice, we report CIs alongside p-values because they convey both statistical significance and practical magnitude of the effect.

### Q: What are the pitfalls of A/B testing, and how do you handle the peeking problem?
**Strong answer:**
The main pitfalls are peeking (checking results early inflates Type I error rate), multiple comparisons (testing many metrics), novelty effects, network effects, and sample ratio mismatch. For peeking, the best fix is committing to a sample size upfront and not making ship/kill decisions until that target is reached. Alternatively use sequential testing methods (always-valid p-values or group sequential tests) which are mathematically sound even with multiple looks.

### Q: What is bootstrapping and when would you use it?
**Strong answer:**
Bootstrapping is a resampling technique: draw many samples of size n with replacement from the original data, compute your statistic on each, and use the resulting distribution to estimate standard error and build confidence intervals. Use it when no analytical formula exists (e.g., median, model AUC, complex ratios), when the population distribution is unknown, or when your sample is too small to rely on asymptotic normality.

### Q: What is multicollinearity and how do you detect and fix it?
**Strong answer:**
Multicollinearity occurs when two or more predictors in a regression are highly correlated, inflating coefficient standard errors and making estimates unstable. Detect it with Variance Inflation Factors (VIF > 5–10 is problematic). Fixes include dropping one of the correlated predictors, combining them (e.g., PCA), or using Ridge regression which is robust to multicollinearity by adding L2 regularization.

### Q: What is the difference between R² and adjusted R²?
**Strong answer:**
R² measures the fraction of variance in the target explained by the model. It always increases when you add more predictors, even useless ones. Adjusted R² penalizes for the number of predictors: it can decrease if an added variable contributes less explanatory power than expected by chance. Always use adjusted R² (or information criteria like AIC/BIC) when comparing models with different numbers of features.

### Q: What is p-hacking and how do you defend against it?
**Strong answer:**
P-hacking is running many statistical tests (across metrics, subgroups, time windows) and selectively reporting only significant results. With 20 independent tests at α = 0.05, you expect one false positive by chance. Defenses: pre-register hypotheses before data collection; apply Bonferroni correction (use α/m as threshold) or control the False Discovery Rate (Benjamini-Hochberg); strictly separate exploratory analysis from confirmatory testing.

### Q: What is the Law of Large Numbers vs Central Limit Theorem?
**Strong answer:**
The Law of Large Numbers says the sample mean converges to the population mean μ as n → ∞ — it's about a value converging. The Central Limit Theorem says the distribution of the sample mean converges to a normal distribution — it's about a shape emerging. Both rely on independent, identically distributed samples with finite variance, but they answer different questions: LLN ensures accuracy, CLT enables inference.

---

## References

- [Top 35 Statistics Interview Questions — DataCamp](https://www.datacamp.com/blog/statistics-interview-questions)
- [40 Probability & Statistics Data Science Interview Questions (FAANG & Wall Street) — Nick Singh](https://www.nicksingh.com/posts/40-probability-statistics-data-science-interview-questions-asked-by-fang-wall-street)
- [Top 30 A/B Testing Interview Questions — DataInterview](https://www.datainterview.com/blog/ab-testing-interview-questions)
- [Full Explanation of MLE, MAP and Bayesian Inference — Towards Data Science](https://towardsdatascience.com/full-explanation-of-mle-map-and-bayesian-inference-1db9a7fb1d2b/)
- [Variance Inflation Factors (VIFs) — Statistics By Jim](https://statisticsbyjim.com/regression/variance-inflation-factors/)
- [15 Probability and Statistics Interview Questions Every Data Scientist Must Master — Analytics Vidhya](https://www.analyticsvidhya.com/blog/2026/02/probability-and-statistics-interview-questions/)

---

---

## Glossary

| Term | Simple explanation | Purpose |
|---|---|---|
| **Mean** | The sum of all values divided by the count | Summarises the centre of a distribution; sensitive to outliers |
| **Median** | The middle value when data is sorted | A robust measure of centre; preferred over mean when outliers are present |
| **Mode** | The most frequently occurring value in a dataset | Useful for categorical data and identifying peaks in multi-modal distributions |
| **Variance** | The average squared deviation of each value from the mean | Quantifies how spread out the data is; used in many statistical tests |
| **Standard Deviation** | The square root of the variance, expressed in the same units as the data | More interpretable than variance for describing data spread |
| **IQR (Interquartile Range)** | The difference between the 75th percentile (Q3) and the 25th percentile (Q1) | A robust measure of spread that is unaffected by extreme outliers |
| **Skewness** | A measure of the asymmetry of a distribution's tails | Positive skew indicates a long right tail; negative skew indicates a long left tail |
| **Kurtosis** | A measure of how heavy the tails of a distribution are relative to a normal distribution | High kurtosis means extreme values occur more often than a normal distribution would predict |
| **Conditional Probability** | The probability of event A given that event B has already occurred | Allows updating beliefs when partial information is known |
| **Independence (Probability)** | Two events are independent if knowing one tells you nothing about the other | A foundational assumption in many statistical models including Naive Bayes |
| **Expected Value** | The probability-weighted average of all possible outcomes of a random variable | Represents the long-run average result of a random process |
| **Bayes' Theorem** | A formula for updating the probability of a hypothesis given new evidence: P(H\|E) = P(E\|H) × P(H) / P(E) | The mathematical foundation for probabilistic reasoning and Bayesian inference |
| **Prior Probability** | Your belief in a hypothesis before seeing any evidence | The starting point in Bayesian reasoning; reflects existing domain knowledge |
| **Likelihood** | The probability of observing the data assuming a particular hypothesis is true | The bridge between a model's parameters and the observed data in statistical inference |
| **Posterior Probability** | Your updated belief in a hypothesis after incorporating the evidence | The output of applying Bayes' theorem; combines prior and likelihood |
| **Base-Rate Fallacy** | Ignoring how rare an event is when interpreting a test result, leading to over-confidence in the result | A common error in medical testing; highlights the importance of priors |
| **Bernoulli Distribution** | A distribution with two outcomes (success or failure) characterised by probability p | Models a single binary trial; the building block for the Binomial distribution |
| **Binomial Distribution** | The distribution of the number of successes in n independent Bernoulli trials | Used to model counts of successes across repeated binary experiments |
| **Poisson Distribution** | The distribution of the number of events occurring in a fixed time or space when events happen at a constant average rate | Models rare event counts such as server errors per hour or customer arrivals per minute |
| **Normal (Gaussian) Distribution** | A symmetric bell-shaped distribution fully described by its mean and standard deviation | Arises naturally from sums of many random variables; underpins most statistical tests |
| **Exponential Distribution** | A continuous distribution modelling the time between consecutive Poisson events | Has the memoryless property: past wait time does not affect future wait time |
| **Central Limit Theorem (CLT)** | The theorem stating that the sample mean of many independent draws approaches a normal distribution regardless of the underlying population shape | Justifies using normal-based tests (z-test, t-test) on non-normal data when samples are large |
| **Standard Error (SE)** | The standard deviation of the sampling distribution of the mean, equal to σ / √n | Quantifies the precision of a sample mean estimate; shrinks as sample size grows |
| **Law of Large Numbers (LLN)** | As sample size increases, the sample mean converges to the true population mean | Guarantees that larger experiments produce more reliable estimates |
| **Selection Bias** | A systematic error arising when the sample is not representative of the population | Leads to models that work on the sample but fail on the real population |
| **Survivorship Bias** | Analysing only units that "survived" a filter, ignoring those that did not | Produces misleadingly optimistic conclusions; e.g., studying successful companies only |
| **Null Hypothesis (H₀)** | The default assumption that no effect or difference exists | The claim that hypothesis testing attempts to find evidence against |
| **Alternative Hypothesis (H₁)** | The claim that an effect or difference does exist | What you are trying to demonstrate with your experiment or test |
| **P-Value** | The probability of observing data at least as extreme as the actual data, assuming the null hypothesis is true | If p ≤ α, you reject the null; small p-values are evidence against H₀ |
| **Significance Level (α)** | The pre-set threshold below which the p-value is considered significant, typically 0.05 | Controls the acceptable rate of Type I errors |
| **Type I Error** | Rejecting a true null hypothesis (a false positive) | Controlled by the significance level α; e.g., concluding a useless drug works |
| **Type II Error** | Failing to reject a false null hypothesis (a false negative) | Controlled by statistical power; e.g., missing a drug that actually works |
| **Statistical Power** | The probability of correctly detecting a real effect (1 − β) | Increases with larger sample sizes or larger true effect sizes |
| **Confidence Interval (CI)** | A range of values that, over many repeated experiments, would contain the true parameter a specified percentage of the time | Conveys both the estimate and its precision; narrower CI = more precise estimate |
| **z-Test** | A hypothesis test for comparing means when the sample size is large and the population variance is known | Applied when CLT guarantees approximately normal sampling distributions |
| **t-Test** | A hypothesis test for comparing means with small samples where the population variance is unknown | More conservative than the z-test; uses the t-distribution with heavier tails |
| **Chi-Square Test** | A test for association between two categorical variables based on comparing observed and expected cell counts | Common test for independence in contingency tables |
| **ANOVA** | A test comparing the means of three or more groups simultaneously | Avoids inflating the false-positive rate that would result from running multiple t-tests |
| **Mann-Whitney U Test** | A non-parametric test comparing two independent groups when normality cannot be assumed | Used in place of the t-test for ordinal data or small non-normal samples |
| **A/B Testing** | A randomised experiment comparing a control group to a treatment group on a single change | The primary method for making data-driven product decisions |
| **Minimum Detectable Effect (MDE)** | The smallest true effect size an experiment is designed to detect reliably | Determines the required sample size; smaller MDE = larger experiment needed |
| **Sample Ratio Mismatch (SRM)** | When the observed split between control and treatment groups differs significantly from the planned split | Indicates a logging or bucketing bug; results should not be trusted until resolved |
| **Peeking** | Stopping an A/B test early because the p-value crossed the significance threshold | Inflates the effective false-positive rate; requires sequential testing methods to do correctly |
| **Multiple Comparisons Problem** | Testing many hypotheses simultaneously inflates the overall chance of at least one false positive | Addressed with Bonferroni correction (α/m) or the Benjamini-Hochberg FDR procedure |
| **P-Hacking** | Running many tests and selectively reporting only those with p < α | Leads to false discoveries; defended against through pre-registration and reporting all tests |
| **Bonferroni Correction** | Dividing the significance threshold α by the number of tests m when testing multiple hypotheses | Keeps the family-wise error rate below α; conservative when tests are correlated |
| **Pearson Correlation** | A normalised measure of the linear relationship between two variables, ranging from −1 to 1 | Standard correlation metric; sensitive to outliers and captures only linear relationships |
| **Spearman Correlation** | A correlation metric based on ranks rather than raw values | Robust to outliers and captures monotonic (not just linear) relationships |
| **Covariance** | A measure of the joint variability of two variables expressed in the product of their units | Foundational to correlation and PCA; harder to interpret than correlation due to units |
| **Causation** | A relationship where one variable directly produces a change in another | Established through randomised experiments or causal inference methods, not correlation alone |
| **Confounder** | A variable that is correlated with both the treatment and the outcome, creating a spurious association | Must be controlled for to avoid drawing incorrect causal conclusions |
| **MLE (Maximum Likelihood Estimation)** | Estimating model parameters by maximising the probability of the observed data under the model | The standard estimation method; equivalent to MAP with a uniform prior |
| **MAP (Maximum a Posteriori)** | Estimating model parameters by maximising the posterior, which incorporates a prior over parameters | Adding a Gaussian prior is equivalent to L2 regularisation; Laplace prior gives L1 |
| **Bayesian Inference** | Computing the full posterior distribution over parameters rather than just a point estimate | Provides complete uncertainty quantification; used in small-data and multi-armed bandit settings |
| **Bootstrapping** | Resampling the dataset with replacement many times and computing a statistic on each resample | Estimates sampling distributions without assuming any parametric form |
| **Multicollinearity** | High correlation among predictor variables in a regression model, inflating coefficient standard errors | Makes individual coefficients unstable and hard to interpret; detected with VIF |
| **VIF (Variance Inflation Factor)** | A score measuring how much a predictor's variance is inflated by collinearity with other predictors | VIF > 5–10 signals a collinearity problem requiring action |
| **R² (R-squared)** | The fraction of variance in the target explained by the model | Always increases with more predictors; use adjusted R² for model comparison |
| **Adjusted R²** | R² penalised for the number of predictors to avoid rewarding unnecessary complexity | Can decrease when adding a predictor that contributes less than expected by chance |
| **AIC / BIC** | Information criteria that balance model fit against complexity as an alternative to adjusted R² | Used to compare models; lower values indicate a better trade-off between fit and simplicity |

*Previous: [Deep Learning Fundamentals](02-deep-learning-fundamentals.md) | Next: [ML System Design](04-ml-system-design.md)*
