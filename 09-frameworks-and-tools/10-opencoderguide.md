# OpenCoder: AI Coding Agents Landscape

The AI coding agent landscape has exploded. This guide covers open-weight coding models, agentic IDEs, open-source agents, and how to choose the right tool for your engineering workflow.

## Table of Contents

- [The AI Coding Landscape (2026)](#landscape)
- [Open-Weight Coding Models](#models)
- [AI-Native IDEs](#ides)
- [Open-Source Coding Agents](#agents)
- [Benchmark Deep Dive](#benchmarks)
- [Cost Comparison](#costs)
- [Selection Guide](#selection)
- [Production Architecture](#production)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## The AI Coding Landscape (2026)

The coding AI landscape has three distinct layers:

```
┌─────────────────────────────────────────────────────────────┐
│                    AI CODING STACK (2026)                    │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  LAYER 3: CODING AGENTS (Autonomous, multi-turn)           │
│  ┌──────────────┐ ┌────────────┐ ┌────────────────────┐   │
│  │  Claude Code │ │  OpenHands │ │  Cline / Aider     │   │
│  │  (Anthropic) │ │  (Open)    │ │  (Open)            │   │
│  └──────────────┘ └────────────┘ └────────────────────┘   │
│                                                             │
│  LAYER 2: AI IDEs (Completion + editing, developer-in-loop)│
│  ┌──────────────┐ ┌────────────┐ ┌────────────────────┐   │
│  │    Cursor    │ │  Windsurf  │ │  GitHub Copilot    │   │
│  └──────────────┘ └────────────┘ └────────────────────┘   │
│                                                             │
│  LAYER 1: CODING MODELS (The brains behind everything)     │
│  ┌──────────────┐ ┌────────────┐ ┌────────────────────┐   │
│  │  Opus 4.7    │ │  GPT-5.5   │ │ DeepSeek V4 Pro    │   │
│  │  Sonnet 4.6  │ │ Gemini 3.1 │ │ Qwen 3.6 Coder     │   │
│  └──────────────┘ └────────────┘ └────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Open-Weight Coding Models

> **Why (the rationale):** Open-weight models remove the two blockers that prevent enterprises from using closed API models at scale: data-privacy requirements (no code leaves the network) and per-token cost at high volume.
> **When to use:** When compliance mandates on-premises execution, when volume exceeds ~500 tasks/day and self-hosting becomes cost-competitive, or when fine-tuning on domain-specific code (internal DSLs, proprietary frameworks) is needed.
> **Nuances & gotchas:** The quality gap between the best open models (Qwen2.5-Coder-32B) and frontier closed models (Claude Sonnet 4.6) is real but narrowing for completions; for complex multi-file agentic tasks, the gap is still meaningful and should be benchmarked on your specific workload before switching.

These models can be self-hosted, fine-tuned, and deployed without any API dependency.

### Qwen2.5-Coder (Alibaba)

> **Why (the rationale):** Provides the strongest commercially-licensed open coding model family for self-hosted deployments, from a 1.5B edge model to a 32B model that approaches frontier closed-model quality.
> **When to use:** Best open choice for general self-hosted coding tasks; 32B for quality-first deployments (needs 2× A100), 7B for cost-sensitive or single-GPU setups, 1.5B for edge/on-device.
> **Nuances & gotchas:** Apache 2.0 license permits commercial use freely; benchmark on SWE-bench-style tasks for your codebase before assuming HumanEval+ scores translate to agentic task quality — function completion benchmarks overstate agentic ability.

A strong open-source coding model family. As of May 2026, the open-source coding leaders are Qwen 3.6 Coder and DeepSeek V4 Pro; Qwen 2.5 Coder remains a popular pick for self-hosted deployments on smaller hardware:

| Model | Parameters | Context | HumanEval+ | Notes |
|-------|------------|---------|------------|-------|
| Qwen2.5-Coder-32B-Instruct | 32B | 128K | 88.2% | Best open coding model |
| Qwen2.5-Coder-7B-Instruct | 7B | 128K | 79.3% | Excellent small model |
| Qwen2.5-Coder-1.5B | 1.5B | 32K | 65.8% | Edge/on-device use |

**Strengths:**
- Strong on coding benchmarks; competitive with frontier closed models on SWE-bench Verified
- 100+ programming languages
- Excellent fill-in-the-middle (FIM) for completions
- Apache 2.0 license — fully commercial

```python
# Self-hosted with vLLM
from vllm import LLM

model = LLM(
    model="Qwen/Qwen2.5-Coder-32B-Instruct",
    tensor_parallel_size=2,  # 2× A100 80GB
)
response = model.generate("def fibonacci(n: int) -> list[int]:")
```

### DeepSeek-Coder-V2 (DeepSeek)

> **Why (the rationale):** MoE architecture means only ~21B parameters activate per token from a 236B total model, giving frontier-quality reasoning at significantly lower inference cost than a dense model of equivalent benchmark performance.
> **When to use:** Competitive programming, algorithmic tasks, or any workload where reasoning depth matters and you have hardware that can run the full model (the Lite at 16B is workable on a single node).
> **Nuances & gotchas:** Chinese-owned model — some enterprise security policies restrict its use for proprietary code; check your organization's third-party AI policy before deploying.

| Model | Parameters | Architecture | HumanEval+ |
|-------|------------|-------------|------------|
| DeepSeek-Coder-V2-Instruct | 236B (MoE) | MoE | 90.2% |
| DeepSeek-Coder-V2-Lite | 16B (MoE) | MoE | 81.1% |

**Strengths:**
- MoE architecture → activates only 21B params per token (efficient)
- Strong on competitive programming (CodeForces problems)
- Open weights; strong Chinese language support

### StarCoder2 (BigCode / Hugging Face)

> **Why (the rationale):** Trained on permissively licensed code with a fully open BigCode OpenRAIL-M license, making it the safest open choice for teams worried about training data IP; optimized for low-latency IDE completions.
> **When to use:** IDE autocompletion, low-latency self-hosted inference, or teams where the model's training data provenance must be fully auditable.
> **Nuances & gotchas:** 16K context window is limiting for agentic tasks that require reading entire files; use Qwen2.5-Coder for agentic workflows and StarCoder2 specifically for completion-focused use cases.

| Model | Parameters | Context | Notes |
|-------|------------|---------|-------|
| StarCoder2-15B | 15B | 16K | Best mid-size open coding LM |
| StarCoder2-7B | 7B | 16K | Efficient, 80+ languages |
| StarCoder2-3B | 3B | 16K | Lightweight, on-device |

**Strengths:**
- Fully open (BigCode OpenRAIL-M license)
- Excellent for IDE completions (low latency)
- Strong on Stack Overflow / GitHub data

### DeepSeek-R1-Distill (for coding)

| Model | Parameters | Math/Code | Notes |
|-------|------------|-----------|-------|
| DeepSeek-R1-Distill-Qwen-32B | 32B | Excellent | Reasoning distilled into smaller model |
| DeepSeek-R1-Distill-Llama-8B | 8B | Good | Tiny reasoning model |

**Use case**: When you need reasoning-quality code generation at self-hosted scale.

### Open Model Selection Guide

```
Simple completions (< 100ms latency needed)?
  → StarCoder2-3B or Qwen2.5-Coder-1.5B (local, fast)

Best quality self-hosted?
  → Qwen2.5-Coder-32B-Instruct (2× A100)

Budget < 1× A100 GPU?
  → Qwen2.5-Coder-7B-Instruct (1× RTX 4090 sufficient)

Need reasoning + coding?
  → DeepSeek-R1-Distill-Qwen-32B

Competitive programming / algorithmic?
  → DeepSeek-Coder-V2 or DeepSeek-R1
```

---

## AI-Native IDEs

> **Why (the rationale):** AI-native IDEs embed the agent loop directly in the editing environment, eliminating the context-switching cost of moving between a terminal agent and an editor — developers see the changes as they are applied, enabling faster review and steering.
> **When to use:** Daily development work where the developer wants to stay in the editing experience, with AI assistance ranging from completions to multi-file agentic edits; contrast with headless agents (Claude Code SDK) for batch/CI use.
> **Nuances & gotchas:** IDE agents have medium autonomy by design — they require user confirmations for file changes in most modes, so they are not suitable replacements for headless CI agents; choose based on the human-in-loop level your workflow requires.

### Cursor

> **Why (the rationale):** Cursor's multi-file Composer + predictive Tab completions give the highest-quality interactive AI editing experience among IDE tools, with model flexibility (any frontier model) and a familiar VS Code interface.
> **When to use:** Frontend/full-stack developers who spend most time in the editor and want the best AI-assisted editing quality; teams willing to pay $20/mo per developer for IDE-level productivity gains.
> **Nuances & gotchas:** Closed-source — your code is sent to Cursor's servers (Privacy Mode available but worth confirming for regulated industries); the underlying editing model is tightly coupled to Cursor's infrastructure, meaning behavior can change on Cursor releases regardless of the underlying LLM.

**Website:** cursor.sh | **Base:** VS Code fork | **Pricing:** $20/mo Pro

Cursor is the leading AI-native IDE. Key capabilities:

| Feature | Description |
|---------|-------------|
| **Composer** | Multi-file agentic editing (Cursor's equivalent of Claude Code) |
| **Ctrl+K** | Inline code generation |
| **Tab** | Predictive completions (smarter than Copilot) |
| **@-mentions** | Attach files, URLs, docs to context |
| **.cursorrules** | Project-level AI instructions (like CLAUDE.md) |
| **Model choice** | GPT-5.5, Claude Sonnet 4.6 / Opus 4.7, Gemini 3.1 Pro, DeepSeek V4 Pro |

**Best for**: Frontend/full-stack developers who want agentic editing within a familiar GUI.

**Limitations**: Closed-source; your code is sent to Cursor's servers (they offer a Privacy Mode).

### Windsurf (by Codeium)

> **Why (the rationale):** Windsurf offers a Cursor-comparable experience with a free tier and full model flexibility, lowering the adoption barrier for teams not ready to commit to a paid subscription or to a single model provider.
> **When to use:** Teams that want AI-native IDE features on a budget, or developers who regularly switch between model providers and want that flexibility without lock-in.
> **Nuances & gotchas:** The free tier has usage limits that can surprise users mid-session; Flows (deterministic agentic sequences) are Windsurf-specific and not interoperable with other agent frameworks if you later want to migrate workflow logic.

**Website:** codeium.com/windsurf | **Base:** VS Code fork | **Pricing:** Free tier + $15/mo Pro

Windsurf differentiates via **Flows** (not to be confused with CrewAI Flows):

| Feature | Description |
|---------|-------------|
| **Cascade** | Windsurf's agentic editing mode |
| **Flows** | Deterministic agentic sequences (agent + user in harmony) |
| **Model choice** | Any: GPT-5.5, Claude Sonnet 4.6 / Opus 4.7, Gemini 3.1 Pro, DeepSeek V4 |
| **Free tier** | Generous free credits |

**Best for**: Teams that want Cursor-like experience with a free tier and model flexibility.

### GitHub Copilot (Microsoft/OpenAI)

| Feature | Status (May 2026) |
|---------|---------------------|
| Completions | ✅ Still the market leader by install base |
| Copilot Workspace | ✅ Multi-file agentic editing (in GA) |
| Model | GPT-5.5 (default), Claude Sonnet 4.6 / Opus 4.7 (available) |
| Enterprise features | ✅ IP protection, org policies, code referencing off |

**Best for**: Enterprise teams already on Microsoft/GitHub ecosystem.

**2026 reality**: Copilot's completion quality has been surpassed by Cursor/Windsurf for most developers, but its enterprise features and GitHub integration keep it dominant in large orgs.

### Google Antigravity

Antigravity is Google's agentic development platform, the successor to the Gemini CLI. It is less a text editor and more an **agent-first workspace** built around Gemini 3:

| Feature | Detail |
|---------|--------|
| **Agent Manager** | A dedicated view to launch, watch, and steer multiple async coding agents instead of editing files one at a time |
| **Planning + artifacts** | Agents produce a plan and reviewable artifacts (diffs, task lists, live browser sessions) before and during execution |
| **Built-in browser** | Agents can run and visually test the UI they build |
| **Model optionality** | Gemini 3 Pro by default, with support for Anthropic Claude and open models |
| **Platform** | Cross-platform (macOS, Windows, Linux); public preview, free for individuals |

**Best for**: Developers who want to operate at the "task" level (delegate a goal, review the plan and result) rather than the "edit" level. It competes with Cursor's Composer and Claude Code's agentic loop, with Google's bet being the multi-agent manager UI and tight Gemini 3 integration.

---

## Open-Source Coding Agents

> **Why (the rationale):** Open-source agents give teams full control over the execution environment, model choice, and code handling — critical for regulated industries where proprietary SaaS agents cannot process the code.
> **When to use:** Enterprise security teams requiring auditability; teams wanting to self-host in CI pipelines; developers who want model flexibility without a subscription.
> **Nuances & gotchas:** Open-source agents typically score 10-15% below Claude Code on SWE-bench Verified with the same model backend, because Claude Code's agent loop has been more extensively tuned; the quality gap narrows significantly when paired with the best available models (Claude Sonnet 4.6 / Opus 4.7).

### OpenHands (formerly OpenDevin)

> **Why (the rationale):** OpenHands provides the most complete open-source autonomous agent stack (controller, Docker sandbox, browser, file editor) with a web UI and REST API, making it the reference open alternative to Claude Code.
> **When to use:** Self-hosted CI pipelines, teams with open-source or data-privacy requirements, or any scenario where you need to swap models (Claude, GPT, local Ollama) behind the same agent framework.
> **Nuances & gotchas:** Docker-in-Docker is required; the REST API is less battle-tested than Claude Code's SDK for high-volume CI; SWE-bench scores (~55-60%) are meaningfully below Claude Code (~87%) — validate on your actual workload.

**GitHub:** github.com/All-Hands-AI/OpenHands | **License:** MIT

The leading open-source autonomous coding agent:

```bash
# Run with Docker
docker pull docker.all-hands.dev/all-hands-ai/openhands:latest
docker run -it --rm \
  -e SANDBOX_RUNTIME_CONTAINER_IMAGE=docker.all-hands.dev/all-hands-ai/runtime:latest \
  -e LLM_API_KEY=$ANTHROPIC_API_KEY \
  -e LLM_MODEL=claude-3-7-sonnet-20250219 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -p 3000:3000 \
  docker.all-hands.dev/all-hands-ai/openhands:latest
# Access at http://localhost:3000
```

**Architecture:**
```
User request
    ↓
OpenHands Controller
    ├── CodeActAgent (main strategy)
    ├── Docker Sandbox (isolated execution)
    ├── File editor (str_replace_editor)
    └── Browser (playwright for web tasks)
```

**Key features:**
- **Any LLM**: Works with Claude Sonnet 4.6 / Opus 4.7, GPT-5.5, Gemini 3.1 Pro, DeepSeek V4, local Ollama
- **Docker sandbox**: Agent runs in isolated container
- **Web UI**: Chat-like interface; shows agent's reasoning
- **API access**: REST API for CI integration
- **SWE-bench score**: ~55-60% (depending on backend model)

### Aider

> **Why (the rationale):** Aider's git-native design means every agent change is committed incrementally with a clean message, producing a reviewable audit trail that other agents (which dump changes as a blob diff) don't provide; the context map lets it reason about cross-file dependencies without overwhelming context.
> **When to use:** CLI-first developers who want full git history as the agent works; iterative refactoring tasks where auditability of each change matters; model-agnostic setups needing any OpenAI-compatible backend.
> **Nuances & gotchas:** No CI/headless REST API (pure CLI); the codebase context map adds startup overhead on large repos; voice mode is a novelty for most production use cases.

**GitHub:** github.com/paul-gauthier/aider | **License:** Apache 2.0

Terminal-first, git-native coding agent:

```bash
pip install aider-chat

# Works directly with your git repo
aider --model claude-3-7-sonnet-20250219

# Add files to context
/add src/auth.py src/models.py

# Give task
> Add JWT authentication to the User model
```

**What makes Aider different:**
- **Git-native**: Commits changes as it goes; maintains clean git history
- **Context maps**: Maintains a map of your entire codebase (even files not in context)
- **Voice mode**: Speak tasks aloud  
- **Architecture mode**: Discusses design before touching code

```bash
# SWE-bench Verified benchmarks (May 2026)
# Aider + Claude Sonnet 4.6  → ~74%
# Aider + Claude Opus 4.7    → ~87%
# Aider + GPT-5.5            → ~88%
```

### Cline (VS Code Extension)

> **Why (the rationale):** Cline brings full autonomous coding into VS Code for free, with per-action permission prompts that keep humans in the loop on every shell command — the safest autonomous agent model for developers new to agentic workflows.
> **When to use:** Developers who want Cursor-like autonomy inside VS Code without a subscription, with full model flexibility (local Ollama included) and native MCP support.
> **Nuances & gotchas:** Per-action prompts become fatiguing for long tasks; no headless/CI mode makes it unsuitable for batch automation; MCP support is strong but the extension's permission model can conflict with corporate VS Code policies.

**GitHub:** github.com/cline/cline | **License:** Apache 2.0

Open-source VS Code extension for autonomous coding:

```
VS Code
  └── Cline Extension
        ├── Any model (Claude, GPT, Gemini, Ollama)
        ├── File system access (read/write any file)
        ├── Terminal (bash commands)
        ├── Browser (playwright)
        └── MCP servers (any MCP tool)
```

**Key differentiators:**
- **MCP-native**: Full MCP support out of the box
- **Permission per action**: Every shell command, file edit requires user approval
- **Model flexibility**: Supports any OpenAI-compatible API endpoint (including local Ollama)
- **Free**: Open-source, no subscription

**Best for**: Developers who want Cursor-like experience for free, with full model flexibility.

---

## Benchmark Deep Dive

### SWE-bench Verified (March 2026)

The gold standard for agentic software engineering. Measures ability to resolve real GitHub issues.

| Agent / System | Score | Model Backend | Notes |
|---------------|-------|---------------|-------|
| GPT-5.5 (single-shot leader) | 88.7% | OpenAI | Holds #1 on SWE-Bench Verified (May 2026) |
| Claude Opus 4.7 (Anthropic) | 87.6% | Anthropic | Leads SWE-Bench Pro at 64.3% |
| Claude Code | ~87% | Claude Opus 4.7 / Sonnet 4.6 | Anthropic's official agent |
| OpenHands (best config) | ~75% | Claude Sonnet 4.6 | Open-source |
| Aider | ~74% | Claude Sonnet 4.6 / Opus 4.7 / GPT-5.5 | Open-source CLI |
| SWE-agent | ~55% | GPT-5.5 | Princeton research baseline |

> [!NOTE]
> SWE-bench scores are highly sensitive to backend model. The same agent with claude-3-7-sonnet typically scores 10-15% higher than with GPT-4o.

### HumanEval+ (Open Models)

| Model | HumanEval+ Score |
|-------|-----------------|
| Claude 3.7 Sonnet | 93.6% |
| GPT-4o | 90.2% |
| Qwen2.5-Coder-32B-Instruct | 88.2% |
| DeepSeek-Coder-V2-Instruct | 90.2% |
| StarCoder2-15B | 73.3% |

### LiveCodeBench (Runtime evaluation, stronger signal)

LiveCodeBench uses fresh competitive programming problems (not in training data):

| Model | LiveCodeBench Score |
|-------|---------------------|
| o3 (high) | 68.1% |
| Claude 3.7 Sonnet | 54.2% |
| GPT-4.5 | 38.7% |
| Qwen2.5-Coder-32B | 43.2% |
| DeepSeek-R1 | 57.0% |

**Insight**: LiveCodeBench scores are much lower than HumanEval because it tests novel problems. o3 and DeepSeek-R1 dominate due to their reasoning capabilities.

---

## Cost Comparison

### Closed API vs. Open Self-Hosted

**Scenario: 1,000 coding tasks/day, avg 5K tokens each**

| Approach | Monthly Cost | Quality | Latency |
|----------|-------------|---------|---------|
| Claude 3.7 Sonnet (API) | ~$9,000 | ★★★★★ | Medium |
| GPT-4o (API) | ~$7,500 | ★★★★ | Medium |
| o3-mini (API) | ~$3,300 | ★★★★★ (reasoning) | Slow |
| Qwen2.5-Coder-32B (4×A100) | ~$4,000 (infra) | ★★★★ | Fast |
| DeepSeek-V3 (Together AI) | ~$1,350 | ★★★★ | Medium |

**Key insight**: Self-hosting Qwen2.5-Coder-32B becomes cost-competitive at ~500+ tasks/day compared to Claude API. For <200 tasks/day, API is almost always cheaper when you factor in engineering overhead.

---

## Selection Guide

### Quick Decision Tree

```
What is your primary need?

├─ IDE coding assistance (completions + chat)?
│  ├─ Microsoft ecosystem / enterprise? → GitHub Copilot
│  ├─ Want best quality? → Cursor (Pro)
│  └─ Want free + model choice? → Windsurf or Cline
│
├─ Autonomous agent for standalone coding tasks?
│  ├─ Best quality, don't mind proprietary? → Claude Code
│  ├─ Need open-source? → OpenHands
│  ├─ CLI-first, git-native? → Aider
│  └─ VS Code embedded, MCP-native? → Cline
│
├─ Self-hosted model for custom deployment?
│  ├─ Best quality? → Qwen2.5-Coder-32B
│  ├─ Need reasoning? → DeepSeek-R1-Distill-32B
│  ├─ Fast completions? → Qwen2.5-Coder-7B or StarCoder2-7B
│  └─ Edge/on-device? → Qwen2.5-Coder-1.5B or StarCoder2-3B
│
└─ CI/CD pipeline integration?
   ├─ Best results? → Claude Code SDK (headless)
   ├─ Open-source? → OpenHands REST API
   └─ Git-native? → Aider CLI in GitHub Actions
```

### Comparison Matrix

| Dimension | Claude Code | Cursor | OpenHands | Aider | Cline |
|-----------|-------------|--------|-----------|-------|-------|
| Autonomy | Full | Medium | Full | Full | Full |
| Model lock | Claude | Any | Any | Any | Any |
| Open Source | ❌ | ❌ | ✅ | ✅ | ✅ |
| CI/Headless | ✅ | ❌ | ✅ | ✅ | ❌ |
| GUI | CLI | Full IDE | Web UI | Terminal | VS Code |
| MCP | ✅ | ✅ | Partial | ❌ | ✅ |
| Git-native | Partial | Partial | ✅ | ✅ | Partial |
| Price | API costs | $20/mo | Free + API | Free + API | Free + API |

---

## Production Architecture

### Enterprise Coding Agent Platform

Here's how to build an internal AI coding platform:

```
┌────────────────────────────────────────────────────────────┐
│             ENTERPRISE CODING AGENT PLATFORM                │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  Developer                                                 │
│     ↓ (Jira ticket / PR description)                      │
│  ┌──────────────────────────────────┐                      │
│  │        TASK INTAKE LAYER         │                      │
│  │  • Parse task from Jira/GitHub   │                      │
│  │  • Classify: simple/complex      │                      │
│  │  • Route to appropriate agent    │                      │
│  └──────────────┬───────────────────┘                      │
│                 │                                          │
│    Simple fix   │   Complex feature                        │
│        ↓        │        ↓                                 │
│  ┌──────────┐   │  ┌──────────────────┐                    │
│  │  Aider   │   │  │   Claude Code    │                    │
│  │ (cheap)  │   └→ │  SDK (headless)  │                    │
│  └────┬─────┘      └────────┬─────────┘                    │
│       │                     │                              │
│       └─────────────────────┘                              │
│                 ↓                                          │
│  ┌──────────────────────────────────┐                      │
│  │         REVIEW LAYER             │                      │
│  │  • Git diff → PR creation        │                      │
│  │  • Auto-run CI tests             │                      │
│  │  • Human review (required)       │                      │
│  └──────────────────────────────────┘                      │
│                 ↓                                          │
│         Merge to main (human approved)                     │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

### Key Production Decisions

| Decision | Options | Recommendation |
|----------|---------|----------------|
| Model for agent | Claude 3.7, GPT-4o, open | Claude 3.7 Sonnet for best results |
| Task intake | Manual, Jira webhook, GitHub label | GitHub label triggers Actions workflow |
| Code execution | Local, Docker, E2B | Docker (reproducible, isolated) |
| Human review | PR, Slack approval, automated | Required PR review, never auto-merge |
| Cost control | Max turns, model routing | max_turns=20, Haiku for simple tasks |

---

## Interview Questions

### Q: How do you choose between Claude Code, Cursor, and OpenHands?

**Strong answer:**
It depends on three axes:

1. **Interface need**: If developers want GUI (see changes in context), use Cursor or Windsurf. If the task is scripted/headless (bug fixing, test generation in CI), use Claude Code SDK or OpenHands.

2. **Model control**: If you need to use any model (or your own fine-tuned model), use OpenHands or Aider. If you're okay with Anthropic only and want best-in-class results, use Claude Code.

3. **Open-source requirement**: Enterprise security teams often require open-source tools they can audit. OpenHands (MIT) and Aider (Apache 2.0) are the answer.

For a typical startup, I'd recommend: Cursor for daily development, Claude Code for batch tasks (PRs from GitHub issues), and OpenHands for self-hosted CI pipelines.

### Q: Why are open-weight coding models like Qwen2.5-Coder important for enterprise?

**Strong answer:**
Three reasons:

1. **Data privacy**: Code sent to closed APIs is potentially used for training or exposed to third parties. For healthcare (HIPAA), finance (SOX), and government teams, no proprietary code can leave the network. Qwen2.5-Coder-32B running on-prem solves this.

2. **Cost at scale**: At 1M+ code generation requests/month, self-hosting becomes 40-60% cheaper than API pricing, especially for completions (vs agentic tasks).

3. **Fine-tuning**: Open weights can be domain-specialized. A legal tech company can fine-tune on their internal DSL (domain-specific language). APIs don't allow this.

The quality gap between Qwen2.5-Coder-32B and Claude 3.7 Sonnet is real but shrinking. For completions and simpler tasks, the open model is often "good enough."

### Q: How would you design the testing strategy for an AI coding agent in CI?

**Strong answer:**
I'd use a three-tier evaluation:

**1. Functional tests** (automated, every run):
```
Agent output → Run pytest → Pass rate metric
```

**2. Ground truth comparison** (weekly):
```
Known bug → Agent fix → Compare to expert fix
Metric: Semantic similarity of diff (not byte-exact)
```

**3. Human evaluation** (sample 5% of agent PRs):
```
Senior engineer rates: Correctness, Style, Safety, 1-5 scale
```

I also track **regression rate** — if an agent fix introduces a new failing test, that's a hard failure. The agent should run the full test suite and only succeed if it improves or maintains the passing rate.

---

## References

- Qwen2.5-Coder: https://qwenlm.github.io/blog/qwen2.5-coder/
- DeepSeek-Coder-V2: https://github.com/deepseek-ai/DeepSeek-Coder-V2
- StarCoder2: https://huggingface.co/blog/starcoder2
- OpenHands: https://github.com/All-Hands-AI/OpenHands
- Aider: https://aider.chat/
- Cline: https://github.com/cline/cline
- Cursor: https://cursor.sh/
- Windsurf: https://codeium.com/windsurf
- Google Antigravity: https://developers.googleblog.com/build-with-google-antigravity-our-new-agentic-development-platform/
- SWE-bench Leaderboard: https://www.swebench.com/
- LiveCodeBench: https://livecodebench.github.io/

---

---

## Glossary

| Term | Simple explanation | Purpose |
|---|---|---|
| **Open-Weight Model** | An LLM whose trained weights are publicly released, allowing anyone to download, self-host, and fine-tune it. | Enables enterprises with data-privacy or cost requirements to run capable coding models entirely on their own infrastructure. |
| **Qwen2.5-Coder** | Alibaba's family of open-weight coding models ranging from 1.5B to 32B parameters. | Provides a commercially licensed, self-hostable coding model competitive with frontier closed models for many tasks. |
| **DeepSeek-Coder-V2** | DeepSeek's open-weight coding model using a Mixture-of-Experts architecture with 236B total but only ~21B active parameters. | Delivers high-quality code generation with lower inference cost than a dense model of comparable capability. |
| **StarCoder2** | BigCode's open-weight coding model family, released under the BigCode OpenRAIL-M license. | Provides a fully open coding model well-suited to low-latency IDE completions. |
| **DeepSeek-R1-Distill** | A smaller model produced by distilling the reasoning capability of DeepSeek-R1 into a Qwen or Llama backbone. | Brings reasoning-quality code generation to self-hosted deployments that cannot run the full 671B R1 model. |
| **MoE (Mixture of Experts)** | A model architecture where the full parameter set is divided into specialized sub-networks ("experts"), and only a subset activates for each token. | Reduces inference cost per token compared to a dense model of the same total parameter count. |
| **FIM (Fill-in-the-Middle)** | A model capability that predicts missing code given both the text before and after the gap. | Powers IDE autocomplete features where the cursor position is in the middle of existing code. |
| **vLLM** | A high-throughput inference server for open-weight LLMs. | Efficiently serves large models on GPU hardware, enabling production-grade self-hosted deployments. |
| **SWE-bench Verified** | A benchmark measuring an agent's ability to resolve real GitHub issues from open-source repositories. | Industry-standard metric for comparing autonomous coding agents on real software engineering tasks. |
| **HumanEval+** | An extended version of the HumanEval benchmark with additional test cases to reduce false positives. | Evaluates a model's ability to write correct Python functions from docstring descriptions. |
| **LiveCodeBench** | A benchmark of fresh competitive programming problems not included in model training data. | Provides a harder, contamination-free signal of a model's true algorithmic reasoning ability. |
| **Cursor** | A VS Code fork with deep AI integration featuring multi-file agentic editing (Composer) and predictive completions. | The leading AI-native IDE for developers who want agentic editing within a familiar graphical interface. |
| **Windsurf** | A VS Code fork by Codeium that differentiates via Flows (deterministic agentic sequences) and a generous free tier. | Provides a Cursor-like experience with model flexibility and no subscription required for basic use. |
| **OpenHands** | An open-source autonomous coding agent that runs in a Docker sandbox and supports any LLM backend. | Gives teams full control over the agent runtime, model choice, and code isolation without proprietary infrastructure. |
| **Aider** | An open-source, terminal-first coding agent that integrates natively with git, committing changes as it works. | Maintains a clean, reviewable git history as the agent makes changes, making code audit straightforward. |
| **Cline** | An open-source VS Code extension for autonomous coding with full MCP support and per-action permission prompts. | Delivers autonomous coding inside VS Code with no subscription and full model flexibility. |
| **Google Antigravity** | Google's agentic development platform, the successor to the Gemini CLI, with a multi-agent manager UI. | Lets developers delegate coding goals to multiple concurrent agents and review plans and artifacts before execution. |
| **Cascade** | Windsurf's internal name for its agentic editing mode. | The engine that handles multi-file autonomous code changes within the Windsurf IDE. |
| **Composer** | Cursor's multi-file agentic editing feature, analogous to Claude Code's autonomous loop. | Enables Cursor users to give a high-level instruction and have the agent implement changes across multiple files. |
| **Context Map** | Aider's internal representation of the entire codebase structure, even for files not explicitly added to the active context. | Allows Aider to make correct cross-file changes without the user having to manually specify every relevant file. |
| **Domain-Specific Language (DSL)** | A programming language or syntax designed for a specific problem domain within one organization. | Understanding this term motivates fine-tuning open models on internal code, since closed APIs cannot be customized. |
| **Tensor Parallel** | A GPU distribution strategy that splits a model's weight matrices across multiple GPUs so large models fit in memory. | Enables self-hosting of 32B+ parameter models on multi-GPU configurations without quantization. |
| **Regression Rate** | The proportion of agent-generated changes that introduce new failing tests in a previously passing test suite. | A hard failure metric for CI-integrated coding agents; a good agent should never increase the number of failing tests. |

*Previous: [Claude Code](09-claude-code.md) | Next: [Framework Selection Guide](08-framework-selection-guide.md)*
