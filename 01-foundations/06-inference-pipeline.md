# Inference Pipeline

This chapter covers how LLMs generate text at inference time, the computational phases involved, and the key metrics for production serving.

## Table of Contents

- [Generation Basics](#generation-basics)
- [Prefill and Decode Phases](#prefill-and-decode-phases)
- [Sampling Strategies](#sampling-strategies)
- [Stopping Conditions](#stopping-conditions)
- [Latent Optimization: Speculative Decoding](#speculative-decoding)
- [Latency Metrics & TTFT vs. TPS](#latency-metrics)
- [Memory and Compute Requirements](#memory-and-compute-requirements)
- [Continuous Batching & Prefix Caching](#continuous-batching-and-prefix-caching)
- [Multi-LoRA Serving](#multi-lora-serving)
- [Streaming](#streaming)
- [Production Considerations](#production-considerations)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## Generation Basics

LLMs generate text autoregressively: one token at a time, using all previous tokens as context.

```
Input: "The quick brown"
Step 1: Generate "fox" -> "The quick brown fox"
Step 2: Generate "jumps" -> "The quick brown fox jumps"
Step 3: Generate "over" -> "The quick brown fox jumps over"
...
```

### The Generation Loop

```python
def generate(prompt: str, max_tokens: int, model) -> str:
    tokens = tokenize(prompt)
    
    for _ in range(max_tokens):
        # Forward pass: get logits for next token
        logits = model.forward(tokens)
        
        # Sample next token from probability distribution
        next_token = sample(logits[-1])
        
        # Check for stop condition
        if next_token == EOS_TOKEN:
            break
        
        tokens.append(next_token)
    
    return detokenize(tokens)
```

---

## Prefill and Decode Phases

Inference has two distinct phases with different characteristics:

### Prefill Phase

Processes the entire input prompt in parallel.

```
Input: "The quick brown fox" (4 tokens)

Prefill:
- Process all 4 tokens simultaneously
- Compute attention across all pairs
- Populate KV cache for all positions
- Output: logits for next token
```

**Characteristics:**
- Compute-bound (lots of matrix operations)
- Parallelizable across tokens
- Time scales with prompt length
- Happens once per generation

### Decode Phase

Generates one token at a time.

```
Decode step 1:
- Input: new token position only
- Attend to all KV cache (prompt + previously generated)
- Generate one token

Decode step 2:
- Append new K, V to cache
- Input: newest token position
- Generate next token

...repeat until done
```

**Characteristics:**
- Memory-bound (loading KV cache from HBM)
- Sequential (must complete each step to start next)
- Time per token roughly constant
- Repeated until stopping condition

### Why This Matters

| Phase | Bottleneck | Optimization |
|-------|------------|--------------|
| Prefill | Compute (GPU cores) | Flash Attention, better GPU |
| Decode | Memory bandwidth | GQA, batching, quantization |

**Implication for serving:**
- Long prompts increase prefill time (affects TTFT)
- Long generations increase decode time (affects total latency)
- Batching helps decode efficiency more than prefill

---

## Sampling Strategies

After computing logits, we need to select the next token. Different strategies produce different outputs.

### Greedy Decoding

Always pick the highest probability token:

```python
def greedy_sample(logits):
    return torch.argmax(logits)
```

**Properties:**
- Deterministic
- Often repetitive for long generations
- Good for factual/structured outputs

### Temperature Sampling

Scale logits before softmax to control randomness:

```python
def temperature_sample(logits, temperature=1.0):
    scaled_logits = logits / temperature
    probs = torch.softmax(scaled_logits, dim=-1)
    return torch.multinomial(probs, num_samples=1)
```

**Temperature effects:**

| Temperature | Behavior | Use Case |
|-------------|----------|----------|
| 0 | Greedy (deterministic) | Factual Q&A, code |
| 0.3-0.7 | Low randomness | General tasks |
| 1.0 | Baseline | Creative writing |
| 1.5+ | High randomness | Brainstorming |

### Top-K Sampling

Only consider the K highest probability tokens:

```python
def top_k_sample(logits, k=50):
    values, indices = torch.topk(logits, k)
    probs = torch.softmax(values, dim=-1)
    sampled_idx = torch.multinomial(probs, num_samples=1)
    return indices[sampled_idx]
```

**Effect:** Filters out low-probability tokens that might be nonsensical.

### Top-P (Nucleus) Sampling

Include tokens until cumulative probability exceeds P:

```python
def top_p_sample(logits, p=0.9):
    sorted_probs, sorted_indices = torch.sort(
        torch.softmax(logits, dim=-1), descending=True
    )
    cumulative_probs = torch.cumsum(sorted_probs, dim=-1)
    
    # Find cutoff
    cutoff_idx = torch.searchsorted(cumulative_probs, p)
    
    # Sample from truncated distribution
    selected_probs = sorted_probs[:cutoff_idx + 1]
    selected_probs = selected_probs / selected_probs.sum()
    sampled_idx = torch.multinomial(selected_probs, num_samples=1)
    
    return sorted_indices[sampled_idx]
```

**Advantage over Top-K:** Dynamically adjusts based on probability distribution. High-confidence predictions include fewer tokens; uncertain predictions include more.

### Common Configurations

| Use Case | Temperature | Top-P | Top-K |
|----------|-------------|-------|-------|
| Code generation | 0-0.2 | 0.95 | - |
| Factual Q&A | 0.1-0.3 | 1.0 | - |
| General chat | 0.7 | 0.9 | - |
| Creative writing | 1.0 | 0.95 | - |
| Brainstorming | 1.2 | 1.0 | - |

### Repetition Penalties

Reduce probability of recently generated tokens:

```python
def apply_repetition_penalty(logits, generated_tokens, penalty=1.2):
    for token_id in set(generated_tokens):
        logits[token_id] /= penalty
    return logits
```

**Variants:**
- Presence penalty: Penalize all tokens that appeared
- Frequency penalty: Penalize proportional to occurrence count

---

## Stopping Conditions

Generation continues until a stopping condition is met:

### EOS Token

Model generates end-of-sequence token:

```python
if next_token == tokenizer.eos_token_id:
    break
```

### Max Tokens

Hard limit on generation length:

```python
for i in range(max_tokens):
    # generate...
```

### Stop Sequences

Custom strings that terminate generation:

```python
stop_sequences = ["###", "\n\n", "Human:"]

for seq in stop_sequences:
    if output.endswith(seq):
        output = output[:-len(seq)]
        break
```

## Latent Optimization: Speculative Decoding

**The current standard for high-bandwidth serving.**

Speculative decoding uses a smaller "draft model" to predict multiple future tokens in a single step, which the larger "target model" then verifies in parallel.

```
Draft Model (Small): Predicts 5 tokens -> "The", "quick", "brown", "fox", "jumps"
Target Model (Large): Verifies all 5 tokens in ONE forward pass.
Result: If target agrees on 4 tokens, we've generated 4 tokens for the cost of 1 large forward pass.
```

| Method | Approach | Speedup | Example |
|--------|----------|---------|---------|
| Draft Model | Small model (e.g., 1B) + Large (70B) | 2x-3x | vLLM, TGI |
| **Medusa Heads** | Multiple LM heads on the same model | 1.5x-2x | Medusa, Eagle |
| Prompt Lookup | Uses substrings from prompt as speculation| 1.2x | RAG / Code completion |

---

## Latency Metrics

### Time to First Token (TTFT)

Time from request to first generated token.

```
TTFT = network_latency + queue_time + prefill_time
```

**What affects TTFT:**
- Prompt length (prefill is O(n))
- Model size
- GPU speed
- Queue depth

**Targets:**
- Interactive chat: < 500ms
- Real-time: < 200ms
- Batch: Less critical

### Tokens Per Second (TPS)

Rate of token generation after first token.

```
TPS = (total_tokens - 1) / (total_time - TTFT)
```

**What affects TPS:**
- Model size
- Batch size
- GPU memory bandwidth
- KV cache size

**Typical values:**
- Llama 70B on H100: 30-50 tokens/sec per request
- GPT-4 via API: 20-80 tokens/sec (varies)
- Small model (7B): 100+ tokens/sec

### Total Latency

```
Total = TTFT + (output_tokens / TPS)
```

**Example:**
- TTFT: 200ms
- TPS: 50 tokens/sec
- Output: 100 tokens
- Total: 200ms + 2000ms = 2.2s

### Throughput

Requests completed per unit time:

```
Throughput = concurrent_requests * TPS / average_output_tokens
```

Higher batch sizes increase throughput but may increase per-request latency.

---

## Memory and Compute Requirements

### Model Weights

```
Memory = parameters * bytes_per_parameter

70B model in FP16:
= 70B * 2 bytes
= 140 GB

70B model in INT4:
= 70B * 0.5 bytes
= 35 GB
```

### KV Cache

```
Per token: 2 * layers * heads * head_dim * bytes
Per request: per_token * sequence_length

Llama 70B (80 layers, 64 heads, 128 dim, FP16):
= 2 * 80 * 64 * 128 * 2 bytes
= 2.6 MB per token

At 4K context: 10.5 GB per request
At 8K context: 21 GB per request
```

### Total GPU Memory

```
Total = model_weights + kv_cache * batch_size + activations

Example: Llama 70B serving
- Weights (INT4): 35 GB
- KV cache (8K, batch 4): 84 GB
- Activations: ~5 GB
- Total: ~124 GB (fits on 2x H100 80GB)
```

### FLOPs per Token

```
Forward pass FLOPs ≈ 2 * parameters

70B model:
≈ 140 TFLOPs per token

At 40 tokens/sec:
≈ 5.6 PFLOPs sustained
```

---

## Streaming

For interactive applications, stream tokens as they are generated:

### Server-Side Events (SSE)

```python
# Server
async def generate_stream(prompt: str):
    for token in model.generate_iter(prompt):
        yield f"data: {json.dumps({'token': token})}\n\n"
    yield "data: [DONE]\n\n"

# Client
async for event in sse_client.stream("/generate"):
    token = json.loads(event.data)["token"]
    display(token)
```

### Benefits

| Aspect | Streaming | Non-streaming |
|--------|-----------|---------------|
| Perceived latency | TTFT only | Full generation time |
| User experience | Progressive | Waiting, then complete |
| Early termination | User can stop | Must wait |
| Memory | Lower | Higher (buffer response) |

### Implementation Details

- Flush after each token
- Handle connection drops gracefully
- Consider buffering for very fast generation
- Some frameworks buffer by default; disable for streaming

---

## Production Considerations

### Batching for Throughput

Combine multiple requests to maximize GPU utilization:

```python
# Without batching: GPU underutilized
for request in requests:
    response = model.generate(request)

# With batching: parallel processing
batch = collect_requests(timeout=10ms, max_batch=32)
responses = model.generate_batch(batch)
```

### Continuous Batching and Prefix Caching

**Continuous Batching (Iteration-level Scheduling):**
Unlike static batching, continuous batching injects new requests as soon as any request in the batch hits an EOS token. This increases throughput by up to 20x.

**Prefix Caching (RAD-O):**
Caches the KV tensors of common prefixes (e.g., system prompts, few-shot examples).
- **TTFT Reduction**: 90%
- **Mechanism**: Use a hash of the prefix to lookup KV tensors in a GPU-memory LRU cache.

### Multi-LoRA Serving

**Scenario:** Serving 1000 different fine-tuned models (adapters) on one base model.
**The Challenge:** Loading 1000 separate models would take terabytes of VRAM.

**The Solution (LoRAX / S-LoRA):**
1. Load one base model in VRAM.
2. Store LoRA adapters (megabytes) in host RAM or SSD.
3. Dynamically swap adapters during the forward pass based on the request ID.
4. **Implementation**: Use a specialized kernel (S-LoRA) that performs matrix-vector multiplication for multiple different adapters in the same batch.

### Request Prioritization

```python
class RequestQueue:
    def __init__(self):
        self.high_priority = asyncio.Queue()
        self.low_priority = asyncio.Queue()
    
    async def get_next(self):
        if not self.high_priority.empty():
            return await self.high_priority.get()
        return await self.low_priority.get()
```

**Priority criteria:**
- Customer tier
- Request type
- Wait time
- Estimated compute cost

### Timeout Handling

```python
async def generate_with_timeout(prompt: str, timeout: float):
    try:
        result = await asyncio.wait_for(
            model.generate(prompt),
            timeout=timeout
        )
        return result
    except asyncio.TimeoutError:
        return {"error": "timeout", "partial": partial_output}
```

### Graceful Degradation

```python
async def generate_with_fallback(prompt: str):
    try:
        return await primary_model.generate(prompt)
    except RateLimitError:
        return await fallback_model.generate(prompt)
    except TimeoutError:
        return await small_fast_model.generate(prompt)
```

### Cost Tracking

```python
@dataclass
class RequestMetrics:
    input_tokens: int
    output_tokens: int
    model: str
    latency_ms: float
    cost_usd: float

def calculate_cost(metrics: RequestMetrics) -> float:
    pricing = {
        "gpt-4o": {"input": 2.50, "output": 10.00},
        "gpt-4o-mini": {"input": 0.15, "output": 0.60},
    }
    rates = pricing[metrics.model]
    return (
        (metrics.input_tokens / 1_000_000) * rates["input"] +
        (metrics.output_tokens / 1_000_000) * rates["output"]
    )
```

---

## Interview Questions

### Q: Explain the difference between prefill and decode phases.

**Strong answer:**
LLM inference has two distinct phases:

**Prefill:**
- Processes the entire input prompt at once
- All tokens attend to each other in parallel
- Populates the KV cache for all prompt positions
- Compute-bound: uses GPU cores efficiently
- Time scales with prompt length

**Decode:**
- Generates one token at a time
- New token attends to all KV cache entries
- Appends new K, V to cache
- Memory-bound: bottlenecked by loading KV cache
- Time per token is roughly constant

This matters for system design because:
- Long prompts increase TTFT (prefill intensive)
- Batching helps decode more than prefill
- Different optimization strategies for each phase

### Q: How do temperature and top-p affect generation?

**Strong answer:**
Both control randomness in token selection:

**Temperature:**
- Scales logits before softmax
- Low (0-0.3): More deterministic, picks high-probability tokens
- High (1.0+): More random, flattens probability distribution
- Zero: Greedy decoding

**Top-p (nucleus sampling):**
- Filters to smallest set of tokens with cumulative probability > p
- Dynamically adjusts cutoff based on distribution
- High confidence: few tokens considered
- Low confidence: many tokens considered

Typical production settings:
- Factual Q&A: temperature 0.1, top-p 0.95
- General chat: temperature 0.7, top-p 0.9
- Creative: temperature 1.0+, top-p 0.95

The key insight is that these work together. Temperature reshapes the distribution; top-p truncates it.

### Q: What determines TTFT vs TPS?

**Strong answer:**
**TTFT (Time to First Token):**
- Network latency to reach the server
- Queue wait time
- Prefill computation time
- Dominated by: prompt length, GPU compute speed

**TPS (Tokens Per Second):**
- Decode phase efficiency
- Memory bandwidth for loading KV cache
- Dominated by: memory bandwidth, batch size, model size

Optimization strategies differ:
- TTFT: Reduce prompt when possible, use faster networking, minimize queueing
- TPS: Increase batch size, use GQA/MQA models, optimize memory access

The tradeoff: batching improves TPS (throughput) but may increase TTFT (latency) if requests wait for batch formation.

### Q: How would you estimate GPU requirements for serving a model?

**Strong answer:**
Three main memory consumers:

1. **Model weights:**
   - FP16: parameters * 2 bytes
   - INT8: parameters * 1 byte
   - INT4: parameters * 0.5 bytes

2. **KV cache:**
   - Per token: 2 * layers * kv_heads * head_dim * 2 bytes (FP16)
   - Per request: per_token * sequence_length
   - Total: per_request * batch_size

3. **Activations:** Typically 5-10% overhead

Example for Llama 70B serving:
- Weights (INT4): 35 GB
- KV cache (8K context, batch 8): 168 GB
- Need: ~200 GB total

Hardware options:
- 3x A100 80GB with tensor parallelism
- 2x H100 80GB with tensor parallelism
- 8x A100 40GB with more parallelism

Then verify throughput meets requirements via benchmarking.

---

## References

- Holtzman et al. "The Curious Case of Neural Text Degeneration" (nucleus sampling, 2020)
- Kwon et al. "Efficient Memory Management for Large Language Model Serving with PagedAttention" (vLLM, 2023)
- [vLLM Documentation](https://docs.vllm.ai/)
- [TensorRT-LLM](https://github.com/NVIDIA/TensorRT-LLM)
- [OpenAI API Documentation](https://platform.openai.com/docs/api-reference)

---

---

## Glossary

| Term | Simple explanation | Purpose |
|---|---|---|
| **Autoregressive Generation** | Producing one token at a time, feeding each generated token back as input for the next step | The fundamental mechanism of all decoder-only LLMs; enables open-ended text generation |
| **Inference** | Running a trained model on new inputs to produce outputs (as opposed to training) | The production workload; all serving, cost, and latency concerns stem from inference efficiency |
| **Prefill Phase** | The one-time parallel processing of all input prompt tokens to populate the KV cache | Compute-bound; determines Time to First Token; scales with prompt length |
| **Decode Phase** | The sequential per-token generation loop after prefill, loading the KV cache on each step | Memory-bandwidth-bound; determines tokens-per-second throughput |
| **KV Cache** | Stored Key and Value tensors from all prior positions reused on each decode step without recomputation | Reduces per-step KV projection cost from O(n) to O(1); dominant GPU memory consumer during serving |
| **HBM (High Bandwidth Memory)** | The main GPU VRAM (~80 GB on H100) where weights and KV cache reside | Loading KV cache from HBM is the bottleneck in decode; motivates GQA, MQA, and PagedAttention |
| **Logits** | Raw unnormalized scores from the LM head, one per vocabulary token, before probability conversion | Input to all sampling strategies; their distribution shapes which tokens are likely to be sampled |
| **Greedy Decoding** | Always selecting the single highest-probability token at each step | Deterministic and fast; tends to produce repetitive outputs for long generations |
| **Temperature Sampling** | Dividing logits by a temperature value before softmax to control randomness | Low temperature → near-deterministic; high temperature → more diverse but potentially incoherent |
| **Temperature** | A scalar divisor applied to logits before softmax; controls the sharpness of the probability distribution | Key hyperparameter for generation quality; typically 0–1.5 depending on use case |
| **Top-K Sampling** | Restricting sampling to only the K tokens with the highest probability | Prevents the model from selecting very unlikely tokens while preserving variety |
| **Top-P (Nucleus) Sampling** | Including only the smallest set of tokens whose cumulative probability exceeds P | Dynamically adjusts candidate set size; fewer tokens when confident, more when uncertain |
| **Greedy Decoding** | Always picking the argmax token; equivalent to temperature = 0 | Deterministic baseline; best for structured outputs like JSON or code where creativity is unwanted |
| **Repetition Penalty** | Reducing the logit score of tokens that have already appeared in the generated sequence | Discourages looping or repetitive outputs in long generations |
| **Presence Penalty** | A flat penalty applied to any token that has appeared at least once in the output | Encourages the model to introduce new topics rather than dwelling on already-mentioned ones |
| **Frequency Penalty** | A penalty scaled by how many times a token has appeared in the output | Stronger deterrent for very repetitive tokens; proportional to occurrence count |
| **EOS Token** | End-of-Sequence special token; when generated, signals the model has finished its response | Primary stopping condition for generation; model learns to produce it at natural endpoints |
| **Stop Sequences** | Custom strings (e.g., "###", "Human:") that halt generation when detected in the output | Enables structured generation; used to prevent the model from generating beyond its intended turn |
| **Speculative Decoding** | Using a small draft model to propose multiple tokens, then verifying them in parallel with the large model | Achieves 2–3× speedup without changing output distribution; current standard for high-throughput serving |
| **Draft Model** | The small, fast model in speculative decoding that proposes candidate token sequences | Must be fast enough that verification saves net time; often a 1B model paired with a 70B model |
| **Target Model** | The large model in speculative decoding that verifies the draft's proposed tokens in one forward pass | Accepts any token the draft proposed that it would have generated itself; rejects divergences |
| **Medusa Heads** | Multiple auxiliary LM heads attached to the same model that each predict a future token position | Enables speculative decoding without a separate draft model; 1.5–2× speedup |
| **TTFT (Time to First Token)** | Elapsed time from request submission until the first generated token is received by the client | Primary latency metric for interactive applications; dominated by prefill and queue wait |
| **TPS (Tokens Per Second)** | Rate of token generation after the first token, measuring decode throughput | Key throughput metric; dominated by GPU memory bandwidth and batch size |
| **Throughput** | Number of tokens or requests completed per second across all concurrent users | System-level efficiency metric; maximized by batching multiple requests together |
| **Batching** | Processing multiple inference requests simultaneously in one GPU forward pass | Increases GPU utilization and throughput; trades off slightly higher per-request latency |
| **Continuous Batching** | Injecting new requests into an in-progress batch as soon as any request finishes (iteration-level scheduling) | Up to 20× throughput improvement over static batching; standard in vLLM and TGI |
| **Static Batching** | Waiting for a full batch to form before starting computation, then waiting for all requests to finish | Simple but wastes GPU time when requests in a batch have very different lengths |
| **PagedAttention** | Storing KV cache in non-contiguous virtual memory pages, similar to OS page tables | Cuts VRAM fragmentation from ~60–80% to under 4%; enables much larger effective batch sizes in vLLM |
| **Prefix Caching** | Caching the KV tensors of common prompt prefixes (e.g., system prompts) across requests using a hash lookup | Reduces TTFT by 90% and cost by 50–90% for requests sharing a long common prefix |
| **LoRA (Low-Rank Adaptation)** | Fine-tuning technique that adds small trainable low-rank matrices to frozen base model weights | Enables parameter-efficient fine-tuning; adapters are megabytes, not gigabytes |
| **Multi-LoRA Serving** | Dynamically swapping different LoRA adapters during inference while keeping one base model in VRAM | Serves hundreds of fine-tuned model variants without duplicating the full model in memory |
| **S-LoRA / LoRAX** | Specialized serving systems that batch multiple different LoRA adapters in the same forward pass | Enables efficient multi-tenant fine-tuned model serving at scale |
| **Streaming (SSE)** | Sending each generated token to the client immediately via Server-Sent Events rather than buffering the full response | Reduces perceived latency to TTFT; enables the user to start reading before generation completes |
| **FLOPs per Token** | The number of floating-point operations needed for one inference step; approximately 2 × parameter count | Used to estimate inference latency and choose appropriate hardware |
| **INT4 Quantization** | Representing model weights in 4-bit integers to reduce memory by 4× versus FP16 | Makes large models like Llama 70B fit on fewer GPUs; enables practical serving with modest quality loss |
| **Tensor Parallelism** | Splitting weight matrices across multiple GPUs so computation is distributed within a single layer | Required when a single model layer is too large for one GPU's VRAM |
| **vLLM** | An open-source LLM serving framework implementing PagedAttention and continuous batching | De-facto standard for high-throughput open-source LLM serving |
| **TGI (Text Generation Inference)** | HuggingFace's serving framework supporting continuous batching and quantization | Alternative to vLLM; widely used in HuggingFace-ecosystem deployments |
| **TensorRT-LLM** | NVIDIA's optimized inference library for running LLMs on NVIDIA hardware | Highest-performance option for NVIDIA GPUs; supports FP8 and custom CUDA kernels |
| **Graceful Degradation** | Falling back to a smaller or faster model when the primary model is unavailable or overloaded | Maintains availability during traffic spikes or outages at the cost of lower quality |
| **Request Prioritization** | Ordering queued requests by customer tier, wait time, or estimated cost before scheduling | Ensures SLA guarantees for high-priority users while fairly serving lower-priority traffic |
| **Cost per Token** | The dollar price charged per million input or output tokens by an API provider | Primary economic variable in LLM system design; drives model selection and optimization decisions |

*Previous: [Embeddings and Vector Spaces](05-embeddings-and-vector-spaces.md) | Next: [Model Taxonomy](../02-model-landscape/01-model-taxonomy.md)*
