# Chapter 10: Batch Processing

## Core Concept

> "A batch processing system takes a large amount of input data, runs a job to process it, and produces output data."

**Why This Matters:** Batch processing is how we handle data that's too large to fit on one machine or process in real-time. Understanding MapReduce, joins, and modern alternatives (Spark) is essential for designing data pipelines and answering "how would you process petabytes of logs?" questions.

**Three Types of Systems:**
```
┌─────────────────────────────────────────────────────────────┐
│              Services vs Batch vs Stream                     │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   Services (Online):                                       │
│   • Request/response pattern                               │
│   • Low latency (milliseconds)                             │
│   • Example: Web servers, REST APIs, databases             │
│   • Use when: User waiting for response                    │
│                                                             │
│   Batch Processing (Offline):                              │
│   • Large input → process → large output                   │
│   • High throughput, high latency (minutes to hours)       │
│   • Example: MapReduce, Spark jobs, daily reports          │
│   • Use when: Processing historical data, not time-critical│
│                                                             │
│   Stream Processing (Near-realtime):                       │
│   • Continuous input → process → continuous output         │
│   • Low latency (seconds to minutes)                       │
│   • Example: Kafka Streams, Flink, Spark Streaming         │
│   • Use when: Need fresh data, can't wait for batch        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

> **Interview Tip:** When asked "batch or stream?", the answer is often "both." Use batch for backfilling, reprocessing, and complex analytics. Use stream for real-time dashboards and alerting. Many systems use Lambda or Kappa architecture to combine both.

---

## Unix Philosophy

### The Original "Big Data"

Before Hadoop, Unix pipelines were doing batch processing on single machines. Understanding this helps explain why MapReduce works the way it does.

```bash
# Classic Unix pipeline - batch processing in its simplest form
cat /var/log/nginx/access.log |
    awk '{print $7}' |           # Extract URL path (mapper)
    sort |                        # Sort alphabetically (shuffle)
    uniq -c |                     # Count unique occurrences (reducer)
    sort -rn |                    # Sort by count (descending)
    head -n 10                    # Top 10 pages
```

**Why this matters:** MapReduce is essentially this pattern, distributed across thousands of machines.

### Unix Design Principles

```
┌─────────────────────────────────────────────────────────────┐
│                Unix Philosophy                               │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   1. Each program does ONE thing well                      │
│      sort, grep, uniq, awk - focused, reusable tools       │
│                                                             │
│   2. Programs communicate via text streams                 │
│      stdin/stdout = universal interface                    │
│      Any program can connect to any other                  │
│                                                             │
│   3. Favor composability over monolithic tools             │
│      grep + sort + uniq > complex custom tool              │
│                                                             │
│   MapReduce adopted this: mappers and reducers are         │
│   simple functions connected by shuffle (like pipes)       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## MapReduce

### What It Is

**Simple definition:** A programming model for processing large datasets in parallel across a cluster. You write two functions: `map` (extract/transform) and `reduce` (aggregate). The framework handles distribution, parallelism, and fault tolerance.

**Why it was revolutionary:** Before MapReduce, distributed batch processing was ad-hoc. Google's 2004 paper showed how to make it simple and fault-tolerant. Hadoop made it open-source and accessible.

### The Model

```
┌─────────────────────────────────────────────────────────────┐
│                  MapReduce Flow                              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   Input         Map           Shuffle        Reduce        │
│   ─────         ───           ───────        ──────        │
│                                                             │
│   Block 1 ──► Mapper 1 ─┐                ┌─► Reducer 1 ──► Output 1
│                         │    Partition   │
│   Block 2 ──► Mapper 2 ─┼─── by key ────►├─► Reducer 2 ──► Output 2
│                         │    & Sort      │
│   Block 3 ──► Mapper 3 ─┘                └─► Reducer 3 ──► Output 3
│                                                             │
│   Map: (key, value) → list of (key, value)                 │
│   Reduce: (key, list of values) → (key, aggregated value)  │
│                                                             │
│   Shuffle: The "magic" part - groups all values by key,    │
│   sends them to the right reducer, and sorts.              │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Word Count Example

The "Hello World" of MapReduce:

```python
# Map function: for each document, emit (word, 1) for each word
def map(document_id, document_text):
    for word in document_text.split():
        emit(word.lower(), 1)

# Input: ("doc1", "Hello World Hello")
# Output: [("hello", 1), ("world", 1), ("hello", 1)]

# After shuffle: all values for same key grouped together
# ("hello", [1, 1]), ("world", [1])

# Reduce function: sum up all the 1s for each word
def reduce(word, counts):
    emit(word, sum(counts))

# Input: ("hello", [1, 1])
# Output: ("hello", 2)
```

**Key insight:** Mappers run in parallel (one per input block). Reducers run in parallel (one per key partition). The shuffle phase moves data between them.

### Why "Move Computation to Data"

```
┌─────────────────────────────────────────────────────────────┐
│            MapReduce + HDFS Architecture                     │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   ┌──────────────────────────────────────────────────┐     │
│   │                    HDFS                          │     │
│   │  ┌────────┐  ┌────────┐  ┌────────┐             │     │
│   │  │Block 1 │  │Block 2 │  │Block 3 │  (128MB each)│    │
│   │  │Node A  │  │Node B  │  │Node C  │             │     │
│   │  └────────┘  └────────┘  └────────┘             │     │
│   └──────────────────────────────────────────────────┘     │
│          │              │              │                    │
│          ▼              ▼              ▼                    │
│   ┌──────────┐  ┌──────────┐  ┌──────────┐                │
│   │ Mapper 1 │  │ Mapper 2 │  │ Mapper 3 │                │
│   │ (Node A) │  │ (Node B) │  │ (Node C) │                │
│   └──────────┘  └──────────┘  └──────────┘                │
│                                                             │
│   Mapper runs on SAME node as data it reads.               │
│   Avoids moving terabytes over network.                    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

> **Interview Tip:** "Move computation to data, not data to computation" is a key principle. Reading data over the network is slow; starting a process locally is fast. This is why HDFS stores data blocks on multiple nodes.

---

## Distributed Joins

**Why joins are hard in batch processing:** Unlike a database that has indexes, batch systems must process ALL data. Joining two large datasets requires moving data across the network—which is slow.

**Three main strategies:**
1. **Shuffle join (Sort-Merge):** Move data over network, group by key
2. **Broadcast join:** Send small dataset to all nodes
3. **Partition join:** Pre-partition both datasets by join key

### Sort-Merge Join (Reduce-Side / Shuffle Join)

**When to use:** Both datasets are large; neither fits in memory.

**How it works:** Both datasets are partitioned by join key and sent to reducers. Each reducer sees all records with the same key from both datasets.

```
┌─────────────────────────────────────────────────────────────┐
│                Sort-Merge Join                               │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   Users file:              Activity file:                   │
│   user_id,name             user_id,page_viewed             │
│   1,Alice                  1,/home                         │
│   2,Bob                    2,/products                     │
│                            1,/checkout                     │
│                                                             │
│   Map phase (tag each record with source):                 │
│   Users mapper: (1, ("user", "Alice"))                     │
│   Activity mapper: (1, ("activity", "/home"))              │
│                                                             │
│   After shuffle (grouped by user_id):                      │
│   Reducer for key=1 receives:                              │
│   [("user", "Alice"), ("activity", "/home"),               │
│    ("activity", "/checkout")]                              │
│                                                             │
│   Reduce: Join user with all their activities              │
│   Output: (1, "Alice", ["/home", "/checkout"])             │
│                                                             │
│   Cost: Full shuffle of BOTH datasets over network         │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Broadcast Hash Join (Map-Side)

**When to use:** One dataset is SMALL enough to fit in memory on each node (e.g., dimension tables, lookup tables).

**How it works:** Small dataset is loaded into a hash map on every mapper. Large dataset is processed locally—no shuffle needed!

```
┌─────────────────────────────────────────────────────────────┐
│               Broadcast Hash Join                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   Small dataset (users):   Large dataset (events):         │
│   ┌─────────────┐          ┌────────────────────┐         │
│   │ user_id:name│ ◄─load─  │ Block 1: events    │         │
│   │ 1:Alice     │   into   │ Block 2: events    │         │
│   │ 2:Bob       │  memory  │ Block 3: events    │         │
│   └─────────────┘          └────────────────────┘         │
│         │                           │                      │
│         │       Each mapper:        │                      │
│         ▼                           ▼                      │
│   ┌──────────────────────────────────────┐                │
│   │ 1. Load small table into HashMap     │                │
│   │ 2. For each event record:            │                │
│   │    - Look up user in HashMap (O(1))  │                │
│   │    - Emit joined record              │                │
│   └──────────────────────────────────────┘                │
│                                                             │
│   NO SHUFFLE NEEDED! 10-100x faster than shuffle join.     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

> **Interview Tip:** "How would you optimize a join?" First ask: "How big are the datasets?" If one fits in memory (< few GB), broadcast it. Spark does this automatically when it detects a small table (`spark.sql.autoBroadcastJoinThreshold`).

### Partition Hash Join (Co-located Join)

**When to use:** Both datasets are already partitioned by the join key (e.g., data lake tables bucketed by user_id).

**How it works:** Since both datasets use same partitioning, each partition only needs to join with its matching partition—no cross-partition data movement.

```
┌─────────────────────────────────────────────────────────────┐
│              Partition Hash Join                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   Both datasets pre-partitioned by user_id:                │
│                                                             │
│   Users partitions:        Events partitions:              │
│   ┌───────────────┐        ┌───────────────┐              │
│   │ Part 1: A-M   │◄──────►│ Part 1: A-M   │              │
│   │ Part 2: N-Z   │◄──────►│ Part 2: N-Z   │              │
│   └───────────────┘        └───────────────┘              │
│                                                             │
│   Mapper for partition 1 only reads:                       │
│   - Users partition 1 + Events partition 1                 │
│   - Join locally, no shuffle needed                        │
│                                                             │
│   Requires: Both datasets bucketed same way beforehand     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Join Strategy Summary

| Strategy | When to Use | Network Cost |
|----------|-------------|--------------|
| **Broadcast** | One side < few GB | None (small table copied) |
| **Partition** | Both pre-bucketed | None (local join) |
| **Shuffle** | Large + large | High (both shuffled) |

---

## Beyond MapReduce: Why Spark Won

### MapReduce Limitations

MapReduce was groundbreaking but has significant limitations:

```
1. Rigid two-stage model
   - What if you need 10 steps? Chain 10 MapReduce jobs!
   - Each job writes output to HDFS before next reads it
   - Example: Join → Filter → Aggregate = 3 jobs, 3 disk writes

2. No iteration support  
   - Machine learning needs many passes over data
   - Each iteration = read from disk, write to disk
   - PageRank, K-means become painfully slow

3. Manual optimization
   - Developer chooses join strategies
   - Developer sets number of reducers
   - Easy to get wrong, hard to tune
```

### Dataflow Engines (Spark, Flink)

**What changed:** Instead of forcing Map→Shuffle→Reduce, modern engines let you chain arbitrary operations. Intermediate results stay in memory, not disk.

```
┌─────────────────────────────────────────────────────────────┐
│           Dataflow Engine vs MapReduce                       │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   MapReduce (3 jobs for join→filter→aggregate):            │
│   [HDFS] → Map → Shuffle → Reduce → [HDFS]                 │
│                                        ↓                   │
│   [HDFS] → Map → Shuffle → Reduce → [HDFS]                 │
│                                        ↓                   │
│   [HDFS] → Map → Shuffle → Reduce → [Output]               │
│                                                             │
│   Spark (same pipeline, ONE job):                          │
│   [Input] ─► Read ─► Join ─► Filter ─► Aggregate ─► [Out]  │
│              │        │        │          │                 │
│              └────────┴────────┴──────────┘                 │
│              All in memory! Write to disk only at end.     │
│                                                             │
│   • Operators form a DAG (directed acyclic graph)          │
│   • Shuffle only when necessary (groupBy, join)            │
│   • Automatic query optimization (Catalyst)                │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Spark Example

```python
from pyspark.sql import SparkSession
from pyspark.sql.functions import col, count

spark = SparkSession.builder.appName("PageViews").getOrCreate()

# Load data (lazy - no execution yet)
events = spark.read.parquet("s3://data/events")
users = spark.read.parquet("s3://data/users")

# Build pipeline (still lazy - building DAG)
result = (
    events
    .join(users, events.user_id == users.id)  # Join
    .filter(col("country") == "US")           # Filter
    .groupBy("page")                          # Group
    .agg(count("*").alias("views"))           # Aggregate
    .orderBy(col("views").desc())             # Sort
    .limit(10)                                # Top 10
)

# Execute (triggers actual computation)
result.write.parquet("s3://output/top_pages")
```

**Key concept - Lazy Evaluation:** Spark builds a DAG of transformations but doesn't execute until you call an action (write, collect, count). This lets the optimizer see the entire pipeline and optimize it.

### Why Spark is 10-100x Faster

```
┌─────────────────────────────────────────────────────────────┐
│              Why Spark is Faster                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   1. In-memory processing                                  │
│      • Intermediate results cached in RAM                  │
│      • 10-100x faster for iterative algorithms (ML)        │
│                                                             │
│   2. DAG execution                                         │
│      • No forced Map-then-Reduce stages                    │
│      • Pipeline narrow operations (map, filter) together   │
│      • Only shuffle when necessary (groupBy, join)         │
│                                                             │
│   3. Query optimization (Catalyst)                         │
│      • Predicate pushdown (filter before join)             │
│      • Join reordering (small table first)                 │
│      • Broadcast join auto-detection                       │
│                                                             │
│   4. Columnar processing (Tungsten)                        │
│      • Process columns, not rows                           │
│      • Better CPU cache utilization                        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

> **Interview Tip:** "Why Spark over MapReduce?" Answer: (1) In-memory = no disk I/O between stages, (2) Flexible DAG = fewer jobs, (3) Automatic optimization = developer writes SQL, optimizer figures out best plan.

---

## Batch vs Stream: Architectures

### Lambda Architecture (Historical)

**The problem it solves:** Batch processing gives accurate results but is slow (hours). Stream processing is fast but may be approximate or miss late data.

**The solution:** Run both! Batch layer handles historical data (accurate), speed layer handles recent data (fast). Merge at query time.

```
┌─────────────────────────────────────────────────────────────┐
│               Lambda Architecture                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│                      ┌─────────────────┐                   │
│   Events ───────────►│  Batch Layer    │──► Batch Views    │
│      │               │  (daily Spark)  │    (accurate)     │
│      │               └─────────────────┘          │        │
│      │                                            │        │
│      │               ┌─────────────────┐          │        │
│      └──────────────►│  Speed Layer    │──► Real-time     │
│                      │  (Kafka/Flink)  │    Views (fast)   │
│                      └─────────────────┘          │        │
│                                                   ▼        │
│                      ┌─────────────────────────────────┐   │
│                      │       Serving Layer             │   │
│                      │ Query = batch_view + speed_view │   │
│                      └─────────────────────────────────┘   │
│                                                             │
│   Problem: Must maintain SAME logic in two codebases!      │
│   Batch in Spark, streaming in Flink = double the work.    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Kappa Architecture (Modern Approach)

**The insight:** If your streaming system can reprocess historical data (by replaying from Kafka), you don't need a separate batch layer.

```
┌─────────────────────────────────────────────────────────────┐
│               Kappa Architecture                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│                     ┌───────────────┐                      │
│   Events ──────────►│ Kafka (log)   │  Retains weeks/months│
│                     └───────┬───────┘  of data             │
│                             │                              │
│                             ▼                              │
│                     ┌───────────────┐                      │
│                     │ Stream Proc.  │──────► Views         │
│                     │ (Flink/Kafka) │                      │
│                     └───────────────┘                      │
│                                                             │
│   Need to reprocess? Replay from Kafka with new logic.     │
│   Single codebase for both real-time AND backfill!         │
│                                                             │
│   Trade-off: Kafka retention costs; replay can be slow     │
│   for very large historical datasets.                      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

> **Interview Tip:** When asked "batch or stream?", consider: (1) Latency requirements - can you wait hours? (2) Data volume - TB+ historical data? (3) Reprocessing needs - how often does logic change? Kappa is simpler but Lambda may be necessary for petabyte-scale historical analysis.

---

## Output Handling and Idempotency

### The Output Problem

**Why this matters:** Batch jobs can fail partway through. If a job fails after writing half its output, retrying may cause duplicates or inconsistencies.

```
┌─────────────────────────────────────────────────────────────┐
│            Batch Job Output Considerations                   │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   BAD: Write directly to database during job               │
│   • Job fails halfway = partial/corrupted results          │
│   • Retries cause duplicates                               │
│   • Other jobs see inconsistent state                      │
│                                                             │
│   GOOD: Write to new location, swap atomically             │
│   • Job writes to /output/2024-01-15/                     │
│   • On success: atomic rename/symlink to /output/latest   │
│   • Failure? Just delete incomplete output, retry          │
│                                                             │
│   ALSO GOOD: Idempotent writes with deterministic keys     │
│   • Same input always produces same output key             │
│   • Retry overwrites same keys = no duplicates             │
│   • Example: key = hash(input_record)                      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

> **Interview Tip:** Always mention idempotency when discussing batch jobs. "The job writes output with deterministic keys, so retries are safe—they just overwrite the same records."

---

## Modern Data Stack

### Ecosystem Overview

```
┌─────────────────────────────────────────────────────────────┐
│            Modern Data Processing Stack                      │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   Orchestration:    Airflow, Dagster, Prefect              │
│   (Schedule and monitor pipelines)                         │
│                                                             │
│   Processing:                                              │
│   ├── Batch: Spark, Trino/Presto, dbt                     │
│   └── Stream: Flink, Kafka Streams, Spark Structured Str. │
│                                                             │
│   Storage:                                                 │
│   ├── Object Store: S3, GCS, Azure Blob                   │
│   ├── Table Format: Delta Lake, Iceberg, Hudi             │
│   └── Warehouse: Snowflake, BigQuery, Redshift, Databricks│
│                                                             │
│   File Formats:                                            │
│   ├── Columnar: Parquet (most common), ORC                │
│   └── Row: Avro (for streaming), JSON (for humans)        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Why Parquet Matters

Parquet is columnar: stores all values of one column together, then next column.

**Benefits:**
- **Compression:** Similar values in same column = better compression (10x smaller)
- **Predicate pushdown:** `WHERE country='US'` only reads country column
- **Column pruning:** `SELECT name, email` doesn't read other columns

> **Interview Tip:** When designing data pipelines, always mention: "Store data in Parquet format for analytics—it's columnar, so aggregations only read the columns they need."

---

## Key Takeaways

1. **MapReduce** = simple model (map → shuffle → reduce) that scaled to petabytes, but is now largely replaced by Spark
2. **Move computation to data** = run processing on the node that has the data, not vice versa
3. **Join strategies matter** = broadcast small tables, shuffle large ones, pre-partition for frequent joins
4. **Spark beat MapReduce** = in-memory processing, flexible DAG, automatic optimization
5. **Idempotent outputs** = make reprocessing safe by using deterministic keys
6. **Columnar formats** (Parquet) = 10x compression, column pruning for analytics
7. **Kappa architecture** = unify batch and stream with a single codebase

---

## Quick Reference

| Need | Tool |
|------|------|
| Ad-hoc SQL on data lake | Trino / Presto / Spark SQL |
| Scheduled ETL pipelines | Airflow + Spark / dbt |
| Machine learning at scale | Spark MLlib / Ray / PyTorch distributed |
| Real-time + historical | Flink (unified batch/stream) |
| Data lake tables with ACID | Delta Lake / Iceberg |
| Simple analytics queries | BigQuery / Snowflake (serverless) |

---

## System Design Interview Tips

### When to Bring Up Batch Processing

- **"How would you build a recommendation system?"** → Batch job to compute user-item similarities, serve from cache
- **"Design a data pipeline for analytics"** → Events → Kafka → Spark → Parquet → Warehouse
- **"How do you handle late data?"** → Batch reprocessing or watermarks in streaming

### Trade-off Discussions

| Factor | Batch | Stream |
|--------|-------|--------|
| Latency | Hours | Seconds-minutes |
| Complexity | Simpler | More complex (exactly-once, late data) |
| Throughput | Very high | Lower (per-record overhead) |
| Reprocessing | Easy (just re-run) | Harder (need Kafka retention) |
| Cost | Lower (spot instances) | Higher (always running) |

---

## Common Interview Questions

1. **"Explain MapReduce"**
   - Map: Transform each record independently, emit (key, value) pairs
   - Shuffle: Group all values by key, send to reducer
   - Reduce: Aggregate values for each key
   - Framework handles parallelism, fault tolerance, scheduling

2. **"Why did Spark replace MapReduce?"**
   - In-memory processing (no disk I/O between stages)
   - Flexible DAG (not forced into Map→Reduce pattern)
   - Automatic query optimization (Catalyst)
   - Better API (DataFrames, SQL)

3. **"How would you optimize a slow batch job?"**
   - Check for data skew (one key has most data → overwhelms one reducer)
   - Use broadcast joins for small tables
   - Increase parallelism (more partitions)
   - Use columnar format (Parquet) with predicate pushdown
   - Cache intermediate results if reused

4. **"Batch or stream processing?"**
   - Batch: Historical analysis, ML training, reports, backfilling
   - Stream: Real-time dashboards, alerting, fraud detection
   - Often both (Lambda/Kappa architecture)

5. **"What's a shuffle and why is it expensive?"**
   - Shuffle = redistributing data across the cluster by key
   - Expensive because: network transfer, disk I/O for spilling, sorting
   - Minimize by: broadcast joins, pre-partitioning, avoiding unnecessary groupBy
