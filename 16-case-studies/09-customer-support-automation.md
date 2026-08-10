# Case Study: AI-Powered Customer Support

## The Problem

An e-commerce company handles **2 million support tickets per month**. They want an AI system that can automatically resolve 60% of tickets without human intervention, while seamlessly escalating complex issues.

**Constraints given in the interview:**
- 24/7 operation across 12 languages
- Must integrate with existing Zendesk and Salesforce
- Cannot make false promises (refunds, shipping dates)
- Human agents must be able to take over mid-conversation
- Cost target: $0.05 per resolved ticket

---

## The Interview Question

> "Design a customer support AI that handles 'Where is my order?' automatically but knows when to escalate 'I want to sue you for fraud' to a human."

---

## Solution Architecture

```mermaid
flowchart TB
    subgraph Intake["Ticket Intake"]
        TICKET[New Ticket] --> CLASSIFY[Intent Classifier<br/>GPT-4o-mini]
        CLASSIFY --> INTENT{Intent Type}
    end

    subgraph Routing["Smart Routing"]
        INTENT -->|Simple| AUTO[Auto-Resolution Path]
        INTENT -->|Complex| HYBRID[Hybrid Path]
        INTENT -->|Escalate| HUMAN[Immediate Escalation]
    end

    subgraph AutoResolve["Auto-Resolution"]
        AUTO --> TOOLS[Tool Calls<br/>Order API, FAQ DB]
        TOOLS --> DRAFT[Draft Response]
        DRAFT --> SAFETY[Safety Check]
        SAFETY -->|Pass| SEND[Send to Customer]
        SAFETY -->|Fail| HUMAN
    end

    subgraph HybridPath["Hybrid Resolution"]
        HYBRID --> AGENT_DRAFT[AI Drafts Response]
        AGENT_DRAFT --> QUEUE[Human Review Queue]
        QUEUE --> APPROVE{Approve?}
        APPROVE -->|Yes| SEND
        APPROVE -->|Edit| EDIT[Human Edits]
        EDIT --> SEND
    end
```

---

## Key Design Decisions

### 1. Three-Tier Routing (Auto / Hybrid / Escalate)

**Answer:** Not all tickets are equal. We classify into three paths:

> **Why (the rationale):** The Hybrid path (AI drafts, human approves) is the key innovation — it captures the efficiency of AI generation while keeping a human in the loop for medium-risk decisions, avoiding the binary choice between "full automation" and "full human handling" for refund requests.
> **When to use:** Three-tier routing is justified when your ticket population naturally clusters into three risk bands. If your distribution is bimodal (either clearly safe or clearly unsafe), a simpler two-tier approach avoids the complexity of calibrating the Hybrid path thresholds.
> **Nuances & gotchas:** The Hybrid path requires human agents to review AI drafts quickly enough that the customer's response time SLO is still met. If agents take >10 minutes to review a draft, the Hybrid path's latency advantage over full escalation disappears. Staff the hybrid review queue based on hybrid ticket volume, not just escalation volume.

| Path | Criteria | Example | Human Involvement |

| Path | Criteria | Example | Human Involvement |
|------|----------|---------|-------------------|
| **Auto** | High confidence, low risk | "Where is my order?" | None |
| **Hybrid** | Medium confidence or medium risk | "I want a refund" | Reviews AI draft |
| **Escalate** | Legal, threats, VIP, low confidence | "This is fraud" | Full human handling |

### 2. Tool-Based Resolution, Not Pure Generation

**Answer:** The AI does not "know" where the order is. It calls the Order API tool.

> **Why (the rationale):** Grounding every factual claim in a tool call (Order API, FAQ DB) rather than model memory eliminates hallucinated tracking numbers, fabricated ETAs, and incorrect policy details — the most common failure modes in support AI that destroy customer trust.
> **When to use:** Tool-based resolution is mandatory whenever the answer involves time-sensitive or customer-specific data (order status, account balance, subscription tier). For general policy questions answered from a static knowledge base, RAG without live tool calls is sufficient.
> **Nuances & gotchas:** Tool calls add latency (each API round-trip is 50-200ms). For tickets requiring 3+ tool calls (look up order → check warehouse → check carrier), the cumulative latency can push total response time above the 5-second target. Cache tool responses within a session and batch tool calls in parallel where dependencies allow.

The LLM orchestrates tools but never fabricates data. This is critical for accuracy:

```python
@tool
def get_order_status(order_id: str) -> dict:
    """Retrieve real-time order status from OMS."""
    order = oms_client.get_order(order_id)
    return {
        "status": order.status,
        "shipped_date": order.shipped_at,
        "estimated_delivery": order.eta,
        "tracking_url": order.tracking_url
    }
```

The LLM orchestrates tools but never fabricates data.

### 3. Why Safety Check Before Send?

**Answer:** Even auto-resolved tickets go through a safety filter:

> **Why (the rationale):** The safety check is the last gate between AI output and the customer — it catches the specific failure modes that create legal liability (unauthorized promises) or brand damage (tone-deaf responses) that are impossible to prevent purely through prompt engineering.
> **When to use:** Safety checks are non-negotiable before sending any AI-generated text to an external customer. For internal-facing AI tools where mistakes are correctable, a lighter-weight review (sentiment only, no promise detection) is proportionate.
> **Nuances & gotchas:** Promise detection based on phrases like "I guarantee" or "We will pay" misses indirect commitments ("Our standard policy is to refund within 3 days" — which a customer may interpret as a promise). Include a policy-statement classifier that flags assertions about what the company will do, not just explicit guarantees.

1. **Promise Detection**: Flags statements like "I guarantee" or "We will pay"

1. **Promise Detection**: Flags statements like "I guarantee" or "We will pay"
2. **Sentiment Mismatch**: Catches if AI sounds happy when customer is angry
3. **PII Leak**: Ensures no internal notes or other customer data appear
4. **Competitor Mention**: Flags if AI recommends a competitor

---

## The Escalation Intelligence

The hardest part is knowing **when** to escalate. We use a confidence score with multiple signals:

```mermaid
flowchart LR
    subgraph Signals["Confidence Signals"]
        S1[Intent Confidence<br/>0.92] --> COMBINE
        S2[Sentiment Score<br/>Negative] --> COMBINE
        S3[Customer Tier<br/>VIP] --> COMBINE
        S4[Topic Risk<br/>Legal = High] --> COMBINE
    end

    COMBINE[Weighted Aggregation] --> SCORE{Final Score}
    SCORE -->|> 0.85| AUTO[Auto-Resolve]
    SCORE -->|0.5 - 0.85| HYBRID[Human Review]
    SCORE -->|< 0.5| ESCALATE[Immediate Escalate]
```

**Key insight:** A VIP customer asking a simple question still goes to Hybrid path because the cost of a mistake is higher.

---

## Multilingual Support

12 languages without 12 separate models:

```mermaid
flowchart LR
    INPUT[Customer Message<br/>Spanish] --> DETECT[Language Detection]
    DETECT --> TRANSLATE_IN[Translate to English]
    TRANSLATE_IN --> PROCESS[Process in English<br/>Tools + LLM]
    PROCESS --> TRANSLATE_OUT[Translate to Spanish]
    TRANSLATE_OUT --> RESPONSE[Response in Spanish]
```

**Why not native multilingual models?**

Cost. GPT-4o handles all 12 languages well. Using specialized models per language would require 12 deployments. Translation adds latency but keeps infrastructure simple.

> **Why (the rationale):** Translate-to-English → process → translate-back allows all prompting, tool-call schemas, and safety rules to be maintained in a single language, avoiding the combinatorial burden of 12 language-specific system prompts and 12 separate evaluation datasets.
> **When to use:** The translate-in/process/translate-out pattern works well for factual support queries where meaning survives translation. For highly idiomatic or culturally nuanced queries (local regulations, culture-specific complaints), native multilingual processing in the target language produces better results.
> **Nuances & gotchas:** Double translation accumulates errors — a nuanced Spanish complaint translated to English may lose the emotional register, leading the generator to reply in a tone that is inappropriately formal or casual when translated back. Monitor CSAT scores by language to detect translation-induced quality degradation for specific languages.

---

## Human Takeover (Mid-Conversation)

> **Why (the rationale):** Generating an LLM summary of the conversation for the human agent instead of passing the raw transcript reduces the time the human spends reading before acting — in a high-volume support environment, a 30-second context transfer vs a 3-minute transcript read has a significant impact on queue throughput.
> **When to use:** LLM-generated handoff summaries are most valuable when conversations are long (5+ turns) or when the agent attempted multiple failed solutions that the human needs to know about. For simple one-turn escalations, the raw message is sufficient.
> **Nuances & gotchas:** The handoff summary is generated by the same model that may have given an incorrect answer — if the model misunderstood the customer's issue, its summary will also misrepresent it. Include the full transcript as a collapsible fallback in the handoff UI so human agents can verify the summary against the raw exchange.

When a human takes over, they need full context:

```python
def handoff_to_human(conversation_id: str, agent_id: str):
    conversation = get_conversation(conversation_id)
    
    # Generate summary for human agent
    summary = llm.generate(f"""
    Summarize this conversation for a human agent:
    - Customer issue
    - What AI already tried
    - Why escalation happened
    
    Conversation:
    {conversation.messages}
    """)
    
    # Create handoff package
    return {
        "summary": summary,
        "customer_sentiment": conversation.sentiment,
        "attempted_solutions": conversation.tool_calls,
        "full_transcript": conversation.messages,
        "customer_tier": conversation.customer.tier
    }
```

---

## Cost Analysis

| Component | Cost per Ticket |
|-----------|-----------------|
| Intent classification (GPT-4o-mini) | $0.002 |
| Tool calls (Order API, FAQ search) | $0.001 |
| Response generation (GPT-4o-mini) | $0.008 |
| Safety check | $0.003 |
| Translation (if needed, 30% of tickets) | $0.004 |
| **Average total** | **$0.018** |

At a 60% auto-resolution rate: **$0.03 per resolved ticket** (well under $0.05 target)

---

## Interview Follow-Up Questions

**Q: What if the AI keeps apologizing but never actually helps?**

A: We track "resolution effectiveness" not just "response sent." If a customer replies again within 24 hours on the same issue, that ticket is marked as "unresolved" and the AI pattern is flagged for review. We also run weekly analysis: "What phrases correlate with customer follow-ups?"

**Q: How do you handle a customer who insists on talking to a human?**

A: Explicit escalation phrases ("talk to a human", "speak to manager") trigger immediate handoff regardless of confidence score. We never argue with escalation requests.

**Q: What about customers who try to jailbreak the support AI?**

A: Input sanitization plus strict tool-only responses. The AI cannot be prompted to reveal system prompts because it does not generate free-form answers: it calls tools and summarizes their outputs. The system prompt is also extremely narrow: "You help with order issues for [Company]. You cannot discuss other topics."

---

## Key Takeaways for Interviews

1. **Tiered routing balances automation with risk**: not every ticket should be auto-resolved
2. **Tool-based grounding prevents hallucination**: the AI retrieves facts, it does not generate them
3. **Confidence is multi-dimensional**: intent clarity + sentiment + customer tier + topic risk
4. **Human handoff needs context**: summarize, do not just dump the transcript

---

---

## Glossary

| Term | Simple explanation | Purpose |
|---|---|---|
| **Intent Classifier** | A model that reads an incoming ticket and labels what the customer wants (track order, request refund, report fraud, etc.) | The entry point for all routing decisions; determines which automated path—if any—is safe to take |
| **Three-Tier Routing** | Splitting tickets into Auto (AI handles fully), Hybrid (AI drafts, human approves), and Escalate (full human handling) | Matches the automation level to the risk level of each ticket rather than treating all tickets the same |
| **Auto-Resolution Path** | The pipeline branch where the AI calls tools, drafts a response, passes a safety check, and sends it without human review | Handles simple high-confidence queries at near-zero marginal cost |
| **Hybrid Path** | The pipeline branch where the AI drafts a response but a human agent must review and optionally edit before it is sent | Balances automation speed with human oversight for medium-risk tickets like refund requests |
| **Tool Call** | An LLM action that invokes an external function or API (e.g., Order API, FAQ search) to retrieve real data | Prevents hallucination by ensuring the AI retrieves facts from systems of record rather than generating them |
| **OMS (Order Management System)** | The backend system that tracks real-time order status, shipping dates, and tracking links | The authoritative data source for "Where is my order?" queries; the AI calls it rather than guessing |
| **Safety Check** | An automated filter that scans the AI's draft response for promises, PII leaks, sentiment mismatches, and competitor mentions | The last gate before a response reaches the customer; catches outputs that could create legal or reputational risk |
| **Promise Detection** | A safety-check component that flags language like "I guarantee" or "We will refund" that make binding commitments | Prevents the AI from making unauthorized financial or service promises that the company cannot fulfill |
| **Sentiment Mismatch** | A safety flag triggered when the AI's tone is positive but the customer's message is angry or distressed | Catches tone-deaf responses that would worsen customer satisfaction even if factually correct |
| **PII Leak** | Accidentally including another customer's data or internal notes in an outbound message | A privacy violation that also erodes customer trust; checked for in every outbound response |
| **Confidence Score** | A weighted aggregate of intent certainty, customer sentiment, customer tier, and topic risk | The single number that determines which routing path a ticket takes |
| **Customer Tier** | A classification of a customer's account value or contract level (e.g., VIP, standard) | High-tier customers are routed to Hybrid or Escalate even for simple questions because errors cost more |
| **Topic Risk** | A severity label for the category of the customer's request (legal, fraud, standard inquiry) | Legal and fraud topics trigger immediate escalation regardless of confidence score |
| **Escalation** | Handing a conversation to a human agent to resolve | The safety valve that handles everything the AI should not attempt alone—threats, legal claims, VIP issues |
| **Handoff Package** | The structured summary (issue, prior attempts, sentiment, full transcript) generated when passing to a human | Lets the human agent understand the situation instantly without re-reading every message |
| **Multilingual Support** | The ability to receive messages in one language, process them in another, and reply in the original language | Enables 12-language support without deploying 12 separate specialized models |
| **Language Detection** | Automatically identifying which language a message is written in | Routes the message into the translate-in → process-in-English → translate-out pipeline |
| **Input Sanitization** | Removing or neutralizing malicious instructions embedded in customer messages | Prevents prompt injection attacks where customers try to override the AI's persona or access restricted functions |
| **Jailbreak** | An attempt by a user to bypass the AI's safety instructions through clever prompt phrasing | Mitigated here by restricting the AI to tool outputs only, so it cannot be prompted into free-form harmful responses |
| **Resolution Effectiveness** | A metric that marks a ticket as unresolved if the customer replies again within 24 hours on the same issue | Catches cases where the AI sent a response that technically "answered" but did not actually help the customer |
| **Zendesk / Salesforce** | CRM and ticketing platforms that manage customer communication and case records | The existing systems the AI must integrate with via APIs rather than replacing |
| **GPT-4o-mini** | OpenAI's fast, low-cost model used for intent classification, response generation, and safety checks | Keeps the blended cost per ticket at $0.018, well within the $0.05 target |

*Related chapters: [Human-in-the-Loop Patterns](../07-agentic-systems/08-human-in-the-loop-patterns.md), [Guardrails Implementation](../13-reliability-and-safety/01-guardrails.md)*
