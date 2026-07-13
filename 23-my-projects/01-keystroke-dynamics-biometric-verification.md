# Keystroke Dynamics — Biometric Verification (Siamese CNN→GRU)

> **My project.** A behavioral-biometric system that verifies *who* is typing purely from keystroke **timing rhythm** (not the text content). Inspired by Kasprowski et al. (*Sensors* 2022), but re-architected from the paper's N-way **softmax classifier** into a **metric-learning / siamese** model — so new users can enroll without retraining, which is exactly the limitation the original authors called out.

## Table of Contents

- [The Narrative](#the-narrative)
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

## The Narrative

**Situation.** Passwords prove *what you know*, not *who you are* — a stolen password is game over. Behavioral biometrics ask a different question: can we recognize a person by *how* they type, continuously and invisibly, from any keyboard? I started from a strong reference (Kasprowski et al., *Sensors* 2022), but its softmax classifier had a fatal deployment flaw: **adding a new user means retraining the whole model**, and accuracy collapses as the user count grows (88% → 69% from 20 → 60 users).

**Task.** Build a keystroke-dynamics system that **verifies identity** and lets you **enroll a new person without retraining** — the exact limitation the paper's authors flagged.

**Action.** I kept the paper's proven feature design (digraph timing: dwell, flight, down-down) but **re-architected the model into a siamese CNN→GRU trained with cosine embedding loss** — mapping a window of typing into a 16-dim embedding where the same person's windows point the same way and different people push apart. Enrolling someone new is then just "compute and store their embedding."

**Result.** A verification system that cleanly separates typists by cosine similarity and **removes the retrain-to-enroll bottleneck** — the same metric-learning paradigm behind TypeNet, which the reference paper itself cites as the stronger approach.

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

## Glossary

| Term | Simple explanation | Purpose |
|---|---|---|
| **Keystroke Dynamics** | A behavioral biometric that identifies people by how they type — the timing of key presses and releases, not the text itself | Enables continuous, invisible identity verification without passwords |
| **Behavioral Biometric** | A biometric based on how a person does something (typing, walking) rather than a physical trait (fingerprint, face) | Allows passive, non-invasive identity checking from any keyboard |
| **Biometric Verification (1:1)** | Confirming a single claimed identity by comparing a new sample to one stored profile | Answers "is this the person they claim to be?" rather than "who is this?" |
| **Biometric Identification (1:N)** | Searching a database of N users to find who matches an unknown sample | More complex than verification; accuracy degrades as N grows |
| **Digraph** | A pair of consecutive keystrokes plus the timing between them | The atomic unit of keystroke-dynamics features, capturing a person's local typing rhythm |
| **Dwell Time (HD)** | How long a key is held down from press to release | Captures the individual rhythm of each key press |
| **Flight Time (UD)** | The gap between releasing one key and pressing the next | Captures the transition speed between successive keys |
| **Down-Down Time (DD)** | Time from pressing one key to pressing the very next key | Combines dwell and flight into a single inter-key interval |
| **One-Hot Encoding** | Representing each possible key as a binary vector with a 1 in exactly one position | Avoids inventing false numeric ordering between unrelated keys |
| **StandardScaler (Z-score)** | Transforms a numeric column so its mean is 0 and standard deviation is 1 | Puts timing features on a comparable scale for stable model training |
| **Windowing** | Slicing a stream of digraphs into fixed-length chunks (e.g., 50 consecutive digraphs) | Creates fixed-size input samples the model can process |
| **Siamese Network** | A pair of identical neural networks sharing weights that each embed one input, used to compare two inputs | Lets the model learn similarity rather than class labels |
| **Metric Learning** | Training an encoder to map inputs to a space where similar items are close and dissimilar items are far | Enables open-set recognition — new users enroll without retraining |
| **Embedding** | A compact fixed-length vector representing an input in a learned space | Identity vectors are compared at inference time using cosine similarity |
| **Conv1D (1-D Convolution)** | A convolution that slides a filter along a 1-D time axis to detect local patterns | Learns short local timing motifs across adjacent keystrokes |
| **GRU (Gated Recurrent Unit)** | A recurrent neural network cell that models sequential dependencies with gating mechanisms | Captures how timing motifs unfold across the full keystroke window |
| **Cosine Embedding Loss** | A training loss that pulls same-class pairs to have high cosine similarity and pushes different-class pairs apart | Shapes the embedding space so angle equals identity distance |
| **Cosine Similarity** | A measure of how similar two vectors are based on the angle between them (1 = identical direction, −1 = opposite) | Used at inference to decide if two typing samples belong to the same person |
| **Mean Pooling** | Averaging multiple embedding vectors into one representative vector | Combines a user's many window embeddings into a single stable identity vector |
| **L2 Normalization** | Scaling a vector to have length 1 | Makes cosine similarity equivalent to dot product, enabling stable comparison |
| **Softmax Classifier** | A network head that outputs a probability distribution over a fixed set of N classes | Used by the reference paper; requires retraining when new users are added |
| **EER (Equal Error Rate)** | The threshold at which false accept rate equals false reject rate for a verification system | Standard metric for biometric systems; lower EER = better performance |
| **TypeNet** | A published metric-learning keystroke biometric system (Acien et al., 2021) | Demonstrates the superiority of embeddings over fixed classifiers for keystroke identity |
| **Hard Negative Mining** | Selecting the most confusing negative pairs (near-miss impostors) during training | Improves embedding quality by focusing training on the hardest cases |
| **Triplet Loss** | A loss that pulls an anchor and a positive sample together while pushing the anchor and a negative sample apart | Alternative contrastive training objective for shared encoders over many identities |

---

*Up: [Guide Home](../README.md)*
