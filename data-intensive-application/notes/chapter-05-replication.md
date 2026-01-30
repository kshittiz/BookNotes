# Chapter 5: Replication

## Core Concept

> "Replication means keeping a copy of the same data on multiple machines."

### What it is
Replication stores copies of your data on multiple servers. When you write to a database, that write needs to appear on all copies eventually.

### Why it matters
Three main reasons drive every replication decision:
- **Latency:** Keep data close to users (EU users hit EU servers, not US)
- **Availability:** System survives node failures (if one server dies, others serve traffic)
- **Read throughput:** Scale reads horizontally (add more replicas to handle more queries)

**The Hard Part:** Replication is easy when data never changes. The challenge is handling *writes* - how do changes propagate? What if two people write at the same time?

> 💡 **Interview Tip:** When asked "why replicate?", always mention all three: latency, availability, AND throughput. Most candidates only mention one or two.

---

## Replication Strategies Overview

```
┌─────────────────────────────────────────────────────────────┐
│                  Replication Topologies                      │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   Single-Leader          Multi-Leader         Leaderless   │
│                                                             │
│       [L]                [L1]───[L2]         [N1]──[N2]    │
│      / | \                  \  /             /  \  /  \    │
│    [F] [F] [F]              [L3]           [N3]  [N4]      │
│                                                             │
│   One writer           Multiple writers    Any node writes │
│   Simple               Complex conflicts   Quorum-based    │
│   PostgreSQL           CockroachDB         Cassandra       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**When to use each:**
| Strategy | Use When | Avoid When |
|----------|----------|------------|
| Single-Leader | Most cases, strong consistency needed | Multi-DC writes required |
| Multi-Leader | Multi-DC, offline-capable apps | Conflicts are unacceptable |
| Leaderless | Maximum availability, eventual consistency OK | Need strong consistency |

---

## Single-Leader Replication

### What it is
One node (the leader/primary) accepts all writes. It sends changes to followers (replicas/secondaries) who can serve reads.

### Why it's the default choice
Single-leader is simple: there's exactly ONE place where writes happen, so there's no confusion about which write is "correct". This makes reasoning about your data much easier.

### How It Works

```
┌─────────────────────────────────────────────────────────────┐
│                    Leader-Follower Flow                      │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   Client                                                    │
│     │                                                       │
│     │ Write                                                 │
│     ▼                                                       │
│   ┌────────────┐                                           │
│   │   Leader   │ ──────────────────────────────────────    │
│   │ (Primary)  │           Replication Log                 │
│   └────────────┘                │                          │
│         │                       │                          │
│         │ Replicate             │                          │
│         ▼                       ▼                          │
│   ┌────────────┐         ┌────────────┐                   │
│   │ Follower 1 │         │ Follower 2 │  ← Reads can      │
│   │ (Replica)  │         │ (Replica)  │    go here        │
│   └────────────┘         └────────────┘                   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Real-world example:** PostgreSQL streaming replication, MySQL replication, MongoDB replica sets.

---

## Sync vs Async Replication

### What it is
When a client writes to the leader, should the leader wait for followers to confirm they received the write before acknowledging to the client?

### The Core Trade-off
- **Synchronous:** Wait for followers → **Durable but slow**
- **Asynchronous:** Don't wait → **Fast but may lose data**

```
┌─────────────────────────────────────────────────────────────┐
│                    Synchronous                               │
├─────────────────────────────────────────────────────────────┤
│   Client ──► Leader ──► Follower                           │
│      │         │            │                               │
│      │         │   write    │                               │
│      │         ├───────────►│                               │
│      │         │            │                               │
│      │         │◄───────────┤ ack                          │
│      │◄────────┤                                            │
│      │   ack   │  Guarantees: Data on 2 nodes before ack   │
│                   Downside: One slow follower blocks all   │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                    Asynchronous                              │
├─────────────────────────────────────────────────────────────┤
│   Client ──► Leader ──► Follower                           │
│      │         │            │                               │
│      │         │   write    │                               │
│      │◄────────┤            │  (happens eventually)        │
│      │   ack   ├───────────►│                               │
│                                                             │
│   Guarantees: Better performance                           │
│   Downside: Data loss possible if leader fails            │
└─────────────────────────────────────────────────────────────┘
```

### Semi-Synchronous (What's Actually Used)
One follower is synchronous, others are async. This guarantees data exists on at least 2 nodes before acknowledgment, while avoiding the "one slow node blocks everything" problem.

> 💡 **Interview Tip:** If asked "sync or async?", always mention semi-synchronous as the practical middle ground. Pure sync is rarely used because it's too slow; pure async risks data loss.

### When to Use Each

| Mode | Use Case | Example |
|------|----------|---------|
| Synchronous | Financial transactions where data loss is unacceptable | Banking systems |
| Asynchronous | Social media, logging, analytics where speed > durability | Twitter likes |
| Semi-sync | Most production databases | PostgreSQL default |

---

### Setting Up New Followers

```
1. Take consistent snapshot of leader
   └─► pg_basebackup (PostgreSQL)
   └─► mysqldump --single-transaction (MySQL)

2. Copy snapshot to new follower

3. Follower requests all changes since snapshot
   └─► Uses replication log position (LSN, binlog position)

4. Follower catches up ("caught up")
```

---

## Handling Node Failures

### Follower Failure (Easy)
```
Follower restarts → Reads local log → Requests missing changes from leader
```
Followers keep track of their position in the replication log. When they recover, they just ask for everything since that position.

### Leader Failure: Failover (The Hard Part!)

**What is failover?** When the leader dies, you need to promote a follower to become the new leader. Sounds simple, but it's one of the hardest problems in distributed systems.

```
┌─────────────────────────────────────────────────────────────┐
│                     Failover Process                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   1. Detect leader failure (timeout-based, ~30s)           │
│                                                             │
│   2. Choose new leader                                      │
│      - Most up-to-date replica                             │
│      - Consensus among nodes                               │
│                                                             │
│   3. Reconfigure system                                     │
│      - Clients write to new leader                         │
│      - Old leader becomes follower (when it recovers)      │
│                                                             │
│   DANGERS:                                                  │
│   - Split brain: Two nodes think they're leader!           │
│   - Lost writes: Async replication = data loss             │
│   - Cascading failures: Failover under load                │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Why Failover is Dangerous

**Split Brain:** Two nodes both think they're the leader and accept writes. You now have conflicting data that's nearly impossible to reconcile.

**Lost Writes:** With async replication, the new leader might be behind. Writes that were acknowledged to clients may be lost forever.

**The GitHub 2012 Incident:** MySQL failover caused primary key conflicts because the new leader was behind the old one. The old leader had assigned auto-increment IDs that the new leader reassigned to different rows.

> 💡 **Interview Tip:** Failover is a favorite interview topic. Know the three dangers (split brain, lost writes, cascading failures) and be able to explain why each happens.

---

## Replication Lag Problems

### What it is
With async replication, followers may be seconds, minutes, or even hours behind the leader. This causes subtle bugs where users see stale or inconsistent data.

### 1. Read-Your-Own-Writes

**The Problem:** User updates their profile, refreshes, and sees the OLD profile!

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│   User: Update profile photo                               │
│   User: Refresh page → Still sees old photo!               │
│                                                             │
│   Why?                                                      │
│   ┌────────┐ write  ┌────────┐                             │
│   │ Client ├───────►│ Leader │                             │
│   └───┬────┘        └───┬────┘                             │
│       │                 │ (replicating...)                 │
│       │ read            ▼                                   │
│       │            ┌────────┐                              │
│       └───────────►│Follower│ ← Stale data!               │
│                    └────────┘                              │
│                                                             │
│   Solutions:                                                │
│   - Read from leader for user's own data                   │
│   - Track update timestamp, read from leader if recent     │
│   - Ensure replica caught up before reading                │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Practical solution:** For a short window after a write (e.g., 1 minute), route that user's reads to the leader. After that, reads can go to any replica.

### 2. Monotonic Reads

**The Problem:** User sees data go backwards in time! They see a comment, refresh, and it disappears.

**Why it happens:** Request 1 hits an up-to-date replica. Request 2 hits a stale replica.

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│   Request 1: Read from Follower A → sees new data          │
│   Request 2: Read from Follower B → sees OLD data!         │
│                                                             │
│   User sees time go backwards (comment disappears!)        │
│                                                             │
│   Solution: Always route same user to same replica         │
│   (hash user_id to choose replica - sticky sessions)       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 3. Consistent Prefix Reads

**The Problem:** Causality violations - seeing an answer before the question!

**Why it happens:** In a partitioned database, different partitions may replicate at different speeds.

```
┌─────────────────────────────────────────────────────────────┐
│                Causality Violation                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   Actual order:                                            │
│   1. Alice: "How's the weather?"                           │
│   2. Bob: "It's sunny!"                                    │
│                                                             │
│   Reader sees (due to replication lag):                    │
│   1. Bob: "It's sunny!"                                    │
│   2. Alice: "How's the weather?"                           │
│                                                             │
│   Looks like Bob answered before question was asked!       │
│                                                             │
│   Solution: Causally related writes go to same partition   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

> 💡 **Interview Tip:** When designing a system, ask yourself: "What happens if reads hit a stale replica?" These three problems (read-your-writes, monotonic reads, consistent prefix) cover most of the subtle bugs.

---

## Multi-Leader Replication

### What it is
Multiple nodes can accept writes. Each leader replicates to the others.

### Why it's needed
Single-leader has ONE point where all writes must go. If your users are globally distributed, this means someone is always far from the leader, experiencing high latency.

### When to Use Multi-Leader

1. **Multi-datacenter operation:** Each DC has a leader for local writes
2. **Offline-capable apps:** Each device is effectively a leader (think mobile apps)
3. **Collaborative editing:** Google Docs-style real-time editing

### Use Cases

```
┌─────────────────────────────────────────────────────────────┐
│                Multi-Datacenter Setup                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   US-East DC                     EU-West DC                │
│   ┌─────────────┐               ┌─────────────┐            │
│   │   Leader    │◄─────────────►│   Leader    │            │
│   │    (US)     │   async       │    (EU)     │            │
│   └──────┬──────┘   replication └──────┬──────┘            │
│          │                             │                    │
│     ┌────┴────┐                   ┌────┴────┐              │
│     │Followers│                   │Followers│              │
│     └─────────┘                   └─────────┘              │
│                                                             │
│   Benefits:                                                │
│   - Low latency (write to local DC)                        │
│   - Tolerates DC outage                                    │
│                                                             │
│   Also used for:                                           │
│   - Offline-capable apps (each device is a "leader")       │
│   - Collaborative editing (Google Docs)                    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### The Big Problem: Write Conflicts

If two users edit the same document at the same time at different leaders, which version wins?

```
┌─────────────────────────────────────────────────────────────┐
│                    Conflict Example                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   User edits title on two devices simultaneously:          │
│                                                             │
│   Leader 1: title = "B"   Leader 2: title = "C"           │
│        │                       │                           │
│        └───────────────────────┘                           │
│                    ▼                                        │
│              Conflict! Which wins?                         │
│                                                             │
└─────────────────────────────────────────────────────────────┘

Resolution Strategies:

1. Last Write Wins (LWW)
   - Attach timestamp, highest wins
   - Simple but LOSES DATA
   
2. Merge values
   - title = "B/C" or ["B", "C"]
   - User resolves manually

3. Custom conflict resolution
   - Application-specific logic
   - On write (immediate) or on read (lazy)

4. CRDTs (Conflict-free Replicated Data Types)
   - Data structures that auto-merge mathematically
   - G-Counter, OR-Set, LWW-Register
```

> 💡 **Interview Tip:** Last Write Wins (LWW) is simple but dangerous - it silently loses data. Always mention this trade-off. In interviews, suggest CRDTs or application-level resolution for important data.

### CRDT Example: G-Counter

```
┌─────────────────────────────────────────────────────────────┐
│            G-Counter (Grow-only Counter)                     │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   Each node maintains its own count:                       │
│                                                             │
│   Node A: {A: 5, B: 0, C: 0}  (A incremented 5 times)     │
│   Node B: {A: 0, B: 3, C: 0}  (B incremented 3 times)     │
│   Node C: {A: 0, B: 0, C: 2}  (C incremented 2 times)     │
│                                                             │
│   Merge: Take max of each:                                 │
│   {A: 5, B: 3, C: 2} → Total = 10                         │
│                                                             │
│   No conflicts possible!                                   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Where CRDTs are used:** Riak (database), Figma (collaborative design), Redis (CRDT-based features).

---

## Leaderless Replication

### What it is
No leader at all! Clients write to multiple nodes directly and read from multiple nodes, using quorums to ensure consistency.

### Why it exists
Maximum availability and fault tolerance. No single point of failure, no failover process needed.

### Quorum Reads and Writes

**The key insight:** If you write to enough nodes and read from enough nodes, at least one node will have the latest data.

```
┌─────────────────────────────────────────────────────────────┐
│                    Quorum Formula                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   n = total nodes                                          │
│   w = write quorum (nodes that must ack write)             │
│   r = read quorum (nodes to read from)                     │
│                                                             │
│   Rule: w + r > n  (guarantees overlap)                    │
│                                                             │
│   Example: n=3, w=2, r=2                                   │
│                                                             │
│   Write to nodes:     Read from nodes:                     │
│   ┌───┐ ┌───┐ ┌───┐   ┌───┐ ┌───┐ ┌───┐                  │
│   │ A │ │ B │ │ C │   │ A │ │ B │ │ C │                  │
│   │ ✓ │ │ ✓ │ │   │   │ ✓ │ │   │ │ ✓ │                  │
│   └───┘ └───┘ └───┘   └───┘ └───┘ └───┘                  │
│                                                             │
│   Node A is in both → guarantees seeing latest write       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Why w + r > n Works
If you write to w nodes and read from r nodes, and w + r > n, there's always at least one node that was both written to AND read from. That node has the latest data.

> 💡 **Interview Tip:** Be able to explain w + r > n intuitively. Draw the Venn diagram: write set overlaps with read set.

### Common Configurations

| Config | w | r | Behavior |
|--------|---|---|----------|
| n=3, w=2, r=2 | 2 | 2 | Balanced (typical) |
| n=3, w=3, r=1 | 3 | 1 | Fast reads, slow writes |
| n=3, w=1, r=3 | 1 | 3 | Fast writes, slow reads |

**Cassandra/DynamoDB typical:** n=3, w=2, r=2 (majority quorum)

### Read Repair & Anti-Entropy

**Read Repair:** When a client reads stale data from a node, it writes back the latest version to repair it.

```
┌─────────────────────────────────────────────────────────────┐
│                     Read Repair                              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   Client reads from A, B, C                                │
│                                                             │
│   A: version 5  ← stale                                    │
│   B: version 7  ← latest                                   │
│   C: version 7  ← latest                                   │
│                                                             │
│   Client returns version 7, AND writes version 7 to A      │
│   (repairs the stale replica)                              │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Anti-Entropy Process:** Background daemon that continuously compares replicas and repairs differences. Uses Merkle trees for efficient comparison.

### Sloppy Quorums & Hinted Handoff

**What it is:** When the "home" nodes for a key are unavailable, temporarily write to other nodes. When home nodes recover, hand off the data.

**Trade-off:** Higher availability but you might miss recent writes when reading (until handoff completes).

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│   Normal: Key X → Nodes [A, B, C]                          │
│                                                             │
│   If A is down:                                            │
│   Strict quorum: Write fails (can't reach w=2 of ABC)     │
│   Sloppy quorum: Write to D instead, D hints it's for A   │
│                                                             │
│   When A recovers: D hands off data to A (hinted handoff) │
│                                                             │
│   Trade-off: Higher availability, but reads might miss     │
│              data until handoff completes                  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Version Vectors

### Detecting Concurrent Writes

```
┌─────────────────────────────────────────────────────────────┐
│                   Version Vectors                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   Each node tracks versions per-key:                       │
│                                                             │
│   Client 1 writes via Node A:                              │
│   Key: cart, Value: [milk], Version: {A:1}                │
│                                                             │
│   Client 2 writes via Node B (concurrent):                 │
│   Key: cart, Value: [eggs], Version: {B:1}                │
│                                                             │
│   Neither version dominates → CONFLICT                     │
│   {A:1} vs {B:1} - incomparable                           │
│                                                             │
│   After merge by client:                                   │
│   Key: cart, Value: [milk,eggs], Version: {A:1, B:1}      │
│                                                             │
│   Later write overwrites (dominates):                      │
│   Key: cart, Value: [milk,eggs,bread], Version: {A:2,B:1} │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Practical Configuration Examples

### PostgreSQL Streaming Replication

```ini
# postgresql.conf (primary)
wal_level = replica
max_wal_senders = 3
synchronous_standby_names = 'standby1'  # sync replication

# recovery.conf (standby)  
standby_mode = 'on'
primary_conninfo = 'host=primary port=5432'
```

### MongoDB Replica Set

```javascript
// Initialize replica set
rs.initiate({
  _id: "myReplicaSet",
  members: [
    { _id: 0, host: "mongo1:27017", priority: 2 },  // Preferred primary
    { _id: 1, host: "mongo2:27017", priority: 1 },
    { _id: 2, host: "mongo3:27017", priority: 1 }
  ]
});

// Write concern for durability
db.collection.insertOne(
  { name: "test" },
  { writeConcern: { w: "majority", wtimeout: 5000 } }
);
```

---

## Key Takeaways

1. **Single-leader** is simplest, use unless you need multi-DC or offline support
2. **Async replication** = possible data loss, but better performance
3. **Replication lag** causes subtle bugs (read-your-writes, monotonic reads)
4. **Multi-leader** requires conflict resolution - avoid if possible
5. **Leaderless** (Dynamo-style) good for high availability, but quorum limitations
6. **Failover is hard** - test it regularly!

---

## Quick Reference

| Need | Replication Strategy | Example |
|------|---------------------|---------|
| Simple reads scaling | Single-leader + read replicas | PostgreSQL |
| Multi-DC writes | Multi-leader | CockroachDB |
| Maximum availability | Leaderless with sloppy quorum | Cassandra |
| Strong consistency | Single-leader, sync replication | PostgreSQL sync |
| Offline-first apps | Multi-leader (each device) | CouchDB |

---

## System Design Interview Tips

### How to Choose Replication Strategy

**Start with single-leader** (90% of systems). Only consider alternatives when:
- You need multi-DC with local writes → Multi-leader
- You need offline capability → Multi-leader
- Availability is more important than consistency → Leaderless

### Common Mistakes to Avoid

1. **Over-engineering:** Don't use multi-leader for a single-region app
2. **Ignoring replication lag:** Always think about stale reads
3. **Assuming failover is automatic:** It's never as smooth as you think
4. **Forgetting about conflicts:** Multi-leader without conflict strategy = data corruption

### What Interviewers Look For

- Understanding of trade-offs (not just listing options)
- Ability to match solution to requirements
- Knowledge of real-world examples (PostgreSQL, Cassandra, DynamoDB)
- Awareness of failure modes

---

## Common Interview Questions

**Q: How would you handle reads scaling in a social media app?**
> Single-leader with read replicas. Route writes to leader, distribute reads across followers. Implement read-your-writes for profile updates using session stickiness or leader reads for recent changes.

**Q: User complains they don't see their own changes. What's happening?**
> Likely read-your-writes issue. They wrote to leader but read from a stale replica. Fix: Route reads to leader for X seconds after a write, or track the write timestamp and ensure replica is caught up.

**Q: How does Cassandra achieve high availability?**
> Leaderless replication with quorum writes/reads. No single point of failure. Uses sloppy quorums and hinted handoff to stay available even when nodes are down.

**Q: What happens during failover in PostgreSQL?**
> 1) Detect leader failure via timeout. 2) Elect most up-to-date follower. 3) Reconfigure clients to write to new leader. Risks: split brain if old leader comes back, lost writes if async replication.

**Q: When would you choose DynamoDB over PostgreSQL?**
> DynamoDB when you need: automatic scaling, global distribution, or very high write throughput. PostgreSQL when you need: complex queries, transactions, or strong consistency guarantees.
