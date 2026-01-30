# Chapter 11: Stream Processing

## Core Concept

> "Stream processing is batch processing where the input is unbounded."

**What it is:** Processing data continuously as it arrives, rather than waiting to collect a complete dataset.

**Why it matters:** Real-time analytics, fraud detection, live dashboards - any time you need results in seconds, not hours.

```
Batch:   [────────── Finite input ──────────] → Process → Output
Stream:  ─────────► Continuous input ─────────► Process ─► Output
                    (never ends)
```

---

## Messaging Systems

### Message Brokers vs Direct Messaging

**What it is:** A message broker is a middleman that receives, stores, and delivers messages between producers and consumers.

**Why use a broker instead of direct messaging?**
- **Decoupling:** Producer doesn't need to know who consumes
- **Buffering:** Handles bursts when consumers are slow
- **Persistence:** Messages survive restarts
- **Fanout:** One message, many consumers

```
Direct (TCP/UDP):     Producer ──────────────► Consumer
                      (Consumer must be online!)

Message Broker:       Producer ──► [Broker] ──► Consumer
                      (Broker buffers if consumer is down)
```

> **💡 Interview Tip:** When asked "How do microservices communicate?", mention both sync (REST/gRPC) and async (message broker) patterns. Async is better for decoupling and handling failures.

---

## Kafka vs RabbitMQ: Log vs Queue

This is one of the **most asked** messaging questions in system design interviews.

### Traditional Message Queues (RabbitMQ, SQS)

**What it is:** A queue where messages are deleted after being consumed. Think of a task list that gets checked off.

**How it works:**
- Message goes to **one** consumer (competing consumers pattern)
- After consumer ACKs, message is **deleted**
- No way to replay old messages

```
┌────────────────────────────────────────────────────────────┐
│  Queue: [msg1][msg2][msg3][msg4]                           │
│         ─────────┬───────────────────┬────────             │
│                  ▼                   ▼                     │
│            Consumer 1           Consumer 2                 │
│            (gets msg1)          (gets msg2)                │
│                                                            │
│  After ACK: msg1 and msg2 are GONE forever                │
└────────────────────────────────────────────────────────────┘
```

**When to use RabbitMQ/SQS:**
- Task distribution (background jobs, email sending)
- Work queues where order doesn't matter
- When you don't need replay capability

### Log-Based Message Brokers (Kafka)

**What it is:** An append-only log where messages are retained. Think of a commit log that you can replay.

**How it works:**
- Messages written to **partitions** (shards)
- Multiple **consumer groups** can read the same data independently
- Consumers track their **offset** (position in log)
- Messages retained for configurable time (not deleted on read)

```
┌────────────────────────────────────────────────────────────┐
│  Topic: orders                                             │
│  Partition 0: [0][1][2][3][4][5]►                          │
│  Partition 1: [0][1][2][3]►                                │
│  Partition 2: [0][1][2][3][4][5][6][7]►                    │
│                                                            │
│  Consumer Group A:              Consumer Group B:          │
│  • Consumer 1 → Partition 0     • Consumer X → All         │
│  • Consumer 2 → Partition 1,2   (separate offset!)         │
│                                                            │
│  Both groups read ALL messages independently               │
└────────────────────────────────────────────────────────────┘
```

**Why partitions?**
- **Parallelism:** Each partition processed by one consumer in a group
- **Ordering:** Messages within a partition are ordered
- **Scalability:** Add partitions to scale throughput

> **💡 Interview Tip:** "Kafka guarantees ordering within a partition, not across partitions. If you need global ordering, use one partition - but you lose parallelism."

### Kafka vs RabbitMQ Decision Matrix

| Need | Choose | Why |
|------|--------|-----|
| Task queue (jobs) | RabbitMQ | Delete after processing |
| Event streaming | Kafka | Retain, replay, multiple consumers |
| Strict ordering | Kafka (1 partition) | Guaranteed order |
| High throughput | Kafka | 1M+ msg/sec |
| Complex routing | RabbitMQ | Flexible exchange patterns |
| Replay old events | Kafka | Retention + offset seeking |
| Multiple teams consuming same events | Kafka | Consumer groups |

> **💡 Interview Tip:** "For an event-driven architecture where multiple services need the same events (orders, user signups), I'd use Kafka. For background job processing where tasks should be done exactly once and forgotten, I'd use RabbitMQ or SQS."

---

## Event Time vs Processing Time

**What it is:**
- **Event time:** When the event actually happened (timestamp in the event)
- **Processing time:** When your system processes the event

**Why it matters:** Events arrive out of order and late. If you use processing time for analytics, your results will be wrong.

```
Scenario: Mobile app user clicks offline

10:00 - User clicks button (device offline) ← Event time
10:05 - User clicks again (still offline)
11:00 - Device reconnects, sends both events
11:01 - Your system processes them ← Processing time

If you count "clicks per hour by processing time":
  - 10:00-11:00: 0 clicks (WRONG!)
  - 11:00-12:00: 2 clicks

If you count "clicks per hour by event time":
  - 10:00-11:00: 2 clicks (CORRECT!)
```

> **💡 Interview Tip:** "For any analytics or metrics, use event time. Processing time only makes sense for operational metrics like 'how fast is our pipeline?'"

---

## Windows: Aggregating Unbounded Data

**What it is:** Windows group events into finite chunks for aggregation. Without windows, you can't compute "sum of orders in last hour" on an infinite stream.

### Window Types with Examples

**Tumbling Window** (fixed, non-overlapping)
```
Time: 0──────5──────10──────15──────20
      ├──────┤├──────┤├──────┤├──────┤
      Window1 Window2 Window3 Window4
```
**Example:** "Orders per 5-minute period" - each order counted in exactly one window.

**Hopping Window** (fixed, overlapping)
```
Time: 0──────5──────10──────15──────20
      ├───────────────┤
            ├───────────────┤
                  ├───────────────┤
      (10 min window, 5 min hop)
```
**Example:** "5-minute moving average updated every minute" - each event counted in multiple windows.

**Session Window** (gap-based, variable size)
```
Events: *  *  * * *          * *    * * * *
        ├────────────┤       ├──┤   ├─────────┤
         Session 1          Sess2    Session 3
        (ends after 30min of no activity)
```
**Example:** "User sessions on website" - new session after 30 min inactivity.

> **💡 Interview Tip:** When designing analytics, always clarify: "Do you want tumbling (distinct periods) or sliding (moving average)? How do you want to handle late events?"

---

## Watermarks and Late Events

**What it is:** A watermark is the system's estimate of "we've seen all events up to time T." It's how the system knows when it's safe to close a window.

**Why it's hard:** Events arrive out of order. How long do you wait before declaring a window complete?

```
Events arriving:     [10:00][10:01][10:03][10:02][10:05]
                                           ↑
                                   Out of order!

Watermark: "I believe I've seen all events up to 10:03"

Window 10:00-10:05 can close when watermark > 10:05
But what if event with timestamp 10:04 arrives after?
```

**Handling late events - Three strategies:**
1. **Drop:** Ignore late events (simplest, lossy)
2. **Reprocess:** Update and re-emit the window result
3. **Side output:** Send late events to separate stream for special handling

> **💡 Interview Tip:** "Watermarks are a trade-off between latency and correctness. Wait longer = more correct but slower. In practice, you set an allowed lateness (e.g., 1 hour) and drop anything later."

---

## Stream Joins

### Stream-Stream Join

**What it is:** Joining two event streams based on a key within a time window.

**Example:** Match ad impressions with clicks within 1 hour.

```
Impressions:  [ad_123, 10:00] [ad_456, 10:05] [ad_123, 10:30]
Clicks:       [ad_123, 10:15] [ad_456, 10:45]

Join output:
  ad_123: impression@10:00 matched with click@10:15 ✓
  ad_456: impression@10:05 matched with click@10:45 ✓
```

**Challenge:** Must buffer events waiting for their match. Memory grows with window size!

### Stream-Table Join (Enrichment)

**What it is:** Enriching stream events with data from a table (e.g., adding user info to orders).

**Example:**
```
Order events: {order_id: 1, user_id: 123, amount: $50}
              ↓ join with
User table:   {user_id: 123, name: "Alice", tier: "gold"}
              ↓
Output:       {order_id: 1, user_id: 123, amount: $50, 
               name: "Alice", tier: "gold"}
```

**Three approaches:**
1. **Query database per event** - Simple but slow, hammers DB
2. **Load table into memory** - Fast but uses lots of memory
3. **CDC (recommended)** - Stream table changes, maintain local copy

> **💡 Interview Tip:** "For enrichment at scale, use CDC to maintain a local copy of the lookup table. This avoids DB roundtrips and handles table updates in real-time."

---

## Exactly-Once Semantics

**What it is:** Processing each message exactly once, even with failures and retries.

**The reality:** "Exactly-once" is marketing. What you actually get is "effectively once" through:
- At-least-once delivery (retries) + deduplication, OR
- Transactional processing (atomic commits)

```
┌─────────────────────────────────────────────────────────────┐
│  Delivery Guarantees                                        │
├─────────────────────────────────────────────────────────────┤
│  At-most-once:  Fire and forget. May lose messages.        │
│  At-least-once: Retry until ACK. May have duplicates.      │
│  Exactly-once:  At-least-once + deduplication/transactions │
└─────────────────────────────────────────────────────────────┘
```

### How Flink achieves exactly-once (Checkpointing)

```
Source → Map → Aggregate → Sink
  │        │        │
  └────────┴────────┘
           │
    Checkpoint barrier flows through
           ↓
    Save: source offset + all operator state
           ↓
    On failure: restore checkpoint, replay from offset
```

**Why it works:** By atomically saving the source offset and all processing state together, replay produces the same output.

### Idempotent Writes (The practical approach)

```python
# NON-IDEMPOTENT (dangerous - duplicates cause wrong results)
def process(event):
    count = db.get(event.user_id)
    db.set(event.user_id, count + 1)  # Replays increment again!

# IDEMPOTENT (safe - duplicates have no effect)
def process(event):
    if db.has_processed(event.id):
        return  # Already done
    db.transaction([
        ("INCR", f"count:{event.user_id}"),
        ("SET", f"processed:{event.id}", True)
    ])
```

> **💡 Interview Tip:** "In practice, I design for at-least-once delivery and make operations idempotent. Store event IDs you've processed, or use upserts with version numbers."

---

## Change Data Capture (CDC)

**What it is:** Capturing changes (inserts, updates, deletes) from a database and streaming them as events.

**Why it matters:** Turns your database into an event stream. Enables real-time sync to search, cache, analytics without changing application code.

```
┌─────────────────────────────────────────────────────────────┐
│  MySQL → Debezium → Kafka → Elasticsearch                   │
│                                                             │
│  1. App writes to MySQL (no changes needed)                │
│  2. Debezium reads MySQL binlog                            │
│  3. Changes streamed to Kafka topic                        │
│  4. Consumer updates Elasticsearch index                    │
│                                                             │
│  Result: Search index always up-to-date!                   │
└─────────────────────────────────────────────────────────────┘
```

**CDC event example (Debezium):**
```json
{
  "before": {"id": 1, "name": "Alice", "tier": "silver"},
  "after":  {"id": 1, "name": "Alice", "tier": "gold"},
  "op": "u",
  "source": {"table": "users", "db": "mydb"}
}
```

> **💡 Interview Tip:** "CDC is essential for keeping derived systems in sync. Instead of dual-writes (error-prone), write to one database and let CDC propagate changes."

---

## Event Sourcing and CQRS

### Event Sourcing

**What it is:** Store state as a sequence of events, not as current values.

```
Traditional:  user_123: {balance: 150}

Event Sourced:
  1. AccountCreated(user_123, balance=100)
  2. Deposited(user_123, amount=100)  
  3. Withdrawn(user_123, amount=50)
  
Current balance = replay events = 100 + 100 - 50 = 150
```

**Why use it:**
- Complete audit trail (what happened and when)
- Time travel (what was balance last Tuesday?)
- Rebuild views by replaying events
- Natural fit for streaming architectures

**When NOT to use:** Simple CRUD apps, when history doesn't matter, when events are huge.

### CQRS (Command Query Responsibility Segregation)

**What it is:** Separate write model from read models. Writes go to event store, reads come from optimized views.

```
         Commands (writes)
              │
              ▼
        ┌─────────────┐
        │ Event Store │
        │  (Kafka)    │
        └──────┬──────┘
               │
    ┌──────────┼──────────┐
    ▼          ▼          ▼
┌────────┐ ┌────────┐ ┌────────┐
│ View 1 │ │ View 2 │ │ View 3 │
│(Search)│ │(Reports│ │(API)   │
└────────┘ └────────┘ └────────┘
    ▲                     ▲
    └───── Queries ───────┘
```

**Why use CQRS:**
- Read and write models can scale independently
- Each view optimized for its queries
- Views can be rebuilt from events

> **💡 Interview Tip:** "Event sourcing + CQRS is great for domains with complex business rules and audit requirements (finance, healthcare). For simple CRUD, it's overkill."

---

## Lambda vs Kappa Architecture

### Lambda Architecture

**What it is:** Run both batch and stream processing, combine results.

```
            Raw Events
           /          \
          /            \
    Batch Layer      Speed Layer
   (complete,slow)  (recent,fast)
          \            /
           \          /
         Merge results
```

**Problem:** Maintaining two codebases (batch + stream) for same logic.

### Kappa Architecture

**What it is:** Stream processing only. Reprocess by replaying the log.

```
Raw Events → Stream Processing → Views
                  ↑
             (replay to reprocess)
```

**Why Kappa is often preferred:** Single codebase, simpler to maintain. With Kafka retention and Flink's reprocessing, you can handle both real-time and historical.

> **💡 Interview Tip:** "I'd start with Kappa (streaming only). Lambda adds complexity - only use it if you have fundamentally different batch and real-time requirements."

---

## System Design Interview Tips

### When to mention stream processing:
- Real-time analytics/dashboards
- Fraud detection (need immediate response)
- Event-driven microservices
- Keeping derived data in sync (CDC)
- IoT sensor data processing

### Key points interviewers want to hear:
1. **Kafka vs queue:** Know when to use each
2. **Ordering guarantees:** Partition-level only
3. **Exactly-once:** It's really at-least-once + idempotency
4. **Watermarks:** Trade-off between latency and completeness
5. **State management:** Where does aggregation state live?

### Common mistakes to avoid:
- Assuming global ordering in Kafka (it's per-partition)
- Forgetting about late events in analytics
- Not considering state size for windowed joins
- Treating exactly-once as magic (always think about failure modes)

---

## Common Interview Questions

**Q: When would you use Kafka vs RabbitMQ?**
A: Kafka for event streaming, replay, multiple consumers. RabbitMQ for task queues, complex routing, when messages should be deleted after processing.

**Q: How do you handle out-of-order events?**
A: Use event timestamps (not processing time), watermarks to track progress, and either drop late events or update results when they arrive.

**Q: What is exactly-once semantics?**
A: Processing each event exactly once despite failures. Achieved through checkpointing + atomic commits OR at-least-once + idempotent operations.

**Q: How would you design a real-time analytics dashboard?**
A: Kafka for ingestion → Flink for windowed aggregations → Redis/ClickHouse for serving. Use tumbling windows for distinct periods, handle late events with allowed lateness.

**Q: What's the difference between event sourcing and CDC?**
A: Event sourcing is an application design pattern (you write events). CDC captures changes from existing databases (passive observation). Both produce event streams.

---

## Quick Reference

| Pattern | Use Case |
|---------|----------|
| Kafka | Event streaming, replay, pub/sub |
| RabbitMQ/SQS | Task queues, background jobs |
| Tumbling windows | Fixed-period aggregations |
| Session windows | User activity tracking |
| CDC (Debezium) | Database → event stream |
| Event sourcing | Audit trails, time travel |
| CQRS | Separate read/write optimization |
| Idempotency | Making operations safely retryable |
