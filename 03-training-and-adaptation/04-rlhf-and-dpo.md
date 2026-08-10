# RLHF and DPO (Alignment)

Alignment is the process of ensuring an LLM's behavior matches human values and instructions. The field has moved from traditional RLHF to more efficient and scalable methods like DPO and Online RL.

## Table of Contents

- [The Alignment Problem](#the-alignment-problem)
- [RLHF: The Foundation](#rlhf-foundation)
- [DPO: Direct Preference Optimization](#dpo)
- [Online Alignment](#online-alignment)
- [Alignment for Reasoning Models](#alignment-for-reasoning)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## The Alignment Problem

Pretrained models are "knowledgeable but uncontrolled." They may:
1. Generate harmful content (Safety).
2. Fail to follow instructions (Instruction Following).
3. Hallucinate wildly (Factuality).

Alignment creates "Reward Models" and "Policy Updates" to steer the model.

---

## RLHF: The Foundation

Reinforcement Learning from Human Feedback (RLHF) involves three steps:
1. **SFT**: Supervised Fine-Tuning.
2. **Reward Model (RM)**: Train a model on `(Prompt, Winning_Response, Losing_Response)` to predict human scores.
3. **PPO (Proximal Policy Optimization)**: Use the RM to provide a "reward signal" to the LLM via Reinforcement Learning.

**Nuance**: Traditional RLHF is now considered too complex/unstable for most teams due to the overhead of training a separate Reward Model and the instability of PPO.

> **Why (the rationale):** SFT alone teaches the model to mimic good responses but cannot directly optimize for human preference rankings. RLHF bridges this gap by encoding human preferences into a reward model and then using RL to maximize that reward — enabling the model to be steered toward responses humans prefer even in novel situations not covered by SFT data.
> **When to use:** RLHF (PPO-based) is justified when you have significant human annotation capacity, a large team that can manage RL infrastructure, and need very precise control over preference modeling. It is still used at labs (OpenAI, Anthropic) for frontier alignment but is rarely the right first choice for most product teams.
> **Nuances & gotchas:** RLHF requires four models in memory simultaneously (policy, reference, reward model, and value network), which is extremely VRAM-intensive. PPO is notoriously sensitive to hyperparameters and subject to reward hacking — the policy learns to exploit the reward model's blind spots rather than genuinely improving. The reward model itself can have biases (e.g., preferring longer, verbose responses).

---

## DPO: Direct Preference Optimization

DPO is the industry standard. It eliminates the Reward Model.

> **Why (the rationale):** DPO skips the reward model of RLHF — it is mathematically equivalent to RLHF's optimal solution but implemented as a simple classification loss on preference pairs. This removes the most fragile and expensive components: no separate reward model to train, no PPO with its four-model memory requirement, and no reward hacking via an exploitable neural reward surface.
> **When to use:** DPO is the right default alignment method for almost all teams. Use it after SFT when you have preference data (winning/losing response pairs) and want to align the model without the infrastructure overhead of RLHF. It is simpler, stabler, and cheaper than PPO.
> **Nuances & gotchas:** DPO needs preference pairs (chosen vs rejected), not just good examples — collecting these is itself a labeling cost. DPO is an offline method: it learns from a fixed dataset and hits a quality ceiling once the model's distribution diverges from the reference data distribution. It does NOT eliminate the alignment tax — over-applying DPO or drifting too far from the reference model can still erode general capabilities. The KL penalty (reference model) is essential; without it the model can collapse.

### How it Works:
DPO uses the LLM itself as the Reward Model by mathematically deriving the optimal policy directly from preference data.
- **Goal**: Maximize the probability of the "winning" response and minimize the "losing" response, relative to a fixed "reference model."

### The Multi-Stage Alignment Pattern:
1. **Base SFT**: 5k-10k high-quality samples.
2. **DPO Step 1**: Alignment for instruction following.
3. **DPO Step 2**: Alignment for safety and specific tone.

---

## Online Alignment

**The Problem with Offline DPO**: It only learns from static data. If the model improves beyond that data, it hits a ceiling.

**The Solution: Online DPO (or RLOO)**:
1. The model generates 4-8 responses to a prompt.
2. A **Judge Model** (e.g., GPT-5.5, Claude Opus 4.7) or a **Rule-based Reward** (e.g., Code Execution) ranks them in real-time.
3. The model updates its policy immediately based on this "Online" feedback.

> **Why (the rationale):** Offline DPO trains on a fixed dataset — once the model improves past the quality level of that dataset, the preference pairs are no longer at its frontier and the gradient signal shrinks. Online DPO continuously generates fresh preference pairs from the current model, keeping the training signal at the edge of the model's capability.
> **When to use:** Use online DPO/RLOO when you have a scalable automated judge (code executor, math verifier, or a strong LLM judge) and your model has already been through an initial offline DPO phase and plateaued. This is the bridge between simple DPO and full RLVR.
> **Nuances & gotchas:** Online alignment is significantly more expensive than offline DPO — you pay inference cost for every training step. If the judge model is itself imperfect (LLM judge rather than rule-based), the model can learn to fool the judge rather than genuinely improve. Requires careful monitoring for distribution collapse and reward hacking.

---

## Alignment for Reasoning Models (o1/DeepSeek-R1 style)

Aligning "Thinking" models requires a shift from **Response Preference** to **Process Preference**.

> **Why (the rationale):** Standard alignment rewards the final answer, but reasoning models produce long chains of thought. Rewarding only correct final answers fails because the model can reach the right answer through unfaithful or flawed reasoning — and that flawed reasoning gets reinforced. Rewarding the CoT process ensures the model learns correct thinking, not just correct guessing.
> **When to use:** Apply process-preference alignment (verification-based RL) when the model is a "thinking" model producing CoT and you have access to a verifiable correctness oracle (math checker, code test runner). Do not apply this to general conversational alignment where "process" is subjective.
> **Nuances & gotchas:** Verifying intermediate reasoning steps (not just the final answer) is extremely hard outside math and code — in most domains you cannot programmatically check whether a reasoning step is valid. "Verification-based RL" is therefore not a general solution; it is domain-constrained to problems with objectively checkable answers.

| Feature | Standard Alignment | Reasoning Alignment |
|---------|-------------------|---------------------|
| Reward Target | The final answer | The **Chain of Thought (CoT)** |
| Reward Signal | Helpful/Safe | **Correctness + Conciseness** |
| Method | Human Ranking | Rule-based (e.g., "Did the code run?") |

**Principal-level Nuance**: "Verification-based RL" is the secret to today's frontier models. Instead of humans saying what is better, we use hard verifiable outcomes (Math answers, Code test cases) as the reward signal.

---

## Interview Questions

### Q: Why is DPO often preferred over RLHF/PPO?

**Strong answer:**
DPO is preferred primarily due to its simplicity and stability. PPO requires maintaining four models in memory (Policy, Reference, Value, and Reward), which is extremely VRAM-intensive. Furthermore, PPO is notoriously sensitive to hyperparameters and often suffers from "reward hacking" or sudden collapse. DPO treats alignment as a simple classification problem on preference pairs, making it much more robust, easier to tune, and significantly cheaper to run.

### Q: What is the risk of "Alignment Tax"?

**Strong answer:**
The "Alignment Tax" refers to the decline in a model's raw capabilities (e.g., coding, creative writing, or logical reasoning) after it is aligned for safety or specific personas. Because the model is being forced to prioritize safety or adherence to a specific style, it may become "too cautious" or lose the nuance it learned during pretraining. Modern techniques like **Steerable Alignment** and **DPO-with-KL-penalty** aim to minimize this by ensuring the model's policy doesn't drift too far from the original pretrained distribution.

---

## References
- Rafailov et al. "Direct Preference Optimization: Your Language Model is Secretly a Reward Model" (2023)
- Schulman et al. "Proximal Policy Optimization Algorithms" (2017)
- OpenAI. "Learning to Reason with LLMs" (2024)

---

## Glossary

| Term | Simple explanation | Purpose |
|---|---|---|
| **Alignment** | The process of making an LLM's outputs match human values, instructions, and safety requirements | Turns a raw pretrained model into one that is helpful, honest, and safe to deploy |
| **RLHF (Reinforcement Learning from Human Feedback)** | A three-stage training process using human preference labels and a reward model to steer LLM behavior | The foundational alignment method used to train models like early InstructGPT and ChatGPT |
| **Reward Model (RM)** | A separate neural network trained on human preference pairs to predict which response a human would prefer | Provides a differentiable signal for the RL policy to optimize toward human preferences |
| **PPO (Proximal Policy Optimization)** | A reinforcement learning algorithm that updates the policy in small, clipped steps to avoid catastrophic policy collapse | The RL algorithm used in classic RLHF to apply the reward model's signal to the LLM |
| **Policy** | In RL, the model being trained — here, the LLM that generates responses | The entity whose behavior is being shaped by the reward signal |
| **DPO (Direct Preference Optimization)** | An alignment method that treats the LLM itself as an implicit reward model, using preference pairs without a separate RM | Simpler and more stable than PPO-based RLHF; the current industry standard for alignment |
| **Preference Data** | Pairs of (Prompt, Winning Response, Losing Response) labeled by humans or a judge model | The training signal for both RLHF reward models and DPO |
| **Reference Model** | A frozen copy of the SFT model used as a baseline in DPO to prevent the policy from drifting too far | Keeps the aligned model close to the original pretrained behavior, preserving general capabilities |
| **KL Penalty (KL Divergence Penalty)** | A regularization term that penalizes the policy for deviating too much from the reference model | Prevents "alignment collapse" where the model becomes unhelpfully constrained or loses general capabilities |
| **Offline DPO** | DPO trained on a static, pre-collected dataset of preference pairs | Simple to run but hits a quality ceiling once the model improves beyond the fixed dataset |
| **Online DPO** | DPO where the model generates new responses in real time, which are then scored and used for immediate updates | Continuously improves the model beyond a fixed dataset ceiling; more effective but harder to run |
| **RLOO (REINFORCE Leave-One-Out)** | An online RL variant that uses a leave-one-out baseline from multiple samples rather than a learned critic | A simpler alternative to PPO for online alignment that avoids training a separate value network |
| **Judge Model** | A strong LLM (e.g., GPT-5.5, Claude Opus) used to automatically score or rank model outputs in place of human annotators | Scales preference data collection without requiring expensive human labeling at each step |
| **Hallucination** | When a model generates confident but factually incorrect or fabricated information | A key alignment failure mode that alignment training attempts to reduce through factuality rewards |
| **Reward Hacking** | When the model learns to score high on the reward model through unintended shortcuts rather than genuine quality | A failure mode in RLHF where the policy exploits the reward model's blind spots |
| **Chain of Thought (CoT)** | A reasoning trace where the model explicitly writes out intermediate steps before giving a final answer | The target for reasoning alignment — rewarding correct CoT rather than just correct final answers |
| **Verification-based RL** | Using external, rule-based checks (math solvers, code executors) as the reward signal instead of a neural reward model | Sidesteps reward model hacking by grounding rewards in objective, unchallengeable facts |
| **Alignment Tax** | The drop in raw capability (coding, creativity, reasoning) that can occur after aligning a model for safety or tone | The key trade-off to manage when aligning models — avoid over-constraining general intelligence |
| **Steerable Alignment** | Alignment techniques that allow the model's safety or persona level to be adjusted at inference time | Lets deployments tune the balance between safety and capability without retraining |
| **Process Preference** | Rewarding the quality of the reasoning steps (CoT) rather than just the final answer | Required for aligning thinking/reasoning models like o1 or DeepSeek-R1 |
| **Instruction Following** | The ability of a model to reliably carry out specific user directives in its responses | A primary goal of alignment — makes the model actually useful for task completion |

---

*Next: [Knowledge Distillation](05-knowledge-distillation.md)*
