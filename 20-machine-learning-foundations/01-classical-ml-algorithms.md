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

**Model:** ŷ = β₀ + β₁x₁ + … + βₚxₚ  
**Loss:** Mean Squared Error (MSE) = (1/n) Σ(yᵢ − ŷᵢ)²  
**Solution:** Closed-form normal equations (β = (XᵀX)⁻¹Xᵀy) or gradient descent.

**Classical assumptions (LINE):**
1. **L**inearity — relationship between X and y is linear.
2. **I**ndependence — residuals are independent.
3. **N**ormality — residuals are normally distributed.
4. **E**qual variance (homoscedasticity) — residual variance is constant.

Violations → biased or inefficient estimates. Fixes: log/Box-Cox transform, add interaction terms, switch to a non-parametric model.

---

## Logistic Regression <a name="logistic-regression"></a>

**Model:** P(y=1|x) = σ(wᵀx + b) where σ(z) = 1/(1+e⁻ᶻ) (sigmoid)  
**Loss:** Binary cross-entropy (log-loss) = −[y log(p̂) + (1−y) log(1−p̂)]  
**Output:** Probability in (0,1), threshold at 0.5 for class decision (tunable).

Despite the name, logistic regression is a **classification** algorithm. It is linear in the log-odds space: log(p/(1−p)) = wᵀx.

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

**Idea:** To classify a new point, find its K nearest training points (by Euclidean / cosine distance) and return the majority class (or mean for regression).

- **K=1**: very low bias, very high variance (decision boundary follows every training point).
- **Large K**: smoother boundary, more bias, less variance.
- **No training phase** — all computation is at inference time (lazy learner).
- **Must scale features** — KNN is distance-based; unscaled features dominate.
- Curse of dimensionality hits KNN hard: in high dimensions, all points are roughly equidistant.

---

## Naive Bayes <a name="naive-bayes"></a>

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

---

## References <a name="references"></a>

- [Crash Course to Crack Machine Learning Interview — Bias vs Variance (Analytics Vidhya, 2025)](https://www.analyticsvidhya.com/blog/2025/08/bias-variance-tradeoff/)
- [Random Forest vs Gradient Boosting vs XGBoost: Key Differences Explained (DataLoopr)](https://dataloopr.com/blog/gradient-boosting-vs-random-forest-vs-xgboost-detailed-guide-123/)
- [Beyond Accuracy: A Deep Dive into Classification Metrics — Precision, Recall, F1, ROC-AUC, PR-AUC (Manish Mazumder, Substack)](https://manishmazumder5.substack.com/p/beyond-accuracy-a-deep-dive-into)
- [Imbalanced Dataset: Strategies to Fix Skewed Class Distributions 2026 (Label Your Data)](https://labelyourdata.com/articles/imbalanced-dataset)
- [Principal Component Analysis Interview Questions (Analytics Vidhya)](https://www.analyticsvidhya.com/blog/2022/09/principal-component-analysis-interview-questions/)
- [Feature Engineering in Machine Learning: A Practical Guide (DataCamp)](https://www.datacamp.com/tutorial/feature-engineering)

---

*Previous: [Multimodal Generation](../19-multimodal-generation/01-multimodal-generation.md) | Next: [Deep Learning Fundamentals](02-deep-learning-fundamentals.md)*
