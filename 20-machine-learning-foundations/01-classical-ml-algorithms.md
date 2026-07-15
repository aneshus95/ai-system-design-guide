# Classical Machine Learning Algorithms

Classical (non-deep) machine learning algorithms remain the backbone of production data science: they are interpretable, computationally cheap, and often outperform neural networks on tabular data. This chapter covers every concept that routinely appears in Data Scientist and ML Engineer interviews — from the bias-variance tradeoff through ensemble methods and evaluation metrics — so you can answer any classical-ML question with precision and confidence.

## Table of Contents

- [Learning Paradigms](#learning-paradigms)
- [The ML Workflow](#ml-workflow)
- [Bias-Variance Tradeoff](#bias-variance)
- [Overfitting & Underfitting](#overfitting-underfitting)
- [Regularization — L1, L2, ElasticNet](#regularization)
- [Linear Regression](#linear-regression)
- [Logistic Regression](#logistic-regression)
- [Decision Trees](#decision-trees)
- [Random Forests — Bagging](#random-forests)
- [Gradient Boosting & XGBoost — Boosting](#gradient-boosting)
- [Bagging vs Boosting](#bagging-vs-boosting)
- [Support Vector Machines](#svm)
- [K-Nearest Neighbours](#knn)
- [Naive Bayes](#naive-bayes)
- [K-Means Clustering](#k-means)
- [PCA & Dimensionality Reduction](#pca)
- [Curse of Dimensionality](#curse-of-dimensionality)
- [Cross-Validation](#cross-validation)
- [Evaluation Metrics](#evaluation-metrics)
- [Class Imbalance](#class-imbalance)
- [Feature Engineering & Selection](#feature-engineering)
- [Encoding Categoricals](#encoding)
- [Handling Missing Data](#missing-data)
- [Hyperparameter Tuning](#hyperparameter-tuning)
- [Ensemble Methods — Stacking](#ensemble-methods)
- [Generative vs Discriminative Models](#generative-vs-discriminative)
- [Parametric vs Non-Parametric Models](#parametric-vs-nonparametric)
- [Interpreting What a Model Learned](#interpreting-models)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## Learning Paradigms <a name="learning-paradigms"></a>

| Paradigm | What the model learns from | Typical goal | Examples |
|---|---|---|---|
| **Supervised** | Labelled (X, y) pairs | Predict y for new X | Regression, Classification |
| **Unsupervised** | Unlabelled X only | Find structure / compress | Clustering, PCA, Autoencoders |
| **Semi-supervised** | Small labelled + large unlabelled | Leverage cheap unlabelled data | Self-training, Label Propagation |
| **Reinforcement** | Reward signal from environment | Learn a policy to maximise reward | Q-learning, PPO |

**Plain English:** supervised learning is like studying with an answer key; unsupervised is finding patterns in a pile of unlabelled flashcards; reinforcement is learning by trial-and-error with a score.

---

## The ML Workflow <a name="ml-workflow"></a>

```
Raw data
  │
  ▼
Data collection & labelling
  │
  ▼
EDA (distributions, missing values, correlations)
  │
  ▼
Feature engineering & preprocessing
  │
  ▼
Train / Validation / Test split
  │
  ▼
Model selection + hyperparameter tuning
  │
  ▼
Evaluation on held-out test set
  │
  ▼
Deployment + monitoring (data drift, performance decay)
```

Key rule: **the test set must never influence any preprocessing or tuning decision** — it is touched exactly once for final reporting.

---

## Bias-Variance Tradeoff <a name="bias-variance"></a>

**Total error = Bias² + Variance + Irreducible noise**

| Term | What it means | Cause |
|---|---|---|
| **Bias** | Systematic error — model consistently misses the true signal | Model too simple (underfits) |
| **Variance** | Sensitivity to noise — predictions change a lot across training sets | Model too complex (overfits) |
| **Irreducible** | Noise inherent in the data | Cannot be reduced |

**The U-curve (test error vs model complexity):**

```
Error
  │
  │\                        /
  │ \          Variance   /
  │  \       grows fast  /
  │   \                 /
  │    \_______________/   ← sweet spot
  │       Bias shrinks
  │
  └─────────────────────────► Complexity
     Simple          Complex
     (high bias)   (high variance)
```

**Plain English:** A ruler fit through noisy sine-wave data always misses (high bias). A degree-20 polynomial fits the training points perfectly but wobbles wildly on new data (high variance). You want the sweet spot in between.

---

## Overfitting & Underfitting <a name="overfitting-underfitting"></a>

- **Underfitting**: training error is high. Fix: more features, higher-capacity model, less regularization.
- **Overfitting**: training error is low, validation error is much higher. Fix: more data, regularization, dropout, early stopping, simpler model.

**Diagnostic**: plot learning curves (train vs validation error as a function of training-set size). Persistent gap = overfitting; both curves plateau high = underfitting.

---

## Regularization — L1, L2, ElasticNet <a name="regularization"></a>

Regularization adds a penalty to the loss function to constrain coefficient magnitude, trading a small increase in bias for a large decrease in variance.

| Method | Penalty added to loss | Effect on weights | Use case |
|---|---|---|---|
| **L1 (Lasso)** | λ Σ\|wᵢ\| | Drives some weights exactly to 0 → **sparse model / feature selection** | Many irrelevant features |
| **L2 (Ridge)** | λ Σwᵢ² | Shrinks all weights toward 0, none exactly 0 | Correlated features, stable predictions |
| **ElasticNet** | α·L1 + (1-α)·L2 | Groups correlated features, some sparsity | Both scenarios |

**Constraint-shape intuition (why L1 gives sparsity):**

```
       L2 constraint (circle)      L1 constraint (diamond)

            |                              |
         ___|___                        ___|___
        /   |   \                      /   |   \
       |    |    |                    /    |    \
  ─────┼────+────┼─────          ────+─────+─────+────
       |    |    |                    \    |    /
        \___|___/                      \___|___/
            |                              |

  Elliptical loss contours meet       Elliptical loss contours most
  circle at a smooth point →          likely meet diamond at a CORNER →
  weights close to 0 but not 0       weight = exactly 0 (sparse!)
```

**Plain English:** L1 penalty is shaped like a diamond; the loss ellipses tend to hit the diamond at a corner, which sits on an axis → one or more weights become exactly zero.

---

## Linear Regression <a name="linear-regression"></a>

**In plain English:** draw the **straight line (or flat plane) that sits as close as possible to all the points**, then read predictions off that line. Each feature gets a slope that says "how much does the outcome move when I nudge this feature."

**Model:** ŷ = β₀ + β₁x₁ + … + βₚxₚ  
**Loss:** Mean Squared Error (MSE) = (1/n) Σ(yᵢ − ŷᵢ)²  
**Solution:** Closed-form normal equations (β = (XᵀX)⁻¹Xᵀy) or gradient descent.

**Interpreting the coefficients (the part interviewers probe):**
- **Slope βⱼ** = *"holding every other feature fixed, a **one-unit** increase in xⱼ changes the predicted y by βⱼ (in y's units)."* Example: `price = 50,000 + 300·(sqft)` → each extra square foot adds **$300** to the predicted price. That "**holding others fixed**" clause is the whole meaning — it's the effect of *that* feature *after accounting for* the others.
- **Intercept β₀** = the predicted y when **all features are 0** (often not physically meaningful — e.g. a house with 0 sqft — it's just where the line crosses the axis).
- **Sign** = direction (positive → y rises with the feature; negative → y falls). **Magnitude** = strength, *but only comparable across features if you standardize them* — otherwise a coefficient's size just reflects the feature's units (a "per-gram" coefficient looks 1000× bigger than the same effect "per-kg").
- **Standardized (beta) coefficients** — refit on z-scored features so every coefficient is "effect per 1 standard deviation"; now magnitudes are comparable → you can rank feature importance.
- **Statistical significance** — each coefficient has a **p-value** (is it distinguishable from 0?) and a **confidence interval** (plausible range). A big coefficient with a huge CI isn't trustworthy.
- **Caution — multicollinearity:** when features are correlated, individual coefficients become unstable and hard to interpret (the model can't tell which correlated feature deserves the credit). Check with **VIF**.

**Classical assumptions (LINE):**
1. **L**inearity — relationship between X and y is linear.
2. **I**ndependence — residuals are independent.
3. **N**ormality — residuals are normally distributed.
4. **E**qual variance (homoscedasticity) — residual variance is constant.

Violations → biased or inefficient estimates. Fixes: log/Box-Cox transform, add interaction terms, switch to a non-parametric model.

> **Interpreting a log-transformed target:** if you model `log(y)`, a coefficient βⱼ means a one-unit rise in xⱼ multiplies y by ≈ `e^βⱼ` — i.e. roughly a **βⱼ × 100% percent change** in y (for small β). Common for prices/counts.

**How good is the fit? — R² and Adjusted R²**

- **R² (coefficient of determination)** = `1 − SS_res/SS_tot` = the **fraction of the target's variance the model explains**, measured against the "just predict the mean" baseline. R²=0.8 → 80% of the variation is explained; R²=0 → no better than the mean; R²=1 → perfect; **R²<0 → worse than the mean** (can happen on a test set).
- **The problem with R²:** it **never decreases when you add a feature — even a useless, random one**. So a high R² can just mean "many predictors," not a better model. You can't use plain R² to compare models with different numbers of features.
- **Adjusted R²** fixes this by **penalizing extra predictors** — it only rises if a new feature improves the fit *more than chance would*:

```
              (1 − R²)(n − 1)
 Adj R² = 1 − ───────────────      n = # rows,  p = # predictors
                 n − p − 1
```

| | R² | Adjusted R² |
|---|---|---|
| Adding a useless feature | **goes up** (or stays) | **goes down** (penalized) |
| Adding a genuinely useful feature | goes up | goes up (if it beats the penalty) |
| Compare models with different # of features | ❌ misleading | ✅ the right tool |
| Range | 0→1 (can be <0) | ≤ R², can be more negative |

> **Rule of thumb:** report **R²** to say how much variance is explained; use **Adjusted R²** when comparing models or doing feature selection (so you're not rewarded for just piling on predictors). And pair either with **RMSE/MAE** — R² tells you the *proportion* explained, not the error in real units.

---

## Logistic Regression <a name="logistic-regression"></a>

**In plain English:** it's linear regression bent through an **S-curve (sigmoid)** so the output is squeezed into a **probability between 0 and 1**. The straight-line part scores how strongly the evidence points to class 1; the S-curve turns that score into "% chance."

**Model:** P(y=1|x) = σ(wᵀx + b) where σ(z) = 1/(1+e⁻ᶻ) (sigmoid)  
**Loss:** Binary cross-entropy (log-loss) = −[y log(p̂) + (1−y) log(1−p̂)]  
**Output:** Probability in (0,1), threshold at 0.5 for class decision (tunable).

Despite the name, logistic regression is a **classification** algorithm. It is linear in the log-odds space: log(p/(1−p)) = wᵀx.

**Interpreting the coefficients (odds ratios — the key skill):**
- The coefficient wⱼ is an effect on the **log-odds**, which is awkward — so you exponentiate it: **`exp(wⱼ)` is the *odds ratio*.**
- Reading it: *"holding other features fixed, a one-unit increase in xⱼ **multiplies the odds** of the positive class by `exp(wⱼ)`."* `exp(w) = 1.5` → 50% higher odds; `exp(w) = 0.8` → 20% lower odds; `exp(w) = 1` → no effect.
- **Sign:** positive wⱼ → feature pushes toward class 1; negative → toward class 0.
- **Odds ≠ probability** — an odds ratio of 2 doubles the *odds*, which is not "doubles the probability" (the probability change depends on where you start on the S-curve). This nuance is a classic interview trap.

**Why use it?** Interpretable coefficients (exp(wᵢ) = odds ratio), calibrated probabilities, fast to train, good baseline before trying complex models.

---

## Decision Trees <a name="decision-trees"></a>

**Idea:** Recursively split the feature space using axis-aligned cuts that maximise "purity" of the resulting child nodes.

**Split criteria:**

| Criterion | Formula | Used for |
|---|---|---|
| **Gini impurity** | 1 − Σpᵢ² | CART (sklearn default) |
| **Entropy** | −Σpᵢ log₂(pᵢ) | ID3 / C4.5 |
| **Information gain** | Entropy(parent) − weighted Entropy(children) | Measures quality of split |
| **Variance reduction** | For regression trees | Regression |

**Plain English:** Gini asks "how often would you misclassify a randomly drawn item if you labelled it randomly according to this node's distribution?" Lower = purer.

**Pros:** Interpretable, handles mixed types, no scaling needed.  
**Cons:** High variance (unstable to small data changes), prone to overfitting without pruning.

---

## Random Forests — Bagging <a name="random-forests"></a>

**In plain English:** instead of trusting one opinionated decision tree, **ask hundreds of slightly different trees and take a vote**. Each tree sees a different random slice of the data and features, so their individual mistakes cancel out — wisdom of the crowd.

**Key idea (Bagging = Bootstrap AGGregatING):**
1. Draw B bootstrap samples (sampling with replacement) from training data.
2. Train a deep decision tree on each sample — with an additional twist: at each split, only a random subset of features (√p for classification, p/3 for regression) is considered.
3. Aggregate: majority vote (classification) or mean (regression).

```
Training data
  │
  ├──Bootstrap 1──► Tree 1 ──►  prediction₁ ─┐
  ├──Bootstrap 2──► Tree 2 ──►  prediction₂ ─┤──► Vote/Average ──► Final
  └──Bootstrap B──► Tree B ──►  predictionB ─┘
```

**Why it works:** Trees built on different bootstraps are decorrelated. Averaging uncorrelated models reduces variance without increasing bias.

**Out-of-bag (OOB) error:** ~37% of samples are left out of each bootstrap — these can be used as a free validation set.

---

## Gradient Boosting & XGBoost — Boosting <a name="gradient-boosting"></a>

**In plain English:** build trees **one after another, where each new tree focuses on fixing the mistakes the previous ones made**. It's like a student reviewing only the questions they got wrong on each pass — slowly grinding the error down. (Random forest builds trees *in parallel and votes*; boosting builds them *in sequence to correct*.)

**Key idea:** Build trees **sequentially**, each new tree fitting the **residuals (negative gradient of the loss)** of the current ensemble.

```
F₀(x) = initial prediction (e.g., mean of y)
For m = 1 to M:
    rᵢ = −∂L/∂F(xᵢ)   (pseudo-residuals)
    hₘ = tree fit to rᵢ
    F_m(x) = F_{m-1}(x) + η·hₘ(x)   (η = learning rate)
```

**XGBoost extras over vanilla GBM:**
- Second-order (Newton) gradient approximation → faster convergence.
- Built-in L1 & L2 regularization on leaf weights.
- Parallelized split-finding (column blocks).
- Handles sparse data / missing values natively.
- Tree pruning using `max_depth` and `min_child_weight`.

**Key hyperparameters:** `n_estimators`, `learning_rate` (η), `max_depth`, `subsample`, `colsample_bytree`, `lambda` (L2), `alpha` (L1).

---

## Bagging vs Boosting <a name="bagging-vs-boosting"></a>

| Dimension | Bagging (Random Forest) | Boosting (GBM/XGBoost) |
|---|---|---|
| Tree order | Parallel (independent) | Sequential (each corrects previous) |
| Goal | Reduce variance | Reduce bias (and variance) |
| Bias-variance | Low bias, lower variance than single tree | Lower bias, can overfit if too many rounds |
| Typical weakness | Can be slow for prediction (many trees) | Sensitive to noisy labels; needs tuning |
| Speed of training | Easily parallelisable | Sequential; XGBoost parallelises within each tree |
| When to prefer | Noisy data, fast training needed | Clean data, competition-style accuracy needed |

---

## Support Vector Machines <a name="svm"></a>

**In plain English:** find the **widest possible "street" between the two classes** and put the boundary down the middle. Only the points right on the curb (the **support vectors**) matter — everything else is ignored. A wider street generalizes better.

**Goal:** Find the hyperplane that maximises the **margin** (gap) between the two classes.

```
        ●  ●
    ●         support vectors ──►  ○  ○
  ●    ←margin→  ‖ ←──  ○
           hyperplane        ○   ○
```

- **Hard margin:** Only works if data is linearly separable.
- **Soft margin (C parameter):** Allows some misclassifications. High C = narrow margin (low bias, high variance); low C = wide margin (high bias, low variance).

**Kernel trick:** Map data into a high-dimensional space implicitly using a kernel function K(xᵢ, xⱼ) = φ(xᵢ)·φ(xⱼ), so the dot product in the high-dim space is computed without ever computing φ explicitly.

| Kernel | Formula | Use |
|---|---|---|
| Linear | xᵢ·xⱼ | Linearly separable / text |
| Polynomial | (xᵢ·xⱼ + c)ᵈ | Mild non-linearity |
| RBF (Gaussian) | exp(−γ‖xᵢ−xⱼ‖²) | General-purpose; most popular |
| Sigmoid | tanh(κxᵢ·xⱼ + c) | Neural-network-like |

**Plain English:** The kernel trick lets you find a linear boundary in a curved, high-dimensional space without paying the computational cost of transforming every point there.

---

## K-Nearest Neighbours <a name="knn"></a>

**In plain English:** *"you are the company you keep."* To classify a new point, **look at the K closest known points and copy the majority** — no training, no equation, just "what are my neighbours?"

**Idea:** To classify a new point, find its K nearest training points (by Euclidean / cosine distance) and return the majority class (or mean for regression).

- **K=1**: very low bias, very high variance (decision boundary follows every training point).
- **Large K**: smoother boundary, more bias, less variance.
- **No training phase** — all computation is at inference time (lazy learner).
- **Must scale features** — KNN is distance-based; unscaled features dominate.
- Curse of dimensionality hits KNN hard: in high dimensions, all points are roughly equidistant.

---

## Naive Bayes <a name="naive-bayes"></a>

**In plain English:** for each class, ask *"how typical are these feature values for this class?"*, **multiply those likelihoods together**, and pick the class with the highest score. "Naive" = it pretends the features are independent (e.g. treats each word in an email separately), which is wrong but works shockingly well for text.

**Model:** Applies Bayes' theorem with the "naive" assumption that features are conditionally independent given the class:

P(y|x₁,…,xₙ) ∝ P(y) · Π P(xᵢ|y)

| Variant | Likelihood P(xᵢ|y) | Use case |
|---|---|---|
| Gaussian NB | Normal distribution | Continuous features |
| Multinomial NB | Multinomial distribution | Word counts (text) |
| Bernoulli NB | Bernoulli (0/1) | Binary features (text, spam) |

**Pros:** Extremely fast, works well with small data, great baseline for text classification.  
**Con:** Independence assumption rarely holds; calibrated probabilities can be poor.

---

## K-Means Clustering <a name="k-means"></a>

**In plain English:** drop **K flags** on the map, let every point join its nearest flag, then **move each flag to the center of its crowd**, and repeat until the flags stop moving. You end up with K groups, each represented by its center point.

**Algorithm:**
1. Randomly initialise K centroids.
2. Assign each point to its nearest centroid.
3. Recompute centroids as the mean of assigned points.
4. Repeat 2–3 until centroids stop moving (convergence).

**Loss (inertia):** Σ Σ ‖xᵢ − μₖ‖²  
**Choosing K:** Elbow plot (inertia vs K) or silhouette score.  
**Limitations:** Assumes spherical clusters; sensitive to outliers and initialisation (use k-means++ for better init); must specify K in advance.

---

## PCA & Dimensionality Reduction <a name="pca"></a>

**In plain English:** **rotate the data to the angle where it's most spread out**, then keep only those few "most informative" directions and throw away the rest. Like photographing a 3-D object from the angle that shows the most detail, then working with the flat photo — you lose a little, but keep the essentials in far fewer numbers.

**PCA (Principal Component Analysis):**
- Finds orthogonal directions (principal components) of maximum variance in the data.
- Steps: standardise → compute covariance matrix → eigen-decompose → project onto top-k eigenvectors.
- **% variance explained** tells you how much information is retained.

```
Original 2D data         PCA: 1st PC (max variance)

    ○  ○                       ↗ PC1
  ○  ○  ○            ○ ○ ○ ○ ○ ○ ○ (projected, 1D)
    ○  ○
           ──► rotate & project onto PC1 ──►
```

**Other dimensionality reduction methods:**

| Method | Type | Preserves | Notes |
|---|---|---|---|
| PCA | Linear | Global variance | Fast, interpretable |
| t-SNE | Non-linear | Local neighbourhoods | Visualisation only (2-3D) |
| UMAP | Non-linear | Local + some global | Faster than t-SNE, better for large data |
| LDA | Supervised linear | Class separability | Classification preprocessing |
| Autoencoders | Non-linear | Learned features | Deep learning, flexible |

---

## Curse of Dimensionality <a name="curse-of-dimensionality"></a>

As the number of features grows, the volume of the feature space grows exponentially. Consequences:
- **Data sparsity:** You need exponentially more data to cover the space.
- **Distance concentration:** In high-D, the nearest and farthest neighbour become nearly equidistant — KNN fails.
- **Model complexity explosion:** More features → more parameters → overfitting risk.

**Mitigation:** Feature selection, PCA/UMAP, L1 regularisation, domain-driven feature engineering.

---

## Cross-Validation <a name="cross-validation"></a>

**Why?** A single train/val split has high variance — the split you happened to pick affects results. CV averages over many splits for a more reliable estimate.

**k-Fold CV:**
```
Fold 1:  [VAL | TRN | TRN | TRN | TRN]
Fold 2:  [TRN | VAL | TRN | TRN | TRN]
Fold 3:  [TRN | TRN | VAL | TRN | TRN]
Fold 4:  [TRN | TRN | TRN | VAL | TRN]
Fold 5:  [TRN | TRN | TRN | TRN | VAL]
                                   ↓
                         Average metric across 5 folds
```

| Variant | When to use |
|---|---|
| k-Fold (k=5 or 10) | Standard default |
| Stratified k-Fold | Class imbalance — preserves class ratios per fold |
| Leave-One-Out (LOO) | Very small datasets |
| Time-series split | Temporal data — always train on past, validate on future |

**Golden rule:** Fit any preprocessing (scalers, imputers, encoders) **inside** each fold's training set, not on the whole dataset — to avoid data leakage.

---

## Evaluation Metrics <a name="evaluation-metrics"></a>

### Confusion Matrix

```
                   Predicted
                 Positive  Negative
Actual Positive │   TP   │   FN   │
       Negative │   FP   │   TN   │
```

| Metric | Formula | When to use |
|---|---|---|
| **Accuracy** | (TP+TN)/(TP+TN+FP+FN) | Balanced classes only |
| **Precision** | TP/(TP+FP) | FP cost is high (spam filter) |
| **Recall (Sensitivity)** | TP/(TP+FN) | FN cost is high (cancer detection) |
| **F1** | 2·P·R/(P+R) | Balance P & R; imbalanced data |
| **ROC-AUC** | Area under TPR vs FPR curve | Overall discrimination; balanced |
| **PR-AUC** | Area under Precision vs Recall curve | Imbalanced classes (better than ROC-AUC) |
| **Log-loss** | −(1/n)Σ[y·log(p̂)+(1−y)·log(1−p̂)] | When calibrated probabilities matter |

**Regression metrics:**

| Metric | Formula | Note |
|---|---|---|
| **MAE** | (1/n)Σ\|yᵢ−ŷᵢ\| | Robust to outliers |
| **RMSE** | √(MSE) | Same units as y; penalises large errors |
| **R²** | 1 − SS_res/SS_tot | Proportion of variance explained; can be negative |

### ROC Curve (conceptual)

```
TPR (Recall)
1.0 ┤         ╭───────────
    │      ╭──╯
    │    ╭─╯
    │  ╭─╯
    │ ╭╯   Random (diagonal)
0.0 ┼──────────────────── FPR
   0.0                   1.0
   AUC = area under the curve (1.0 = perfect, 0.5 = random)
```

---

## Class Imbalance <a name="class-imbalance"></a>

**Problem:** A classifier can achieve 99% accuracy by predicting "majority class" always — completely useless for the minority class.

**Strategies:**

| Strategy | Mechanism | Notes |
|---|---|---|
| **Class weights** | Penalise misclassifying minority more | Built into sklearn (`class_weight='balanced'`) |
| **Oversampling (SMOTE)** | Synthesise minority samples by interpolating between existing ones | Avoids duplicating exact points |
| **Undersampling** | Remove majority class samples | Risk of discarding useful data |
| **Threshold tuning** | Move decision threshold below 0.5 to increase recall on minority | Requires probability output |
| **Ensemble (BalancedRF, EasyEnsemble)** | Combine resampling + boosting | Strong practical results |

**SMOTE (Synthetic Minority Over-sampling Technique):** For each minority sample, pick a random neighbour from its K nearest minority neighbours and create a new point on the line segment between them.

**Preferred metrics under imbalance:** F1, PR-AUC, Matthews Correlation Coefficient (MCC) — not accuracy.

---

## Feature Engineering & Selection <a name="feature-engineering"></a>

**Feature engineering** is the process of creating, transforming, or combining raw variables to better represent the underlying patterns for a model.

Common techniques:
- **Polynomial / interaction features:** x₁², x₁·x₂
- **Log / Box-Cox transform:** Compress skewed distributions
- **Binning / discretisation:** Convert continuous to ordinal buckets
- **Date decomposition:** Extract day-of-week, month, hour from timestamps
- **Aggregations:** Rolling mean, group-level statistics

**Feature selection methods:**

| Method | Type | How |
|---|---|---|
| Filter (χ², ANOVA F, mutual info) | Statistical | Score each feature independently of model |
| Wrapper (RFE) | Model-based | Recursively remove weakest features |
| Embedded (L1, tree importance) | Model trains with selection | Lasso zeroes out; tree counts splits |
| Permutation importance | Model-agnostic | Shuffle feature, measure performance drop |

---

## Encoding Categoricals <a name="encoding"></a>

| Method | When | Risk |
|---|---|---|
| **One-hot encoding** | Low cardinality (< ~20 levels) | Dimensionality explosion for high-cardinality |
| **Label / ordinal encoding** | Ordinal categories or tree models | Implies spurious order for nominal categories |
| **Target encoding** | High cardinality | Data leakage if not done inside CV folds |
| **Binary encoding** | Medium cardinality | More compact than OHE |
| **Embeddings (learned)** | Very high cardinality + neural nets | Requires training |

---

## Handling Missing Data <a name="missing-data"></a>

**Types of missingness:**
- **MCAR** (Missing Completely At Random): safe to drop rows.
- **MAR** (Missing At Random): missingness depends on other observed features; impute.
- **MNAR** (Missing Not At Random): missingness depends on the missing value itself; model it.

**Imputation strategies:**

| Strategy | Use when |
|---|---|
| Mean/median imputation | MCAR/MAR; quick baseline |
| Mode imputation | Categorical features |
| KNN imputation | Correlated features available |
| Model-based (IterativeImputer) | Complex patterns; computationally expensive |
| Missing indicator flag | When "missingness" itself is informative |

---

## Hyperparameter Tuning <a name="hyperparameter-tuning"></a>

| Method | How | Pros / Cons |
|---|---|---|
| **Grid Search** | Exhaustive over all combinations | Complete but slow; exponential cost |
| **Random Search** | Sample combinations randomly | Often finds good solutions with far fewer evaluations |
| **Bayesian Optimisation** | Build a surrogate model of the objective; pick next point to maximise expected improvement | Efficient; best for expensive models (Optuna, Hyperopt) |
| **Successive Halving / Hyperband** | Allocate more resources to promising configs early | Very fast; good with large search spaces |

---

## Ensemble Methods — Stacking <a name="ensemble-methods"></a>

**Bagging:** Train many copies of the same model type on bootstrap samples; aggregate. Reduces variance.  
**Boosting:** Train models sequentially; each corrects the previous. Reduces bias and variance.  
**Stacking (Stacked Generalisation):**

```
Level-0 (base models, trained on train set via CV):
   ┌──────────┐  ┌──────────┐  ┌──────────┐
   │  Model A │  │  Model B │  │  Model C │
   └────┬─────┘  └────┬─────┘  └────┬─────┘
        │              │              │
        └──────────────┼──────────────┘
                       ▼
             Meta-feature matrix (OOF predictions)
                       │
Level-1 (meta-learner, e.g. Ridge or Logistic Regression):
                  ┌────┴────┐
                  │  Meta   │
                  │ Learner │
                  └────┬────┘
                       ▼
                  Final prediction
```

**Stacking tips:** Use out-of-fold (OOF) predictions so base models never see their own training data when generating the meta-features — prevents leakage.

---

## Generative vs Discriminative Models <a name="generative-vs-discriminative"></a>

| | Generative | Discriminative |
|---|---|---|
| **Models** | P(x, y) or P(x\|y) and P(y) | P(y\|x) directly |
| **Examples** | Naive Bayes, GMM, LDA, VAE | Logistic Regression, SVM, Random Forest, Neural Nets |
| **Can generate new samples?** | Yes | No |
| **Better with small labelled data?** | Often yes (can exploit unlabelled x) | Usually needs more labelled data |
| **Decision boundary quality** | Usually worse with enough data | Usually better with enough data |

---

## Parametric vs Non-Parametric Models <a name="parametric-vs-nonparametric"></a>

| | Parametric | Non-Parametric |
|---|---|---|
| **Fixed # of parameters?** | Yes (fixed by architecture) | No — grows with data |
| **Examples** | Linear/Logistic Regression, Naive Bayes | KNN, Decision Trees, SVMs with RBF kernel, kernel density estimation |
| **Strong assumptions?** | Yes (e.g., linearity) | Fewer |
| **Need less data?** | Yes | No — needs more data to infer structure |
| **Faster to train?** | Usually | Varies |

---

## Interpreting What a Model Learned <a name="interpreting-models"></a>

Interviewers don't just ask "does it work?" — they ask **"what did the model learn, and can you explain a prediction?"** Every model exposes *something* you can read; the trick is knowing *what* and *how carefully*. There's a spectrum from **glass-box** (you can read the rule directly) to **black-box** (you need an external tool like SHAP).

### The glass-box models — read the parameters directly

- **Linear Regression → slopes (coefficients).** *"Holding others fixed, +1 unit of xⱼ changes y by βⱼ."* Sign = direction, magnitude = strength (**only comparable if features are standardized**). See [Linear Regression](#linear-regression).
- **Logistic Regression → odds ratios.** `exp(wⱼ)` = *"+1 unit of xⱼ multiplies the odds of the positive class by exp(wⱼ)."* >1 raises odds, <1 lowers them. See [Logistic Regression](#logistic-regression).
- **Decision Tree → the path.** The most literal of all: **read the if/then rules from root to leaf** — "income > 50k AND age < 30 → approve." Feature *importance* = how much each feature reduced impurity across all its splits (features used near the root, on more rows, matter more).
- **Naive Bayes → likelihood ratios.** Which feature values are most lopsided toward one class (e.g. the words most predictive of "spam").

### The ensemble models — feature importance (with a caveat)

Random Forests and Gradient Boosting are too big to read tree-by-tree, so they report **feature importance**:
- **Impurity/gain importance** (built-in, fast) — total impurity reduction a feature provided across all trees. **Caveat:** it's **biased toward high-cardinality / continuous features** and can be misleading with correlated features.
- **Permutation importance** (model-agnostic, more trustworthy) — shuffle one feature's values and measure how much accuracy *drops*; a big drop = the model relied on it. Slower but honest.
> Importance tells you a feature *mattered* — **not the direction** (did more of it push the prediction up or down?). For that, use SHAP.

### The universal tool — SHAP (works on *any* model)

**SHAP (SHapley Additive exPlanations)** is the modern answer to "explain this specific prediction." It assigns each feature a **signed contribution** (from game theory) showing how it pushed *this one prediction* above or below the average.
- **Direction + magnitude:** "+$40k income pushed the approval score **up** by 0.12; the recent late payment pushed it **down** by 0.30."
- **Local** (why *this* customer was denied) **and global** (average |SHAP| across all rows = overall importance, *with* direction).
- **Model-agnostic** — turns *any* black-box (XGBoost, a neural net) into something explainable. This is what you cite in an interview for "how do you explain a black-box model's decision." (See the [seller-analytics project](../23-my-projects/08-seller-behavior-analytics.md) for a worked use.)

### The geometry / structure models — read the shape

- **SVM → support vectors + margin.** For a **linear** SVM, the weight vector `w` is interpretable like a regression coefficient (which features tilt the boundary). For an **RBF kernel** SVM it's a black box — the "explanation" is *which support vectors* (borderline examples) define the boundary.
- **KNN → the neighbours themselves.** No coefficients at all (it's instance-based) — the explanation *is* the K nearest training points: "we predicted fraud because the 5 most similar past transactions were fraud."
- **K-Means → centroids.** Each cluster is summarized by its **center** (the mean feature values), which you read as a persona: "cluster 2 = high-spend, low-frequency." (See [K-Means](#k-means).)
- **PCA → loadings.** Each principal component is a weighted mix of original features; the **loadings** tell you what a component "means" ("PC1 is mostly size-related features").

### The interpretability spectrum (summary)

| Model | What you read | Gives direction? |
|---|---|---|
| Linear regression | slopes (coefficients) | ✅ yes |
| Logistic regression | odds ratios (`exp(w)`) | ✅ yes |
| Decision tree | the if/then path + split importance | ✅ (via the rules) |
| Random forest / GBM | feature importance (impurity or permutation) | ❌ magnitude only → use SHAP |
| SVM (linear) | weight vector | ✅ ; RBF → black box |
| KNN | the nearest neighbours | n/a (instance-based) |
| Naive Bayes | per-class likelihoods | ✅ yes |
| K-Means | cluster centroids | describes groups |
| PCA | component loadings | describes directions |
| **Any black box** | **SHAP values** | ✅ signed, per-prediction |

**The one-liner:** *Glass-box models (linear/logistic/trees) you read directly — slopes, odds ratios, and paths. Ensembles give importance (magnitude, not direction). For direction on any model — including black boxes — reach for SHAP, which gives a signed per-feature contribution for every individual prediction.*

---

## Interview Questions <a name="interview-questions"></a>

### Q: Explain the bias-variance tradeoff and how regularization addresses it.
**Strong answer:**
Bias is the systematic error from overly simplistic assumptions; variance is the error from excessive sensitivity to training-data noise. A complex model (many parameters, no constraints) has low bias but high variance — it memorises training data. Regularization (L1/L2) adds a penalty that shrinks weight magnitudes, introducing a small bias increase but substantially reducing variance. The λ hyperparameter lets you navigate the tradeoff: higher λ = simpler model = higher bias, lower variance.

### Q: When would you choose L1 over L2 regularization?
**Strong answer:**
Use L1 (Lasso) when you suspect many features are irrelevant and you want an inherently sparse model — L1 drives unimportant coefficients to exactly zero, performing automatic feature selection. Use L2 (Ridge) when most features are expected to contribute and you have correlated predictors — L2 distributes coefficients more evenly across correlated features, improving stability. ElasticNet blends both when you want some sparsity but the number of selected features matters.

### Q: How does gradient boosting differ from random forest?
**Strong answer:**
Random forest builds trees in parallel on bootstrap samples (bagging) and averages predictions, primarily reducing variance. Gradient boosting builds trees sequentially, with each tree fitting the negative gradient (residuals) of the loss from the current ensemble, primarily reducing bias. GBM/XGBoost often outperforms random forest on well-structured tabular data because it can learn complex patterns incrementally, but is more sensitive to hyperparameters and noisy labels. XGBoost adds built-in L1/L2 regularization and parallelized split-finding on top of vanilla GBM.

### Q: Why is accuracy a poor metric for imbalanced classes?
**Strong answer:**
With a 99:1 class ratio, predicting the majority class always yields 99% accuracy — but zero utility for the minority class. A model that never predicts "positive" will outscore a useful model on accuracy alone. Use precision-recall AUC (PR-AUC) or F1 for imbalanced problems because they explicitly account for the minority class. PR-AUC is preferable to ROC-AUC when the negative class vastly outnumbers the positive class.

### Q: What is the kernel trick in SVMs?
**Strong answer:**
The kernel trick allows SVMs to learn non-linear decision boundaries without explicitly computing a high-dimensional feature transformation. A kernel function K(xᵢ, xⱼ) implicitly computes the dot product φ(xᵢ)·φ(xⱼ) in a high-dimensional feature space, at the cost of just evaluating the kernel function. The RBF kernel, for example, corresponds to an infinite-dimensional feature space, yet evaluation is O(n) in the original dimensions. This means SVMs can learn arbitrarily complex boundaries while the optimisation stays tractable.

### Q: Describe SMOTE and its potential pitfalls.
**Strong answer:**
SMOTE (Synthetic Minority Over-sampling Technique) generates synthetic minority samples by interpolating between an existing minority point and one of its K nearest minority neighbours. This is better than naive duplication because it adds diversity rather than exact copies. Pitfalls: SMOTE must be applied only inside training folds — never before the train/test split, or you risk leakage (synthetic test-set information in training). SMOTE can also generate unrealistic points in sparse regions and does not adapt to the majority class distribution.

### Q: What is cross-validation and why must preprocessing be inside the CV loop?
**Strong answer:**
Cross-validation estimates generalisation error by rotating a validation fold across k partitions of training data, then averaging performance. Preprocessing (scaling, imputation, encoding) must be fitted inside each fold's training portion and then applied to the validation fold — never fitted on the whole dataset including the validation fold. If you scale using the entire dataset first, then split, the validation fold's statistics "leak" into the scaler, producing optimistically biased error estimates that won't hold on truly unseen data.

### Q: Explain information gain and how a decision tree uses it to choose splits.
**Strong answer:**
Information gain measures the reduction in entropy (or impurity) achieved by splitting on a feature: IG = Entropy(parent) − weighted average Entropy(children). The tree evaluates all possible splits on all features and chooses the one with the highest information gain. This is repeated recursively at each node until a stopping criterion is met (max depth, min samples per leaf, or zero impurity). Gini impurity is a slightly faster alternative that approximates entropy and is the default in scikit-learn's CART implementation.

### Q: When does PCA hurt rather than help?
**Strong answer:**
PCA can hurt when the principal components capturing most variance are not the components most predictive of the label — variance and predictive relevance are different things. It also destroys feature interpretability, complicating debugging. PCA is unsupervised; if the class-discriminating signal is in a low-variance direction, PCA will discard it (use LDA or supervised dimensionality reduction instead). Finally, PCA assumes linear relationships; for non-linear manifolds, UMAP or autoencoders are better choices.

### Q: What is the curse of dimensionality and how does it affect KNN?
**Strong answer:**
As dimensionality grows, the volume of the space grows exponentially, so data becomes extremely sparse. For KNN specifically, in high dimensions all pairwise distances converge to the same value — the nearest and farthest neighbours are nearly equidistant — so the "K nearest" concept loses meaning and predictions become random. Mitigation: reduce dimensionality (PCA, feature selection), use fewer but more informative features, or switch to models less sensitive to dimensionality (linear models, tree-based).

### Q: How does bagging reduce variance without increasing bias?
**Strong answer:**
Bagging trains B models on independent (bootstrap) samples and averages their predictions. If each model has variance σ² and the models were independent, the ensemble variance would be σ²/B. In practice, trees are correlated (they see overlapping training data), so variance reduction is less than σ²/B but still substantial. Because each individual tree is trained to near-zero bias, the average also has near-zero bias — bagging only affects variance.

### Q: What hyperparameters matter most for XGBoost and how do they interact?
**Strong answer:**
`n_estimators` and `learning_rate` (η) are inversely related — lower η requires more trees but generalises better; a common strategy is to set η low (0.01–0.1) and tune n_estimators via early stopping. `max_depth` controls tree complexity: deeper trees = more bias reduction but more variance. `subsample` and `colsample_bytree` inject randomness (similar to random forest's feature sampling), reducing variance. `lambda` (L2) and `alpha` (L1) directly regularise leaf weights. Start with `max_depth=6`, `learning_rate=0.1`, then tune with early stopping on a validation set.

### Q: Compare Naive Bayes and Logistic Regression for text classification.
**Strong answer:**
Both are linear classifiers in log-space. Naive Bayes is generative (models P(x|y)); Logistic Regression is discriminative (directly models P(y|x)). With small labelled data, Naive Bayes often outperforms LR because its generative assumptions act as a strong prior. With large data, LR typically wins because it doesn't need the (false) independence assumption to hold. Naive Bayes is extremely fast and streaming-friendly; LR produces better-calibrated probabilities and handles feature correlations more gracefully.

### Q: What is stacking and how do you prevent target leakage in the meta-learner?
**Strong answer:**
Stacking trains multiple diverse base models and then trains a meta-learner on their predictions. To prevent leakage, base model predictions for the meta-features must be generated via out-of-fold (OOF) cross-validation on the training set — each training example's meta-feature is the prediction from a model that was never trained on that example. On the test set, base models trained on the full training set generate the meta-features. This ensures the meta-learner never sees any training example "inside its own training loop."

### Q: How do you interpret a linear regression coefficient? What about logistic regression?
**Strong answer:**
A linear regression slope βⱼ means "holding all other features fixed, a one-unit increase in xⱼ changes the predicted y by βⱼ, in y's units." Sign is direction; magnitude is strength but only comparable across features if they're standardized (otherwise it just reflects units). For logistic regression, coefficients are on the log-odds scale, so I exponentiate: `exp(wⱼ)` is the odds ratio — a one-unit increase in xⱼ multiplies the odds of the positive class by that factor (>1 raises odds, <1 lowers). Key nuance: an odds ratio of 2 doubles the *odds*, not the *probability*.

### Q: A stakeholder asks why your XGBoost model denied a specific customer. How do you answer?
**Strong answer:**
XGBoost is a black box, so built-in feature importance only tells me which features mattered *globally* and in *magnitude* — not why this one customer was denied. I'd use **SHAP**, which gives a signed contribution per feature for that individual prediction: e.g. "recent late payment pushed the score down 0.30, high income pushed it up 0.12, net result below the approval threshold." SHAP is model-agnostic and gives both local (this customer) and global (average |SHAP|) explanations with direction, which plain importance lacks.

### Q: Feature importance says feature A is most important — can you trust it?
**Strong answer:**
Depends which importance. Built-in impurity/gain importance is fast but biased toward high-cardinality and continuous features, and it's unreliable when features are correlated (correlated features split the credit unpredictably). Permutation importance is more trustworthy — shuffle the feature and measure the accuracy drop. And importance only gives magnitude, not direction; for that I'd use SHAP. So I'd corroborate before acting on it.

---

## References <a name="references"></a>

- [Crash Course to Crack Machine Learning Interview — Bias vs Variance (Analytics Vidhya, 2025)](https://www.analyticsvidhya.com/blog/2025/08/bias-variance-tradeoff/)
- [Random Forest vs Gradient Boosting vs XGBoost: Key Differences Explained (DataLoopr)](https://dataloopr.com/blog/gradient-boosting-vs-random-forest-vs-xgboost-detailed-guide-123/)
- [Beyond Accuracy: A Deep Dive into Classification Metrics — Precision, Recall, F1, ROC-AUC, PR-AUC (Manish Mazumder, Substack)](https://manishmazumder5.substack.com/p/beyond-accuracy-a-deep-dive-into)
- [Imbalanced Dataset: Strategies to Fix Skewed Class Distributions 2026 (Label Your Data)](https://labelyourdata.com/articles/imbalanced-dataset)
- [Principal Component Analysis Interview Questions (Analytics Vidhya)](https://www.analyticsvidhya.com/blog/2022/09/principal-component-analysis-interview-questions/)
- [Feature Engineering in Machine Learning: A Practical Guide (DataCamp)](https://www.datacamp.com/tutorial/feature-engineering)

---

---

## Glossary

| Term | Simple explanation | Purpose |
|---|---|---|
| **Supervised Learning** | Training a model on labelled input-output pairs so it learns to predict outputs for new inputs | The most common ML paradigm for classification and regression tasks |
| **Unsupervised Learning** | Finding patterns in data that has no labels, such as grouping similar items together | Useful for clustering, compression, and exploratory data analysis |
| **Semi-supervised Learning** | Combining a small labelled dataset with a large unlabelled one to train better models | Reduces labelling costs while maintaining predictive quality |
| **Reinforcement Learning** | Training an agent to take actions by rewarding good outcomes and penalising bad ones | Used for game-playing agents, robotics, and recommendation policies |
| **Bias (ML)** | A systematic error where a model's predictions are consistently off in one direction | High bias means the model is too simple and underfits the data |
| **Variance (ML)** | How much a model's predictions change when trained on different samples of the same data | High variance means the model overfits and does not generalise |
| **Overfitting** | A model memorises training data so well it performs poorly on new, unseen data | The central problem regularisation and cross-validation are designed to prevent |
| **Underfitting** | A model is too simple to capture the underlying pattern, performing poorly even on training data | Signals the need for more features, a more complex model, or less regularisation |
| **Regularisation** | Adding a penalty to the loss function to shrink model weights and reduce overfitting | Controls the bias–variance tradeoff by limiting model complexity |
| **L1 Regularisation (Lasso)** | A penalty equal to the sum of absolute weight values; drives some weights exactly to zero | Performs automatic feature selection by producing sparse models |
| **L2 Regularisation (Ridge)** | A penalty equal to the sum of squared weight values; shrinks all weights but not to zero | Stabilises predictions when features are correlated |
| **ElasticNet** | A combined L1+L2 penalty that blends sparsity and weight shrinkage | Useful when you want some feature selection alongside stable coefficient estimates |
| **Linear Regression** | A model that predicts a continuous output as a weighted sum of input features | The go-to baseline for regression tasks; closed-form solution exists |
| **Logistic Regression** | A classification model that applies a sigmoid function to a linear combination of features to output a probability | Despite its name, it classifies; provides interpretable odds-ratio coefficients |
| **Sigmoid Function** | A smooth S-shaped function that maps any real number to a value between 0 and 1 | Used to convert raw scores into probabilities in logistic regression and neural networks |
| **Decision Tree** | A flowchart-like model that repeatedly splits data on feature values to reach a prediction | Highly interpretable; forms the building block for Random Forests and Gradient Boosting |
| **Gini Impurity** | A measure of how often a randomly chosen item would be mis-labelled if labelled randomly from a node's distribution | Used by CART to choose the best feature split at each tree node |
| **Entropy (Information)** | A measure of disorder or uncertainty in a set of class labels | Used by ID3/C4.5 decision trees; higher entropy means more mixed classes |
| **Information Gain** | The reduction in entropy achieved by splitting on a particular feature | Guides decision tree algorithms to choose the most useful splits |
| **Random Forest** | An ensemble of many decision trees, each trained on a bootstrap sample with random feature subsets | Reduces variance over a single tree; robust and hard to overfit |
| **Bagging** | Training multiple models on bootstrap samples and averaging their predictions | Reduces variance by decorrelating models; the mechanism behind Random Forest |
| **Bootstrap Sample** | A random sample drawn with replacement from the original dataset, typically the same size | Allows each tree to see a slightly different version of the data |
| **Out-of-Bag (OOB) Error** | Validation error estimated using the ~37% of samples not included in each bootstrap | A free internal validation score without needing a separate validation set |
| **Gradient Boosting** | Building trees sequentially where each tree corrects the residuals (errors) of the previous ensemble | Achieves very low bias; the principle behind XGBoost and LightGBM |
| **XGBoost** | An optimised gradient boosting library with built-in L1/L2 regularisation and parallelised split-finding | Industry standard for tabular data competitions and production ML |
| **Learning Rate (η)** | A small multiplier that scales how much each new tree contributes to the ensemble | Controls the speed and stability of gradient boosting; must be balanced with the number of trees |
| **Support Vector Machine (SVM)** | A classifier that finds the widest possible margin boundary separating two classes | Works well in high-dimensional spaces; can handle non-linear data via the kernel trick |
| **Kernel Trick** | Computing dot products in a high-dimensional space implicitly without ever transforming the data | Lets SVMs learn non-linear boundaries at the cost of just evaluating a kernel function |
| **RBF Kernel** | A kernel function that measures similarity by the squared distance between two points | The most popular SVM kernel; effective general-purpose non-linear classifier |
| **K-Nearest Neighbours (KNN)** | A prediction method that assigns a new point the majority class of its K closest training examples | Simple, non-parametric baseline; expensive at inference time |
| **Naive Bayes** | A probabilistic classifier that applies Bayes' theorem assuming features are independent given the class | Fast and effective for text classification, especially with small labelled datasets |
| **K-Means Clustering** | An unsupervised algorithm that partitions data into K groups by iteratively updating cluster centres | Common starting point for clustering tasks; requires specifying K in advance |
| **Inertia (K-Means)** | The total sum of squared distances from each point to its assigned cluster centroid | Lower inertia = tighter clusters; used in the elbow plot to choose K |
| **Silhouette Score** | A metric measuring how similar a point is to its own cluster compared to other clusters | Helps evaluate clustering quality when the true labels are unknown |
| **PCA (Principal Component Analysis)** | A method that finds the directions of maximum variance in data and projects it onto fewer dimensions | Reduces dimensionality while preserving as much information as possible |
| **t-SNE** | A non-linear dimensionality reduction technique that preserves local neighbourhood structure | Used almost exclusively for 2D/3D visualisations of high-dimensional data |
| **UMAP** | A faster non-linear technique than t-SNE that preserves both local and some global structure | Preferred for large datasets where t-SNE is too slow |
| **LDA (Linear Discriminant Analysis)** | A supervised dimensionality reduction method that maximises class separability | Used as a preprocessing step before classification to improve signal-to-noise |
| **Curse of Dimensionality** | As the number of features grows exponentially, data becomes sparse and distance metrics fail | Motivates feature selection, PCA, and regularisation in high-dimensional settings |
| **Cross-Validation (k-Fold)** | Splitting data into k equal folds and rotating which fold is used for validation | Gives a more reliable estimate of model performance than a single train/val split |
| **Stratified k-Fold** | Cross-validation that preserves the class ratio in each fold | Essential when dealing with imbalanced datasets to avoid misleading validation results |
| **Data Leakage** | When information from the validation or test set accidentally influences model training or preprocessing | Causes optimistically biased metrics that do not hold on truly unseen data |
| **Confusion Matrix** | A table showing the counts of true positives, false positives, true negatives, and false negatives | The foundation for computing most classification evaluation metrics |
| **Precision** | The fraction of predicted positives that are actually positive | Matters when the cost of false positives is high, such as spam filters |
| **Recall (Sensitivity)** | The fraction of actual positives that are correctly identified | Matters when the cost of false negatives is high, such as disease detection |
| **F1 Score** | The harmonic mean of precision and recall | Balances both metrics; preferred over accuracy for imbalanced datasets |
| **ROC-AUC** | The area under the curve of true positive rate vs. false positive rate at all thresholds | Measures overall discriminative ability; 1.0 is perfect, 0.5 is random |
| **PR-AUC** | The area under the precision-recall curve | More informative than ROC-AUC when the negative class vastly outnumbers the positive class |
| **MAE (Mean Absolute Error)** | The average of the absolute differences between predicted and actual values | A robust, interpretable regression metric that ignores outliers |
| **RMSE (Root Mean Squared Error)** | The square root of the average squared prediction errors | Penalises large errors more than MAE; in the same units as the target |
| **R² (Coefficient of Determination)** | The proportion of target variance explained by the model | 1.0 = perfect fit; can be negative if the model is worse than the mean |
| **Adjusted R²** | R² penalized for the number of predictors — only rises if a new feature helps more than chance | Fair comparison of models with different feature counts; avoids R²'s "always goes up" flaw |
| **SMOTE** | A technique that generates synthetic minority-class samples by interpolating between real ones | Addresses class imbalance without simply duplicating existing points |
| **Matthews Correlation Coefficient (MCC)** | A balanced metric that accounts for all four cells of the confusion matrix | More reliable than F1 for imbalanced binary classification |
| **One-Hot Encoding** | Converting a categorical variable into binary columns, one per category | Required for algorithms that cannot handle categorical inputs directly |
| **Target Encoding** | Replacing a category with the mean target value for that category | Effective for high-cardinality features but must be done inside CV folds to avoid leakage |
| **MCAR / MAR / MNAR** | Three types of missing data: completely random, dependent on other features, or dependent on the missing value itself | The type of missingness determines the safest imputation or handling strategy |
| **Bayesian Optimisation** | Using a surrogate model of the objective to intelligently choose the next hyperparameter configuration to try | Much more sample-efficient than grid or random search for expensive model training |
| **Stacking** | Training a meta-learner on the out-of-fold predictions of multiple base models | Combines diverse models to improve final predictions beyond any single model |
| **Generative Model** | A model that learns the joint distribution P(x, y) and can generate new samples | Can leverage unlabelled data and naturally handles uncertainty |
| **Discriminative Model** | A model that directly learns the conditional distribution P(y\|x) to predict labels | Typically achieves better decision boundaries with enough labelled data |
| **Parametric Model** | A model with a fixed number of parameters regardless of dataset size | Assumes a specific functional form; fast but may underfit complex data |
| **Non-Parametric Model** | A model whose complexity grows with the training data | Makes fewer assumptions; more flexible but needs more data and compute |
| **Permutation Importance** | A feature importance method that measures performance drop when a feature's values are randomly shuffled | Model-agnostic and avoids the biases of tree-based importance measures |
| **Recursive Feature Elimination (RFE)** | A wrapper feature selection method that repeatedly removes the weakest features and retrains | Computationally expensive but gives a ranked feature importance based on the chosen model |
| **Coefficient / Slope (β)** | In a linear model, the change in the prediction per one-unit change in a feature, holding others fixed | Reads out each feature's effect — direction (sign) and strength (magnitude) |
| **Intercept (β₀)** | The predicted value when all features are zero | Where the fitted line crosses the axis (often not physically meaningful) |
| **Standardized coefficient** | A coefficient fit on z-scored features (effect per 1 standard deviation) | Makes feature effects comparable so you can rank importance |
| **Odds ratio** | `exp(coefficient)` in logistic regression — the factor by which odds change per unit | Turns log-odds coefficients into an interpretable "×odds" statement |
| **Multicollinearity / VIF** | Correlated predictors; VIF quantifies how inflated a coefficient's variance is | Warns that individual coefficients are unstable and hard to interpret |
| **Feature Importance** | How much each feature contributed to a model's predictions overall | Ranks features; note it gives magnitude, usually not direction |
| **SHAP (SHapley Additive exPlanations)** | A model-agnostic method giving each feature a signed contribution to a single prediction | Explains *any* model (incl. black boxes) locally and globally, with direction |
| **Support vector** | The training points sitting on the margin that define an SVM's boundary | The only points that matter to the SVM; the model's "explanation" |
| **Centroid** | The mean feature-vector at the center of a K-Means cluster | Summarizes/interprets each cluster as a persona |
| **Loading (PCA)** | The weight of an original feature in a principal component | Tells you what a component "means" in terms of original features |
| **Glass-box vs black-box** | Models you can read directly (linear/trees) vs those needing external explanation | Determines whether you read parameters or reach for SHAP |

*Previous: [Multimodal Generation](../19-multimodal-generation/01-multimodal-generation.md) | Next: [Deep Learning Fundamentals](02-deep-learning-fundamentals.md)*
