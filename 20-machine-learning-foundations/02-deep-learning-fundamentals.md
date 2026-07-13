# Deep Learning Fundamentals

This chapter covers the core building blocks every Data Scientist is expected to explain clearly in an interview: neurons, forward/backward passes, optimizers, normalization, regularization, CNNs, RNNs, and generative models. Transformers and self-attention are treated as a separate deep-dive in [LLM Internals](../01-foundations/01-llm-internals.md); this chapter supplies the DL foundations that make those architectures intelligible.

---

## Table of Contents

- [The Perceptron and MLP](#the-perceptron-and-mlp)
- [Forward Pass](#forward-pass)
- [Backpropagation and the Chain Rule](#backpropagation-and-the-chain-rule)
- [Activation Functions](#activation-functions)
- [Loss Functions](#loss-functions)
- [Optimizers](#optimizers)
- [Learning Rate and Schedules](#learning-rate-and-schedules)
- [Vanishing and Exploding Gradients](#vanishing-and-exploding-gradients)
- [Weight Initialization](#weight-initialization)
- [Batch Normalization vs Layer Normalization](#batch-normalization-vs-layer-normalization)
- [Dropout](#dropout)
- [Epochs, Batch Size, and Mini-Batch Training](#epochs-batch-size-and-mini-batch-training)
- [Overfitting in Deep Learning](#overfitting-in-deep-learning)
- [Early Stopping](#early-stopping)
- [Convolutional Neural Networks (CNNs)](#convolutional-neural-networks-cnns)
- [RNNs, LSTMs, and GRUs](#rnns-lstms-and-grus)
- [Sequence Models and the Bottleneck That Motivated Attention](#sequence-models-and-the-bottleneck-that-motivated-attention)
- [Embeddings](#embeddings)
- [Transfer Learning and Fine-Tuning](#transfer-learning-and-fine-tuning)
- [Generative Models — Autoencoders, VAEs, and GANs](#generative-models--autoencoders-vaes-and-gans)
- [Hyperparameters vs Parameters](#hyperparameters-vs-parameters)
- [Why Deep Learning Needs Lots of Data and Compute](#why-deep-learning-needs-lots-of-data-and-compute)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## The Perceptron and MLP

A **perceptron** is a single neuron: it takes a weighted sum of inputs, adds a bias, and passes the result through an activation function.

```
x1 ─┐
x2 ─┼──► [Σ w·x + b] ──► activation ──► output
x3 ─┘
```

**In plain English:** Think of a perceptron as a weighted vote. Each input casts a vote weighted by how important it is; the result gets squashed into a usable range.

A **Multi-Layer Perceptron (MLP)** stacks layers of perceptrons:

```
Input Layer    Hidden Layers    Output Layer
  [x1]           [h1] [h2]
  [x2]  ──────►  [h3] [h4]  ──────► [ŷ]
  [x3]           [h5] [h6]
```

Each layer learns increasingly abstract features. Without non-linear activations, any number of stacked linear layers collapses to a single linear transformation — making depth useless.

---

## Forward Pass

The forward pass computes predictions layer by layer:

1. `z = W · x + b` (linear combination)
2. `a = activation(z)` (non-linearity)
3. Repeat for each layer until you reach the output.
4. Compute loss `L(ŷ, y)`.

No learning happens in the forward pass — it is purely inference.

---

## Backpropagation and the Chain Rule

**In plain English:** After computing the loss, we need to know "which weight was most responsible?" Backpropagation answers this by propagating the error signal backwards through the network using the chain rule of calculus.

For a loss `L` dependent on weights through a chain of functions:

```
∂L/∂W₁ = (∂L/∂ŷ) · (∂ŷ/∂a₂) · (∂a₂/∂z₂) · (∂z₂/∂a₁) · (∂a₁/∂W₁)
```

**Backprop flow (ASCII):**

```
Forward ──────────────────────────────────►
  x → [Layer 1] → [Layer 2] → [Layer 3] → Loss
       ◄──────────────────────────────────
Backward  ∂L/∂W₁ ← ∂L/∂W₂ ← ∂L/∂W₃ ← ∂L
```

Each weight update: `W ← W − η · ∂L/∂W`

The chain rule is what makes this tractable: you only need local gradients at each node, and the full gradient is the product of local gradients along the path.

---

## Activation Functions

Non-linearities allow networks to approximate any function (Universal Approximation Theorem). Without them, a deep network is just a linear model.

| Function | Formula | Range | Key Property |
|----------|---------|-------|-------------|
| Sigmoid | `1 / (1 + e^{-x})` | (0, 1) | Smooth, saturates → vanishing gradients |
| Tanh | `(e^x − e^{-x}) / (e^x + e^{-x})` | (−1, 1) | Zero-centered, still saturates |
| ReLU | `max(0, x)` | [0, ∞) | Fast, sparse, can "die" |
| Leaky ReLU | `max(0.01x, x)` | (−∞, ∞) | Fixes dying ReLU |
| GELU | `x · Φ(x)` | (−∞, ∞) | Smooth stochastic gate, used in BERT/GPT |
| Softmax | `e^{xᵢ} / Σ e^{xⱼ}` | (0,1), sums to 1 | Multi-class output probabilities |

**Activation shapes (ASCII):**

```
Sigmoid:    ___/‾‾‾        Tanh:      __/‾‾
           /                         /
      ____/                    ‾‾\__/

ReLU:    /              Leaky ReLU:  /
        /                      ____/
  _____/                  ___./

GELU: similar to ReLU but slightly negative for small negative x, smooth curve
```

**Why ReLU is the default:** It does not saturate in the positive region, avoids vanishing gradients for deep networks, and is computationally cheap. GELU is preferred in Transformer architectures because its smooth probabilistic gating interacts better with attention.

---

## Loss Functions

| Loss | Use Case | Formula (simplified) |
|------|----------|-----------------------|
| MSE (Mean Squared Error) | Regression | `(1/n) Σ (y − ŷ)²` |
| MAE (Mean Absolute Error) | Regression, robust to outliers | `(1/n) Σ |y − ŷ|` |
| Binary Cross-Entropy | Binary classification | `−[y log ŷ + (1−y) log(1−ŷ)]` |
| Categorical Cross-Entropy | Multi-class classification | `−Σ yᵢ log ŷᵢ` |
| KL Divergence | VAEs, distribution matching | `Σ p log(p/q)` |

**In plain English:** MSE penalizes large errors heavily (squares them). Cross-entropy is better for classification because it directly penalizes confidence in the wrong answer — a model that says 99% wrong gets punished far more than one that says 51% wrong.

---

## Optimizers

All gradient-based optimizers answer the same question: given the gradient, how should I update the weights?

| Optimizer | Key Idea | Weakness |
|-----------|---------|----------|
| SGD | `W ← W − η·g` | Slow, noisy, sensitive to LR |
| SGD + Momentum | Add velocity term to smooth updates | Still needs careful LR tuning |
| RMSProp | Divide by running RMS of past gradients (adaptive per-param LR) | No momentum |
| Adam | Momentum + RMSProp; bias-corrected | Can generalize worse than SGD on some tasks |
| AdamW | Adam + decoupled weight decay | Default for most modern DL |

**Adam update rule:**

```
m ← β₁·m + (1−β₁)·g          # first moment (momentum)
v ← β₂·v + (1−β₂)·g²         # second moment (adaptive LR)
m̂ = m/(1−β₁ᵗ)                # bias correction
v̂ = v/(1−β₂ᵗ)
W ← W − η · m̂ / (√v̂ + ε)
```

**In plain English:** Adam keeps a memory of recent gradients (like momentum) and also tracks how noisy each gradient direction has been (like RMSProp). Noisy directions get a smaller step; consistent directions get a larger one. AdamW fixes a subtle bug where L2 regularization and weight decay are not equivalent in Adam.

---

## Learning Rate and Schedules

The learning rate `η` controls the step size. Too large → diverges. Too small → never converges.

**Common schedules:**

| Schedule | Behavior |
|----------|----------|
| Step decay | Drop LR by factor at fixed epochs |
| Cosine annealing | Smooth cosine decay, often with warm restarts |
| Linear warmup | Start small, ramp up — standard for Transformers |
| Cyclical LR | Oscillate between bounds to escape local minima |
| OneCycleLR | Single cycle up then down; very effective in practice |

**In plain English:** Start warm, let the model settle, then cool down. Warmup prevents large early updates from destabilizing freshly initialized weights.

---

## Vanishing and Exploding Gradients

**Vanishing:** Gradients shrink exponentially as they propagate back through many layers, especially with sigmoid/tanh. Early layers learn extremely slowly or not at all.

**Exploding:** Gradients grow exponentially, causing weight updates that blow up training.

```
Gradient magnitude through 10 sigmoid layers:

Layer:  10    9     8     7     6     5     4     3     2     1
        1.0 → 0.25 → 0.06 → 0.016 → 0.004 → ...  → ~0  (vanishing)
        1.0 → 4.0  → 16   → 64    → 256   → ...  → ∞   (exploding)
```

**Fixes:**

| Problem | Solution |
|---------|----------|
| Vanishing | ReLU activations; residual connections; careful init |
| Exploding | Gradient clipping (`clip_grad_norm_`); careful init |
| Both | Batch/Layer normalization; LSTM gating |

**Residual connection (skip connection):**

```
x ──────────────────────────► (+) ──► output
 └──► [Layer] ──► [Layer] ──►─┘
```

This gives gradients a "highway" directly back to earlier layers.

---

## Weight Initialization

Random initialization breaks symmetry (all-zeros → all neurons learn the same thing). The goal is to keep activation and gradient variances roughly constant across layers.

| Method | Formula (std) | Best For |
|--------|--------------|----------|
| Xavier / Glorot | `√(2 / (fan_in + fan_out))` | Sigmoid, Tanh |
| He / Kaiming | `√(2 / fan_in)` | ReLU, Leaky ReLU |
| LeCun | `√(1 / fan_in)` | SELU |

**In plain English:** If weights start too large, activations saturate immediately. Too small, signals die before reaching deep layers. Xavier keeps variance stable for symmetric activations; He compensates for ReLU zeroing out half its inputs.

---

## Batch Normalization vs Layer Normalization

Both normalize activations to speed up training and improve stability, but they differ in **which axis they normalize over**.

```
Batch Norm:                     Layer Norm:
Normalize across BATCH           Normalize across FEATURES
for each feature channel.        for each sample.

  Sample:  S1  S2  S3  S4          Sample: S1
Feature:                           F1 F2 F3 F4
  F1:     [x  x  x  x] ← μ,σ      [x  x  x  x] ← μ,σ
  F2:     [x  x  x  x]
  F3:     [x  x  x  x]
```

| Property | Batch Norm | Layer Norm |
|----------|-----------|-----------|
| Normalizes over | Batch dimension | Feature dimension |
| Batch-size sensitive | Yes — unstable at small batch | No |
| Train vs inference behavior | Different (running stats at inference) | Same |
| Best for | CNNs | Transformers, RNNs |
| Learnable params | γ, β per channel | γ, β per feature |

**In plain English:** Batch Norm compares each neuron's activation across the batch — this is unstable when batches are small or in online settings. Layer Norm compares each neuron within a single sample — safe regardless of batch size, which is why Transformers use it.

---

## Dropout

During training, randomly zero out a fraction `p` of activations at each forward pass. At inference, scale outputs by `(1−p)` (or equivalently use "inverted dropout" by dividing during training).

```
Training:                       Inference:
  [h1] [h2] [h3] [h4]           [h1] [h2] [h3] [h4]
    ↑         ↑                   ↑    ↑    ↑    ↑
  [x1] [x2] [x3] [x4]          [x1] [x2] [x3] [x4]
  (h2 and h4 zeroed out)        (all active, scaled)
```

**In plain English:** Dropout forces the network to learn redundant representations — no single neuron can be relied upon. This acts as training an ensemble of exponentially many sub-networks simultaneously. Typical values: 0.2–0.5 for hidden layers; do not apply after the final layer.

---

## Epochs, Batch Size, and Mini-Batch Training

| Term | Definition |
|------|-----------|
| Epoch | One full pass through the training set |
| Batch size | Number of samples per gradient update |
| Mini-batch SGD | Update weights every `B` samples (typical: 32–512) |

**Batch size trade-offs:**

| Small batch | Large batch |
|------------|------------|
| Noisy gradients → regularizing effect | Smoother gradients, faster convergence |
| Generalizes better empirically | May converge to sharper minima (worse generalization) |
| Slow wall-clock time (less GPU utilization) | Better GPU utilization |

---

## Overfitting in Deep Learning

A model overfits when it memorizes training data and fails to generalize. In DL, overfitting is common because models have millions of parameters.

**Diagnostics:** training loss ↓ while validation loss ↑ (gap widens).

**Remedies:**

- More data / data augmentation
- Dropout
- Weight decay (L2 regularization)
- Batch normalization (mild regularizing effect)
- Early stopping
- Simpler architecture

---

## Early Stopping

Monitor validation loss during training. Stop when it has not improved for `patience` epochs. Restore the best checkpoint.

```
Loss
│   ·· Train loss
│  ·  ·  ·  ·  ·
│ ·             ·  ·  ·  ·  ←── overfitting begins
│
│    ── Val loss
│   /‾‾‾\____
│            ‾‾‾‾‾‾‾‾‾‾‾ ←── val loss rises: stop here
└────────────────────────► Epoch
              ↑ best checkpoint
```

---

## Convolutional Neural Networks (CNNs)

CNNs exploit the spatial structure of images via **local connectivity** and **weight sharing**.

**Key concepts:**

| Term | Plain-English Definition |
|------|--------------------------|
| Kernel / Filter | Small weight matrix slid over input to detect a pattern |
| Stride | How many pixels the kernel jumps each step |
| Padding | Zeros added around the input to control output size |
| Pooling | Downsampling (max or average) to reduce spatial dimensions |
| Receptive field | The region of the original input a neuron "sees" |

**Output size formula:**
`(W − F + 2P) / S + 1` where W=input width, F=filter size, P=padding, S=stride.

**Feature hierarchy (ASCII):**

```
Input image
    │
  [Conv + ReLU]  ──► edges, blobs
    │
  [Conv + ReLU]  ──► textures, corners
    │
  [Conv + ReLU]  ──► object parts
    │
  [Pooling]       ──► spatial downsampling
    │
  [Fully Connected] ──► class scores
    │
  [Softmax]        ──► probabilities
```

**In plain English:** Early layers detect low-level features (edges). Deeper layers combine these into higher-level concepts (eyes, wheels). Pooling reduces spatial size, giving some translation invariance. Weight sharing means the same edge detector is reused across the whole image — far fewer parameters than a fully-connected approach.

---

## RNNs, LSTMs, and GRUs

**RNNs** process sequences by maintaining a hidden state that is updated at each time step:

```
RNN unrolled:

  x₁    x₂    x₃    x₄
  │     │     │     │
 [h₀]─►[h₁]─►[h₂]─►[h₃]─► output
```

**Problem:** Vanilla RNNs suffer from vanishing gradients over long sequences — they cannot remember events far in the past.

**LSTM** (Long Short-Term Memory) adds a **cell state** (`c`) — a separate "memory lane" — controlled by three gates:

```
LSTM cell:

        ┌─────────────────────────────────────┐
c_{t-1} ─┤ forget gate ──────────────────────►├── c_t
        │ input gate ──► new candidate ──────►│
h_{t-1} ─┤                                    ├── h_t
  x_t   ─┤ output gate ──────────────────────►│
        └─────────────────────────────────────┘
```

| Gate | What it controls |
|------|----------------|
| Forget | How much of c_{t-1} to keep |
| Input | How much new info to write to cell |
| Output | How much of cell state to expose as h_t |

**GRU** (Gated Recurrent Unit) merges cell state and hidden state, reducing to two gates (reset, update). Fewer parameters, similar performance to LSTM in many tasks.

| Model | Gates | Parameters | Good for |
|-------|-------|-----------|---------|
| RNN | None | Fewest | Very short sequences |
| GRU | 2 | Medium | Medium sequences, faster training |
| LSTM | 3 | Most | Long-range dependencies |

---

## Sequence Models and the Bottleneck That Motivated Attention

Encoder-decoder RNNs compress the entire input sequence into a **single fixed-length vector** before decoding. For long sequences, this is a severe information bottleneck:

```
"The cat sat on the mat" ──► [single vector] ──► translation
(information loss for long sentences)
```

This bottleneck motivated the **attention mechanism**: instead of one summary vector, the decoder can attend to all encoder hidden states at each decoding step. This is the bridge to Transformers — see [LLM Internals](../01-foundations/01-llm-internals.md) for the full derivation of self-attention and the Transformer architecture.

---

## Embeddings

An **embedding** is a dense, low-dimensional representation of a discrete object (word, user, item). Embeddings are learned during training and encode semantic relationships in geometric space.

```
"king"   → [0.2,  0.8, -0.1, ...]
"queen"  → [0.2,  0.8, -0.1, ...]  (similar direction)
"apple"  → [-0.9, 0.1,  0.7, ...]  (different cluster)
```

King − Man + Woman ≈ Queen is the classic example of linear relationships in embedding space.

---

## Transfer Learning and Fine-Tuning

**Transfer learning:** Take a model pre-trained on a large dataset (e.g., ImageNet, large text corpora), and reuse its weights for a new task.

**Why it works:** Lower layers learn general features (edges, grammar) that transfer across domains; only higher layers need to be adapted.

| Strategy | What you do | When to use |
|----------|------------|------------|
| Feature extraction | Freeze all layers; train only head | Tiny target dataset |
| Fine-tuning | Unfreeze some/all layers; train with small LR | Medium-to-large target dataset |
| Full pre-training | Train from scratch on target domain | Huge target dataset, very different domain |

**In plain English:** Fine-tuning is like hiring a trained surgeon and teaching them a new procedure — far easier than training a medical student from scratch.

---

## Generative Models — Autoencoders, VAEs, and GANs

### Autoencoders

Compress input to a latent code, then reconstruct it.

```
Input x ──► [Encoder] ──► z (latent) ──► [Decoder] ──► x̂
                           (bottleneck)
```

Trained to minimize reconstruction loss. Not generative — sampling random z often yields garbage.

### Variational Autoencoders (VAEs)

VAEs impose a probability distribution (typically Gaussian) on the latent space, enabling generation.

```
Input x ──► [Encoder] ──► (μ, σ) ──► z = μ + σ·ε ──► [Decoder] ──► x̂
                           ↑ reparameterization trick
                           ε ~ N(0,1)
```

**ELBO (Evidence Lower BOund) loss:**
`ELBO = E[log p(x|z)] − KL(q(z|x) ∥ p(z))`

- First term: reconstruction quality.
- Second term: forces posterior to stay close to prior N(0,1).

**Reparameterization trick:** Sampling `z ~ N(μ, σ²)` is not differentiable. Rewrite as `z = μ + σ·ε` where `ε ~ N(0,1)`. Now gradients flow through `μ` and `σ` while `ε` is just a fixed random sample.

**Posterior collapse:** When the decoder is too powerful (e.g., an LSTM), it can reconstruct x well using only local context, making the encoder redundant — the posterior q(z|x) collapses to the prior p(z) and latent codes become uninformative. Fixes: KL annealing, β-VAE, weaker decoder.

**Why VAEs matter for interviews:** Synthetic data generation, anomaly detection, and drug discovery pipelines all use VAEs. Interviewers frequently ask about the reparameterization trick and posterior collapse in ML engineering/research roles.

### GANs (Generative Adversarial Networks)

```
Noise z ──► [Generator G] ──► fake x̂
                                  │
Real x ───────────────────────► [Discriminator D] ──► real / fake?
```

G and D play a minimax game:

```
min_G max_D  E[log D(x)] + E[log(1 − D(G(z)))]
```

| Model | Pros | Cons |
|-------|------|------|
| VAE | Stable training, explicit latent space | Blurry outputs |
| GAN | Sharp, high-quality outputs | Unstable (mode collapse, vanishing gradients) |

**Mode collapse:** Generator learns to produce only a few modes of the data distribution, ignoring diversity.

---

## Hyperparameters vs Parameters

| Type | Examples | How Set |
|------|---------|--------|
| Parameters | Weights W, biases b | Learned via gradient descent |
| Hyperparameters | Learning rate, batch size, #layers, dropout rate | Set by the practitioner; tuned via search |

**In plain English:** Parameters are what the model *learns*; hyperparameters are the settings you *choose* before training begins.

---

## Why Deep Learning Needs Lots of Data and Compute

- Deep networks have millions to billions of parameters — they need enough data to avoid memorizing training examples.
- Expressive models can represent arbitrarily complex functions but require broad data coverage to generalize.
- GPUs exploit massive parallelism for matrix operations (the core of forward/backward passes).
- Larger models empirically benefit from more data (scaling laws: loss ∝ data/compute^{-α}).

**In plain English:** More data reduces the risk that the model "cheats" by memorizing. More compute enables larger models that capture more complex patterns.

---

## Interview Questions

### Q: Explain backpropagation and why it uses the chain rule.
**Strong answer:**
Backpropagation computes the gradient of the loss with respect to every weight by applying the chain rule of calculus backwards through the computational graph. At each layer, we need the gradient of the loss with respect to that layer's pre-activation output, which depends on gradients from all subsequent layers. The chain rule lets us decompose this into a product of local gradients, each of which is easy to compute. This is what makes training deep networks tractable — you only need to store and multiply local Jacobians.

---

### Q: Why do we use ReLU instead of sigmoid for hidden layers?
**Strong answer:**
Sigmoid saturates for large positive or negative inputs, producing near-zero gradients that prevent early layers from learning (vanishing gradient problem). ReLU has a gradient of exactly 1 for positive inputs, so gradients flow back unchanged in the positive region. ReLU is also computationally cheap (a simple threshold). The main downside is "dying ReLU" — neurons that receive a negative input and produce zero gradient forever — which Leaky ReLU and GELU mitigate.

---

### Q: What is the difference between batch normalization and layer normalization? When would you use each?
**Strong answer:**
Batch norm normalizes each feature across the batch dimension, so its statistics depend on batch size and can be unstable with small batches. Layer norm normalizes each sample across the feature dimension, making it batch-size independent and behaving identically at train and inference time. Use batch norm for CNNs where large batch sizes are typical and features share spatial meaning. Use layer norm for Transformers and RNNs where batch sizes may be small and sequence positions vary.

---

### Q: What is the vanishing gradient problem and how do residual connections fix it?
**Strong answer:**
In deep networks with sigmoid/tanh activations, gradients are repeatedly multiplied by numbers less than 1 as they propagate backwards, causing them to shrink exponentially. Early layers receive near-zero gradients and fail to learn. Residual connections add a skip path (`output = F(x) + x`) so that the gradient during backpropagation always has a direct additive path — the gradient of `x` is always at least 1, regardless of how small `∂F/∂x` becomes. This is why ResNets can be trained with hundreds of layers.

---

### Q: How does Adam differ from SGD with momentum?
**Strong answer:**
SGD with momentum adds a velocity term that accumulates past gradients, smoothing noisy updates. Adam adds an adaptive per-parameter learning rate: it tracks the second moment (running average of squared gradients) and divides the update by its square root. This means parameters with historically large gradients get smaller updates and vice versa. In practice, Adam converges faster and is less sensitive to learning rate choice, but SGD with momentum can generalize better on image classification tasks when tuned carefully.

---

### Q: What is Xavier initialization and when should you use He initialization instead?
**Strong answer:**
Xavier (Glorot) initialization sets weights with std = `√(2 / (fan_in + fan_out))`, designed to preserve variance through tanh/sigmoid activations. He (Kaiming) initialization uses std = `√(2 / fan_in)`, which compensates for ReLU zeroing out roughly half its inputs. Use Xavier with tanh or sigmoid; use He with ReLU or Leaky ReLU. For GELU (used in Transformers), He initialization also works well in practice because the variance reduction is similar in magnitude to ReLU.

---

### Q: Explain the reparameterization trick in VAEs and why it is necessary.
**Strong answer:**
A VAE encoder outputs parameters `(μ, σ)` of a Gaussian distribution, and we need to sample `z ~ N(μ, σ²)` before passing it to the decoder. The problem is that sampling is a stochastic operation — gradients cannot flow through a random node during backpropagation. The reparameterization trick rewrites `z = μ + σ·ε` where `ε ~ N(0, 1)` is sampled independently. Now the stochasticity is in `ε` (which we do not differentiate through) and the learnable parameters `μ` and `σ` lie on a deterministic path, so gradients flow normally.

---

### Q: What is posterior collapse in VAEs and how do you fix it?
**Strong answer:**
Posterior collapse occurs when the VAE decoder becomes powerful enough to reconstruct input without using the latent code — it relies on local context (e.g., from an autoregressive decoder) instead. The KL term in the ELBO then drives the posterior to match the prior exactly, making `z` uninformative. Fixes include: KL annealing (gradually increase the KL weight from 0 during training), using a weaker decoder, free bits (ignoring KL below a threshold per dimension), or the β-VAE formulation that down-weights KL relative to reconstruction.

---

### Q: What is the bottleneck problem in encoder-decoder RNNs and how did it motivate attention?
**Strong answer:**
Classic seq2seq models compress the entire input sequence into a single fixed-length vector before the decoder begins generating. For long sequences, this vector cannot retain all relevant information, causing quality degradation. Attention mechanisms solve this by allowing the decoder to compute a weighted sum over all encoder hidden states at each decoding step, effectively giving it direct access to any part of the input. This idea of "attend to relevant positions" was generalized to self-attention in the Transformer, which eliminated recurrence entirely.

---

### Q: What is dropout and how does it act as regularization?
**Strong answer:**
Dropout randomly zeros out a fraction `p` of neurons during each training forward pass. This prevents co-adaptation — neurons cannot rely on specific partners being present — forcing the network to learn redundant representations. Conceptually, it trains an ensemble of exponentially many sub-networks, and at inference the full network approximates averaging over that ensemble. Typical dropout rates are 0.1–0.5 for hidden layers. It is ineffective in convolutional layers (use spatial dropout instead) and is disabled at inference.

---

### Q: Why does a large batch size sometimes hurt generalization?
**Strong answer:**
Large batches produce smoother, more accurate gradient estimates, which causes the optimizer to converge to sharp minima in the loss landscape. Sharp minima are associated with poor generalization because small perturbations in weights cause large increases in loss. Small batches introduce gradient noise that effectively acts as regularization, helping the optimizer find flatter minima that generalize better. The "linear scaling rule" (scale LR proportionally with batch size) partially compensates but does not fully recover generalization.

---

### Q: What is mode collapse in GANs and why does it happen?
**Strong answer:**
Mode collapse is when the GAN generator learns to produce only a small subset of the data distribution — often a single mode — because that output reliably fools the discriminator. It happens because the discriminator cannot "reward" the generator for diversity, only for realism. Once the generator finds one successful output, gradient signals do not push it to explore. Solutions include Wasserstein GAN (which provides more stable gradients), minibatch discrimination (discriminator sees batches not individual samples), and spectral normalization.

---

### Q: What distinguishes a hyperparameter from a model parameter, and give examples of each?
**Strong answer:**
Model parameters (weights, biases) are learned automatically via gradient descent during training. Hyperparameters are configuration choices set by the practitioner before training — they are not updated by the optimizer. Examples of hyperparameters: learning rate, batch size, number of layers, number of hidden units, dropout rate, optimizer choice, regularization strength, and number of epochs. Hyperparameter tuning uses separate validation data and methods like grid search, random search, or Bayesian optimization.

---

### Q: Explain the receptive field in CNNs and why deeper networks see more of the image.
**Strong answer:**
The receptive field of a neuron is the region of the original input image that influences its activation. A single convolutional layer with a 3×3 kernel gives each neuron a 3×3 receptive field. After a second 3×3 conv, each neuron's receptive field grows to 5×5. After pooling (which halves spatial dimensions), the effective receptive field grows even faster. Stacking layers lets neurons in deep layers integrate information from increasingly large image regions, enabling detection of progressively larger and more abstract features.

---

## References

- [Top Deep Learning Interview Questions (2026) — DataInterview](https://www.datainterview.com/blog/deep-learning-interview-questions)
- [Batch Normalization vs Layer Normalization — Outcome School](https://outcomeschool.com/blog/batch-normalization-vs-layer-normalization)
- [ML Interview Q Series: What are Variational Autoencoders? — Rohan Paul](https://www.rohan-paul.com/p/ml-interview-q-series-what-are-variational)
- [Posterior Collapse as a Phase Transition in VAEs — arXiv 2510.01621](https://arxiv.org/pdf/2510.01621)
- [He and Xavier Initialization Functions — The Deep Hub (Medium)](https://medium.com/thedeephub/he-and-xavier-weight-initialization-functions-acedc5322ce5)
- [RNN, LSTM, GRU in NLP: A Deep Dive — Medium](https://medium.com/@amitkharche/rnn-lstm-gru-in-nlp-a-deep-dive-into-sequence-modeling-c241c9b8e713)

---

*Previous: [Classical ML Algorithms](01-classical-ml-algorithms.md) | Next: [Statistics and Probability](03-statistics-and-probability.md)*
