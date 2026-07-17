# Last 24 Hours — Final Review

The single page to read the night before your **AWS Certified AI Practitioner (AIF-C01)** and/or **Machine Learning Engineer – Associate (MLA-C01)** exam. Skim it, sleep, show up sharp.

---

## Exam logistics reminders

| | AIF-C01 (AI Practitioner) | MLA-C01 (ML Engineer – Associate) |
|---|---|---|
| **Passing score** | 700 / 1000 (scaled) | 720 / 1000 (scaled) |
| **Total questions** | 65 | 65 |
| **Scored questions** | 50 (15 unscored/pilot) | 50 (15 unscored/pilot) |
| **Time** | 90 min | 130 min |
| **Scoring** | Compensatory — pass the **overall** exam, not each domain | Compensatory — same |
| **Guessing** | No penalty — **answer every question** | No penalty — **answer every question** |

- **Scaled scoring:** 100–1000 range; equates difficulty across exam forms, so your raw % ≠ your reported score.
- **You can't see which 15 are unscored** — treat every question as if it counts.
- **Question types:** multiple choice (1 correct), multiple response (2+ correct — the question tells you how many), plus newer formats on these exams: **ordering**, **matching**, and **case study** (one scenario, several independently-scored questions).

---

## Exam-day tactics

- **Flag-and-move.** If a question stalls you >90 sec, mark it and come back. Don't bleed time.
- **Read the qualifier — it decides the answer.** Underline the constraint: **MOST cost-effective / LEAST operational overhead / FASTEST / most SECURE / real-time vs batch**. Two "correct" answers usually differ only on this.
- **"Least operational overhead" → managed / serverless.** Prefer Bedrock, SageMaker JumpStart, serverless inference, Firehose, Fargate, managed services over self-managed EC2/clusters.
- **Eliminate out-of-scope services first.** Cross off answers naming a service that doesn't do the job (e.g., Textract for object detection, Comprehend for images). Usually clears 2 of 4.
- **Watch absolute words** ("always," "never," "only") — often wrong on associate-level questions.
- **Never leave a blank.** No guessing penalty. Do a final sweep and make sure all 65 have an answer.

---

## The 40 highest-yield facts

1. **Real-time endpoint** = always-on, ms latency, max **6 MB** payload, 60-sec timeout.
2. **Serverless inference** = scales to zero, intermittent traffic, ~**4 MB** payload, cold starts; you pick memory (up to 6 GB).
3. **Asynchronous inference** = queued requests, large payloads (**up to 1 GB**), long processing (up to 1 hr), scales to zero — use for big payloads / long jobs needing an endpoint.
4. **Batch Transform** = no persistent endpoint, reads from S3, **no strict payload limit**, cheapest for large offline volumes.
5. **SageMaker Clarify** = **bias detection + explainability** (feature attribution, SHAP), pre-training and post-training bias, feature-attribution drift.
6. **SageMaker Model Monitor** = **data quality + model quality drift** in production (accuracy/RMSE against a baseline).
7. **RAG** = inject external/current knowledge at inference, no retraining — cheap, fast, cites sources, reduces hallucination.
8. **Fine-tuning** = change model weights for domain tone/format/skill — expensive, needs labeled data, slower to update.
9. Cost tradeoff: **prompt engineering < RAG < fine-tuning < pre-training** (from cheapest/fastest to most expensive).
10. **Rekognition** = images/video (objects, faces, moderation, celebrities). **Textract** = extract text/forms/tables from documents (OCR+).
11. **Comprehend** = NLP on text (sentiment, entities, PII, key phrases, language). **Transcribe** = speech→text. **Polly** = text→speech. **Translate** = language translation.
12. **CloudTrail** = **who did what** (API call audit log). **Config** = **resource configuration state / compliance** over time.
13. **CloudWatch** = metrics, logs, alarms (monitoring/observability) — not an audit trail.
14. **Bedrock Guardrails** = content filters, denied topics, **PII redaction**, word filters, and **contextual grounding** checks to block hallucinations.
15. **Temperature** ↑ = more random/creative output; ↓ (near 0) = more deterministic/focused. Top-p/top-k also control randomness.
16. **Precision** = trust the positives (minimize false positives → spam, fraud alerts). **Recall** = catch them all (minimize false negatives → disease, security).
17. **F1 score** = harmonic mean of precision & recall — use when classes are imbalanced.
18. **AUC-ROC** = ranking/threshold-independent classifier quality.
19. **Spot** = up to 90% off, interruptible → fault-tolerant **training**, not inference. **Reserved/Savings Plans** = commit 1–3 yr for steady-state. **On-Demand** = spiky/unpredictable.
20. **Savings Plans** = flexible commitment ($/hr) across instance families/services; **Reserved Instances** = specific instance-type commitment.
21. **Kinesis Data Streams** = real-time, **custom consumers**, retention/replay, you manage shards. **Firehose** = fully managed **delivery** to S3/Redshift/OpenSearch, near-real-time, no consumer code.
22. **Parquet** = columnar, compressed → cheaper Athena/analytics scans; **JSON/CSV** = row-based. Convert to Parquet to cut query cost.
23. **SageMaker Feature Store** = central, reusable, consistent features for training **and** inference (avoids train/serve skew).
24. **SageMaker Data Wrangler** = visual data prep/feature engineering; **Processing Jobs** = scaled preprocessing.
25. **Ground Truth** = data labeling (human + auto-labeling). **Ground Truth Plus** = fully managed labeling.
26. **Bedrock** = serverless access to foundation models (Anthropic, Meta, Amazon Titan/Nova, etc.) — no infra. **SageMaker JumpStart** = pre-built models/solutions you deploy.
27. **Bedrock Knowledge Bases** = managed RAG; **Bedrock Agents** = multi-step tasks calling APIs/tools.
28. **Overfitting** = great on train, poor on new data → add data, regularization, dropout, early stopping, simpler model.
29. **Underfitting** = poor on both → more complex model, more features, train longer.
30. **Supervised** = labeled data (classification/regression). **Unsupervised** = no labels (clustering, anomaly). **Reinforcement** = reward-driven.
31. **Inferentia** = cost-efficient **inference** chips; **Trainium** = cost-efficient **training** chips.
32. **Macie** = discover/protect **PII in S3**. **Guardrails/Comprehend PII** = redact PII in text pipelines.
33. **SageMaker Pipelines** = ML workflow orchestration/CI-CD; **Model Registry** = version & approve models for deployment.
34. **Blue/green & canary deployment** on endpoints = safe rollout with automatic rollback on CloudWatch alarms.
35. **Multi-model endpoint (MME)** = many models behind one endpoint (cost-saving for many low-traffic models).
36. **Hyperparameter tuning (AMT)** = automatic search (Bayesian/random/grid) to optimize a chosen objective metric.
37. **Embeddings** = numeric vectors capturing meaning; power semantic search, RAG retrieval, and clustering.
38. **Vector stores** for RAG: OpenSearch, Aurora/pgvector, Kendra (managed intelligent search) — pick managed for least overhead.
39. **Responsible AI dimensions:** fairness, explainability, robustness, privacy/security, governance, transparency, veracity — Bedrock supports via **AI Service Cards** and Guardrails.
40. **Confusion matrix** basics: TP/TN correct, FP = false alarm, FN = missed case — precision = TP/(TP+FP), recall = TP/(TP+FN).

---

## Must-know "if you see X → pick Y" reflexes

| If the question says / needs… | Pick this |
|---|---|
| Real-time, low-latency single prediction | SageMaker **real-time endpoint** |
| Huge offline dataset, no endpoint needed | **Batch Transform** |
| Large payload (>4 MB) or long processing, but async | **Asynchronous inference** |
| Spiky/intermittent traffic, scale to zero | **Serverless inference** |
| Detect **bias** or explain predictions | SageMaker **Clarify** |
| Detect **data/model drift** in production | **Model Monitor** |
| Extract text/tables/forms from a PDF/scan | **Textract** |
| Detect objects/faces/moderation in images/video | **Rekognition** |
| Sentiment / entities / PII in text | **Comprehend** |
| Speech to text | **Transcribe** |
| Text to lifelike speech | **Polly** |
| Add current/company knowledge without retraining | **RAG** (Bedrock Knowledge Bases) |
| Change model tone/format/domain skill | **Fine-tuning** |
| Serverless access to foundation models | **Bedrock** |
| Block harmful output / redact PII from GenAI | **Bedrock Guardrails** |
| Multi-step task that calls tools/APIs | **Bedrock Agents** |
| "Who made this API call?" (audit) | **CloudTrail** |
| "Is this resource compliant / what changed?" | **AWS Config** |
| Metrics, logs, alarms | **CloudWatch** |
| Real-time stream with custom consumers/replay | **Kinesis Data Streams** |
| Just deliver streaming data to S3/Redshift, managed | **Kinesis Data Firehose** |
| Cheaper analytics queries on big data | Convert to **Parquet** (columnar) |
| Fault-tolerant training, cut cost up to 90% | **Spot Instances** |
| Steady 24/7 workload, lowest committed cost | **Savings Plans / Reserved** |
| Cheapest inference hardware | **Inferentia** |
| Cheapest training hardware | **Trainium** |
| Reusable, consistent features train+serve | **Feature Store** |
| Discover/protect PII in S3 | **Macie** |
| Data labeling at scale | **Ground Truth** |
| Managed intelligent/semantic search | **Kendra** |
| Orchestrate an ML CI/CD workflow | **SageMaker Pipelines** |
| Version & approve models before deploy | **Model Registry** |
| Minimize **false negatives** (catch all) | Optimize **recall** |
| Minimize **false positives** (avoid false alarms) | Optimize **precision** |
| More deterministic LLM output | **Lower temperature** |

---

## GenAI concept one-liners

- **Tokens** — chunks of text (≈¾ word) the model reads/generates; you pay per token and context windows are measured in tokens.
- **Embeddings** — vectors that capture meaning; enable semantic search, RAG retrieval, and clustering.
- **Temperature** — randomness dial: low = focused/deterministic, high = creative/varied.
- **RAG** — retrieve relevant external documents and feed them into the prompt so the model answers with current, grounded facts (no retraining).
- **Fine-tuning** — update model weights on your labeled data to specialize tone, format, or domain skill.
- **Hallucination** — confident but false output; mitigate with RAG, grounding, and Guardrails contextual checks.
- **Agents** — LLMs that plan multi-step tasks and call tools/APIs to act, not just chat.
- **ROUGE / BLEU** — text-generation eval metrics: ROUGE for summarization (recall of overlap), BLEU for translation (precision of overlap).
- **RLHF** — Reinforcement Learning from Human Feedback: humans rank outputs to align the model with preferences.

---

## Deep breath

- **Bring valid government-issued photo ID** (name must match your registration exactly).
- **Arrive / log in ~30 min early.** Online proctored: clear your desk, close all apps, plan a quiet room.
- **Sleep** — a rested brain beats one more hour of cramming.
- **Eat and hydrate** before you start; test center rules limit breaks.
- **You have time** — 90 or 130 minutes is plenty. Flag hard ones, keep moving, sweep at the end.
- **Compensatory scoring + no guessing penalty** = answer everything; one hard domain won't sink you.
- **Trust the qualifier, trust your prep.** You've got this. Good luck.
