# PO Automation — GPT Extraction + Fine-Tuned BERT Classifier

> **My project.** Cut seller friction by **40%** by automating purchase-order (PO) attachment to sales quotes: a **fine-tuned BERT classifier (Azure ML)** identifies PO emails, then **Azure OpenAI GPT-3.5 with chain-of-thought prompting** extracts structured PO data from PDFs at **95% F1**.

## Table of Contents

- [The Narrative](#the-narrative)
- [What I Built — Two-Stage Architecture](#what-i-built--two-stage-architecture)
- [Deep Dive 1 — Why Two Stages (BERT Gate → GPT Extract)](#deep-dive-1--why-two-stages-bert-gate--gpt-extract)
- [Deep Dive 2 — Structured Extraction from PDFs](#deep-dive-2--structured-extraction-from-pdfs)
- [Deep Dive 3 — Chain-of-Thought Prompting](#deep-dive-3--chain-of-thought-prompting)
- [Deep Dive 4 — Measuring 95% F1](#deep-dive-4--measuring-95-f1)
- [Deep Dive 5 — Fine-Tuning BERT in Azure ML](#deep-dive-5--fine-tuning-bert-in-azure-ml)
- [Interview Q&A](#interview-qa)
- [Honest Caveats](#honest-caveats)
- [References](#references)

---

## The Narrative

**Situation.** Sellers had to manually find the right purchase-order document and attach it to the matching sales quote — slow, error-prone busywork repeated across thousands of emails, most of which weren't even POs.

**Task.** Automate the pipeline end-to-end: figure out *which* incoming emails are POs, pull the *structured data* out of the attached PDFs, and attach them to the right quote — reliably enough that sellers trust it.

**Action.** I built a **two-stage system**. A cheap, high-volume **fine-tuned BERT classifier** (trained/deployed in Azure ML) gates the firehose of email down to actual PO emails. Only those go to the expensive stage: **Azure OpenAI GPT-3.5 with chain-of-thought prompting** extracts the PO fields (number, line items, quantities, prices, dates) from the PDF into structured JSON, validated before attachment.

**Result.** **95% F1** on field extraction and a **40% reduction in seller friction** — the manual find-and-attach step largely disappeared.

---

## What I Built — Two-Stage Architecture

```
 incoming emails (high volume, mostly NOT POs)
        │
        ▼
 ┌────────────────────────┐   cheap, fast, one forward pass
 │  STAGE 1: BERT gate     │   "is this a PO email?"  (binary classifier,
 │  (fine-tuned, Azure ML) │    fine-tuned + deployed as an endpoint)
 └───────────┬────────────┘
             │ only PO emails pass  (expensive stage runs on the small subset)
             ▼
 ┌────────────────────────────────────────────────┐
 │  STAGE 2: GPT-3.5 extraction                     │
 │  chain-of-thought prompt over the PDF text →     │
 │  structured JSON: {po_number, line_items[],      │
 │  quantities, prices, dates}                      │
 │  + schema validation / retry                     │
 └───────────┬────────────────────────────────────┘
             ▼
    attach PO to the matching sales quote
```

---

## Deep Dive 1 — Why Two Stages (BERT Gate → GPT Extract)

This is the most important design decision and the first thing an interviewer will probe.

- **The classifier is a cheap filter over *all* email; the LLM is expensive and only runs on the PO subset.** Fine-tuned BERT-family encoders run **~1–2 orders of magnitude lower cost/latency** and far higher throughput than LLM prompting (a label in **one forward pass** vs token-by-token generation). Running GPT on every email would be wasteful and slow.
- **Separation of concerns:** routing (high-volume, stable, binary) vs extraction (complex, schema-driven, reasoning-heavy). Each stage uses the right tool.
- **Why fine-tune BERT rather than use GPT for classification too:** with labeled PO/non-PO data available, **fine-tuned encoders match or beat zero-shot LLMs on accuracy** *and* are far cheaper/faster at scale. GPT's open-ended reasoning is wasted on a binary routing decision.

Sources: [Cost-aware model selection (arXiv 2602.06370)](https://arxiv.org/html/2602.06370) · [Fine-tuned BERT vs zero-shot LLMs (arXiv 2406.08660)](https://arxiv.org/pdf/2406.08660)

---

## Deep Dive 2 — Structured Extraction from PDFs

**The task:** given a PO PDF, return fields (PO number, line items, quantities, prices, dates) as valid JSON. The model is prompted with the target schema and returns a structured object.

**The hard parts (know these):**
- **Digital vs scanned PDFs.** A digital PDF has extractable embedded text; a **scanned/image PDF needs OCR first** — the LLM can't "see" pixels, and naive extraction loses layout context.
- **Tables are the crux.** Line-item tables have merged cells, multi-line rows, and column structure that simple text parsers mangle (reading row-by-row or column-by-column incorrectly).
- **Robust pipeline:** OCR/layout extraction (e.g., **Azure AI Document Intelligence** Layout model, which returns text, key-value pairs, and table row/column structure) → feed that *clean, structured* text to the LLM so it reasons over layout, not raw pixels.

**Guarding against wrong/hallucinated extractions:**
1. **Schema constraint** — a JSON schema (or function-calling) forces valid structure. *(Note: strict Structured Outputs that guarantee schema adherence require GPT-4o+; on GPT-3.5 I used JSON-mode / function-calling + validation, not the strict feature — see Caveats.)*
2. **Validation + retry** — validate the JSON (e.g., Pydantic) and retry on failure.
3. **Grounding & cross-checks** — extract only from the source text; cross-check numeric fields (line-item sums vs total).
4. **Human-in-the-loop** — route low-confidence extractions for review.

Sources: [Azure AI Document Intelligence — Read/Layout OCR](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/prebuilt/read?view=doc-intel-4.0.0) · [Azure OpenAI — Structured Outputs](https://learn.microsoft.com/en-us/azure/foundry/openai/how-to/structured-outputs)

---

## Deep Dive 3 — Chain-of-Thought Prompting

**Chain-of-thought (CoT)** prompting elicits a series of **intermediate reasoning steps** before the final answer (Wei et al., 2022 — note Jason *Wei* is first author, Denny *Zhou* last; it's often mis-cited as "Zhou et al.").

**Why it helps extraction:** for a complex multi-line PO, forcing the model to first *reason about structure* — "locate the line-items table, then parse each row, then read the totals" — before emitting JSON reduces field-mapping errors versus a single-shot dump. This is the same "reason-before-answer" mechanism that gave CoT its benchmark gains on math/reasoning tasks.

**Trade-offs (be ready):**
- **Cost/latency** — CoT typically increases token use **2–4×** and adds latency (reasoning tokens generated before the answer). *This is another reason the BERT gate matters — we only pay CoT cost on the PO subset.*
- **Needs capable models** — CoT gains are strongest on larger models; small models can produce "coherent but wrong" reasoning.
- **Faithfulness** — the stated reasoning doesn't always reflect the model's true internal process.
- **Keeping JSON clean** — let the model reason in a scratch field, then emit the final structured object (or reason then strip), so CoT prose doesn't pollute the JSON.

Sources: [Chain-of-Thought Prompting (arXiv 2201.11903)](https://arxiv.org/abs/2201.11903) · [CoT trade-offs — Comet](https://www.comet.com/site/blog/chain-of-thought-prompting/)

---

## Deep Dive 4 — Measuring 95% F1

**Why F1, not accuracy:** extraction is sparse/imbalanced — most possible fields are absent on any given document, so "accuracy" is inflated by trivially-correct negatives. **F1** focuses on the fields actually present and balances the two real failure modes: **recall** (not missing fields) and **precision** (not inventing fields).

- **Precision** = TP / (TP + FP) — of the values we extracted, how many were correct.
- **Recall** = TP / (TP + FN) — of the true values, how many we extracted.
- **F1** = harmonic mean = **2·P·R / (P + R)**.

**How it's computed per field:** for each field on each document, a correct value = **TP**, a wrong/spurious value = **FP**, a missed field = **FN**. Two things I'd specify when asked:
- **Match criterion** — exact match for IDs (PO number), **normalized** match for dates/currency/whitespace.
- **Micro vs macro** — **micro-F1** pools TP/FP/FN across all fields (dominated by common fields); **macro-F1** averages per-field F1 (rare fields count equally). Best practice: report both plus a per-field breakdown.

Sources: [F1 for extraction — Galileo](https://galileo.ai/blog/f1-score-ai-evaluation-precision-recall) · [Micro vs macro F1 — PMC](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC8936911/)

---

## Deep Dive 5 — Fine-Tuning BERT in Azure ML

**BERT recap:** Bidirectional Encoder Representations from Transformers — an **encoder-only** Transformer that conditions on left+right context. A special **`[CLS]` token**'s final hidden state is the aggregate sequence representation fed to a classification head. Fine-tuning adds one output layer and updates all parameters on the labeled task.

**For the PO-email classifier:** binary head (PO vs not) on the `[CLS]` embedding, fine-tuned on labeled emails. Considerations: **small learning rate** (~2e-5–5e-5, Adam), **class imbalance** (PO emails are the minority → class weighting/resampling; evaluate with F1/PR, not accuracy), **few epochs** (2–4) + dropout/early stopping (BERT overfits fast).

**Azure ML workflow:** provision a **GPU compute cluster** → run the fine-tune as an Azure ML **job/experiment** (metrics tracked per run) → **register** the model to the workspace → **deploy** to a **managed online endpoint** for real-time inference → send new emails to the endpoint to classify.

Sources: [BERT (arXiv 1810.04805)](https://arxiv.org/abs/1810.04805) · [Azure ML — foundation models & fine-tuning](https://learn.microsoft.com/en-us/azure/machine-learning/how-to-use-foundation-models?view=azureml-api-2) · [Deep learning with BERT on Azure ML — Microsoft Community Hub](https://techcommunity.microsoft.com/t5/ai-customer-engineering-team/deep-learning-with-bert-on-azure-ml-for-text-classification/ba-p/1149262)

---

## Interview Q&A

**Q: Why a two-stage design instead of one model doing everything?**
BERT is a cheap, fast filter over *all* email; GPT extraction is expensive and only runs on the small PO subset. Encoders are ~1–2 orders of magnitude cheaper/faster and much higher throughput than LLM prompting, so gating with BERT is the cost-correct architecture. It also separates routing (high-volume, binary) from extraction (complex, schema-driven).

**Q: Why fine-tune BERT instead of using GPT for classification too?**
At high volume, fine-tuned BERT matches or beats zero-shot LLMs on accuracy *and* runs far cheaper/faster (single forward pass vs token-by-token). With labeled PO/non-PO data available, fine-tuning is the better trade-off; GPT's open-ended reasoning is wasted on a binary decision.

**Q: How did you measure 95% F1?**
Per-field TP/FP/FN against a gold-labeled test set; F1 = harmonic mean of precision and recall. I'd specify exact-match for IDs and normalized match for dates/currency, and whether 95% is micro (pooled) or macro (per-field average). F1 over accuracy because extraction is sparse — accuracy is inflated by absent fields.

**Q: How did chain-of-thought help?**
Forcing step-by-step reasoning (locate table → parse rows → extract fields → assemble JSON) before output improves accuracy on complex, multi-line POs. Trade-off is 2–4× more tokens and latency, which we accepted only on the PO subset thanks to the BERT gate.

**Q: How do you handle hallucinated or wrong extractions?**
Schema constraint (JSON schema / function-calling) so structure is valid; validation + retry (Pydantic); grounding — extract only from source text and cross-check numeric fields (line-item sums vs total); and human-in-the-loop review for low-confidence cases.

**Q: How did you handle scanned PDFs?**
The LLM can't read pixels, so scanned PDFs go through OCR/layout first (e.g., Azure AI Document Intelligence Layout, which returns table row/column structure), and the clean structured text is fed to the model.

---

## Honest Caveats

- **GPT-3.5 does *not* support strict Structured Outputs** (GPT-4o+ only). On 3.5 I constrained output via JSON-mode / function-calling + a validation-and-retry loop — **don't claim strict schema guarantees on 3.5.**
- **Confirm the OCR step** actually used for scanned POs (Document Intelligence vs another engine) — a likely follow-up.
- **Specify micro vs macro F1** and the exact/fuzzy match rules behind "95% F1."
- BERT fine-tuning LR/epochs are standard ranges — verify exact values before quoting.
- The **40% friction reduction** and **95% F1** are the project's own numbers; external throughput/token figures are cited from papers — keep them attributed.

---

## References

- [BERT: Pre-training of Deep Bidirectional Transformers (arXiv 1810.04805)](https://arxiv.org/abs/1810.04805)
- [Chain-of-Thought Prompting (arXiv 2201.11903)](https://arxiv.org/abs/2201.11903)
- [Azure ML — foundation models & fine-tuning](https://learn.microsoft.com/en-us/azure/machine-learning/how-to-use-foundation-models?view=azureml-api-2) · [BERT on Azure ML — Community Hub](https://techcommunity.microsoft.com/t5/ai-customer-engineering-team/deep-learning-with-bert-on-azure-ml-for-text-classification/ba-p/1149262)
- [Azure AI Document Intelligence — Read/Layout OCR](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/prebuilt/read?view=doc-intel-4.0.0) · [Azure OpenAI — Structured Outputs](https://learn.microsoft.com/en-us/azure/foundry/openai/how-to/structured-outputs)
- [F1 for extraction — Galileo](https://galileo.ai/blog/f1-score-ai-evaluation-precision-recall) · [Micro vs macro F1 — PMC](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC8936911/)

---

*Previous: [Shipping-Time Forecasting](06-shipping-time-forecasting-catboost.md) | Next: [Seller Behavior Analytics](08-seller-behavior-analytics.md) | Up: [Guide Home](../README.md)*
