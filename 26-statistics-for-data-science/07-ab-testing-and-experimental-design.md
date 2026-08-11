# A/B Testing and Experimental Design

Online controlled experiments are the gold standard for measuring the causal effect of a product change. This guide walks through every stage—from formulating a hypothesis to analyzing results and handling the many pitfalls that invalidate real-world experiments—at the depth expected in a product data science interview.

> **In plain English:** An A/B test randomly splits users into two groups, shows each group a different experience, and measures whether the difference in outcomes is large enough to be unlikely by chance. The randomization is what lets you claim *causation* rather than mere correlation.

## Table of Contents

1. [Causal vs. Observational Inference](#1-causal-vs-observational-inference)
2. [Designing an A/B Test](#2-designing-an-ab-test)
3. [Power Analysis and Sample Size](#3-power-analysis-and-sample-size)
4. [Running the Test](#4-running-the-test)
5. [Analyzing Results](#5-analyzing-results)
6. [Pitfalls](#6-pitfalls)
   - 6.1 Peeking and Early Stopping
   - 6.2 Multiple Testing
   - 6.3 Simpson's Paradox in Segments
   - 6.4 Sample Ratio Mismatch (SRM)
   - 6.5 Interference and Network Effects
   - 6.6 Twyman's Law
7. [Variance Reduction: CUPED](#7-variance-reduction-cuped)
8. [Sequential Testing and Always-Valid P-values](#8-sequential-testing-and-always-valid-p-values)
9. [Bayesian A/B Testing](#9-bayesian-ab-testing)
10. [Quasi-Experiments](#10-quasi-experiments)
11. [🎯 In the Interview](#-in-the-interview)
12. [Glossary](#glossary)
13. [References](#references)

---

## 1. Causal vs. Observational Inference

**The fundamental problem of causal inference** is that you can never simultaneously observe the same user in both the treated and untreated states. Every method for causal estimation is a strategy for approximating the counterfactual.

| Approach | Counterfactual strategy | Key threat |
|---|---|---|
| Observational / regression | Statistical control of confounders | Omitted variable bias, selection bias |
| A/B test (RCT) | Randomization makes groups exchangeable | Interference, non-compliance |
| Quasi-experiment (DiD, IV, RD) | Natural variation as instrument | Assumption violations (parallel trends, exclusion restriction) |

**Why experiment?** Any observational comparison of "users who saw feature X" vs. "users who didn't" confounds the feature effect with self-selection. Users who opt into a feature differ systematically from those who don't—this is selection bias. Randomization breaks the link between treatment assignment and any pre-existing characteristic, making the two groups statistically identical in expectation.

---

## 2. Designing an A/B Test

### 2.1 Hypothesis

State hypotheses before data collection:

- **H₀ (null):** The treatment has no effect on the primary metric (δ = 0).
- **H₁ (alternative):** The treatment changes the primary metric by at least MDE (δ ≥ MDE).

Frame the alternative as one-tailed (you expect improvement) or two-tailed (direction unknown). Two-tailed tests are safer for guardrail metrics; one-tailed tests are used only when you are certain about direction and want more power.

### 2.2 Primary and Guardrail Metrics

- **Primary metric:** The one metric the test is powered to detect. Changing it is the definition of success. Examples: 7-day retention rate, revenue per user, click-through rate.
- **Secondary metrics:** Directional signals you report but do not use as stopping criteria.
- **Guardrail metrics:** Metrics that must not degrade. Examples: app crash rate, page load time, support ticket rate. A test passes only if (a) primary metric improves significantly and (b) guardrail metrics do not significantly worsen.

> **Why / When to use / Nuances:** Guardrail metrics catch "metric hacking"—a feature that improves clicks by making everything clickbait, for instance. Always specify guardrails before launch.

### 2.3 Randomization Unit

The randomization unit determines what gets independently assigned to treatment or control.

| Unit | Use when | Tradeoff |
|---|---|---|
| User | Most common; ensures stable experience | Cannot test features affecting cross-user interactions |
| Session | Short-lived interventions (e.g., ad ranking) | Same user sees both arms → carryover contamination |
| Cookie | Anonymous traffic | Cookie deletion inflates "new user" counts |
| Device | Mobile-native features | Multi-device users get inconsistent experiences |
| Geo / market | Network products, marketplace features | Fewer randomization units → low power |

**The SUTVA assumption** (Stable Unit Treatment Value Assumption) requires that one unit's treatment does not affect another unit's outcome. Network products violate SUTVA; see §6.5 for solutions.

### 2.4 Control and Treatment

- **Control:** Current production experience (or a holdout with no feature).
- **Treatment:** The variant being tested.
- Use a **single primary treatment** per experiment where possible. Multi-arm tests require multiple-testing corrections.
- **Ramp carefully:** Start at 1–10% to catch crashes or data quality issues before full exposure.

---

## 3. Power Analysis and Sample Size

### 3.1 The Four Drivers

| Parameter | Symbol | Typical default | Effect on sample size |
|---|---|---|---|
| Significance level (Type I error rate) | α | 0.05 | Lower α → larger n |
| Statistical power (1 − Type II error rate) | 1 − β | 0.80 | Higher power → larger n |
| Baseline rate / variance | p₀ or σ² | From historical data | Higher variance → larger n |
| Minimum detectable effect | MDE | Business judgment | Smaller MDE → larger n |

### 3.2 Sample Size Formula

**For a two-proportion z-test** (binary metric, e.g., conversion rate):

```
n = (z_{α/2} + z_β)² × [p₀(1−p₀) + p₁(1−p₁)] / MDE²
```

where p₁ = p₀ + MDE and z_{α/2} = 1.96 (for α = 0.05, two-tailed), z_β = 0.84 (for 80% power).

**For a t-test on a continuous metric** (e.g., revenue per user with variance σ²):

```
n = 2 × (z_{α/2} + z_β)² × σ² / MDE²
```

**Key intuition:** n scales as **1/MDE²**. Halving the MDE quadruples the required sample size. This is the single most important lever when experiments are constrained by traffic volume. ([Statsig, 2024](https://www.statsig.com/perspectives/power-analysis-ab-testing))

### 3.3 Worked Example

Baseline conversion rate p₀ = 0.10 (10%). MDE = 0.01 (absolute lift to 11%). α = 0.05 (two-tailed), power = 80%.

```
p₁ = 0.11
z_{α/2} = 1.96, z_β = 0.84
numerator = (1.96 + 0.84)² × [0.10×0.90 + 0.11×0.89]
          = 7.84 × [0.09 + 0.0979]
          = 7.84 × 0.1879 ≈ 1.47
n = 1.47 / 0.01² = 1.47 / 0.0001 = 14,700 per arm
```

You need roughly **14,700 users per arm** (29,400 total) to detect a 1pp lift.

> **Why / When to use / Nuances:** If your traffic is limited, you have three levers: (a) increase MDE (accept only detecting larger effects), (b) raise α to 0.10 (accept more false positives), (c) use variance reduction (CUPED, §7) to shrink σ² without more traffic.

---

## 4. Running the Test

### 4.1 Test Duration

Minimum duration considerations:
- **Statistical:** Run until the pre-planned sample size is reached.
- **Seasonality:** Always run for **at least one full week** (ideally two) to capture Mon–Sun behavioral variation. A test run only Mon–Wed will be biased toward weekday users.
- **Novelty effect:** Users may interact more with *any* change purely because it is new. This inflates early treatment metrics but fades over days/weeks. Measure whether treatment effect is stable toward the end of the experiment.
- **Primacy effect:** Conversely, loyal power users may perform better in control (familiar UI) early on, understating the long-term treatment benefit.

> **Why / When to use / Nuances:** For features targeting habit formation (e.g., daily active use), novelty effects can dominate for 1–2 weeks. Run longer or analyze only the tail of the experiment to get the steady-state effect.

### 4.2 Pre-experiment Checks

Before the experiment starts:
1. Verify randomization is working (run an A/A test; p-values should be uniform).
2. Confirm logging pipelines are firing correctly.
3. Lock down the assignment hash and salt so users are consistently bucketed.
4. Document the start timestamp; events before that timestamp belong to pre-period analysis only.

---

## 5. Analyzing Results

### 5.1 Two-Proportion Z-Test

For binary metrics (conversion rate, click-through rate):

```
p̂_c = conversions_c / n_c
p̂_t = conversions_t / n_t
p̂_pooled = (conversions_c + conversions_t) / (n_c + n_t)

SE = sqrt[ p̂_pooled × (1 − p̂_pooled) × (1/n_c + 1/n_t) ]

z = (p̂_t − p̂_c) / SE
```

Reject H₀ if |z| > z_{α/2} (e.g., 1.96 for α = 0.05, two-tailed).

### 5.2 Two-Sample T-Test

For continuous metrics (revenue per user, session length):

```
t = (ȳ_t − ȳ_c) / sqrt(s_t²/n_t + s_c²/n_c)
```

With large samples (n > 30 per arm), the t-distribution converges to normal by CLT.

### 5.3 Confidence Intervals on Lift

Report the **95% CI on absolute lift** (not just the p-value):

```
Lift = p̂_t − p̂_c
95% CI = Lift ± z_{0.025} × SE_lift
```

Relative lift = Lift / p̂_c. Prefer absolute lift for decision-making; relative lift can be misleading for rare events (a 50% relative lift on a 0.002% baseline is still tiny).

> **Why / When to use / Nuances:** A p-value tells you whether an effect exists; a CI tells you how big it might be. Always report both. A tiny, statistically significant effect may be commercially irrelevant.

---

## 6. Pitfalls

### 6.1 Peeking and Early Stopping

**The problem:** Every time you check the p-value, you give yourself another chance to cross α = 0.05. If you check daily and stop as soon as p < 0.05, your actual false positive rate can exceed 20–30% even though you intended α = 5%.

**Why it happens:** Under H₀, p-values are uniformly distributed. The running minimum of a sequence of uniform draws tends toward zero, meaning some peeking schedule will always produce p < 0.05 by chance.

**Fixes:**
- Pre-register the end date and do not stop early.
- Use sequential testing / always-valid p-values (§8).
- Use Bonferroni correction across interim looks (conservative).

### 6.2 Multiple Testing

**The problem:** Testing k metrics or k variants simultaneously inflates the family-wise error rate (FWER). With k = 20 independent tests at α = 0.05, the probability of at least one false positive is 1 − 0.95²⁰ ≈ 64%.

**Corrections:**

| Method | Controls | Formula | When to use |
|---|---|---|---|
| **Bonferroni** | FWER | α* = α / k | Confirmatory, few hypotheses, independent tests |
| **Benjamini-Hochberg (BH)** | FDR | Rank p-values; reject p_(k) ≤ (k/m)·α | Exploratory, many metrics, some correlation allowed |

**Bonferroni** is conservative (assumes independence and penalizes all tests equally). **BH** (introduced in the 1995 *Journal of the Royal Statistical Society* paper "Controlling the false discovery rate: A practical and powerful approach to multiple testing") controls the *expected proportion* of false discoveries among all rejections (FDR), making it more powerful for large metric families. ([Benjamini & Hochberg, 1995 via ResearchGate](https://www.researchgate.net/post/How_to_carry_out_the_Benjamini-Hochberg_procedure_for_multiple_testing))

> **Why / When to use / Nuances:** In product experimentation, most companies use Bonferroni for a handful of primary metrics and accept higher FDR for exploratory secondary metrics. BH is standard in genomics and increasingly adopted in large-scale experimentation platforms. Never apply corrections post-hoc to only the metrics that "almost" reached significance.

**BH procedure (worked example):**
- 5 metrics, target FDR = 0.05.
- Sorted p-values: 0.003, 0.012, 0.031, 0.064, 0.210.
- Thresholds: k/m × α = 0.01, 0.02, 0.03, 0.04, 0.05.
- Largest k where p_(k) ≤ threshold: k=2 (p=0.012 ≤ 0.02). Reject hypotheses 1 and 2.

### 6.3 Simpson's Paradox in Segments

A treatment can appear to underperform overall while outperforming in *every* segment—or vice versa—when segment composition differs between arms. This happens if randomization was imperfect or if you are cutting post-hoc segments of unequal size.

**Example:** Mobile users convert at 5%, desktop at 15%. If treatment arm has 70% mobile users vs. control's 40%, the overall treatment average will be lower even if treatment improved conversion in both segments.

**Fix:** Pre-stratify randomization by important dimensions (platform, country, user tier). Verify balance tables before analysis. Analyze segments only as pre-registered secondary analyses.

### 6.4 Sample Ratio Mismatch (SRM)

**Definition:** SRM occurs when the actual ratio of users in treatment vs. control differs significantly from the assigned ratio (e.g., you assigned 50/50 but observe 52/48).

**Detection:** Run a chi-squared test on actual vs. expected assignment counts. p < 0.05 indicates SRM.

**Causes:** Logging bugs, bot filtering applied only to one arm, redirect latency causing some users to bail before being tracked, overlapping experiments assigning the same user twice.

**Action:** Do not analyze the experiment until SRM is resolved. SRM invalidates the randomization guarantee and can bias effect estimates in any direction.

### 6.5 Interference and Network Effects

On social networks, marketplaces, and ride-sharing platforms, treating one user affects others (their friends, riders, drivers). This violates SUTVA and biases estimated treatment effects—typically *downward* because control users benefit from the network spill.

**Solutions:**

| Method | Mechanism | Tradeoff |
|---|---|---|
| **Cluster randomization** | Randomize at the community / geographic level | Fewer independent units → much lower power |
| **Ego-network randomization** | Assign treatment to user + all friends | Reduces contamination; cannot detect spillover magnitude |
| **Switchback / time-based experiments** | Alternate treatment and control over time periods | Eliminates cross-user interference; introduces time confounds |
| **Graph cluster techniques** | Detect communities; assign entire community | Depends on community structure quality |

> **Why / When to use / Nuances:** Cluster randomization is the right default for any product with strong social ties. The variance of cluster-level estimates is much higher than user-level variance, so you must account for cluster-level correlation in the SE calculation (use clustered standard errors or HLM).

### 6.6 Twyman's Law

> *"The more unusual or interesting the data, the more likely it is to be wrong."*

Coined by Tony Twyman. Extremely large measured effects (e.g., +40% conversion) usually indicate a tracking bug, an SRM, or a logging issue rather than a real causal effect. Always sanity-check implausibly large results before shipping.

---

## 7. Variance Reduction: CUPED

**Motivation:** A larger sample is not the only way to get a narrower confidence interval. If you can *reduce the variance* of your metric estimator, you achieve the same effect with fewer users—or detect smaller effects with the same sample.

**CUPED (Controlled-experiment Using Pre-Experiment Data)** was introduced by Deng, Xu, Kohavi, and Walker at Microsoft (2013): "Improving the Sensitivity of Online Controlled Experiments by Utilizing Pre-Experiment Data." ([Semantic Scholar](https://www.semanticscholar.org/paper/Improving-the-sensitivity-of-online-controlled-by-Deng-Xu/c65d71c42dedb3329c2b81499950bf12786a3e8e))

**Core idea:**

```
Y_cuped = Y − θ × X_pre
```

where:
- Y = post-experiment outcome for the user.
- X_pre = pre-experiment value of the same metric (e.g., revenue in the prior 2 weeks).
- θ = Cov(Y, X_pre) / Var(X_pre) — the OLS regression coefficient.

Because X_pre is independent of treatment assignment (it was measured before randomization), subtracting θ·X_pre removes the portion of outcome variance explained by pre-existing user behavior without introducing bias.

**Variance reduction:**

```
Var(Y_cuped) = Var(Y) × (1 − ρ²)
```

where ρ = Cor(Y, X_pre). If ρ = 0.7 (common for revenue metrics), variance is reduced by 51%, equivalent to roughly **doubling your effective sample size**. In practice, Microsoft and Netflix report 30–50% variance reductions. ([Stanford CUPED overview, Matteocourthoud.github.io](https://matteocourthoud.github.io/post/cuped/))

**Requirements:**
- X_pre must be collected before experiment assignment.
- The higher the correlation between pre- and post-period, the larger the reduction.
- Works for any metric—binary, continuous, revenue.

---

## 8. Sequential Testing and Always-Valid P-values

**The problem with fixed-horizon testing:** You must commit to a sample size *before* the experiment. You cannot stop early (even when the effect is obvious) without inflating the false positive rate—see §6.1.

**Sequential testing** constructs test statistics that remain valid at *any* stopping time, permitting continuous monitoring.

**Always Valid Inference (Johari, Pekelis, Walsh, 2015/2021):** Published in *Operations Research* ([arXiv:1512.04922](https://arxiv.org/abs/1512.04922)), this framework defines **always-valid p-values** using the mixture Sequential Probability Ratio Test (mSPRT), building on Robbins (1970).

**Intuition:**
- Instead of computing a fresh p-value at each look, maintain a running **likelihood ratio** (evidence score) that accumulates across the full data stream.
- The likelihood ratio is a martingale under H₀, so Ville's inequality bounds the probability that it ever exceeds 1/α—giving type I error control at any stopping time.
- The cost: slightly larger expected sample size versus a perfectly-timed fixed-horizon test, but the experiment can stop as soon as strong evidence accumulates.

> **Why / When to use / Nuances:** Sequential testing is ideal for live products where waiting weeks for a fixed-horizon test is costly. It's deployed by Optimizely's "Stats Engine," Statsig, and similar platforms. The tradeoff: slightly lower power at any *fixed* sample size compared to a classical test (because the test must guard against all future peek points). Use fixed-horizon testing when you can truly commit to a single analysis; use sequential testing when business pressure will cause people to peek anyway.

---

## 9. Bayesian A/B Testing

**Framework:** Instead of computing p-values, maintain a posterior distribution over the treatment effect δ.

- Choose prior: Beta(α₀, β₀) for conversion rates (often a weak prior matching the historical rate).
- After observing data, update to posterior: Beta(α₀ + conversions, β₀ + non-conversions) per arm.
- Decision rule: Stop when P(δ_treatment > δ_control) > threshold (e.g., 95%) or when the **expected loss** from choosing incorrectly drops below a business threshold.

**Advantages:**
- Directly answers "What is the probability that treatment is better?" rather than "How unlikely is this data under H₀?"
- Naturally supports early stopping without the mSPRT overhead.
- Incorporates prior knowledge (e.g., historical conversion rates).

**Disadvantages:**
- Prior choice can materially affect small-sample conclusions.
- "Probability that treatment is better" ≠ "expected magnitude of uplift"—need to examine the posterior mean and credible interval.
- Less interpretable to stakeholders who expect p-values.

> **Why / When to use / Nuances:** Bayesian methods are preferred when experiment velocity is high, sample sizes are small, or you want to roll out incrementally (e.g., stop as soon as 95% probability of improvement). Frequentist methods are preferred when regulatory or audit requirements demand FWER control.

---

## 10. Quasi-Experiments

When randomization is impossible (ethical, logistical, or competitive constraints), quasi-experimental designs attempt to approximate it using natural variation.

### 10.1 Difference-in-Differences (DiD)

**Setup:** Observe a treatment group and a control group before and after a naturally occurring intervention.

```
DiD estimate = (Ȳ_treatment_post − Ȳ_treatment_pre) − (Ȳ_control_post − Ȳ_control_pre)
```

**Key assumption — Parallel trends:** In the absence of treatment, the average outcome would have evolved identically in both groups. Test this by checking pre-intervention trends visually and with a pre-trend F-test. ([World Bank Development Impact Blog](https://blogs.worldbank.org/en/impactevaluations/revisiting-difference-differences-parallel-trends-assumption-part-i-pre-trend))

**Failure mode:** If treatment adoption is triggered by an event correlated with the outcome trend (e.g., struggling companies adopt a new feature), parallel trends fails.

### 10.2 Regression Discontinuity (RD)

**Setup:** Treatment is assigned based on a threshold of a running variable (e.g., users with score > 700 receive a promotion). Users just above and just below the threshold are nearly identical—creating a local randomized comparison.

**Estimate:** LATE (Local Average Treatment Effect) at the cutoff.

**Assumption:** The relationship between the running variable and outcome is continuous at the cutoff in the absence of treatment.

### 10.3 Switchback Experiments

**Setup:** Alternate treatment and control across time windows (e.g., odd hours = treatment, even hours = control) within the same market or system. Used heavily in marketplaces (Uber, Lyft) where geo-cluster randomization is insufficient.

**Advantage:** Eliminates cross-user contamination (everyone in the market is in the same arm at any given time).

**Disadvantage:** Carryover effects—a surge pricing period may deplete driver supply, affecting the next control window. Requires careful choice of window length and wash-out periods. ([arXiv:2606.03012 — Powerful Switchback Experiments](https://arxiv.org/pdf/2606.03012))

> **Why / When to use / Nuances:** Use switchback for marketplace features where both supply and demand are interdependent, making user-level randomization infeasible. Use DiD when you have panel data and a sharp external intervention. Use RD when treatment eligibility is determined by a score or threshold.

---

## 🎯 In the Interview

**Trap 1 — "You can stop the experiment as soon as p < 0.05."**
No. Peeking and stopping inflates the actual false positive rate well above α. Either commit to the pre-planned sample size or use sequential testing (§8).

**Trap 2 — "Our test showed +2% conversion. Ship it."**
Check: (a) Was the pre-planned sample size reached? (b) Did guardrail metrics hold? (c) Is there an SRM? (d) Was this the pre-registered primary metric or one of 20 metrics you checked afterward? (e) Is the confidence interval economically meaningful?

**Trap 3 — "We ran the test, it wasn't significant, so there's no effect."**
Absence of evidence ≠ evidence of absence. Report the CI—if the upper bound of the CI is commercially insignificant, that is informative. If the CI is wide due to insufficient power, the test was inconclusive.

**Trap 4 — "We'll just segment the results to find where it works."**
Post-hoc segmentation without multiple-testing correction inflates false discoveries. Pre-register any segment analyses.

**Trap 5 — "Our product is a social network. We'll randomly assign users."**
User-level randomization violates SUTVA for network products. Consider geo-cluster or ego-network randomization and bias direction (network effects inflate control performance, understating treatment benefit).

**Trap 6 — "Bonferroni is always the right correction."**
Bonferroni is extremely conservative for large metric families. BH FDR control is more powerful when testing dozens of metrics simultaneously and some false discoveries are acceptable.

**Common interview questions:**
- "Walk me through designing an A/B test for feature X end-to-end."
- "How do you calculate sample size? What if your traffic is too low?"
- "What is CUPED? How does it work?"
- "What is an SRM and how do you detect it?"
- "What is the problem with peeking at a test?"
- "When would you use a quasi-experiment instead of an A/B test?"

---

## Glossary

| Term | Definition |
|---|---|
| **MDE** | Minimum Detectable Effect — the smallest true effect the experiment is powered to detect with probability (1 − β). |
| **Power (1 − β)** | Probability of correctly rejecting H₀ when the true effect equals MDE. |
| **α (Type I error)** | Probability of rejecting H₀ when it is actually true (false positive rate). |
| **FWER** | Family-Wise Error Rate — probability that at least one true null hypothesis is rejected across k tests. |
| **FDR** | False Discovery Rate — expected proportion of rejections that are false positives. |
| **CUPED** | Controlled-experiment Using Pre-Experiment Data — variance reduction technique using pre-period covariate. |
| **SRM** | Sample Ratio Mismatch — observed assignment ratio diverges significantly from intended ratio. |
| **SUTVA** | Stable Unit Treatment Value Assumption — one unit's treatment does not affect another unit's outcome. |
| **mSPRT** | Mixture Sequential Probability Ratio Test — likelihood ratio test enabling always-valid inference at any stopping time. |
| **DiD** | Difference-in-Differences — causal estimator using pre/post × treatment/control contrast. |
| **Parallel trends** | DiD assumption that treatment and control groups would have followed the same outcome trend absent the intervention. |
| **Novelty effect** | Temporary increase in engagement caused by the newness of a feature, not its long-term value. |
| **Primacy effect** | Temporary decrease in performance for familiar users adapting to a UI change. |
| **Twyman's Law** | The more unusual a result, the more likely it reflects measurement error rather than truth. |
| **Cluster randomization** | Assigning treatment at the group level (geo, community) to prevent interference. |

---

## References

1. Deng, A., Xu, Y., Kohavi, R., & Walker, T. (2013). "Improving the Sensitivity of Online Controlled Experiments by Utilizing Pre-Experiment Data." *KDD 2013*. [Semantic Scholar](https://www.semanticscholar.org/paper/Improving-the-sensitivity-of-online-controlled-by-Deng-Xu/c65d71c42dedb3329c2b81499950bf12786a3e8e) | [PDF](https://robotics.stanford.edu/~ronnyk/2013-02CUPEDImprovingSensitivityOfControlledExperiments.pdf)
2. Johari, R., Pekelis, L., & Walsh, D. (2022). "Always Valid Inference: Bringing Sequential Analysis to A/B Testing." *Operations Research*, 70(3). [arXiv:1512.04922](https://arxiv.org/abs/1512.04922)
3. Benjamini, Y., & Hochberg, Y. (1995). "Controlling the False Discovery Rate: A Practical and Powerful Approach to Multiple Testing." *Journal of the Royal Statistical Society, Series B*, 57(1), 289–300.
4. Kohavi, R., Tang, D., & Xu, Y. (2020). *Trustworthy Online Controlled Experiments: A Practical Guide to A/B Testing*. Cambridge University Press.
5. Statsig. (2024). "Power Analysis for A/B Testing: How to Size Experiments Correctly." [statsig.com](https://www.statsig.com/perspectives/power-analysis-ab-testing)
6. World Bank Development Impact Blog. "Revisiting the Difference-in-Differences Parallel Trends Assumption." [worldbank.org](https://blogs.worldbank.org/en/impactevaluations/revisiting-difference-differences-parallel-trends-assumption-part-i-pre-trend)
7. Matteo Courthoud. "Understanding CUPED." [matteocourthoud.github.io](https://matteocourthoud.github.io/post/cuped/)

---

*Part of the [Statistics for Data Science](README.md) section · [Guide Home](../README.md).*
