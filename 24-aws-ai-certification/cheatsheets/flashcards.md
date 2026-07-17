# Flashcards — Rapid Recall

Rapid-recall deck for **AWS AI Practitioner (AIF-C01)** and **AWS ML Engineer Associate (MLA-C01)**. Cover the answer, say it out loud, then click to reveal and check yourself.

> How to use: read the **Q**, answer from memory, then expand the card. Aim for reflexes on the "If you see X → pick Y" cards — those mirror how the exam frames questions.

---

## 1. Core AI / ML Vocabulary

<details><summary><b>Q: AI vs ML vs DL — how do they nest?</b></summary>

A: **AI ⊃ ML ⊃ DL.** AI = any technique that mimics human intelligence. ML = subset that learns patterns from data instead of explicit rules. DL = subset of ML using multi-layer neural networks (learns features automatically).
</details>

<details><summary><b>Q: Supervised learning?</b></summary>

A: Learns from **labeled** data (input → known output). Used for classification and regression. E.g. predict churn from historical labeled examples.
</details>

<details><summary><b>Q: Unsupervised learning?</b></summary>

A: Learns from **unlabeled** data by finding structure. Used for clustering, dimensionality reduction, anomaly/association. E.g. customer segmentation.
</details>

<details><summary><b>Q: Reinforcement learning?</b></summary>

A: An **agent** learns by trial-and-error, taking actions in an environment to maximize a cumulative **reward**. No labeled dataset — feedback comes from rewards/penalties. E.g. robotics, game-playing, RLHF.
</details>

<details><summary><b>Q: Semi-supervised learning?</b></summary>

A: Trains on a **small labeled set + large unlabeled set**. Useful when labeling is expensive.
</details>

<details><summary><b>Q: Self-supervised learning?</b></summary>

A: Model generates its own labels from the data (e.g. predict the next/masked token). This is how foundation models are pre-trained.
</details>

<details><summary><b>Q: Classification vs regression?</b></summary>

A: **Classification** predicts a discrete category/label (spam vs not). **Regression** predicts a continuous number (house price).
</details>

<details><summary><b>Q: Binary vs multiclass vs multilabel classification?</b></summary>

A: **Binary** = 2 classes. **Multiclass** = one label from >2 classes. **Multilabel** = each example can have several labels at once.
</details>

<details><summary><b>Q: Clustering?</b></summary>

A: Unsupervised grouping of similar data points (e.g. k-means). No predefined labels.
</details>

<details><summary><b>Q: Overfitting — symptom and cause?</b></summary>

A: Model memorizes training data; **high train accuracy, low test/validation accuracy**. Too complex / trained too long. Fix: more data, regularization, dropout, early stopping, simpler model.
</details>

<details><summary><b>Q: Underfitting — symptom and cause?</b></summary>

A: Model too simple to capture patterns; **poor on both train and test**. Fix: more features, more complex model, train longer, less regularization.
</details>

<details><summary><b>Q: Bias–variance tradeoff?</b></summary>

A: **High bias** = underfitting (too simple). **High variance** = overfitting (too sensitive to training data). Goal: balance both to minimize total error.
</details>

<details><summary><b>Q: What is regularization (L1 vs L2)?</b></summary>

A: Penalizes large weights to reduce overfitting. **L1 (Lasso)** drives weights to zero → feature selection. **L2 (Ridge)** shrinks weights smoothly.
</details>

<details><summary><b>Q: Precision — definition and when it matters?</b></summary>

A: **TP / (TP + FP)** — of predicted positives, how many were right. Matters when **false positives are costly** (e.g. flagging good email as spam).
</details>

<details><summary><b>Q: Recall (sensitivity) — definition and when it matters?</b></summary>

A: **TP / (TP + FN)** — of actual positives, how many were caught. Matters when **false negatives are costly** (e.g. missing a fraud case or a disease).
</details>

<details><summary><b>Q: F1 score?</b></summary>

A: **Harmonic mean of precision and recall.** Use when you need a single balanced metric, especially on **imbalanced** classes.
</details>

<details><summary><b>Q: Accuracy — and its trap?</b></summary>

A: (TP+TN)/total. **Misleading on imbalanced data** — 99% "not fraud" gets 99% accuracy while catching zero fraud. Prefer F1/precision/recall/AUC there.
</details>

<details><summary><b>Q: AUC-ROC?</b></summary>

A: Area under the ROC curve (TPR vs FPR across thresholds). **1.0 = perfect, 0.5 = random.** Threshold-independent measure of ranking quality for classifiers.
</details>

<details><summary><b>Q: Confusion matrix?</b></summary>

A: Table of TP, FP, TN, FN — the basis for computing precision, recall, F1, accuracy.
</details>

<details><summary><b>Q: RMSE / MAE — what kind of metric?</b></summary>

A: **Regression** error metrics (lower = better). RMSE penalizes large errors more (squared); MAE is the plain average absolute error.
</details>

<details><summary><b>Q: R² (coefficient of determination)?</b></summary>

A: Proportion of variance in the target explained by the model. Closer to **1.0 = better** fit (regression).
</details>

<details><summary><b>Q: Batch (offline) vs real-time (online) inference?</b></summary>

A: **Batch** = predict on a large stored dataset at once, no persistent endpoint, latency not critical. **Real-time** = low-latency predictions on individual requests via an always-on endpoint.
</details>

<details><summary><b>Q: Training vs validation vs test set?</b></summary>

A: **Train** fits the model, **validation** tunes hyperparameters / picks the model, **test** gives a final unbiased performance estimate (used once).
</details>

<details><summary><b>Q: What is a hyperparameter vs a parameter?</b></summary>

A: **Parameters** are learned by training (weights). **Hyperparameters** are set before training (learning rate, epochs, tree depth) and are tuned.
</details>

<details><summary><b>Q: Feature engineering?</b></summary>

A: Transforming raw data into features that improve model performance — scaling, encoding categoricals, binning, creating interactions, handling missing values.
</details>

<details><summary><b>Q: Label vs feature?</b></summary>

A: **Feature** = input variable (X). **Label** = target/output you predict (y).
</details>

---

## 2. Generative AI Concepts

<details><summary><b>Q: What is a token?</b></summary>

A: The basic chunk an LLM reads/generates — a word, sub-word, or character piece. **Pricing and context limits are measured in tokens** (~4 chars ≈ 1 token in English).
</details>

<details><summary><b>Q: What is an embedding?</b></summary>

A: A dense numeric **vector** representing the meaning of text/image/audio. Semantically similar items sit close together in vector space. Backbone of semantic search and RAG.
</details>

<details><summary><b>Q: What is a vector database used for?</b></summary>

A: Stores embeddings and does fast **similarity search** (nearest neighbors). On AWS: OpenSearch Serverless, Aurora pgvector, Neptune Analytics, S3 Vectors, Pinecone, Redis.
</details>

<details><summary><b>Q: What is a transformer?</b></summary>

A: The neural-network architecture behind modern LLMs, built on the **self-attention** mechanism, which weighs the relevance of every token to every other token (enables long-range context, parallel training).
</details>

<details><summary><b>Q: What is a foundation model (FM)?</b></summary>

A: A large model **pre-trained on massive broad data** that can be adapted to many downstream tasks via prompting or fine-tuning. LLMs are text FMs.
</details>

<details><summary><b>Q: What is an LLM?</b></summary>

A: Large Language Model — a transformer-based FM trained to predict/generate text. E.g. Claude, Llama, Titan/Nova.
</details>

<details><summary><b>Q: What is a diffusion model?</b></summary>

A: A generative model that learns to **remove noise step-by-step** to create data (mainly images). Powers image generators like Stable Diffusion.
</details>

<details><summary><b>Q: What is a multimodal model?</b></summary>

A: A model that handles more than one data type (e.g. text + image + audio) as input and/or output. E.g. Amazon Nova, Claude vision.
</details>

<details><summary><b>Q: Temperature — what does raising it do?</b></summary>

A: Controls randomness. **Low temp → deterministic, focused** output. **High temp → more diverse/creative** (and riskier) output.
</details>

<details><summary><b>Q: Top-p (nucleus sampling)?</b></summary>

A: Samples from the smallest set of tokens whose cumulative probability ≥ p. **Lower p = safer/narrower**, higher p = more varied.
</details>

<details><summary><b>Q: Top-k?</b></summary>

A: Restricts sampling to the k most probable next tokens. Smaller k = more focused output.
</details>

<details><summary><b>Q: Max tokens / context window?</b></summary>

A: **Context window** = max tokens the model can consider (prompt + output). **Max tokens** = cap on generated output length.
</details>

<details><summary><b>Q: Zero-shot prompting?</b></summary>

A: Ask the task with **no examples** in the prompt.
</details>

<details><summary><b>Q: Few-shot prompting?</b></summary>

A: Provide a **few examples** in the prompt to steer the format/behavior (in-context learning).
</details>

<details><summary><b>Q: Chain-of-thought (CoT) prompting?</b></summary>

A: Ask the model to **reason step-by-step** ("think step by step"), improving multi-step/logic tasks.
</details>

<details><summary><b>Q: What is a hallucination?</b></summary>

A: When a model produces **fluent but false/fabricated** content stated confidently. Mitigate with RAG (grounding), guardrails, lower temperature, citations.
</details>

<details><summary><b>Q: What is RAG?</b></summary>

A: **Retrieval-Augmented Generation** — retrieve relevant docs from a knowledge/vector store and inject them into the prompt so the model answers from **your data**, reducing hallucination. No model retraining needed.
</details>

<details><summary><b>Q: In-context learning?</b></summary>

A: Steering the model **purely via the prompt** (instructions + examples) at inference time — no weight updates.
</details>

<details><summary><b>Q: Pre-training vs fine-tuning?</b></summary>

A: **Pre-training** = costly initial training on huge broad data to create the FM. **Fine-tuning** = further training on a smaller labeled/domain dataset to specialize it (updates weights).
</details>

<details><summary><b>Q: Fine-tuning vs RAG — when to pick which?</b></summary>

A: **Fine-tune** to change style/behavior or teach a task/format. **RAG** to inject up-to-date or proprietary **facts** without retraining. RAG is cheaper for frequently changing knowledge.
</details>

<details><summary><b>Q: Instruction tuning?</b></summary>

A: Fine-tuning an FM on (instruction, response) pairs so it follows natural-language instructions well.
</details>

<details><summary><b>Q: Continued pre-training vs fine-tuning (Bedrock terms)?</b></summary>

A: **Continued pre-training** uses large **unlabeled** domain data to deepen domain knowledge. **Fine-tuning** uses **labeled** examples for specific tasks.
</details>

<details><summary><b>Q: What is RLHF?</b></summary>

A: **Reinforcement Learning from Human Feedback** — humans rank model outputs, a reward model is trained, then the LLM is optimized against it to align with human preferences (helpful/harmless).
</details>

<details><summary><b>Q: What is an AI agent?</b></summary>

A: An LLM-driven system that **plans multi-step tasks and calls tools/APIs** (function calling) to act, observe results, and iterate toward a goal.
</details>

<details><summary><b>Q: ROUGE — what's it for?</b></summary>

A: Evaluates **text summarization** by overlap of n-grams/sequences between generated and reference text (recall-oriented).
</details>

<details><summary><b>Q: BLEU — what's it for?</b></summary>

A: Evaluates **machine translation** by n-gram precision vs reference translations.
</details>

<details><summary><b>Q: BERTScore — what's it for?</b></summary>

A: Uses **contextual embeddings** to measure semantic similarity between generated and reference text (captures meaning, not just word overlap).
</details>

<details><summary><b>Q: Perplexity?</b></summary>

A: Measures how well a language model predicts text — **lower = better** (less "surprised").
</details>

<details><summary><b>Q: What is prompt engineering?</b></summary>

A: Crafting inputs (instructions, context, examples, constraints) to get better/more reliable outputs without changing the model.
</details>

<details><summary><b>Q: Negative prompting?</b></summary>

A: Telling the model what to **avoid** (e.g. exclude a style/topic in image or text generation).
</details>

<details><summary><b>Q: Prompt injection / jailbreaking?</b></summary>

A: An attack where crafted input makes the model ignore its instructions or safety rules. Mitigate with guardrails, input validation, and least-privilege tool access.
</details>

<details><summary><b>Q: Model distillation?</b></summary>

A: Training a smaller "student" model to mimic a larger "teacher" — cheaper/faster inference with much of the quality.
</details>

<details><summary><b>Q: Non-deterministic output — why do LLMs vary?</b></summary>

A: Sampling (temperature/top-p) introduces randomness, so the same prompt can yield different outputs. Set temperature low / use deterministic decoding for reproducibility.
</details>

---

## 3. Amazon Bedrock & GenAI on AWS

<details><summary><b>Q: What is Amazon Bedrock?</b></summary>

A: A **fully managed, serverless** service giving a **single API** to many foundation models (Amazon Nova/Titan, Anthropic Claude, Meta Llama, Mistral, Cohere, AI21, Stability, DeepSeek) plus RAG, agents, guardrails, and customization. No infrastructure to manage.
</details>

<details><summary><b>Q: Do your Bedrock prompts train the base models?</b></summary>

A: **No.** Your prompts/data are not used to train base FMs and are not shared with providers; data stays in your account.
</details>

<details><summary><b>Q: Converse vs InvokeModel API?</b></summary>

A: **Converse** = standardized message format across models (easy model swap, multi-turn). **InvokeModel** = raw single-call invocation with model-specific payload.
</details>

<details><summary><b>Q: Bedrock Knowledge Bases — what does it give you?</b></summary>

A: **Fully managed RAG** — point it at S3/data sources; it handles chunking, embedding, vector storage, retrieval, and **citation-backed** answers. No custom retrieval infra.
</details>

<details><summary><b>Q: Bedrock Agents — what do they do?</b></summary>

A: Break a goal into steps and call your APIs/**Lambda action groups**, query Knowledge Bases, and reason across turns (managed tool use / function calling).
</details>

<details><summary><b>Q: Multi-agent collaboration in Bedrock?</b></summary>

A: A **supervisor** agent can orchestrate up to **5 collaborator** agents (GA March 2025) for complex tasks.
</details>

<details><summary><b>Q: Bedrock Guardrails — what can it enforce?</b></summary>

A: Configurable safety independent of the model: **content filters, denied topics, word filters, PII redaction (sensitive info filters), contextual grounding checks**, and **automated reasoning checks** to catch hallucinations. Can be applied via IAM policy-based enforcement.
</details>

<details><summary><b>Q: What are Automated Reasoning checks in Guardrails?</b></summary>

A: Use **formal logic/math** to verify factual claims against your rules and detect hallucinations — sound reasoning, not another LLM guessing.
</details>

<details><summary><b>Q: Contextual grounding check?</b></summary>

A: A Guardrail that scores whether a response is **grounded in the retrieved source** and relevant to the query — filters ungrounded (hallucinated) answers in RAG.
</details>

<details><summary><b>Q: Bedrock Flows / Prompt Management?</b></summary>

A: **Flows** = visually orchestrate prompts, KBs, agents, Lambda into a workflow. **Prompt Management** = version and reuse prompts.
</details>

<details><summary><b>Q: Bedrock Model Evaluation?</b></summary>

A: Compare FMs using automatic metrics or **human evaluation** (and LLM-as-a-judge) to pick the best model for your task.
</details>

<details><summary><b>Q: What is SageMaker JumpStart?</b></summary>

A: A hub of **pre-built models and solutions** (including open FMs) deployable/fine-tunable in a few clicks. Bedrock = managed API FMs; JumpStart = models you deploy on your SageMaker infra.
</details>

<details><summary><b>Q: Amazon Q Business vs Q Developer?</b></summary>

A: **Q Business** = enterprise GenAI assistant over your company data (connectors, RAG) with access controls. **Q Developer** = AI coding assistant in the IDE/CLI/console (code, debug, AWS help).
</details>

<details><summary><b>Q: What powers Amazon Q under the hood?</b></summary>

A: **Bedrock** foundation models. Q Business, Q Developer, and GenAI features in QuickSight/Connect all call FMs via Bedrock.
</details>

<details><summary><b>Q: PartyRock?</b></summary>

A: A no-code Bedrock playground to build/share GenAI apps quickly (learning/prototyping).
</details>

<details><summary><b>Q: Bedrock on-demand pricing?</b></summary>

A: Pay **per token** (separate input/output rates), no commitment. Best for sporadic/unpredictable traffic. Images priced per image.
</details>

<details><summary><b>Q: Bedrock batch inference pricing?</b></summary>

A: Process large jobs asynchronously at **~50% lower** cost than on-demand for supported models. Best when you don't need real-time responses.
</details>

<details><summary><b>Q: Bedrock Provisioned Throughput — what and when?</b></summary>

A: Reserve **Model Units (MUs)** for a fixed hourly price (1- or 6-month commitment) to guarantee capacity/throughput. **Required for hosting custom/fine-tuned models**; best for steady high-volume workloads.
</details>

<details><summary><b>Q: A Model Unit (MU) measures what?</b></summary>

A: A guaranteed throughput level: input tokens processed and output tokens generated **per minute** for a given model.
</details>

<details><summary><b>Q: Custom Model Import cost in Bedrock?</b></summary>

A: **No charge to import**; you pay for inference based on active model copies (billed in 5-minute windows). Custom models need Provisioned Throughput to serve.
</details>

<details><summary><b>Q: Bedrock cross-region inference?</b></summary>

A: Automatically routes requests across regions to improve throughput/availability during traffic bursts.
</details>

---

## 4. AWS Managed AI Services (Trigger → Service)

<details><summary><b>Q: Extract text, forms, tables, handwriting from scanned docs/PDFs → ?</b></summary>

A: **Amazon Textract** (OCR + form/table extraction).
</details>

<details><summary><b>Q: Just detect words in an image (basic OCR) → ?</b></summary>

A: **Amazon Rekognition** (text-in-image) for simple detection; **Textract** for document structure/forms.
</details>

<details><summary><b>Q: Sentiment analysis, entities, key phrases, language, PII in text → ?</b></summary>

A: **Amazon Comprehend** (NLP).
</details>

<details><summary><b>Q: Detect and redact PII in text → ?</b></summary>

A: **Amazon Comprehend** (PII detection/redaction). (For PII in S3 objects broadly → **Macie**.)
</details>

<details><summary><b>Q: Medical entities/PHI from clinical text → ?</b></summary>

A: **Amazon Comprehend Medical**.
</details>

<details><summary><b>Q: Translate text between languages → ?</b></summary>

A: **Amazon Translate** (neural machine translation).
</details>

<details><summary><b>Q: Speech → text (transcription) → ?</b></summary>

A: **Amazon Transcribe** (ASR; speaker diarization, custom vocab; Transcribe Medical/Call Analytics).
</details>

<details><summary><b>Q: Text → lifelike speech → ?</b></summary>

A: **Amazon Polly** (text-to-speech; SSML, neural voices).
</details>

<details><summary><b>Q: Object/scene/face detection, content moderation in images & video → ?</b></summary>

A: **Amazon Rekognition** (computer vision).
</details>

<details><summary><b>Q: Build a chatbot / voice IVR bot (intents, slots) → ?</b></summary>

A: **Amazon Lex** (conversational AI; same tech as Alexa).
</details>

<details><summary><b>Q: Intelligent enterprise search with natural-language answers over docs → ?</b></summary>

A: **Amazon Kendra** (ML-powered semantic enterprise search).
</details>

<details><summary><b>Q: Real-time personalized product/content recommendations → ?</b></summary>

A: **Amazon Personalize** (recommendation engine from your interaction data).
</details>

<details><summary><b>Q: Detect online fraud (fake accounts, payment fraud) with little ML expertise → ?</b></summary>

A: **Amazon Fraud Detector**.
</details>

<details><summary><b>Q: Add human review to low-confidence ML predictions → ?</b></summary>

A: **Amazon Augmented AI (A2I)** — human-in-the-loop review workflows.
</details>

<details><summary><b>Q: Time-series forecasting (managed) → ?</b></summary>

A: **Amazon Forecast** (note: legacy; also **SageMaker Canvas / DeepAR** for forecasting).
</details>

<details><summary><b>Q: Detect anomalies in metrics/business data → ?</b></summary>

A: **Amazon Lookout for Metrics** (and Lookout for Equipment/Vision for industrial/visual).
</details>

<details><summary><b>Q: Code review, security scan, and coding assistance → ?</b></summary>

A: **Amazon Q Developer** (formerly CodeWhisperer/CodeGuru capabilities).
</details>

<details><summary><b>Q: Summarize meeting audio and generate transcript with speakers → ?</b></summary>

A: **Amazon Transcribe** (speaker ID) → **Comprehend/Bedrock** for summarization.
</details>

<details><summary><b>Q: Content moderation on user-uploaded images/video → ?</b></summary>

A: **Amazon Rekognition** content moderation (often + **A2I** for human review).
</details>

<details><summary><b>Q: Convert a chatbot's text reply to voice → ?</b></summary>

A: **Amazon Polly**.
</details>

<details><summary><b>Q: Classify support tickets by topic automatically → ?</b></summary>

A: **Amazon Comprehend** custom classification.
</details>

<details><summary><b>Q: Detect PII across all objects in an S3 bucket → ?</b></summary>

A: **Amazon Macie** (data security/discovery for S3), not Comprehend.
</details>

<details><summary><b>Q: Build custom entity recognition beyond built-in types → ?</b></summary>

A: **Amazon Comprehend custom entity recognition**.
</details>

---

## 5. Amazon SageMaker

<details><summary><b>Q: What is SageMaker Studio?</b></summary>

A: The web-based **IDE for the full ML lifecycle** — notebooks, data prep, training, tuning, deploy, and monitoring in one place.
</details>

<details><summary><b>Q: SageMaker Canvas?</b></summary>

A: **No-code** ML — build models and generate predictions via a visual UI (for analysts, no coding).
</details>

<details><summary><b>Q: What are SageMaker built-in algorithms — name a few?</b></summary>

A: Ready-to-train algorithms: **XGBoost, Linear Learner, K-Means, PCA, DeepAR (forecasting), BlazingText, Image Classification, Object Detection, Random Cut Forest (anomaly), Factorization Machines**.
</details>

<details><summary><b>Q: Random Cut Forest — used for?</b></summary>

A: **Anomaly detection** (unsupervised) in SageMaker.
</details>

<details><summary><b>Q: DeepAR — used for?</b></summary>

A: **Time-series forecasting** with recurrent neural networks.
</details>

<details><summary><b>Q: Script mode vs BYOC?</b></summary>

A: **Script mode** = bring your training script and run it in an AWS-managed framework container (TensorFlow/PyTorch/etc). **BYOC** = bring your own custom Docker container for full control.
</details>

<details><summary><b>Q: What is Automatic Model Tuning (AMT)?</b></summary>

A: **Hyperparameter optimization** — runs many training jobs to find the best hyperparameters (Bayesian, random, grid, or Hyperband search).
</details>

<details><summary><b>Q: SageMaker Feature Store?</b></summary>

A: Managed **repository to store, share, retrieve, and reuse ML features** — with an **online store** (low-latency real-time) and **offline store** (S3, for training/batch). Prevents train/serve skew.
</details>

<details><summary><b>Q: SageMaker Data Wrangler?</b></summary>

A: **Visual data prep** — import, clean, transform, and analyze data with minimal code. Handles class imbalance via **random under/oversampling and SMOTE**; exports features to Feature Store.
</details>

<details><summary><b>Q: SageMaker Ground Truth?</b></summary>

A: **Data labeling** service — human labelers (or automated labeling) to build accurate training datasets; supports images, text, video.
</details>

<details><summary><b>Q: SageMaker Clarify — two jobs?</b></summary>

A: **Bias detection** (pre-training data bias + post-training model bias) and **explainability** (feature importance via SHAP). Runs without code.
</details>

<details><summary><b>Q: SageMaker Model Monitor — what does it watch?</b></summary>

A: Monitors deployed endpoints for **data quality drift, model quality drift, bias drift, and feature attribution drift**; emits CloudWatch metrics/alarms.
</details>

<details><summary><b>Q: Data drift vs concept drift?</b></summary>

A: **Data (covariate) drift** = input distribution changes. **Concept drift** = the relationship between inputs and target changes. Both degrade a model → retrain.
</details>

<details><summary><b>Q: SageMaker Model Registry?</b></summary>

A: **Catalog/versioning of models** with approval status (approved/rejected) for CI/CD deployment governance.
</details>

<details><summary><b>Q: SageMaker Pipelines?</b></summary>

A: **Managed CI/CD workflow orchestration** for ML — defines repeatable steps (process → train → evaluate → register → deploy) as a DAG.
</details>

<details><summary><b>Q: SageMaker Neo?</b></summary>

A: **Compiles/optimizes trained models** to run faster on specific target hardware (cloud instances, edge devices) — "train once, run anywhere."
</details>

<details><summary><b>Q: SageMaker Model Cards?</b></summary>

A: Documented record of a model's intended use, training data, metrics, and risk ratings — governance/transparency artifact.
</details>

<details><summary><b>Q: SageMaker real-time endpoint — when?</b></summary>

A: Sustained traffic needing **low, consistent millisecond latency**. Always-on instance(s); payload up to 25 MB; ~60s response (8 min streaming). You pay for provisioned capacity even when idle.
</details>

<details><summary><b>Q: SageMaker Serverless Inference — when?</b></summary>

A: **Intermittent/unpredictable** traffic; SageMaker manages infra and you pay only for usage (no idle cost). Payload ≤ 4 MB, ≤ 60s. Watch for **cold starts**.
</details>

<details><summary><b>Q: SageMaker Asynchronous Inference — when?</b></summary>

A: **Large payloads (up to 1 GB) and long processing (up to 1 hr)**; requests/responses queued via S3. Can **autoscale to zero** when idle.
</details>

<details><summary><b>Q: SageMaker Batch Transform — when?</b></summary>

A: **Offline predictions on a large dataset** all at once; no persistent endpoint; input/output in S3. Best when latency doesn't matter and no real-time endpoint is needed.
</details>

<details><summary><b>Q: If you see "large payload, long-running, queue it, scale to zero" → which endpoint?</b></summary>

A: **Asynchronous Inference.**
</details>

<details><summary><b>Q: If you see "spiky/unpredictable traffic, no idle cost, tolerate cold starts" → which endpoint?</b></summary>

A: **Serverless Inference.**
</details>

<details><summary><b>Q: If you see "predict on a whole S3 dataset overnight, no endpoint" → which?</b></summary>

A: **Batch Transform.**
</details>

<details><summary><b>Q: Multi-model vs multi-container endpoint?</b></summary>

A: **Multi-model endpoint** hosts many models behind one endpoint (loaded on demand — cost-efficient at scale). **Multi-container** hosts different containers, optionally chained (inference pipeline).
</details>

<details><summary><b>Q: SageMaker inference pipeline?</b></summary>

A: Chain of containers (e.g. preprocess → model → postprocess) served as a **single endpoint** for consistent train/serve transforms.
</details>

<details><summary><b>Q: Deployment guardrails (blue/green, canary, linear)?</b></summary>

A: Safe endpoint update strategies: **blue/green** (shift all traffic after validation), **canary** (small % first), **linear** (gradual steps) — with auto-rollback on alarms.
</details>

<details><summary><b>Q: Shadow testing?</b></summary>

A: Send a copy of live traffic to a new model **without affecting users**, to compare performance before promoting it.
</details>

<details><summary><b>Q: SageMaker Debugger?</b></summary>

A: Captures training tensors to **detect training issues** (vanishing gradients, overfitting, loss-not-decreasing) in real time.
</details>

<details><summary><b>Q: SageMaker Experiments?</b></summary>

A: Tracks/compares training runs (parameters, metrics, artifacts) for reproducibility.
</details>

<details><summary><b>Q: Managed Spot Training in SageMaker?</b></summary>

A: Uses **Spot instances** for training to cut cost up to ~90%, with checkpointing to survive interruptions.
</details>

<details><summary><b>Q: Distributed training — data vs model parallelism?</b></summary>

A: **Data parallel** = split the data across GPUs (large datasets). **Model parallel** = split the model across GPUs (model too big for one GPU).
</details>

<details><summary><b>Q: SageMaker JumpStart vs building from scratch?</b></summary>

A: JumpStart gives **pre-trained models + example solutions** deployable in a few clicks — faster than training from zero.
</details>

<details><summary><b>Q: Inference Recommender?</b></summary>

A: Runs load tests to **recommend the best instance type and config** for your model's latency/throughput/cost targets.
</details>

---

## 6. Responsible AI, Security & Governance

<details><summary><b>Q: Core dimensions of Responsible AI (AWS)?</b></summary>

A: **Fairness, Explainability, Robustness, Privacy & Security, Governance, Transparency, Veracity, Safety, Controllability.**
</details>

<details><summary><b>Q: Which service detects bias and explains predictions?</b></summary>

A: **SageMaker Clarify** (bias metrics + SHAP explainability). For GenAI response safety → **Bedrock Guardrails**.
</details>

<details><summary><b>Q: Which artifact documents a model's intended use, data, and risks?</b></summary>

A: **SageMaker Model Cards.**
</details>

<details><summary><b>Q: AWS Shared Responsibility Model — who owns what?</b></summary>

A: AWS = security **OF** the cloud (hardware, infra, managed service internals). Customer = security **IN** the cloud (data, IAM, encryption config, access). For AI: you own your data, prompts, and access controls.
</details>

<details><summary><b>Q: What is IAM used for in AI workloads?</b></summary>

A: **Least-privilege access control** — who/what can call Bedrock models, SageMaker resources, S3 data. Use roles, not long-lived keys.
</details>

<details><summary><b>Q: What does KMS do?</b></summary>

A: Manages **encryption keys** — encrypt data at rest (S3, SageMaker volumes, model artifacts) and control key access.
</details>

<details><summary><b>Q: Amazon Macie — role in AI security?</b></summary>

A: Discovers and protects **sensitive data / PII in S3** using ML — use to scan training data buckets.
</details>

<details><summary><b>Q: PrivateLink / VPC endpoints — why for AI?</b></summary>

A: Keep traffic to **Bedrock / SageMaker on the AWS private network** (no public internet) — data doesn't traverse the open internet.
</details>

<details><summary><b>Q: CloudTrail vs CloudWatch vs Config?</b></summary>

A: **CloudTrail** = who did what (API audit logs). **CloudWatch** = metrics/logs/alarms (performance & operations). **Config** = resource configuration state & compliance over time.
</details>

<details><summary><b>Q: If asked "audit who invoked a Bedrock model / API call history" → ?</b></summary>

A: **AWS CloudTrail.**
</details>

<details><summary><b>Q: If asked "is this resource compliant / track config changes" → ?</b></summary>

A: **AWS Config.**
</details>

<details><summary><b>Q: GenAI Security Scoping Matrix — what is it?</b></summary>

A: An AWS framework classifying GenAI use cases into **5 scopes** by ownership/control, mapped to 5 disciplines: **governance & compliance, legal & privacy, risk management, controls, resilience**.
</details>

<details><summary><b>Q: Scoping Matrix — Scope 1?</b></summary>

A: **Consumer app** — using a public third-party GenAI service (e.g. ChatGPT, PartyRock); you don't own the model or data.
</details>

<details><summary><b>Q: Scoping Matrix — Scope 2?</b></summary>

A: **Enterprise app** — using a third-party GenAI app/SaaS under an enterprise license/subscription.
</details>

<details><summary><b>Q: Scoping Matrix — Scope 3?</b></summary>

A: **Pre-trained models** — building your own app on an FM via API (e.g. Amazon Bedrock).
</details>

<details><summary><b>Q: Scoping Matrix — Scope 4?</b></summary>

A: **Fine-tuned models** — customizing an FM with your proprietary data.
</details>

<details><summary><b>Q: Scoping Matrix — Scope 5?</b></summary>

A: **Self-trained models** — building and training your own FM from scratch on data you own; you own every aspect.
</details>

<details><summary><b>Q: On-Demand vs Reserved vs Spot vs Savings Plans — quick cost map?</b></summary>

A: **On-Demand** = pay-as-you-go, no commitment. **Reserved/Savings Plans** = commit 1–3 yrs for big discounts (steady workloads). **Spot** = spare capacity up to ~90% off but can be interrupted (great for fault-tolerant training).
</details>

<details><summary><b>Q: Cheapest option for interruptible model training?</b></summary>

A: **Spot instances** (SageMaker Managed Spot Training) with checkpointing.
</details>

<details><summary><b>Q: Which encryption keeps data safe "in transit"?</b></summary>

A: **TLS** for data in transit; **KMS-managed encryption** for data at rest.
</details>

<details><summary><b>Q: Toxicity / harmful content filtering for GenAI responses → ?</b></summary>

A: **Bedrock Guardrails** (content filters + denied topics + PII redaction).
</details>