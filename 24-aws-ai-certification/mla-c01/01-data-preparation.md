# Domain 1: Data Preparation for Machine Learning

This is **Domain 1 of the AWS Certified Machine Learning Engineer – Associate (MLA-C01)** exam and it carries **28% of the scored questions — the single largest domain**. It tests whether you can *ingest and store* raw data, *transform and engineer features* from it, and *guarantee its integrity* (bias, PII, quality) before a model ever sees it. Master this chapter and you have locked down more than a quarter of the exam plus the vocabulary the rest of the exam assumes you already know.

> **Plain English:** A model is only as good as the data you feed it. Domain 1 is "kitchen prep" for ML — sourcing the ingredients (ingest), washing and chopping them (transform), and making sure nothing is spoiled or unfair (integrity) *before* you cook (train).

---

## Table of Contents
- [The data-prep pipeline at a glance](#pipeline)
- [1.1a Data formats: row vs columnar](#formats)
- [1.1b AWS data sources & storage options](#sources)
- [1.1c Streaming ingestion](#streaming)
- [1.1d Extracting from storage & performance knobs](#extract)
- [1.1e Ingesting into Data Wrangler & Feature Store](#ingest-sm)
- [1.1f Merging data from multiple sources](#merge)
- [1.2a Cleaning & transformation](#cleaning)
- [1.2b Feature engineering](#feature-eng)
- [1.2c Encoding categorical & text data](#encoding)
- [1.2d Transformation tools](#tools)
- [1.2e Data labeling: Ground Truth & Mechanical Turk](#labeling)
- [1.2f SageMaker Feature Store deep-dive](#feature-store)
- [1.3a Pre-training bias metrics (CI, DPL)](#bias-metrics)
- [1.3b Fixing class imbalance](#imbalance)
- [1.3c Encryption, classification, anonymization, PII/PHI](#security)
- [1.3d Validating data quality](#quality)
- [1.3e Bias sources & SageMaker Clarify](#clarify)
- [1.3f Reducing prediction bias: split, shuffle, augment](#split)
- [1.3g Loading data into the training resource](#input-modes)
- [Exam traps & quick-fire review](#traps)
- [References](#references)

---

## The data-prep pipeline at a glance <a name="pipeline"></a>

🧠 **Mental model:** Think of Domain 1 as three gates every dataset must pass through. **Gate 1 (1.1)** = get the data *in* and *stored* in the right shape. **Gate 2 (1.2)** = *reshape* it into features a model can learn from. **Gate 3 (1.3)** = *certify* it's fair, legal, and clean before training.

```mermaid
flowchart LR
    subgraph SRC["Sources"]
        S3[("Amazon S3")]
        RDS[("Amazon RDS")]
        DDB[("DynamoDB")]
        STR["Kinesis / MSK<br/>(streaming)"]
    end
    subgraph G1["Gate 1 — Ingest & Store (1.1)"]
        FMT["Pick format<br/>Parquet/CSV/JSON/ORC/Avro/RecordIO"]
        STORE[("S3 / EFS / FSx")]
    end
    subgraph G2["Gate 2 — Transform & Feature-engineer (1.2)"]
        CLEAN["Clean: outliers,<br/>impute, dedup"]
        FE["Scale, bin, encode,<br/>log-transform"]
        FS[("Feature Store")]
    end
    subgraph G3["Gate 3 — Integrity (1.3)"]
        BIAS["Bias check<br/>(Clarify: CI, DPL)"]
        PII["PII/PHI, encrypt,<br/>mask, quality"]
        SPLIT["Split / shuffle /<br/>augment"]
    end
    SRC --> G1 --> G2 --> G3 --> TRAIN["Training job"]
```

| Task | Weight-driver | Key AWS services you MUST know |
|---|---|---|
| **1.1** Ingest & store | Formats, sources, streaming | S3, EFS, FSx for NetApp ONTAP, Kinesis, MSK, Glue, RDS, DynamoDB |
| **1.2** Transform & feature-engineer | Cleaning, encoding, labeling | Data Wrangler, Glue, Glue DataBrew, Lambda, Ground Truth, Feature Store |
| **1.3** Ensure integrity | Bias, PII, quality | SageMaker Clarify, Glue Data Quality, Glue DataBrew, Macie, KMS |

---

## 1.1a Data formats: row vs columnar <a name="formats"></a>

🧠 **Mental model:** A **row format** is a spreadsheet read one *record at a time* (great when you need whole rows — transactional writes, streaming). A **columnar format** stores each *column together* (great when analytics scans a few columns over billions of rows — you skip the columns you don't need and compress each column tightly because similar values sit side by side).

```mermaid
flowchart TB
    subgraph ROW["Row storage (CSV, JSON, Avro)"]
        R1["r1: age,city,label"]
        R2["r2: age,city,label"]
        R3["read a full record fast"]
    end
    subgraph COL["Columnar storage (Parquet, ORC)"]
        C1["all ages | all cities | all labels"]
        C2["scan 1 column, skip rest → cheap analytics"]
    end
```

> **Why (the rationale):** Format choice directly affects query cost, throughput, and compatibility with SageMaker algorithms. Picking the wrong format (e.g., CSV for a columnar analytics query) wastes money and slows training.
> **When to use:** Choose columnar (Parquet/ORC) when queries scan a few columns over large datasets; choose row formats (Avro/CSV/JSON) for streaming, write-heavy, or full-record access; choose RecordIO-protobuf for SageMaker built-in algorithms in Pipe mode.
> **Nuances & gotchas:** RecordIO-protobuf is required (not optional) for best performance with many SageMaker built-in algorithms in Pipe mode; CSV has no schema enforcement so bad types silently corrupt training; Parquet/ORC require a read engine (Athena, Glue, Spark) — you cannot use them as plain text.

⚙️ **Exam-critical format table:**

| Format | Row/Columnar | Validated schema? | Best for | Exam signal |
|---|---|---|---|---|
| **CSV** | Row | No (plain text) | Small/simple tabular, human-readable | Default but no schema, no compression, no nesting |
| **JSON / JSON Lines** | Row | Semi (self-describing) | Nested/semi-structured, APIs, DeepAR input | Flexible but bulky |
| **Apache Avro** | Row | **Yes** (schema embedded) | Streaming, Kafka, schema evolution, write-heavy | "row-based + schema evolution" → **Avro** |
| **Apache Parquet** | **Columnar** | **Yes** | Analytics, Athena/Spark/Glue, ML feature stores | "columnar + query few columns + cheap S3 scan" → **Parquet** |
| **Apache ORC** | **Columnar** | **Yes** | Hive/analytics, high compression | Columnar alternative to Parquet (Hive ecosystem) |
| **RecordIO (protobuf)** | Row (binary) | Yes | **SageMaker built-in algorithms**, Pipe mode | "most efficient input for built-in algos" → **RecordIO-protobuf** |

**Validated vs non-validated:** *Validated* formats (Parquet, ORC, Avro, RecordIO-protobuf) carry a schema/typing so bad records are caught on read; *non-validated* (CSV, raw JSON) are just text and errors surface later.

⚙️ **RecordIO-protobuf** (`application/x-recordio-protobuf`) is the format many **SageMaker built-in algorithms** (Linear Learner, Factorization Machines, K-Means, PCA, NTM) prefer because it is compact binary and streams efficiently in **Pipe mode**, shrinking download time before training. ([SageMaker common data formats](https://docs.aws.amazon.com/sagemaker/latest/dg/sagemaker-algo-common-data-formats.html))

🎯 **On the exam — "if you see X pick Y":**
- "Query a *few columns* over a huge dataset in S3 cost-effectively (Athena)" → **Parquet** (or ORC).
- "Schema evolution + streaming/Kafka + row-oriented" → **Avro**.
- "Most efficient input format for a SageMaker *built-in* algorithm / Pipe mode" → **RecordIO-protobuf**.
- "Convert CSV to a columnar format to cut Athena/Glue cost" → convert to **Parquet** (use Glue or DataBrew).

---

## 1.1b AWS data sources & storage options <a name="sources"></a>

🧠 **Mental model:** Match the *shape* of your data to the *shape* of the store. Object blobs → **S3**. A shared POSIX folder many instances mount → **EFS**. A blazing-fast scratch disk for one big training run → **FSx for Lustre**. Enterprise NAS with snapshots/dedup that also speaks S3 → **FSx for NetApp ONTAP**.

> **Why (the rationale):** Each storage tier has a different cost-latency tradeoff. Using S3 alone is cheap but slow for repeated reads; FSx for Lustre is fast but expensive — matching storage to workload prevents either wasted spend or throttled training.
> **When to use:** S3 for the data lake; EFS when multiple training instances or notebooks need shared POSIX access to the same files simultaneously; FSx for Lustre for large-scale distributed training where I/O is the bottleneck; FSx for NetApp ONTAP when you need enterprise snapshot/dedup features AND want S3-API access for SageMaker/Bedrock/Athena.
> **Nuances & gotchas:** EFS costs ~3× more per GB than S3 and is regional; FSx for Lustre is significantly more expensive still — only use it if I/O is the measured bottleneck. Serverless Inference does NOT support EFS or FSx mounts. FSx for Lustre can be linked to an S3 bucket and lazily hydrates data on demand (no pre-copy needed).

⚙️ **Storage decision table (memorize the "pick when"):**

| Service | Type | Throughput / latency | Pick when… | Exam trap |
|---|---|---|---|---|
| **Amazon S3** | Object store | High throughput, higher latency per object | Default ML data lake; cheap, 11-nines durability, tiers | Not a file system; use Fast File/Pipe mode to stream to training |
| **Amazon EFS** | Elastic NFS file system | Shared, moderate | **Multiple** instances/notebooks share the *same* files; no copy needed | Higher $/GB than S3; regional |
| **Amazon FSx for Lustre** | High-perf parallel FS | **Sub-ms, 100s GB/s** | Large-scale distributed training needing fastest reads; can link to an S3 bucket | Costly; best for heavy repeated epochs |
| **Amazon FSx for NetApp ONTAP** | Managed NetApp NAS | Multi-protocol (NFS, SMB, **iSCSI**, S3) | Enterprise ONTAP features (snapshots, dedup, cloning) + **S3-API access** for SageMaker/Bedrock/Athena | Named explicitly in blueprint — know it does dedup/snapshots and speaks S3 |
| **Amazon EBS** | Block volume (single instance) | Configurable IOPS | Attached disk for one EC2/notebook instance | Not shared across instances |

⚙️ **FSx for NetApp ONTAP** is called out by name in the exam guide: it is a fully managed NetApp ONTAP file system that supports NFS, SMB, and iSCSI, offers snapshots/dedup/compression, and lets **S3-API-based apps (SageMaker training, Bedrock, Athena)** read the data directly. ([FSx for NetApp ONTAP](https://aws.amazon.com/fsx/netapp-ontap/), [access datasets](https://docs.aws.amazon.com/sagemaker/latest/dg/model-access-training-data.html))

🎯 **On the exam:**
- "Fastest possible I/O for large distributed training" → **FSx for Lustre**.
- "Many teams/notebooks share the *same* dataset with POSIX semantics" → **EFS**.
- "Enterprise NAS features (snapshots/dedup) + also readable via S3 API" → **FSx for NetApp ONTAP**.
- "Cheapest durable landing zone / data lake" → **S3**.

---

## 1.1c Streaming ingestion <a name="streaming"></a>

🧠 **Mental model:** Batch is a delivery truck (data arrives in bulk on a schedule). Streaming is a conveyor belt (records arrive continuously and you act within seconds). AWS gives you three belts: **Kinesis** (AWS-native), **Apache Flink / Managed Service for Apache Flink** (real-time processing on the belt), and **Amazon MSK** (managed Apache Kafka).

> **Why (the rationale):** Streaming ingestion enables near-real-time ML feature freshness and event-driven retraining triggers. Without it, feature lag causes stale predictions (e.g., a fraud model trained on yesterday's patterns).
> **When to use:** Kinesis Data Streams when you need ordering, replay, and custom sub-second consumers; Firehose when you just want streaming data to land in S3/Redshift/OpenSearch with zero code; MSK when your team already uses Apache Kafka; Flink when you need stateful windowed transformations on the stream.
> **Nuances & gotchas:** Firehose is a **delivery** service — it buffers (default 60 s or 1 MB) before writing, so it is near-real-time, not sub-second. Kinesis Data Streams charges per shard-hour regardless of traffic; MSK charges per broker-hour. Flink (Managed Service for Apache Flink) cannot be used without a Kinesis or MSK source — it is a processing layer, not a storage layer.

```mermaid
flowchart LR
    PROD["Producers<br/>(apps, IoT, clickstream)"] --> KDS["Kinesis Data Streams<br/>(shards, ordered, replay)"]
    PROD --> FH["Kinesis Data Firehose<br/>(no code, buffers → S3)"]
    PROD --> MSK["Amazon MSK<br/>(managed Kafka)"]
    KDS --> FLINK["Managed Service<br/>for Apache Flink<br/>(SQL / stream analytics)"]
    MSK --> FLINK
    FH --> S3[("S3 data lake")]
    FLINK --> S3
```

⚙️ **Streaming service table:**

| Service | What it is | Pick when… |
|---|---|---|
| **Kinesis Data Streams** | Sharded, ordered, replayable stream (retention up to 365 days) | Custom consumers, sub-second, need ordering/replay, multiple readers |
| **Kinesis Data Firehose** | Zero-admin **delivery** stream | "Just land streaming data in **S3 / Redshift / OpenSearch** with buffering & optional format conversion (to Parquet)"; near-real-time (buffer interval), no code |
| **Managed Service for Apache Flink** | Managed **Apache Flink** for stream processing | Real-time aggregations/windowing/transforms in-flight (SQL or Java/Python) |
| **Amazon MSK** | Managed **Apache Kafka** | Already on Kafka, need Kafka ecosystem/partitions, portability |

🎯 **On the exam — "if you see X pick Y":**
- "Deliver streaming data to S3/Redshift/OpenSearch with **no code / no admin**, optional Parquet conversion" → **Kinesis Data Firehose**.
- "Need ordering, replay, custom consumers, sub-second" → **Kinesis Data Streams**.
- "Team already uses **Kafka**" → **Amazon MSK**.
- "Real-time windowed aggregation/transform on the stream" → **Managed Service for Apache Flink**.
- "Real-time transform of each record before landing" (lightweight) → **AWS Lambda** on the stream (see 1.2d).

---

## 1.1d Extracting from storage & performance knobs <a name="extract"></a>

> **Why (the rationale):** The bottleneck is often not the compute but the I/O pipeline pulling data from storage. Knowing the right "speed knob" per source (Transfer Acceleration, Provisioned IOPS, export-to-S3) prevents training jobs from spending most of their time waiting for data.
> **When to use:** S3 Transfer Acceleration when producers or consumers are geographically distant from the bucket's region; EBS Provisioned IOPS when a single-instance job has consistent high-IOPS disk access requirements; DynamoDB export to S3 when you need a point-in-time snapshot of a table for ML without impacting production read capacity.
> **Nuances & gotchas:** S3 Transfer Acceleration incurs additional per-GB transfer cost — only worth it for cross-continent uploads; it does NOT speed up intra-region S3 transfers. DynamoDB export to S3 produces JSON (DynamoDB JSON format) not CSV — you may need Glue to convert it. EBS volumes cannot be shared across instances (use EFS for that).

⚙️ **Where data lives and how you pull it:**

| Source | Extract with | Speed knob you must know |
|---|---|---|
| **S3** | SDK, Glue, Athena, Fast File/Pipe mode | **S3 Transfer Acceleration** (CloudFront edge for long-distance uploads); multipart upload; S3 Select |
| **Amazon EBS** | Mount to instance | **Provisioned IOPS (io1/io2)** for guaranteed high IOPS; gp3 for baseline |
| **Amazon EFS** | Mount (NFS) | Elastic/Provisioned throughput mode |
| **Amazon RDS** | JDBC, Glue connection, DMS | Read replicas to offload extraction |
| **Amazon DynamoDB** | Scan/Query, **DynamoDB export to S3**, Glue connector | Export-to-S3 (no read-capacity impact) for bulk ML extraction |

⚙️ **Two named performance features the exam loves:**
- **S3 Transfer Acceleration** — speeds up *uploads/downloads over long geographic distances* by routing through the nearest CloudFront edge location. Pick it when data must cross regions/continents fast.
- **EBS Provisioned IOPS (io1/io2)** — guarantees a set IOPS level for I/O-intensive workloads on a single instance's block volume.

🎯 **On the exam:**
- "Users worldwide uploading large files to one S3 bucket, slow" → **S3 Transfer Acceleration**.
- "Need consistent high disk IOPS on a training/notebook instance" → **EBS Provisioned IOPS**.
- "Bulk-export a DynamoDB table for ML without hurting production traffic" → **DynamoDB export to S3**.

---

## 1.1e Ingesting into Data Wrangler & Feature Store <a name="ingest-sm"></a>

> **Why (the rationale):** Data Wrangler removes the need to write custom ETL scripts for ML feature prep; Feature Store eliminates the re-engineering effort that causes training/serving skew when two teams build the same feature differently.
> **When to use:** Use Data Wrangler when a data scientist wants a GUI to explore, clean, and engineer features and then export that work to a repeatable pipeline or Feature Store ingestion job. Use Feature Store ingestion when features need to be available both in real time (online store) and for training (offline store).
> **Nuances & gotchas:** Data Wrangler flows run as SageMaker Processing jobs — they incur compute cost; idle Studio sessions also incur charges if you forget to shut them down. Feature Store batch ingestion writes to S3 within ~15 minutes of the processing job completing, not instantly. Feature Store does NOT transform features — it only stores and retrieves them.

⚙️ **SageMaker Data Wrangler** is the visual data-prep tool inside SageMaker Studio. It **imports from S3, Athena, Amazon Redshift, Snowflake, and other sources**, applies 300+ built-in transforms, and can **export the flow** to a Processing job, a Feature Store ingestion job, a SageMaker Pipeline, or Python code. ([Data Wrangler](https://aws.amazon.com/sagemaker/ai/data-wrangler/), [Transform Data](https://docs.aws.amazon.com/sagemaker/latest/dg/data-wrangler-transform.html))

⚙️ **SageMaker Feature Store** ingests two ways (details in [1.2f](#feature-store)):
- **Streaming ingest** — synchronous `PutRecord` API for real-time single/small-batch writes.
- **Batch ingest** — a SageMaker Processing job (often exported from Data Wrangler) writes features in bulk. ([Data sources and ingestion](https://docs.aws.amazon.com/sagemaker/latest/dg/feature-store-ingest-data.html))

🎯 **On the exam:** "Author features visually, then push them to Feature Store" → **Data Wrangler → Feature Store ingestion job**.

---

## 1.1f Merging data from multiple sources <a name="merge"></a>

🧠 **Mental model:** Real ML data is scattered across a database, a data lake, and an API. You need to *join and unify* it. Three tools, escalating in power:

> **Why (the rationale):** Training a model on partial data — from just one source — causes it to miss patterns that only emerge when sources are joined. Glue automates schema discovery and join at scale without provisioning servers.
> **When to use:** Glue when you need serverless Spark ETL with catalog integration across S3, RDS, and JDBC sources; Spark on EMR when you need full cluster control, complex custom transforms, or very large distributed joins that exceed Glue job limits; pandas only for small in-notebook experiments.
> **Nuances & gotchas:** Glue Crawlers infer schema automatically but can mistype columns (e.g., treating an integer ID as a string) — always verify the Catalog after crawling. Glue DPU billing starts at 2 DPUs minimum and rounds up; short jobs are relatively more expensive. Glue jobs have a default timeout of 2880 minutes; set it explicitly or you may accumulate zombie job charges.

| Approach | Use when |
|---|---|
| **Programming (pandas / boto3)** | Small data, ad-hoc, in a notebook |
| **AWS Glue** (serverless Spark ETL + Data Catalog) | Serverless, catalog-driven joins across S3/RDS/JDBC at scale; crawlers infer schema |
| **Apache Spark** (on Glue or EMR) | Very large distributed joins/aggregations; full control |

⚙️ **AWS Glue** = serverless ETL. **Crawlers** infer schemas into the **Glue Data Catalog**; **Glue jobs** (Spark) do the joins/transforms; visual authoring via **Glue Studio**. It is the default AWS answer for "merge/ETL multiple sources at scale without managing servers."

🎯 **On the exam:**
- "Merge data from S3 + RDS + a JDBC source, serverless, at scale" → **AWS Glue**.
- "Massive distributed join, need full Spark control / already on EMR" → **Apache Spark on EMR**.
- "Combine a couple of small CSVs in a notebook" → **pandas**.

---

## 1.2a Cleaning & transformation <a name="cleaning"></a>

🧠 **Mental model:** Cleaning is quality control on the assembly line — pull out the defective parts (outliers), fill the gaps (impute), and remove duplicates so you don't count the same thing twice.

> **Why (the rationale):** Dirty data — outliers, missing values, duplicates — distorts model weights and inflates training metrics while degrading real-world performance. Clean data is the single highest-ROI preprocessing step.
> **When to use:** Apply imputation (not row-dropping) when missingness is less than ~20% of a feature; use outlier treatment when IQR/z-score analysis shows extreme values that don't reflect realistic future inputs; dedup before any split to avoid the same example appearing in both train and test sets (data leakage).
> **Nuances & gotchas:** Dropping rows with missing values can introduce selection bias if missingness is non-random (e.g., high-income respondents skip income fields). Mean imputation reduces feature variance and can hurt models that rely on spread. Deduplication must happen before train/test split — deduplication after the split may still leave near-duplicates across splits.

⚙️ **Core cleaning operations:**

| Problem | Technique | Notes |
|---|---|---|
| **Missing values** | **Imputation** — mean/median/mode, constant, KNN, or drop | Median is robust to outliers; drop only if few rows/columns affected |
| **Outliers** | Detect via **IQR, z-score/standard deviation, robust z-score**; then cap (winsorize), transform, or remove | Data Wrangler has a "Handle outliers" transform |
| **Duplicates** | **Deduplication** — drop exact/near-duplicate rows | Prevents leakage and skewed weighting |
| **Inconsistent formats** | Standardize types, parse dates, fix casing | DataBrew/Data Wrangler transforms |
| **Combining** | Join/concatenate/merge sources | See [1.1f](#merge) |

🎯 **On the exam:** "Numeric column has extreme values distorting the model" → detect with **z-score/IQR** and treat (cap/transform); "many missing values in a feature" → **impute** (median for skewed data), don't blindly drop.

---

## 1.2b Feature engineering <a name="feature-eng"></a>

🧠 **Mental model:** Feature engineering is translating raw numbers into a language the model understands well — putting everything on a comparable scale, and reshaping skewed or lumpy signals into cleaner ones.

> **Why (the rationale):** Algorithms like KNN, SVM, and neural networks are sensitive to feature scale — a feature with range [0, 1,000,000] will dominate one in [0, 1] purely due to magnitude, not predictive power. Transforms fix this without changing the underlying information.
> **When to use:** Normalize (min-max) for distance-based and neural-net models where a bounded range is expected; standardize (z-score) for linear models, PCA, and any algorithm that assumes zero-mean inputs; log-transform right-skewed numeric features (income, counts, prices) before training; bin continuous features when domain knowledge suggests meaningful thresholds (e.g., age groups).
> **Nuances & gotchas:** Apply scaling/normalization parameters fitted on the **training set only** — then apply the same transform to validation and test sets. Fitting on the full dataset before splitting is a common form of data leakage. Log-transform requires all values > 0; use `log(x+1)` if zeros exist. Binning discards within-bin variation — use cautiously.

⚙️ **Numeric transforms:**

| Technique | What it does | When to use |
|---|---|---|
| **Normalization (min-max)** | Rescale to **[0,1]** | Bounded range needed; distance-based algos (KNN, NN) |
| **Standardization (z-score)** | Mean 0, std 1 | Many linear models, PCA; robust to differing units |
| **Log transformation** | `log(x)` compresses large values | Right-skewed data (income, counts) → more normal |
| **Binning / bucketing** | Continuous → discrete buckets | Reduce noise, capture non-linear thresholds (age groups) |
| **Feature splitting** | Break one field into parts | `2026-07-17` → year/month/day; address → city/zip |
| **Scaling** | Umbrella for min-max/standard | Required when features have very different magnitudes |

> **Plain English — normalize vs standardize:** *Normalize* squeezes values into a fixed box [0,1]. *Standardize* re-centers around 0 with unit spread. Distance/NN models like normalization; linear models/PCA like standardization.

🎯 **On the exam:** "Feature is heavily right-skewed" → **log transform**. "Features on wildly different scales hurt a distance-based/gradient model" → **scale/standardize**. "Turn continuous age into groups" → **binning**.

---

## 1.2c Encoding categorical & text data <a name="encoding"></a>

🧠 **Mental model:** Models eat numbers, not words. Encoding turns categories into numbers *without inventing a fake ordering*.

> **Why (the rationale):** Assigning arbitrary integers (1, 2, 3) to nominal categories like {Red, Blue, Green} implies an ordering that does not exist, misleading linear models and distance-based algorithms. One-hot encoding avoids this; label encoding is only safe for truly ordinal data or tree models that ignore the numeric ordering.
> **When to use:** One-hot for nominal categories with low cardinality (< ~30 unique values); label encoding for ordinal categories or when feeding a tree-based model (XGBoost/Random Forest tolerate arbitrary integers); binary encoding for medium-cardinality categories where one-hot would create too many columns; tokenization as the first step before embedding for any NLP input.
> **Nuances & gotchas:** One-hot on high-cardinality features (hundreds of cities) creates a column explosion that slows training and inflates memory — use hashing or embedding instead. Label-encoding a nominal feature in a linear model is a classic mistake that introduces a false gradient. Tokenization vocabulary must be built on training data only; unseen tokens at inference must be handled with an `<UNK>` token or subword fallback.

⚙️ **Encoding table:**

| Encoding | How | Use when | Trap |
|---|---|---|---|
| **One-hot** | One binary column per category | **Nominal** (no order), low cardinality | High cardinality → column explosion (use sparse or hashing) |
| **Label encoding** | Each category → integer | **Ordinal** (has order: low/med/high) or tree models | Implies false order if applied to nominal data |
| **Binary encoding** | Category → integer → binary digits across columns | Medium/high cardinality (fewer columns than one-hot) | Middle ground between label and one-hot |
| **Tokenization** | Split **text** into tokens/subwords → IDs | NLP / LLM inputs | Precedes embeddings |

🎯 **On the exam — "if you see X pick Y":**
- "Categorical with **no inherent order**, few values" → **one-hot**.
- "Ordinal categories (S/M/L) or feeding a tree model" → **label encoding**.
- "High-cardinality category, one-hot too wide" → **binary** (or hashing).
- "Prepare raw text for an NLP model" → **tokenization**.

---

## 1.2d Transformation tools <a name="tools"></a>

> **Why (the rationale):** Data transformation tools differ in their target user (data engineer vs data scientist vs analyst), their execution model (serverless Spark vs visual GUI vs Lambda function), and their ML-awareness. Picking the right one avoids building a hand-coded Spark job when a no-code DataBrew recipe would suffice, or using DataBrew when you need ML-specific leakage detection.
> **When to use:** Data Wrangler when an ML practitioner wants visual feature engineering with direct export to SageMaker Pipelines or Feature Store; Glue for code/Spark ETL at scale with catalog integration; DataBrew for no-code cleaning and profiling by analysts; Lambda for lightweight, per-record streaming transforms on Kinesis/Firehose.
> **Nuances & gotchas:** Data Wrangler does NOT run continuously in production — you must export the flow to a Processing job or Pipeline. DataBrew cannot export directly to Feature Store. Lambda has a 15-minute timeout and 10 GB memory limit; it is not suitable for heavy batch transforms. Glue DataBrew and AWS Glue Data Quality are separate services — DataBrew is visual prep, Data Quality is rule-based pipeline validation.

⚙️ **Which tool for which job — the exam's favorite disambiguation:**

| Tool | What it is | Pick when… |
|---|---|---|
| **SageMaker Data Wrangler** | Visual, 300+ transforms, in Studio, ML-aware (data-quality/leakage insights, export to Feature Store/Pipelines) | End-to-end **ML** feature prep with code export; data scientists |
| **AWS Glue** | Serverless **Spark** ETL + Data Catalog | Code/Spark ETL, cataloging, scheduled pipelines at scale |
| **AWS Glue DataBrew** | **No-code** visual data prep, 250+ pre-built transforms, profiling | Analysts who want point-and-click cleaning/profiling, **no code** |
| **AWS Lambda** | Serverless functions | **Lightweight per-record streaming transforms** (e.g., on Firehose/Kinesis) |
| **Apache Spark (EMR/Glue)** | Distributed compute | Very large custom transforms |

> **Plain English — Data Wrangler vs Glue vs DataBrew:** Data Wrangler is *ML-focused* prep with code export. Glue is *code/Spark ETL* for engineers. DataBrew is *no-code* point-and-click for analysts. All three overlap on cleaning; the exam distinguishes by "no-code" (DataBrew), "ML feature prep with export" (Data Wrangler), or "serverless Spark ETL/catalog" (Glue).

⚙️ **Streaming transforms:** For per-record transforms *in the stream*, use **AWS Lambda** (attach to Kinesis/Firehose) or **Apache Spark / Managed Service for Apache Flink** for stateful/windowed processing.

🎯 **On the exam:**
- "No-code visual cleaning + profiling for analysts" → **Glue DataBrew**.
- "Visual ML feature prep, export to Feature Store/Pipeline" → **Data Wrangler**.
- "Serverless Spark ETL across sources with a catalog" → **AWS Glue**.
- "Transform each streaming record cheaply" → **Lambda**.

---

## 1.2e Data labeling: Ground Truth & Mechanical Turk <a name="labeling"></a>

🧠 **Mental model:** Supervised learning needs labels. **SageMaker Ground Truth** is the labeling factory; the *workforce* doing the labeling can be your own team, a vendor, or the anonymous crowd (**Mechanical Turk**). Ground Truth's **automated (active) labeling** lets a model label the easy examples so humans only handle the hard ones — cutting cost up to ~70%.

> **Why (the rationale):** High-quality labels are the most expensive ingredient in supervised ML. Ground Truth cuts cost via automated labeling while maintaining quality by routing only uncertain examples to humans. Mechanical Turk provides massive scale but at the cost of data privacy.
> **When to use:** Ground Truth + Mechanical Turk for large-volume, non-sensitive image/text labeling tasks; Ground Truth + private workforce when data is confidential, contains PII/PHI, or requires domain expertise (medical imaging, legal docs); automated labeling when the dataset is large enough that a model can confidently label the "easy" majority.
> **Nuances & gotchas:** Mechanical Turk requires you to set the `FreeOfPersonallyIdentifiableInformation` flag — if your data contains PII the job will fail or violate terms. Automated labeling requires a minimum dataset size to train the internal labeling model (~1,250 labeled examples as a starting point). Ground Truth consolidates annotations from multiple workers using annotation consolidation algorithms — the raw individual labels are not the final labels.

⚙️ **Workforce options:**

| Workforce | Who | Use when |
|---|---|---|
| **Private** | Your own employees | Confidential/PII/PHI data |
| **Vendor** | AWS Marketplace vendor | Specialized/scaled labeling |
| **Amazon Mechanical Turk** | Public crowd (most workers) | Large volume, **non-sensitive** data |

🎯 **On the exam — critical trap:** When you use **Mechanical Turk**, the input data must be **free of PII** — you set the `FreeOfPersonallyIdentifiableInformation` flag or the labeling job fails. So for **sensitive/PII data, use a private workforce**, never Mechanical Turk. ([Mechanical Turk workforce](https://docs.aws.amazon.com/sagemaker/latest/dg/sms-workforce-management-public.html), [Ground Truth](https://docs.aws.amazon.com/sagemaker/latest/dg/sms.html))

- "Need to label a huge public image set cheaply" → **Ground Truth + Mechanical Turk**.
- "Data contains PII/PHI and needs labeling" → **Ground Truth + private workforce** (not Mechanical Turk).
- "Cut labeling cost — model labels easy items" → **Ground Truth automated data labeling**.

---

## 1.2f SageMaker Feature Store deep-dive <a name="feature-store"></a>

🧠 **Mental model:** Feature Store is a shared pantry of ready-to-use features so every team stops re-cooking the same ingredients — and so the features used in *training* exactly match those used in *serving* (no training/serving skew).

> **Why (the rationale):** Without a Feature Store, two teams often compute the same feature differently (one uses `log(x+1)`, one uses `log(x)`) causing silent training/serving skew. Feature Store enforces a single definition and makes features reusable across models and teams.
> **When to use:** Use the **online store** when a real-time inference endpoint needs low-latency (single-digit ms) feature lookups at prediction time; use the **offline store** to pull historical feature snapshots for training and batch inference jobs.
> **Nuances & gotchas:** The online store (backed by a managed key-value store) and offline store (S3 Parquet) are updated independently — you must enable **both** when creating a Feature Group to avoid skew. Online store does NOT support arbitrary queries — only `GetRecord` by a record identifier. Offline store data appears in S3 within ~15 minutes of a batch ingestion job; it is NOT updated in real time from `PutRecord` calls. Feature Store charges separately for online storage (per GB/month) and offline writes.

```mermaid
flowchart LR
    ING["PutRecord (stream)<br/>or batch Processing job"] --> FG["Feature Group"]
    FG --> ON[("Online store<br/>low-latency, real-time inference")]
    FG --> OFF[("Offline store<br/>S3, Parquet, historical / training")]
    ON --> INF["Real-time endpoint lookup"]
    OFF --> TRAIN["Training & batch inference"]
```

⚙️ **Exam facts you must retain:**

| Concept | Detail |
|---|---|
| **Online store** | Low-latency, real-time single-record lookups for inference |
| **Offline store** | Sits in **your S3**, stored as **Parquet**; historical data for training/batch |
| **Both enabled** | They sync to prevent training/serving discrepancies |
| **Offline table formats** | Supports **AWS Glue** (default) and **Apache Iceberg** table formats |
| **Ingestion — streaming** | Synchronous **`PutRecord`** API |
| **Ingestion — batch** | Processing job (e.g., exported from Data Wrangler); data buffered and written to S3 **within ~15 minutes** |
| **Feature group** | The schema/table definition; needs a record identifier + event-time feature |

Source: [Feature Store](https://docs.aws.amazon.com/sagemaker/latest/dg/feature-store.html), [offline store format](https://docs.aws.amazon.com/sagemaker/latest/dg/feature-store-offline.html), [ingestion](https://docs.aws.amazon.com/sagemaker/latest/dg/feature-store-ingest-data.html).

🎯 **On the exam:**
- "Reuse features across teams + keep training and serving consistent" → **SageMaker Feature Store**.
- "Real-time inference feature lookup" → **online store**; "historical features for training" → **offline store** (Parquet in S3).

---

## 1.3a Pre-training bias metrics: CI and DPL <a name="bias-metrics"></a>

🧠 **Mental model:** Before training, **SageMaker Clarify** measures whether the *raw data* is already unfair to a group (a **facet**, e.g., a demographic). These metrics are **model-agnostic** — they only look at the data and labels, not any model output.

> **Why (the rationale):** A model trained on biased data will amplify that bias in production. CI and DPL let you quantify and document the bias before training so you can decide whether to fix the data (resample, augment) or at minimum disclose the limitation.
> **When to use:** Run Clarify pre-training bias analysis any time you have a sensitive attribute (gender, age, race, zip code) in your dataset that could lead to discriminatory model behavior. Especially important in regulated domains (hiring, credit, healthcare).
> **Nuances & gotchas:** CI measures *representation* (how many samples per group); DPL measures *outcome* (positive label rate per group) — they are complementary, not redundant. Clarify runs these as a SageMaker **Processing job** — it is not a real-time check. A CI or DPL value of 0 does not guarantee the trained model will be unbiased; post-training bias metrics are needed too.

⚙️ **The two blueprint-named metrics:**

| Metric | Question it answers | Formula (intuition) | Range |
|---|---|---|---|
| **Class Imbalance (CI)** | Is one facet **under-represented** in the data (fewer samples)? | `(n_a − n_d) / (n_a + n_d)` — normalized difference in group sizes | **[−1, 1]**; 0 = balanced; +1 = only facet *a* present |
| **Difference in Proportions of Labels (DPL)** | Do groups get **positive labels at different rates**? | `q_a − q_d` (positive-label proportion of *a* minus that of *d*) | **[−1, 1]**; 0 = equal positive-label rates |

- **CI** = *representation* imbalance (how many of each group). **DPL** = *outcome* imbalance (how favorably each group is labeled).
- Clarify computes these in a **Processing job**; you select metrics via `methods=["CI","DPL"]` (or `"all"`). ([Class Imbalance](https://docs.aws.amazon.com/sagemaker/latest/dg/clarify-bias-metric-class-imbalance.html), [Pre-training bias metrics](https://docs.aws.amazon.com/sagemaker/latest/dg/clarify-measure-data-bias.html))

🎯 **On the exam:**
- "One demographic has far **fewer rows**" → **Class Imbalance (CI)**.
- "One group is **labeled positive** far more often than another" → **DPL**.
- "Measure bias **before** training, model-agnostic" → **SageMaker Clarify pre-training metrics**.

---

## 1.3b Fixing class imbalance <a name="imbalance"></a>

🧠 **Mental model:** If 99% of rows are "not fraud," a model can score 99% accuracy by never predicting fraud. You must rebalance so the minority class gets heard.

> **Why (the rationale):** Class imbalance causes the model to optimize for the majority class, ignoring the rare class that is often the most valuable (fraud, disease, equipment failure). Rebalancing realigns the loss function so minority errors are penalized appropriately.
> **When to use:** Apply oversampling/SMOTE when the minority class has very few examples and you need more signal; apply class weights (cheapest, no data change) when the dataset is large enough for the algorithm to see the minority class but you still want to up-weight its errors; undersampling only when the majority class truly adds no value (rare).
> **Nuances & gotchas:** SMOTE generates synthetic examples by interpolating between existing minority examples — it does NOT generate new information, and can create unrealistic examples in sparse feature spaces. SMOTE must be applied only on the **training set** after splitting; applying it before the split leaks synthetic data into the test set. Class weights are supported natively by XGBoost (`scale_pos_weight`) and Linear Learner but not by all algorithms.

⚙️ **Strategies:**

| Strategy | How | Note |
|---|---|---|
| **Oversampling minority** | Duplicate/synthesize minority rows (e.g., **SMOTE**) | Risk of overfitting on duplicates |
| **Undersampling majority** | Drop majority rows | Throws away data |
| **Synthetic data generation** | Generate realistic new minority samples (SMOTE, GANs) | Blueprint-named; boosts rare-class signal |
| **Class weights** | Penalize minority errors more in the loss | No data change needed |
| **Better metric** | Use **F1 / recall / AUC**, not raw accuracy | Accuracy misleads on imbalance |

🎯 **On the exam:** "Severe class imbalance, model ignores the rare class" → **resample (over/under) or generate synthetic data (SMOTE)** and evaluate with **F1/AUC**, not accuracy.

---

## 1.3c Encryption, classification, anonymization, PII/PHI <a name="security"></a>

🧠 **Mental model:** Sensitive data has legal handcuffs. You must know how to *find* it, *lock* it, and *scrub* it — and which compliance regime applies.

> **Why (the rationale):** Regulatory penalties for mishandling PII/PHI are severe (GDPR fines, HIPAA violations). Encryption, discovery (Macie), and masking must be wired in before data ever reaches a training job or labeling workforce.
> **When to use:** Amazon Macie when you need to automatically scan S3 buckets for PII/sensitive data at scale; AWS KMS for encrypting data at rest on S3, EBS, SageMaker notebook volumes, and model artifacts; Glue DataBrew or Comprehend for masking/redacting PII before sharing data downstream; data residency controls (Region choice + replication restrictions) when regulations require data to stay in a specific country.
> **Nuances & gotchas:** Macie detects PII in S3 objects but does NOT automatically mask or delete it — it only classifies and alerts. KMS SSE-KMS adds per-request API call cost and can hit KMS throughput limits under heavy parallel training. Anonymization is irreversible; masking is reversible with the key — choose correctly for the compliance requirement. "De-identification" under HIPAA has specific Safe Harbor or Expert Determination methods — not just deleting a name field.

⚙️ **Techniques & services:**

| Need | Technique / Service |
|---|---|
| **Encrypt at rest** | **AWS KMS** (S3 SSE-KMS, EBS, EFS, FSx, SageMaker volumes) |
| **Encrypt in transit** | TLS / HTTPS |
| **Discover & classify PII in S3** | **Amazon Macie** (ML-based sensitive-data discovery) |
| **Anonymization** | Irreversibly remove identifiers (no re-identification) |
| **Masking / redaction** | Hide/replace values (e.g., `****`); Glue DataBrew has masking/redaction transforms; Comprehend detects PII in text |
| **Data classification** | Tag data by sensitivity level to drive controls |

⚙️ **Compliance regimes the exam names:**

| Term | Meaning |
|---|---|
| **PII** | Personally Identifiable Information (name, SSN, email) |
| **PHI** | Protected Health Information — **HIPAA** regulated |
| **Data residency** | Data must stay in a specific **Region/country** → choose Region, block cross-Region copy |

🎯 **On the exam:**
- "Automatically **discover/classify PII** sitting in S3" → **Amazon Macie**.
- "Health data" → **PHI / HIPAA** controls (encrypt, private workforce for labeling).
- "Data must not leave a country" → **data residency** → pick the Region and restrict replication.
- "Mask/redact sensitive columns before sharing" → **Glue DataBrew** (or Comprehend for text PII).

---

## 1.3d Validating data quality <a name="quality"></a>

🧠 **Mental model:** Before training, run a health inspection. Two AWS services do this: **Glue DataBrew** (visual profiling + no-code rules) and **AWS Glue Data Quality** (rule-based checks in your pipeline using **DQDL**).

> **Why (the rationale):** "Garbage in, garbage out." Catching data quality problems before training is far cheaper than debugging a model that underperforms because 30% of its rows had corrupted values.
> **When to use:** Glue DataBrew when a data analyst wants a no-code visual profile of a new dataset; AWS Glue Data Quality (DQDL) when you need automated rule-based gates in a Glue ETL pipeline to block bad data from flowing downstream; Data Wrangler's Data Quality & Insights report when you want ML-aware checks (leakage, class imbalance) inside SageMaker Studio.
> **Nuances & gotchas:** Glue Data Quality (DQDL) is a separate service from Glue DataBrew — they are frequently confused. Data Wrangler's leakage detection is heuristic, not guaranteed — it flags suspicious correlations between features and the target but cannot definitively prove leakage. Glue DataBrew profiling jobs have their own cost (DPU-hour charges); they do not run for free.

⚙️ **Tools:**

| Tool | What it validates | Style |
|---|---|---|
| **AWS Glue DataBrew** | Profiling: missing values, cardinality, distributions, duplicates; data-quality rules | **No-code** visual |
| **AWS Glue Data Quality** | Rule-based checks (completeness, uniqueness, freshness) via **DQDL** rulesets on Catalog tables/Glue jobs | In-pipeline, automated |
| **SageMaker Data Wrangler** | **Data Quality and Insights report**: missing values, duplicate rows, outliers, class imbalance, **target/data leakage** | ML-aware, in Studio |

🎯 **On the exam:**
- "Automated quality rules inside a Glue ETL pipeline" → **AWS Glue Data Quality (DQDL)**.
- "No-code visual profiling of a dataset" → **Glue DataBrew**.
- "Detect **data leakage** / class imbalance during ML prep" → **Data Wrangler Data Quality and Insights report**.

---

## 1.3e Bias sources & SageMaker Clarify <a name="clarify"></a>

🧠 **Mental model:** Bias sneaks in at the *source*. Know the two named types and that **SageMaker Clarify** is the AWS tool that detects bias (pre- and post-training) and explains predictions (SHAP).

> **Why (the rationale):** Bias that enters at the data-collection or labeling stage cannot be trained away — it gets amplified by the model. Identifying the source of bias (selection vs. measurement) points to the correct mitigation (collect more representative data vs. fix the labeling process).
> **When to use:** Run Clarify's bias detection when a sensitive attribute is present in the dataset; investigate selection bias when your data-collection method could systematically exclude a subpopulation; investigate measurement bias when data was collected with inconsistent instruments or labelers.
> **Nuances & gotchas:** Clarify detects and measures bias — it does NOT fix it. SHAP (explainability) is computationally expensive; for large datasets use the KernelSHAP approximation (default in Clarify). Post-training bias metrics (DPPL, DI, etc.) require you to specify the predicted label column and the facet column — misconfiguring these produces meaningless results.

| Bias source | What it is | Example |
|---|---|---|
| **Selection bias** | Sample doesn't represent the population | Survey only online users → misses offline population |
| **Measurement bias** | Data collected/labeled inconsistently or with faulty instruments | One sensor miscalibrated; inconsistent labelers |

⚙️ **SageMaker Clarify** roles across the lifecycle:
- **Pre-training bias** (data): CI, DPL, and more (this domain).
- **Post-training bias** (model predictions): measured in Domain 2/4.
- **Explainability**: feature-attribution via **SHAP**.

🎯 **On the exam:** "Detect and quantify bias / explain predictions" → **SageMaker Clarify**. "Sample skewed toward one group" → **selection bias**; "labels/measurements inconsistent" → **measurement bias**.

---

## 1.3f Reducing prediction bias: split, shuffle, augment <a name="split"></a>

🧠 **Mental model:** Even clean data can mislead if you split it carelessly. Shuffle so order doesn't leak, split so the model is tested on unseen data, and augment to add variety the model would otherwise never see.

> **Why (the rationale):** An ordered dataset split without shuffling can give the model a train set from one time period and a test set from another — the "accuracy" measurement tells you nothing about real generalization. Stratification ensures every split faithfully represents the class distribution.
> **When to use:** Shuffle before splitting for any non-time-series data; use stratified splits when classes are imbalanced (even mild imbalance can cause a split to have zero minority-class examples in a small test set); chronological splits for time-series to avoid future leakage; data augmentation when you have fewer than a few thousand examples per class or when the model consistently overfits.
> **Nuances & gotchas:** Random shuffling on time-series data is incorrect — it leaks future information into training. Data augmentation must be applied only to the **training** set; augmenting the test set would distort your evaluation. Keep the test set "locked" until final evaluation — any peeking (e.g., tuning hyperparameters on test results) converts it into a validation set.

| Technique | Purpose | Watch-out |
|---|---|---|
| **Train/validation/test split** | Honest evaluation on unseen data | Common: 70/15/15 or 80/10/10; keep test untouched |
| **Shuffling** | Break ordering artifacts before splitting | Time-series: do **not** random-shuffle — split chronologically |
| **Stratified split** | Preserve class proportions in each split | Essential for imbalanced classes |
| **Data augmentation** | Synthetically expand data (flip/rotate images, paraphrase text) | Adds robustness; helps minority class |

🎯 **On the exam:**
- "Model memorized order / leaked patterns" → **shuffle before splitting** (but chronological split for time series).
- "Imbalanced classes in the split" → **stratified split**.
- "Small image dataset, model overfits" → **data augmentation**.

---

## 1.3g Loading data into the training resource <a name="input-modes"></a>

🧠 **Mental model:** How training *reads* the data changes speed and cost. You either **copy it all first** (File mode), **stream on demand** (Pipe / Fast File), or **mount a shared file system** (EFS / FSx).

> **Why (the rationale):** File mode downloads the entire dataset before the first epoch, wasting time and requiring a large local disk. Fast File mode and Pipe mode let training start immediately and reduce instance disk requirements — critical when datasets are hundreds of GB.
> **When to use:** Fast File mode as the default replacement for File mode on large S3 datasets (same POSIX file access, no full download); Pipe mode for very large sequential datasets with built-in algorithms that support streaming (RecordIO); EFS when multiple concurrent training jobs share the same feature data; FSx for Lustre for the highest-throughput distributed training on datasets accessed many times (multiple epochs).
> **Nuances & gotchas:** Pipe mode requires your training script to read from a named pipe (not a regular file path); not all custom frameworks support this natively. Fast File mode does NOT cache data locally — each read fetches from S3, so repeated random reads over many epochs can be slower than File mode on small datasets. FSx for Lustre incurs cost even when idle; tear it down after training. File mode requires the instance's local disk to be large enough to hold the full dataset.

⚙️ **SageMaker training input modes ([choosing input mode](https://docs.aws.amazon.com/sagemaker/latest/dg/model-access-training-data-best-practices.html)):**

| Input mode | Source | Behavior | Pick when… |
|---|---|---|---|
| **File mode** (default) | S3 → local disk | **Downloads the whole dataset** before training starts | Dataset fits on disk; simplest |
| **Fast File mode** | S3 | Exposes S3 as a **POSIX file system**, streams objects **on demand**, starts fast | Large S3 data, want File-like access without full download |
| **Pipe mode** | S3 | Streams data as a **Unix pipe** (often RecordIO) | Very large datasets, sequential streaming, minimize disk |
| **EFS** | EFS | **Mounts** the file system (no copy) | Data already in EFS, shared across jobs |
| **FSx for Lustre** | FSx | **Mounts** high-performance FS (can link to S3) | Fastest repeated reads for large distributed training |

🎯 **On the exam — "if you see X pick Y":**
- "Huge S3 dataset, want training to **start quickly** without downloading everything, POSIX access" → **Fast File mode**.
- "Stream a massive dataset with minimal local disk" → **Pipe mode**.
- "Data already sits in a shared file system used by many jobs" → **EFS**.
- "Need the **fastest** I/O for large-scale distributed training" → **FSx for Lustre**.
- "Small dataset, keep it simple" → **File mode**.

---

## Exam traps & quick-fire review <a name="traps"></a>

| If you see… | Pick / Answer |
|---|---|
| Query a few columns over huge S3 data cheaply (Athena) | **Parquet / ORC** (columnar) |
| Schema evolution + streaming/Kafka, row-based | **Avro** |
| Most efficient input for SageMaker **built-in** algorithms / Pipe mode | **RecordIO-protobuf** |
| Deliver streaming data to S3/Redshift/OpenSearch, **no code** | **Kinesis Data Firehose** |
| Ordering, replay, custom consumers, sub-second | **Kinesis Data Streams** |
| Team already on **Kafka** | **Amazon MSK** |
| Real-time windowed stream processing | **Managed Service for Apache Flink** |
| Fastest I/O for large distributed training | **FSx for Lustre** |
| Many notebooks share the **same** files (POSIX) | **Amazon EFS** |
| Enterprise NAS (snapshots/dedup) **+ S3-API** access | **FSx for NetApp ONTAP** |
| Speed up cross-region S3 **uploads** | **S3 Transfer Acceleration** |
| Guaranteed high disk IOPS on one instance | **EBS Provisioned IOPS** |
| Bulk-export DynamoDB without hurting prod | **DynamoDB export to S3** |
| Serverless Spark ETL / merge sources + catalog | **AWS Glue** |
| **No-code** visual data prep / profiling | **AWS Glue DataBrew** |
| Visual ML feature prep → export to Feature Store/Pipeline | **SageMaker Data Wrangler** |
| Lightweight per-record streaming transform | **AWS Lambda** |
| Label a huge **non-sensitive** dataset cheaply | **Ground Truth + Mechanical Turk** |
| Label **PII/PHI** data | **Ground Truth + private workforce** (never Mechanical Turk) |
| Cut labeling cost, model labels easy items | **Ground Truth automated data labeling** |
| Reuse features + training/serving consistency | **SageMaker Feature Store** (online + offline/Parquet) |
| Group under-represented (fewer rows) | **Class Imbalance (CI)** |
| Group labeled positive at different rates | **DPL** |
| Fix severe class imbalance | **Resample / synthetic data (SMOTE)**; evaluate with **F1/AUC** |
| Discover/classify PII in S3 | **Amazon Macie** |
| Health data / HIPAA | **PHI** controls (encrypt, private labeling) |
| Data must stay in-country | **Data residency** → choose Region, restrict replication |
| Automated quality rules in a Glue pipeline | **AWS Glue Data Quality (DQDL)** |
| Detect data leakage during ML prep | **Data Wrangler Data Quality & Insights report** |
| Detect bias / explain predictions | **SageMaker Clarify** |
| Right-skewed numeric feature | **Log transform** |
| Nominal categorical, low cardinality | **One-hot encoding** |
| Ordinal categorical / tree model | **Label encoding** |
| Start training fast on huge S3 data, POSIX | **Fast File mode** |

**Five reflexes to carry into the exam:**
1. **Columnar (Parquet/ORC) = analytics/scan-few-columns; Row (Avro/CSV/JSON) = write/stream whole records; RecordIO = built-in algos.**
2. **Firehose = no-code land-to-S3; Data Streams = replay/order; MSK = Kafka; Flink = process the stream.**
3. **Mechanical Turk ⇒ NO PII.** Sensitive data ⇒ private workforce.
4. **CI = representation imbalance; DPL = label-rate imbalance.** Both are **pre-training, model-agnostic, from Clarify.**
5. **Fast File/Pipe = stream from S3; EFS/FSx = mount a file system; File = copy-then-train.**

---

---

## Glossary

| Term | Simple explanation | Purpose |
|---|---|---|
| MLA-C01 | The AWS Certified Machine Learning Engineer – Associate exam code | Identifies the certification exam this chapter prepares you for |
| Domain 1 | The first scored section of MLA-C01, covering data preparation (28% of the exam) | Anchors all data-prep topics under a single exam weight |
| Data pipeline | A series of steps that move and transform data from source to a model-ready form | Structures the ingestion → transformation → integrity workflow |
| Ingest | The act of pulling raw data from a source and loading it into a storage system | First gate before any transformation or training |
| Feature engineering | Reshaping raw data columns into numeric signals a model can learn from effectively | Directly improves model accuracy without changing the algorithm |
| Integrity (data) | Ensuring data is fair, free of illegal identifiers, and high quality before training | Prevents biased or legally problematic models |
| CSV | Comma-Separated Values; plain text table, one row per line, no schema | Simple row-based format, human-readable but no type enforcement |
| JSON / JSON Lines | JavaScript Object Notation; semi-structured, supports nesting | Flexible row-based format used by APIs and streaming events |
| Apache Avro | Binary row-based format with an embedded schema supporting schema evolution | Best for Kafka/streaming pipelines where schema changes over time |
| Apache Parquet | Columnar binary format with schema and efficient compression | Default format for analytics and ML feature stores queried by Athena/Glue |
| Apache ORC | Optimized Row Columnar; columnar format common in Hive ecosystems | Columnar alternative to Parquet, especially for Hive-based workloads |
| RecordIO-protobuf | Binary row-based format preferred by SageMaker built-in algorithms | Enables efficient Pipe mode streaming, shrinking pre-training download time |
| Columnar storage | Stores each column's values together rather than each row together | Allows analytic queries to skip irrelevant columns, reducing cost |
| Row storage | Stores each complete record together | Efficient for reading or writing one full record at a time |
| Schema | A formal definition of a dataset's columns, types, and constraints | Lets systems catch bad records at read time instead of silently corrupting training |
| Schema evolution | Ability to add or change fields in a data format without breaking existing consumers | Essential for long-lived streaming pipelines |
| Amazon S3 | AWS object store with 11-nines durability and tiered pricing | Default data lake for raw data, features, and model artifacts |
| Amazon EFS | Elastic File System; managed NFS that multiple instances can mount simultaneously | Lets many training jobs or notebooks share the same files without copying |
| Amazon FSx for Lustre | High-performance parallel file system with sub-millisecond latency | Fastest I/O option for large-scale distributed training jobs |
| Amazon FSx for NetApp ONTAP | Managed NetApp NAS supporting NFS, SMB, iSCSI, and S3 API | Enterprise features (snapshots, dedup) combined with S3-API access for SageMaker |
| Amazon EBS | Elastic Block Store; block volume attached to a single EC2 or notebook instance | Provides configurable IOPS for single-instance workloads |
| Kinesis Data Streams | Sharded, ordered, replayable streaming service with up to 365-day retention | Real-time ingest for sub-second, ordered, multi-consumer pipelines |
| Kinesis Data Firehose | Zero-code delivery stream that buffers and lands data in S3, Redshift, or OpenSearch | Easiest way to continuously load streaming data into a data lake |
| Managed Service for Apache Flink | AWS-managed Apache Flink for real-time windowed stream processing | Runs SQL or code-based transformations on live data streams |
| Amazon MSK | Managed Streaming for Apache Kafka; fully managed Kafka service | Used when teams need Kafka's ecosystem or require cross-cloud portability |
| Shard | A unit of throughput in Kinesis Data Streams; each shard handles ordered records | Determines throughput capacity and ordering guarantees |
| S3 Transfer Acceleration | Speeds up S3 uploads/downloads over long distances via CloudFront edge nodes | Reduces latency when producers and the S3 bucket are far apart geographically |
| EBS Provisioned IOPS | A guaranteed IOPS level on an EBS block volume (io1/io2) | Ensures consistent high-speed disk access for I/O-intensive single-instance jobs |
| DynamoDB export to S3 | Bulk export of a DynamoDB table directly to S3 without consuming read capacity | Lets ML teams extract training data without impacting production database performance |
| SageMaker Data Wrangler | Visual, ML-aware data preparation tool inside SageMaker Studio | End-to-end feature prep with 300+ transforms, quality insights, and export to Pipelines/Feature Store |
| SageMaker Feature Store | Managed repository for reusable, versioned ML features with online and offline stores | Eliminates training/serving skew by storing features consistently for both real-time inference and model training |
| Online store (Feature Store) | Low-latency, single-record lookup layer of Feature Store | Serves features to real-time inference endpoints with millisecond latency |
| Offline store (Feature Store) | Historical feature data stored as Parquet in S3 | Used for model training and batch inference; supports Glue and Iceberg table formats |
| PutRecord | The Feature Store streaming API call that writes a single record synchronously | Enables real-time feature ingestion for online serving |
| Feature group | The schema and table definition in Feature Store, requiring a record identifier and event-time column | Organizes features into a logical, queryable unit |
| AWS Glue | Serverless Spark ETL service with a Data Catalog for schema management | Merges, transforms, and catalogs data from multiple sources without managing servers |
| Glue Crawler | Glue component that scans data sources and infers schemas into the Data Catalog | Automates schema discovery, avoiding hand-written table definitions |
| Glue Data Catalog | Central metadata repository for tables, schemas, and partitions across AWS | Makes data discoverable by Athena, SageMaker, and other services |
| Glue Studio | Visual authoring interface for creating and monitoring Glue ETL jobs | Allows low-code pipeline authoring for non-expert users |
| AWS Glue DataBrew | No-code visual data preparation service with 250+ pre-built transforms | Lets analysts clean, profile, and transform data without writing code |
| DQDL | Data Quality Definition Language; rule syntax for AWS Glue Data Quality checks | Specifies completeness, uniqueness, freshness, and other rules for automated quality gates |
| AWS Glue Data Quality | Rule-based in-pipeline data quality checking service | Automatically validates datasets against defined rules and blocks bad data from flowing downstream |
| Apache Spark | Distributed computing framework for large-scale data processing | Handles very large joins, aggregations, and transforms across many nodes |
| Amazon EMR | Elastic MapReduce; managed Hadoop/Spark cluster service | Runs full-scale Spark or Hadoop workloads with complete control over the cluster |
| pandas | Python data manipulation library for in-memory tabular data | Suitable for small, ad-hoc data merging or exploration in notebooks |
| Imputation | Filling in missing values using statistical estimates (mean, median, mode, KNN) | Preserves rows that would otherwise be dropped, maintaining dataset size |
| Outlier | A data point that falls far outside the normal distribution of its column | Can distort model training if not detected and treated |
| IQR | Interquartile Range; the spread between the 25th and 75th percentile | Used to detect outliers as points beyond 1.5×IQR from the quartiles |
| Z-score | Number of standard deviations a value is from the mean | Detects outliers as values with absolute z-score above a threshold (e.g., 3) |
| Deduplication | Removing duplicate rows from a dataset | Prevents over-weighting repeated examples and data leakage |
| Normalization | Rescaling feature values to [0, 1] | Required for distance-based and neural-network algorithms sensitive to scale |
| Standardization | Transforming a feature to have mean 0 and standard deviation 1 | Preferred by linear models and PCA where units differ across features |
| Log transformation | Applying log(x) to compress a right-skewed column toward a normal distribution | Reduces the influence of very large values on model training |
| Binning / bucketing | Converting a continuous numeric feature into discrete categorical buckets | Captures non-linear thresholds and reduces noise from minor variations |
| Feature splitting | Breaking one compound field into multiple separate fields | Extracts more signal from structured strings such as dates or addresses |
| One-hot encoding | Creating one binary column per category value | Represents nominal categories as numbers without implying false ordering |
| Label encoding | Mapping each category to an integer | Suitable for ordinal categories or tree-based models |
| Binary encoding | Converting a category to its integer representation then to binary digits | More space-efficient than one-hot for medium-to-high cardinality features |
| Tokenization | Splitting text into tokens or subwords and mapping them to integer IDs | Required preprocessing step for feeding text to NLP and LLM models |
| Cardinality | The number of unique values in a categorical column | High cardinality makes one-hot encoding impractical; drives choice of alternative encoding |
| SageMaker Ground Truth | AWS managed data labeling service with workforce options and automated labeling | Generates labeled training data at scale, reducing cost via machine-assisted labeling |
| Mechanical Turk | Amazon's public crowd-sourced workforce for annotation tasks | Provides large-scale labeling for non-sensitive public data |
| Private workforce | A labeling workforce made up of your own employees or a vendor | Required for PII/PHI or confidential data that must not be shared publicly |
| Automated data labeling | Ground Truth feature where a model labels easy examples so humans handle only hard ones | Reduces labeling cost by up to ~70% by minimizing human review |
| PII | Personally Identifiable Information; data that can identify an individual (name, SSN, email) | Triggers legal obligations to protect, anonymize, or mask before sharing or labeling |
| PHI | Protected Health Information; health data regulated under HIPAA | Requires stricter controls including encryption, private labeling workforce, and access restriction |
| HIPAA | U.S. Health Insurance Portability and Accountability Act | Compliance regime governing how health data must be stored, processed, and accessed |
| Amazon Macie | ML-powered service that discovers and classifies sensitive data in S3 | Automatically detects PII and other sensitive content to trigger protective controls |
| AWS KMS | Key Management Service; creates and manages encryption keys | Used to encrypt S3 buckets, EBS volumes, SageMaker model artifacts, and notebooks at rest |
| SSE-KMS | Server-Side Encryption using AWS KMS-managed keys | Protects S3 objects at rest with customer-controlled encryption keys |
| Anonymization | Irreversibly removing or transforming identifiers so re-identification is impossible | Meets legal requirements for sharing datasets without exposing personal details |
| Data residency | Requirement that data must remain within a specific geographic region or country | Drives selection of AWS Region and disables cross-region replication |
| SageMaker Clarify | AWS tool that computes pre-training bias metrics, post-training bias metrics, and SHAP explainability | Detects unfairness and explains model decisions throughout the ML lifecycle |
| Class Imbalance (CI) | Pre-training bias metric measuring under-representation of a demographic group by sample count | Signals when one group has too few training examples for the model to learn from fairly |
| DPL | Difference in Proportions of Labels; measures whether groups receive positive labels at different rates | Reveals outcome bias in the training data before any model is trained |
| Facet | A sensitive attribute or demographic group used in bias analysis (e.g., gender, age) | Defines which subgroup comparisons are made when computing bias metrics |
| Selection bias | Bias that occurs when the training sample does not represent the true population | Causes the model to underperform on groups that were under-sampled during data collection |
| Measurement bias | Bias from inconsistent data collection instruments or labeling practices | Introduces systematic errors that skew model learning |
| SMOTE | Synthetic Minority Over-sampling Technique; generates new synthetic minority-class examples | Addresses class imbalance without simply duplicating existing rows |
| Oversampling | Duplicating or generating minority-class rows to balance class distribution | Gives the model more minority-class signal at the cost of potential overfitting |
| Undersampling | Removing majority-class rows to balance class distribution | Simpler than oversampling but discards potentially useful data |
| F1 score | Harmonic mean of precision and recall | Preferred metric when class imbalance makes raw accuracy misleading |
| AUC | Area Under the ROC Curve; measures a classifier's ability across all thresholds | Robust to class imbalance; used to compare classifiers independent of threshold |
| Train/validation/test split | Partitioning data into sets for training, tuning, and final unbiased evaluation | Ensures the model is tested on data it has never seen during training or hyperparameter tuning |
| Stratified split | Split that preserves each class's proportion in every partition | Essential for imbalanced datasets to prevent a split from having no minority-class examples |
| Shuffling | Randomly reordering rows before splitting | Removes any ordering artifacts that could leak patterns into the model |
| Data augmentation | Synthetically expanding a dataset (e.g., flipping images, paraphrasing text) | Improves robustness and helps models generalize from small or imbalanced datasets |
| File mode | SageMaker training input mode that downloads the full dataset to local disk before starting | Simplest input mode; suitable when the dataset fits on the training instance's disk |
| Fast File mode | SageMaker training input mode that exposes S3 as a POSIX file system and streams objects on demand | Allows training to start immediately without downloading the full dataset |
| Pipe mode | SageMaker training input mode that streams data as a Unix pipe (typically RecordIO) | Minimizes local disk usage for very large sequential datasets |
| Apache Iceberg | Open table format supporting time-travel and schema evolution | Supported as an offline Feature Store table format alongside the default AWS Glue format |
| Glue DataBrew masking | Glue DataBrew transform that replaces sensitive values with redacted placeholders | Enables sharing datasets without exposing PII or other sensitive column values |
| Amazon Comprehend | AWS NLP service that detects entities, sentiment, key phrases, and PII in text | Can identify PII inside unstructured text fields before data is shared or labeled |

---

## References <a name="references"></a>

- SageMaker – Choosing an input mode and storage unit: https://docs.aws.amazon.com/sagemaker/latest/dg/model-access-training-data-best-practices.html
- SageMaker – Setting up training jobs to access datasets (S3/EFS/FSx): https://docs.aws.amazon.com/sagemaker/latest/dg/model-access-training-data.html
- SageMaker – Common data formats for built-in algorithms (RecordIO-protobuf): https://docs.aws.amazon.com/sagemaker/latest/dg/sagemaker-algo-common-data-formats.html
- SageMaker Data Wrangler (product): https://aws.amazon.com/sagemaker/ai/data-wrangler/
- SageMaker Data Wrangler – Transform Data: https://docs.aws.amazon.com/sagemaker/latest/dg/data-wrangler-transform.html
- SageMaker Feature Store: https://docs.aws.amazon.com/sagemaker/latest/dg/feature-store.html
- Feature Store – Data sources and ingestion: https://docs.aws.amazon.com/sagemaker/latest/dg/feature-store-ingest-data.html
- Feature Store – Offline store data format (Parquet, Glue/Iceberg): https://docs.aws.amazon.com/sagemaker/latest/dg/feature-store-offline.html
- SageMaker Clarify – Pre-training bias metrics: https://docs.aws.amazon.com/sagemaker/latest/dg/clarify-measure-data-bias.html
- SageMaker Clarify – Class Imbalance (CI): https://docs.aws.amazon.com/sagemaker/latest/dg/clarify-bias-metric-class-imbalance.html
- SageMaker Clarify – Run processing jobs for bias & explainability: https://docs.aws.amazon.com/sagemaker/latest/dg/clarify-processing-job-run.html
- SageMaker Ground Truth – Data labeling: https://docs.aws.amazon.com/sagemaker/latest/dg/sms.html
- SageMaker Ground Truth – Mechanical Turk workforce (PII requirement): https://docs.aws.amazon.com/sagemaker/latest/dg/sms-workforce-management-public.html
- SageMaker Ground Truth – Enhanced/automated data labeling: https://docs.aws.amazon.com/sagemaker/latest/dg/sms-data-labeling.html
- Amazon FSx for NetApp ONTAP: https://aws.amazon.com/fsx/netapp-ontap/
- AWS Blog – Choose the best data source for your SageMaker training job: https://aws.amazon.com/blogs/machine-learning/choose-the-best-data-source-for-your-amazon-sagemaker-training-job/
- MLA-C01 exam guide: https://docs.aws.amazon.com/aws-certification/latest/examguides/machine-learning-engineer-associate-01.html
