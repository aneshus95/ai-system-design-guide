# AWS AI/ML Service Cheatsheet

*What each service does & when to reach for it — scan this the day before the exam (AIF-C01 + MLA-C01).*

---

## 1. Generative AI

| Service | One-line what it does | Reach for it when… (trigger phrase) |
|---|---|---|
| **Amazon Bedrock** | Fully managed API to foundation models (Anthropic, Meta, Amazon Nova/Titan, Mistral, AI21, Cohere, Stability) — no infra to manage. | "Serverless access to multiple FMs", "call an LLM via API", "build a GenAI app without managing model servers". |
| **Bedrock Knowledge Bases** | Managed RAG — ingests your docs, chunks + embeds them into a vector store, retrieves context for the FM. | "Ground the model in my company data", "RAG without building the pipeline", "reduce hallucinations with my documents". |
| **Bedrock Agents** | Orchestrates multi-step tasks — breaks a goal into steps, calls APIs/Lambda (action groups), chains reasoning. | "LLM that takes actions / calls tools", "multi-step workflow", "book a flight / query a DB via the model". |
| **Bedrock Guardrails** | Safety layer — filters harmful content, blocks denied topics, redacts PII, checks for hallucination/contextual grounding. | "Content filtering", "block toxic or off-topic output", "PII redaction on model I/O", "responsible-AI policy". |
| **PartyRock** | Free, no-code Bedrock playground to build & share small GenAI apps in the browser. | "Learn/prototype GenAI with no AWS account or code", "hands-on FM experimentation". |
| **Amazon Q Business** | Managed GenAI assistant for the enterprise — connects 40+ data sources (S3, SharePoint, Salesforce, Slack…) to answer, summarize, act. | "Employee assistant over internal data", "chat with company knowledge", "connectors to SaaS + citations". |
| **Amazon Q Developer** | GenAI coding assistant in IDE/CLI/Console — code suggestions, autonomous agents, security scans, AWS troubleshooting. | "AI pair programmer", "generate/debug code", "modernize a codebase", "explain an AWS error". |
| **SageMaker JumpStart** | Hub of pre-trained, deployable FMs & solution templates — one-click deploy or fine-tune inside SageMaker. | "Pre-built model I can fine-tune/deploy in SageMaker", "GenAI with full MLOps control". |

## 2. Managed AI — Language & Text

| Service | One-line what it does | Reach for it when… (trigger phrase) |
|---|---|---|
| **Comprehend** | NLP: entities, key phrases, sentiment, language, PII, topic modeling, custom classification. | "Extract sentiment/entities from text", "detect PII in documents", "topic modeling". |
| **Comprehend Medical** | HIPAA-eligible NLP for clinical text — extracts conditions, meds, dosages, ICD-10/RxNorm codes. | "Understand medical/clinical notes", "extract PHI + medical ontology codes". |
| **Translate** | Neural machine translation between languages, with custom terminology. | "Translate text at scale", "localize app content", "real-time language translation". |
| **Textract** | OCR+ — extracts text, forms, tables, and key-value pairs from scanned docs/PDFs (structure-aware). | "Read text/tables/forms from a document", "digitize invoices/IDs", "OCR with structure". |
| **Kendra** | ML-powered enterprise search — natural-language queries return direct answers + confidence + source. | "Intelligent search over documents", "NL Q&A search with citations", "retriever for RAG". |

## 3. Managed AI — Speech & Vision

| Service | One-line what it does | Reach for it when… (trigger phrase) |
|---|---|---|
| **Transcribe** | Speech-to-text — diarization, custom vocab, PII redaction; Transcribe Medical for clinical audio. | "Audio → text", "meeting/call transcription", "subtitles/captions". |
| **Polly** | Text-to-speech with lifelike neural voices, SSML, and speech marks. | "Text → natural speech", "voice output / IVR narration". |
| **Rekognition** | Image & video analysis — objects, faces, celebrities, moderation, text-in-image, face compare. | "Detect objects/faces in images or video", "content moderation", "facial recognition". |
| **Lex** | Conversational chatbot engine (ASR + NLU) — same tech as Alexa; intents & slots. | "Build a chatbot / voice bot", "intent + slot dialog", "IVR conversation flows". |

## 4. Managed AI — Other

| Service | One-line what it does | Reach for it when… (trigger phrase) |
|---|---|---|
| **Personalize** | Real-time recommendations & personalization (same tech as Amazon.com). | "Product/content recommendations", "personalized ranking", "next-best-action". |
| **Fraud Detector** | Managed ML to detect online fraud (fake accounts, payment fraud) from your event data. | "Detect fraudulent transactions/signups", "risk score events without building a model". |
| **Augmented AI (A2I)** | Human-in-the-loop review workflows for low-confidence ML predictions. | "Route uncertain predictions to humans", "human review of ML output", "human oversight/QA". |

## 5. SageMaker (core + key features)

| Service / Feature | One-line what it does | Reach for it when… (trigger phrase) |
|---|---|---|
| **SageMaker (core)** | End-to-end platform to build, train, tune, deploy & manage ML models (Studio, notebooks, training jobs, endpoints). | "Full custom ML lifecycle", "train my own model", "I need control over training/inference". |
| **Data Wrangler** | Visual data prep — import, transform, and analyze features with little/no code. | "Prep/clean/transform ML data visually", "feature engineering pipeline". |
| **Feature Store** | Central repo for curated features — online (low-latency) + offline (training) stores. | "Reuse/share features", "consistent features for training & inference". |
| **Ground Truth** | Managed data labeling (human + automated/active learning). | "Label a training dataset", "human labeling at scale". |
| **Clarify** | Detects **bias** (pre/post-training) and provides model **explainability** (SHAP feature attribution). | "Is my model biased?", "explain predictions", "fairness + explainability report". |
| **Model Monitor** | Monitors deployed models for **drift** — data quality, model quality, bias & feature-attribution drift. | "Detect drift in production", "alert when accuracy/data degrades". *(Closing to new customers 7/30/26.)* |
| **Model Registry** | Catalog & version models, manage approval status for deployment. | "Version/approve models", "central model catalog", "CI/CD gate for models". |
| **Pipelines** | CI/CD workflow orchestration for ML (build→train→eval→deploy as DAG). | "Automate/repeat the ML workflow", "MLOps pipeline". |
| **Automatic Model Tuning (AMT)** | Hyperparameter optimization — searches for best params (Bayesian, random, grid, Hyperband). | "Tune hyperparameters automatically", "optimize model accuracy". |
| **Neo** | Compiles/optimizes trained models for target hardware (edge, CPU, GPU) for faster, smaller inference. | "Optimize a model for edge/specific hardware", "speed up inference / shrink model". |
| **Debugger** | Captures & inspects training tensors in real time to catch issues (vanishing gradients, overfitting). | "Debug/profile a training job", "why is training misbehaving?". |
| **Model Cards** | Documents model details, intended use, risk rating for governance. | "Document a model for governance/audit", "model documentation". |
| **Inference Recommender** | Automated load-testing to pick best instance type/config for cost & latency. | "Which instance for my endpoint?", "right-size inference for cost/perf". |

## 6. Data & Analytics

| Service | One-line what it does | Reach for it when… (trigger phrase) |
|---|---|---|
| **S3** | Object storage — the data lake foundation for datasets, models, and artifacts. | "Store data/models/artifacts", "data lake", "training data source". |
| **Glue** | Serverless ETL + Data Catalog (Spark-based) to discover, prep, and move data. | "Serverless ETL", "schema catalog", "join/transform data for ML". |
| **DataBrew** | Visual, no-code data prep with 250+ built-in transforms. | "Clean/normalize data without code", "point-and-click data prep". |
| **Glue Data Quality** | Auto-generates & runs data-quality rules on Catalog/pipeline data. | "Validate data quality", "catch bad data before training". |
| **Lake Formation** | Central permissions & governance layer over the S3 data lake. | "Fine-grained data-lake access control", "govern who sees which rows/columns". |
| **Athena** | Serverless SQL queries directly on S3 (Presto/Trino). | "Ad-hoc SQL on S3", "query data lake without a cluster". |
| **EMR** | Managed big-data clusters (Spark, Hadoop, Hive, Presto) for large-scale processing. | "Big-data / large-scale Spark jobs", "petabyte processing", "custom cluster control". |
| **Kinesis Data Streams** | Durable, ordered, replayable real-time stream (shards; retain up to 365 days). | "Ingest real-time data with replay/ordering", "custom stream consumers". |
| **Amazon Data Firehose** | Managed delivery of streaming data to S3/Redshift/OpenSearch/Splunk/Snowflake (near-real-time, no code). | "Just load streaming data into a store", "easiest stream-to-destination", "buffer + deliver". *(Formerly Kinesis Data Firehose.)* |
| **Managed Service for Apache Flink** | Managed stateful stream processing — windows, joins, aggregations, exactly-once. | "Real-time analytics/aggregation on a stream", "windowed/stateful processing". *(Formerly Kinesis Data Analytics.)* |
| **MSK** | Fully managed Apache Kafka. | "Already using Kafka", "open-source Kafka ecosystem in the cloud". |
| **Redshift** | Petabyte-scale cloud data warehouse (SQL analytics, Redshift ML). | "Data warehouse", "BI/analytics SQL at scale", "SQL-based ML on warehouse data". |
| **OpenSearch** | Search & log analytics engine (Elasticsearch fork); vector search for RAG. | "Log/operational analytics", "full-text or vector search", "observability dashboards". |
| **QuickSight** | Serverless BI dashboards with ML insights + natural-language Q. | "Visualize data / build dashboards", "self-service BI". |

## 7. Security, Identity & Governance

| Service | One-line what it does | Reach for it when… (trigger phrase) |
|---|---|---|
| **IAM** | Identity & access — users, roles, policies (who can do what). | "Control access / permissions", "least-privilege role for a job". |
| **KMS** | Managed encryption keys for data at rest/in transit. | "Encrypt data / manage keys", "customer-managed keys (CMK)". |
| **Macie** | ML-based discovery & protection of sensitive data (PII) in S3. | "Find PII/sensitive data in S3", "data-privacy scanning". |
| **PrivateLink** | Private connectivity to AWS services via VPC endpoints (no internet). | "Keep traffic off the public internet", "private endpoint to Bedrock/SageMaker". |
| **Secrets Manager** | Stores & rotates secrets (DB creds, API keys). | "Store/rotate credentials/API keys", "no hard-coded secrets". |
| **CloudTrail** | Records **API calls / who-did-what** across the account (audit log). | "Audit trail of actions", "who called this API?", "security forensics". |
| **CloudWatch** | Metrics, logs, alarms, dashboards — **operational monitoring**. | "Monitor performance/logs", "alarm on a metric", "endpoint latency/errors". |
| **Config** | Records resource **configuration state** & evaluates compliance rules over time. | "Is my resource configured compliantly?", "config history / drift from policy". |
| **Inspector** | Automated vulnerability scanning of EC2/containers/Lambda. | "Scan for CVEs/vulnerabilities", "software security assessment". |
| **Audit Manager** | Automates evidence collection for audits (compliance frameworks). | "Prep for a compliance audit", "collect audit evidence automatically". |
| **Artifact** | Self-service portal for AWS compliance reports (SOC, PCI, ISO). | "Download AWS compliance docs", "get SOC/PCI report". |
| **Trusted Advisor** | Best-practice checks across cost, security, performance, limits. | "Recommendations to optimize/secure account", "health-check my account". |

## 8. Cost & Purchasing

| Option | One-line what it does | Reach for it when… (trigger phrase) |
|---|---|---|
| **Cost Explorer** | Visualize & analyze historical spend + forecasts. | "See/analyze/forecast my costs", "where is spend going?". |
| **Budgets** | Set custom cost/usage thresholds with alerts. | "Alert me when spend exceeds X", "budget guardrail". |
| **Spot Instances** | Spare capacity at up to ~90% off — interruptible. | "Cheap compute for fault-tolerant/training jobs", "cost-optimize batch/training". |
| **Reserved Instances** | 1–3 yr commitment for steady-state EC2 at big discount. | "Predictable long-running workload", "commit for savings". |
| **SageMaker Savings Plans** | Commit to $/hr of SageMaker usage (1–3 yr) for up to ~64% off. | "Cut steady SageMaker cost", "commit to ML compute spend". |

---

## Confusable-service quick calls

| If you see… | Pick this | Because… |
|---|---|---|
| **Rekognition vs Textract** | Rekognition = *analyze image content* (objects/faces/moderation); Textract = *extract text/forms/tables* from documents. | Both "read images," but Textract is document OCR + structure. |
| **Comprehend vs Kendra** | Comprehend = *analyze/understand* text (sentiment, entities, PII); Kendra = *search & answer* over documents. | Comprehend extracts insights; Kendra retrieves answers. |
| **Kendra vs Q Business vs OpenSearch** | Kendra = ML enterprise **search/retrieval** (RAG retriever); Q Business = full **GenAI assistant** with connectors + generation; OpenSearch = **log/vector search** engine you configure. | Q Business generates answers; Kendra retrieves; OpenSearch is the DIY engine. |
| **CloudTrail vs Config vs CloudWatch** | CloudTrail = *who did what* (API audit); Config = *how resources are configured* (compliance/state); CloudWatch = *how it's performing* (metrics/logs/alarms). | Audit vs configuration vs performance. |
| **Clarify vs Model Monitor vs A2I** | Clarify = **bias + explainability** analysis; Model Monitor = **drift** detection in production; A2I = **human review** of low-confidence predictions. | Fairness vs drift vs human-in-the-loop. |
| **Bedrock vs SageMaker** | Bedrock = serverless **use** of foundation models via API; SageMaker = **build/train/deploy** your own models (full control). | Buy/consume FMs vs build custom ML. |
| **RAG vs Fine-tuning** | RAG = inject **fresh/proprietary knowledge** at query time (Knowledge Bases); Fine-tuning = change **behavior/style/domain** by retraining on labeled data. | RAG for facts/recency; fine-tune for tone/task adaptation. |
| **Real-time vs Serverless vs Async vs Batch endpoints** | Real-time = low-latency, always-on; Serverless = intermittent/spiky, scale-to-zero; Async = large payloads/long processing (queued); Batch Transform = offline predictions on a whole dataset. | Latency + payload + traffic pattern decide it. |
| **Kinesis Streams vs Firehose vs MSK vs Flink** | Streams = durable **replayable** ingest; Firehose = managed **delivery** to a store (no code); MSK = managed **Kafka**; Flink = **stateful processing** (windows/joins). | Ingest vs deliver vs Kafka vs process. |
| **Glue vs EMR vs DataBrew** | Glue = **serverless ETL** + catalog (code/Spark); EMR = **big-data clusters** (max control/scale); DataBrew = **no-code** visual prep. | Serverless ETL vs cluster vs point-and-click. |

---

## References

- [Amazon Bedrock](https://aws.amazon.com/bedrock/)
- [Amazon SageMaker AI](https://aws.amazon.com/sagemaker/)
- [Amazon Q](https://aws.amazon.com/q/)
- [AWS Machine Learning services](https://aws.amazon.com/machine-learning/ai-services/)
- [Choosing a RAG option on AWS](https://docs.aws.amazon.com/prescriptive-guidance/latest/retrieval-augmented-generation-options/choosing-option.html)
- [AIF-C01 exam guide](https://aws.amazon.com/certification/certified-ai-practitioner/)
- [MLA-C01 exam guide](https://aws.amazon.com/certification/certified-machine-learning-engineer-associate/)
