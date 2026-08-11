# 11 — Causal Inference and Biases

Correlation is cheap; causation is expensive. Machine learning excels at finding patterns and correlations in data, but making causal claims — "if we change X, Y will change" — requires far more care. Causal inference is the set of tools and frameworks that let us move from observational data to actionable, interventional conclusions. It is fundamental to A/B test design, policy evaluation, fairness in ML, and anywhere you need to answer "what would have happened if...?"

> **In plain English:** Just because ice cream sales and drowning deaths both peak in summer does not mean eating ice cream causes drowning. Both are driven by a third variable: hot weather. Causal inference gives you the mathematical machinery to untangle these relationships and — when you cannot run an experiment — extract causal signals from messy observational data.

## Table of Contents

1. [Correlation ≠ Causation](#1-correlation--causation)
2. [Confounding and the Third-Variable Problem](#2-confounding-and-the-third-variable-problem)
3. [Simpson's Paradox — Worked Example](#3-simpsons-paradox--worked-example)
4. [Selection, Survivorship, and Collider Bias](#4-selection-survivorship-and-collider-bias)
5. [Potential Outcomes and Counterfactuals](#5-potential-outcomes-and-counterfactuals)
6. [RCTs — The Gold Standard](#6-rcts--the-gold-standard)
7. [Observational Causal Methods](#7-observational-causal-methods)
8. [DAGs and the Backdoor Criterion](#8-dags-and-the-backdoor-criterion)
9. [Berkson's Paradox](#9-berksons-paradox)
10. [Common Data Biases in ML](#10-common-data-biases-in-ml)
11. [🎯 In the Interview](#11--in-the-interview)
12. [Glossary](#glossary)
13. [References](#references)

---

## 1. Correlation ≠ Causation

A statistical association between X and Y can arise from three distinct structural relationships:

| Structure | Description | Example |
|---|---|---|
| **Direct causation** | X → Y | Smoking → lung cancer |
| **Common cause (confounding)** | Z → X and Z → Y | Hot weather → ice cream sales AND hot weather → drowning |
| **Reverse causation** | Y → X | Sick people in hospitals more than healthy people → "hospitals cause illness" |
| **Coincidence** | Random co-occurrence | Nicolas Cage movies vs pool drownings |

Observational data cannot distinguish these without additional assumptions. Causal inference provides frameworks — experiments, DAGs, design assumptions — to make causal claims defensible.

**The fundamental problem of causal inference:** To know whether X caused Y in a specific instance, you would need to observe the same unit with and without X simultaneously. You cannot. You can only ever observe one of the potential outcomes. The entire field is built on how to handle this "missing counterfactual."

---

## 2. Confounding and the Third-Variable Problem

A **confounder** Z is a variable that causally influences both the treatment X and the outcome Y, creating a spurious association between X and Y that is not due to a causal effect of X on Y.

```
Confounder structure:
    Z
   / \
  X   Y
  ↕ (spurious)
```

**Example:** An observational study finds that people who carry lighters are more likely to develop lung cancer. The confounder is smoking: smokers carry lighters AND develop lung cancer. Lighters do not cause cancer.

### Why Confounding is Dangerous in ML

- A model trained on observational data will learn the confounded association and make predictions that assume the spurious relationship.
- If you act on these predictions (e.g. recommend lighters cessation as a lung cancer intervention), the policy will fail.
- Feature importance in predictive models cannot distinguish causal features from proxies of confounders.

> **The key question in causal inference:** Is the relationship between X and Y causal, or is there an open backdoor path through a confounder? This requires domain knowledge or explicit causal assumptions — not just data.

---

## 3. Simpson's Paradox — Worked Example

**Simpson's paradox** occurs when a relationship between two variables reverses (or disappears) when data are aggregated across subgroups, due to confounding.

Source: [Understanding Simpson's Paradox Using a Graph — Columbia Statistical Modeling Blog](https://statmodeling.stat.columbia.edu/2014/04/08/understanding-simpsons-paradox-using-graph/)

### Classic Example: University Admissions

Suppose a university is accused of gender bias in admissions.

**Aggregate data:**

| Gender | Applied | Admitted | Rate |
|---|---|---|---|
| Men | 800 | 480 | **60%** |
| Women | 800 | 320 | **40%** |

"Men are admitted at a 60% rate vs 40% for women — clear bias!"

But now look by department:

**Department A** (competitive; small):

| Gender | Applied | Admitted | Rate |
|---|---|---|---|
| Men | 50 | 35 | 70% |
| Women | 400 | 280 | 70% |

**Department B** (less competitive; large):

| Gender | Applied | Admitted | Rate |
|---|---|---|---|
| Men | 750 | 445 | 59% |
| Women | 400 | 40 | 10% |

**Wait — within both departments, men and women have similar or identical rates!**

The paradox: women disproportionately applied to the highly competitive Department B (10% admit rate), dragging down their overall rate. The confounder is **department choice** — it affects both gender composition (treatment proxy) and admission rate (outcome).

### When Should You Stratify?

- If the "third variable" (department) is a **confounder**: stratify — look at within-group rates.
- If the "third variable" is a **mediator** (part of the causal pathway): do NOT stratify — you block the causal effect.
- The correct choice depends on the causal DAG, not just the data.

Source: [Simpson's Paradox — NCBI / NIH](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC7175433/)

---

## 4. Selection, Survivorship, and Collider Bias

### Selection Bias

Occurs when the sample is not representative of the target population because the probability of being included in the sample depends on characteristics related to the outcome.

**Examples:**
- Polling only landline phones in 1936 (Literary Digest predicted Landon over FDR).
- A clinical trial that excludes the sickest patients (underestimates efficacy for the target population).
- Predicting churn based only on users who are still active (excludes already-churned users).

### Survivorship Bias

A specific form of selection bias: you only observe the **survivors** — the entities that made it through some filter — and incorrectly conclude that the filter had no effect or that survival conditions are typical.

**Examples:**
- WWII aircraft analysis: engineers reinforced the damaged areas of returning planes; Abraham Wald showed they should reinforce the undamaged areas — the planes shot in those spots did not return.
- "Successful startups all pivoted early" — ignores the many that pivoted and still failed.
- "This investment strategy has a great track record" — the strategy with a bad track record was shut down and removed from the dataset.

### Collider Bias

A **collider** is a variable C that is caused by two other variables: X → C ← Y. When you **condition on a collider** (control for it, filter on it, stratify by it), you **induce** a spurious association between X and Y even if none exists.

```
Collider structure:
X → C ← Y
```

**Example — Berkson's paradox preview:**
- Among hospitalised patients (conditioning on C = hospitalised), COVID-19 status and smoking may appear correlated — not because they are causally related in the population, but because both independently increase hospitalisation probability. Conditioning on "hospitalised" opens a backdoor path X ↔ Y through the collider.

**Example — talent vs likability:**
- Suppose among successful actors (C = got the part), acting talent (X) and physical appearance (Y) are negatively correlated, even if they are independent in the population. Why? Because either quality can independently cause success — conditioning on "got the part" creates a negative correlation.

> **Key rule: Do NOT condition on a collider.** Conditioning on a collider creates bias where none previously existed. This is the opposite of confounding (where you want to condition on confounders to remove bias).

---

## 5. Potential Outcomes and Counterfactuals

The **Rubin Causal Model (potential outcomes framework)** formalises causal inference using counterfactuals.

### Notation

For each unit i and treatment value t ∈ {0, 1}:
- **Yᵢ(1):** potential outcome for unit i if treated.
- **Yᵢ(0):** potential outcome for unit i if untreated.
- **Individual Treatment Effect (ITE):** τᵢ = Yᵢ(1) − Yᵢ(0).

**The fundamental problem:** We only observe either Yᵢ(1) or Yᵢ(0), never both. The unobserved value is the **counterfactual**.

### Key Estimands

| Quantity | Definition | Use Case |
|---|---|---|
| **ATE** (Average Treatment Effect) | E[Y(1) − Y(0)] | Effect on the whole population |
| **ATT** (ATE on the Treated) | E[Y(1) − Y(0) \| T=1] | Effect on those who actually received treatment |
| **ATC** (ATE on the Control) | E[Y(1) − Y(0) \| T=0] | Effect if we were to treat the untreated |
| **CATE** (Conditional ATE) | E[Y(1) − Y(0) \| X=x] | Heterogeneous treatment effects by subgroup |

### Key Assumptions for Causal Identification

| Assumption | Meaning |
|---|---|
| **SUTVA** (Stable Unit Treatment Value Assumption) | No interference between units; treatment has one version |
| **Ignorability / Unconfoundedness** | Yᵢ(0), Yᵢ(1) ⊥ Tᵢ \| Xᵢ — treatment is independent of potential outcomes given observed covariates |
| **Overlap / Positivity** | 0 < P(T=1 \| X=x) < 1 for all x — every unit could have received either treatment |

Source: [Causal Inference — An Introduction (Dablander, Penn State)](https://faculty.ist.psu.edu/vhonavar/Courses/causality/Causal-inference.pdf)

---

## 6. RCTs — The Gold Standard

A **Randomised Controlled Trial (RCT)** randomly assigns units to treatment (T=1) or control (T=0). Randomisation ensures:

```
{Y(0), Y(1)} ⊥ T
```

The treatment and potential outcomes are independent — there are no confounders because randomisation balances all covariates (observed and unobserved) in expectation. The ATE can be estimated as a simple difference in means:

```
ATE = E[Y | T=1] − E[Y | T=0]
```

### Limitations of RCTs

| Limitation | Detail |
|---|---|
| **Ethical constraints** | Cannot randomise to harmful treatments (e.g. smoking) |
| **Feasibility** | Some treatments cannot be randomised (gender, nationality, historical events) |
| **External validity** | RCT populations may not generalise to the real-world population |
| **Compliance / SUTVA violations** | Participants may not comply; treatment may spill over to control group |
| **Cost and time** | Expensive; may take years |
| **Online A/B tests** | Closer to RCTs but still face network effects (social platforms), novelty effects, and multi-armed bandit tradeoffs |

---

## 7. Observational Causal Methods

When randomisation is not possible, these quasi-experimental methods estimate causal effects under specific identifying assumptions.

### 7.1 Difference-in-Differences (DiD)

**Idea:** Compare the change over time in outcomes for a treated group vs a control group. The "difference in differences" removes time trends and time-invariant confounders.

```
DiD = [E(Y | Treated, Post) − E(Y | Treated, Pre)]
    − [E(Y | Control, Post) − E(Y | Control, Pre)]
```

**Example:** A city introduces a minimum wage increase (treatment). Compare employment growth before/after in that city vs a comparable city without the increase (control).

**Key assumption — Parallel trends:** In the absence of treatment, the treated and control groups would have trended identically over time. This is untestable but can be supported by showing pre-treatment parallel trends.

Source: [Difference-in-Differences Estimation — Columbia Public Health](https://www.publichealth.columbia.edu/research/population-health-methods/difference-difference-estimation)

> **When to use DiD:** When you have panel data (same units observed pre and post), a natural experiment introduces a treatment to some units but not others, and the parallel trends assumption is plausible.

### 7.2 Propensity Score Matching (PSM)

**Idea:** Match each treated unit to one or more control units with similar **propensity scores** — the estimated probability of receiving treatment given observed covariates.

```
e(X) = P(T = 1 | X)   ← propensity score (estimated via logistic regression)
```

By matching on e(X) instead of the full covariate vector X, dimensionality is reduced (propensity score theorem: balancing the score balances all included covariates in expectation).

**Procedure:**
1. Estimate propensity scores via logistic regression.
2. Match treated units to control units with similar scores (nearest-neighbour, caliper, or kernel matching).
3. Check covariate balance: standardised mean differences < 0.1 for all covariates.
4. Estimate ATT as the mean difference in outcomes between matched pairs.

**Critical limitation:** PSM only balances on **observed** covariates. Unmeasured confounders still bias estimates. This is the most important limitation to mention in interviews.

Source: [Chapter 14 — Matching, The Effect (Huntington-Klein)](https://theeffectbook.net/ch-Matching.html)

> **When to use PSM:** When you have rich observational data on potential confounders, and the unconfoundedness assumption is plausible given the covariates available.

### 7.3 Instrumental Variables (IV)

**Idea:** Find a variable Z (the **instrument**) that:
1. Is correlated with the treatment T (relevance).
2. Affects the outcome Y **only through T** (exclusion restriction).
3. Is independent of unmeasured confounders (exogeneity / as-good-as-random).

If such a Z exists, it provides "natural randomisation" — variation in T driven by Z is unconfounded.

```
Z → T → Y    (and Z does NOT directly affect Y)
```

**Estimation:** Two-Stage Least Squares (2SLS):
1. Stage 1: Regress T on Z (and any controls) to get T̂ (the part of T explained by Z).
2. Stage 2: Regress Y on T̂ (and controls).

**Classic example:** Estimating the return to education. Education is confounded by ability. Use **proximity to college** as an instrument: proximity affects how much education you get (relevance), but (arguably) affects income only through the education it enables (exclusion), and is roughly independent of ability (exogeneity).

**IV estimates the LATE (Local Average Treatment Effect):** The effect for "compliers" — those whose treatment status is changed by the instrument. This is not the ATE for the whole population.

Source: [Instrumental Variables — MCP Analytics](https://mcpanalytics.ai/articles/instrumental-variables-practical-guide-for-data-driven-decisions)

> **IV challenges:** The exclusion restriction is untestable. Weak instruments (F-statistic < 10 in Stage 1) lead to biased IV estimates. Finding valid instruments requires domain knowledge and creativity.

### 7.4 Regression Discontinuity Design (RDD)

**Idea:** When treatment is assigned based on a continuous **running variable** R crossing a threshold c (R ≥ c → treated), units just above and just below the cutoff are comparable — they differ only in treatment assignment.

```
Example: Students scoring ≥ 70 on an entrance exam get a scholarship.
Compare outcomes for students scoring 68–69 (control) vs 70–71 (treatment).
```

The causal effect is estimated as the **jump (discontinuity) in the outcome at the cutoff**.

**Sharp RDD:** Treatment is a deterministic function of R (everyone above c is treated).  
**Fuzzy RDD:** Crossing the threshold increases the probability of treatment but doesn't guarantee it (use IV on the discontinuity).

**Key assumptions:**
- Units cannot precisely manipulate the running variable to land just above the cutoff (test with density tests, e.g. McCrary density test).
- No other policy changes at the cutoff.

**Limitation:** Estimates a **local** effect at the cutoff, not necessarily generalisable to other subgroups.

Source: [J-PAL Research Resources — Estimation Methods](https://www.povertyactionlab.org/sites/default/files/research-resources/Estimation-methods-2019-07.pdf)

### 7.5 Comparison of Observational Methods

| Method | Key Assumption | Handles Unmeasured Confounders? | Estimates |
|---|---|---|---|
| **Propensity Score Matching** | Unconfoundedness given X | No | ATT |
| **Difference-in-Differences** | Parallel trends | Partially (time-invariant only) | ATT |
| **Instrumental Variables** | Exclusion restriction + exogeneity | Yes (for Z-driven variation) | LATE |
| **Regression Discontinuity** | Continuity at cutoff; no manipulation | Yes (locally at cutoff) | LATE at cutoff |

> **Which method to choose:** PSM when you have rich covariate data and believe unconfoundedness; DiD when you have pre/post panel data and a natural experiment; IV when you have a credible instrument; RDD when treatment is determined by a threshold rule.

---

## 8. DAGs and the Backdoor Criterion

**Directed Acyclic Graphs (DAGs)** are a graphical language for encoding causal assumptions. Nodes are variables; directed edges represent direct causal effects; "acyclic" means no variable causes itself.

Source: [Causal Inference — A Tale of Three Frameworks](https://arxiv.org/html/2511.21516v1)

### Three Structural Patterns

| Structure | Graph | Effect of Conditioning on M |
|---|---|---|
| **Chain (mediator)** | X → M → Y | Blocks the causal path — do NOT condition if you want the total effect |
| **Fork (confounder)** | X ← Z → Y | Removing it closes the backdoor path — DO condition to block confounding |
| **Collider** | X → C ← Y | Opens a spurious path — do NOT condition |

### The Backdoor Criterion

A set of variables S satisfies the **backdoor criterion** for estimating the causal effect of X on Y if:
1. No variable in S is a descendant of X.
2. S blocks every **backdoor path** from X to Y (paths that start with an arrow into X).

If S satisfies the backdoor criterion, then:
```
P(Y | do(X)) = Σ_s P(Y | X, S=s) P(S=s)
```
i.e., conditioning on S (e.g. in a regression) identifies the causal effect.

### Confounder vs Mediator vs Collider — Decision Table

| Variable type | Role | Should you condition? | Effect of conditioning |
|---|---|---|---|
| **Confounder** | Common cause of X and Y | YES | Removes confounding bias |
| **Mediator** | On the causal path X → M → Y | ONLY if you want direct effect | Blocks the mediated effect |
| **Collider** | Common effect of X and Y | NEVER | Creates spurious association |

**The single most dangerous mistake in observational analysis:** Including a collider as a covariate in a regression or stratification. It looks like responsible "controlling for covariates" but actually creates bias.

---

## 9. Berkson's Paradox

**Berkson's paradox** is a form of collider bias that arises from conditioning on a selected / hospitalised / joint-positive subpopulation.

**Classic example:**
- In the general population, diabetes and gallstones are independent.
- Among hospitalised patients: conditioning on "hospitalised" (a collider — both diabetes and gallstones independently cause hospitalisation) induces a negative correlation between them. Researchers may falsely conclude that diabetes protects against gallstones.

**Modern ML example:**
- Suppose you collect training data only from users who made a purchase.
- Price sensitivity and product quality may appear negatively correlated in this dataset (conditioning on "purchased").
- A model trained on this data will learn a spurious inverse relationship.

> **Berkson's paradox is a special case of collider bias and selection bias combined.** It is particularly insidious in medical/clinical research (where only patients receiving care appear in databases), in e-commerce (where only purchasing customers appear), and in social media (where engagement selects for certain content types).

---

## 10. Common Data Biases in ML

### Label Bias

The labels in the training data are systematically wrong or biased.

**Example:** Historical criminal recidivism labels reflect biased policing (more arrests in over-policed communities). A model trained to predict recidivism learns policing patterns, not true recidivism.

### Measurement Bias

The way a feature or outcome is measured systematically differs across groups.

**Example:** Proxies for health (e.g. healthcare expenditure) may be lower for minorities not because they are healthier but because they have less access to care. A model using this proxy will systematically underestimate health needs for minorities.

### Sampling Bias

The sample used to train the model does not represent the deployment population.

**Example:** A face recognition system trained primarily on lighter-skinned faces from North America will perform poorly on darker-skinned faces — the training sample does not represent the global population.

### Historical Bias

Even with perfectly collected data, if history reflects systemic inequities, the model learns and perpetuates those inequities.

### Aggregation Bias

Using a single model for subgroups with fundamentally different underlying relationships. Treating a heterogeneous population as homogeneous.

### Automation Bias

Humans defer too readily to model predictions, which can amplify model errors — particularly dangerous when model errors cluster in minority subgroups that operators have less experience with.

### Bias Summary Table

| Bias Type | Source | Example | Mitigation |
|---|---|---|---|
| **Label bias** | Biased historical decisions | Biased hiring labels | Audit labels; re-label; use counterfactual fairness |
| **Measurement bias** | Proxy variables measured differently by group | Healthcare spend as health proxy | Use better measures; audit proxies |
| **Sampling bias** | Non-representative training data | Face recognition on limited demographics | Stratified sampling; data collection from underrepresented groups |
| **Survivorship bias** | Only observing "survivors" | Successful companies in training data | Include failed examples; understand data generating process |
| **Confounding** | Spurious associations from common causes | Shoe size predicts reading level (age is confounder) | Causal modelling; control for confounders |
| **Collider / Berkson's** | Conditioning on a joint effect | Hospital data paradoxes | Causal DAG analysis; avoid conditioning on colliders |

---

## 11. 🎯 In the Interview

### Common Traps

**Trap 1 — Conditioning on a collider:**
> "I should control for as many variables as possible in my regression to be safe."

**Correct answer:** Conditioning on a collider **creates** bias where none existed. You must use a causal DAG to determine which variables should and should not be included. Blindly adding controls is dangerous. Conditioning on a mediator also blocks the causal path you are trying to estimate.

**Trap 2 — Correlation causation in A/B tests:**
> "Our A/B test showed that users who saw the new feature had 20% higher retention."

**Correct answer:** If the comparison is not fully randomised (e.g. users opted into the feature, or the test was not properly isolated), this is still observational. Self-selection into the feature may confound the result. Proper A/B tests require random assignment and proper isolation.

**Trap 3 — PSM handles all confounders:**
> "I used propensity score matching, so confounding is no longer a concern."

**Correct answer:** PSM only balances on **observed** covariates. If there are unmeasured confounders, PSM does not remove their bias. The unconfoundedness assumption is untestable. Always perform sensitivity analysis (e.g. Rosenbaum bounds) to assess robustness to unmeasured confounding.

**Trap 4 — DiD parallel trends:**
> "I used difference-in-differences — this must be causal."

**Correct answer:** DiD is causal only if the parallel trends assumption holds: in the absence of treatment, the treated and control groups would have had the same time trend. If treated groups were trending differently before treatment (e.g. because treatment was assigned to the worst-performing units — Ashenfelter's dip), DiD estimates are biased.

**Trap 5 — Simpson's paradox aggregation:**
> "The drug works in the aggregate data — the overall success rate is higher."

**Correct answer:** Disaggregate by subgroups. If the subgroup-level rates reverse (Simpson's paradox), the aggregate conclusion is misleading. Determine whether the third variable is a confounder (stratify) or a mediator (do not stratify), based on the causal DAG.

**Trap 6 — Survivorship bias in ML:**
> "I trained on all historical data I had — it should be representative."

**Correct answer:** Ask how the data were generated. If the data only include surviving products, companies, or users, you may be learning from a filtered subset. Failed products/users that were removed from the dataset carry crucial information.

### Key Causal Vocabulary for Interviews

| Term | One-line definition |
|---|---|
| Confounder | Common cause of treatment and outcome; controls for it removes bias |
| Collider | Common effect of two variables; conditioning on it creates bias |
| Mediator | Variable on the causal path; conditioning on it blocks the effect |
| ATE | Average causal effect across the whole population |
| ATT | Average causal effect for those who received treatment |
| LATE | Effect for "compliers" in IV; local to the instrument |
| Parallel trends | DiD assumption: groups would trend identically without treatment |
| Exclusion restriction | IV assumption: instrument only affects outcome through treatment |
| Backdoor path | Non-causal path creating spurious association; must be blocked |

---

## Glossary

| Term | Definition |
|---|---|
| **Confounding** | Bias from a common cause of treatment and outcome creating spurious associations |
| **Simpson's paradox** | An association that reverses direction when data are disaggregated into subgroups |
| **Selection bias** | Non-representative sample due to systematic inclusion/exclusion criteria |
| **Survivorship bias** | Observing only survivors from a filtering process; ignoring non-survivors |
| **Collider** | Variable caused by two other variables; conditioning on it opens a spurious path |
| **Berkson's paradox** | Collider bias induced by conditioning on hospitalisation or selection |
| **Potential outcomes** | Yᵢ(0) and Yᵢ(1) — the outcomes a unit would have under each treatment value |
| **ATE** | E[Y(1) − Y(0)] — average causal effect across all units |
| **ATT** | E[Y(1) − Y(0) \| T=1] — causal effect for the treated |
| **LATE** | Causal effect for compliers in instrumental variables designs |
| **RCT** | Randomised Controlled Trial — randomised assignment to treatment/control |
| **Difference-in-Differences** | Causal method comparing changes over time between treated and control groups |
| **Propensity score** | P(T=1 \| X) — probability of treatment given covariates; used for matching |
| **Instrumental variable** | Variable that affects treatment but affects outcome only through treatment |
| **Regression discontinuity** | Design that exploits a threshold rule for treatment to estimate causal effects |
| **DAG** | Directed Acyclic Graph — graphical encoding of causal assumptions |
| **Backdoor criterion** | Condition on a set of variables to identify causal effects from observational data |
| **Mediator** | Variable on the causal path from treatment to outcome |
| **Parallel trends** | DiD assumption: treated and control groups trend identically absent treatment |
| **Exclusion restriction** | IV assumption: instrument affects outcome only through treatment |

---

## References

1. [Causal Inference — A Tale of Three Frameworks (arXiv 2511.21516)](https://arxiv.org/html/2511.21516v1)
2. [An Introduction to Causal Inference — Fabian Dablander (Penn State)](https://faculty.ist.psu.edu/vhonavar/Courses/causality/Causal-inference.pdf)
3. [Understanding Simpson's Paradox Using a Graph — Andrew Gelman, Columbia](https://statmodeling.stat.columbia.edu/2014/04/08/understanding-simpsons-paradox-using-graph/)
4. [Simpson's Paradox in Observational Study Designs — NCBI / NIH](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC7175433/)
5. [Difference-in-Differences Estimation — Columbia Public Health](https://www.publichealth.columbia.edu/research/population-health-methods/difference-difference-estimation)
6. [Chapter 14 — Matching — The Effect (Nick Huntington-Klein)](https://theeffectbook.net/ch-Matching.html)
7. [Instrumental Variables — MCP Analytics](https://mcpanalytics.ai/articles/instrumental-variables-practical-guide-for-data-driven-decisions)
8. [J-PAL Estimation Methods (DiD, PSM, IV, RDD)](https://www.povertyactionlab.org/sites/default/files/research-resources/Estimation-methods-2019-07.pdf)
9. [Causal Inference — So Much More Than Statistics (IJE, Oxford)](https://academic.oup.com/ije/article/45/6/1895/2999350)

---

*Part of the [Statistics for Data Science](README.md) section · [Guide Home](../README.md).*
