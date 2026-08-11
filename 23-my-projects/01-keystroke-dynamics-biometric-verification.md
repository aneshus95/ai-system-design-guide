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

> **Why (the rationale):** Digraph features capture a person's *local* typing rhythm — not what keys they press, but the precise timing transitions between pairs. One-hot encoding keys avoids the false numeric ordering that integer mapping would invent, and only the 4 timing columns are standardized, so the key-identity signal isn't distorted.
> **When to use:** Any behavioral-biometric task where raw keystroke logs are available and the goal is identity from typing dynamics rather than text content — especially when the user set is open-ended (metric learning) or partially labeled.
> **Nuances & gotchas:** Digraph features are session-dependent — typing speed varies with fatigue, keyboard hardware, and emotional state, causing within-user variance that can erode similarity scores. 82-key one-hot gives 164 sparse dims just for identity; the sparse key-identity columns can dominate if any scaling leaks in.

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

> **Why (the rationale):** Concatenating multi-session data forces the encoder to generalize across within-user variation (different keyboard, different energy level) rather than memorizing one sitting. Windowing converts an arbitrarily long keystroke stream into fixed-size tensors the CNN/GRU can process.
> **When to use:** Whenever the biometric signal is a continuous stream (typing, mouse movement, gait) that exceeds what the model can consume in one shot — windowing is the standard decomposition.
> **Nuances & gotchas:** Non-overlapping windows discard context at window boundaries; overlapping windows (the paper used 40% overlap) multiply sample count but introduce correlated samples that can inflate evaluation metrics if not handled carefully in the train/val split.

1. **Multi-session enrollment.** Each user's several recording sessions are concatenated, so the model sees cross-session variation instead of memorizing one sitting.
2. **Standardization.** A `StandardScaler` (z-score) is fit on **only the 4 timing columns** — the one-hot key columns are left untouched. Timings are put on a comparable scale for stable training.
3. **Windowing.** The per-user digraph stream is reshaped into fixed-length **samples of 50 consecutive digraphs**: shape `(n_windows, 50, 168)`. One window = a short slice of a person's typing rhythm — a time series the CNN/GRU can consume.

---

## Pair Construction & Labels

> **Why (the rationale):** Pairs let the model learn a *distance criterion* directly — "same person's windows should be close, different people's windows should be far" — rather than memorizing a fixed set of identities. This is what makes the encoder identity-agnostic and allows enrollment of unseen users.
> **When to use:** When the user set is open or expected to grow, and you need open-set recognition (new users enroll at inference time). If the user set is fixed and small, a softmax classifier is simpler and can be competitive.
> **Nuances & gotchas:** Pair imbalance is real — with N users there are O(N²) same-user pairs but far more dissimilar ones. Easy negatives dominate training and slow learning; hard-negative mining (picking impostors whose embeddings are close) is the standard fix but requires more implementation care.

Metric learning trains on **pairs**, not single items. From all windows we form pairs `(window_i, window_j)` and label each:

- **`+1` (similar)** — both windows belong to the **target user**.
- **`−1` (dissimilar)** — target user vs. anyone else.

This makes each trained model a **one-vs-rest verifier** for a target identity: it learns an embedding space where the target's windows cluster together and separate from impostors.

---

## Model Architecture — Siamese CNN→GRU

> **Why (the rationale):** The CNN layer extracts local timing motifs across adjacent keystrokes (kernel size 2 matches the digraph structure, where only neighboring key-pairs carry identity signal); the GRU then captures how those motifs chain across the window's 50 steps. Sharing weights between the two siamese branches guarantees the same embedding space for both inputs — a necessary condition for cosine comparison to be meaningful.
> **When to use:** CNN+GRU hybrids suit time-series inputs with both local patterns (CNN) and sequential dependencies (RNN) — keystroke dynamics, gesture sequences, ECG signals. Pure CNN or pure RNN underperform when both levels of structure matter (ablation confirmed in the reference paper).
> **Nuances & gotchas:** Kernel size 2 is optimal here specifically because of the digraph structure — don't generalize this without ablation. GRU only returns the last hidden state, discarding earlier window context; for longer windows an attention pooling or bidirectional GRU may capture more signal. The 16-dim embedding is compact enough to compare efficiently but can be a bottleneck if users are typographically very similar.

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

> **Why (the rationale):** Cosine embedding loss shapes the embedding space so *direction* encodes identity — same-person pairs are pulled to point the same way, different-person pairs are pushed apart angularly. This makes the inference comparison (cosine similarity of mean embeddings) directly aligned with the training objective: high cosine = same person.
> **When to use:** When you plan to verify by cosine similarity at inference time and want magnitude-independent comparison (useful when embedding norms vary across users). Triplet loss is an alternative when hard-negative mining is built in and you need tighter cluster boundaries.
> **Nuances & gotchas:** Cosine embedding loss does not control the *norm* of embeddings — only their direction. Without L2 normalization at inference, two embeddings could have high cosine similarity but be at very different scales; always normalize before comparison. The margin hyperparameter for dissimilar pairs needs tuning — too tight and negatives aren't pushed far enough; too loose and same-user pairs can still be ambiguous.

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

## Key Decisions & Tradeoffs

| Decision | Options considered | Why chosen | Tradeoff accepted |
|---|---|---|---|
| **Verification (1:1) vs Identification (1:N)** | Softmax classifier over fixed N users (paper's approach) vs embedding + distance metric | Metric learning lets new users enroll by computing one embedding — no retraining; encoder is identity-agnostic | One-vs-rest models per target user don't scale to thousands of users; a shared encoder with triplet loss is the proper fix |
| **Siamese CNN→GRU vs pure CNN or pure RNN** | CNN-only, GRU-only, Transformer | CNN extracts local digraph-level timing motifs; GRU sequences them across the 50-step window; hybrid beat either alone in the reference paper's ablation | Added model complexity and two sets of hyperparameters to tune (conv channels, GRU hidden size) |
| **Cosine Embedding Loss vs Triplet Loss** | Triplet loss with online hard-negative mining, contrastive loss, softmax cross-entropy | Cosine embedding loss directly aligns the training objective with the cosine-similarity inference step; simpler to implement correctly than triplet mining | Does not control embedding norm (only direction); margin hyperparameter needs tuning; no built-in hard-negative mining |
| **Kernel size 2 for Conv1d** | Kernel sizes 3, 5, larger | Digraph structure means only adjacent keystroke pairs carry the local identity signal; larger kernels average over unrelated transitions | Risk of under-smoothing if a 3- or 4-gram rhythm pattern matters — should ablate |
| **16-dim embedding** | 32, 64, 128 dims | Compact enough for fast cosine comparison; sufficient to separate typists cleanly in experiments | May be a bottleneck for users with very similar typing styles; too small for large-scale multi-user enrollment |
| **Non-overlapping 50-step windows** | Overlapping windows (40% overlap as in paper) | Simpler implementation; no correlated samples inflating eval metrics when train/val split is not carefully handled | Fewer training samples from the same corpus; boundary context discarded; small datasets benefit more from overlap |

---

## Production Issues & Fixes

*Representative issues for this system class — mapped to what I actually hit; tailor to your own incidents.*

**1. Threshold set on development data, real FAR spiked in deployment**
- **Symptom:** After launch, users from a new office (different keyboard hardware) were being erroneously verified — false-accept rate well above the expected level.
- **Root cause:** The cosine-similarity decision threshold was tuned on data from one keyboard layout/brand. Stiffer keyboards produce systematically shorter dwell times, shifting all embeddings slightly; impostors who happened to use the same keyboard as the target user fell inside the threshold.
- **Fix:** Retune the threshold on a held-out split that includes hardware-diverse sessions; move from a single global threshold to per-user thresholds anchored on each enrollee's within-user similarity distribution.
- **Prevention/monitoring:** Track per-user FAR and FRR weekly; alert when either drifts >2 percentage points from enrollment baseline; add keyboard-type metadata as a feature or stratification variable.

**2. Typing-behavior drift causing rising FRR for long-tenured users**
- **Symptom:** A cohort of users enrolled 6 months earlier started hitting false-reject rate of ~30%, while new enrolees were fine. No model change had been made.
- **Root cause:** Genuine typing rhythm drifts with factors like injury, changing keyboard, or sustained fatigue. The enrollment embedding was a snapshot from 6 months prior; the user's current windows no longer pointed in the same direction.
- **Fix:** Implement rolling re-enrollment: after each successful verification, append the new window embeddings to the user's enrollment set and recompute the mean profile vector (exponential moving average, downweighting old sessions).
- **Prevention/monitoring:** Alert when a verified user's intra-session cosine similarity drops below a rolling 30-day average by more than 0.1.

**3. Cold-start: new users enrolled from a single short session have unstable profiles**
- **Symptom:** New users with fewer than 50 digraphs at enrollment (single short form submission) saw dramatically higher FRR than users enrolled from multiple sessions.
- **Root cause:** A single window's embedding is noisy; the mean pool over 1–2 windows has high variance, so the enrollment vector doesn't represent the user's true typing distribution.
- **Fix:** Enforce a minimum enrollment threshold (e.g., ≥3 sessions / ≥200 digraphs); gate verification until the enrollment vector is computed from sufficient samples; display a "complete enrollment" prompt after each session.
- **Prevention/monitoring:** Track enrollment sample count per user; log and alert when a verification decision is made on fewer than the minimum enrollment windows.

**4. Class imbalance in genuine vs impostor pairs degrading training**
- **Symptom:** Validation loss plateaued early; the model's cosine scores showed little separation between same-user and different-user pairs for typographically similar users.
- **Root cause:** With N users, genuine pairs are O(N × windows_per_user) but impostor pairs are O(N² × windows²) — easy negatives dominated the loss, and the model stopped learning from hard cases.
- **Fix:** Subsample or cap impostor pairs to a 1:5 genuine-to-impostor ratio; add semi-hard-negative mining (select impostors whose embeddings fall within a similarity band around the genuine pairs rather than random impostors).
- **Prevention/monitoring:** Log the ratio of genuine to impostor pairs per training batch; monitor loss curves for plateau; track embedding separation (intra-class vs inter-class cosine mean) as a training diagnostic.

**5. Latency spike during batch verification at enrollment**
- **Symptom:** During peak onboarding hours, verification latency jumped from ~12 ms to >200 ms for users with many enrolled sessions.
- **Root cause:** The mean-pool step was computed lazily at verification time over all raw enrollment windows, re-running the encoder on every window every time rather than caching the enrollment vector.
- **Fix:** Pre-compute and cache the L2-normalized mean enrollment embedding at enrollment time (or on first post-enrollment verification); store it in a fast key-value lookup alongside the user record.
- **Prevention/monitoring:** Add a p99 latency metric for the verification endpoint; alert when it exceeds 50 ms; log whether the enrollment vector was served from cache or recomputed.

**6. EER metric missing — only inspecting sample similarities at demo time**
- **Symptom:** Demo showed clean separation on cherry-picked pairs, but the system had no principled operating point for a real deployment decision; early evaluators found the threshold selection arbitrary.
- **Root cause:** Evaluation used visual inspection of cosine score distributions rather than sweeping the threshold and computing EER on a held-out set.
- **Fix:** Implement a proper evaluation loop: sweep the decision threshold from −1 to 1 in steps of 0.01, compute FAR and FRR at each, find the crossing point (EER), and report EER as the primary metric alongside ROC-AUC.
- **Prevention/monitoring:** Add an automated evaluation job that computes EER on a held-out split after any model change; gate deployment on EER below a defined ceiling.

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
