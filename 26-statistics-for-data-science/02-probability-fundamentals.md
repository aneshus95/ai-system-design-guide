# 02 — Probability Fundamentals

Probability is the mathematical language of uncertainty. Data scientists use it to reason about data-generating processes, build generative models, evaluate classifiers, and avoid the most dangerous cognitive traps in quantitative work.

> **In plain English:** Probability answers the question "how likely is this?" It forces you to be precise about what you mean by "likely" and what information you are conditioning on. Almost every ML algorithm either maximises a probability, samples from one, or makes assumptions about one — knowing this foundation stops you from making classic blunders like confusing P(cancer | positive test) with P(positive test | cancer).

---

## Table of Contents

1. [Sample Spaces and Events](#1-sample-spaces-and-events)
2. [Axioms of Probability](#2-axioms-of-probability)
3. [Addition and Multiplication Rules](#3-addition-and-multiplication-rules)
4. [Conditional Probability](#4-conditional-probability)
5. [Independence vs Mutual Exclusivity](#5-independence-vs-mutual-exclusivity)
6. [Law of Total Probability](#6-law-of-total-probability)
7. [Bayes' Theorem and the Base-Rate Fallacy](#7-bayes-theorem-and-the-base-rate-fallacy)
8. [Combinatorics — Permutations and Combinations](#8-combinatorics--permutations-and-combinations)
9. [Random Variables](#9-random-variables)
10. [Joint, Marginal, and Conditional Distributions](#10-joint-marginal-and-conditional-distributions)
11. [Expectation and Variance Rules](#11-expectation-and-variance-rules)
12. [Odds vs Probability](#12-odds-vs-probability)
13. [Common Probability Fallacies](#13-common-probability-fallacies)
14. [Glossary](#glossary)
15. [References](#references)

---

## 1. Sample Spaces and Events

| Term | Definition | Example (fair die) |
|---|---|---|
| **Experiment** | A procedure with uncertain outcomes | Roll a six-sided die |
| **Sample space Ω** | The set of ALL possible outcomes | {1, 2, 3, 4, 5, 6} |
| **Event A** | A subset of the sample space | A = {2, 4, 6} (even numbers) |
| **Elementary event** | A single outcome | {3} |
| **Complement Aᶜ** | All outcomes NOT in A | {1, 3, 5} |
| **Union A ∪ B** | Outcomes in A or B or both | A ∪ {1,2,3} = {1,2,3,4,6} |
| **Intersection A ∩ B** | Outcomes in both A and B | {2,4,6} ∩ {1,2,3} = {2} |

### 🎯 In the interview

- "Define event vs outcome." → An outcome is one element of Ω; an event is a **set** of outcomes (it may contain zero, one, or many).

---

## 2. Axioms of Probability

The **Kolmogorov axioms** (1933) are the foundation of all of probability theory:

```
1. P(A) ≥ 0            for any event A
2. P(Ω) = 1            the sample space has probability 1
3. Countable additivity: if A₁, A₂, … are mutually exclusive,
   P(A₁ ∪ A₂ ∪ …) = P(A₁) + P(A₂) + …
```

Everything else in probability is derived from these three axioms.

**Key consequences:**
- `P(∅) = 0`
- `P(Aᶜ) = 1 − P(A)`
- `0 ≤ P(A) ≤ 1`

---

## 3. Addition and Multiplication Rules

### 3.1 Addition Rule (Union)

For any two events:

```
P(A ∪ B) = P(A) + P(B) − P(A ∩ B)
```

The subtraction removes the double-counted intersection.

For mutually exclusive events (A ∩ B = ∅):

```
P(A ∪ B) = P(A) + P(B)
```

**Mini-example:**  
Drawing one card: P(King) = 4/52, P(Heart) = 13/52, P(King of Hearts) = 1/52  
P(King or Heart) = 4/52 + 13/52 − 1/52 = 16/52 ≈ 0.308

### 3.2 Multiplication Rule (Intersection)

In general:

```
P(A ∩ B) = P(A) × P(B | A)
```

If A and B are **independent**:

```
P(A ∩ B) = P(A) × P(B)
```

**Mini-example:**  
Flipping two fair coins: P(H on flip 1 AND H on flip 2) = 0.5 × 0.5 = 0.25 (independent events).

### 🎯 In the interview

- "P(A or B) — can you just add?" → Only if mutually exclusive. Always check for overlap.

---

## 4. Conditional Probability

```
P(A | B) = P(A ∩ B) / P(B),   provided P(B) > 0
```

**In plain English:** Given that B has occurred, what is the probability of A? We restrict the sample space to B and renormalise.

**Mini-example:**  
Deck of cards. P(King | Heart) = P(King ∩ Heart) / P(Heart) = (1/52) / (13/52) = 1/13. There are 13 hearts; of those, 1 is the King.

**Key asymmetry:** `P(A | B) ≠ P(B | A)` in general. This confusion is one of the most common errors in applied statistics (the "transpose" or "confusion of the inverse"), and lies at the heart of both the base-rate fallacy and the prosecutor's fallacy.

### 🎯 In the interview

- "Is P(disease | positive test) the same as P(positive test | disease)?" → **No.** P(positive | disease) is sensitivity (a test property). P(disease | positive) is positive predictive value (depends on prevalence). Confusing them is the base-rate fallacy.

---

## 5. Independence vs Mutual Exclusivity

These two concepts are often confused. They are almost opposites for events with positive probability.

| Property | Definition | Formula |
|---|---|---|
| **Independent** | Knowing B gives no information about A | `P(A ∩ B) = P(A) × P(B)` |
| **Mutually exclusive** | A and B cannot both occur | `P(A ∩ B) = 0` |

**Critical nuance:** If A and B are mutually exclusive and both have P > 0, they **cannot be independent** — knowing B happened tells you A definitely didn't, so B gives information about A.

> **Why / When to use / Nuances:**
> - Independent events: coin flips, draws with replacement — occurrence of one does not affect probability of the other.
> - Mutually exclusive events: rolling a 1 OR a 2 on a die — they cannot both occur simultaneously.
> - Two events can be neither independent nor mutually exclusive (overlapping, correlated events).

**Mini-example:**  
Die roll. A = {1,2}, B = {2,3}.  
P(A) = 1/3, P(B) = 1/3, P(A∩B) = P({2}) = 1/6.  
P(A)×P(B) = 1/9 ≠ 1/6 → **not independent**.  
P(A∩B) = 1/6 ≠ 0 → **not mutually exclusive**.

### 🎯 In the interview

- "Can two events be both independent and mutually exclusive?" → Only if at least one of them has probability zero. For positive-probability events, mutual exclusivity implies dependence.

---

## 6. Law of Total Probability

If {B₁, B₂, …, Bₙ} is a **partition** of the sample space (mutually exclusive, collectively exhaustive):

```
P(A) = Σᵢ P(A | Bᵢ) × P(Bᵢ)
```

You compute the probability of A by summing its conditional probability under each scenario, weighted by how likely each scenario is.

**Mini-example:**  
Factory: 70% of widgets come from Machine 1 (defect rate 2%), 30% from Machine 2 (defect rate 5%).  
P(defective) = P(defect | M1)×P(M1) + P(defect | M2)×P(M2)  
             = 0.02×0.7 + 0.05×0.3  
             = 0.014 + 0.015 = 0.029 (2.9%)

This law is the denominator in Bayes' theorem.

---

## 7. Bayes' Theorem and the Base-Rate Fallacy

### 7.1 Bayes' Theorem

```
P(H | E) = P(E | H) × P(H) / P(E)
```

Or equivalently (expanding the denominator via total probability):

```
P(H | E) = P(E | H) × P(H) / [P(E | H)×P(H) + P(E | Hᶜ)×P(Hᶜ)]
```

| Term | Name | Meaning |
|---|---|---|
| `P(H)` | Prior | Probability of hypothesis before seeing evidence |
| `P(E \| H)` | Likelihood | How probable is this evidence if H is true |
| `P(H \| E)` | Posterior | Updated probability after seeing evidence |
| `P(E)` | Marginal likelihood / evidence | Normalising constant |

### 7.2 Worked Disease-Testing Example (Base-Rate Fallacy)

**Setup:**
- A disease affects 1% of the population: `P(Disease) = 0.01`
- A test has **sensitivity** (true positive rate) of 99%: `P(Positive | Disease) = 0.99`
- A test has **specificity** (true negative rate) of 95%: `P(Negative | No Disease) = 0.95`, so `P(Positive | No Disease) = 0.05`

**Question:** A patient tests positive. What is the probability they actually have the disease?

**Most people's intuition:** "The test is 99% accurate, so about 99%." This is **wrong**.

**Applying Bayes' theorem:**

```
P(Disease | Positive)
  = P(Positive | Disease) × P(Disease)
    / [P(Positive | Disease)×P(Disease) + P(Positive | No Disease)×P(No Disease)]

  = (0.99 × 0.01) / [(0.99 × 0.01) + (0.05 × 0.99)]

  = 0.0099 / (0.0099 + 0.0495)

  = 0.0099 / 0.0594

  ≈ 0.167 (16.7%)
```

**The answer is only ~16.7%.** Even with a highly sensitive and specific test, a positive result for a **rare disease** is more likely a false positive than a true positive.

**Intuition via frequency format (Gigerenzer, 2002):**  
Imagine 10,000 people.  
- 100 have the disease → 99 test positive (true positives).  
- 9,900 don't → 495 test positive (false positives).  
- Total positives = 594. True positives = 99.  
- P(Disease | Positive) = 99/594 ≈ 16.7% ✓

**The base-rate fallacy:** Ignoring the prior probability P(H) and focusing only on P(E|H). When the base rate is low, even a good test has a high false discovery rate.

**Real-world implications:**
- Mass cancer screening for rare cancers — many false positives.
- Fraud detection — flagging rate depends critically on the base rate of fraud.
- DNA evidence in court — a 1-in-a-million match probability is meaningless without knowing how many people were in the reference database.

### 🎯 In the interview

- "A classifier has 95% accuracy but the dataset is 99% negative. Is it good?" → No! Predicting all-negative achieves 99% accuracy. Use precision, recall, F1 or AUC. This is the class-imbalance / base-rate problem.
- "P(A|B) vs P(B|A) — when does this distinction matter?" → Always. In medical testing, legal evidence, and any classifier evaluation.

---

## 8. Combinatorics — Permutations and Combinations

### 8.1 Permutations (Order Matters)

Number of ways to arrange k items from n distinct items:

```
P(n, k) = n! / (n−k)!
```

**Example:** How many ways to award gold/silver/bronze among 10 athletes?  
P(10, 3) = 10×9×8 = 720

Number of arrangements of all n items: `n!`

### 8.2 Combinations (Order Does Not Matter)

Number of ways to choose k items from n (without replacement, without regard to order):

```
C(n, k) = "n choose k" = n! / (k! × (n−k)!)
```

**Example:** How many 5-card hands from a 52-card deck?  
C(52, 5) = 52! / (5! × 47!) = 2,598,960

**Key identity:** `C(n, k) = C(n, n−k)` — choosing k items to include is the same as choosing n−k items to exclude.

### 8.3 With Replacement vs Without Replacement

| | Order matters | Order doesn't matter |
|---|---|---|
| **Without replacement** | P(n,k) = n!/(n−k)! | C(n,k) = n!/(k!(n−k)!) |
| **With replacement** | nᵏ | C(n+k−1, k) |

**Mini-example:**  
Probability of a specific 5-card hand (e.g., royal flush) = 1 / C(52,5) = 1 / 2,598,960 ≈ 3.8 × 10⁻⁷

### 🎯 In the interview

- "How many ways to split a team of 10 into two equal groups?" → C(10,5)/2 = 126 (divide by 2 because the two groups are interchangeable).
- "What is the birthday problem?" → In a group of 23 people, P(at least two share a birthday) > 50%. Uses complement: P(all different birthdays) = 365/365 × 364/365 × … × 343/365 ≈ 0.493, so P(at least one match) ≈ 0.507.

---

## 9. Random Variables

### 9.1 Definition

A **random variable** X is a function from the sample space Ω to the real numbers ℝ. It assigns a numerical value to each outcome.

### 9.2 Discrete Random Variables

Takes on a countable set of values (finite or countably infinite).

**Probability Mass Function (PMF):**

```
p(x) = P(X = x),   where Σₓ p(x) = 1 and p(x) ≥ 0
```

**Cumulative Distribution Function (CDF):**

```
F(x) = P(X ≤ x) = Σ_{t ≤ x} p(t)
```

**Examples:** Number of heads in 10 flips (Binomial), number of arrivals in 1 hour (Poisson), result of a die roll (Discrete Uniform).

### 9.3 Continuous Random Variables

Takes values in an uncountable set (a continuum). P(X = x) = 0 for any single point.

**Probability Density Function (PDF):**

```
f(x) ≥ 0,   ∫_{-∞}^{∞} f(x) dx = 1
P(a ≤ X ≤ b) = ∫_a^b f(x) dx
```

**Cumulative Distribution Function (CDF):**

```
F(x) = P(X ≤ x) = ∫_{-∞}^{x} f(t) dt
```

The PDF is the derivative of the CDF: `f(x) = dF(x)/dx`.

**Examples:** Height (Normal), time until event (Exponential), proportion (Beta), any variable on an interval (Uniform).

### 9.4 Key Properties of the CDF (Applies to Both Types)

- F(−∞) = 0, F(+∞) = 1
- F is non-decreasing
- F is right-continuous

**Mini-example (Discrete):**  
X = number of heads in 2 fair coin flips.  
P(X=0) = 0.25, P(X=1) = 0.50, P(X=2) = 0.25.  
F(0) = 0.25, F(1) = 0.75, F(2) = 1.00.

> **Why / When to use / Nuances:**
> - PMF: probability at a point (discrete). PDF: probability density at a point — NOT probability (continuous). You must integrate a PDF to get probability.
> - CDF is the most fundamental — it works for both discrete and continuous RVs, and is what the ECDF estimates from data.

### 🎯 In the interview

- "Can a PDF be greater than 1?" → **Yes.** PDF is a density, not a probability. It just needs to integrate to 1. E.g., Uniform(0, 0.5) has pdf = 2 everywhere on [0, 0.5].
- "What is the CDF useful for?" → Computing percentiles, P(a < X < b), goodness-of-fit tests, generating random variates (inverse CDF method).

---

## 10. Joint, Marginal, and Conditional Distributions

### 10.1 Joint Distribution

For two RVs X and Y:

```
Discrete:    p(x, y) = P(X = x, Y = y)
Continuous:  f(x, y)  such that ∬ f(x,y) dx dy = 1
```

### 10.2 Marginal Distribution

Obtained by "summing out" or "integrating out" the other variable:

```
Discrete:    p_X(x) = Σ_y p(x, y)
Continuous:  f_X(x) = ∫_{-∞}^{∞} f(x, y) dy
```

### 10.3 Conditional Distribution

```
Discrete:    p(x | y) = p(x, y) / p_Y(y)
Continuous:  f(x | y) = f(x, y) / f_Y(y)
```

Note this is exactly Bayes' theorem in distribution form:

```
f(x | y) = f(y | x) × f_X(x) / f_Y(y)
```

### 10.4 Independence of RVs

X and Y are independent if and only if:

```
f(x, y) = f_X(x) × f_Y(y)    for all x, y
```

Equivalently: conditional distribution equals marginal: `f(x|y) = f_X(x)`.

**Mini-example:**  
Roll two dice. X = result of die 1, Y = result of die 2.  
Joint: p(x,y) = 1/36 for all (x,y) ∈ {1..6}².  
Marginal: p_X(x) = 1/6. Marginal: p_Y(y) = 1/6.  
p(x,y) = 1/36 = (1/6)(1/6) = p_X(x)×p_Y(y) → independent. ✓

---

## 11. Expectation and Variance Rules

### 11.1 Expectation (Expected Value)

```
Discrete:    E[X] = Σ_x x × p(x)
Continuous:  E[X] = ∫_{-∞}^{∞} x × f(x) dx
```

**Law of the Unconscious Statistician (LOTUS):**

```
E[g(X)] = Σ_x g(x) p(x)    (discrete)
E[g(X)] = ∫ g(x) f(x) dx   (continuous)
```

### 11.2 Linearity of Expectation

```
E[aX + b] = a × E[X] + b
E[X + Y]  = E[X] + E[Y]    (always, even if X and Y are dependent)
```

Linearity of expectation holds regardless of the dependence structure of X and Y. This is powerful.

**Mini-example:**  
Expected number of heads in 100 fair coin flips. Let Xᵢ = 1 if flip i is heads.  
E[X₁ + X₂ + … + X₁₀₀] = E[X₁] + … + E[X₁₀₀] = 100 × 0.5 = 50. (No need to compute the Binomial directly.)

### 11.3 Variance Rules

```
Var(X) = E[(X − μ)²] = E[X²] − (E[X])²
Var(aX + b) = a² × Var(X)    (constants shift don't change variance; scaling by a scales variance by a²)
```

For independent X and Y:

```
Var(X + Y) = Var(X) + Var(Y)         (independent)
Var(X − Y) = Var(X) + Var(Y)         (independent — variances ADD even for difference)
```

For dependent X and Y:

```
Var(X + Y) = Var(X) + Var(Y) + 2×Cov(X,Y)
Var(X − Y) = Var(X) + Var(Y) − 2×Cov(X,Y)
```

### 11.4 Sums of Independent Random Variables

If X₁, X₂, …, Xₙ are iid with mean μ and variance σ²:

```
E[X̄] = μ
Var(X̄) = σ²/n
SD(X̄) = σ/√n    (standard error of the mean)
```

This is foundational for the Central Limit Theorem and all sampling theory.

**Mini-example (Var(aX+b)):**  
If X ~ N(μ, σ²), then Y = 2X + 3 ~ N(2μ+3, 4σ²).  
The constant 3 shifts the mean but doesn't affect variance.  
The factor 2 multiplies the SD by 2 and the variance by 4.

> **Why / When to use / Nuances:**
> - Linearity of expectation is unconditional — no independence required. Use it whenever you can decompose a complex RV into a sum of simpler ones.
> - Var(X+Y) = Var(X)+Var(Y) requires independence. Check before applying.
> - Var(X−Y) is NOT Var(X)−Var(Y). Variance is always non-negative and differences of variances can go negative.

### 🎯 In the interview

- "If you average n iid observations, how does the variance change?" → Var(X̄) = σ²/n — variance shrinks by factor n. This is why bigger samples give better estimates.
- "Var(X − Y) when X and Y are independent?" → Var(X) + Var(Y). Subtracting two uncertain quantities makes the result more uncertain, not less.

---

## 12. Odds vs Probability

### 12.1 Definitions

```
If P(A) = p, then:
Odds in favour of A = p / (1 − p)
Odds against A      = (1 − p) / p
```

Conversion:

```
p = odds / (1 + odds)
odds = p / (1 − p)
```

### 12.2 Log-Odds (Logit)

```
logit(p) = log(p / (1 − p)) = log(odds)
```

- Range: (−∞, +∞)
- This is what logistic regression models linearly. The output is converted to probability via the **sigmoid** (logistic) function.

```
sigmoid(z) = 1 / (1 + e^{−z})
```

**Mini-example:**  
P(win) = 0.8.  
Odds = 0.8/0.2 = 4:1 ("four to one on").  
Log-odds = log(4) ≈ 1.386.

Conversely: if a bookie gives odds of 3:1, implied probability = 3/(3+1) = 75%.

### 🎯 In the interview

- "What does a logistic regression coefficient of 0.5 mean?" → The log-odds of the outcome increases by 0.5 for a one-unit increase in the predictor. In odds ratio terms: exp(0.5) ≈ 1.65 — the odds multiply by 1.65.
- "What is an odds ratio > 1 vs < 1?" → OR > 1: the factor increases the odds of the event; OR < 1: it decreases the odds.

---

## 13. Common Probability Fallacies

Understanding these is just as important as knowing the formulas — they appear in interviews as "what is wrong with this reasoning?"

### 13.1 Gambler's Fallacy

**Belief:** After a sequence of the same outcome (e.g., 5 heads in a row), the opposite outcome is "due."

**Why it is wrong:** Each flip is **independent**. The coin has no memory. P(H on flip 6) = 0.5 regardless of history. The law of large numbers says the *fraction* converges to 0.5 over infinite flips — it does not imply *corrective* runs.

**In data science:** Assuming that a model that has been wrong several times in a row is "due" to be right, or that a random seed that has given poor splits will eventually give a good one.

### 13.2 Base-Rate Fallacy (Neglect of Prior)

*Fully worked above in Section 7.* The short version: people overweight the likelihood `P(E|H)` and underweight the prior `P(H)`. For rare events, the prior dominates.

**In data science:** Confusing accuracy with utility on imbalanced datasets; ignoring prevalence when interpreting a classifier's positive predictive value.

### 13.3 Conjunction Fallacy

**Belief:** A conjunction (A and B) is more probable than one of its components alone.

```
P(A ∩ B) ≤ min(P(A), P(B))    always
```

**Classic example (Tversky & Kahneman, 1983):** "Linda is 31, intelligent, outspoken, and majored in philosophy. Is it more likely that Linda is a bank teller, or that Linda is a bank teller and feminist activist?" Most people say the latter — but `P(teller ∩ feminist) ≤ P(teller)` always.

**In data science:** Assuming that a model with two specific properties (e.g., accurate AND fast) is more likely to exist than a model that is merely accurate.

### 13.4 Prosecutor's Fallacy (Transposition of the Conditional)

**Belief:** If P(evidence | innocent) is tiny, then P(innocent | evidence) must also be tiny.

This is confusing P(E|H) with P(H|E) — exactly the conditional probability inversion error.

**Classic example:** DNA matches at 1 in a million frequency. Prosecutor argues "there is only a 1 in a million chance the defendant is innocent." But if 10 million people live in the city, you'd expect ~10 DNA matches. P(innocent | match) ≈ 10/11 ≈ 90.9%.

**Requires Bayes' theorem** to reason correctly: multiply the likelihood by the prior.

**In data science:** "The p-value is 0.001, so there is a 99.9% probability the alternative hypothesis is true." Wrong. The p-value is `P(data | H₀)`, not `P(H₀ | data)`.

### 13.5 Survivorship Bias

Only observing the units that "survived" a selection process.

**Classic example (Abraham Wald):** WWII analysts recommended armour where returning planes were hit. Wald pointed out they should armour where returning planes were *not* hit — planes hit there didn't return.

**In data science:** Training on available (non-missing) data when missingness is related to the target; evaluating hedge fund performance using only currently existing funds; studying successful startups to learn the keys to success.

### 13.6 Simpson's Paradox

A trend that appears in several groups of data can disappear or reverse when the groups are combined.

**Classic example (UC Berkeley admissions, 1973):** Overall admission rates favoured men. But within every single department, women were admitted at higher rates. Women were applying to more competitive departments, causing the aggregate to reverse.

**In data science:** Aggregate metrics can be misleading when the data has confounders. Always segment and check.

### 🎯 In the interview

- "What is wrong with: 'Our test has p=0.001 so we are 99.9% confident our hypothesis is true'?" → P-value is P(data | H₀), not P(H₀ | data). You need Bayes' theorem, a prior, and the base rate of true hypotheses to get the latter.
- "How do you check for Simpson's paradox?" → Compute the metric disaggregated by potential confounders and compare with the aggregate.

---

## Glossary

| Term | Simple explanation | Purpose |
|---|---|---|
| **Sample space Ω** | Set of all possible outcomes | Defines the universe of the experiment |
| **Event** | A subset of the sample space | The outcome we compute probability for |
| **Axioms (Kolmogorov)** | Non-negativity, normalisation, countable additivity | Foundation of all probability |
| **Conditional probability P(A\|B)** | P(A∩B)/P(B) | Probability of A given B has occurred |
| **Independence** | P(A∩B) = P(A)P(B) | Events carry no information about each other |
| **Mutual exclusivity** | P(A∩B) = 0 | Events cannot both happen simultaneously |
| **Law of Total Probability** | P(A) = Σ P(A\|Bᵢ)P(Bᵢ) | Marginalise over a partition |
| **Bayes' theorem** | P(H\|E) = P(E\|H)P(H)/P(E) | Update beliefs given new evidence |
| **Prior P(H)** | Belief about H before evidence | Starting point for Bayesian inference |
| **Likelihood P(E\|H)** | How probable is this evidence if H is true | Evidence model |
| **Posterior P(H\|E)** | Updated belief after seeing evidence | The answer we want |
| **PMF** | p(x) = P(X=x), discrete RV | Assigns probability to each discrete value |
| **PDF** | f(x) such that ∫f=1, continuous RV | Density; integrate to get probability |
| **CDF** | F(x) = P(X≤x) | Cumulative probability; unified for all RVs |
| **Expectation E[X]** | Probability-weighted average of values | Population mean of a RV |
| **Variance Var(X)** | E[(X−μ)²] | Average squared deviation from mean |
| **Linearity of expectation** | E[X+Y]=E[X]+E[Y] always | Powerful: holds without independence |
| **Permutation P(n,k)** | n!/(n−k)! | Ordered selections |
| **Combination C(n,k)** | n!/(k!(n−k)!) | Unordered selections |
| **Odds** | p/(1−p) | Alternative to probability; used in logistic regression |
| **Log-odds (logit)** | log(p/(1−p)) | Linear output of logistic regression |
| **Gambler's fallacy** | Belief that past results affect independent future ones | Common error with independent events |
| **Base-rate fallacy** | Ignoring prior probability when updating on evidence | Leads to over-estimation of rare-event posterior |
| **Conjunction fallacy** | Believing P(A∩B) > P(A) | Violates basic axiom P(A∩B) ≤ P(A) |
| **Prosecutor's fallacy** | Confusing P(E\|H) with P(H\|E) | Requires Bayes' theorem to correct |
| **Survivorship bias** | Only observing successful/surviving units | Selection bias; results not generalisable |
| **Simpson's paradox** | Aggregate trend reverses in subgroups | Confounders can invert conclusions |

---

## References

1. Kolmogorov, A. N. (1933). *Grundbegriffe der Wahrscheinlichkeitsrechnung*. Springer. (Foundations of probability; axioms from §I–II.)
2. NIST/SEMATECH e-Handbook of Statistical Methods, §5.3 — *Probability Distributions*. https://www.itl.nist.gov/div898/handbook/
3. Tversky, A. & Kahneman, D. (1983). "Extensional versus intuitive reasoning: The conjunction fallacy in probability judgment." *Psychological Review, 90*(4), 293–315. https://doi.org/10.1037/0033-295X.90.4.293
4. Gigerenzen, G. (2002). *Calculated Risks: How to Know When Numbers Deceive You*. Simon & Schuster. — Natural frequencies approach to Bayes; base-rate fallacy.
5. Kahneman, D. (2011). *Thinking, Fast and Slow*. Farrar, Straus and Giroux. — Chapters 14–16 on base rates and probability fallacies.
6. Bickel, P. J., Hammel, E. A., & O'Connell, J. W. (1975). "Sex Bias in Graduate Admissions: Data from Berkeley." *Science, 187*(4175), 398–404. — Original Simpson's Paradox example. https://doi.org/10.1126/science.187.4175.398
7. Thompson, W. C. & Schumann, E. L. (1987). "Interpretation of Statistical Evidence in Criminal Trials." *Law and Human Behavior, 11*(3), 167–187. — Prosecutor's fallacy. https://doi.org/10.1007/BF01044641
8. DeGroot, M. H. & Schervish, M. J. (2012). *Probability and Statistics* (4th ed.). Pearson. — Rigorous treatment of all topics in this file.
9. Walpole, R. E., Myers, R. H., Myers, S. L., & Ye, K. (2016). *Probability and Statistics for Engineers and Scientists* (9th ed.). Pearson. — Combinatorics (§2.3), conditional probability (§2.6), Bayes (§2.8).

---

*Part of the [Statistics for Data Science](README.md) section · [Guide Home](../README.md).*
