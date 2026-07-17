# MLA-C01 Practice Questions

A bank of ~80 scenario-based practice questions for the **AWS Certified Machine Learning Engineer – Associate (MLA-C01)** exam, organized by the four official exam domains in proportion to their scored weight, plus a mixed mini-exam that blends domains the way the real test does.

> **How to use this bank**
> 1. Read the scenario carefully and note the **qualifier** ("MOST cost-effective", "LEAST operational overhead", "highest throughput", "minimal latency"). On the real exam the qualifier is usually what separates the right answer from a technically-workable-but-wrong distractor.
> 2. Pick your answer(s) *before* expanding the explanation. **Multiple response** questions tell you how many to pick — select exactly that many.
> 3. Expand **Answer & explanation** to check. Read the "why the distractors are wrong" notes even when you got it right — that is where the exam-day reflexes are built.
> 4. Target **~70%+** correct per domain before moving on. Re-drill any domain below that.
>
> **Question types used** (same mix as the real exam): **[MCQ]** multiple choice (1 of 4), **[MRQ]** multiple response (choose 2+ of 5), **[ORDER]** ordering/sequencing, **[CASE]** extended scenario/case-style.
>
> **Domain weighting (matches the official exam guide):** Domain 1 Data Preparation 28% · Domain 2 ML Model Development 26% · Domain 3 Deployment & Orchestration 22% · Domain 4 Monitoring, Maintenance & Security 24%.

---

## Table of Contents
- [Domain 1 — Data Preparation (28%)](#domain-1)
- [Domain 2 — ML Model Development (26%)](#domain-2)
- [Domain 3 — Deployment & Orchestration (22%)](#domain-3)
- [Domain 4 — Monitoring, Maintenance & Security (24%)](#domain-4)
- [Mixed mini-exam](#mixed)
- [Score guide](#score-guide)

---

## Domain 1 — Data Preparation (28%) <a name="domain-1"></a>

### Q1. [MCQ]
A data science team stores 4 TB of tabular training data in Amazon S3 as gzip-compressed CSV. Athena queries and SageMaker training jobs both scan the full dataset but only need ~12 of the 90 columns. The team wants to reduce query cost and training I/O with the **LEAST** ongoing effort. What should they do?

- **A.** Keep CSV but enable S3 Transfer Acceleration.
- **B.** Convert the data to Apache Parquet (columnar) using an AWS Glue ETL job.
- **C.** Split the CSV into one file per column.
- **D.** Increase the SageMaker training instance size so scans finish faster.

<details><summary>Answer & explanation</summary>

**Correct: B.** Parquet is a **columnar** format, so both Athena and SageMaker read only the ~12 needed columns instead of scanning all 90. Athena bills by bytes scanned, so column pruning + compression cuts cost directly, and training I/O drops. A one-time Glue job (or DataBrew/Data Wrangler export) does the conversion.

- **A** is wrong — Transfer Acceleration speeds up S3 *uploads over distance*; it does nothing for scan cost or column pruning.
- **C** is wrong — hand-splitting by column is high operational overhead, brittle, and reinvents what Parquet does natively.
- **D** is wrong — a bigger instance costs *more* and still reads every column; it treats the symptom, not the format.
</details>

### Q2. [MCQ]
A retailer ingests clickstream events that must land in Amazon S3 for analytics. There is **no** custom real-time processing, ordering, or replay requirement — data just needs to be reliably delivered, optionally compressed and format-converted, with the **LEAST** operational overhead. Which service fits best?

- **A.** Amazon Kinesis Data Streams with a custom Lambda consumer.
- **B.** Amazon Managed Streaming for Apache Kafka (MSK).
- **C.** Amazon Data Firehose (formerly Kinesis Data Firehose).
- **D.** Amazon Managed Service for Apache Flink.

<details><summary>Answer & explanation</summary>

**Correct: C.** Amazon Data Firehose is the fully managed "easy button" to deliver streaming data into S3/Redshift/OpenSearch/Splunk. It buffers, can compress and convert to Parquet/ORC, and requires **no shard or consumer management** — the lowest ops overhead for simple delivery.

- **A** is wrong — Kinesis Data Streams gives you ordering, replay, and sub-second latency, but you must manage shards and write/operate the consumer. Overkill here.
- **B** is wrong — MSK requires you to build and operate all consumers (EC2/EKS/Flink) and cannot scale down easily; highest overhead.
- **D** is wrong — Managed Flink is for *stateful stream computation* (windows, joins, aggregations). There is no processing requirement here.
</details>

### Q3. [MCQ]
An ML team needs a streaming pipeline that computes **10-minute tumbling-window aggregations** and **stream-to-stream joins** with exactly-once semantics before writing features to a store. Which service is purpose-built for this?

- **A.** Amazon Data Firehose.
- **B.** Amazon Managed Service for Apache Flink.
- **C.** Amazon SQS.
- **D.** Amazon Kinesis Data Streams alone.

<details><summary>Answer & explanation</summary>

**Correct: B.** Windowed aggregations, stream-to-stream joins, and exactly-once **stateful** processing are exactly what Apache Flink does; Managed Service for Apache Flink runs it without you operating a cluster.

- **A** is wrong — Firehose *moves and delivers* data; it does not compute stateful windows/joins.
- **C** is wrong — SQS is a message queue, not a stream-processing engine.
- **D** is wrong — Kinesis Data Streams *transports* records; it has no built-in windowing/join compute. You would still need Flink (or a custom consumer) on top of it.
</details>

### Q4. [MCQ]
Model inference must read the latest feature values for a single customer ID in **single-digit milliseconds** during a real-time API call. Which SageMaker Feature Store configuration should you use?

- **A.** Offline store (Amazon S3), queried with Athena at inference time.
- **B.** Online store for real-time lookup.
- **C.** A Redshift materialized view.
- **D.** Batch transform reading from S3.

<details><summary>Answer & explanation</summary>

**Correct: B.** The Feature Store **online store** is a low-latency, high-availability store providing millisecond real-time feature lookups for model serving.

- **A** is wrong — the offline store lives in S3 and is queried via Athena for **training/batch**; Athena latency is seconds, not milliseconds.
- **C** is wrong — Redshift is an analytics warehouse, not a millisecond key-value lookup; also not the native Feature Store serving path.
- **D** is wrong — batch transform is for offline scoring of large datasets, not a per-request real-time lookup.
</details>

### Q5. [MRQ]
Which situations are appropriate uses of the SageMaker Feature Store **offline** store? (Choose TWO.)

- **A.** Serving features to a real-time endpoint at p99 < 20 ms.
- **B.** Building a point-in-time-correct training dataset with row-level time travel.
- **C.** Running batch scoring over the full historical feature history.
- **D.** Acting as a millisecond cache for a mobile app.
- **E.** Enforcing sub-second SLA lookups for a fraud API.

<details><summary>Answer & explanation</summary>

**Correct: B and C.** The offline store is an append-only history in S3, ideal for **training datasets with point-in-time (time-travel) correctness** and for **batch scoring over full history**.

- **A, D, E** are wrong — all three demand millisecond/sub-second real-time lookups, which is the **online** store's job, not the offline (S3/Athena) store.
</details>

### Q6. [MCQ]
A team wants a **visual, low-code** tool to profile a dataset, detect anomalies, and apply 250+ built-in transformations (normalize, deduplicate, mask) without writing Spark code, producing a reusable recipe. Which service is the best fit?

- **A.** AWS Glue DataBrew.
- **B.** Amazon EMR with a hand-written PySpark job.
- **C.** Amazon Athena.
- **D.** AWS Lambda.

<details><summary>Answer & explanation</summary>

**Correct: A.** AWS Glue DataBrew is the **visual, no-code** data-prep tool with 250+ prebuilt transformations, profiling, and reusable recipes — minimal code, minimal ops.

- **B** is wrong — EMR + PySpark is powerful but requires cluster management and code; far more overhead than a no-code visual tool.
- **C** is wrong — Athena runs SQL queries over S3; it is not a visual transformation/profiling tool with reusable recipes.
- **D** is wrong — Lambda is generic compute; you'd build everything yourself.
</details>

### Q7. [MCQ]
An engineer needs a **serverless, Spark-based** ETL service to run scheduled jobs that join several S3 datasets, with an integrated Data Catalog and crawler-driven schema discovery, at the **LEAST** operational overhead. Which service should they choose?

- **A.** Amazon EMR on EC2.
- **B.** AWS Glue (ETL jobs + Data Catalog + crawlers).
- **C.** AWS Batch.
- **D.** Amazon Redshift Spectrum.

<details><summary>Answer & explanation</summary>

**Correct: B.** AWS Glue is serverless Spark ETL with the **Glue Data Catalog** and **crawlers** for automatic schema discovery — no clusters to manage.

- **A** is wrong — EMR gives maximum control and is best for very large or highly-customized Spark/Hadoop workloads, but you manage the cluster (more overhead). Choose EMR when you need custom frameworks or the largest scale, not for a "least overhead" ask.
- **C** is wrong — AWS Batch runs containerized batch jobs generically; no native catalog/crawler/Spark ETL tooling.
- **D** is wrong — Redshift Spectrum queries S3 from Redshift; it is not a general ETL service with crawlers.
</details>

### Q8. [MCQ]
A regulated dataset contains free-text notes with names, emails, and SSNs. Before training, the team must **automatically detect and redact PII** at scale. Which AWS service is designed for this?

- **A.** Amazon Comprehend (PII detection/redaction).
- **B.** Amazon Rekognition.
- **C.** Amazon Translate.
- **D.** Amazon Kendra.

<details><summary>Answer & explanation</summary>

**Correct: A.** Amazon Comprehend has built-in **PII entity detection and redaction** for text.

- **B** is wrong — Rekognition analyzes images/video, not text PII.
- **C** is wrong — Translate performs language translation.
- **D** is wrong — Kendra is enterprise search, not PII redaction. (Note: **Macie** is the S3-object PII/sensitive-data discovery service — also valid for *finding* PII in S3, but for in-text redaction Comprehend is the tool.)
</details>

### Q9. [MRQ]
A classification training set has 98% "not-fraud" and 2% "fraud" — severe class imbalance. Which techniques directly address the imbalance so the model learns the minority class? (Choose TWO.)

- **A.** Apply SMOTE / oversample the minority class.
- **B.** Increase the learning rate.
- **C.** Undersample the majority class.
- **D.** Switch the file format from CSV to Parquet.
- **E.** Add more S3 buckets.

<details><summary>Answer & explanation</summary>

**Correct: A and C.** **Oversampling the minority** (e.g., SMOTE) and/or **undersampling the majority** rebalance the class distribution. (Class weights are another valid technique not listed here.)

- **B** is wrong — learning rate affects convergence, not class balance.
- **D** is wrong — Parquet is a storage-format optimization; it has no effect on class balance.
- **E** is wrong — more buckets is irrelevant to imbalance.
</details>

### Q10. [MCQ]
You need to convert a high-cardinality categorical column (10,000 unique product IDs) into model input for a neural network **without** creating 10,000 sparse columns. Which encoding is most appropriate?

- **A.** One-hot encoding.
- **B.** Learned embeddings (embedding layer / entity embeddings).
- **C.** Min-max scaling.
- **D.** Label encoding fed directly as an ordinal numeric feature to a linear model.

<details><summary>Answer & explanation</summary>

**Correct: B.** **Embeddings** map high-cardinality categoricals into a dense low-dimensional space, avoiding the explosion of one-hot and capturing similarity between categories.

- **A** is wrong — one-hot on 10,000 categories creates 10,000 sparse columns (the exact thing to avoid).
- **C** is wrong — min-max scaling is for continuous numeric features, not categoricals.
- **D** is wrong — feeding arbitrary integer labels as ordinals to a linear model invents a false numeric ordering between unrelated IDs.
</details>

### Q11. [MCQ]
A SageMaker training job repeatedly reads the same large dataset from S3, and startup time is dominated by copying data to the instance. Which input mode **streams** data directly from S3 without first downloading the entire dataset to local disk, reducing startup time and disk needs?

- **A.** File mode.
- **B.** Pipe mode (or FastFile mode).
- **C.** Local mode.
- **D.** Inference mode.

<details><summary>Answer & explanation</summary>

**Correct: B.** **Pipe mode** streams data from S3 into the algorithm on the fly, and **FastFile mode** exposes S3 objects as local files while streaming on demand — both avoid the full up-front download that File mode requires, cutting startup time and local disk usage.

- **A** is wrong — File mode downloads the **entire** dataset to the instance's disk before training starts (the slow behavior described).
- **C** is wrong — "Local mode" runs training on your local machine for debugging; not an S3 input-streaming mode.
- **D** is wrong — there is no training "inference mode."
</details>

### Q12. [MCQ]
Before training, the team must measure whether the dataset is **biased against a demographic group** (e.g., class imbalance across a sensitive attribute) using metrics like Class Imbalance (CI) and Difference in Positive Proportions in Labels (DPL). Which tool computes these **pre-training bias** metrics?

- **A.** SageMaker Clarify.
- **B.** SageMaker Model Monitor data-quality baseline.
- **C.** Amazon CloudWatch.
- **D.** SageMaker Debugger.

<details><summary>Answer & explanation</summary>

**Correct: A.** **SageMaker Clarify** computes pre-training bias metrics (CI, DPL, and others) on the raw dataset, plus post-training bias and explainability (SHAP).

- **B** is wrong — Model Monitor operates on **deployed endpoint** traffic, not pre-training dataset bias analysis.
- **C** is wrong — CloudWatch is metrics/logging, not bias analysis.
- **D** is wrong — Debugger inspects **training-time** tensors (gradients, weights) for issues like vanishing gradients, not dataset bias.
</details>

### Q13. [MCQ]
A team wants a **visual, notebook-free** SageMaker experience to import, join, transform, and analyze data with 300+ built-in transforms and quick bias/leakage checks, then export a processing job or pipeline step. Which tool is this?

- **A.** SageMaker Data Wrangler.
- **B.** SageMaker Ground Truth.
- **C.** SageMaker Neo.
- **D.** SageMaker JumpStart.

<details><summary>Answer & explanation</summary>

**Correct: A.** **SageMaker Data Wrangler** provides a visual flow to import, transform (300+ transforms), analyze, and detect bias/leakage, then export to a Processing job, Pipeline, or Feature Store.

- **B** is wrong — Ground Truth is for **data labeling**.
- **C** is wrong — Neo **compiles/optimizes** models for target hardware.
- **D** is wrong — JumpStart is a catalog of pretrained models and solutions.
</details>

### Q14. [MCQ]
A dataset must be **labeled** by human workers, but the team wants to minimize labeling cost by using **active learning** so that only ambiguous examples are sent to humans while confident ones are auto-labeled. Which service supports this?

- **A.** SageMaker Ground Truth (automated data labeling / active learning).
- **B.** Amazon Mechanical Turk used directly with no automation.
- **C.** AWS Glue.
- **D.** Amazon Textract.

<details><summary>Answer & explanation</summary>

**Correct: A.** **SageMaker Ground Truth** supports **automated data labeling (active learning)**: a model auto-labels high-confidence items and routes only uncertain items to human labelers, reducing cost.

- **B** is wrong — using Mechanical Turk directly gives you a human workforce but **no** active-learning automation; you'd send everything to humans.
- **C** is wrong — Glue is ETL, not labeling.
- **D** is wrong — Textract extracts text from documents; it is not a labeling workflow with active learning.
</details>

### Q15. [MCQ]
Analysts run ad-hoc **SQL** directly against Parquet files in S3 with **no servers to manage** and pay only per query. Which service should they use?

- **A.** Amazon Athena.
- **B.** Amazon EMR.
- **C.** Amazon RDS.
- **D.** AWS Glue crawler.

<details><summary>Answer & explanation</summary>

**Correct: A.** **Athena** is serverless, presto-based SQL over S3, billed per bytes scanned — ideal for ad-hoc queries with no cluster.

- **B** is wrong — EMR requires a managed cluster.
- **C** is wrong — RDS is a managed relational database, not S3 query-in-place.
- **D** is wrong — a Glue crawler discovers schema into the Data Catalog; it does not run analytical SQL.
</details>

### Q16. [MRQ]
Which storage/format choices reduce **Amazon Athena** query cost on large S3 datasets? (Choose TWO.)

- **A.** Store data in columnar Parquet/ORC.
- **B.** Store data as uncompressed JSON.
- **C.** Partition the data (e.g., by date) so queries prune scanned partitions.
- **D.** Store each row as a separate small S3 object.
- **E.** Store everything in a single 5 TB CSV file.

<details><summary>Answer & explanation</summary>

**Correct: A and C.** Athena bills by **bytes scanned**. Columnar **Parquet/ORC** lets Athena read only needed columns, and **partitioning** lets it skip irrelevant partitions — both cut bytes scanned.

- **B** is wrong — uncompressed JSON scans more bytes.
- **D** is wrong — many tiny objects cause the "small files problem," hurting performance and cost.
- **E** is wrong — a single monolithic CSV forces full scans with no pruning.
</details>

### Q17. [MCQ]
A pipeline must merge customer records from Amazon RDS, DynamoDB, and S3 into one training dataset. The team wants a **serverless** way to catalog all three sources and join them in a Spark ETL job. What is the best approach?

- **A.** Use Glue crawlers to catalog each source, then a Glue Spark ETL job to join and write to S3.
- **B.** Export everything manually to CSV and merge in Excel.
- **C.** Query each source in the SageMaker notebook one row at a time.
- **D.** Replicate all sources into a single DynamoDB table.

<details><summary>Answer & explanation</summary>

**Correct: A.** Glue **crawlers** catalog the schemas of the varied sources into the Data Catalog, and a **serverless Glue Spark job** joins and writes the unified dataset to S3 — low overhead, scalable.

- **B** is wrong — manual CSV/Excel does not scale and is error-prone.
- **C** is wrong — row-by-row querying is extremely slow and not a join strategy.
- **D** is wrong — DynamoDB is a key-value store, not a join engine; forcing all sources in is a costly anti-pattern.
</details>

### Q18. [ORDER]
Order the following steps of a typical **feature-engineering pipeline** into the correct sequence.

- **1.** Encode categorical variables and scale numeric features.
- **2.** Ingest raw data from S3 into a transformation tool (e.g., Data Wrangler).
- **3.** Clean data (handle missing values, remove duplicates, fix outliers).
- **4.** Write the engineered features to SageMaker Feature Store.

<details><summary>Answer & explanation</summary>

**Correct order: 2 → 3 → 1 → 4.**
1. **Ingest** the raw data (2).
2. **Clean** it — impute missing values, dedupe, handle outliers (3).
3. **Engineer/encode/scale** features (1).
4. **Persist** engineered features to the Feature Store for reuse in training and serving (4).

You must clean before encoding/scaling (garbage in, garbage out), and features are only stored **after** they are finalized so training and inference share the same definitions.
</details>

---

## Domain 2 — ML Model Development (26%) <a name="domain-2"></a>

### Q19. [MCQ]
A team has clean **tabular** data and needs a high-accuracy binary classifier with minimal engineering, using a managed SageMaker **built-in algorithm**. Which algorithm is the standard choice?

- **A.** XGBoost.
- **B.** BlazingText.
- **C.** Semantic Segmentation.
- **D.** DeepAR.

<details><summary>Answer & explanation</summary>

**Correct: A.** **XGBoost** is the go-to built-in for structured/**tabular** classification and regression — strong accuracy with little tuning.

- **B** is wrong — BlazingText is for **text** (word2vec / text classification).
- **C** is wrong — Semantic Segmentation is a **computer-vision** pixel-labeling algorithm.
- **D** is wrong — DeepAR is for **time-series forecasting**.
</details>

### Q20. [MCQ]
You must forecast demand for **thousands of related time series** (per-SKU sales) and want a built-in algorithm that learns a single global model across series. Which should you choose?

- **A.** Linear Learner.
- **B.** DeepAR.
- **C.** Random Cut Forest.
- **D.** K-Means.

<details><summary>Answer & explanation</summary>

**Correct: B.** **DeepAR** is a recurrent-network forecasting algorithm designed to train **one global model across many related time series**, improving accuracy over fitting each series independently.

- **A** is wrong — Linear Learner does classification/regression, not native multi-series forecasting.
- **C** is wrong — Random Cut Forest is **anomaly detection**, not forecasting.
- **D** is wrong — K-Means is unsupervised **clustering**.
</details>

### Q21. [MCQ]
You need **unsupervised anomaly detection** on a stream of numeric metrics to flag unusual spikes. Which SageMaker built-in algorithm is purpose-built for this?

- **A.** Random Cut Forest (RCF).
- **B.** XGBoost.
- **C.** Object2Vec.
- **D.** Image Classification.

<details><summary>Answer & explanation</summary>

**Correct: A.** **Random Cut Forest** is SageMaker's unsupervised **anomaly-detection** algorithm (also available in Kinesis Analytics), assigning an anomaly score to each record.

- **B** is wrong — XGBoost is supervised; you'd need labeled anomalies.
- **C** is wrong — Object2Vec learns embeddings of paired objects (recommendations/similarity), not anomaly scores.
- **D** is wrong — Image Classification is for images.
</details>

### Q22. [MCQ]
A team already has a custom **PyTorch** training script and wants to run it on SageMaker using AWS's **pre-built PyTorch container**, passing only their script and dependencies — with the **LEAST** effort. Which approach fits?

- **A.** Script mode with the managed PyTorch framework container.
- **B.** Bring Your Own Container (BYOC) — build a Docker image from scratch.
- **C.** Rewrite the model as a SageMaker built-in algorithm.
- **D.** Use SageMaker Autopilot.

<details><summary>Answer & explanation</summary>

**Correct: A.** **Script mode** lets you supply your training script (`entry_point`) plus a `requirements.txt` and run it in AWS's **maintained** framework container — no Dockerfile to write.

- **B** is wrong — BYOC (custom Docker) is only needed when your framework/dependencies aren't covered by a managed container; it is *more* effort here.
- **C** is wrong — rewriting to a built-in algorithm discards the existing PyTorch code.
- **D** is wrong — Autopilot is AutoML for tabular data; it won't run your custom PyTorch script.
</details>

### Q23. [MCQ]
When must you use **Bring Your Own Container (BYOC)** on SageMaker instead of script mode?

- **A.** When your framework version or system dependencies are not supported by any managed SageMaker container.
- **B.** Whenever you use XGBoost.
- **C.** Whenever the dataset is larger than 1 GB.
- **D.** Whenever you want hyperparameter tuning.

<details><summary>Answer & explanation</summary>

**Correct: A.** **BYOC** is the right choice when you need a **custom runtime/framework/system dependency** that no AWS-managed (built-in or framework) container provides. You package everything in your own Docker image conforming to SageMaker's container contract.

- **B, C, D** are wrong — XGBoost has a built-in and framework container, dataset size does not force BYOC, and hyperparameter tuning (AMT) works with built-in, script-mode, and BYOC alike.
</details>

### Q24. [MCQ]
You are tuning a **deep neural network** trained for many epochs and want hyperparameter search that reallocates budget to promising configs and **early-stops** poor ones, finishing up to ~3x faster than Bayesian. Which AMT strategy should you pick?

- **A.** Grid search.
- **B.** Hyperband.
- **C.** Random search.
- **D.** Bayesian optimization.

<details><summary>Answer & explanation</summary>

**Correct: B.** **Hyperband** is a multi-fidelity strategy that uses intermediate (per-epoch) results to reallocate resources to strong configs and stop weak ones early — up to ~3x faster than random/Bayesian for **iterative** algorithms like DNNs and gradient-boosted trees.

- **A** is wrong — grid search is exhaustive and does not early-stop; slowest.
- **C** is wrong — random search does not use intermediate results to reallocate.
- **D** is wrong — Bayesian is strong for non-iterative/expensive objectives but does not exploit per-epoch early stopping the way Hyperband does; slower for DNNs. (Bayesian is a fine default when the algorithm does **not** emit intermediate metrics.)
</details>

### Q25. [MCQ]
A model achieves 99% accuracy on training data but only 71% on validation data, and the gap keeps widening as you train longer. What is happening, and what is a valid fix?

- **A.** Underfitting; add more layers.
- **B.** Overfitting; apply regularization (L2/dropout), early stopping, or more training data.
- **C.** Data leakage; switch to serverless inference.
- **D.** Class imbalance; increase the learning rate.

<details><summary>Answer & explanation</summary>

**Correct: B.** A large **train-high / validation-low** gap that widens is classic **overfitting** (high variance). Fixes: **L2 regularization, dropout, early stopping, more/augmented data, or a simpler model**.

- **A** is wrong — underfitting shows **low training AND low validation** accuracy; adding capacity would worsen this overfit.
- **C** is wrong — the symptom describes overfitting, not leakage; inference type is irrelevant to training fit.
- **D** is wrong — the symptom is not imbalance, and raising the learning rate doesn't fix overfitting.
</details>

### Q26. [MCQ]
A model shows **high training error AND high validation error** (both ~65% on a task where 90%+ is expected). Which describes the problem and a correct remedy?

- **A.** Overfitting; add dropout.
- **B.** Underfitting; increase model capacity/features or train longer / reduce regularization.
- **C.** Overfitting; collect more data.
- **D.** Perfect fit; deploy as-is.

<details><summary>Answer & explanation</summary>

**Correct: B.** Both errors high = **underfitting (high bias)**. Remedies: more capacity (bigger model / more features), train longer, or **reduce** regularization.

- **A, C** are wrong — those are overfitting remedies; overfitting has **low** training error.
- **D** is wrong — 65% where 90% is expected is not a good fit.
</details>

### Q27. [MCQ]
For a **fraud-detection** model, missing an actual fraud (false negative) is far costlier than a false alarm. Which metric should you primarily optimize?

- **A.** Precision.
- **B.** Recall (sensitivity).
- **C.** Specificity only.
- **D.** RMSE.

<details><summary>Answer & explanation</summary>

**Correct: B.** **Recall** = TP / (TP + FN). Minimizing **false negatives** (missed fraud) means **maximizing recall**.

- **A** is wrong — precision minimizes false *positives*; it's the priority when false alarms are the costly error (e.g., flagging legit customers unnecessarily).
- **C** is wrong — specificity alone ignores catching the positives you care about.
- **D** is wrong — RMSE is a **regression** metric, not for classification.
</details>

### Q28. [MCQ]
A spam filter must **avoid flagging legitimate email** as spam (a false positive is very costly to the user), while still catching most spam. Which metric best captures the balance, and which do you weight higher?

- **A.** Optimize precision (weight false positives heavily), and use F1 to balance.
- **B.** Optimize recall only.
- **C.** Optimize RMSE.
- **D.** Optimize R².

<details><summary>Answer & explanation</summary>

**Correct: A.** Costly **false positives** (good mail marked spam) means favor **precision** = TP / (TP + FP); **F1** (harmonic mean of precision and recall) is the standard single metric when you must balance both.

- **B** is wrong — recall-only would tolerate flagging legit mail.
- **C, D** are wrong — RMSE and R² are **regression** metrics, not classification.
</details>

### Q29. [MCQ]
You want a **threshold-independent** measure of a binary classifier's ability to rank positives above negatives across all thresholds, useful when the decision threshold isn't fixed. Which metric?

- **A.** Accuracy.
- **B.** AUC (area under the ROC curve).
- **C.** RMSE.
- **D.** Log-loss at threshold 0.5.

<details><summary>Answer & explanation</summary>

**Correct: B.** **AUC-ROC** measures ranking quality **across all thresholds** and is robust to the specific cutoff and to moderate class imbalance.

- **A** is wrong — accuracy is computed at a **single** threshold and is misleading under imbalance.
- **C** is wrong — RMSE is for regression.
- **D** is wrong — log-loss at a fixed threshold is threshold-dependent.
</details>

### Q30. [MCQ]
A regression model predicts house prices; you want a metric in the **same units** as the target that **penalizes large errors** more heavily. Which should you report?

- **A.** RMSE.
- **B.** Accuracy.
- **C.** F1.
- **D.** AUC.

<details><summary>Answer & explanation</summary>

**Correct: A.** **RMSE** is in the target's units (dollars) and, because it squares errors before averaging, **penalizes large errors** more than MAE.

- **B, C, D** are wrong — accuracy, F1, and AUC are **classification** metrics, not applicable to continuous price prediction.
</details>

### Q31. [MRQ]
Which are valid ways to speed up training of a large deep-learning model on SageMaker? (Choose TWO.)

- **A.** Use SageMaker distributed training (data-parallel or model-parallel) across multiple GPUs/instances.
- **B.** Switch the endpoint to serverless inference.
- **C.** Use GPU/accelerated instances (e.g., ml.p or ml.g families) instead of CPU.
- **D.** Convert the model to ONNX after training.
- **E.** Enable S3 Object Lock.

<details><summary>Answer & explanation</summary>

**Correct: A and C.** **Distributed training** (SageMaker Distributed Data Parallel / Model Parallel) scales across GPUs/instances, and **GPU instances** dramatically accelerate deep-learning training vs CPU.

- **B** is wrong — serverless inference is a *deployment* option; it does not affect training speed.
- **D** is wrong — ONNX conversion targets **inference** optimization, not training speed.
- **E** is wrong — S3 Object Lock is data retention/compliance, unrelated to training speed.
</details>

### Q32. [MCQ]
A team wants SageMaker to **automatically** try multiple algorithms and preprocessing steps on their tabular dataset, produce candidate models with explainability, and let them pick the best — with minimal manual ML. Which service?

- **A.** SageMaker Autopilot.
- **B.** SageMaker Neo.
- **C.** SageMaker Clarify.
- **D.** SageMaker Debugger.

<details><summary>Answer & explanation</summary>

**Correct: A.** **SageMaker Autopilot** is AutoML for tabular data: it explores feature engineering + algorithms + hyperparameters, ranks candidates, and provides notebooks and explainability.

- **B** is wrong — Neo compiles/optimizes trained models for hardware.
- **C** is wrong — Clarify does bias/explainability, not full AutoML training.
- **D** is wrong — Debugger inspects training tensors for issues.
</details>

### Q33. [MCQ]
During training, loss becomes **NaN** and gradients explode. You want automatic rules that detect vanishing/exploding gradients and other training problems in near real time. Which tool do you enable?

- **A.** SageMaker Debugger.
- **B.** SageMaker Model Monitor.
- **C.** Amazon Macie.
- **D.** AWS Config.

<details><summary>Answer & explanation</summary>

**Correct: A.** **SageMaker Debugger** captures training tensors and runs built-in rules (e.g., `exploding_tensor`, `vanishing_gradient`, `loss_not_decreasing`) to detect problems during training.

- **B** is wrong — Model Monitor watches **deployed endpoints**, not training internals.
- **C** is wrong — Macie discovers sensitive data in S3.
- **D** is wrong — AWS Config tracks resource configuration compliance.
</details>

### Q34. [MCQ]
A team wants to run the **same training script** on 8 GPUs where each GPU processes a different shard of the same batch, keeping the full model on every GPU. Which distributed strategy is this?

- **A.** Data parallelism.
- **B.** Model parallelism.
- **C.** Pipeline parallelism only.
- **D.** No parallelism.

<details><summary>Answer & explanation</summary>

**Correct: A.** **Data parallelism** replicates the **full model** on each GPU and splits the **data** (batch shards) across them, syncing gradients — the right fit when the model fits in a single GPU's memory.

- **B** is wrong — model parallelism splits the **model** across GPUs; used when the model is **too large** to fit on one GPU.
- **C** is wrong — pipeline parallelism is a form of model parallelism, not the described data-sharding pattern.
- **D** is wrong — the scenario explicitly uses 8 GPUs in parallel.
</details>

### Q35. [MCQ]
You want to track experiments — hyperparameters, metrics, and artifacts — across many training runs so you can compare and reproduce them. Which capability should you use?

- **A.** SageMaker Experiments.
- **B.** SageMaker Ground Truth.
- **C.** Amazon QuickSight.
- **D.** AWS CloudTrail.

<details><summary>Answer & explanation</summary>

**Correct: A.** **SageMaker Experiments** logs and organizes runs (parameters, metrics, artifacts) for comparison and reproducibility.

- **B** is wrong — Ground Truth is labeling.
- **C** is wrong — QuickSight is BI dashboards.
- **D** is wrong — CloudTrail logs **API activity** for audit, not ML experiment metrics.
</details>

---

## Domain 3 — Deployment & Orchestration (22%) <a name="domain-3"></a>

### Q36. [MCQ]
An application needs **synchronous, low-latency (single-digit ms)** predictions with steady, predictable traffic and payloads under 25 MB. Which SageMaker inference option fits best?

- **A.** Real-time endpoint.
- **B.** Serverless inference.
- **C.** Asynchronous inference.
- **D.** Batch transform.

<details><summary>Answer & explanation</summary>

**Correct: A.** A **real-time endpoint** gives persistent, low-latency synchronous inference for **steady traffic**, supporting payloads up to **25 MB** and up to ~60 s per request.

- **B** is wrong — serverless is best for **intermittent/unpredictable** traffic and can incur cold starts; less ideal for steady sub-ms-critical traffic (and caps payload at 4 MB).
- **C** is wrong — async is for **large payloads / long processing**, queued (not synchronous).
- **D** is wrong — batch transform is offline scoring, not per-request real time.
</details>

### Q37. [MCQ]
A model receives **sporadic, spiky** traffic with long idle gaps. The team wants to pay only for compute used and avoid managing/scaling instances, and can tolerate occasional cold starts. Payload is < 4 MB. Which option is **MOST cost-effective**?

- **A.** Provisioned real-time endpoint running 24/7.
- **B.** Serverless inference.
- **C.** Asynchronous inference with a large instance always on.
- **D.** A fleet of EC2 instances behind an ALB.

<details><summary>Answer & explanation</summary>

**Correct: B.** **Serverless inference** auto-provisions compute per request and charges only for usage (no idle cost) — ideal for **intermittent/unpredictable** traffic with small (≤ 4 MB) payloads. Cold starts are acceptable per the scenario.

- **A** is wrong — a 24/7 real-time endpoint pays for idle time during the long gaps.
- **C** is wrong — an always-on instance defeats the pay-per-use goal.
- **D** is wrong — self-managed EC2 fleet is high operational overhead and still pays for idle capacity.
</details>

### Q38. [MCQ]
Inference requests carry **500 MB** payloads (large documents) and take several minutes to process; clients can poll for results and don't need a synchronous response. Which endpoint type is designed for this?

- **A.** Real-time endpoint.
- **B.** Serverless inference.
- **C.** Asynchronous inference.
- **D.** Multi-model real-time endpoint.

<details><summary>Answer & explanation</summary>

**Correct: C.** **Asynchronous inference** queues requests, supports payloads up to **1 GB** and processing up to ~60 minutes, returns results to S3, and can **scale to zero** when idle — perfect for large payloads / long jobs.

- **A** is wrong — real-time caps payload at **25 MB** and ~60 s.
- **B** is wrong — serverless caps payload at **4 MB**.
- **D** is wrong — multi-model endpoints are about hosting many small models cheaply, not large-payload/long-processing async work.
</details>

### Q39. [MCQ]
A team must score a **static 2 TB dataset once per night**, writing predictions back to S3. There is no need for a persistent endpoint. Which option is **MOST cost-effective**?

- **A.** Batch transform.
- **B.** A real-time endpoint invoked row by row.
- **C.** Serverless inference invoked per record.
- **D.** Asynchronous inference for each record.

<details><summary>Answer & explanation</summary>

**Correct: A.** **Batch transform** spins up compute, scores the whole S3 dataset, writes results to S3, and **tears down** — no persistent endpoint cost. Ideal for scheduled bulk scoring.

- **B, C, D** are wrong — invoking any per-request endpoint millions of times is slower and far costlier than a single batch job; a persistent endpoint also pays for idle time.
</details>

### Q40. [MCQ]
You must host **100 small, similar models** (one per customer) cost-effectively behind a single endpoint, loading each model into memory only when invoked. Which feature fits best?

- **A.** SageMaker Multi-Model Endpoint (MME).
- **B.** 100 separate real-time endpoints.
- **C.** Serverless inference with 100 configs.
- **D.** Batch transform.

<details><summary>Answer & explanation</summary>

**Correct: A.** A **Multi-Model Endpoint** hosts many models on **shared** infrastructure, dynamically loading a model from S3 on demand — far cheaper than one endpoint per model.

- **B** is wrong — 100 dedicated endpoints multiply cost and ops.
- **C** is wrong — serverless doesn't natively share one endpoint across 100 models the way MME does.
- **D** is wrong — batch transform is offline scoring, not on-demand per-customer serving.
</details>

### Q41. [MCQ]
You are updating a production model and want to shift traffic to the new version in **two steps** — a small validation slice first, then the remainder — with **automatic rollback** if CloudWatch alarms trip. Which deployment guardrail mode is this?

- **A.** All-at-once traffic shifting.
- **B.** Canary traffic shifting.
- **C.** Linear traffic shifting.
- **D.** Shadow (production variant mirroring) only.

<details><summary>Answer & explanation</summary>

**Correct: B.** **Canary** blue/green shifting moves traffic in **two steps**: a small canary portion is validated on the green fleet during a baking period, then the rest shifts; alarms trigger auto-rollback.

- **A** is wrong — all-at-once shifts 100% in a single step (fastest, riskiest).
- **C** is wrong — linear shifts in **multiple equal steps** you configure, not two.
- **D** is wrong — shadow testing mirrors traffic to a variant for evaluation without serving it to users; it's not a two-step traffic-shift with rollback.
</details>

### Q42. [MCQ]
For a high-risk model update, the team wants to shift traffic **gradually in many equal increments** (e.g., 10% every 5 minutes) to minimize blast radius, with alarms and auto-rollback. Which mode?

- **A.** Linear traffic shifting.
- **B.** Canary traffic shifting.
- **C.** All-at-once.
- **D.** Cold deployment.

<details><summary>Answer & explanation</summary>

**Correct: A.** **Linear** shifting moves traffic in the **number of equal steps you specify**, minimizing disruption — the "many small increments" pattern.

- **B** is wrong — canary is specifically **two** steps.
- **C** is wrong — all-at-once has no gradual increments.
- **D** is wrong — "cold deployment" is not a SageMaker guardrail mode.
</details>

### Q43. [MCQ]
Which **auto-scaling target metric** is the recommended default for scaling a SageMaker real-time endpoint based on per-instance load?

- **A.** `SageMakerVariantInvocationsPerInstance`.
- **B.** `CPUUtilization` of the notebook.
- **C.** S3 request count.
- **D.** DynamoDB consumed capacity.

<details><summary>Answer & explanation</summary>

**Correct: A.** The predefined target-tracking metric **`SageMakerVariantInvocationsPerInstance`** scales the number of instances based on invocations per instance — the standard endpoint auto-scaling signal.

- **B** is wrong — that's a notebook metric, unrelated to endpoint throughput.
- **C, D** are wrong — S3/DynamoDB metrics don't reflect endpoint invocation load.
</details>

### Q44. [MCQ]
A team needs to orchestrate an **ML-specific** workflow (process → train → evaluate → conditional register → deploy) natively integrated with the SageMaker Model Registry and lineage, with the **LEAST** extra infrastructure. Which orchestrator is the best fit?

- **A.** SageMaker Pipelines.
- **B.** Amazon Managed Workflows for Apache Airflow (MWAA).
- **C.** AWS Step Functions with hand-written Lambdas for every step.
- **D.** Cron on an EC2 instance.

<details><summary>Answer & explanation</summary>

**Correct: A.** **SageMaker Pipelines** is the purpose-built, serverless ML CI/CD orchestrator with native steps (Processing, Training, Tuning, Model, Conditional, Register), Model Registry integration, and automatic **lineage** — least extra infra for pure-SageMaker ML flows.

- **B** is wrong — MWAA (managed Airflow) is excellent for **broad, cross-service** DAGs or teams standardized on Airflow, but adds a managed Airflow environment to run and is not ML-native.
- **C** is wrong — Step Functions works but you build every step yourself; more effort and no ML-native registry/lineage.
- **D** is wrong — cron on EC2 is fragile, unmanaged, and has no lineage or retries.
</details>

### Q45. [ORDER]
Put the stages of a **SageMaker Pipelines** MLOps flow in the correct execution order.

- **1.** Register the approved model in the Model Registry.
- **2.** Processing step: transform raw data.
- **3.** Training step: train the model.
- **4.** Condition step: check evaluation metric threshold.
- **5.** Evaluation (Processing) step: compute test metrics.

<details><summary>Answer & explanation</summary>

**Correct order: 2 → 3 → 5 → 4 → 1.**
Process the data (2) → train the model (3) → evaluate on the test set (5) → **conditionally** check whether the metric clears the threshold (4) → if it passes, register the model in the Model Registry for approval/deployment (1). The condition gates registration so only good models advance.
</details>

### Q46. [MRQ]
Which AWS **developer/CI-CD** services are commonly combined to automate building, testing, and deploying an ML application's code and infrastructure? (Choose TWO.)

- **A.** AWS CodePipeline (orchestrates the release pipeline).
- **B.** AWS CodeBuild (builds and tests artifacts/containers).
- **C.** Amazon Rekognition.
- **D.** Amazon Polly.
- **E.** Amazon Lex.

<details><summary>Answer & explanation</summary>

**Correct: A and B.** **CodePipeline** orchestrates the CI/CD stages (source → build → deploy), and **CodeBuild** compiles/tests and builds container images — the standard automation pair (often with CodeDeploy for deployment).

- **C, D, E** are wrong — Rekognition (vision), Polly (text-to-speech), and Lex (chatbots) are AI **application** services, not CI/CD tooling.
</details>

### Q47. [MCQ]
An ML container image built by CodeBuild must be stored and versioned for SageMaker to pull at deploy time. Which service holds the image?

- **A.** Amazon ECR (Elastic Container Registry).
- **B.** Amazon S3 Glacier.
- **C.** AWS CodeCommit.
- **D.** Amazon EFS.

<details><summary>Answer & explanation</summary>

**Correct: A.** **Amazon ECR** is the managed Docker/OCI **image registry**; SageMaker pulls training/inference images from ECR.

- **B** is wrong — Glacier is cold object archival, not a container registry.
- **C** is wrong — CodeCommit is a **Git** source repo, not an image registry.
- **D** is wrong — EFS is a file system, not an image registry.
</details>

### Q48. [MCQ]
You want to package and deploy **infrastructure as code** (endpoints, roles, pipelines) repeatably across dev/prod accounts. Which service is designed for declarative IaC provisioning?

- **A.** AWS CloudFormation (or CDK).
- **B.** Amazon Athena.
- **C.** AWS Glue.
- **D.** Amazon SNS.

<details><summary>Answer & explanation</summary>

**Correct: A.** **CloudFormation** (and the **CDK**, which synthesizes CloudFormation) provisions AWS resources declaratively and repeatably across environments.

- **B, C, D** are wrong — Athena (SQL), Glue (ETL), and SNS (pub/sub messaging) are not IaC provisioning tools.
</details>

### Q49. [MCQ]
An endpoint must scale to handle a predictable **9 a.m. traffic surge** every weekday. Which scaling approach adds capacity **ahead** of the known spike?

- **A.** Scheduled scaling.
- **B.** Only reactive target-tracking scaling.
- **C.** Manual instance changes each morning.
- **D.** No scaling — over-provision permanently.

<details><summary>Answer & explanation</summary>

**Correct: A.** **Scheduled scaling** adds capacity at known times (e.g., before 9 a.m.), avoiding the lag of reacting after load rises.

- **B** is wrong — reactive-only scaling lags the surge and can drop early requests; best combined with scheduled scaling for **known** patterns.
- **C** is wrong — manual changes are error-prone and not automated.
- **D** is wrong — permanent over-provisioning wastes money.
</details>

---

## Domain 4 — Monitoring, Maintenance & Security (24%) <a name="domain-4"></a>

### Q50. [MCQ]
Weeks after deployment, the **statistical distribution of incoming features** has shifted from the training baseline (e.g., a feature's mean moved), though the target relationship is unknown. Which SageMaker Model Monitor type detects this **data drift**?

- **A.** Data quality monitoring.
- **B.** Model quality monitoring.
- **C.** Bias drift monitoring.
- **D.** Feature attribution drift monitoring.

<details><summary>Answer & explanation</summary>

**Correct: A.** **Data quality monitoring** compares live input data statistics against a baseline and flags **data drift** (distribution changes, missing values, type violations).

- **B** is wrong — model quality compares predictions to **ground-truth labels** (accuracy/RMSE), which requires labels you don't have here.
- **C** is wrong — bias drift tracks changes in **bias metrics** across groups.
- **D** is wrong — feature attribution drift tracks changes in **feature importance** (via Clarify/SHAP).
</details>

### Q51. [MCQ]
You have delayed ground-truth labels arriving for past predictions and want to detect that the model's **accuracy is degrading** over time (concept drift). Which Model Monitor type do you use?

- **A.** Model quality monitoring.
- **B.** Data quality monitoring.
- **C.** Feature attribution drift.
- **D.** Bias drift.

<details><summary>Answer & explanation</summary>

**Correct: A.** **Model quality monitoring** merges predictions with **ingested ground-truth labels** to compute live metrics (accuracy, precision, RMSE, etc.) and detect performance degradation — the signature of **concept drift**.

- **B** is wrong — data quality watches inputs, not label-based accuracy.
- **C** is wrong — feature attribution drift watches importances, not accuracy.
- **D** is wrong — bias drift watches fairness metrics, not overall accuracy.
</details>

### Q52. [MRQ]
Which are the **four** monitoring types offered by SageMaker Model Monitor (with Clarify)? (Choose the two that are Clarify-based from the list below.)

- **A.** Bias drift.
- **B.** Feature attribution drift.
- **C.** Network latency monitoring.
- **D.** Disk-space monitoring.
- **E.** Billing anomaly monitoring.

<details><summary>Answer & explanation</summary>

**Correct: A and B.** The four Model Monitor types are **Data quality, Model quality, Bias drift, and Feature attribution drift**. The last two — **bias drift** and **feature attribution drift** — are powered by **SageMaker Clarify**.

- **C, D, E** are wrong — latency, disk, and billing anomalies are CloudWatch/Cost tooling, not Model Monitor types.
</details>

### Q53. [MCQ]
The difference between **data drift** and **concept drift** is best described as:

- **A.** Data drift = input feature distribution changes; concept drift = the relationship between inputs and the target changes.
- **B.** They are the same thing.
- **C.** Data drift only affects images; concept drift only affects text.
- **D.** Concept drift = the S3 bucket region changed.

<details><summary>Answer & explanation</summary>

**Correct: A.** **Data (covariate) drift** is a change in the **input distribution** P(X); **concept drift** is a change in **P(Y|X)** — the input→output relationship — which degrades accuracy even if inputs look similar.

- **B, C, D** are wrong — they conflate or trivialize the definitions; drift types are not tied to a specific modality or to bucket region.
</details>

### Q54. [MCQ]
A security review requires that a SageMaker training job have **no route to the public internet** and reach S3 only over the AWS network. Which combination achieves this?

- **A.** Run the job in a private VPC subnet and use an S3 **VPC endpoint** (gateway endpoint).
- **B.** Attach an Internet Gateway to the subnet.
- **C.** Store credentials in the training script.
- **D.** Make the S3 bucket public.

<details><summary>Answer & explanation</summary>

**Correct: A.** Running in **VPC-only mode** (private subnets, no NAT/IGW) plus an **S3 VPC gateway endpoint** keeps traffic on the AWS network with no public internet exposure. (Interface/VPC endpoints for the SageMaker APIs complete the isolation.)

- **B** is wrong — an Internet Gateway does the opposite: it grants public internet access.
- **C** is wrong — embedding credentials is an anti-pattern; use IAM roles.
- **D** is wrong — a public bucket is a severe security violation.
</details>

### Q55. [MCQ]
Following **least privilege**, which IAM policy design is correct for a SageMaker execution role that only needs to read one training bucket and write to one output bucket?

- **A.** Grant `s3:GetObject` on the input bucket ARN and `s3:PutObject` on the output bucket ARN only.
- **B.** Attach `AmazonS3FullAccess`.
- **C.** Attach `AdministratorAccess`.
- **D.** Use the account root credentials.

<details><summary>Answer & explanation</summary>

**Correct: A.** **Least privilege** grants only the specific actions (`GetObject`/`PutObject`) scoped to the specific bucket/prefix ARNs the job needs.

- **B** is wrong — `AmazonS3FullAccess` grants far more than needed across **all** buckets.
- **C** is wrong — admin access is the opposite of least privilege.
- **D** is wrong — root credentials should never be used for workloads.
</details>

### Q56. [MCQ]
Data at rest in the training/output S3 buckets must be encrypted with a **customer-managed key** you can audit and rotate. Which service provides the keys?

- **A.** AWS KMS (customer-managed key).
- **B.** AWS Shield.
- **C.** Amazon GuardDuty.
- **D.** AWS WAF.

<details><summary>Answer & explanation</summary>

**Correct: A.** **AWS KMS** provides **customer-managed keys (CMKs)** with rotation and CloudTrail-audited usage for S3 SSE-KMS (and EBS/endpoint volume) encryption.

- **B** is wrong — Shield is DDoS protection.
- **C** is wrong — GuardDuty is threat detection.
- **D** is wrong — WAF filters web traffic.
</details>

### Q57. [MCQ]
A **training workload runs nightly and can tolerate interruption/restart**. The team wants the **lowest compute cost**. Which purchasing option should they use?

- **A.** Managed Spot Training (Spot Instances).
- **B.** On-Demand only.
- **C.** A 3-year Reserved Instance for the training instance.
- **D.** Dedicated Hosts.

<details><summary>Answer & explanation</summary>

**Correct: A.** **Managed Spot Training** uses Spot capacity for up to ~90% savings and, with checkpointing, tolerates interruptions — ideal for **fault-tolerant, restartable** training.

- **B** is wrong — On-Demand is the most expensive per-hour option with no discount.
- **C** is wrong — Reserved/Savings Plans suit **steady, long-running** usage (e.g., a 24/7 endpoint), not intermittent interruptible jobs.
- **D** is wrong — Dedicated Hosts are for licensing/compliance isolation, not cost savings here.
</details>

### Q58. [MCQ]
An inference endpoint runs **24/7 at steady utilization for the next year**. Which option **MOST** reduces its cost while keeping it always-on?

- **A.** A SageMaker Savings Plan (or Reserved capacity) commitment.
- **B.** Managed Spot (interruptible) for the endpoint.
- **C.** On-Demand with no commitment.
- **D.** Move it to batch transform.

<details><summary>Answer & explanation</summary>

**Correct: A.** For **steady, predictable, always-on** compute, a **SageMaker Savings Plan** (1- or 3-year commitment) gives the deepest discount vs On-Demand.

- **B** is wrong — Spot instances can be interrupted, which is unacceptable for an always-on **real-time** endpoint (Spot is for training, not persistent endpoints).
- **C** is wrong — On-Demand has no discount.
- **D** is wrong — batch transform can't serve real-time traffic.
</details>

### Q59. [MCQ]
You must capture **every request/response** hitting a real-time endpoint to build a Model Monitor baseline and for auditing. Which feature enables this?

- **A.** SageMaker Endpoint **Data Capture**.
- **B.** VPC Flow Logs.
- **C.** S3 Access Logs.
- **D.** AWS Config rules.

<details><summary>Answer & explanation</summary>

**Correct: A.** **Data Capture** on the endpoint logs inbound requests and outbound responses to S3 — the data source Model Monitor uses for baselining and drift analysis.

- **B** is wrong — VPC Flow Logs record network metadata, not payloads.
- **C** is wrong — S3 access logs record bucket access, not endpoint inference payloads.
- **D** is wrong — Config tracks resource configuration state, not inference traffic.
</details>

### Q60. [MCQ]
To trigger an **automated alert and retraining pipeline** when Model Monitor detects drift, which pattern is standard?

- **A.** Model Monitor emits CloudWatch metrics → CloudWatch alarm → EventBridge rule → triggers SageMaker Pipeline / Lambda.
- **B.** Poll the endpoint manually every hour.
- **C.** Email the whole company on every inference.
- **D.** Delete the endpoint whenever latency rises.

<details><summary>Answer & explanation</summary>

**Correct: A.** Model Monitor publishes results/metrics to **CloudWatch**; an **alarm** on a drift metric fires an **EventBridge** rule that invokes a retraining **SageMaker Pipeline** or Lambda — a fully automated MLOps loop.

- **B** is wrong — manual polling isn't automation.
- **C** is wrong — alerting on every inference is noise, not drift-based.
- **D** is wrong — deleting the endpoint on latency is not drift remediation.
</details>

### Q61. [MRQ]
Which practices support **secure, auditable** SageMaker operations? (Choose TWO.)

- **A.** Use IAM roles with least-privilege policies scoped to specific resources.
- **B.** Enable AWS CloudTrail to log SageMaker API calls for audit.
- **C.** Hardcode long-lived access keys in notebooks.
- **D.** Disable all encryption to simplify debugging.
- **E.** Share one admin role across all data scientists.

<details><summary>Answer & explanation</summary>

**Correct: A and B.** **Least-privilege IAM roles** and **CloudTrail** API logging are foundational for secure, auditable operations.

- **C** is wrong — hardcoded keys are a major security risk; use roles.
- **D** is wrong — disabling encryption violates data-protection requirements.
- **E** is wrong — a shared admin role destroys least privilege and accountability.
</details>

### Q62. [MCQ]
Explanations for **individual predictions** are required for a loan model (regulators want feature contributions per decision). Which SageMaker capability provides SHAP-based per-prediction explainability?

- **A.** SageMaker Clarify.
- **B.** SageMaker Neo.
- **C.** Amazon CloudWatch.
- **D.** AWS Config.

<details><summary>Answer & explanation</summary>

**Correct: A.** **SageMaker Clarify** produces **SHAP-based feature-attribution** explanations for both global and **per-prediction** explainability, and it also detects bias.

- **B** is wrong — Neo optimizes models for hardware, no explainability.
- **C, D** are wrong — CloudWatch and Config are monitoring/compliance, not model explainability.
</details>

### Q63. [MCQ]
An endpoint's p99 **latency spikes** and error rate rises intermittently. Which service should you use first to view endpoint metrics (invocations, latency, errors) and set alarms?

- **A.** Amazon CloudWatch (metrics, logs, alarms).
- **B.** Amazon Macie.
- **C.** AWS Trusted Advisor.
- **D.** Amazon Inspector.

<details><summary>Answer & explanation</summary>

**Correct: A.** **CloudWatch** collects SageMaker endpoint metrics (`ModelLatency`, `Invocation4XXErrors`, etc.) and logs, and lets you set alarms — the first stop for operational troubleshooting.

- **B** is wrong — Macie finds sensitive data in S3.
- **C** is wrong — Trusted Advisor gives cost/security best-practice checks, not real-time endpoint metrics.
- **D** is wrong — Inspector scans for software vulnerabilities.
</details>

### Q64. [MCQ]
A team wants to **retrain automatically** only when drift is detected rather than on a fixed schedule, to save cost. Which is the more cost-effective trigger?

- **A.** Event-driven retraining triggered by a Model Monitor drift alarm.
- **B.** Retrain hourly regardless of drift.
- **C.** Retrain the model on every single inference request.
- **D.** Never retrain.

<details><summary>Answer & explanation</summary>

**Correct: A.** **Event-driven (drift-triggered)** retraining runs the expensive training job **only when needed**, saving compute versus fixed schedules while keeping the model fresh.

- **B** is wrong — hourly retraining wastes compute when no drift has occurred.
- **C** is wrong — per-request retraining is absurdly expensive and infeasible.
- **D** is wrong — never retraining lets the model decay as drift accumulates.
</details>

---

## Mixed mini-exam <a name="mixed"></a>

*These blend multiple domains, like the real exam. Read every qualifier carefully.*

### Q65. [CASE]
A fintech runs fraud scoring. Traffic is **bursty and unpredictable**, payloads are small (< 1 MB), and end-user latency matters but occasional cold starts are acceptable. Features must be read in **milliseconds** at request time, and regulators require **per-decision explanations**. Which combination best satisfies all requirements at the **LEAST** cost/overhead?

- **A.** Serverless inference endpoint + Feature Store **online** store + SageMaker Clarify for explanations.
- **B.** 24/7 real-time endpoint + Feature Store offline store + CloudWatch.
- **C.** Batch transform nightly + Athena feature lookups + Macie.
- **D.** Asynchronous inference + DynamoDB features + Neo.

<details><summary>Answer & explanation</summary>

**Correct: A.** Bursty/unpredictable small-payload traffic with acceptable cold starts → **serverless inference** (pay-per-use, no idle cost). Millisecond feature reads → Feature Store **online** store. Per-decision explanations → **Clarify** (SHAP).

- **B** is wrong — a 24/7 endpoint wastes money on idle for bursty traffic; the **offline** store (Athena) is seconds-latency, failing the millisecond requirement; CloudWatch doesn't explain decisions.
- **C** is wrong — batch transform can't serve real-time; Athena is too slow for per-request lookups; Macie is data discovery, not explainability.
- **D** is wrong — async adds queueing latency unsuited to interactive scoring, and Neo optimizes compilation, not explanations.
</details>

### Q66. [CASE]
A media company must transcode-and-score **1 GB video files**, each taking ~15 minutes, submitted a few times per hour. Clients poll for results. Cost should be minimized during idle periods. Which inference option is correct?

- **A.** Asynchronous inference (scales to zero when idle; supports up to 1 GB / long processing).
- **B.** Real-time endpoint (25 MB / 60 s limits).
- **C.** Serverless inference (4 MB limit).
- **D.** A permanently running GPU fleet.

<details><summary>Answer & explanation</summary>

**Correct: A.** **Asynchronous inference** supports **up to 1 GB** payloads and **up to ~60 min** processing, queues requests, and **scales to zero** during idle gaps — matching the large-file, long-job, sporadic-traffic, cost-sensitive profile.

- **B** is wrong — real-time caps at **25 MB / ~60 s** (fails both).
- **C** is wrong — serverless caps at **4 MB** (fails payload).
- **D** is wrong — an always-on fleet pays for idle, violating the cost goal.
</details>

### Q67. [CASE]
A team's model performs great in offline tests but its **live accuracy silently declines** over two months. They have delayed ground-truth labels available. They want automated detection and an automated retrain. Order the correct MLOps response.

- **1.** EventBridge rule invokes a SageMaker Pipeline to retrain and register a new model.
- **2.** Enable endpoint **Data Capture** and ingest ground-truth labels.
- **3.** Configure **Model quality** monitoring against a baseline; it emits CloudWatch metrics.
- **4.** A CloudWatch alarm fires when accuracy drops below threshold.

<details><summary>Answer & explanation</summary>

**Correct order: 2 → 3 → 4 → 1.**
Capture requests/responses and ingest labels (2) → run **Model quality** monitoring that publishes accuracy to CloudWatch (3) → an **alarm** trips when accuracy falls below threshold (4) → **EventBridge** triggers the retraining **Pipeline** (1). This is the standard drift-detection-to-retrain loop; model quality (not data quality) is used because labels are available and the concern is accuracy decay (concept drift).
</details>

### Q68. [CASE]
A company ingests IoT sensor telemetry that needs **stateful 5-minute windowed anomaly detection** in real time, then must deliver both raw events and flagged anomalies to S3 for later training. Which architecture is best?

- **A.** Kinesis Data Streams → Managed Service for Apache Flink (windowed anomaly logic) → Amazon Data Firehose → S3.
- **B.** Amazon Data Firehose only → S3.
- **C.** SQS → Lambda → RDS.
- **D.** MSK with no consumers.

<details><summary>Answer & explanation</summary>

**Correct: A.** **Kinesis Data Streams** ingests ordered, replayable telemetry; **Managed Service for Apache Flink** performs the **stateful windowed** anomaly computation; **Firehose** delivers raw + flagged records to **S3** for training — each service used for its strength.

- **B** is wrong — Firehose can't do stateful windowing.
- **C** is wrong — SQS/Lambda/RDS is not a streaming windowed-analytics stack and doesn't scale for high-throughput telemetry.
- **D** is wrong — MSK "with no consumers" does nothing; you'd still have to build all processing.
</details>

### Q69. [CASE]
A data scientist must convert 20 TB of CSV logs to a query- and training-efficient format, then let analysts run ad-hoc SQL, and finally train an XGBoost model with **minimal data-copy startup time**. Which combination is best?

- **A.** Glue job to Parquet → Athena for SQL → SageMaker XGBoost with Pipe/FastFile input mode.
- **B.** Keep CSV → EMR for every query → File mode training.
- **C.** Convert to JSON → RDS → Local mode training.
- **D.** Firehose to S3 → QuickSight only → BYOC.

<details><summary>Answer & explanation</summary>

**Correct: A.** Convert to **Parquet** (columnar, cheaper scans) via **Glue**; **Athena** serves serverless ad-hoc SQL; **XGBoost** with **Pipe/FastFile** input mode streams from S3, minimizing startup copy time.

- **B** is wrong — CSV + EMR-per-query is costly/high-overhead, and File mode copies the whole dataset first (the thing to avoid).
- **C** is wrong — JSON is verbose, RDS is not a 20 TB analytics store, and Local mode is for debugging.
- **D** is wrong — Firehose is ingestion (data is already in S3), QuickSight is BI not SQL exploration, and BYOC is unnecessary for XGBoost.
</details>

### Q70. [MCQ]
A model must be updated with **zero downtime** and instant rollback if the new version misbehaves; the team accepts running two fleets briefly. Which deployment strategy is most appropriate?

- **A.** Blue/green deployment with automatic rollback on CloudWatch alarms.
- **B.** Stop the endpoint, replace the model, restart.
- **C.** Edit the model files in place on the running instance.
- **D.** Delete and recreate the endpoint.

<details><summary>Answer & explanation</summary>

**Correct: A.** **Blue/green** stands up a new (green) fleet alongside the old (blue), shifts traffic with a baking period, and **auto-rolls back** to blue if alarms trip — zero downtime, fast rollback.

- **B, D** are wrong — stopping/deleting the endpoint causes **downtime**.
- **C** is wrong — editing files in place on a live instance is unsafe, unversioned, and unsupported.
</details>

### Q71. [CASE]
An image-classification model must be deployed to **edge devices** (cameras) with limited compute, requiring the model to be **compiled/optimized** for the target hardware to reduce latency and size. Which service handles this?

- **A.** SageMaker Neo.
- **B.** SageMaker Clarify.
- **C.** SageMaker Data Wrangler.
- **D.** Amazon Comprehend.

<details><summary>Answer & explanation</summary>

**Correct: A.** **SageMaker Neo** compiles and optimizes trained models to run efficiently on specific **edge/target hardware** (often paired with AWS IoT Greengrass for deployment).

- **B** is wrong — Clarify is bias/explainability.
- **C** is wrong — Data Wrangler is data prep.
- **D** is wrong — Comprehend is an NLP service, unrelated to edge model compilation.
</details>

### Q72. [MCQ]
A classifier for a **rare disease** (0.5% prevalence) reports **99.5% accuracy** by predicting "healthy" for everyone. Which metrics should the team use instead to reflect real performance? (Choose the best single answer.)

- **A.** Precision, recall, and F1 (and/or AUC-PR), not raw accuracy.
- **B.** Accuracy alone is fine.
- **C.** RMSE and R².
- **D.** Bytes scanned per query.

<details><summary>Answer & explanation</summary>

**Correct: A.** Under severe class imbalance, **accuracy is misleading** (a trivial "all healthy" model scores 99.5%). **Precision, recall, F1**, and **AUC-PR** reveal how well the minority (disease) class is actually caught.

- **B** is wrong — that's exactly the trap the question exposes.
- **C** is wrong — RMSE/R² are regression metrics.
- **D** is wrong — bytes scanned is an Athena cost metric, unrelated to model quality.
</details>

### Q73. [CASE]
A regulated healthcare ML platform requires: no public internet from training jobs, encryption with auditable keys, least-privilege access, and full API audit trails. Which set of controls satisfies this? (Choose the best single answer.)

- **A.** VPC-only training + S3/interface VPC endpoints; SSE-KMS with customer-managed keys; least-privilege IAM roles; CloudTrail enabled.
- **B.** Public subnet with IGW; SSE-S3 default; AdministratorAccess role; no logging.
- **C.** Public S3 bucket; keys in the script; shared admin role; email logs.
- **D.** Disable encryption; allow 0.0.0.0/0; root credentials.

<details><summary>Answer & explanation</summary>

**Correct: A.** This bundles all four controls correctly: **VPC isolation** (private subnets + VPC endpoints, no internet), **KMS customer-managed keys** (auditable, rotatable encryption), **least-privilege IAM**, and **CloudTrail** for API audit.

- **B, C, D** are wrong — each violates one or more requirements (public networking, over-privileged roles, hardcoded/absent keys, disabled encryption, or no logging).
</details>

### Q74. [CASE]
An MLOps team wants a single **ML-native** pipeline that processes data, trains, evaluates, conditionally registers the model, and — on approval — deploys, all with built-in **lineage tracking** and minimal extra infrastructure. They are fully on SageMaker. Which orchestrator and why?

- **A.** SageMaker Pipelines — native ML steps, Model Registry integration, and automatic lineage with no extra infra to run.
- **B.** MWAA — because Airflow is always required for ML.
- **C.** Manual bash scripts on a laptop.
- **D.** Cron on EC2.

<details><summary>Answer & explanation</summary>

**Correct: A.** For a **SageMaker-native** flow needing lineage and Model Registry integration with least infra, **SageMaker Pipelines** is purpose-built and serverless.

- **B** is wrong — MWAA suits **cross-service** or Airflow-standardized orgs but adds a managed Airflow environment; it's not required for ML and is more infra here.
- **C, D** are wrong — laptop scripts and EC2 cron lack lineage, retries, governance, and reliability.
</details>

### Q75. [MCQ]
A team wants to A/B test two model versions by sending **90% of traffic to model A and 10% to model B** on the **same** endpoint, measuring live metrics before fully switching. Which SageMaker feature supports this?

- **A.** Production variants with weighted traffic on one endpoint.
- **B.** Two completely separate endpoints with no traffic control.
- **C.** Batch transform.
- **D.** Feature Store online store.

<details><summary>Answer & explanation</summary>

**Correct: A.** **Production variants** let you host multiple models behind one endpoint and assign **traffic weights** (e.g., 90/10) for A/B testing, then adjust weights to roll out the winner.

- **B** is wrong — separate endpoints don't give you weighted split on one endpoint and complicate comparison.
- **C** is wrong — batch transform is offline, not live A/B traffic splitting.
- **D** is wrong — the Feature Store serves features, not model traffic splitting.
</details>

### Q76. [CASE]
A cost review finds three workloads: (1) a nightly interruptible training job, (2) a 24/7 steady real-time endpoint, (3) an experimentation notebook used a few hours/week. What is the **MOST cost-effective** purchasing choice for each?

- **A.** (1) Managed Spot Training, (2) Savings Plan/Reserved, (3) stop the notebook when idle / on-demand.
- **B.** (1) Reserved 3-yr, (2) Spot, (3) always-on large instance.
- **C.** (1) On-Demand, (2) On-Demand, (3) On-Demand.
- **D.** (1) Dedicated Host, (2) Spot, (3) Reserved 3-yr.

<details><summary>Answer & explanation</summary>

**Correct: A.** Interruptible training → **Managed Spot** (up to ~90% off). Steady 24/7 endpoint → **Savings Plan / Reserved** (deep discount for predictable always-on). Occasional notebook → **stop it when idle** (or on-demand), since committing or running it 24/7 wastes money.

- **B** is wrong — reserving a 3-yr term for a nightly job over-commits; Spot on a 24/7 real-time endpoint risks interruptions.
- **C** is wrong — all On-Demand leaves the largest discounts on the table.
- **D** is wrong — Dedicated Hosts are for licensing/isolation, Spot is unsafe for the always-on endpoint, and a 3-yr reservation on an occasional notebook wastes money.
</details>

### Q77. [MCQ]
A model behind a real-time endpoint sees **feature attribution values shifting** (a once-minor feature now dominates predictions), even though top-line accuracy hasn't dropped yet. Which Model Monitor type surfaces this early-warning signal?

- **A.** Feature attribution drift monitoring (Clarify-based).
- **B.** Data quality monitoring.
- **C.** Model quality monitoring.
- **D.** Infrastructure monitoring.

<details><summary>Answer & explanation</summary>

**Correct: A.** **Feature attribution drift** (via Clarify) tracks changes in **feature importance** over time and can flag a problem **before** accuracy visibly degrades — exactly the described early signal.

- **B** is wrong — data quality flags input-distribution/schema issues, not importance shifts.
- **C** is wrong — model quality needs labels and reacts to accuracy changes, which haven't dropped yet.
- **D** is wrong — infra monitoring watches latency/CPU, not attributions.
</details>

### Q78. [CASE]
A retailer needs to forecast **per-store, per-product** demand for 50,000 series and deploy predictions as a **nightly batch** written to S3. Which end-to-end choice is best?

- **A.** Train DeepAR (multi-series forecasting) → run Batch transform nightly to S3.
- **B.** Train Random Cut Forest → real-time endpoint per store.
- **C.** Train XGBoost per store as 50,000 separate endpoints.
- **D.** Train BlazingText → serverless inference.

<details><summary>Answer & explanation</summary>

**Correct: A.** **DeepAR** learns one global model across many related **time series** (per-store/product) and **batch transform** is the cost-effective way to score them all nightly to S3.

- **B** is wrong — RCF is anomaly detection, not forecasting; 50,000 endpoints is absurd cost.
- **C** is wrong — XGBoost isn't native multi-series forecasting, and 50,000 endpoints is unmanageable and hugely expensive.
- **D** is wrong — BlazingText is for text, not demand forecasting.
</details>

### Q79. [CASE]
An ML pipeline built as **infrastructure-as-code** must deploy identically to dev, staging, and prod accounts, and the release must automatically rebuild the inference container and redeploy on each merge to main. Which combination fits?

- **A.** CloudFormation/CDK for IaC + CodePipeline (source→build→deploy) + CodeBuild to build the image to ECR.
- **B.** Manually click through the console in each account.
- **C.** Copy files over SSH to each server.
- **D.** Store the pipeline as a Word document.

<details><summary>Answer & explanation</summary>

**Correct: A.** **CloudFormation/CDK** gives repeatable multi-account IaC; **CodePipeline** orchestrates the merge-triggered release; **CodeBuild** rebuilds the container and pushes to **ECR** for redeploy — a standard MLOps CI/CD stack.

- **B, C, D** are wrong — manual console clicks, SSH copies, and documents are not automated, repeatable, or auditable IaC/CI-CD.
</details>

### Q80. [MCQ]
A near-real-time recommendation service needs to enrich each request with the user's **latest** aggregated features (updated seconds ago by a streaming job) at **millisecond** read latency, and the **same** feature definitions must be reused to build the training set. Which design meets both online and offline needs with the least duplication?

- **A.** SageMaker Feature Store with **both** online (real-time reads) and offline (training history) stores enabled on the feature group.
- **B.** Store features only in S3 and query with Athena at request time.
- **C.** Recompute features from raw data on every request.
- **D.** Keep two independent, hand-synced databases.

<details><summary>Answer & explanation</summary>

**Correct: A.** Enabling **both stores** on a Feature Store feature group gives **millisecond online reads** for serving and an **append-only offline history** for training from the **same** definitions — eliminating training/serving skew and duplication (online writes replicate to offline automatically).

- **B** is wrong — Athena/S3 latency is seconds, failing the millisecond read requirement.
- **C** is wrong — recomputing per request adds latency and risks train/serve skew.
- **D** is wrong — two hand-synced databases are duplicative and drift-prone (the exact problem Feature Store solves).
</details>

---

## Score guide <a name="score-guide"></a>

Tally your correct answers (multiple-response counts as correct only if **all** required options are right).

| Score (out of 80) | Percentage | Interpretation |
|---|---|---|
| **72–80** | 90–100% | Exam-ready. You have the reflexes down — do a final timed run and book the exam. |
| **60–71** | 75–89% | On track. AWS scales MLA-C01 to a **720/1000** pass. Re-drill the 1–2 weakest domains and the qualifiers you misread. |
| **48–59** | 60–74% | Borderline. Revisit the domain concept files (`01`–`04`) for every miss, then re-attempt this bank. |
| **Below 48** | < 60% | Not ready yet. Study each domain end-to-end, focus on endpoint selection, metrics, drift types, and cost/purchasing options, then return. |

**Per-domain check:** aim for **≥ 70%** in *every* domain, not just overall — the real exam can fail you on lopsided weakness in a heavily weighted domain (Data Prep 28%, Model Dev 26%, Monitoring/Security 24%, Deploy/Orchestration 22%).

**Highest-yield reflexes to over-learn:**
- **Endpoint selection matrix:** real-time (25 MB / 60 s, steady) · serverless (4 MB, intermittent, scale-to-zero, cold starts) · async (1 GB / 60 min, queued, scale-to-zero) · batch transform (bulk offline, no persistent endpoint).
- **Streaming:** Firehose = simple delivery · Data Streams = ordered/replayable low latency · Managed Flink = stateful windows/joins · MSK = Kafka, highest ops.
- **Metrics:** recall (minimize false negatives) · precision (minimize false positives) · F1 (balance) · AUC (threshold-free ranking) · RMSE (regression, penalizes big errors).
- **Model Monitor 4 types:** data quality · model quality · bias drift · feature attribution drift (last two = Clarify). Data drift = P(X) change; concept drift = P(Y|X) change.
- **Deployment guardrails:** all-at-once (fastest) · canary (2 steps) · linear (many steps) · blue/green with auto-rollback.
- **Cost:** Managed Spot for interruptible training · Savings Plan/Reserved for steady endpoints · serverless/async scale-to-zero for spiky/idle.

---

### References
- [Inference options in Amazon SageMaker AI](https://docs.aws.amazon.com/sagemaker/latest/dg/deploy-model-options.html)
- [Deploy models with Amazon SageMaker Serverless Inference](https://docs.aws.amazon.com/sagemaker/latest/dg/serverless-endpoints.html)
- [Data and model quality monitoring with Amazon SageMaker Model Monitor](https://docs.aws.amazon.com/sagemaker/latest/dg/model-monitor.html)
- [Feature attribution drift for models in production](https://docs.aws.amazon.com/sagemaker/latest/dg/clarify-model-monitor-feature-attribution-drift.html)
- [Bias drift for models in production](https://docs.aws.amazon.com/sagemaker/latest/dg/clarify-model-monitor-bias-drift.html)
- [Understand the hyperparameter tuning strategies (AMT: Bayesian, Hyperband, Random, Grid)](https://docs.aws.amazon.com/sagemaker/latest/dg/automatic-model-tuning-how-it-works.html)
- [Deployment guardrails for updating models in production](https://docs.aws.amazon.com/sagemaker/latest/dg/deployment-guardrails.html)
- [Blue/Green Deployments](https://docs.aws.amazon.com/sagemaker/latest/dg/deployment-guardrails-blue-green.html)
- [SageMaker Feature Store storage configurations (online vs offline)](https://docs.aws.amazon.com/sagemaker/latest/dg/feature-store-storage-configurations.html)
- [Built-in algorithms and pretrained models in Amazon SageMaker](https://docs.aws.amazon.com/sagemaker/latest/dg/algos.html)
- [Amazon Managed Service for Apache Flink (renamed from Kinesis Data Analytics)](https://aws.amazon.com/blogs/aws/announcing-amazon-managed-service-for-apache-flink-renamed-from-amazon-kinesis-data-analytics/)
- [Amazon Data Firehose (renamed from Kinesis Data Firehose)](https://aws.amazon.com/blogs/big-data/top-6-game-changers-from-aws-that-redefine-streaming-data/)
