# Keystroke Dynamics — Biometric Verification (Siamese CNN→GRU)

> **My project.** A behavioral-biometric system that verifies *who* is typing purely from keystroke **timing rhythm** (not the text content). Inspired by Kasprowski et al. (*Sensors* 2022), but re-architected from the paper's N-way **softmax classifier** into a **metric-learning / siamese** model — so new users can enroll without retraining, which is exactly the limitation the original authors called out.

## Table of Contents

- [Problem & Design Choice](#problem--design-choice)
- [End-to-End Pipeline](#end-to-end-pipeline)
- [Feature Engineering — the Digraph Vector](#feature-engineering--the-digraph-vector)
- [Preprocessing & Windowing](#preprocessing--windowing)
- [Pair Construction & Labels](#pair-construction--labels)
- [Model Architecture — Siamese CNN→GRU](#model-architecture--siamese-cnngru)
- [Training — Cosine Embedding Loss](#training--cosine-embedding-loss)
- [Inference — Verification by Similarity](#inference--verification-by-similarity)
- [Why Metric Learning Over the Paper's Classifier](#why-metric-learning-over-the-papers-classifier)
- [Results & Honest Limitations](#results--honest-limitations)
- [Interview Talking Points](#interview-talking-points)
- [References](#references)

---

## Problem & Design Choice

**Keystroke dynamics** is a *behavioral* biometric: not *what* you type, but *how* — your personal rhythm of pressing and releasing keys. Everyone has a typing "signature," and it can be measured continuously and non-invasively from any keyboard.

Two ways to frame the task:

| | Identification (1:N) — *the paper* | **Verification (1:1) — this project** |
|---|---|---|
| Question | "Who, among N enrolled users, is this?" | "Is this person who they claim to be?" |
| Head | Softmax over N classes | **Embedding + distance metric** |
| New user | **Retrain** the classifier | **Just compute their embedding** (no retrain) |
| Scales to many users | Degrades (paper: 88% → 69%, 20 → 60 users) | Encoder is identity-agnostic |

I chose **verification via metric learning** because it enrolls new users without retraining and generalizes past a fixed user set.

---

## End-to-End Pipeline

```
 raw keystroke logs (per user, multiple sessions)
        │  parse: Key, Press Time, Release Time
        ▼
 FEATURE ENGINEERING  → digraph vectors  [L1, L2, HD1, HD2, UD, DD]  (168 dims)
        │
        ▼
 PREPROCESS  → standardize timing cols → window into (50 timesteps × 168 feats)
        │
        ▼
 PAIRING     → build (window_i, window_j) pairs, label same/different person
        │
        ▼
 SIAMESE CNN→GRU  → 16-dim embedding per window (shared weights)
        │  CosineEmbeddingLoss: pull same-person together, push apart otherwise
        ▼
 VERIFY      → cosine similarity of two people's mean embeddings → same / different
```

---

## Feature Engineering — the Digraph Vector

The atomic unit is a **digraph**: two consecutive keys plus the timing relationship between them. For each consecutive key pair `(i, i+1)`:

```
   D        U     D        U        D = key down (press)
   │  Key1  │     │  Key2  │        U = key up   (release)
   ▼        ▼     ▼        ▼
   ├──HD1───┤     ├──HD2───┤        HD = hold/dwell time (Release − Press)
   ├────DD──────────┤              DD = press-to-next-press
            ├──UD──┤                UD = release-prev → press-next (flight gap)
```

Each digraph becomes a **168-dim vector**:

```
 ┌──────────────────┬──────────────────┬─────┬─────┬────┬────┐
 │ L1  one-hot(82)  │ L2  one-hot(82)  │ HD1 │ HD2 │ UD │ DD │
 └──────────────────┴──────────────────┴─────┴─────┴────┴────┘
   which key first    which key second   two dwell    gap  down-
   (82 possible keys) (82 possible keys)  times            down
```

- **Keys are one-hot encoded** (82 possible keys → 82 dims each), not integer-mapped — integers would invent a false ordering between unrelated keys.
- **Timings are in seconds** (release/press deltas).

`82 + 82 + 4 = 168` features per digraph.

---

## Preprocessing & Windowing

1. **Multi-session enrollment.** Each user's several recording sessions are concatenated, so the model sees cross-session variation instead of memorizing one sitting.
2. **Standardization.** A `StandardScaler` (z-score) is fit on **only the 4 timing columns** — the one-hot key columns are left untouched. Timings are put on a comparable scale for stable training.
3. **Windowing.** The per-user digraph stream is reshaped into fixed-length **samples of 50 consecutive digraphs**: shape `(n_windows, 50, 168)`. One window = a short slice of a person's typing rhythm — a time series the CNN/GRU can consume.

---

## Pair Construction & Labels

Metric learning trains on **pairs**, not single items. From all windows we form pairs `(window_i, window_j)` and label each:

- **`+1` (similar)** — both windows belong to the **target user**.
- **`−1` (dissimilar)** — target user vs. anyone else.

This makes each trained model a **one-vs-rest verifier** for a target identity: it learns an embedding space where the target's windows cluster together and separate from impostors.

---

## Model Architecture — Siamese CNN→GRU

Two windows go through **the same network with shared weights** (that's what "siamese" means) and each produces a 16-dim embedding.

```
        inp1 (50×168)                 inp2 (50×168)
             │                             │
             ▼            shared weights    ▼
   ┌───────────────────────────────────────────────┐
   │  transpose → (features on channel axis)        │
   │  Conv1d(168 → 64, kernel=2, ReLU)              │  ← local timing motifs
   │  Conv1d(64  → 64, kernel=2, ReLU)              │    across adjacent keys
   │  transpose back → (time, channels)             │
   │  GRU(64 → 64, 1 layer) → last hidden state     │  ← sequence dependency
   │  Linear(64 → 16)                               │  ← embedding head
   └───────────────────────────────────────────────┘
             │                             │
             ▼                             ▼
          emb1 (16-d)                   emb2 (16-d)
```

**Design notes**
- **1-D convolutions slide over the time axis**, learning short local patterns (the paper found only *neighboring* keystrokes carry identity signal, so **kernel size = 2** is the right inductive bias — larger kernels hurt).
- **GRU** captures the sequential dependency across the 50-step window; the **last hidden state** summarizes the window.
- **`Linear(64 → 16)`** is the embedding head — the compact identity representation compared at inference.
- CNN + RNN **hybrid** beats either alone (paper's Experiment 2): CNN extracts local timing motifs, RNN orders them.

| Hyperparameter | Value |
|---|---|
| Window length (timesteps) | 50 |
| Input features | 168 |
| Conv channels / kernel | 64 / 2 |
| GRU hidden / layers | 64 / 1 |
| Embedding dim | 16 |
| Optimizer / LR | Adam / 3e-4 |
| Loss | Cosine Embedding Loss |

---

## Training — Cosine Embedding Loss

```python
criterion = torch.nn.CosineEmbeddingLoss()
# label = +1  → maximize cosine similarity(emb1, emb2)   (same person)
# label = -1  → push cosine similarity below a margin     (different person)
loss = criterion(emb1, emb2, label)
```

The loss is **contrastive on direction**: same-person embeddings are pulled to point the same way; different-person embeddings are pushed apart. This shapes an embedding space where **angle = identity distance** — exactly what cosine-similarity verification needs at inference.

---

## Inference — Verification by Similarity

```
 person A windows ─► encoder ─► mean-pool ─► L2-normalize ─┐
                                                            ├─► cosine similarity ─► same / different
 person B windows ─► encoder ─► mean-pool ─► L2-normalize ─┘
```

Each person's windows are embedded and **mean-pooled** into one identity vector, L2-normalized, then compared by cosine similarity. High positive similarity ⇒ same person; strongly negative ⇒ different person. In testing, different-person comparisons produced clearly negative similarities, cleanly separating identities.

---

## Why Metric Learning Over the Paper's Classifier

The paper (Kasprowski et al.) uses a **softmax classifier** over a fixed set of N users. Two consequences:

1. **Adding a user requires retraining** the whole classification head.
2. **Accuracy degrades as N grows** (Rank-1: 88% at 20 users → 69% at 60).

This project uses a **learned embedding + distance metric** instead — the same paradigm as **TypeNet** (Acien et al., 2021), which the paper itself cites as the stronger approach. Enrolling a new user is just "compute and store their embedding," and the encoder never needs to know the identity set in advance.

---

## Results & Honest Limitations

**What worked:** the siamese encoder cleanly separates different typists by cosine similarity; the digraph + windowing pipeline reproduces the paper's feature design; the metric-learning reframing removes the retrain-to-enroll bottleneck.

**Limitations I'd address next (engineering maturity matters in interviews):**
- **Report EER (Equal Error Rate)** on a held-out split — the right metric for a verification system — instead of only inspecting sample similarities.
- **Overlapping windows** (the paper used 40% overlap) to expand a small dataset.
- **One shared encoder trained on many identities** with triplet loss + hard-negative mining, rather than per-target one-vs-rest models — the proper way to scale enrollment.
- **Regularization actually wired in** (dropout applied in the forward pass) and consistent train/inference tensor shapes.

---

## Interview Talking Points

**Elevator pitch:**
> *"I built keystroke-dynamics identity verification. I engineered digraph timing features — dwell, flight, and down-to-down times — with one-hot keys, standardized and windowed them into 50-step sequences, then trained a siamese CNN→GRU with cosine embedding loss to produce 16-dim embeddings so the same person's typing pulls together and different people push apart. At inference I verify by cosine similarity of mean embeddings. I deliberately chose metric learning over the reference paper's softmax classifier because it lets you enroll new users without retraining — the exact limitation the original authors flagged."*

**If asked "why a CNN *and* a GRU?"** — The CNN learns local timing motifs across adjacent keystrokes (and kernel size 2 is optimal because only neighboring keys carry identity signal); the GRU models how those motifs unfold in sequence. The hybrid beat either component alone in the paper's ablation.

**If asked "why cosine embedding loss?"** — I wanted an angular embedding space so identity distance is a cosine, which is cheap and stable to compare at enrollment/verification time.

**If asked "what would you improve?"** — Report EER on a held-out set, add overlapping windows, apply dropout in the forward pass, and move from per-user one-vs-rest models to a single shared encoder trained with triplet loss and hard-negative mining.

---

## References

- [Kasprowski, P.; Borowska, Z.; Harezlak, K. *Biometric Identification Based on Keystroke Dynamics*. Sensors 2022, 22, 3158](https://doi.org/10.3390/s22093158) — the reference paper (code: [github.com/kasprowski/keystroke2022](https://github.com/kasprowski/keystroke2022))
- [Acien, A. et al. *TypeNet: Deep Learning Keystroke Biometrics*. IEEE TBIOM, 2021](https://arxiv.org/abs/2101.05570) — metric-learning keystroke embeddings
- [PyTorch `CosineEmbeddingLoss` docs](https://pytorch.org/docs/stable/generated/torch.nn.CosineEmbeddingLoss.html)
- [Siamese networks & metric learning — pytorch-metric-learning](https://kevinmusgrave.github.io/pytorch-metric-learning/)

---

*Up: [Guide Home](../README.md)*
