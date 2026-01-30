# Chapter 6: Partitioning (Sharding)

## Core Concept

> "For very large datasets or very high throughput, replication alone is not enough. We need to break data into partitions."

### What it is
Partitioning splits your data across multiple machines. Each piece of data belongs to exactly one partition (though that partition may be replicated).

### Why it matters
A single server has hard limits:
- Storage: ~10 TB practical limit
- CPU: Can't parallelize beyond cores  
- Network: Single NIC bottleneck

Partitioning breaks these limits by adding more machines.

**Terminology:**
- Partition = Shard (MongoDB) = Region (HBase) = Tablet (Bigtable) = vNode (Cassandra)

```
┌─────────────────────────────────────────────────────────────┐
│                Why Partition?                                │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   With Partitioning:                                        │
│   • 1000 nodes × 10 TB = 10 PB storage                    │
│   • Queries can run in parallel across partitions          │
│   • Each partition on different node = more bandwidth      │
│                                                             │
│   Goal: Spread data AND load evenly across nodes            │
│   Challenge: HOT SPOTS (one partition gets all traffic)    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

> 💡 **Interview Tip:** Partitioning is about SCALABILITY, replication is about AVAILABILITY. They're often used together but solve different problems.

---

## Partitioning Strategies

There are two main approaches: partition by key range or by hash of key.

### 1. Key Range Partitioning

**What it is:** Assign continuous ranges of keys to partitions (like encyclopedia volumes A-C, D-F, etc.)

```
┌─────────────────────────────────────────────────────────────┐
│              Key Range Partitioning                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   Keys: a-zzz                                               │
│                                                             │
│   Partition 1: a-g                                         │
│   Partition 2: h-n                                         │
│   Partition 3: o-z                                         │
│                                                             │
│   Like volumes of encyclopedia!                            │
│                                                             │
│   Pros:                                                    │
│   ✓ Range queries efficient (all keys a-c on same node)   │
│   ✓ Easy to understand                                     │
│                                                             │
│   Cons:                                                    │
│   ✗ Hot spots if keys not uniformly distributed           │
│   ✗ Example: Timestamp keys → all writes to latest partition│
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Used by:** HBase, Bigtable, MongoDB (before 4.4)

### Hot Spot Example (Critical!)

**The Problem:** If you partition a time-series database by timestamp, ALL current writes go to the "today" partition while historical partitions sit idle.

```
Keys: Timestamps (2024-01-01, 2024-01-02, ...)

All writes go to the "latest" partition!
┌─────────────────────────────────────────────────────────────┐
│   Jan-Mar: █░░░░░░░░░░                                     │
│   Apr-Jun: █░░░░░░░░░░                                     │
│   Jul-Sep: █░░░░░░░░░░                                     │
│   Oct-Dec: ████████████████████  ← HOT SPOT!               │
└─────────────────────────────────────────────────────────────┘
```

**Solution:** Prefix with something that distributes (like sensor_id + timestamp). Now writes spread across partitions.

> 💡 **Interview Tip:** When asked about partition key design, ALWAYS mention hot spots. It's the #1 cause of partition-related production issues.

---

### 2. Hash Partitioning

**What it is:** Hash the key, use hash value to determine partition. Spreads data evenly regardless of key patterns.

**Why it's popular:** Eliminates hot spots from key patterns. "user_1", "user_2", "user_3" hash to completely different values.

```
┌─────────────────────────────────────────────────────────────┐
│                Hash Partitioning                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   partition = hash(key) % num_partitions                   │
│                                                             │
│   Key "user_123" → hash → 2847291 → 2847291 % 4 = 3       │
│   Key "user_456" → hash → 9182736 → 9182736 % 4 = 0       │
│                                                             │
│   ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐             │
│   │ Part 0 │ │ Part 1 │ │ Part 2 │ │ Part 3 │             │
│   │user_456│ │user_789│ │user_012│ │user_123│             │
│   └────────┘ └────────┘ └────────┘ └────────┘             │
│                                                             │
│   Pros:                                                    │
│   ✓ Even distribution (no hot spots)                       │
│   ✓ Works for any key type                                │
│                                                             │
│   Cons:                                                    │
│   ✗ Range queries require scatter-gather to ALL partitions │
│   ✗ Changing partition count = rehash everything           │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Used by:** Cassandra, DynamoDB, MongoDB (default), Redis Cluster

### Trade-off Summary

| | Range Partitioning | Hash Partitioning |
|---|---|---|
| **Range queries** | ✅ Efficient (single partition) | ❌ Scatter-gather (all partitions) |
| **Even distribution** | ❌ Depends on key patterns | ✅ Always even |
| **Hot spots** | ❌ Common problem | ✅ Rare (unless one key is hot) |
| **Use when** | Need range scans (time-series) | Even distribution is priority |

---

### Consistent Hashing

**What it is:** A way to add/remove nodes without rehashing everything. Keys and nodes both map to a ring; keys belong to the next node clockwise.

**Why it matters:** With simple `hash % N`, changing N moves MOST keys. With consistent hashing, only K/N keys move on average.

```
┌─────────────────────────────────────────────────────────────┐
│                 Consistent Hashing                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   Hash ring (0 to 2^32):                                   │
│                                                             │
│                    Node A                                   │
│                      │                                      │
│            ┌─────────┼─────────┐                           │
│         Node D                Node B                        │
│            │                   │                           │
│            └─────────┬─────────┘                           │
│                    Node C                                   │
│                                                             │
│   Key "user_123" → hash → walks clockwise to find node     │
│                                                             │
│   Adding/removing node only affects neighboring partitions │
│   (not complete rehash!)                                   │
│                                                             │
│   Virtual nodes: Each physical node owns multiple points   │
│   on ring for better balance                               │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

> 💡 **Interview Tip:** "Consistent hashing" is technically different from what databases like Cassandra use. They use fixed partitions assigned to nodes. Know the distinction if asked.

---

### Compound/Composite Keys (Best of Both Worlds)

**What it is:** Hash the first part of the key (for distribution), then range-partition within each hash bucket (for efficient scans).

**This is the Cassandra pattern - memorize it!**

```
┌─────────────────────────────────────────────────────────────┐
│       Cassandra Compound Key Strategy                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   CREATE TABLE posts (                                      │
│     user_id UUID,           -- Partition key (hashed)      │
│     post_time TIMESTAMP,    -- Clustering column (sorted)  │
│     content TEXT,                                          │
│     PRIMARY KEY (user_id, post_time)                       │
│   );                                                        │
│                                                             │
│   ┌─────────────────────────────────────────────────────┐  │
│   │ Partition (user_123)                                │  │
│   │ ┌─────────────────────────────────────────────────┐ │  │
│   │ │ 2024-01-01 09:00 │ Hello world                  │ │  │
│   │ │ 2024-01-01 10:00 │ Another post                 │ │  │
│   │ │ 2024-01-02 08:00 │ Good morning                 │ │  │
│   │ └─────────────────────────────────────────────────┘ │  │
│   └─────────────────────────────────────────────────────┘  │
│                                                             │
│   Result:                                                  │
│   • All posts for user_123 on same partition              │
│   • Posts sorted by time within partition                 │
│   • Efficient: "Get user_123's posts from Jan 2024"       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**DynamoDB equivalent:** Partition Key (PK) + Sort Key (SK). Same concept, different names.

---

## Secondary Indexes with Partitioning

Secondary indexes are tricky with partitioned data because the index doesn't align with the partition key.

### Two Approaches:

| | Local Index | Global Index |
|---|---|---|
| **Structure** | Each partition indexes its own data | Index is partitioned separately |
| **Writes** | ✅ Fast (update one partition) | ❌ Slow (may update many partitions) |
| **Reads** | ❌ Scatter-gather (query all partitions) | ✅ Fast (query one index partition) |
| **Use when** | Write-heavy workloads | Read-heavy on secondary attributes |

### Document-Partitioned (Local) Index

**What it is:** Each partition maintains its own index covering only its local data.

```
┌─────────────────────────────────────────────────────────────┐
│            Local Index (Document-Partitioned)                │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   Each partition has its own index for its data            │
│                                                             │
│   Partition 0                    Partition 1                │
│   ┌───────────────────┐         ┌───────────────────┐      │
│   │ Data: cars 0-499  │         │ Data: cars 500-999│      │
│   │ Index: color=red  │         │ Index: color=red  │      │
│   │   → [car_123]     │         │   → [car_567]     │      │
│   └───────────────────┘         └───────────────────┘      │
│                                                             │
│   Query: "Find all red cars"                               │
│   → Must query ALL partitions (scatter-gather)             │
│   → Combine results                                        │
│                                                             │
│   Pros: Writes update only one partition                   │
│   Cons: Reads may need to hit all partitions (expensive!)  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Used by:** MongoDB, Cassandra, Elasticsearch, SolrCloud

### Term-Partitioned (Global) Index

**What it is:** The index itself is partitioned by the indexed term. All "color=red" entries go to one index partition.

```
┌─────────────────────────────────────────────────────────────┐
│            Global Index (Term-Partitioned)                   │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   Index is also partitioned (by index term)                │
│                                                             │
│   Index Partition A-M            Index Partition N-Z       │
│   ┌───────────────────┐         ┌───────────────────┐      │
│   │ color=blue →      │         │ color=red →       │      │
│   │   [car_1, car_99] │         │   [car_5, car_42] │      │
│   └───────────────────┘         └───────────────────┘      │
│                                                             │
│   Query: "Find all red cars"                               │
│   → Query only the "r" index partition                     │
│   → Single partition read!                                 │
│                                                             │
│   Pros: Efficient reads (single partition for a term)      │
│   Cons: Writes may update multiple index partitions        │
│         (often async to avoid slow writes)                 │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Used by:** DynamoDB Global Secondary Indexes, Riak search

> 💡 **Interview Tip:** When asked about secondary indexes in a distributed database, mention this trade-off. There's no free lunch - either writes or reads will be expensive.

---

## Rebalancing Partitions

### What it is
Moving data between nodes to maintain even distribution as nodes are added/removed or data grows.

### Why it matters
Without proper rebalancing, you end up with overloaded nodes while others sit idle.

### Why NOT to Use: hash(key) mod N

```
┌─────────────────────────────────────────────────────────────┐
│        DON'T: hash(key) mod N                               │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   N=4: hash(key) % 4 = partition                           │
│   N=5: hash(key) % 5 = partition  ← Everything moves!      │
│                                                             │
│   Changing N requires moving MOST data!                    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**This is the classic mistake.** If you go from 4 to 5 nodes, almost every key changes which partition it belongs to.

### The Right Way: Fixed Number of Partitions

**What it is:** Create way more partitions than nodes upfront. When adding nodes, just move whole partitions.

```
┌─────────────────────────────────────────────────────────────┐
│        Fixed number of partitions                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   Create 1000 partitions upfront (more than nodes)         │
│                                                             │
│   4 nodes: each has 250 partitions                         │
│   Add node: steal partitions from others                   │
│                                                             │
│   Node 1: [0-249]    →    Node 1: [0-199]                  │
│   Node 2: [250-499]  →    Node 2: [250-399]                │
│   Node 3: [500-749]  →    Node 3: [500-599]                │
│   Node 4: [750-999]  →    Node 4: [750-849]                │
│                           Node 5: [200-249,400-449,...]    │
│                                                             │
│   Only partition assignment changes, not partitioning!     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Used by:** Elasticsearch, Couchbase, Riak, Voldemort

**Trade-off:** Must guess partition count upfront. Too few = large partitions that are slow to move. Too many = overhead per partition.

### Dynamic Partitioning

**What it is:** Start small, split partitions when they get too big, merge when they shrink.

```
┌─────────────────────────────────────────────────────────────┐
│              Dynamic Partitioning                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   Start with few partitions, split as they grow            │
│                                                             │
│   Initial:      After growth:     After more growth:       │
│   ┌─────┐       ┌─────┬─────┐     ┌──┬──┬──┬──┐           │
│   │  A  │   →   │  A  │  B  │  →  │A1│A2│B1│B2│           │
│   └─────┘       └─────┴─────┘     └──┴──┴──┴──┘           │
│                                                             │
│   Split threshold: ~10GB per partition (HBase)             │
│   Merge threshold: Below half of split size                │
│                                                             │
│   Good for key-range partitioning                          │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Used by:** HBase, RethinkDB, MongoDB

### Rebalancing Strategy Comparison

| Strategy | Pros | Cons | Used By |
|----------|------|------|---------|
| Fixed partitions | Simple, predictable | Must guess count upfront | Elasticsearch, Riak |
| Dynamic split/merge | Adapts to data | More complex, initial bottleneck | HBase, MongoDB |
| Consistent hashing | Minimal data movement | Uneven distribution without vnodes | Cassandra |

---

## Request Routing

### The Problem
Client has a key. Which node holds that key's partition?

### Three Approaches

```
┌─────────────────────────────────────────────────────────────┐
│                 Routing Approaches                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   1. Client-side routing                                   │
│   ┌────────┐                                               │
│   │ Client │ ─── knows partition mapping                   │
│   └───┬────┘     directly contacts correct node            │
│       │                                                     │
│       └──────────► Node 3 (has the partition)              │
│                                                             │
│   2. Routing tier (partition-aware load balancer)          │
│   ┌────────┐    ┌─────────────┐                           │
│   │ Client │───►│ Router/LB   │───► Node 3                │
│   └────────┘    │(partition-  │                           │
│                 │ aware)      │                            │
│                 └─────────────┘                            │
│                                                             │
│   3. Contact any node (gossip-based)                       │
│   ┌────────┐                                               │
│   │ Client │───► Node 1 ──(forwards)──► Node 3            │
│   └────────┘                                               │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

| Approach | Pros | Cons | Used By |
|----------|------|------|---------|
| Client-aware | Lowest latency | Client complexity | Cassandra drivers |
| Routing tier | Simple clients | Extra hop | MongoDB (mongos) |
| Any-node | Simple everything | Extra hop + gossip overhead | Cassandra, Riak |

### Coordination Service (ZooKeeper Pattern)

```
┌─────────────────────────────────────────────────────────────┐
│           ZooKeeper for Partition Metadata                   │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│                 ┌─────────────┐                            │
│                 │  ZooKeeper  │                            │
│                 │             │                            │
│                 │ partition 0 │                            │
│                 │  → node 1   │                            │
│                 │ partition 1 │                            │
│                 │  → node 2   │                            │
│                 └──────┬──────┘                            │
│            subscribe   │   register                        │
│          ┌─────────────┴─────────────┐                    │
│          │                           │                    │
│    ┌─────▼─────┐              ┌──────▼─────┐             │
│    │  Client/  │              │   Nodes    │             │
│    │  Router   │              │ (register  │             │
│    │ (watches) │              │  partition │             │
│    └───────────┘              │  ownership)│             │
│                               └────────────┘             │
│                                                             │
│   When partition moves: ZK notifies clients automatically  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Used by:** HBase, Kafka, SolrCloud

### Gossip Protocol (No Coordinator)

```
┌─────────────────────────────────────────────────────────────┐
│                  Gossip Protocol                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   Each node knows full partition map                       │
│   Nodes periodically exchange state with random peers      │
│                                                             │
│   Node 1 ◄──────► Node 2                                   │
│     │                │                                      │
│     │                │                                      │
│     ▼                ▼                                      │
│   Node 3 ◄──────► Node 4                                   │
│                                                             │
│   Changes propagate eventually (O(log N) rounds)           │
│                                                             │
│   Client contacts any node → node routes if needed         │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Used by:** Cassandra, Riak

---

## Practical Examples

### DynamoDB Partition Design

```python
# Good partition key: High cardinality, evenly distributed
table.put_item(Item={
    'PK': f'USER#{user_id}',           # Partition key
    'SK': f'ORDER#{order_id}',          # Sort key
    'OrderDate': '2024-01-15',
    'Total': 99.99
})

# Query: All orders for a user (single partition)
table.query(
    KeyConditionExpression='PK = :pk AND begins_with(SK, :sk)',
    ExpressionAttributeValues={
        ':pk': 'USER#123',
        ':sk': 'ORDER#'
    }
)

# Anti-pattern: Date as partition key (hot partition!)
# 'PK': '2024-01-15'  # All today's data on one partition
```

### MongoDB Sharding Setup

```javascript
// Enable sharding on database
sh.enableSharding("mydb")

// Shard collection with hashed key (even distribution)
sh.shardCollection("mydb.users", { "_id": "hashed" })

// Or range-based for time-series with compound key
sh.shardCollection("mydb.events", { 
  "tenant_id": 1,      // Partition by tenant first
  "timestamp": 1        // Then range within tenant
})

// Check distribution
db.users.getShardDistribution()
```

---

## Hot Spot Mitigation

**The celebrity problem:** Even with hash partitioning, one key can be extremely hot. A celebrity's post gets millions of writes (likes, comments) all hitting one partition.

```
┌─────────────────────────────────────────────────────────────┐
│              Handling Hot Partitions                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   Problem: Celebrity posts (millions of likes)             │
│   Solution: Add random suffix to split load                │
│                                                             │
│   Instead of:  post_123                                    │
│   Use:         post_123_0, post_123_1, ... post_123_99    │
│                                                             │
│   Writes: Randomly pick suffix (spread across partitions)  │
│   Reads:  Query all 100 keys, aggregate                   │
│                                                             │
│   Trade-off: Faster writes, slower reads                   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Real-world approach:** Maintain a list of "hot" keys (detected via metrics). Only apply key-splitting to those, not everything.

> 💡 **Interview Tip:** Hot spots show you understand that partitioning isn't magic. Even with perfect distribution strategy, application access patterns can create hot spots.

---

## Key Takeaways

1. **Hash partitioning** for even distribution, **range** for ordered access
2. **Compound keys** give best of both (Cassandra/DynamoDB pattern)
3. **Local indexes** = fast writes, scatter-gather reads
4. **Global indexes** = fast reads, complex writes
5. **Fixed partitions** easier than dynamic for rebalancing
6. **Don't use mod N** - causes massive data movement
7. **Hot spots** happen even with hashing - design partition keys carefully

---

## Quick Reference

| Database | Default Partitioning | Rebalancing |
|----------|---------------------|-------------|
| Cassandra | Hash (vnodes) | Automatic |
| MongoDB | Hash or Range | Automatic |
| DynamoDB | Hash | Automatic (hidden) |
| HBase | Range | Dynamic split/merge |
| Elasticsearch | Hash (fixed count) | Manual shard allocation |
| Kafka | Hash by key | Manual partition reassign |

---

## System Design Interview Tips

### Choosing a Partition Key

Ask these questions:
1. **What's my access pattern?** (point lookups vs range scans)
2. **What's the cardinality?** (high cardinality = good distribution)
3. **Are there hot keys?** (celebrity problem)
4. **Do I need efficient range queries?** (consider compound keys)

### Common Design Patterns

**User data:** Partition by `user_id` (high cardinality, even access)

**Time-series:** Partition by `device_id + timestamp` (avoid all-writes-to-today problem)

**Multi-tenant SaaS:** Partition by `tenant_id` (isolates tenants, but watch for large tenants)

### Red Flags to Avoid

- Using timestamp alone as partition key
- Using low-cardinality fields (country, status)
- Ignoring secondary index costs
- Assuming partition count never needs to change

---

## Common Interview Questions

**Q: Design a URL shortener. How would you partition the data?**
> Hash partition by the short URL (high cardinality, even access pattern). Use fixed number of partitions for easy rebalancing. No need for range queries.

**Q: How would you handle a "hot" partition for a viral post?**
> Add random suffix to the key (e.g., `post_123_0` through `post_123_99`). Writes distribute randomly across 100 keys. Reads query all 100 and aggregate. Only do this for detected hot keys.

**Q: What's the difference between local and global secondary indexes?**
> Local: index lives with data partition. Fast writes, but reads must scatter-gather all partitions. Global: index is partitioned by index term. Fast reads for index queries, but writes may touch multiple partitions.

**Q: DynamoDB vs Cassandra - how do their partition strategies differ?**
> Both use hash-based partitioning with partition key + sort key. DynamoDB hides partition management entirely. Cassandra uses vnodes (virtual nodes) and gossip protocol. Both support compound keys for efficient within-partition range queries.

**Q: How do you decide on the number of partitions?**
> Consider: expected data size, query throughput, node count. Rule of thumb: partitions should be 100MB-1GB. Too few = large partitions slow to rebalance. Too many = metadata overhead. Many systems use dynamic partitioning to avoid this choice.
