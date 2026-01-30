# Chapter 3: Storage and Retrieval

## Core Concept

> "A database needs to do two things: store data when given, and return it when asked."

**What it is:** This chapter covers how databases physically store data on disk and how they retrieve it efficiently.

**Why it matters for interviews:** Understanding storage engines helps you:
- Explain why you'd pick Cassandra vs PostgreSQL for a specific workload
- Debug performance issues (why are writes slow? why are reads slow?)
- Design systems that scale appropriately

> 💡 **Interview Tip:** When asked "what database would you use?", never just name a database. Explain the access patterns first, then match to a storage engine type.

---

## Two Families of Storage Engines

**What they are:** All databases fall into two camps based on how they organize data on disk.

```
┌─────────────────────────────────────────────────────────────┐
│                    Storage Engines                           │
├────────────────────────┬────────────────────────────────────┤
│     Log-Structured     │         Page-Oriented              │
│     (LSM-Trees)        │         (B-Trees)                  │
├────────────────────────┼────────────────────────────────────┤
│ • Append-only writes   │ • In-place updates                 │
│ • Compaction needed    │ • Fixed-size pages                 │
│ • Better write perf    │ • Better read perf                 │
├────────────────────────┼────────────────────────────────────┤
│ RocksDB, LevelDB,      │ PostgreSQL, MySQL,                 │
│ Cassandra, HBase       │ Oracle, SQL Server                 │
└────────────────────────┴────────────────────────────────────┘
```

**Why two families?** It's fundamentally about sequential vs random I/O. SSDs are fast, but sequential writes are still 10-100x faster than random writes. LSM-trees exploit this.

---

## Log-Structured Storage (LSM-Trees)

### The Simplest Database

**What it is:** A log is just an append-only file. The simplest database is literally two bash functions:

```bash
#!/bin/bash
# World's simplest key-value store
db_set() { echo "$1,$2" >> database; }
db_get() { grep "^$1," database | sed "s/^$1,//" | tail -n 1; }

# Usage:
db_set user123 '{"name":"Alice"}'
db_get user123  # Returns: {"name":"Alice"}
```

**Why this matters:** This is the foundation of ALL log-structured storage. Every LSM-tree database is a sophisticated version of this.

**The trade-off:**
- Writes: O(1) ✓ (just append to end of file - sequential I/O)
- Reads: O(n) ✗ (scan entire file to find key!)

**Solution:** Add an index to speed up reads without sacrificing write speed!

### Hash Index (In-Memory)

**What it is:** Keep a hashmap in memory that maps each key to its byte offset in the log file.

```
┌─────────────────────────────────────────────────────────────┐
│                    Hash Index                                │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   In-Memory HashMap          Log File on Disk              │
│   ┌───────────────┐          ┌─────────────────┐           │
│   │ key → offset  │          │ 0: user1,{...}  │           │
│   ├───────────────┤   ───►   │ 42: user2,{...} │           │
│   │ user1 → 0     │          │ 87: user1,{...} │           │
│   │ user2 → 42    │          │ 120: user3,{...}│           │
│   │ user3 → 120   │          └─────────────────┘           │
│   └───────────────┘                                        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Why this works:** Writes are still O(1) (append + update hashmap), but reads are now O(1) (lookup offset, seek to position).

**Used by:** Bitcask (Riak's default engine)

**When to use hash indexes:**
- ✅ High write throughput with frequent updates to same keys (like URL shorteners, video view counts)
- ✅ Workloads where all keys fit in RAM

**Limitations:**
- ❌ All keys must fit in memory (can't handle billions of small keys)
- ❌ Range queries inefficient (can't ask "all users from A-M")

> 💡 **Interview Tip:** When someone asks about key-value stores, mention that the choice of index structure determines the trade-offs. Hash = fast point lookups but no range queries.

### SSTables and LSM-Trees

**What it is:** SSTable (Sorted String Table) = log segments where keys are sorted. LSM-Tree = a tree of SSTables at different levels.

**Why sorted?** Sorting enables:
1. Efficient merging (like merge-sort)
2. Sparse indexes (don't need every key in memory)
3. Range queries

```
┌─────────────────────────────────────────────────────────────┐
│                  LSM-Tree Write Path                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   1. Write to memtable (in-memory sorted tree)             │
│                                                             │
│   ┌──────────────┐                                         │
│   │   Memtable   │  ← All writes go here first             │
│   │ (Red-Black)  │    (kept sorted in memory)              │
│   └──────┬───────┘                                         │
│          │ Flush when full (~few MB)                       │
│          ▼                                                  │
│   2. Flush to SSTable on disk                              │
│                                                             │
│   ┌──────────────┐                                         │
│   │  SSTable-1   │  ← Immutable, sorted, compressed        │
│   └──────────────┘                                         │
│   ┌──────────────┐                                         │
│   │  SSTable-2   │  ← Older data                           │
│   └──────────────┘                                         │
│          │                                                  │
│          ▼                                                  │
│   3. Background compaction merges SSTables                 │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Why a memtable?** You can't append to a sorted file (you'd have to rewrite it). Instead, buffer writes in a sorted in-memory structure, then flush as a sorted segment.

**Compaction Strategies:**

```
Size-Tiered (Cassandra default, HBase):
┌────┐ ┌────┐ ┌────┐
│ S1 │ │ S2 │ │ S3 │  → When similar-sized, merge them
└────┘ └────┘ └────┘
- Good for write-heavy workloads
- Can use lots of disk space temporarily

Leveled (LevelDB, RocksDB):
Level 0: ┌──┐ ┌──┐ ┌──┐     (overlapping keys allowed)
Level 1: ┌────────────────┐  (non-overlapping, 10x bigger)
Level 2: ┌────────────────────────────────┐
- Better read performance (fewer files to check)
- More consistent space usage
- Higher write amplification
```

**What is write amplification?** Data gets written multiple times (memtable → SSTable → compacted SSTable). A single 1KB write might result in 10-30KB of actual disk writes.

### LSM-Tree Read Path

```
┌─────────────────────────────────────────────────────────────┐
│                  LSM-Tree Read Path                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   Read key "user123":                                       │
│                                                             │
│   1. Check memtable          ──► Found? Return             │
│          │                                                  │
│          ▼ Not found                                        │
│   2. Check SSTable-1 (newest) ──► Found? Return            │
│          │                                                  │
│          ▼ Not found                                        │
│   3. Check SSTable-2          ──► Found? Return            │
│          │                                                  │
│          ▼ Not found                                        │
│   4. ... older SSTables ...                                │
│          │                                                  │
│          ▼                                                  │
│   5. Not found anywhere → Return null                      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Why LSM reads can be slow:** For a missing key, you check EVERY level. This is the read amplification problem.

**Bloom Filter: The Solution**

**What it is:** A probabilistic data structure that answers "is this key present?" 
- "Definitely NOT present" → Skip this SSTable (saves a disk read!)
- "Maybe present" → Check the SSTable

**Why it works:** A bloom filter is a bit array with multiple hash functions. False positives possible (says "maybe" when key isn't there), but false negatives impossible (never says "no" when key IS there).

> 💡 **Interview Tip:** Always mention bloom filters when discussing LSM-trees. They're crucial for read performance and show you understand the read amplification problem.

---

## B-Trees (Page-Oriented Storage)

### What it is

B-Trees break the database into fixed-size pages (typically 4KB) and organize them as a balanced tree. Unlike LSM-trees, B-trees update data IN-PLACE.

**Why B-Trees are everywhere:** They've been the standard for 40+ years. Every traditional RDBMS uses them because they provide consistent, predictable performance.

### Structure

```
┌─────────────────────────────────────────────────────────────┐
│                      B-Tree Structure                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│                    ┌─────────────┐                          │
│                    │  [100|200]  │  ← Root page             │
│                    └──────┬──────┘                          │
│              ┌────────────┼────────────┐                    │
│              ▼            ▼            ▼                    │
│        ┌─────────┐  ┌─────────┐  ┌─────────┐               │
│        │ [30|70] │  │[130|170]│  │[250|300]│  ← Internal   │
│        └────┬────┘  └────┬────┘  └────┬────┘               │
│        ┌────┴────┐       │       ┌────┴────┐               │
│        ▼         ▼       ▼       ▼         ▼               │
│    ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐           │
│    │leaf  │ │leaf  │ │leaf  │ │leaf  │ │leaf  │  ← Leaves │
│    │data  │ │data  │ │data  │ │data  │ │data  │   (pages) │
│    └──────┘ └──────┘ └──────┘ └──────┘ └──────┘           │
│                                                             │
│   Page size: typically 4KB (matches OS page size)          │
│   Branching factor: ~500 (keys per page)                   │
│   Depth: 3-4 levels handles BILLIONS of keys               │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Why so shallow?** With branching factor of 500: 500^4 = 62.5 billion keys in just 4 levels. Most reads need only 3-4 disk seeks.

### Write-Ahead Log (WAL)

**What it is:** Before modifying any B-tree page, write the intended change to a sequential log file first.

**Why it's necessary:** B-tree writes are in-place. If you crash mid-write, the page is corrupted. WAL ensures you can recover.

```
Write "user123" → {"name": "Alice"}

1. Append to WAL (fast sequential write)
   ┌─────────────────────────────────────┐
   │ WAL: [SET user123 {"name":"Alice"}] │
   └─────────────────────────────────────┘

2. Update B-tree page (slower random I/O)
   ┌─────────────────────────────────────┐
   │ Page 42: [user123|Alice|...]        │
   └─────────────────────────────────────┘

If crash after step 1 but before step 2:
→ On restart, replay WAL to recover the write
```

> 💡 **Interview Tip:** WAL is used for crash recovery AND replication. PostgreSQL streams WAL to replicas. This is "physical replication" vs "logical replication" (streaming SQL statements).

---

## LSM-Trees vs B-Trees: The Key Decision

**When someone asks "which database should I use?", this is the fundamental trade-off:**

| Aspect | LSM-Trees | B-Trees |
|--------|-----------|---------|
| **Write throughput** | Higher (sequential I/O) | Lower (random I/O) |
| **Read latency** | Variable (check multiple levels) | Predictable (single location) |
| **Space efficiency** | Better (compression, no fragmentation) | Some fragmentation |
| **Write amplification** | Higher (compaction rewrites data) | Lower |
| **Read amplification** | Higher (multiple SSTables) | Lower (single tree) |
| **Latency spikes** | Yes (compaction) | No |

### Real-World Decision Guide

**Choose LSM-Tree (Cassandra, RocksDB, HBase) when:**
- Write-heavy workload (logging, IoT, time-series)
- High throughput matters more than latency consistency
- Storage cost matters (better compression)
- Example: "We need to ingest 1 million events/second"

**Choose B-Tree (PostgreSQL, MySQL) when:**
- Read-heavy or mixed workload
- Need consistent, predictable latency (SLAs)
- Transactions are important (easier to implement ACID)
- Example: "Users expect sub-100ms response times"

> 💡 **Interview Tip:** Don't just say "Cassandra is faster for writes." Explain WHY: sequential I/O, no random seeks, no page splits. Show you understand the mechanics.

---

## OLTP vs OLAP: Two Different Worlds

**What's the difference?** OLTP and OLAP aren't just workload types—they need fundamentally different storage designs.

```
┌─────────────────────────────────────────────────────────────┐
│              OLTP vs OLAP Workloads                          │
├────────────────────────┬────────────────────────────────────┤
│        OLTP            │           OLAP                     │
│  (Online Transaction)  │  (Online Analytical)               │
├────────────────────────┼────────────────────────────────────┤
│ Many small txns        │ Few large scans                    │
│ Read/write single rows │ Read millions of rows              │
│ Latest state           │ Historical analysis                │
│ User-facing apps       │ Business intelligence              │
├────────────────────────┼────────────────────────────────────┤
│ MySQL, PostgreSQL      │ Snowflake, Redshift, BigQuery     │
│ MongoDB                │ ClickHouse, DuckDB                 │
└────────────────────────┴────────────────────────────────────┘
```

**Real examples:**
- **OLTP:** "Show me order #12345" (point lookup, low latency)
- **OLAP:** "What were total sales by region last quarter?" (scan millions of rows)

**Why separate systems?** 
- OLTP optimized for random access to individual rows
- OLAP optimized for scanning huge amounts of data
- Running analytics on your production DB can kill it!

### Data Warehouse Architecture

**What it is:** A separate database optimized for analytics, fed by ETL from production systems.

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│   OLTP DBs          ETL              Data Warehouse         │
│   ┌──────┐                           ┌────────────────┐     │
│   │Orders│ ──┐                       │                │     │
│   └──────┘   │    ┌──────────┐       │   Fact Tables  │     │
│   ┌──────┐   ├───►│ Extract  │       │  ┌──────────┐  │     │
│   │Users │ ──┤    │Transform │──────►│  │  Sales   │  │     │
│   └──────┘   │    │  Load    │       │  │  Facts   │  │     │
│   ┌──────┐   │    └──────────┘       │  └──────────┘  │     │
│   │Prods │ ──┘                       │                │     │
│   └──────┘                           │ Dimension Tbls │     │
│                                      │  ┌─────┐┌─────┐│     │
│                                      │  │Date ││Prod ││     │
│                                      │  └─────┘└─────┘│     │
│                                      └────────────────┘     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Star Schema:** Fact tables (events/transactions) reference dimension tables (who, what, when, where). Called "star" because diagram looks like a star.

> 💡 **Interview Tip:** When designing analytics systems, mention star schema and explain why denormalization is OK here (read-heavy, no updates to historical data).

---

## Column-Oriented Storage

### Row vs Column Storage: Why It Matters

**What it is:** Instead of storing all columns of a row together, store all values of each column together.

```
Row-oriented (OLTP - PostgreSQL, MySQL):
┌─────────────────────────────────────┐
│ Row 1: date|product|price|quantity  │  ← Entire row stored together
│ Row 2: date|product|price|quantity  │
│ Row 3: date|product|price|quantity  │
└─────────────────────────────────────┘

Column-oriented (OLAP - ClickHouse, Redshift):
┌────────────────┐
│ date column:   │ 2024-01-01, 2024-01-01, 2024-01-02...
├────────────────┤
│ product column:│ Widget, Gadget, Widget...
├────────────────┤
│ price column:  │ 29.99, 49.99, 29.99...
├────────────────┤
│ quantity col:  │ 100, 50, 200...
└────────────────┘
```

### Why Column Storage Wins for Analytics

```sql
-- Typical analytics query
SELECT product, SUM(quantity * price) as revenue
FROM sales
WHERE date BETWEEN '2024-01-01' AND '2024-12-31'
GROUP BY product;
```

**Row storage:** Must read ALL columns of every row, even though we only need 4 columns. For a 100-column table, we read 25x more data than needed!

**Column storage:** Read ONLY the 4 columns needed. Plus columns compress much better (similar values grouped together).

**The numbers:** Column stores routinely achieve 10x compression ratios and 10-100x faster analytics queries.

### Column Compression Techniques

**Why columns compress well:** Similar values are adjacent. A "country" column might have "USA" repeated millions of times.

```
┌─────────────────────────────────────────────────────────────┐
│               Bitmap Encoding Example                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   product_id values: [A, B, A, A, C, B, A, C, C, A]        │
│                                                             │
│   Bitmap for A: [1,0,1,1,0,0,1,0,0,1]                      │
│   Bitmap for B: [0,1,0,0,0,1,0,0,0,0]                      │
│   Bitmap for C: [0,0,0,0,1,0,0,1,1,0]                      │
│                                                             │
│   WHERE product = 'A' → just scan bitmap A (super fast!)   │
│   WHERE product IN ('A','B') → bitmap A OR bitmap B        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**When to use bitmaps:** Low-cardinality columns (country, status, category). Not for high-cardinality (user_id, email).

### Run-Length Encoding

```
Original:   [A,A,A,A,A,B,B,C,C,C,C,C,C,C,C]
Encoded:    [(A,5), (B,2), (C,8)]

15 values → 6 values (60% compression!)
```

**When this shines:** Sorted columns. If you sort by date, then by product, the product column will have long runs of the same value.

> 💡 **Interview Tip:** When discussing analytics databases, explain that column storage isn't just about reading fewer columns—it's about enabling compression techniques that are impossible with row storage.

---

## Practical Index Types

### Secondary Indexes

**What's the difference?**

```sql
-- Primary key / clustered index: rows physically stored in this order
CREATE INDEX ON users(id);  

-- Secondary index: separate structure pointing to row locations
CREATE INDEX ON users(email);  
```

**Why it matters:** Clustered index lookups are faster (data is right there). Secondary indexes require an extra hop to fetch the actual row.

### Multi-Column (Composite) Indexes

**The leftmost prefix rule:** A composite index on (a, b, c) can be used for queries on (a), (a, b), or (a, b, c), but NOT (b) or (c) alone.

```sql
-- Concatenated index
CREATE INDEX ON events(year, month, day);

-- Query optimization:
WHERE year = 2024                    -- Uses index ✓
WHERE year = 2024 AND month = 1      -- Uses index ✓
WHERE year = 2024 AND month = 1 AND day = 15  -- Uses index ✓
WHERE month = 1                       -- Cannot use index ✗
WHERE month = 1 AND day = 15          -- Cannot use index ✗
```

> 💡 **Interview Tip:** When asked about slow queries, always check if the WHERE clause matches the index's leftmost columns. This is a common gotcha.

### Full-Text Search Index (Inverted Index)

**What it is:** Maps words → documents that contain them (the reverse of a normal index).

```
┌─────────────────────────────────────────────────────────────┐
│                   Inverted Index                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   Documents:                                                │
│   Doc1: "The quick brown fox"                              │
│   Doc2: "The lazy dog"                                     │
│   Doc3: "Quick brown dog"                                  │
│                                                             │
│   Inverted Index:                                          │
│   "brown" → [Doc1, Doc3]                                   │
│   "dog"   → [Doc2, Doc3]                                   │
│   "fox"   → [Doc1]                                         │
│   "lazy"  → [Doc2]                                         │
│   "quick" → [Doc1, Doc3]                                   │
│   "the"   → [Doc1, Doc2]                                   │
│                                                             │
│   Search "brown dog" → intersect([Doc1,Doc3], [Doc2,Doc3]) │
│                      → Doc3                                 │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Used by:** Elasticsearch, Solr, PostgreSQL full-text search

**When to use:** Search engines, log analysis, any "find documents containing..." query

---

## Key Takeaways

1. **Two engine families:** LSM-Trees (write-optimized, sequential I/O) vs B-Trees (read-optimized, in-place updates)
2. **OLTP vs OLAP:** Different workloads need fundamentally different storage designs
3. **Column storage:** Essential for analytics—read less data, compress better
4. **Index trade-off:** Every index speeds up reads but slows down writes
5. **Know your access patterns:** The "best" database depends entirely on how you'll use it

---

## Quick Reference

| Workload | Storage Engine | Database Examples |
|----------|----------------|-------------------|
| Write-heavy OLTP | LSM-Tree | Cassandra, RocksDB, ScyllaDB |
| Read-heavy OLTP | B-Tree | PostgreSQL, MySQL |
| Analytics (OLAP) | Column-oriented | ClickHouse, Redshift, Snowflake |
| Full-text search | Inverted index | Elasticsearch, Solr |
| Time series | LSM + Time partitions | InfluxDB, TimescaleDB |

---

## System Design Interview Tips

1. **Start with access patterns:** "Before choosing a database, let me understand the read/write ratio and query patterns."

2. **Explain the WHY:** Don't just say "I'd use Cassandra." Say "Cassandra uses LSM-trees which are optimized for high write throughput because writes are sequential."

3. **Mention trade-offs:** "LSM-trees give us great write throughput, but we might see latency spikes during compaction. For user-facing reads, we might add a caching layer."

4. **Know real numbers:** 
   - B-tree lookup: 3-4 disk seeks for billions of keys
   - SSD random read: ~100μs
   - SSD sequential read: ~10μs per KB

5. **Connect to real systems:**
   - "Uber uses Cassandra for high-write workloads like trip events"
   - "Facebook uses RocksDB as an embedded storage engine"
   - "Most traditional web apps use PostgreSQL because the B-tree's predictable latency matters"

---

## Common Interview Questions

**Q: "Why would you choose Cassandra over PostgreSQL?"**
A: Cassandra uses LSM-trees optimized for write-heavy workloads. It handles high write throughput well because it turns random writes into sequential writes. Use it when you need to ingest massive amounts of data (logging, IoT, events). PostgreSQL's B-tree is better when you need consistent read latency and complex queries.

**Q: "Why do analytics databases use column storage?"**
A: Analytics queries typically read few columns but many rows. Column storage lets you read only the columns you need and enables much better compression (similar values grouped together). A query scanning 1 billion rows but only 3 columns reads 97% less data in a column store vs row store.

**Q: "What's write amplification and why does it matter?"**
A: Write amplification means one logical write causes multiple physical writes. In LSM-trees, data is written to memtable, then flushed to SSTable, then rewritten during compaction—possibly 10-30x amplification. This affects SSD lifespan and throughput. It's the price you pay for fast sequential writes.

**Q: "How would you handle a slow query in PostgreSQL?"**
A: First, check EXPLAIN ANALYZE to see if indexes are being used. Common issues: missing index, index not matching query's leftmost columns, or too many rows being scanned. Consider if the query pattern matches the storage engine—if it's analytics-heavy, maybe it belongs in a column store.
