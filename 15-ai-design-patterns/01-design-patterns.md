# AI Design Patterns

This chapter catalogs common patterns for building AI systems, similar to design patterns in software engineering. Each pattern includes when to use it, implementation guidance, and tradeoffs.

## Table of Contents

- [RAG Patterns](#rag-patterns)
- [Agent Patterns](#agent-patterns)
- [Optimization Patterns](#optimization-patterns)
- [Reliability Patterns](#reliability-patterns)
- [Cost Patterns](#cost-patterns)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## RAG Patterns

### Pattern: Naive RAG

> **Why (the rationale):** Naive RAG gets a working retrieval system in hours with minimal infrastructure, establishing a quality baseline that proves whether retrieval even helps before you invest in a more complex pipeline.
> **When to use:** Use for MVPs, proof-of-concepts, and when your document set is small and relatively uniform in quality; if your initial recall@5 is already above 80%, naive RAG may be sufficient for production.
> **Nuances & gotchas:** Naive RAG silently degrades when the corpus grows or when queries are ambiguous — there is no reranking to rescue a noisy top-K, and the "lost in the middle" effect can bury the most relevant chunk; treat it as a baseline to beat, not a permanent solution.

The simplest RAG implementation:

```
Query → Embed → Search → Top K → Stuff into prompt → Generate
```

**When to use:**
- MVP and prototyping
- Simple question-answering
- When retrieval quality is sufficient

**Limitations:**
- No reranking
- No query enhancement
- May retrieve irrelevant chunks

---

### Pattern: Advanced RAG

> **Why (the rationale):** Query rewriting, hybrid search, and reranking each address a specific failure mode of naive RAG — vocabulary mismatch, keyword vs. semantic retrieval gaps, and noisy top-K results — delivering 10–15% precision gains that directly translate to fewer hallucinations.
> **When to use:** Adopt Advanced RAG when naive RAG's precision is too low for production, when documents are diverse or uneven in quality, or when users ask complex multi-part questions that naive retrieval handles poorly.
> **Nuances & gotchas:** Each added stage adds latency (reranking alone adds 100–200ms) and an additional failure point; the query rewriting LLM call itself costs tokens and can introduce drift — measure end-to-end latency and quality before and after each stage to confirm the tradeoff is worth it.

Enhanced pipeline with multiple stages:

```
Query → Rewrite → Embed → Hybrid Search → Rerank → Filter → Generate
```

```python
class AdvancedRAG:
    async def query(self, user_query: str) -> str:
        # Step 1: Query rewriting
        enhanced_query = await self.rewrite_query(user_query)
        
        # Step 2: Hybrid retrieval
        semantic_results = await self.vector_search(enhanced_query, top_k=50)
        keyword_results = await self.bm25_search(enhanced_query, top_k=50)
        
        # Step 3: Fusion
        combined = self.reciprocal_rank_fusion(semantic_results, keyword_results)
        
        # Step 4: Reranking
        reranked = await self.rerank(enhanced_query, combined[:20])
        
        # Step 5: Generation with top results
        context = self.format_context(reranked[:5])
        return await self.generate(user_query, context)
```

**When to use:**
- Production systems
- When accuracy matters
- Complex document sets

---

### Pattern: Parent-Child Retrieval

> **Why (the rationale):** Small chunks produce higher-precision embedding matches (less noise dilutes the vector) while large chunks give the generator the surrounding context it needs to answer accurately — parent-child indexing lets you have both simultaneously.
> **When to use:** Use when documents have natural hierarchical structure (sections → paragraphs, articles → passages) and when retrieval precision is high but generated answers lack necessary context; also helpful when documents contain dense, inter-dependent information.
> **Nuances & gotchas:** Parent chunks can be very large — if multiple child hits resolve to the same parent, you may end up stuffing the same large block into the context multiple times; deduplicate by parent ID and cap the number of unique parents returned.

Retrieve small chunks, return larger parent chunks:

```
Document
    └── Parent chunk (2000 tokens)
            ├── Child chunk (200 tokens) ← Retrieve on this
            ├── Child chunk (200 tokens)
            └── Child chunk (200 tokens)
```

```python
class ParentChildRetriever:
    def __init__(self, vector_store):
        self.vector_store = vector_store
    
    async def retrieve(self, query: str, top_k: int = 5) -> list[str]:
        # Search on child chunks (more precise)
        child_results = await self.vector_store.search(
            query, 
            collection="child_chunks",
            top_k=top_k * 3
        )
        
        # Get unique parent chunks
        parent_ids = set(r.metadata["parent_id"] for r in child_results)
        
        # Return parent chunks (more context)
        parents = await self.get_parents(list(parent_ids)[:top_k])
        return parents
```

**When to use:**
- Need precision in retrieval
- Need context in generation
- Document structure is hierarchical

---

### Pattern: Self-RAG

> **Why (the rationale):** Not every query needs retrieval — a model asking to retrieve for "What is 2+2?" wastes compute; Self-RAG lets the model skip retrieval when its parametric knowledge is sufficient, and self-critique the answer when it does retrieve.
> **When to use:** Use Self-RAG when your query mix is heterogeneous (some questions require real-time facts, others are general knowledge) and when retrieval latency is a concern; less useful when virtually all queries need retrieval.
> **Nuances & gotchas:** Self-RAG adds multiple LLM calls per query (retrieval assessment, relevance check, support check) — latency and cost multiply accordingly; the model's own judgment about when to retrieve can be miscalibrated, so monitor cases where it skips retrieval and hallucinates.

Model decides when and what to retrieve:

```python
class SelfRAG:
    async def generate(self, query: str) -> str:
        # Step 1: Decide if retrieval is needed
        needs_retrieval = await self.assess_retrieval_need(query)
        
        if needs_retrieval:
            # Step 2: Retrieve
            context = await self.retrieve(query)
            
            # Step 3: Assess relevance
            relevant_context = await self.filter_relevant(query, context)
            
            # Step 4: Generate with context
            response = await self.generate_with_context(query, relevant_context)
            
            # Step 5: Self-critique
            is_supported = await self.check_support(response, relevant_context)
            if not is_supported:
                response = await self.regenerate(query, relevant_context)
        else:
            response = await self.generate_without_context(query)
        
        return response
```

**When to use:**
- Mixed knowledge (parametric + retrieved)
- Want model to be selective
- Research and experimentation

---

### Pattern: Corrective RAG (CRAG)

> **Why (the rationale):** When the local corpus is incomplete or stale, standard RAG silently generates an answer from poor sources; CRAG grades each retrieved document and falls back to web search, making retrieval quality an explicit, correctable signal rather than a hidden failure mode.
> **When to use:** Use CRAG when your document corpus has known gaps (e.g., fast-moving domains, recent events) or when low-quality retrieval causing hallucinations is the dominant quality problem; skip it when the corpus is comprehensive and well-maintained.
> **Nuances & gotchas:** Grading each document adds LLM calls (latency and cost) and the grader itself can be wrong — an ambiguous document marked "irrelevant" can cause CRAG to trigger an unnecessary and slow web search; tune the grading prompt carefully and measure false-negative rates.

Evaluate and correct retrieval quality:

```python
class CorrectiveRAG:
    async def query(self, user_query: str) -> str:
        # Initial retrieval
        docs = await self.retrieve(user_query)
        
        # Grade each document
        graded = []
        for doc in docs:
            grade = await self.grade_relevance(user_query, doc)
            graded.append((doc, grade))
        
        # Categorize results
        relevant = [d for d, g in graded if g == "relevant"]
        ambiguous = [d for d, g in graded if g == "ambiguous"]
        
        if len(relevant) >= 3:
            # Enough relevant docs
            context = relevant
        elif len(relevant) + len(ambiguous) >= 2:
            # Refine ambiguous docs
            refined = await self.refine_search(user_query, ambiguous)
            context = relevant + refined
        else:
            # Web search fallback
            web_results = await self.web_search(user_query)
            context = relevant + web_results
        
        return await self.generate(user_query, context)
```

**When to use:**
- Unreliable document corpus
- Need high accuracy
- Can afford latency for quality checks

---

## Agent Patterns

### Pattern: ReAct

> **Why (the rationale):** Interleaving thought and action lets the model update its plan after each observation — this is critical for multi-step tasks where earlier results should change what the model does next, which pure planning-then-execution cannot handle.
> **When to use:** Use ReAct for general-purpose agents with moderate task complexity, when explainability of decisions matters (the thought steps are auditable), and when tasks may require dynamic replanning mid-execution.
> **Nuances & gotchas:** ReAct agents can get stuck in repetitive thought-action loops when a tool fails repeatedly — always enforce a maximum step limit and monitor for loop patterns; the verbosity of the thought steps also consumes context quickly in long tasks.

Interleaved reasoning and acting:

```
Thought → Action → Observation → Thought → Action → Observation → Answer
```

See [Agent Architectures](../07-agentic-systems/01-agent-fundamentals.md) for implementation.

**When to use:**
- General-purpose agents
- Explainable decision making
- Moderate complexity tasks

---

### Pattern: Plan-and-Execute

> **Why (the rationale):** Separating planning from execution lets you use a powerful model for high-level decomposition and a cheaper or specialized model for each atomic execution step — improving both quality and cost for complex multi-step tasks.
> **When to use:** Use for complex tasks that benefit from upfront decomposition (e.g., research workflows, codebase migrations, multi-document synthesis) where the full task can be reasoned about before any tools are called.
> **Nuances & gotchas:** The initial plan may be wrong or become stale mid-execution — always implement re-planning logic triggered by unexpected tool results; a plan that locks in all steps upfront can be brittle in dynamic environments where early results change the appropriate next action.

Create a plan first, then execute steps:

```python
class PlanAndExecuteAgent:
    async def run(self, task: str) -> str:
        # Step 1: Create plan
        plan = await self.create_plan(task)
        
        # Step 2: Execute each step
        results = []
        for step in plan.steps:
            result = await self.execute_step(step, results)
            results.append(result)
            
            # Re-plan if needed
            if result.needs_replanning:
                plan = await self.replan(task, results)
        
        # Step 3: Synthesize final answer
        return await self.synthesize(task, results)
    
    async def create_plan(self, task: str) -> Plan:
        prompt = f"""
        Create a step-by-step plan to accomplish this task: {task}
        
        Return as JSON:
        {{
            "steps": [
                {{"id": 1, "description": "...", "tool": "..."}},
                ...
            ]
        }}
        """
        return await self.llm.generate(prompt)
```

**When to use:**
- Complex multi-step tasks
- Need visibility into plan
- Tasks benefit from decomposition

---

### Pattern: Critic/Verifier

> **Why (the rationale):** A single LLM generation has an inherent blind spot — it can't reliably catch its own errors from the same weight configuration; a separate critic call (even using the same model) provides a different sampling trajectory that catches a meaningful fraction of mistakes.
> **When to use:** Use when output quality is critical and the task has clear, checkable success criteria (code that must compile, facts that can be verified, reasoning that must be self-consistent); accept the extra latency only when the quality gain justifies it.
> **Nuances & gotchas:** Critics can be sycophantic — they tend to approve the generator's output more often than warranted; use explicit rubric prompts and require the critic to identify at least one issue before approving; also cap iterations to avoid infinite revision loops.

One agent generates, another critiques:

```python
class CriticPattern:
    async def generate_with_critique(self, task: str, max_iterations: int = 3) -> str:
        response = await self.generator.generate(task)
        
        for i in range(max_iterations):
            # Critique the response
            critique = await self.critic.evaluate(task, response)
            
            if critique.is_acceptable:
                break
            
            # Regenerate with feedback
            response = await self.generator.regenerate(
                task, 
                previous=response, 
                feedback=critique.feedback
            )
        
        return response
```

**When to use:**
- Quality is critical
- Can afford extra latency
- Tasks have clear success criteria

---

### Pattern: Hierarchical Agents

> **Why (the rationale):** A single flat agent trying to handle diverse subtasks with one prompt degrades on all of them; specialist workers with focused prompts and tools outperform generalists, and the manager layer enables parallelization that cuts wall-clock time dramatically.
> **When to use:** Use when the task spans multiple clearly distinct domains (research + coding + writing), when subtasks can run in parallel, and when you have the operational maturity to monitor multiple concurrent agent executions.
> **Nuances & gotchas:** The manager's decomposition quality is the system's bottleneck — a bad plan assignment sends work to the wrong worker and results are hard to recover; also, communication between workers through the manager creates a coordination overhead that can slow things down compared to a well-tuned flat agent for simpler tasks.

Manager delegates to specialist workers:

```python
class ManagerAgent:
    def __init__(self):
        self.workers = {
            "research": ResearchAgent(),
            "coding": CodingAgent(),
            "writing": WritingAgent()
        }
    
    async def run(self, task: str) -> str:
        # Decompose task
        subtasks = await self.decompose(task)
        
        # Assign to workers
        results = {}
        for subtask in subtasks:
            worker = self.workers[subtask.worker_type]
            results[subtask.id] = await worker.execute(subtask)
        
        # Synthesize results
        return await self.synthesize(task, results)
```

**When to use:**
- Complex tasks spanning domains
- Different tools per subtask
- Parallelization opportunities

---

## Optimization Patterns

### Pattern: Cascading Models

> **Why (the rationale):** Most query distributions are bimodal — a large fraction of easy queries can be handled by a cheap model, and only a tail requires a frontier model; cascading automatically exploits this distribution to minimize average cost while preserving quality where it matters.
> **When to use:** Use when you can measure or predict query complexity reliably (via a classifier or confidence score) and when the cost delta between tiers is at least 5–10×; don't cascade if most queries end up escalating anyway.
> **Nuances & gotchas:** The complexity classifier is a single point of failure — miscategorizing hard queries as simple silently degrades output quality with no visible error; monitor escalation rate and quality per tier separately, and set a quality threshold below which the cascade aborts and escalates automatically.

Route to cheapest sufficient model:

```python
class ModelCascade:
    def __init__(self):
        self.models = [
            ("gpt-4o-mini", 0.15),     # Cheapest
            ("gpt-4o", 2.50),           # Mid-tier
            ("claude-3.5-sonnet", 3.00) # Most capable
        ]
    
    async def generate(self, query: str) -> str:
        # Classify complexity
        complexity = await self.classify_complexity(query)
        
        if complexity == "simple":
            return await self.call_model("gpt-4o-mini", query)
        elif complexity == "medium":
            return await self.call_model("gpt-4o", query)
        else:
            return await self.call_model("claude-3.5-sonnet", query)
```

**When to use:**
- High query volume
- Variable query complexity
- Cost optimization priority

---

### Pattern: Speculative Execution

> **Why (the rationale):** Large models are slow because they generate one token at a time sequentially; speculative drafting lets a small fast model propose several tokens in parallel, then the large model verifies a whole batch at once — reducing wall-clock time while producing the large model's output quality.
> **When to use:** Use when TTFT and throughput are critical constraints and you have an aligned smaller model available; works best for predictable generation patterns (code, structured output) where the draft model has a high acceptance rate.
> **Nuances & gotchas:** The speedup depends entirely on the draft acceptance rate — if the large model rejects most drafts, the overhead of running two models exceeds the savings; acceptance rates below ~70% typically make speculative decoding net-negative compared to direct large-model generation.

Draft with small model, verify with large:

```python
class SpeculativeExecution:
    async def generate(self, prompt: str, n_tokens: int = 5) -> str:
        output = []
        
        while len(output) < max_tokens:
            # Draft with small model
            draft = await self.draft_model.generate(
                prompt + "".join(output),
                n_tokens=n_tokens
            )
            
            # Verify with large model
            verified = await self.target_model.verify(
                prompt + "".join(output),
                draft
            )
            
            # Accept verified tokens
            output.extend(verified.accepted_tokens)
            
            if verified.is_complete:
                break
        
        return "".join(output)
```

**When to use:**
- Latency-critical applications
- Have aligned draft model
- Predictable generation patterns

---

### Pattern: Caching Layers

> **Why (the rationale):** Repeated or semantically equivalent queries are common in production (FAQ-style questions, repeated analysis of the same document) — caching eliminates the per-token cost and latency of regenerating identical responses, often cutting 20–40% of total spend with no quality impact.
> **When to use:** Use exact-match caching for FAQ or templated queries; layer semantic caching on top when users paraphrase the same intent; only enable caching where staleness is acceptable (not for real-time data or personalized responses).
> **Nuances & gotchas:** Semantic cache threshold tuning is critical — too low (0.85) returns irrelevant cached answers for semantically similar but actually different queries; too high (0.99) misses most paraphrases; test the threshold on real production query pairs before deploying.

Multi-level caching strategy:

```python
class CachingLLM:
    def __init__(self):
        self.exact_cache = ExactMatchCache()
        self.semantic_cache = SemanticCache(threshold=0.95)
    
    async def generate(self, query: str) -> str:
        # Level 1: Exact match
        cached = await self.exact_cache.get(query)
        if cached:
            return cached
        
        # Level 2: Semantic similarity
        similar = await self.semantic_cache.get_similar(query)
        if similar:
            return similar
        
        # Cache miss: Generate
        response = await self.llm.generate(query)
        
        # Store in caches
        await self.exact_cache.set(query, response)
        await self.semantic_cache.set(query, response)
        
        return response
```

**When to use:**
- Repeated similar queries
- Cost reduction priority
- Can tolerate some staleness

---

## Reliability Patterns

### Pattern: Retry with Fallback

> **Why (the rationale):** LLM API errors (rate limits, transient 5xx, timeouts) are frequent enough in production that a single-provider system will have visible user-facing failures; a prioritized fallback list ensures the system stays up even during provider incidents.
> **When to use:** Always implement retry with fallback for any user-facing production system; for internal batch jobs, simple retry with exponential backoff is usually sufficient.
> **Nuances & gotchas:** Provider outputs are not interchangeable — routing the same prompt to OpenAI as a fallback for Anthropic can produce structurally different responses that break downstream parsers; test all fallback paths against your response schema before enabling them.

```python
class RetryWithFallback:
    async def generate(self, query: str) -> str:
        providers = [
            ("openai", "gpt-4o"),
            ("anthropic", "claude-3.5-sonnet"),
            ("google", "gemini-1.5-pro")
        ]
        
        for provider, model in providers:
            try:
                return await self.call(provider, model, query)
            except RateLimitError:
                continue
            except ServiceError:
                continue
        
        # All providers failed
        raise AllProvidersUnavailable()
```

---

### Pattern: Circuit Breaker

> **Why (the rationale):** Without a circuit breaker, a failing provider receives a flood of retry requests that worsen its recovery and add latency for every user; the circuit breaker detects sustained failure and immediately routes traffic to the fallback, giving the primary time to recover.
> **When to use:** Implement alongside retry-with-fallback for any provider that has been a source of cascading failures; configure the failure threshold based on normal error rates — too low and you open the circuit on transient spikes, too high and you fail too many requests before triggering.
> **Nuances & gotchas:** The half-open state (testing if the provider recovered) must be handled carefully — sending a burst of test traffic can re-trigger the failure; probe with a single request per reset interval rather than resuming full traffic.

```python
class CircuitBreaker:
    def __init__(self, failure_threshold: int = 5, reset_timeout: int = 60):
        self.failures = 0
        self.state = "closed"
        self.last_failure = None
        self.failure_threshold = failure_threshold
        self.reset_timeout = reset_timeout
    
    async def call(self, func, *args):
        if self.state == "open":
            if time.time() - self.last_failure > self.reset_timeout:
                self.state = "half-open"
            else:
                raise CircuitOpenError()
        
        try:
            result = await func(*args)
            self.failures = 0
            self.state = "closed"
            return result
        except Exception as e:
            self.failures += 1
            self.last_failure = time.time()
            if self.failures >= self.failure_threshold:
                self.state = "open"
            raise
```

---

### Pattern: Bulkhead

> **Why (the rationale):** Without isolation, a slow or failing component (e.g., a long-running agentic task) can exhaust all available threads and starve faster components (e.g., a simple RAG query), making the entire system appear degraded even though only one workload type is overloaded.
> **When to use:** Use bulkheads whenever you have mixed workload types with very different latency profiles sharing the same infrastructure — at minimum, separate agent workloads (slow, expensive) from RAG workloads (fast, cheap).
> **Nuances & gotchas:** Setting bulkhead sizes requires load testing — too small and you artificially throttle valid traffic; too large and the isolation benefit disappears; start with a ratio proportional to your expected traffic mix and tune from production metrics.

Isolate failures between components:

```python
class BulkheadExecutor:
    def __init__(self, max_concurrent: int = 10):
        self.semaphore = asyncio.Semaphore(max_concurrent)
    
    async def execute(self, func, *args):
        async with self.semaphore:
            return await func(*args)

# Separate bulkheads for different operations
rag_bulkhead = BulkheadExecutor(max_concurrent=20)
agent_bulkhead = BulkheadExecutor(max_concurrent=5)
```

---

## Cost Patterns

### Pattern: Token Budget

```python
class TokenBudget:
    def __init__(self, max_input: int, max_output: int):
        self.max_input = max_input
        self.max_output = max_output
    
    def constrain_input(self, messages: list[dict]) -> list[dict]:
        total_tokens = 0
        constrained = []
        
        for msg in reversed(messages):
            tokens = count_tokens(msg["content"])
            if total_tokens + tokens > self.max_input:
                break
            constrained.insert(0, msg)
            total_tokens += tokens
        
        return constrained
```

---

### Pattern: Cost Tracking Decorator

```python
def track_cost(model: str):
    def decorator(func):
        async def wrapper(*args, **kwargs):
            start_tokens = get_token_count()
            result = await func(*args, **kwargs)
            end_tokens = get_token_count()
            
            cost = calculate_cost(model, end_tokens - start_tokens)
            metrics.record("llm_cost", cost, tags={"model": model})
            
            return result
        return wrapper
    return decorator

@track_cost("gpt-4o")
async def generate_response(query: str):
    return await llm.generate(query)
```

---

## Interview Questions

### Q: Describe three RAG patterns and when to use each.

**Strong answer:**

"I will describe Naive RAG, Advanced RAG, and Parent-Child Retrieval.

**Naive RAG** is the simplest: embed query, search vectors, stuff top K into prompt, generate. I use this for MVPs and when retrieval quality is already good. It is fast to implement but has no reranking or query enhancement.

**Advanced RAG** adds multiple stages: query rewriting, hybrid search (semantic + keyword), reranking, and filtering. I use this in production when accuracy matters. The additional latency (100-200ms for reranking) is worth it for the 10-15% precision improvement.

**Parent-Child Retrieval** embeds small chunks for precise matching but returns larger parent chunks for context. I use this when documents have structure and I need both precision in retrieval and sufficient context for generation.

The pattern I choose depends on the accuracy requirements, latency budget, and document characteristics. I often start with Naive RAG to establish a baseline, then iterate to Advanced RAG."

### Q: What reliability patterns would you use for a production LLM system?

**Strong answer:**

"I implement multiple layers of reliability:

**Retry with exponential backoff** for transient failures. Rate limits and temporary errors are common with LLM APIs.

**Multi-provider fallback** so if OpenAI is having issues, I automatically route to Anthropic or Google. This requires abstracting the LLM interface.

**Circuit breaker** to stop hammering a failing service. After N failures, I open the circuit and route to fallback immediately, giving the primary time to recover.

**Graceful degradation** when all providers fail. Return cached responses, show fallback messages, or queue for later processing rather than erroring.

**Bulkhead isolation** to prevent one component's failures from cascading. Agent workloads get separate thread pools from RAG workloads.

**Timeouts** at every level. LLM calls can hang; I set aggressive timeouts and handle them gracefully.

The key is assuming failures will happen and designing for them rather than hoping they will not."

---

## References

- Gao et al. "Retrieval-Augmented Generation for Large Language Models: A Survey" (2024)
- Yao et al. "ReAct: Synergizing Reasoning and Acting in Language Models" (2023)
- Microsoft Patterns for AI: https://learn.microsoft.com/azure/architecture/patterns/

---

---

## Glossary

| Term | Simple explanation | Purpose |
|---|---|---|
| **RAG (Retrieval-Augmented Generation)** | A pattern where relevant documents are retrieved from a knowledge store and injected into the prompt before the LLM generates a response | Lets the model answer questions using external or up-to-date knowledge it was not trained on |
| **Naive RAG** | The simplest RAG implementation: embed the query, find the top-K nearest chunks, stuff them into the prompt, generate | A fast starting point for prototypes; establishes a retrieval quality baseline before adding complexity |
| **Advanced RAG** | An enhanced pipeline that adds query rewriting, hybrid search, reranking, and filtering before generation | Delivers significantly higher retrieval precision for production systems where accuracy matters |
| **Query Rewriting** | Using an LLM to rephrase the user's question into a form that retrieves better results from the vector index | Bridges the vocabulary gap between how users ask questions and how documents are written |
| **Hybrid Search** | Combining dense vector search (semantic) with sparse keyword search (BM25) and fusing the results | Outperforms either method alone by capturing both meaning and exact keyword matches |
| **BM25** | A classical keyword-ranking algorithm that scores documents by term frequency and inverse document frequency | The standard sparse retrieval baseline; paired with vector search in hybrid RAG pipelines |
| **Reciprocal Rank Fusion (RRF)** | A score-fusion algorithm that merges ranked lists from multiple retrievers into a single ranked list | Combines semantic and keyword results without needing to tune score weights |
| **Reranking** | A second-pass model (cross-encoder) that re-scores the top retrieved chunks for relevance to the query | Improves precision by spending more compute on the shortlist after an initial fast retrieval |
| **Parent-Child Retrieval** | Indexing small child chunks for precise matching but returning their larger parent chunk for richer context | Balances retrieval precision (small chunks) with generation quality (larger context) |
| **Self-RAG** | A pattern where the model itself decides whether to retrieve, then critiques whether the retrieved context actually supports its answer | Makes retrieval selective and reduces hallucination by adding a self-check step |
| **Corrective RAG (CRAG)** | A pattern that grades each retrieved document for relevance and falls back to web search when local results are insufficient | Improves answer reliability when the local corpus may not contain the right information |
| **ReAct** | An agent pattern that interleaves Thought, Action, and Observation steps in a loop until the answer is found | Makes agent reasoning explainable and auditable, and handles moderate-complexity multi-step tasks |
| **Plan-and-Execute** | An agent pattern that creates a full step-by-step plan first, then executes each step, re-planning if results warrant it | Improves performance on complex tasks by separating high-level planning from low-level execution |
| **Critic/Verifier Pattern** | A two-agent design where one agent generates a response and a second agent critiques and rejects or accepts it | Catches errors before they reach the user and iteratively improves output quality |
| **Hierarchical Agents** | A manager-worker architecture where a manager LLM decomposes a task and delegates subtasks to specialist worker agents | Enables parallel execution and domain specialisation across complex, multi-domain tasks |
| **Model Cascade** | Routing queries to progressively more capable (and expensive) models based on estimated complexity | Minimises average cost by using cheap models for simple queries and reserving expensive ones for hard ones |
| **Speculative Execution** | A draft model generates candidate tokens cheaply, which a larger model then verifies and accepts or rejects | Reduces end-to-end latency because the large model only processes a short draft rather than generating from scratch |
| **Draft Model** | The small, fast model used to generate speculative token drafts in speculative decoding | Must be semantically aligned with the target model to achieve high acceptance rates |
| **Target Model** | The large, high-quality model that verifies and selects from the draft's proposed tokens | The authoritative model whose output quality is preserved even though it delegates generation to the draft |
| **Caching Layers** | A multi-level cache (exact match first, then semantic match) in front of the LLM | Reduces cost and latency for repeated or similar queries without any model call |
| **Circuit Breaker** | A reliability component that stops sending requests to a failing service after N consecutive errors and retries after a timeout | Prevents a degraded provider from consuming all retries and introduces fast-fail behaviour |
| **Bulkhead** | An isolation pattern that gives different workloads separate concurrency pools (semaphores) | Prevents a slow or failing component from consuming all available threads and cascading to healthy ones |
| **Retry with Fallback** | Attempting a request against a list of providers in order, catching errors, and moving to the next provider on failure | Provides high availability without manual intervention when one LLM provider is down |
| **Token Budget** | An explicit upper limit on input and output tokens enforced by trimming the message list before each call | Prevents runaway costs and ensures calls complete within the model's context window |
| **Cost Tracking Decorator** | A Python decorator that wraps LLM calls to measure token usage and record the monetary cost per call | Enables per-model, per-feature cost attribution so teams can optimise spending |
| **Parametric Knowledge** | Facts the LLM learned during pre-training and stores in its weights, not retrieved from external sources | Contrasted with retrieved knowledge; Self-RAG decides when to rely on parametric vs. retrieved knowledge |

*Next: [Anti-Patterns to Avoid](02-anti-patterns.md)*
