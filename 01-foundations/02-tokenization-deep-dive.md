# Tokenization Deep Dive

Tokenization is the process of converting text into discrete units (tokens) that models can process. It directly impacts model capabilities, costs, and performance.

## Table of Contents

- [Why Tokenization Matters](#why-tokenization-matters)
- [Tokenization Algorithms](#tokenization-algorithms)
- [Vocabulary Design Tradeoffs](#vocabulary-design-tradeoffs)
- [Special Tokens](#special-tokens)
- [Multilingual Tokenization](#multilingual-tokenization)
- [Token Counting for Cost Estimation](#token-counting-for-cost-estimation)
- [Common Tokenization Issues](#common-tokenization-issues)
- [Practical Tokenization Patterns](#practical-tokenization-patterns)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## Why Tokenization Matters

### For System Design

1. **Cost**: LLM APIs charge per token. Tokenization efficiency directly affects costs.
2. **Context limits**: Token count, not word count, determines what fits in context.
3. **Capability**: Some tasks (character counting, anagrams) are hard because of tokenization.
4. **Consistency**: Same text tokenizes differently across models.

### For Understanding LLM Behavior

**Classic interview question**: Why does GPT struggle to count letters in "strawberry"?

Because "strawberry" is tokenized as multiple subwords. The model never sees individual characters; it sees subword units. Counting letters requires reasoning about internal structure of tokens.

---

## Tokenization Algorithms

### Byte Pair Encoding (BPE)

The most common algorithm. Used by GPT-series, Llama, Claude.

**Training algorithm:**
1. Start with vocabulary of individual bytes (256 tokens)
2. Count all adjacent token pairs in training corpus
3. Merge most frequent pair into a new token
4. Repeat until vocabulary size reached

**Example:**
```
Corpus: "low lower lowest"
Initial: ['l', 'o', 'w', ' ', 'l', 'o', 'w', 'e', 'r', ' ', 'l', 'o', 'w', 'e', 's', 't']

Step 1: Most frequent pair is ('l', 'o'). Merge to 'lo'.
['lo', 'w', ' ', 'lo', 'w', 'e', 'r', ' ', 'lo', 'w', 'e', 's', 't']

Step 2: Most frequent pair is ('lo', 'w'). Merge to 'low'.
['low', ' ', 'low', 'e', 'r', ' ', 'low', 'e', 's', 't']

Step 3: Most frequent pair is ('low', 'e'). Merge to 'lowe'.
['low', ' ', 'lowe', 'r', ' ', 'lowe', 's', 't']

Continue until vocabulary size target...
```

**Properties:**
- Deterministic tokenization given trained vocabulary
- Common words tend to be single tokens
- Rare words split into subwords

#### In plain English

**The analogy — inventing shorthand.** Imagine you're taking notes and notice you keep writing "l-o-w" over and over. To save effort, you invent a single shorthand squiggle for "low." Then you notice "low" is often followed by "e-r", so you invent another squiggle for "lower." BPE does exactly this: it starts with only single characters and keeps **gluing together the pair it sees most often**, growing longer tokens for whatever is common. Frequent words end up as one token; rare words stay in pieces.

**Visual — merges build bottom-up:**

```
Start:   l  o  w  e  r          (every character is its own token)
              │
merge "l"+"o" (most frequent pair)
              ▼
         lo  w  e  r
              │
merge "lo"+"w"
              ▼
         low  e  r
              │
merge "low"+"e", then "lowe"+"r" ...
              ▼
         lower                  (now a single token)

Rule: repeatedly glue the MOST FREQUENT adjacent pair → new token.
```

The final vocabulary is the ladder of merges it learned. At inference time it just re-applies those merges in order. ([HuggingFace: tokenizer algorithms](https://huggingface.co/docs/transformers/tokenizer_summary))

### WordPiece

Used by BERT-family models.

**Key difference from BPE:**
- BPE: Merge based on frequency
- WordPiece: Merge based on likelihood improvement

```
Score = freq(AB) / (freq(A) * freq(B))
```

This favors merges that are more meaningful than random co-occurrence.

**Visual marker:** WordPiece uses ## prefix for continuation tokens:
```
"embedding" becomes ["em", "##bed", "##ding"]
```

#### In plain English

**The analogy — merge by "surprise," not by count.** BPE merges the *most frequent* pair. WordPiece is pickier: it merges the pair that is **unusually attracted to each other** — appearing together far more than their individual popularity would predict. Think of it as asking "do these two *belong* together, or do they just both happen to be common?"

Example: `play` and `##ing`. Both are common on their own, so BPE might merge other frequent-but-boring pairs first. WordPiece notices that `##ing` follows `play` *way* more than chance predicts — that surplus is a signal they form a real unit — so it merges them. The scoring formula captures exactly this "togetherness beyond chance":

```
            freq(A, B)                    ← how often they actually appear together
score  =  ─────────────────
          freq(A) × freq(B)               ← how often you'd EXPECT them together by chance

High score  → they belong together (merge!)     e.g. play + ##ing
Low score   → just two common tokens near each other (skip)
```

BPE would only know "playing is frequent." WordPiece additionally knows "play and ##ing are magnetically attracted." ([WordPiece explained](https://mbrenndoerfer.com/writing/wordpiece-tokenization-bert-subword-algorithm))

### Unigram (SentencePiece)

Used by T5, ALBERT, some multilingual models.

**Training algorithm:**
1. Start with large candidate vocabulary
2. Compute loss if each token were removed
3. Remove tokens that increase loss least
4. Repeat until vocabulary size reached

**Key difference:** Works with probabilities rather than frequencies. Can recover from suboptimal early merges.

#### In plain English

**The analogy — sculpting, not building.** BPE and WordPiece **build up** from characters by gluing pieces together. Unigram works the *opposite way*: it starts with a giant block containing *every* plausible substring, then **carves away** the tokens that turn out to be least useful — like a sculptor removing marble until only the meaningful shape remains. On each round it asks, for every token, "if I deleted you, how much worse would my model explain the text?" and prunes the ones nobody would miss.

**Visual — pruning top-down:**

```
BPE / WordPiece  (bottom-up: build)      Unigram  (top-down: carve)

  characters                               HUGE vocab (all substrings)
     │  merge                                 │  prune least-useful
     ▼                                         ▼
  subwords                                  smaller vocab
     │  merge                                 │  prune
     ▼                                         ▼
  bigger tokens                             TARGET vocab

  "grow toward the target"                 "shrink toward the target"
```

Because it evaluates the whole vocabulary probabilistically, Unigram can keep a token that an early greedy merge would have destroyed — and it can even offer *multiple* valid segmentations of the same word (useful for "subword regularization"). ([Unigram tokenization explained](https://mbrenndoerfer.com/writing/unigram-language-model-tokenization))

### Comparison

| Algorithm | Direction | Merge/Prune Criterion | One-line intuition | Used By |
|-----------|-----------|-----------------------|--------------------|---------|
| BPE | Bottom-up (build) | Most **frequent** pair | "Glue the pair I see most" | GPT, Llama, Claude |
| WordPiece | Bottom-up (build) | Highest **likelihood** gain | "Glue the pair that *belongs* together" | BERT, DistilBERT |
| Unigram | Top-down (carve) | Remove lowest **probability** loss | "Start with everything, trim the useless" | T5, mT5, XLNet |

**Visual — all three targeting the same vocab size from different directions:**

```
        BPE ─────────►┐
                       │  (build up from characters)
   WordPiece ─────────►┤
                       ├──►  ~32K–200K token vocabulary
     Unigram ◄─────────┘
                       (carve down from all substrings)
```

---

## Vocabulary Design Tradeoffs

### Vocabulary Size

| Size | Example | Pros | Cons |
|------|---------|------|------|
| Small (10K) | Some early models | Smaller embeddings | Long token sequences |
| Medium (32K) | Llama 2 | Good balance | Multilingual inefficiency |
| Large (128K) | Llama 3/4, Claude Sonnet 4.6, Mistral Medium 3.5 | **Current standard.** High compression ratio. | Larger embeddings table |
| Huge (200K+) | GPT-5.5 (o200k), Claude Opus 4.7 | Native multimodal and multilingual efficiency | Memory pressure at the LM Head |

**The vocab-expansion deep dive:**
- **Llama 3/4 (128k)**: By moving from 32k to 128k, Meta improved English compression by ~15% and non-English languages like Hindi by 3-4x. 
- **GPT-4o/5.2 (o200k_base)**: Tiktoken's latest encoding provides superior compression for code and multilingual text, reducing API costs indirectly by using fewer tokens for the same meaning.

### Character vs Subword vs Word

| Granularity | Example | Tokens for "running" | Tradeoffs |
|-------------|---------|---------------------|-----------|
| Character | ByT5 | ['r','u','n','n','i','n','g'] | Handles any text but very long sequences |
| Subword | GPT | ['running'] or ['run','ning'] | Good balance |
| Word | Early NLP | ['running'] | Short sequences but cannot handle OOV |

Modern LLMs universally use subword tokenization for the balance of vocabulary size and sequence length.

### Byte-Level BPE

GPT-2 introduced byte-level BPE:
- Base vocabulary is 256 bytes, not characters
- Can represent any text without UNK tokens
- Unicode handled naturally as byte sequences

```python
# Character-level: Needs explicit handling of characters
text = "cafe"  # Unknown character might become [UNK]

# Byte-level: Works with any text (no UNK needed)
text = "cafe"  # Becomes bytes, then BPE operates on bytes
```

---

## Special Tokens

Special tokens handle structural information outside normal text:

| Token | Purpose | Example |
|-------|---------|---------|
| BOS | Beginning of sequence | Signals start of generation |
| EOS | End of sequence | Signals completion |
| PAD | Padding | Fill batches to equal length |
| UNK | Unknown token | Fallback for OOV (rare with byte BPE) |
| SEP | Separator | Divide segments (BERT-style) |

### Chat Templates

Modern chat models use special tokens for conversation structure:

**Llama 2 format:**
```
[INST] <<SYS>>
You are a helpful assistant.
<</SYS>>

User message here [/INST] Assistant response here
```

**ChatML (OpenAI style):**
```
<|im_start|>system
You are a helpful assistant.<|im_end|>
<|im_start|>user
Hello!<|im_end|>
<|im_start|>assistant
Hi there!<|im_end|>
```

**Why this matters:**
- Wrong formatting leads to poor results
- Special tokens are not in pre-training data
- Libraries like transformers use chat_template for automatic formatting

---

## Multilingual Tokenization

### The Challenge

Tokenizers trained primarily on English have poor efficiency for other languages:

| Language | Tokens for "Hello" | Tokens for equivalent greeting |
|----------|-------------------|-------------------------------|
| English | 1 ("Hello") | - |
| Chinese | - | 2-3+ for equivalent |
| Japanese | - | 3-5+ for equivalent |
| Korean | - | 2-4+ for equivalent |

**Cost implication:** Non-English users pay 2-3x more per semantic unit.

### Solutions

1. **Multilingual training corpus:** Train tokenizer on balanced multilingual data
2. **Larger vocabulary:** More room for non-English tokens
3. **Language-specific tokenizers:** Separate tokenizers per language family

**Models with good multilingual support:**
- mT5, XLM-R: Trained on 100+ languages
- GPT-4, Claude 3.5: Large vocabulary with multilingual coverage
- Gemini: Designed for multilingual from the start

| Model | Chinese | Japanese | Korean | Hindi |
|-------|---------|----------|--------|--------|
| GPT-2 | 2.5x | 3.0x | 2.8x | 6.0x |
| GPT-4 (cl100k) | 1.4x | 1.6x | 1.5x | 3.2x |
| GPT-5.2 (o200k) | 1.1x | 1.2x | 1.1x | 1.4x |
| Llama 3/4 (128k)| 1.2x | 1.3x | 1.2x | 1.5x |

---

## Multimodal Tokenization (pixels-to-tokens)

Modern native multimodal models do not just "see" images; they tokenize them.

### Image Tokenization (Vision Transformers)
Images are split into patches (e.g., 14x14 pixels). Each patch is passed through a vision encoder (like SigLIP) to produce a single visual token.
- **Fixed Token Cost**: Most models use a fixed number of tokens per image at a specific resolution (e.g., 256 or 729 tokens per image).
- **Dynamic Resolution**: Some models (Gemini 3) use a variable number of tokens depending on image aspect ratio and detail level.

### Audio/Video Tokenization
- **Audio**: Compressed into discrete units using codecs like EnCodec, then represented as a sequence of audio tokens.
- **Video**: Treated as a sequence of image frames (temporal tokenization). A 1-second video @ 1FPS might cost as much as 1 high-res image.

---

## Token Counting for Cost Estimation

### Quick Estimation Rules

For English text:
- **Words to tokens:** ~1.3 tokens per word
- **Characters to tokens:** ~4 characters per token
- **Pages to tokens:** ~500-800 tokens per page

```python
def estimate_tokens(text: str) -> int:
    # Rough estimation for English
    word_count = len(text.split())
    return int(word_count * 1.3)
```

### Accurate Counting

Use the model-specific tokenizer:

```python
import tiktoken

# For OpenAI models
encoding = tiktoken.encoding_for_model("gpt-4")
tokens = encoding.encode("Your text here")
token_count = len(tokens)

# For Llama/Anthropic, use transformers
from transformers import AutoTokenizer
tokenizer = AutoTokenizer.from_pretrained("meta-llama/Llama-2-7b")
tokens = tokenizer.encode("Your text here")
token_count = len(tokens)
```

### Cost Calculation

```python
def calculate_cost(input_text: str, output_text: str, model: str) -> float:
    pricing = {
        "gpt-4o": {"input": 2.50, "output": 10.00},  # per 1M tokens
        "gpt-4o-mini": {"input": 0.15, "output": 0.60},
        "claude-3.5-sonnet": {"input": 3.00, "output": 15.00},
    }
    
    encoding = tiktoken.encoding_for_model(model)
    input_tokens = len(encoding.encode(input_text))
    output_tokens = len(encoding.encode(output_text))
    
    cost = (
        (input_tokens / 1_000_000) * pricing[model]["input"] +
        (output_tokens / 1_000_000) * pricing[model]["output"]
    )
    return cost
```

---

## Common Tokenization Issues

### Issue 1: Token Boundary Misalignment

**Problem:** Text operations may not align with token boundaries.

```python
text = "Hello world"
# Tokens: ["Hello", " world"]  # Note: space is part of second token

# Truncating at character 6 ("Hello ") splits a token
```

**Solution:** Always truncate at token boundaries when managing context.

### Issue 2: Inconsistent Tokenization

**Problem:** Same text tokenizes differently based on context.

```python
# GPT tokenizer example
"New York"     # Might be ["New", " York"]
"NewYork"      # Might be ["New", "York"]
" New York"    # Might be [" New", " York"]
```

**Implication:** Token counts can vary based on surrounding text. Always tokenize the full context.

### Issue 3: Code and Structured Data

**Problem:** Code and JSON often tokenize inefficiently.

```python
# Python code often tokenizes poorly
"def calculate_average(numbers):"
# Becomes many tokens: ["def", " calculate", "_", "average", "(", "numbers", "):", ...]

# JSON keys tokenize individually
'{"firstName": "John"}'
# Many tokens for structure
```

**Mitigation:** 
- Some models have code-optimized tokenizers
- Consider compressing JSON before sending
- Use structured output modes when available

### Issue 4: Whitespace Handling

**Problem:** Tokenizers handle whitespace differently.

```python
# Leading spaces often become separate tokens
" Hello"  # [" ", "Hello"] or [" Hello"]

# Multiple spaces may merge or stay separate
"Hello  world"  # Behavior varies by tokenizer
```

**Best practice:** Normalize whitespace before tokenizing.

---

## Practical Tokenization Patterns

### Pattern 1: Context Window Management

```python
def fit_to_context(
    system_prompt: str,
    user_message: str,
    history: list[str],
    max_tokens: int = 8000,
    reserve_for_output: int = 2000
) -> str:
    encoding = tiktoken.encoding_for_model("gpt-4")
    
    available = max_tokens - reserve_for_output
    
    # System prompt always included
    tokens_used = len(encoding.encode(system_prompt))
    available -= tokens_used
    
    # User message always included
    tokens_used = len(encoding.encode(user_message))
    available -= tokens_used
    
    # Add history from most recent, drop oldest if needed
    included_history = []
    for msg in reversed(history):
        msg_tokens = len(encoding.encode(msg))
        if msg_tokens <= available:
            included_history.insert(0, msg)
            available -= msg_tokens
        else:
            break
    
    return format_prompt(system_prompt, included_history, user_message)
```

### Pattern 2: Chunking at Token Boundaries

```python
def chunk_at_token_boundaries(
    text: str,
    chunk_size: int = 500,
    overlap: int = 50
) -> list[str]:
    encoding = tiktoken.encoding_for_model("gpt-4")
    tokens = encoding.encode(text)
    
    chunks = []
    start = 0
    while start < len(tokens):
        end = min(start + chunk_size, len(tokens))
        chunk_tokens = tokens[start:end]
        chunk_text = encoding.decode(chunk_tokens)
        chunks.append(chunk_text)
        start = end - overlap
    
    return chunks
```

### Pattern 3: Token Budget Allocation

```python
class TokenBudget:
    def __init__(self, total: int):
        self.total = total
        self.allocated = {}
    
    def allocate(self, component: str, tokens: int) -> bool:
        used = sum(self.allocated.values())
        if used + tokens > self.total:
            return False
        self.allocated[component] = tokens
        return True
    
    def remaining(self) -> int:
        return self.total - sum(self.allocated.values())

# Usage
budget = TokenBudget(total=8000)
budget.allocate("system_prompt", 500)
budget.allocate("retrieved_context", 2000)
budget.allocate("user_message", 200)
budget.allocate("output_reserve", 2000)
# Remaining: 3300 tokens for conversation history
```

---

## Interview Questions

### Q: Why does GPT-4 struggle with simple character counting?

**Strong answer:**
Tokenization converts text to subword units, not characters. When asked "How many 'r's in strawberry?", the model sees tokens like ["str", "aw", "berry"] rather than individual letters.

The model has to reason about the internal structure of tokens it does not directly observe. This requires memorizing or computing character compositions of tokens, which is an emergent capability that is not always reliable.

The solution is to prompt the model to spell out the word character by character first, then count. This forces the creation of character-level tokens.

### Q: How would you estimate token count for cost planning?

**Strong answer:**
For rough estimation: multiply word count by 1.3 for English text.

For accurate counting: Use the model-specific tokenizer.
- OpenAI: tiktoken library
- Others: transformers AutoTokenizer

Important considerations:
- Non-English text uses 1.5-3x more tokens
- Code and structured data tokenize inefficiently
- Always budget extra for output tokens (typically priced higher)
- Include system prompts and formatting tokens

For production cost estimation, I sample real requests and measure actual token usage, then apply safety margins.

### Q: What happens when switching tokenizers between models?

**Strong answer:**
Every model family has its own tokenizer. You cannot reuse tokens across models because:

1. **Vocabulary differs:** Token IDs mean different strings
2. **Merge rules differ:** Same text splits differently
3. **Special tokens differ:** Chat formatting varies

Practical implications:
- Always use the correct tokenizer for token counting
- Cached embeddings are model-specific
- Prompt templates need per-model adjustment
- Fine-tuned models inherit their base tokenizer

### Q: How do you handle tokenization for RAG chunking?

**Strong answer:**
Key considerations:

1. **Chunk at token boundaries:** Splitting mid-token corrupts text when decoded
2. **Account for template tokens:** System prompt, formatting consume tokens
3. **Leave headroom:** Retrieved chunks plus question must fit context

Implementation approach:
```python
# Determine available tokens for chunks
available = max_context - system_prompt_tokens - question_tokens - output_reserve

# Chunk with overlap at token boundaries
chunks = chunk_at_token_boundaries(document, chunk_size=500, overlap=50)

# Select chunks until budget exhausted
selected = []
tokens_used = 0
for chunk in ranked_chunks:
    chunk_tokens = count_tokens(chunk)
    if tokens_used + chunk_tokens <= available:
        selected.append(chunk)
        tokens_used += chunk_tokens
```

---

## References

- Sennrich et al. "Neural Machine Translation of Rare Words with Subword Units" (BPE, 2016)
- Wu et al. "Google's Neural Machine Translation System" (WordPiece, 2016)
- Kudo and Richardson "SentencePiece: A simple and language independent subword tokenizer" (2018)
- OpenAI tiktoken library: https://github.com/openai/tiktoken
- HuggingFace tokenizers: https://github.com/huggingface/tokenizers

---

---

## Glossary

| Term | Simple explanation | Purpose |
|---|---|---|
| **Token** | The basic unit a language model processes — can be a word, part of a word, or a single character | Every LLM input and output is measured in tokens; they determine cost and context usage |
| **Tokenizer** | The program that converts raw text into a sequence of token IDs the model can read | Must be run on every input before calling a model and on every output to decode results |
| **Byte Pair Encoding (BPE)** | A bottom-up algorithm that repeatedly merges the most frequent adjacent token pair until a target vocabulary size is reached | Produces compact, frequency-driven subword vocabularies; used by GPT, Llama, and Claude |
| **WordPiece** | A bottom-up algorithm like BPE but merges pairs based on likelihood gain rather than raw frequency | Captures linguistically meaningful units more reliably than pure frequency; used in BERT |
| **Unigram (SentencePiece)** | A top-down algorithm that starts with all possible substrings and prunes the least useful ones | Can produce multiple valid segmentations of the same word; used in T5 and multilingual models |
| **Subword Tokenization** | Splitting text into units smaller than words but larger than characters based on learned merge rules | Balances vocabulary size against sequence length; handles rare and OOV words gracefully |
| **Byte-Level BPE** | BPE that operates on raw bytes (256 base tokens) instead of characters | Eliminates UNK tokens entirely; any text is representable; introduced in GPT-2 |
| **Vocabulary** | The fixed set of all tokens a model recognizes, each assigned a unique integer ID | Determines what text the model can represent and influences compression efficiency |
| **Vocabulary Size** | The total number of distinct tokens in a tokenizer (e.g., 32K, 128K, 200K) | Larger vocabularies compress text more efficiently but require larger embedding tables |
| **Merge Rules** | The ordered list of pair-merging operations learned during BPE or WordPiece training | Applied at inference time to reproduce the exact same segmentation as during training |
| **Subword Regularization** | Randomly sampling alternative valid segmentations of words during training | Improves model robustness to different ways the same word might be tokenized |
| **OOV (Out-of-Vocabulary)** | A word or character the tokenizer has never seen and cannot represent with a known token | Handled by splitting into smaller subwords or bytes; byte-level BPE eliminates OOV entirely |
| **UNK Token** | A special token used as a catch-all fallback for characters or words not in the vocabulary | Common in character and word tokenizers; rare or absent in modern byte-level tokenizers |
| **BOS Token** | Beginning-of-Sequence special token that signals the start of a text | Tells the model where a new input begins; required by most modern chat models |
| **EOS Token** | End-of-Sequence special token that signals the model to stop generating | Primary stopping condition during autoregressive generation |
| **PAD Token** | Padding token used to fill sequences to equal length within a batch | Required for efficient batched computation on GPUs where all sequences must be the same length |
| **SEP Token** | Separator token used to divide distinct segments within a single input | Structural token in BERT-style models to separate question from context, for example |
| **Chat Template** | A model-specific formatting pattern that wraps system prompts, user messages, and assistant turns with special tokens | Without correct formatting, models produce degraded outputs; templates vary across model families |
| **ChatML** | OpenAI's conversation formatting standard using `<|im_start|>` and `<|im_end|>` markers | Widely adopted chat template format; used by many open-source models |
| **Context Window** | The maximum number of tokens a model can process at once (input + output combined) | Hard limit on how much text a model can "see"; exceeding it truncates input |
| **Token Count** | The number of tokens in a given piece of text after tokenization | Determines API cost and whether the text fits within the model's context window |
| **tiktoken** | OpenAI's open-source tokenizer library for GPT model families | Used for accurate token counting when building cost estimates and context management |
| **Compression Ratio** | The ratio of bytes (or words) to tokens for a given text and tokenizer | Higher compression means fewer tokens per word; better for cost and fitting more text in context |
| **Multilingual Tokenization** | Tokenizing text in languages other than the training-majority language | Non-English text often uses 2–6× more tokens than equivalent English, increasing API cost |
| **Image Tokenization** | Converting image patches into a sequence of visual tokens via a vision encoder | Enables multimodal models to process images and text in a unified token sequence |
| **SigLIP** | A vision encoder model architecture used to convert image patches into visual token embeddings | Common component in native multimodal models for producing high-quality visual tokens |
| **EnCodec** | A neural audio codec that compresses audio into discrete token sequences | Used for audio tokenization in multimodal models; enables text-like processing of sound |
| **Token Boundary** | The exact character position where one token ends and the next begins | Splitting text mid-token corrupts decoding; chunking must respect token boundaries |
| **Token Budget** | A planned allocation of context-window tokens across system prompt, history, retrieved context, and output | Prevents context overflow and helps predict cost before making an API call |
| **Chunking** | Splitting long documents into smaller segments before embedding or sending to a model | Required when documents exceed model context limits; overlap preserves cross-chunk context |
| **Context Caching** | Pre-computing and storing KV tensors for a fixed long prefix across multiple requests | Reduces TTFT and cost by 50–90% for repeated system prompts or documents |
| **Perplexity** | A measure of how well a language model predicts a test corpus; lower is better | Standard metric for comparing tokenizer and model quality on held-out text |
| **mT5 / XLM-R** | Multilingual transformer models trained on 100+ languages simultaneously | Reference models for evaluating multilingual tokenization quality and coverage |

*Previous: [LLM Internals](01-llm-internals.md) | Next: [Attention Mechanisms](03-attention-mechanisms.md)*
