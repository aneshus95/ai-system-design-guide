# AI Anti-Patterns

Recognizing what NOT to do is as important as knowing best practices. This chapter catalogs common mistakes in AI system design.

## Table of Contents

- [Architecture Anti-Patterns](#architecture-anti-patterns)
- [RAG Anti-Patterns](#rag-anti-patterns)
- [Agent Anti-Patterns](#agent-anti-patterns)
- [Prompting Anti-Patterns](#prompting-anti-patterns)
- [Evaluation Anti-Patterns](#evaluation-anti-patterns)
- [Production Anti-Patterns](#production-anti-patterns)
- [Interview Questions](#interview-questions)

---

## Architecture Anti-Patterns

### The God Prompt

**Problem:** Single massive prompt trying to do everything.

```python
# ANTI-PATTERN: God Prompt
SYSTEM_PROMPT = """
You are a helpful assistant. You can:
1. Answer questions about our products
2. Help with technical support
3. Process refunds
4. Schedule appointments
5. Translate languages
6. Write code
7. Analyze data
8. Generate reports
... [continues for 5000 tokens]
"""
```

**Why it fails:**
- Context consumed by instructions, not user content
- Model struggles with conflicting instructions
- Impossible to optimize for all cases
- Updates affect everything

**Solution:**
```python
# PATTERN: Specialized components
class QueryRouter:
    async def route(self, query: str) -> str:
        intent = await self.classify_intent(query)
        handler = self.handlers[intent]
        return await handler.process(query)
```

---

### Single Provider Dependency

**Problem:** Entire system depends on one LLM provider.

```python
# ANTI-PATTERN: Single provider
async def generate(prompt: str) -> str:
    return await openai.chat.completions.create(...)
```

**Why it fails:**
- Provider outage = complete system failure
- Rate limits affect all traffic
- No price negotiation leverage
- Locked into one model family

**Solution:**
```python
# PATTERN: Multi-provider with failover
class LLMClient:
    def __init__(self):
        self.providers = [OpenAI(), Anthropic(), Google()]
    
    async def generate(self, prompt: str) -> str:
        for provider in self.providers:
            try:
                return await provider.generate(prompt)
            except ProviderError:
                continue
        raise AllProvidersFailedError()
```

---

### Premature Fine-Tuning

**Problem:** Fine-tuning before exhausting simpler approaches.

**Why it fails:**
- Expensive and time-consuming
- Requires quality training data (often unavailable)
- Hard to update and maintain
- Often unnecessary

**Decision flow:**
```
Try prompting first
    ↓ (not working)
Try few-shot examples
    ↓ (not working)
Try RAG for knowledge
    ↓ (not working)
Consider fine-tuning (with 500+ examples)
```

---

## RAG Anti-Patterns

### Retrieve Everything

**Problem:** Retrieving too many documents regardless of relevance.

```python
# ANTI-PATTERN: Retrieve everything
results = vector_db.search(query, top_k=50)
context = "\n".join([r.text for r in results])
```

**Why it fails:**
- Noise drowns out signal
- Exceeds context limits
- Wastes tokens on irrelevant content
- "Lost in the middle" effect

**Solution:**
```python
# PATTERN: Quality over quantity
results = vector_db.search(query, top_k=20)
reranked = await reranker.rerank(query, results)
context = "\n".join([r.text for r in reranked[:5] if r.score > 0.7])
```

---

### No Chunking Strategy

**Problem:** Arbitrary or no chunking of documents.

```python
# ANTI-PATTERN: Fixed-size blind chunking
chunks = [text[i:i+1000] for i in range(0, len(text), 1000)]
```

**Why it fails:**
- Breaks mid-sentence, mid-paragraph
- Loses semantic coherence
- Separates related information
- Poor retrieval quality

**Solution:**
```python
# PATTERN: Semantic-aware chunking
chunks = semantic_chunker.chunk(
    text,
    chunk_size=500,
    overlap=100,
    respect_boundaries=["paragraph", "section"]
)
```

---

### Ignoring Metadata

**Problem:** Treating all documents as equal text.

```python
# ANTI-PATTERN: Ignore metadata
embedding = embed(document.text)
vector_db.insert(embedding, {"text": document.text})
```

**Why it fails:**
- Cannot filter by date, source, type
- No access control per document
- Cannot weight recent vs old
- Loses valuable context

**Solution:**
```python
# PATTERN: Rich metadata
vector_db.insert(embedding, {
    "text": document.text,
    "source": document.source,
    "date": document.date,
    "access_level": document.access_level,
    "document_type": document.type,
    "section": document.section
})

# Filter query
results = vector_db.search(
    query,
    filter={"date": {"$gte": "2024-01-01"}, "access_level": user.level}
)
```

---

## Agent Anti-Patterns

### Infinite Loop Risk

**Problem:** No termination conditions for agents.

```python
# ANTI-PATTERN: No limits
while not done:
    action = await agent.decide_action()
    result = await execute(action)
    done = agent.check_done(result)
```

**Why it fails:**
- Agents can loop forever
- Costs spiral out of control
- Never returns to user
- Resource exhaustion

**Solution:**
```python
# PATTERN: Multiple termination conditions
MAX_STEPS = 20
MAX_COST = 10.0
MAX_TIME = 300  # seconds

for step in range(MAX_STEPS):
    if cost_tracker.total > MAX_COST:
        return "Cost limit reached"
    if time.time() - start > MAX_TIME:
        return "Time limit reached"
    
    action = await agent.decide_action()
    result = await execute(action)
    
    if agent.check_done(result):
        return result
    
return "Step limit reached"
```

---

### Unsafe Tool Access

**Problem:** Giving agents unrestricted tool access.

```python
# ANTI-PATTERN: Full access
tools = [
    delete_file,
    execute_shell_command,
    send_email,
    database_query  # unrestricted!
]
```

**Why it fails:**
- Agent can delete critical files
- Can exfiltrate data
- Can execute malicious commands
- No audit trail

**Solution:**
```python
# PATTERN: Scoped, validated tools
tools = [
    ScopedFileTool(allowed_dirs=["/tmp/agent"]),
    RestrictedShellTool(allowed_commands=["ls", "cat"]),
    EmailTool(requires_confirmation=True),
    ReadOnlyDatabaseTool(allowed_tables=["products"])
]
```

---

### Agent Without Memory

**Problem:** Agent restarts from scratch every turn.

```python
# ANTI-PATTERN: Stateless agent
async def handle_message(message: str) -> str:
    return await agent.run(message)  # No context
```

**Why it fails:**
- Cannot do multi-turn tasks
- Repeats same mistakes
- Cannot learn from experience
- Poor user experience

**Solution:**
```python
# PATTERN: Persistent memory
async def handle_message(session_id: str, message: str) -> str:
    memory = await memory_store.get(session_id)
    response = await agent.run(message, memory=memory)
    await memory_store.update(session_id, memory)
    return response
```

---

## Prompting Anti-Patterns

### Vague Instructions

**Problem:** Ambiguous prompts expecting specific behavior.

```python
# ANTI-PATTERN: Vague
prompt = "Help the user with their request."
```

**Why it fails:**
- "Help" is undefined
- No format specified
- No boundaries
- Inconsistent behavior

**Solution:**
```python
# PATTERN: Specific and structured
prompt = """
You are a customer support agent for TechCorp.

Your role:
- Answer questions about our products
- Help troubleshoot issues
- Escalate to human when unsure

Response format:
1. Acknowledge the issue
2. Provide a solution or ask clarifying questions
3. Offer next steps

Do NOT:
- Make promises about refunds (escalate instead)
- Provide legal or medical advice
- Share internal company information
"""
```

---

### No Output Format

**Problem:** Expecting structured output without specifying format.

```python
# ANTI-PATTERN: Hope for structure
prompt = "Extract the person's name, date, and location from this text."
response = await llm.generate(prompt)
# Response: "The person is John, he was there on March 5th in NYC"
# Now try to parse that...
```

**Solution:**
```python
# PATTERN: Explicit format
prompt = """
Extract information and return as JSON:
{
    "name": "string",
    "date": "YYYY-MM-DD",
    "location": "string"
}

Text: ...
"""
# Or use structured output APIs
response = await llm.generate(prompt, response_format={"type": "json_object"})
```

---

## Evaluation Anti-Patterns

### Vibes-Based Evaluation

**Problem:** "It looks good to me" as the evaluation method.

```python
# ANTI-PATTERN: Manual spot-checking
for i in range(5):
    response = await generate(test_prompts[i])
    print(response)  # Developer looks at it
# "Looks good, ship it!"
```

**Why it fails:**
- Not reproducible
- Cherry-picked examples
- No baseline comparison
- Misses edge cases

**Solution:**
```python
# PATTERN: Systematic evaluation
eval_dataset = load_eval_set()  # 100+ examples
results = []

for example in eval_dataset:
    response = await generate(example["input"])
    score = await evaluate(response, example["expected"])
    results.append(score)

metrics = {
    "accuracy": sum(results) / len(results),
    "failures": [e for e, r in zip(eval_dataset, results) if r < 0.5]
}
```

---

### Training on Test Set

**Problem:** Using evaluation data for development decisions.

```python
# ANTI-PATTERN: Overfitting to eval
for iteration in range(100):
    accuracy = evaluate_on_test_set()  # Same set every time
    tweak_prompt_based_on_failures(test_set)  # Optimizing for test set
```

**Why it fails:**
- Overfits to specific examples
- Real-world performance differs
- No true measure of generalization

**Solution:**
```python
# PATTERN: Proper data splits
dev_set = load_dev_set()      # For iteration
test_set = load_test_set()    # Final evaluation only

# Iterate on dev set
for iteration in range(100):
    accuracy = evaluate(dev_set)
    improve_based_on(dev_set)

# Final evaluation on untouched test set
final_accuracy = evaluate(test_set)
```

---

## Production Anti-Patterns

### No Rate Limiting

**Problem:** Unlimited LLM calls per user.

```python
# ANTI-PATTERN: Open access
@app.route("/generate")
async def generate():
    return await llm.generate(request.prompt)  # No limits!
```

**Why it fails:**
- Single user can exhaust budget
- Denial of service risk
- Cost surprises
- No fair usage

**Solution:**
```python
# PATTERN: Rate limiting
@app.route("/generate")
@rate_limit(requests_per_minute=10, requests_per_day=100)
@cost_limit(max_cost_per_day=1.0)
async def generate():
    return await llm.generate(request.prompt)
```

---

### No Caching

**Problem:** Every identical request hits the LLM.

```python
# ANTI-PATTERN: No cache
async def answer_faq(question: str) -> str:
    return await llm.generate(question)  # Same FAQ, same cost every time
```

**Why it fails:**
- Wasted money on identical queries
- Unnecessary latency
- Inconsistent answers to same question

**Solution:**
```python
# PATTERN: Semantic caching
async def answer_faq(question: str) -> str:
    cached = await cache.get_similar(question, threshold=0.95)
    if cached:
        return cached.response
    
    response = await llm.generate(question)
    await cache.set(question, response)
    return response
```

---

## Interview Questions

### Q: What is the biggest anti-pattern you see in LLM applications?

**Strong answer:**

"The most damaging is the 'God Prompt' anti-pattern: a single massive prompt trying to handle every scenario.

**Why it is common:** It seems simpler to start with one prompt and add instructions as needs arise.

**Why it fails:**
- Context consumed by instructions, not user content
- Conflicting instructions confuse the model
- Cannot optimize for different use cases
- Changes have unpredictable side effects

**The fix:** Route to specialized handlers. Each handler has a focused prompt optimized for one task. The router itself can be simple (keyword-based) or smart (LLM-based for complex cases).

This applies beyond prompts. The general principle is: decompose complexity into specialized components rather than cramming everything into one monolith."

### Q: How do you avoid agent runaway costs?

**Strong answer:**

"Multiple limits at different levels:

**Per-request limits:**
- Maximum steps (e.g., 20)
- Maximum tokens (e.g., 50K)
- Maximum time (e.g., 5 minutes)

**Per-session limits:**
- Daily token budget
- Daily cost cap

**Per-user limits:**
- Rate limiting (requests per minute/hour/day)
- Cost attribution and caps

**Monitoring:**
- Real-time cost tracking
- Alerts for anomalies (single request > $1)
- Circuit breaker if costs spike

**Architecture:**
- Cascade from cheap to expensive models
- Cache common operations
- Batch similar requests

The key is assuming the agent will try to run forever. Build in hard stops at every level. I have seen agents run up $1000 bills in minutes without proper limits."

---

---

## Glossary

| Term | Simple explanation | Purpose |
|---|---|---|
| **God Prompt** | A single massive system prompt that tries to cover every possible use case and role in one block of text | An anti-pattern; knowing it exists helps engineers recognise when to decompose into specialised handlers instead |
| **Query Router** | A component that classifies incoming requests by intent and dispatches each to the appropriate specialised handler | The recommended fix for the God Prompt; keeps each handler's system prompt focused and optimisable |
| **Single Provider Dependency** | Building an application that only works with one LLM vendor, with no failover path | An anti-pattern that creates a single point of failure; multi-provider abstraction is the remedy |
| **Multi-Provider Fallback** | An LLM client abstraction that tries a list of providers in sequence and moves to the next on error | Achieves high availability without manual intervention during provider outages |
| **Premature Fine-Tuning** | Training a custom model before exhausting prompt engineering, few-shot examples, and RAG | An expensive anti-pattern; fine-tuning should be the last resort after simpler approaches fail |
| **Few-Shot Examples** | Including 2-10 example input-output pairs directly in the prompt to guide model behaviour | A cheaper and faster alternative to fine-tuning for improving output format and style |
| **Retrieve Everything** | A RAG anti-pattern where top_k is set very high, flooding the prompt with loosely relevant documents | Causes "lost in the middle" degradation and wastes tokens; quality reranking to a small final set is preferred |
| **Lost in the Middle** | A well-documented LLM failure mode where relevant information buried in the middle of a long context is overlooked | Motivates strict top-k limits and reranking to place the most relevant content at the start of the context |
| **No Chunking Strategy** | Splitting documents by fixed character count without respecting sentence or paragraph boundaries | An anti-pattern that breaks semantic coherence and degrades retrieval quality |
| **Semantic-Aware Chunking** | Splitting text at natural boundaries (sentences, paragraphs, sections) rather than arbitrary character limits | Keeps related ideas together in each chunk, improving both embedding quality and retrieved context |
| **Metadata Filtering** | Using structured attributes (date, source, access level) to pre-filter documents before vector search | Enables access control, date-range queries, and recency weighting that pure semantic search cannot provide |
| **Infinite Loop Risk** | An agent with no maximum step count, cost cap, or timeout that can run indefinitely | The anti-pattern that causes runaway API bills; always pairing an agent loop with hard termination conditions |
| **Termination Conditions** | Explicit limits on steps, cost, and wall-clock time that force an agent loop to stop | The minimum safety net every production agent must implement to prevent cost disasters |
| **Unsafe Tool Access** | Giving an agent unrestricted access to destructive or sensitive tools without scope limits | An anti-pattern that risks data deletion, exfiltration, or arbitrary code execution |
| **Scoped Tool** | A tool implementation that restricts an agent to a subset of operations (e.g., read-only DB, allowed file directories) | The recommended pattern; least-privilege access reduces blast radius if the agent misbehaves |
| **Stateless Agent** | An agent that receives no memory or session context and starts fresh on every turn | An anti-pattern for multi-turn tasks; requires persistent memory to avoid repeating questions and mistakes |
| **Vague Instructions** | A system prompt that uses undefined terms ("help the user") with no format, scope, or constraint | Produces inconsistent, unpredictable behaviour; specific structured prompts are the fix |
| **No Output Format** | Prompting for structured data without specifying the exact format (JSON schema, field names, types) | Makes response parsing unreliable; explicit format instructions or structured-output APIs eliminate the problem |
| **Vibes-Based Evaluation** | Manually spot-checking a handful of outputs and deciding quality is acceptable without metrics or baselines | An anti-pattern that misses regressions and edge cases; systematic eval datasets with scoring are the alternative |
| **Training on Test Set** | Using the evaluation dataset to guide prompt or model improvements, then reporting accuracy on that same set | Causes overfitting to the eval set so reported metrics do not reflect real-world performance |
| **Overfitting** | A model or prompt that performs well on known examples but poorly on unseen inputs | The risk of iterating on a fixed test set; mitigated by using a separate dev set for iteration |
| **No Rate Limiting** | Allowing unlimited LLM API calls per user, exposing the system to cost abuse and denial-of-service | An anti-pattern that can drain budgets in minutes; per-user request and cost caps are mandatory in production |
| **Denial of Service (DoS)** | A scenario where one user or attacker exhausts a shared resource (API quota, budget) for all users | Rate limiting and cost caps per user prevent this from affecting the whole system |
| **Semantic Caching (anti-pattern context)** | Not implementing any form of caching, so identical or equivalent queries always hit the LLM | The "No Caching" anti-pattern highlights that repeated FAQ-style queries waste both money and time |
| **Graceful Degradation** | Returning a cached response, a fallback message, or queueing for later when all LLM providers fail | Preferred over a hard error; keeps the user experience functional even during complete provider outages |
| **Exponential Backoff** | A retry strategy where wait time doubles after each failure to avoid hammering a rate-limited API | Reduces the chance of being permanently rate-limited and plays nicely with provider recovery times |

*Previous: [Design Patterns](01-design-patterns.md)*
