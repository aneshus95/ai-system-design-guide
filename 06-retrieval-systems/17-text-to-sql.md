# Text-to-SQL (Natural Language → SQL)

When a user's question needs **computation over structured data** — "what was total revenue by region last quarter?", "how many orders shipped late in March?" — vector RAG is the wrong tool. Similarity search can *find* a table, but it can't **sum, count, filter, join, or rank**. **Text-to-SQL** solves this: an LLM translates the natural-language question into a SQL query, the database computes the *exact* answer, and you return it. This page explains the flow intuitively, the parts that are hard, and how to make it reliable in production.

> Companion to [Answering Questions *from* Tables](02-chunking-strategies.md#answering-questions-from-tables-interview-deep-dive), which explains *when* to route a query to Text-to-SQL vs vector RAG.

## Table of Contents

- [The Intuition](#the-intuition)
- [Why Not Just Use RAG?](#why-not-just-use-rag)
- [The Flow, Step by Step](#the-flow-step-by-step)
- [The Hard Parts](#the-hard-parts)
- [Guardrails (Non-Negotiable in Production)](#guardrails-non-negotiable-in-production)
- [Evaluation — Execution Accuracy & the Enterprise Cliff](#evaluation--execution-accuracy--the-enterprise-cliff)
- [Text-to-SQL vs RAG — When to Use Which](#text-to-sql-vs-rag--when-to-use-which)
- [Interview Q&A](#interview-qa)
- [Glossary](#glossary)
- [References](#references)

---

## The Intuition

Think of Text-to-SQL as a **translator sitting between a human and a database**. The human speaks English ("top 5 customers by spend this year"); the database only understands SQL. The LLM is the translator — but a translator who has never seen this particular database, so **you have to hand it a phrasebook first**: the table names, columns, and how things connect. Give it that context, and it writes the query; the database does the actual math.

```
  "Top 5 customers by spend in 2025?"
             │  (human language)
             ▼
        ┌──────────┐   given the schema (the phrasebook)
        │   LLM     │   writes ►  SELECT customer, SUM(amount) ...
        │ translator│              FROM orders WHERE year = 2025
        └────┬─────┘               GROUP BY customer ORDER BY 2 DESC LIMIT 5
             ▼  (SQL)
        ┌──────────┐
        │ Database  │   runs the query → EXACT answer (real math, not a guess)
        └────┬─────┘
             ▼
   "1. Acme $2.1M  2. Globex $1.8M ..."  (optionally reworded into a sentence)
```

The key mental shift: **the LLM doesn't compute the answer — it writes the *instructions* for the database to compute it.** That's why the number is exact, not a hallucinated estimate.

---

## Why Not Just Use RAG?

Vector RAG retrieves *text that looks similar* to the question. That fails for structured/analytical questions in two ways:

- **It can't do math.** "Average order value in Q3" isn't written in any chunk — it has to be *computed* across thousands of rows. Similarity search can't sum or average.
- **It can't be exact or complete.** RAG returns the top-K most similar rows, not *all* matching rows — so any "how many / total / per group" answer would be wrong.

SQL is built for exactly this: `WHERE` filters, `GROUP BY` aggregates, `JOIN` combines tables, `ORDER BY`/`LIMIT` rank. So for structured data, you let the database do what it's great at and use the LLM only to *write the query*.

---

## The Flow, Step by Step

A production Text-to-SQL system is not "prompt the LLM and run whatever it says." It's a small pipeline with a feedback loop:

```
 (1) QUESTION
      │
      ▼
 (2) SCHEMA LINKING ───► pick only the relevant tables/columns (+ business rules via RAG)
      │
      ▼
 (3) PROMPT ASSEMBLY ──► question + focused schema + few-shot examples + rules
      │
      ▼
 (4) LLM GENERATES SQL
      │
      ▼
 (5) VALIDATE ─────────► parse it, check it's read-only, add a LIMIT
      │          fails?
      ▼            └────────────┐
 (6) EXECUTE on the database    │
      │                         │
   error / empty / wrong shape? │
      │──────────────► (7) SELF-CORRECT: feed the DB error back to the LLM → regenerate
      ▼                         ▲
 (8) RETURN result  ────────────┘ (loop a bounded number of times)
      │  (optionally: LLM rewrites the rows into a natural-language sentence)
      ▼
   ANSWER
```

**(1) The question** — the user's natural-language ask.

**(2) Schema linking — the single most important step.** A real database has *hundreds* of tables and thousands of columns; you can't (and shouldn't) dump all of that into the prompt — it's expensive, blows the context window, and *confuses* the model with irrelevant options. So you first **select only the tables/columns relevant to this question** (by keyword match, embeddings over the schema, or a small LLM pass). *Intuition: hand the translator the two pages of the phrasebook it needs, not the whole dictionary.* This is also where you pull in **business rules via RAG** — definitions that live outside the schema ("'active customer' means an order in the last 90 days"), because the model can't infer them from column names alone.

**(3) Prompt assembly** — build the prompt: the question + the **focused schema** (table/column names, types, maybe sample rows) + a few **worked examples** (few-shot: "question → correct SQL" pairs teach the model your conventions) + any **business rules**. Good context here is 80% of the accuracy.

**(4) LLM generates the SQL** — the model writes the query. Chain-of-thought / decomposition helps on complex multi-join questions ("first find X, then join to Y").

**(5) Validate before running** — never execute blindly. Parse the SQL (is it valid?), confirm it's **read-only** (reject `DROP`/`DELETE`/`UPDATE`), and inject a **`LIMIT`** so a runaway query can't scan a billion rows. Cheap checks that prevent expensive disasters.

**(6) Execute** — run it against the database and get real rows.

**(7) Self-correct (the feedback loop that makes it reliable)** — if the database throws an error ("no such column `revenue`"), you don't give up: **feed the error message back to the LLM** and ask it to fix the query. This "execution-guided" repair loop is what lifts accuracy dramatically — the DB's error message is a precise, free hint. Cap the retries (e.g., 2–3) so it can't loop forever.

**(8) Return** — hand back the result. Often a final LLM step **rewrites the raw rows into a sentence** ("The top region was APAC with $2.1M") so the user gets language, not a table dump.

---

## The Hard Parts

- **Schema linking at scale** — with hundreds of tables, picking the right ones is the #1 accuracy lever *and* the #1 failure point. Get it wrong and every downstream step is doomed.
- **Business rules & tribal knowledge** — the hardest questions depend on logic that **isn't in the schema**: what "revenue" excludes, which status codes count as "shipped," fiscal-year boundaries. This context lives in PRDs, runbooks, and Slack — so you retrieve it with RAG and inject it.
- **Ambiguity** — "last quarter," "best customer," "region" may map to several columns or interpretations. Production systems ask a clarifying question or expose their assumption.
- **Joins & complex logic** — multi-table joins, nested subqueries, and window functions are where models slip; decomposition and few-shot examples help most here.
- **Dialect differences** — Postgres vs BigQuery vs Snowflake SQL differ (date functions, quoting); tell the model the target dialect.

---

## Guardrails (Non-Negotiable in Production)

You're letting an LLM generate code that runs against your database — treat it as untrusted:

- **Read-only by default** — connect with a role that *cannot* write/delete/alter. The database enforces safety even if the LLM is wrong or the input is malicious. This is your strongest line of defense.
- **Row/time limits** — always inject `LIMIT`, set query timeouts, cap scanned bytes — so a bad query can't take down the warehouse or run up a huge cloud bill.
- **Permission scoping** — the query must run under **the user's** permissions (row-level security / allowed tables), so users can't ask for data they aren't allowed to see. Same "don't return what the user can't access" rule as RAG.
- **Injection / prompt-injection awareness** — validate/parse generated SQL; treat retrieved business rules as untrusted (a poisoned rule could try to widen access).
- **Human approval for anything non-read** — if the system ever writes, gate it behind confirmation.

---

## Evaluation — Execution Accuracy & the Enterprise Cliff

- **Execution Accuracy (EX)** is the standard metric: **run the generated SQL *and* the gold SQL, and check the result sets match.** It rewards *semantic* correctness (two different queries that return the same rows both count) rather than exact string matching.
- **Benchmarks:** **Spider** (cross-domain, unseen schemas, multi-table) and **BIRD** (large, messy, real-world databases + business knowledge). Frontier systems hit **~85%+ EX on Spider**.
- **The enterprise cliff (be honest about this):** benchmark scores *overstate* real performance. On a messy production warehouse with hundreds of tables, cryptic column names, and business rules outside the schema, accuracy drops sharply — "the closer a benchmark gets to a real warehouse, the more the score depends on the **system around the model** (schema linking, business-rule RAG, self-correction) than the model itself." So invest in the pipeline, not just a bigger model.

---

## Text-to-SQL vs RAG — When to Use Which

| | **Text-to-SQL** | **Vector RAG** |
|---|---|---|
| Data | **Structured** (tables, databases) | **Unstructured** (docs, PDFs, wikis) |
| Question | aggregate, count, filter, rank, join | "explain / summarize / find the passage about…" |
| Answer | **exact & complete** (computed) | best-effort, top-K similar text |
| Example | "total sales by region in Q3" | "what's our refund policy?" |

**The router pattern:** a real assistant classifies each question and sends it down the right path — analytical/structured → Text-to-SQL; explanatory/unstructured → RAG. Some systems even do both and combine (SQL for the numbers, RAG for the narrative around them).

---

## Interview Q&A

**Q: Why use Text-to-SQL instead of RAG for a data question?**
Vector search retrieves similar text; it can't sum, count, or aggregate, and it returns top-K rows rather than *all* matching rows — so any analytical answer would be wrong or incomplete. SQL computes the exact answer over the full table. So I use the LLM only to *write* the query and let the database do the math.

**Q: What's the most important part of a Text-to-SQL pipeline?**
Schema linking — selecting the few relevant tables/columns before prompting. Real databases are too big to dump into the prompt (cost, context limits, and it confuses the model). Good schema linking is the biggest accuracy lever; get it wrong and everything downstream fails.

**Q: How do you make it reliable, not just a one-shot prompt?**
An **execution-guided self-correction loop**: validate the SQL (read-only, add LIMIT), run it, and if the DB errors, feed the error message back to the LLM to regenerate — bounded to a few retries. The database's error is a precise, free hint that dramatically improves accuracy.

**Q: How do you keep it safe?**
Read-only DB role (the database enforces safety regardless of what the LLM writes), injected LIMIT + timeouts + scanned-byte caps, and the query runs under the user's permissions/row-level security so it can't return unauthorized data. Never let it write without human approval.

**Q: How do you handle business logic that isn't in the schema?**
Retrieve it with RAG and inject it into the prompt — definitions like "active customer = ordered in last 90 days" live in docs/runbooks, not column names. This is often the difference between benchmark accuracy and real-world accuracy.

**Q: How do you evaluate it?**
Execution Accuracy — run the generated and gold SQL and compare result sets (semantic match, not string match). Benchmark on Spider/BIRD, but expect a drop on a real messy warehouse; that gap is closed by the surrounding system (schema linking, business-rule RAG, self-correction), not a bigger model.

---

## Glossary

| Term | Simple explanation | Purpose |
|---|---|---|
| **Text-to-SQL / NL2SQL** | Translating a natural-language question into a SQL query with an LLM | Lets non-technical users query databases in plain English |
| **Schema** | The structure of the database — tables, columns, types, relationships | The "phrasebook" the LLM needs to write correct queries |
| **Schema linking** | Selecting only the tables/columns relevant to the question before prompting | Reduces noise, cost, and context size; the #1 accuracy lever |
| **Prompt assembly** | Building the prompt from question + focused schema + examples + rules | Provides the context that determines ~80% of accuracy |
| **Few-shot examples** | Sample "question → correct SQL" pairs in the prompt | Teach the model your schema's conventions and query style |
| **Self-correction / execution-guided repair** | Feeding a DB error back to the LLM to fix the query | Turns runtime errors into precise hints; big reliability boost |
| **Execution Accuracy (EX)** | Whether the generated query returns the same rows as the gold query | Rewards semantic correctness (not exact string match) |
| **Spider** | Benchmark for cross-domain, multi-table, unseen-schema Text-to-SQL | Measures generalization to new databases |
| **BIRD** | Benchmark with large, messy, real-world DBs + business knowledge | Measures realistic, database-grounded difficulty |
| **Business rules** | Domain definitions not encoded in the schema (e.g. what "revenue" means) | Retrieved via RAG and injected; closes the real-world accuracy gap |
| **Read-only role** | A DB connection that cannot write/delete/alter | Enforces safety at the database regardless of what the LLM generates |
| **Row-level security (RLS)** | DB rules restricting which rows a user can see | Ensures the query only returns data the user is authorized for |
| **Dialect** | The specific SQL variant (Postgres, BigQuery, Snowflake…) | Must be told to the model so functions/syntax are valid |
| **Query router** | A classifier that sends a question to Text-to-SQL or RAG | Routes structured questions to SQL, unstructured to vector search |

---

## References

- [Text-to-SQL Empowered by LLMs: A Benchmark Evaluation (DAIL-SQL, arXiv 2308.15363)](https://arxiv.org/pdf/2308.15363)
- [MAC-SQL: Multi-Agent Text-to-SQL (Selector / Decomposer / Refiner)](https://arxiv.org/abs/2312.11242)
- [RSL-SQL: Robust Schema Linking (arXiv 2411.00073)](https://arxiv.org/html/2411.00073v1)
- [MAGIC: Self-Correction Guidelines for Text-to-SQL (arXiv 2406.12692)](https://arxiv.org/pdf/2406.12692)
- [How Accurate Is Text-to-SQL, Really? Spider, BIRD, and the Enterprise Cliff — Datost](https://datost.com/blog/text-to-sql-accuracy-benchmarks)

---

*Previous: [Production RAG Considerations](16-production-rag-considerations.md) | Up: [Guide Home](../README.md)*
