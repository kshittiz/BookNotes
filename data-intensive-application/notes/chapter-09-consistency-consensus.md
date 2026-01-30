# Chapter 9: Consistency and Consensus

## Core Concept

> "Consensus is one of the most important and fundamental problems in distributed computing."

**The Goal:** Get all nodes to agree on something, despite failures.

**Why This Matters:** In distributed systems, nodes can crash, messages can be lost, and networks can partition. Without consensus, you can't build reliable leader election, distributed locks, or consistent replication. This chapter covers the fundamental building blocks that tools like ZooKeeper, etcd, and Kafka use internally.

---

## Consistency Models Hierarchy

**What it is:** Different levels of guarantees about what values clients will see when reading data from a distributed system.

**Why it matters:** Stronger consistency = easier to reason about, but higher latency and lower availability. You need to pick the right trade-off for your use case.

```
┌─────────────────────────────────────────────────────────────┐
│       Consistency Models (Strongest → Weakest)               │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   Linearizability (Strongest)                              │
│   └── As if single copy of data                            │
│       └── Real-time ordering                               │
│       └── Use: Locks, unique IDs, leader election          │
│                                                             │
│   Sequential Consistency                                   │
│   └── Operations in some total order                       │
│       └── Consistent with program order per process        │
│       └── Use: Cache coherence in CPUs                     │
│                                                             │
│   Causal Consistency                                       │
│   └── Causally related operations ordered                  │
│       └── Concurrent operations may reorder                │
│       └── Use: Social feeds, collaborative editing         │
│                                                             │
│   Eventual Consistency (Weakest)                           │
│   └── All replicas eventually converge                     │
│       └── No ordering guarantees during convergence        │
│       └── Use: Shopping carts, view counts, DNS            │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

> **Interview Tip:** Don't say "we need strong consistency" without explaining WHY. Most data doesn't need linearizability—user profiles, product catalogs, and analytics can use eventual consistency. Reserve strong consistency for things like payments, inventory counts, and unique constraints.

---

## Linearizability

### What It Is

**Simple definition:** The system behaves as if there's only ONE copy of the data, and every operation happens instantaneously at some point between its start and end time.

**Why it matters:** It's the gold standard for consistency—clients can't see stale data, and concurrent operations behave predictably. It's what you intuitively expect a database to do.

```
┌─────────────────────────────────────────────────────────────┐
│                    Linearizability                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   "Make the system APPEAR as if there's only one copy"     │
│                                                             │
│   Properties:                                              │
│   1. Once write completes, all reads see it                │
│   2. Once read returns value, later reads can't see older  │
│   3. Operations take effect at some instant (not duration) │
│                                                             │
│   Client A:  |--write(x,1)--|                              │
│   Client B:       |----read(x)----| → must return 1        │
│   Client C:            |----read(x)----| → must return 1   │
│                                                             │
│   Linearization point: moment operation "takes effect"     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

> **Interview Tip:** Linearizability ≠ Serializability. Linearizability is about single-object reads/writes appearing instantaneous. Serializability is about multi-object transactions appearing to execute one at a time. "Strict Serializability" = both combined.

### Non-Linearizable Example

**Why this breaks:** With async replication, followers lag behind the leader. If clients read from different replicas, they can see inconsistent views of the world.

```
┌─────────────────────────────────────────────────────────────┐
│              Replication Lag Breaks Linearizability          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   Leader                    Follower                        │
│   ┌──────┐                  ┌──────┐                       │
│   │ x=1  │ ──replicating──► │ x=0  │  (stale)             │
│   └──────┘                  └──────┘                       │
│                                                             │
│   Client A writes x=1 to leader                            │
│   Client B reads x from follower → gets 0!                 │
│                                                             │
│   Client A and B disagree on current value                 │
│   = NOT LINEARIZABLE                                       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Fix options:**
1. Always read from leader (but defeats purpose of replicas)
2. Use synchronous replication (but kills performance)
3. Accept eventual consistency (but changes semantics)

### When You Actually Need Linearizability

```
1. Lock/Lease
   - Only ONE client should hold lock at a time
   - If two clients both think they have the lock → disaster
   - Example: Only one worker processes each job
   
2. Unique Constraints
   - Username must be unique
   - Two users register "bob" simultaneously
   - Must reject one, accept one (deterministically)
   - Example: Stripe charge IDs, order numbers
   
3. Cross-Channel Coordination
   - User uploads image, sends link via separate channel
   - Recipient must be able to see the image
   - Example: Image service + notification service
```

> **Interview Tip:** In system design, explicitly state: "This component needs linearizability because [specific reason]." For example: "The inventory service needs linearizability to prevent overselling—if we sell the last item to two customers simultaneously, one must fail."

### The Cost of Linearizability

Linearizability is expensive:
- **Latency:** Must wait for consensus/replication before acknowledging writes
- **Availability:** During network partitions, must choose to either reject requests (preserve consistency) or serve stale data (preserve availability)
- **Throughput:** Single point of serialization limits parallelism

---

## CAP Theorem

### What It Really Means

**The Theorem:** During a network partition, you must choose between Consistency and Availability. You can't have both.

**What it DOESN'T mean:** "Pick 2 of 3" is misleading. Network partitions WILL happen (you don't get to opt out of P). So the real choice is: what do you do DURING a partition?

**CAP Theorem:**
```
┌─────────────────────────────────────────────────────────────┐
│                    CAP Theorem                               │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   During a network partition, choose:                       │
│                                                             │
│   CP (Consistency + Partition tolerance):                  │
│   • Refuse requests to partitioned nodes                   │
│   • Return errors rather than stale data                   │
│   • Examples: ZooKeeper, etcd, HBase, Spanner              │
│   • Use for: Coordination, locks, leader election          │
│                                                             │
│   AP (Availability + Partition tolerance):                 │
│   • Serve requests from all nodes                          │
│   • Accept that responses may be stale/inconsistent        │
│   • Examples: Cassandra, DynamoDB, CouchDB                 │
│   • Use for: Shopping carts, user preferences, caches      │
│                                                             │
│   When there's NO partition: You get both C and A!         │
│   CAP only matters during failures.                        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

> **Interview Tip:** CAP is often misunderstood. Don't say "Cassandra is AP so it's not consistent." Say: "Cassandra prioritizes availability during partitions, but offers tunable consistency (e.g., QUORUM reads/writes) that provides strong consistency when the network is healthy."

---

## Ordering Guarantees

### Total Order vs Partial Order

**Why this matters:** In distributed systems, events don't have a single timeline. Understanding ordering helps you reason about what guarantees your system provides.

```
┌─────────────────────────────────────────────────────────────┐
│              Total Order vs Partial Order                    │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   Total Order:                                             │
│   • Every pair of events can be compared (A < B or B < A)  │
│   • Single timeline: A → B → C → D                         │
│   • Example: Sequence numbers, single-leader replication   │
│                                                             │
│   Partial Order:                                           │
│   • Some events are concurrent (incomparable)              │
│   • Forms a DAG, not a line                                │
│   • Example: Multi-leader replication, Git branches        │
│                                                             │
│        A ─────► C                                          │
│         \     /                                             │
│          \   /                                              │
│           ▼ ▼                                               │
│            D                                                │
│         /                                                   │
│        ▼                                                    │
│        B       (A and B are concurrent/incomparable)       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Lamport Timestamps

**What it is:** A logical clock that gives every event a unique number, creating a total order across all nodes without synchronized physical clocks.

**Why it works:** By incrementing on each event and taking max(local, received)+1 on message receipt, causal ordering is preserved—if A caused B, A's timestamp < B's timestamp.

```
┌─────────────────────────────────────────────────────────────┐
│                 Lamport Timestamps                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   Each node maintains counter, increments on each event    │
│                                                             │
│   Node A          Node B          Node C                   │
│   ──────          ──────          ──────                   │
│   A.1 ─────────────────────────────►                       │
│                   B.2             C.3                       │
│         ◄─────────────────────────────                     │
│   A.4                                                       │
│                   B.5                                       │
│                                                             │
│   Rules:                                                   │
│   1. Increment counter before each event                   │
│   2. Send counter with messages                            │
│   3. On receive: max(local, received) + 1                  │
│                                                             │
│   Result: (counter, nodeId) gives total order              │
│   Limitation: Can't tell if events were concurrent         │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Vector Clocks

**What it is:** Like Lamport timestamps, but tracks a counter for EACH node. This lets you detect concurrent events (not just causal ones).

**When to use:** Conflict detection in multi-leader/leaderless replication (e.g., Dynamo-style databases).

```
┌─────────────────────────────────────────────────────────────┐
│                   Vector Clocks                              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   Each node maintains vector of counters for all nodes     │
│                                                             │
│   Node A: [A:1, B:0, C:0] → [A:2, B:1, C:0]               │
│   Node B: [A:0, B:1, C:0] → [A:1, B:2, C:0]               │
│   Node C: [A:0, B:0, C:1]                                  │
│                                                             │
│   Comparison:                                              │
│   [A:2, B:1] happens-before [A:3, B:1] (all ≤, one <)     │
│   [A:2, B:1] concurrent-with [A:1, B:2] (neither ≤)       │
│                                                             │
│   Use cases:                                               │
│   • Detecting concurrent writes → need conflict resolution │
│   • Example: Riak, Amazon Dynamo                           │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

> **Interview Tip:** If asked "how do you handle concurrent writes?", vector clocks are one answer. But in practice, many systems use simpler approaches: last-writer-wins (LWW), application-level conflict resolution, or CRDTs.

---

## Total Order Broadcast

### What It Is

**Simple definition:** A way to deliver messages to all nodes in the SAME order. If Node A sees [m1, m2, m3], every other node sees [m1, m2, m3] in that exact order.

**Why it matters:** This is how you implement consistent replication. If all replicas apply the same operations in the same order, they stay in sync. It's also equivalent to consensus!

### Definition

```
┌─────────────────────────────────────────────────────────────┐
│                Total Order Broadcast                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   Properties:                                              │
│   1. Reliable delivery: If one node gets message,          │
│      all nodes eventually get it                           │
│   2. Totally ordered: All nodes see messages in SAME order │
│                                                             │
│   Node A sees: [m1, m2, m3, m4]                           │
│   Node B sees: [m1, m2, m3, m4]  (same order!)            │
│   Node C sees: [m1, m2, m3, m4]                           │
│                                                             │
│   Real-world examples:                                     │
│   • Kafka partition = total order for messages in partition│
│   • Raft log = total order for state machine commands      │
│   • Database replication log                               │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Equivalence to Consensus

This is a key insight: **Total Order Broadcast and Consensus are equivalent problems.** If you can solve one, you can solve the other.

```
Total Order Broadcast ⟺ Consensus

Consensus → TOB:
  - For each message, run consensus to decide its position
  - All nodes agree on position → same order

TOB → Consensus:
  - Each node broadcasts its proposed value
  - First value delivered to everyone = the decided value
```

> **Interview Tip:** When someone asks "how does Kafka maintain message ordering?" the answer involves total order broadcast within a partition. The partition leader sequences all messages, and consumers see them in that order.

---

## Consensus Algorithms

### The Problem

**What we need:** N nodes must agree on a single value, even if some nodes crash or messages are lost.

**Why it's hard:** Without a central coordinator, how do you know when "enough" nodes have agreed? What if the network partitions? What if the node you thought was the leader actually crashed?

```
┌─────────────────────────────────────────────────────────────┐
│                  Consensus Problem                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   Get N nodes to agree on a value, despite:                │
│   • Nodes may crash (and maybe recover)                    │
│   • Network may drop/delay/reorder messages                │
│   • No synchronized clocks                                 │
│                                                             │
│   Properties required:                                     │
│   1. Agreement: All deciding nodes decide same value       │
│   2. Integrity: Decide at most once                        │
│   3. Validity: Decided value was proposed by some node     │
│   4. Termination: All non-crashed nodes eventually decide  │
│                                                             │
│   FLP Impossibility Theorem:                               │
│   No algorithm can GUARANTEE termination in an async       │
│   system with even ONE faulty node. But practical          │
│   algorithms (Raft, Paxos) work well in practice by        │
│   using timeouts and leader election.                      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Raft Consensus

**What it is:** A consensus algorithm designed to be understandable. It elects a leader who sequences all operations, ensuring all nodes see the same order.

**Why Raft over Paxos:** Paxos is notoriously hard to understand and implement correctly. Raft breaks down the problem into leader election, log replication, and safety—making it easier to reason about.

**Where it's used:** etcd, Consul, CockroachDB, TiKV

```
┌─────────────────────────────────────────────────────────────┐
│                    Raft Overview                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   Three roles: Leader, Follower, Candidate                 │
│                                                             │
│   Normal operation:                                        │
│   ┌────────┐     ┌──────────┐     ┌──────────┐            │
│   │ Client │────►│  Leader  │────►│Followers │            │
│   └────────┘     └──────────┘     └──────────┘            │
│                       │                 │                  │
│                       └────────────────►│ replicate        │
│                       │◄────────────────┤ ack              │
│                       │                                    │
│                  committed when majority ack               │
│                                                             │
│   Leader election (when leader times out):                 │
│   1. Follower becomes Candidate (increments term)          │
│   2. Votes for self, requests votes from others            │
│   3. Wins election if gets majority of votes               │
│   4. Becomes leader, sends heartbeats to maintain lead     │
│                                                             │
│   Key insight: Only ONE leader per term (epoch number)     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Raft Log Replication

**How writes work:**
1. Client sends write to leader
2. Leader appends to its log
3. Leader sends AppendEntries to all followers
4. Once majority acknowledge, entry is "committed"
5. Leader responds to client
6. Followers learn commit on next heartbeat

```
┌─────────────────────────────────────────────────────────────┐
│                  Raft Log Replication                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   Leader Log:    [1: x=1] [2: y=2] [3: x=3] [4: z=4]      │
│   Follower A:    [1: x=1] [2: y=2] [3: x=3]               │
│   Follower B:    [1: x=1] [2: y=2]                         │
│                                                             │
│   Commit index: 3 (majority have entries 1-3)              │
│                                                             │
│   Safety: A follower with an incomplete log can't become   │
│   leader—voters reject candidates with less up-to-date     │
│   logs. This ensures committed entries are never lost.     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

> **Interview Tip:** You don't need to implement Raft from scratch—use etcd or Consul. But understand: (1) leader election via terms and majority vote, (2) log replication with majority acknowledgment, (3) why a 5-node cluster tolerates 2 failures.

### Paxos (Classic)

**What it is:** The original consensus algorithm. More flexible than Raft but harder to understand.

**In practice:** You'll rarely implement Paxos directly. Google's Chubby and Spanner use it internally, but most systems use Raft-based tools (etcd, Consul).

```
┌─────────────────────────────────────────────────────────────┐
│                    Paxos Phases                              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   Phase 1: Prepare (get permission to propose)             │
│   Proposer → Acceptors: "I want to propose with ballot n"  │
│   Acceptors → Proposer: "OK" or "here's what I accepted"   │
│                                                             │
│   Phase 2: Accept (propose a value)                        │
│   Proposer → Acceptors: "Accept value V with ballot n"     │
│   Acceptors: Accept if n is still the highest seen         │
│                                                             │
│   Value is chosen when majority of acceptors accept it.    │
│                                                             │
│   Multi-Paxos: Run many rounds with stable leader to       │
│   amortize the cost of Phase 1 (similar to Raft)           │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Raft vs Paxos:**
- Raft: Strong leader, easier to understand, used in etcd/Consul
- Paxos: More flexible, harder to implement, used in Google's systems

---

## Distributed Transactions: Two-Phase Commit (2PC)

### What It Is

**The problem 2PC solves:** You have a transaction that spans multiple databases/services (e.g., transfer money from Bank A to Bank B). How do you ensure either BOTH commit or BOTH abort?

**How it works:** A coordinator asks all participants "can you commit?" If all say yes, coordinator says "commit." If any says no, coordinator says "abort."

### Two-Phase Commit (2PC)

```
┌─────────────────────────────────────────────────────────────┐
│                  Two-Phase Commit                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   Coordinator              Participant A    Participant B   │
│   ───────────              ─────────────    ─────────────   │
│   "Prepare?"  ─────────────────►                           │
│               ─────────────────────────────────►            │
│                                                             │
│               ◄───────────── "Ready"                       │
│               ◄──────────────────────────── "Ready"        │
│                                                             │
│   (If all ready:)                                          │
│   "Commit!"   ─────────────────►                           │
│               ─────────────────────────────────►            │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Why 2PC Is Problematic

**The fundamental problem:** Once a participant votes "yes" to prepare, it CANNOT change its mind. It must wait for the coordinator's decision. If the coordinator crashes, the participant is stuck holding locks indefinitely.

```
┌─────────────────────────────────────────────────────────────┐
│                    2PC Blocking Problem                      │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   Coordinator crashes after sending PREPARE                │
│                                                             │
│   Participant A: "I voted YES, waiting for decision..."    │
│   Participant B: "I voted YES, waiting for decision..."    │
│                                                             │
│   Neither can:                                             │
│   • Commit (what if coordinator decides ABORT?)            │
│   • Abort (what if coordinator decides COMMIT?)            │
│                                                             │
│   Must HOLD LOCKS until coordinator recovers!              │
│   → Can block for hours in production.                     │
│   → Blocked resources affect other transactions.           │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Alternatives to 2PC

| Approach | Trade-off |
|----------|-----------|
| **Saga pattern** | Compensating transactions; eventual consistency |
| **2PC with HA coordinator** | Coordinator uses Raft/Paxos internally |
| **Avoid distributed transactions** | Design services to not need them |

> **Interview Tip:** If asked "how do you handle transactions across microservices?", DON'T say "use 2PC." Say: "I'd try to avoid distributed transactions by redesigning service boundaries. If unavoidable, I'd use the Saga pattern with compensating transactions and accept eventual consistency."

---

## Coordination Services: ZooKeeper and etcd

### What They Are

**Simple definition:** ZooKeeper and etcd are distributed key-value stores that use consensus internally to provide strong consistency. They're designed for storing small amounts of critical data: configuration, leader election state, distributed locks.

**Why not use a regular database?** These services are specifically optimized for coordination patterns. They handle leader election, watches (notifications on changes), and ephemeral nodes (auto-deleted when client disconnects).

### ZooKeeper Use Cases

```
┌─────────────────────────────────────────────────────────────┐
│                    ZooKeeper Uses                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   1. Leader Election                                       │
│      Each node creates ephemeral sequential znode          │
│      Lowest sequence number = leader                       │
│      If leader dies, ephemeral node deleted → new election │
│                                                             │
│   2. Distributed Lock                                      │
│      Create ephemeral znode at /locks/resource             │
│      Success = got lock                                    │
│      Failure = watch existing znode, wait for deletion     │
│                                                             │
│   3. Service Discovery                                     │
│      Services register at /services/api/instance1          │
│      Clients watch /services/api for children changes      │
│      Ephemeral nodes auto-delete when service crashes      │
│                                                             │
│   4. Configuration Management                              │
│      Store config at /config/database/connection_string    │
│      Services watch for changes, hot-reload on update      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### ZooKeeper vs etcd

| Feature | ZooKeeper | etcd |
|---------|-----------|------|
| Consensus | ZAB (Paxos-like) | Raft |
| Data Model | Hierarchical (like filesystem) | Flat key-value |
| Watches | One-time triggers (must re-register) | Streaming (continuous) |
| Language | Java | Go |
| Typical Use | Hadoop, Kafka, HBase | Kubernetes, CoreDNS |

> **Interview Tip:** When designing systems that need leader election, distributed locks, or service discovery, say "I'd use etcd (or ZooKeeper) rather than building my own." Then explain WHY: "Building consensus correctly is extremely hard—bugs in consensus algorithms have caused major outages. Battle-tested implementations have years of bug fixes."

---

## Practical Examples

### Implementing Distributed Lock with etcd

```python
import etcd3

client = etcd3.client()

# Acquire lock with TTL (lease) - auto-releases if we crash
lease = client.lease(ttl=30)
lock_acquired = client.put('/locks/my-resource', 'holder-1', 
                           lease=lease, 
                           prevExist=False)  # Only succeed if key doesn't exist

if lock_acquired:
    try:
        # Keep refreshing lease while doing work
        lease.refresh()
        # Do work that requires mutual exclusion...
    finally:
        client.delete('/locks/my-resource')
```

**Why the lease matters:** If your process crashes, the lease expires and the lock is automatically released. Without this, you'd have a "dead" lock blocking everyone forever.

### Leader Election with etcd

```python
import etcd3

client = etcd3.client()
election = etcd3.Election(client, '/elections/my-service')

# Campaign to become leader - blocks until we win
with election.leader():
    print("I'm the leader now!")
    # Only ONE instance reaches here at a time
    # Do leader-only work: assign tasks, run cron jobs, etc.
# Leadership auto-released when context exits
```

---

## Key Takeaways

1. **Linearizability** is strong but expensive (latency, availability). Use only when you truly need it: locks, unique constraints, leader election.
2. **CAP trade-off** is real—during network partitions, you choose consistency OR availability. Most systems can tolerate eventual consistency for most data.
3. **Consensus** is fundamental but hard. Raft and Paxos solve it; ZAB (ZooKeeper) is similar.
4. **2PC** is blocking and dangerous for availability. Prefer Saga pattern or avoid distributed transactions entirely.
5. **Use existing implementations** (etcd, ZooKeeper, Consul) rather than building your own consensus.
6. **Raft** is easier to understand than Paxos and provides the same guarantees.

---

## Quick Reference

| Need | Solution |
|------|----------|
| Distributed lock | etcd/ZooKeeper with lease/ephemeral node |
| Leader election | etcd election / ZK sequential nodes |
| Config management | etcd / ZooKeeper with watch |
| Consistent metadata | Single-leader DB (PostgreSQL) or consensus service |
| Cross-service transactions | Saga pattern (NOT 2PC) |
| Total ordering | Kafka partitions or Raft log |

---

## System Design Interview Tips

### When to Bring Up These Concepts

- **"How do you ensure only one worker processes each job?"** → Leader election or distributed lock via etcd
- **"What if two users try to register the same username?"** → Linearizable uniqueness constraint (single DB or consensus)
- **"How do services discover each other?"** → Service registry with ZooKeeper/etcd/Consul
- **"How do you handle transactions across services?"** → Saga pattern, NOT 2PC

### Common Interview Questions

1. **"What's the difference between linearizability and eventual consistency?"**
   - Linearizable: reads always see the most recent write; behaves like single copy
   - Eventually consistent: reads may see stale data; replicas converge over time
   - Trade-off: linearizable has higher latency and lower availability

2. **"Explain CAP theorem"**
   - During network partition, choose Consistency (reject requests) or Availability (serve possibly stale data)
   - NOT "pick 2 of 3"—partitions happen, so you're really choosing C or A during failures

3. **"How does Raft work at a high level?"**
   - Elect leader via majority vote, using term numbers to prevent split-brain
   - Leader sequences all operations, replicates to followers
   - Operation committed when majority acknowledge
   - If leader fails, new election with higher term

4. **"When would you use ZooKeeper/etcd?"**
   - Leader election, distributed locks, service discovery, configuration management
   - Small amounts of critical coordination data (NOT bulk storage)
   - When you need strong consistency and can't build your own consensus

5. **"Why is 2PC problematic?"**
   - Coordinator crash leaves participants blocked, holding locks indefinitely
   - Single point of failure; violates availability
   - Alternative: Saga pattern with compensating transactions
