# Apache Spark Architecture

Apache Spark is the distributed engine that powers most large-scale data processing today — including **AWS Glue** (serverless Spark), Databricks, and EMR. For Data Science and Data Engineering interviews, you're expected to explain *how* Spark turns a few lines of code into work spread across a cluster, *why* it's fast, and *where* the costs hide. This page builds that intuition in plain English with visuals, then closes with the interview questions that come up again and again.

## Table of Contents

- [The 60-Second Mental Model](#the-60-second-mental-model)
- [The Players — Driver, Cluster Manager, Executors](#the-players--driver-cluster-manager-executors)
- [RDDs, DataFrames, and Partitions](#rdds-dataframes-and-partitions)
- [Lazy Evaluation and the DAG](#lazy-evaluation-and-the-dag)
- [Jobs → Stages → Tasks](#jobs--stages--tasks)
- [Narrow vs Wide Dependencies (and Why Shuffles Hurt)](#narrow-vs-wide-dependencies-and-why-shuffles-hurt)
- [The Shuffle, Step by Step](#the-shuffle-step-by-step)
- [Fault Tolerance via Lineage](#fault-tolerance-via-lineage)
- [Catalyst, Tungsten, and AQE](#catalyst-tungsten-and-aqe)
- [Why Spark Is Fast (vs Hadoop MapReduce)](#why-spark-is-fast-vs-hadoop-mapreduce)
- [Performance Levers That Matter in Interviews](#performance-levers-that-matter-in-interviews)
- [Spark in the Cloud — AWS Glue, EMR, Databricks](#spark-in-the-cloud--aws-glue-emr-databricks)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## The 60-Second Mental Model

In plain English: **Spark is a driver (the brain) that turns your lazy transformations into a plan (a DAG), splits that plan into shuffle-bounded stages of per-partition tasks, and runs those tasks across executors (the muscle) — keeping data in memory and rebuilding lost pieces from lineage.**

Four ideas carry the whole architecture:

1. **Split the data** into partitions so many machines work at once.
2. **Be lazy** — record the plan, don't run it, until an action forces the issue (so the whole plan can be optimized first).
3. **Shuffles are the expensive part** — moving data across the network between machines — so the entire performance game is minimizing and taming them.
4. **Remember how you built everything** (lineage) so a dead machine just means *recompute that slice*, not *start over*.

---

## The Players — Driver, Cluster Manager, Executors

```
        YOUR PROGRAM  =  the DRIVER
   ┌────────────────────────────────────┐
   │  SparkSession / SparkContext       │   the "brain":
   │   • builds the DAG (the plan)      │   - turns your code into a plan
   │   • splits it into stages & tasks  │   - schedules & tracks every task
   │   • collects results               │   (ONE per application)
   └──────────────┬─────────────────────┘
                  │ "give me executors"
                  ▼
        ┌───────────────────────┐
        │   CLUSTER MANAGER      │   YARN / Kubernetes / Standalone
        └──────────┬────────────┘   (allocates CPU + memory)
                   │ launches
       ┌───────────┼───────────────┐
       ▼           ▼               ▼
  ┌─────────┐ ┌─────────┐    ┌─────────┐
  │EXECUTOR │ │EXECUTOR │    │EXECUTOR │   distributed JVM processes
  │ cores:  │ │ cores:  │    │ cores:  │   on worker nodes:
  │ [t][t]  │ │ [t][t]  │    │ [t][t]  │   • RUN the tasks (compute)
  │ cache   │ │ cache   │    │ cache   │   • hold data in memory/disk (cache)
  └─────────┘ └─────────┘    └─────────┘
   ↑ one TASK processes one PARTITION of data ↑
```

| Component | What it is | Key responsibility |
|-----------|-----------|--------------------|
| **Driver** | The process running your `main()` / `SparkSession` | Builds the DAG, splits it into stages/tasks, schedules them on executors, tracks metadata, collects results. **One per application.** |
| **Cluster Manager** | Resource broker (YARN, Kubernetes, or Spark Standalone) | Allocates CPU + memory; launches executors when the driver asks. On **AWS Glue this is fully managed** — "serverless." |
| **Executor** | A distributed JVM process on a worker node | Runs the tasks (compute) and caches data in memory/disk. Has multiple **cores** = task slots. Reports results back to the driver. |
| **Task** | The smallest unit of work | Processes **exactly one partition**. One task = one core. |

> **The parallelism equation:** *parallelism ≈ number of partitions*, capped by *total executor cores*. If you have 200 partitions but only 50 cores, they run in ~4 waves.

---

## RDDs, DataFrames, and Partitions

- **RDD (Resilient Distributed Dataset)** — the low-level abstraction: an **immutable, partitioned, distributed** collection of objects with **lineage** for fault tolerance. You rarely use it directly today, but it's what everything compiles down to.
- **DataFrame / Dataset** — the high-level, table-like API (rows + named columns). It's optimized by the **Catalyst** optimizer and executed by the **Tungsten** engine. **Prefer DataFrames** — you get the optimizer for free; raw RDDs bypass it.
- **Partition** — a horizontal slice of the data. Data is split into partitions; **each partition is processed by one task on one core.** Partitions are the atom of parallelism.

```
     A DataFrame of 12M rows, split into 4 partitions:
     ┌────────────┬────────────┬────────────┬────────────┐
     │ Partition0 │ Partition1 │ Partition2 │ Partition3 │
     │  3M rows   │  3M rows   │  3M rows   │  3M rows   │
     └─────┬──────┴─────┬──────┴─────┬──────┴─────┬──────┘
           ▼            ▼            ▼            ▼
        Task 0       Task 1       Task 2       Task 3     ← run in parallel
```

---

## Lazy Evaluation and the DAG

Spark operations come in two flavors:

| | Examples | Behavior |
|---|----------|----------|
| **Transformations** (lazy) | `select`, `filter`, `map`, `join`, `groupBy`, `withColumn` | **Don't run.** They just append to a plan — a **DAG** (Directed Acyclic Graph). |
| **Actions** (eager) | `count`, `collect`, `show`, `write`, `save`, `first` | **Trigger execution** of everything the DAG needs to produce the result. |

**Why laziness matters (in plain English):** because Spark waits until an action, it sees the *whole* recipe before cooking. That lets it optimize — e.g., **push a filter down** so it reads fewer rows, **prune unused columns**, or **combine steps** — instead of blindly executing each line as written.

```python
df = spark.read.parquet("s3://.../events")   # lazy
hot = df.filter(df.country == "US")          # lazy  — nothing has run yet
agg = hot.groupBy("day").count()             # lazy  — still just building the DAG
agg.write.parquet("s3://.../out")            # ACTION — NOW Spark plans & runs it
```

---

## Jobs → Stages → Tasks

When an action fires, Spark expands the DAG into a concrete hierarchy:

```
  transformations (lazy)  ──►  DAG (the logical plan)
  action                  ──►  JOB          (one per action)
  Job                     ──►  STAGES        (split at SHUFFLE boundaries)
  Stage                   ──►  TASKS         (one task per partition)
  Tasks                   ──►  run on EXECUTORS
```

- **Job** — one per action.
- **Stage** — a group of transformations that can run **without a shuffle**. A new stage begins wherever a shuffle is required. Narrow operations get **pipelined** together into a single stage (e.g., `filter` → `map` → `filter` fuse into one pass over the data).
- **Task** — one per partition within a stage; the unit actually shipped to an executor core.

---

## Narrow vs Wide Dependencies (and Why Shuffles Hurt)

This is the single most-tested Spark concept. It's about how a child partition depends on parent partitions.

```
  NARROW (no shuffle, pipelined):        WIDE (shuffle → new stage):
    P1 ─► P1'                              P1 ─┐ ┌─► P1'
    P2 ─► P2'    map, filter,              P2 ─┼─┼─► P2'   groupBy, join,
    P3 ─► P3'    select, union             P3 ─┘ └─► P3'   distinct, repartition
    each parent → ONE child                each parent feeds MANY children;
    stays on the same machine              data crosses the NETWORK
```

- **Narrow dependency** — each parent partition feeds **one** child partition. No data movement; Spark **pipelines** these within a stage. Cheap.
- **Wide dependency** — a child partition needs data from **many** parents, so records must be **redistributed by key across the network**. This is a **shuffle**, and it forces a **stage boundary**.

**Why shuffles are the enemy:** a shuffle writes intermediate data to local disk, sends it over the network, and re-reads it — burning **disk I/O + network + serialization**. It's usually the biggest cost in a Spark job. The whole performance craft is *removing shuffles or making them cheaper*.

---

## The Shuffle, Step by Step

```
  STAGE N (map side)                         STAGE N+1 (reduce side)
  ┌───────────┐                              ┌───────────┐
  │ Executor A│  1. compute + partition      │ Executor A│
  │           │──► write shuffle files ──┐   │  3. FETCH shuffle files
  │           │     to LOCAL DISK        │   │     across the network
  └───────────┘                          │   │  4. deserialize + run logic
  ┌───────────┐                          ├──►│  (e.g. sum per key)
  │ Executor B│──► write shuffle files ──┘   └───────────┘
  └───────────┘     to LOCAL DISK
```

1. **Shuffle write** — each map task partitions its output by the target key and writes those partitions to **local disk**.
2. **(barrier)** — the next stage cannot start a task until the data it needs exists.
3. **Fetch** — reduce-side tasks pull the relevant shuffle blocks from every executor over the network.
4. **Deserialize + execute** — run the aggregation/join logic; optionally cache or write out.

That disk-write → network-fetch → disk-read round trip is exactly why you minimize shuffles.

---

## Fault Tolerance via Lineage

Spark does **not** replicate data across machines to survive failures. Instead, every RDD/DataFrame remembers **how it was computed** — its **lineage** (the chain of transformations from the source). If an executor dies and a partition is lost, Spark **recomputes just that partition** by replaying its lineage from the last available data.

```
  source ──filter──► A ──map──► B ──join──► C
                                     ▲
        lose a partition of C?  ─────┘  recompute ONLY that slice
        from B (and A if needed) — not the whole dataset
```

> **Checkpointing** truncates very long lineages (common in iterative ML) by persisting to reliable storage, so recovery doesn't have to replay hundreds of steps.

---

## Catalyst, Tungsten, and AQE

These three are how DataFrame code becomes fast machine work — and they're favorite senior-level interview probes.

- **Catalyst optimizer** — the query optimizer for DataFrames/SQL. It runs in four phases: **Analysis** (resolve column/table references) → **Logical Optimization** (rule-based rewrites like predicate pushdown, constant folding, column pruning) → **Physical Planning** (generate several physical plans, pick the cheapest by cost, e.g. broadcast vs sort-merge join) → **Code Generation** (emit JVM bytecode).
- **Tungsten** — the execution engine that makes each task CPU/memory-efficient: **off-heap memory management** (bypasses JVM garbage-collection overhead) and **whole-stage code generation** (collapses a chain of operators into one tight compiled function instead of many virtual calls).
- **AQE (Adaptive Query Execution, Spark 3.0+)** — re-optimizes **at runtime** using real shuffle statistics that static planning can't know in advance: **coalesce** small post-shuffle partitions, **switch** a sort-merge join to a broadcast join when one side turns out small, and **split skewed partitions** so one giant key doesn't create a straggler.

```
  DataFrame / SQL
      │  Catalyst
      ▼
  Logical Plan ─► Optimized Logical Plan ─► Physical Plan ─► RDDs (bytecode via Tungsten)
                                                  ▲
                                        AQE re-tunes here at runtime
```

---

## Why Spark Is Fast (vs Hadoop MapReduce)

| | Hadoop MapReduce | Apache Spark |
|---|------------------|--------------|
| Intermediate data | Written to **disk** after every step | Kept **in memory** where possible |
| Optimization | Step-by-step, no global view | **Lazy DAG** optimized as a whole (Catalyst) |
| Iterative workloads (ML) | Very slow (re-reads disk each pass) | Fast (data stays resident in RAM) |
| Execution | Generic JVM | **Tungsten** codegen + off-heap memory |

The headline: **in-memory computing + whole-plan optimization** is why Spark is often 10–100× faster than MapReduce on iterative and multi-stage jobs.

---

## Performance Levers That Matter in Interviews

- **Minimize shuffles.** Prefer `reduceByKey` over `groupByKey` (it combines locally first). Filter and select columns **early** to shrink data before wide operations.
- **Broadcast joins.** When one table is small (~≤10s of MB), broadcast it to every executor so the big table is **not shuffled** at all. AQE can do this automatically; you can also hint it.
- **Right-size partitions.** Too few → under-parallelized and huge tasks; too many → scheduling overhead. Repartition before heavy wide ops; coalesce before writing to avoid tiny files.
- **Fix data skew.** One hot key → one giant partition → a **straggler task** that stalls the whole stage. Mitigate with **salting** (spread the hot key) or AQE skew-join handling.
- **Cache / persist** data you reuse across multiple actions — otherwise Spark recomputes it from lineage each time.
- **Use columnar formats** (Parquet/ORC) + **partition pruning** so Spark reads only the files it needs.

---

## Spark in the Cloud — AWS Glue, EMR, Databricks

Most teams don't run raw Spark clusters anymore; they use managed Spark:

- **AWS Glue** — **serverless Spark**. You don't manage a cluster; AWS provisions executors ("DPUs") on demand, runs your job, and tears them down. You pay per second of processing. Great for ETL that runs periodically.
- **Amazon EMR** — managed Spark/Hadoop **clusters** you size and control; better for long-running or heavily customized workloads.
- **Databricks** — the commercial platform from Spark's creators; adds optimized runtime, notebooks, and Delta Lake.

**Interview tie-in (data-engineering ETL migration):** a common modern story is replacing a legacy sequential ETL tool with **AWS Glue (Spark)** feeding **Snowflake**, and seeing large processing-time and cost reductions. The *why* is this architecture: serverless Spark **auto-scales executors** (pay-per-use, no cluster ops), **partitioned parallelism** processes data across many cores at once, and **lazy DAG optimization with fewer shuffles** cuts the total work versus a one-row-at-a-time legacy pipeline. If asked "how does Glue scale your ETL?": *"Glue runs Spark — the driver builds a DAG, splits it into shuffle-bounded stages, and fans tasks across auto-provisioned executors, one task per partition; I tuned it by minimizing shuffles, broadcasting small dimension tables, and right-sizing partitions."*

---

## Interview Questions

### Q: Walk me through what happens when you call an action on a Spark DataFrame.
**Strong answer:**
The action triggers a **job**. The driver takes the lazily-built **DAG** and hands it to the DAG scheduler, which splits it into **stages** at shuffle boundaries. Each stage becomes a set of **tasks** — one per partition. The task scheduler ships tasks to **executors** (via the cluster manager), which process their partition, shuffle data between stages when needed, and return results or write output. Transformations up to that point were lazy; nothing actually computed until the action.

### Q: What's the difference between a narrow and a wide transformation?
**Strong answer:**
In a **narrow** transformation (map, filter, select) each parent partition contributes to exactly one child partition, so no data crosses the network and Spark pipelines them within a stage. In a **wide** transformation (groupBy, join, distinct, repartition) a child partition needs data from many parents, so records are redistributed by key — a **shuffle** — which forces a new stage and is the main performance cost.

### Q: Why is Spark lazy, and what does that buy you?
**Strong answer:**
Transformations only build a plan (DAG); nothing runs until an **action**. Because Spark sees the whole plan before executing, the **Catalyst** optimizer can rewrite it — push filters down to read fewer rows, prune unused columns, choose join strategies, and fuse operators — which it couldn't do if it executed each line eagerly.

### Q: How does Spark achieve fault tolerance?
**Strong answer:**
Through **lineage**, not replication. Every RDD/DataFrame records the sequence of transformations that produced it. If a partition is lost when an executor dies, Spark **recomputes only that partition** by replaying its lineage from the last available data. For very long lineages (iterative jobs) you **checkpoint** to reliable storage to cap recovery cost.

### Q: What is a shuffle and why is it expensive?
**Strong answer:**
A shuffle redistributes data across partitions/executors by key — required by wide operations. The map side **writes** partitioned output to local disk, and the reduce side **fetches** it over the network and deserializes it. That disk + network + serialization round trip, plus the stage barrier it introduces, makes it usually the biggest cost in a job. You minimize it with `reduceByKey`, broadcast joins, and early filtering.

### Q: How would you fix a job where one task runs far longer than the rest?
**Strong answer:**
That's classic **data skew** — one key (or a null-heavy key) lands a disproportionate amount of data in a single partition, creating a **straggler**. Fixes: **salt** the hot key (append a random suffix to spread it across partitions, then aggregate in two passes), enable **AQE skew-join handling** (Spark 3.0+ splits skewed partitions automatically), or broadcast the small side of the join to avoid the shuffle entirely.

### Q: Catalyst vs Tungsten — what's the difference?
**Strong answer:**
**Catalyst** is the **query optimizer** — it decides *what* plan to run (logical rewrites, join selection, predicate pushdown, physical plan choice). **Tungsten** is the **execution engine** — it makes *how* the plan runs efficient via off-heap memory management (less GC pressure) and whole-stage code generation (compiling operator chains into tight bytecode). Catalyst plans; Tungsten executes.

### Q: What does Adaptive Query Execution (AQE) add?
**Strong answer:**
AQE (Spark 3.0+) re-optimizes the plan **at runtime** using actual shuffle statistics that static planning can't know: it **coalesces** small post-shuffle partitions, **converts** sort-merge joins to broadcast joins when a side turns out small, and **splits skewed partitions**. It closes the gap between the optimizer's guesses and the data's real shape.

### Q: `reduceByKey` vs `groupByKey` — which do you prefer and why?
**Strong answer:**
`reduceByKey`, almost always. It performs a **map-side combine** — partially aggregating within each partition *before* the shuffle — so far less data crosses the network. `groupByKey` shuffles **all** values for every key and can blow up memory. Same result, much cheaper.

### Q: How does Spark decide how many tasks to run?
**Strong answer:**
One task **per partition** per stage. So the number of tasks is driven by the number of partitions — set by input splits (e.g., file/block count), `spark.sql.shuffle.partitions` after a shuffle (default 200), or explicit `repartition`/`coalesce`. Effective parallelism is `min(total partitions, total executor cores)`.

---

## References

- [Apache Spark Architecture 101: How Spark Works (2026) — Flexera](https://www.flexera.com/blog/finops/apache-spark-architecture/)
- [Apache Spark Architecture: A Guide for Data Practitioners — DataCamp](https://www.datacamp.com/blog/apache-spark-architecture)
- [Anatomy of a Spark Application — luminousmen](https://luminousmen.com/post/spark-anatomy-of-spark-application/)
- [Apache Spark Internals: Catalyst, Tungsten & Shuffle Performance — Datavidhya](https://datavidhya.com/learn/de-system-design/technology-deep-dives/spark-internals/)
- [Catalyst Optimizer vs Tungsten Engine — Siddharth Ghosh (Medium)](https://medium.com/@ghoshsiddharth25/spark-interview-series-catalyst-optimizer-and-tungsten-engine-c94cfc8cb41d)
- [70 Spark Interview Questions for Data Engineers (2026) — Datavidhya](https://datavidhya.com/blog/apache-spark-data-engineering-interview-questions/)
- [Apache Spark Documentation — Cluster Mode Overview](https://spark.apache.org/docs/latest/cluster-overview.html)

---

## Glossary

| Term | Simple explanation | Purpose |
|---|---|---|
| **Apache Spark** | An open-source distributed computing engine for large-scale data processing; keeps data in memory and optimizes whole-job plans | The standard engine for big-data ETL, ML pipelines, and analytics at scale |
| **Driver** | The single process that runs your application code, builds the execution plan, and coordinates all executors | The "brain" of a Spark job; one per application; failure here ends the job |
| **SparkSession / SparkContext** | The entry-point object your code uses to interact with Spark; creates the execution environment | Required first step in any Spark program; holds configuration and connects to the cluster manager |
| **Cluster Manager** | The resource broker (YARN, Kubernetes, or Spark Standalone) that allocates CPU and memory and launches executors | Decouples compute resource management from Spark logic; on AWS Glue it is fully managed |
| **Executor** | A distributed JVM process running on a worker node that runs tasks and caches data | The "muscle" of Spark; multiple cores per executor allow parallel task execution |
| **Task** | The smallest unit of work in Spark; processes exactly one partition on exactly one executor core | Parallelism = number of tasks that can run simultaneously = min(partitions, total executor cores) |
| **Partition** | A horizontal slice of a dataset; the atom of Spark parallelism | More partitions = more parallelism; too few causes under-utilization, too many causes scheduling overhead |
| **RDD (Resilient Distributed Dataset)** | Spark's low-level abstraction: an immutable, partitioned, distributed collection with lineage recorded for fault tolerance | The foundation everything compiles to; rarely used directly today but essential to understand |
| **DataFrame** | Spark's high-level, table-like API with named columns; optimized by Catalyst and executed by Tungsten | The recommended API for all new code; you get the optimizer for free, which raw RDDs bypass |
| **Lazy Evaluation** | Transformations only record a plan (the DAG) and do not execute until an action is called | Enables whole-plan optimization; Spark sees the full recipe before starting to cook |
| **Transformation** | A lazy operation on a DataFrame (filter, select, map, join, groupBy) that appends to the plan without executing | Building blocks of the DAG; cheap to declare, only paid for when an action triggers execution |
| **Action** | An eager operation (count, collect, write, show) that triggers the DAG to actually execute | The moment Spark starts running; the driver sends tasks to executors at this point |
| **DAG (Directed Acyclic Graph)** | The logical execution plan built from your transformations; a graph of steps with no cycles | The plan Spark optimizes before running; enables Catalyst to rewrite and reorder steps globally |
| **Job** | One unit of Spark work triggered by a single action; may contain multiple stages | Each call to count(), write(), etc. creates one job |
| **Stage** | A group of tasks that can run without a shuffle; a new stage begins wherever a shuffle is required | Stages are the scheduling units; all tasks in a stage run before the next stage starts |
| **Narrow Dependency** | A transformation where each parent partition feeds exactly one child partition (map, filter, select) | No data crosses the network; Spark pipelines multiple narrow transforms into one stage for free |
| **Wide Dependency** | A transformation where a child partition needs data from many parent partitions (groupBy, join, distinct) | Forces a shuffle and a new stage boundary; the primary source of cost in Spark jobs |
| **Shuffle** | The redistribution of data across executors by key required by wide transformations | The most expensive operation in Spark: writes to local disk, sends over network, reads back; minimizing shuffles is the main performance craft |
| **Shuffle Write** | The map-side step of a shuffle where each task partitions its output by target key and writes it to local disk | Creates the intermediate files the reduce side will fetch; disk I/O here is a significant cost |
| **Shuffle Fetch** | The reduce-side step where tasks pull shuffle blocks from every executor over the network | Network bandwidth and serialization overhead here; the reason shuffles hurt latency and cost |
| **Lineage** | The recorded chain of transformations that produced each RDD/DataFrame, stored in the DAG | Enables fault tolerance without data replication; lost partitions are recomputed by replaying lineage |
| **Checkpointing** | Persisting a DataFrame to reliable storage to truncate a very long lineage chain | Limits recovery cost for iterative jobs (like ML training) where replaying hundreds of steps would be slow |
| **Catalyst** | Spark's query optimizer for DataFrames and SQL; runs four phases: Analysis → Logical Optimization → Physical Planning → Code Generation | Turns your high-level DataFrame code into an efficient physical plan; this is why DataFrames beat raw RDDs |
| **Predicate Pushdown** | A Catalyst optimization that moves filter operations as early as possible in the plan to minimize rows processed | Reads less data from disk or from upstream stages; one of the highest-value automatic rewrites |
| **Column Pruning** | A Catalyst optimization that drops columns not needed by later steps | Reduces data size and serialization cost throughout the pipeline |
| **Tungsten** | Spark's execution engine that manages off-heap memory and compiles operator chains into tight JVM bytecode | Eliminates GC pauses and virtual-call overhead; the reason Spark is often 10–100× faster than MapReduce |
| **Off-Heap Memory Management** | Storing data in native memory outside the JVM heap, bypassing Java garbage collection | Avoids GC pauses that cause latency spikes on large JVM heaps |
| **Whole-Stage Code Generation** | Tungsten's technique of compiling a chain of operators into one fused JVM function | Eliminates per-row virtual method dispatch; the biggest single reason Spark is CPU-efficient |
| **AQE (Adaptive Query Execution)** | A Spark 3.0+ feature that re-optimizes the physical plan at runtime using actual shuffle statistics | Fixes the mismatch between the optimizer's estimates and real data shapes; coalesces tiny partitions, switches join strategies, splits skewed partitions |
| **Broadcast Join** | A join strategy where the small table is copied ("broadcast") to every executor so the large table never shuffles | Eliminates the shuffle on the large side; AQE can switch to this automatically when a side turns out small |
| **Sort-Merge Join** | A join strategy where both sides are shuffled and sorted by key, then merged | The fallback for large-large joins; more expensive but correct for any size; switched to broadcast join by AQE when possible |
| **Data Skew** | When one or a few keys contain far more rows than others, creating one overloaded task that stalls the whole stage | The most common Spark performance problem in production |
| **Straggler Task** | A single slow task that delays the completion of its entire stage because the stage must wait for all tasks | The visible symptom of data skew; one hot key = one giant partition = one straggler |
| **Salting** | Appending a random suffix to a hot key to spread its rows across multiple partitions before aggregating | The standard fix for data skew; requires a two-phase aggregation to undo the salt afterward |
| **Repartition** | Explicitly reshuffling data into a new number of partitions using a full shuffle | Used before heavy wide operations to ensure even data distribution; more expensive than coalesce |
| **Coalesce** | Reducing the number of partitions without a full shuffle by merging local partitions | Used before writing to avoid tiny output files; cheaper than repartition but can create uneven partitions |
| **Parquet** | A columnar binary file format designed for efficient analytical queries | The standard storage format for Spark; enables column pruning and partition pruning at read time |
| **ORC** | Optimized Row Columnar format; another columnar binary format similar to Parquet | Alternative to Parquet with good Hive ecosystem support; also enables Spark read-time optimizations |
| **Partition Pruning** | Spark reading only the file partitions (directories) needed to answer a query, skipping the rest | Dramatically reduces data read for time- or category-partitioned datasets |
| **reduceByKey** | An aggregation that performs a local map-side combine within each partition before shuffling | Shuffles far less data than groupByKey; the preferred aggregation for large datasets |
| **groupByKey** | An aggregation that shuffles all values for every key before grouping | Expensive and memory-intensive; almost always replaced by reduceByKey in production |
| **spark.sql.shuffle.partitions** | A Spark config setting controlling the number of output partitions after a shuffle (default 200) | Must be tuned to match your data volume; 200 is too many for small datasets, too few for very large ones |
| **AWS Glue** | Amazon's serverless Spark service; provisions and tears down executors automatically per job | Eliminates cluster management; billed per second of compute; ideal for periodic ETL jobs |
| **Amazon EMR** | Amazon's managed Spark/Hadoop cluster service; you control cluster size and configuration | Better than Glue for long-running or heavily customized workloads with predictable load |
| **Databricks** | The commercial platform from Spark's creators; adds an optimized runtime, Delta Lake, and collaborative notebooks | Offers the fastest Spark runtime (Photon) and best Delta Lake integration; the enterprise SaaS choice |
| **Delta Lake** | An open-source storage layer that adds ACID transactions, schema enforcement, and time travel to Parquet files on cloud storage | Enables reliable data lake patterns (upserts, deletes, schema evolution) without a traditional database |
| **DPU (Data Processing Unit)** | The billing unit for AWS Glue; each DPU provides a fixed amount of CPU and memory | Used to estimate and control Glue job cost |
| **YARN** | Yet Another Resource Negotiator; the cluster resource manager bundled with Hadoop | The traditional cluster manager for on-premises Spark deployments |
| **JVM (Java Virtual Machine)** | The runtime environment that executes Spark's Scala/Java code; also runs Python workers via Py4J | Spark's execution environment; JVM GC behavior is a key performance concern at scale |

---

*Previous: [OOP in Python](../21-python-and-coding/02-oop-in-python.md) | Up: [Guide Home](../README.md)*
