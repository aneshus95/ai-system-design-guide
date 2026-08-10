# Synthetic Data Generation

The industry has hit the "Data Wall", the exhaustion of high-quality human text on the web. Synthetic data is now the primary engine for model improvement, sitting at the core of every modern frontier-model recipe.

## Table of Contents

- [The Data Wall and the Synthetic Shift](#synthetic-shift)
- [Evol-Instruct Pattern](#evol-instruct)
- [Constitutional AI & Self-Correction](#constitutional-ai)
- [Verifiable Synthetic Data (Math/Code)](#verifiable-data)
- [De-biasing and Diversity](#diversity)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## After the "Data Wall": The Synthetic Shift

Frontier models (Llama 4, GPT-5.5, Claude Opus 4.7, Gemini 3.1 Pro) are trained on 100T+ tokens. There simply isn't enough human text to sustain this scaling.
**The reality:** More than **50% of the training mixture** for frontier fine-tuning (and 10% of pretraining) is now synthetic.

| Source | Human Data | Synthetic Data |
|--------|------------|----------------|
| **Volume** | Fixed (Finite) | Infinite |
| **Quality** | Variable (Noisy) | Controllable (Purified) |
| **Cost** | High (Human Labelers) | Cheap (Inference/GPU) |
| **Bias** | Mirror of internet | Can be manually balanced |

---

## Evol-Instruct Pattern

Evol-Instruct is a recursive process where an LLM takes a simple instruction and evolves it into a more complex one.

**The Evolution Directions:**
1. **Breadth**: Increase the number of tasks.
2. **Depth**: Add constraints, complicating factors, or multi-step logic.
3. **De-noising**: Clean up phrasing to remove "AI-isms."

```python
# Simple Instruction: "Write a function to add two numbers."
# Evolved Instruction: "Write a thread-safe Python class that performs 
# matrix addition with error handling and unit tests, adhering to PEP8."
```

> **Why (the rationale):** Manual creation of complex, diverse instruction-response pairs is extremely expensive and slow. Evol-Instruct enables a small seed set of simple instructions to be automatically expanded into thousands of progressively harder and more varied training examples, with no human effort per example.
> **When to use:** Use Evol-Instruct when you have a small high-quality seed dataset and need to scale it to a large, diverse training set. It is especially effective for coding tasks where complexity can be systematically increased (add error handling, multi-threading, edge cases).
> **Nuances & gotchas:** Evolved instructions can become unnaturally convoluted — they pile on constraints in ways real users would not phrase requests. This can cause the model to excel at artificial benchmarks while underperforming on natural user queries. De-noising (removing AI-isms) is important but imperfect; filtering evolved examples for naturalness is recommended.

---

## Constitutional AI & AI Feedback (RLAIF)

Developed by Anthropic and widely adopted across the industry, RLAIF uses a "Constitution" (a set of rules) to guide a model in evaluating and improving its own data.

**The Loop:**
1. **Propose**: Model A generates a response.
2. **Critique**: Model B (the constitutional judge) identifies flaws based on guidelines.
3. **Revise**: Model A produces a better version based on the critique.
4. **Train**: The final (Prompt, Revise) pair is added to the SFT set.

> **Why (the rationale):** Human labelers are expensive, slow, and inconsistent. RLAIF replaces per-example human judgment with a written constitution that an AI judge applies at scale — enabling millions of preference labels and revisions without human annotation at each step.
> **When to use:** Use RLAIF when you need to generate large quantities of preference or alignment training data and cannot afford human annotators at that scale. Constitutional AI is especially effective for safety and tone alignment where explicit principles can be written down.
> **Nuances & gotchas:** RLAIF inherits the biases and blind spots of the judge model — the constitution is only as good as the judge's ability to apply it, and the judge can misapply or circumvent its own rules. The revision loop can also converge to AI-isms: overly hedged, verbose, or unnaturally cautious text that satisfies the constitution but alienates real users. Human spot-checking of the critique-revise loop output is essential.

---

## Verifiable Synthetic Data

The biggest risk of synthetic data is **Model Collapse** (the model learning its own mistakes).
**The 2025 Solution**: Focus on domains where the "Truth" is verifiable without an LLM.

- **Math**: Use Formal Verification (Lean/Isabelle) or Python execution to verify answers.
- **Code**: Run generated code against test cases (Unit Tests).
- **RAG**: Use "Gold Context" to generate questions where the answer is explicitly in the text.

> **Why (the rationale):** Unverified synthetic data risks propagating the teacher model's errors into the training set, causing the student to learn and amplify those errors (model collapse). Verifiable data breaks this cycle — only examples that pass an external, model-independent check enter the training set.
> **When to use:** Always prefer verifiable synthetic data when your domain supports it (math, code, formal logic, structured QA with a known source document). For open-ended domains (creative writing, general conversation) where external verification is not possible, rely on quality filtering and human spot-checks instead.
> **Nuances & gotchas:** Verifiable domains are a minority of real-world tasks. Even within them, verification is not perfect — a unit test suite can pass while the code has logical bugs, and a math answer can match the gold answer through a wrong method. Verification catches the most egregious errors but not all subtle reasoning failures.

---

## De-biasing and Diversity

Synthetic data is used to "fill the gaps" in human data.
- **Languages**: Generating high-quality text in low-resource languages (e.g., Swahili, Marathi) by translating conceptual templates.
- **Logic**: Creating 1,000,000 variations of a specific logical fallacy to "harden" the model against it.

> **Why (the rationale):** Human internet data is skewed — English dominates, certain logical error types are rarely demonstrated with correct resolutions, and certain demographic perspectives are underrepresented. Synthetic data can deliberately oversample underrepresented categories to fill these gaps in ways that human data collection cannot practically achieve.
> **When to use:** Use targeted synthetic de-biasing when you have identified a specific model weakness (fails on a certain language, falls for a specific reasoning fallacy, performs poorly on a demographic group's typical queries) that is confirmed to stem from data underrepresentation.
> **Nuances & gotchas:** Synthetic diversity fixes data gaps but introduces a new risk: the synthesized minority-class data may be lower quality than native examples (e.g., machine-translated Swahili may have unnatural phrasing). Generating "variations of a logical fallacy" can also inadvertently teach the model to recognize only synthetic patterns of that fallacy rather than the real-world versions.

---

## Interview Questions

### Q: What is the risk of "Model Collapse" when training on synthetic data?

**Strong answer:**
Model Collapse occurs when a model is trained on data generated by an earlier version of itself. Because the model's distribution is narrower than the real world (it has preferences/biases for certain words and patterns), the training loop becomes a "positive feedback loop" of errors and blandness. By 2025, we mitigate this by:
1. Mixing in 5-20% "Golden" human-authenticated data.
2. Using "Verifiable" rewards (Math/Code) so mistakes are never learned.
3. Using more powerful "Teacher" models to generate data for "Student" models.

### Q: How do you ensure the *quality* of a synthetic dataset of 10 million rows?

**Strong answer:**
We use a **Multi-Stage Filtering Pipeline**:
1. **Semantic Deduplication**: Using embeddings to remove near-identical clusters.
2. **LLM-as-Judge**: Sampling 1% of the data and having a stronger model (e.g., GPT-5.2) score it for logic and safety.
3. **Perplexity Filtering**: Using a small model to calculate the perplexity of the text. If it's too high (nonsense) or too low (repetitive/simple), it's discarded.
4. **Verifiable Execution**: If the data contains code or math, it must pass a local compiler/interpreter check.

---

## References
- Xu et al. "WizardLM: Empowering Large Language Models to Follow Complex Instructions" (2023)
- Bai et al. "Constitutional AI: Harmlessness from AI Feedback" (2022)
- OpenAI. "Weak-to-Strong Generalization" (2023)

---

## Glossary

| Term | Simple explanation | Purpose |
|---|---|---|
| **Synthetic Data** | Text, code, or examples generated by an AI model rather than written by humans | Overcomes the finite supply of high-quality human text and enables infinite, controllable training data |
| **Data Wall** | The point at which all high-quality human-written text on the internet has been consumed for training | The practical limit that has forced the industry to rely on synthetic data to continue scaling |
| **Evol-Instruct** | A technique where an LLM iteratively rewrites simple instructions into more complex, constrained, multi-step versions | Generates harder and more diverse training examples from a small seed set without human effort |
| **Breadth Evolution** | Increasing the number of tasks or subtasks in an instruction during Evol-Instruct | Produces a wider variety of training examples covering more skill dimensions |
| **Depth Evolution** | Adding constraints, complicating factors, or multi-step logic to an instruction during Evol-Instruct | Forces the model to learn harder reasoning patterns rather than surface-level responses |
| **AI-isms** | Repetitive, formulaic phrases that LLMs tend to overuse (e.g., "Certainly!", "As an AI...") | Removing them during de-noising evolution makes synthetic data sound more natural and human |
| **Constitutional AI (CAI)** | An Anthropic-developed technique using a written set of rules (a "constitution") to guide a model to critique and revise its own outputs | Produces safer, higher-quality synthetic data without requiring human labels at each step |
| **RLAIF (RL from AI Feedback)** | Using an AI judge rather than human annotators to score and select model outputs for preference training | Scales alignment feedback generation far beyond what human labelers can produce |
| **Constitution** | A document listing ethical principles and quality criteria used by the judge model in Constitutional AI | The ruleset that guides the AI feedback loop, replacing human judgment with explicit written standards |
| **Model Collapse** | A failure mode where training repeatedly on model-generated data makes the model increasingly narrow and bland | The central risk of synthetic data pipelines — requires mixing in verified human data to prevent |
| **Verifiable Synthetic Data** | Synthetic data in domains where correctness can be checked by an external tool rather than another model | Eliminates model collapse risk by rejecting incorrect data before it enters the training set |
| **Formal Verification** | Using mathematical proof systems (Lean, Isabelle) to verify the correctness of proofs or reasoning steps | Provides a ground-truth oracle for math synthetic data that is completely model-independent |
| **Lean / Isabelle** | Proof assistant software systems that can formally verify mathematical proofs | Tools used to verify the correctness of math reasoning in synthetic data generation pipelines |
| **Unit Tests** | Small pieces of code that check whether a function produces the expected output for given inputs | Used to verify synthetic code examples — if tests pass, the code is correct; no human review needed |
| **Gold Context (RAG)** | A source document whose text explicitly contains the answer to a generated question | Ensures synthetic QA pairs are factually grounded and verifiable against a known reference |
| **Semantic Deduplication** | Using vector embeddings to find and remove training examples that are semantically similar even if not textually identical | Prevents the model from over-fitting on near-duplicate examples that waste training capacity |
| **LLM-as-Judge** | Using a strong LLM to automatically score samples from a dataset for quality, safety, or correctness | A scalable way to audit synthetic data quality without reviewing every example manually |
| **Perplexity Filtering** | Using a small model to score each training example and discard those that are too confusing or too trivially simple | Removes both nonsense (too high perplexity) and repetitive boilerplate (too low perplexity) from datasets |
| **Low-Resource Languages** | Languages with very little existing digitized text, making it hard to train high-quality models on them | Synthetic translation and generation techniques can create training data for these underrepresented languages |
| **Multi-Stage Filtering Pipeline** | A sequential series of automated quality checks (dedup, LLM judge, perplexity, execution) applied to a synthetic dataset | Ensures large synthetic datasets maintain high quality at scale without human review of every row |
| **Positive Feedback Loop** | In model collapse, the cycle where a model generates data, trains on it, and drifts further from the real distribution each iteration | The mechanism that makes model collapse self-reinforcing and increasingly hard to reverse |

---

*Next: [Quantization Deep Dive](07-quantization-deep-dive.md)*
