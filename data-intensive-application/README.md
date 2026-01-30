# Designing Data-Intensive Applications - Study Notes

> Comprehensive notes from Martin Kleppmann's "Designing Data-Intensive Applications"
> 
> **Focus:** System design interview preparation with practical, industry-relevant knowledge
> 
> ✅ Simple explanations of complex concepts  
> ✅ Interview tips and common questions  
> ✅ Trade-off analyses for decision making  
> ✅ Real-world examples and code snippets

---

## 📚 Table of Contents

### Part I: Foundations of Data Systems

| Chapter | Topic | Key Interview Concepts |
|---------|-------|------------------------|
| [1](notes/chapter-01-reliability-scalability-maintainability.md) | **Reliability, Scalability, Maintainability** | SLAs, percentiles (p99), fault tolerance, horizontal vs vertical scaling |
| [2](notes/chapter-02-data-models-query-languages.md) | **Data Models & Query Languages** | SQL vs NoSQL vs Graph, when to use each, polyglot persistence |
| [3](notes/chapter-03-storage-and-retrieval.md) | **Storage and Retrieval** | LSM-trees vs B-trees, OLTP vs OLAP, column storage, indexes |
| [4](notes/chapter-04-encoding-and-evolution.md) | **Encoding and Evolution** | Protobuf vs Avro vs JSON, schema evolution, API versioning |

### Part II: Distributed Data

| Chapter | Topic | Key Interview Concepts |
|---------|-------|------------------------|
| [5](notes/chapter-05-replication.md) | **Replication** | Leader-follower, multi-leader, quorums, consistency issues |
| [6](notes/chapter-06-partitioning.md) | **Partitioning (Sharding)** | Hash vs range, hot spots, rebalancing, secondary indexes |
| [7](notes/chapter-07-transactions.md) | **Transactions** | ACID, isolation levels, lost updates, write skew |
| [8](notes/chapter-08-distributed-systems-trouble.md) | **Distributed Systems Problems** | Network failures, clock skew, split brain, fencing tokens |
| [9](notes/chapter-09-consistency-consensus.md) | **Consistency and Consensus** | Linearizability, CAP theorem, Raft, ZooKeeper, 2PC |

### Part III: Derived Data

| Chapter | Topic | Key Interview Concepts |
|---------|-------|------------------------|
| [10](notes/chapter-10-batch-processing.md) | **Batch Processing** | MapReduce concepts, Spark, join strategies, idempotency |
| [11](notes/chapter-11-stream-processing.md) | **Stream Processing** | Kafka vs RabbitMQ, event time, windows, exactly-once |
| [12](notes/chapter-12-future-of-data-systems.md) | **Future of Data Systems** | Event sourcing, CQRS, CDC, data mesh, lakehouse |

---

## 🎯 Quick Reference by Use Case

### Database Selection Guide

| Need | Best Choice | Why |
|------|-------------|-----|
| ACID transactions | PostgreSQL | Battle-tested, strong consistency |
| Flexible schema | MongoDB | Schema-on-read, nested documents |
| Graph relationships | Neo4j | Native graph traversal |
| Full-text search | Elasticsearch | Inverted index, relevance scoring |
| Time series | TimescaleDB, InfluxDB | Time-partitioned, compression |
| Cache | Redis | In-memory, sub-ms latency |
| Analytics | ClickHouse, BigQuery | Column storage, aggregations |

### Messaging System Selection

| Need | Best Choice | Why |
|------|-------------|-----|
| Task queue (one consumer) | RabbitMQ, SQS | Message deleted after processing |
| Event log (replay, multiple consumers) | Kafka | Durable log, consumer groups |
| Real-time pub/sub | Redis Pub/Sub | Low latency, fire-and-forget |

### Processing Engine Selection

| Need | Best Choice | Why |
|------|-------------|-----|
| Batch ETL | Spark | Mature, SQL support, large ecosystem |
| Stream processing | Flink | True streaming, exactly-once |
| Simple streaming | Kafka Streams | No separate cluster needed |
| Ad-hoc SQL queries | Trino/Presto | Query data lake directly |

---

## 🔑 Core Principles (The Big Ideas)

### The Three Pillars

Every system design interview answer should address:

1. **Reliability** - System works correctly despite faults
   - What happens when a server dies? Database crashes? Network partitions?
   
2. **Scalability** - System handles growth gracefully  
   - What if traffic 10x? Data 100x? Users worldwide?
   
3. **Maintainability** - System is operable and evolvable
   - Can a new engineer understand this? Can we deploy safely?

### Key Trade-offs You'll Discuss

| Trade-off | Favor Left When... | Favor Right When... |
|-----------|---------------------|---------------------|
| Consistency ↔ Availability | Banking, inventory | Social feeds, analytics |
| Latency ↔ Throughput | User-facing APIs | Batch jobs |
| Simplicity ↔ Flexibility | Startups, MVPs | Complex enterprise needs |
| Strong ↔ Eventual consistency | Transactions matter | Scale matters more |

---

## 💡 Top 20 Concepts for Interviews

1. **Horizontal vs Vertical Scaling** - Add machines vs bigger machine
2. **Replication** - Copies for availability and read scaling
3. **Partitioning/Sharding** - Split data across machines
4. **Consistent Hashing** - Minimize data movement when scaling
5. **CAP Theorem** - Can't have C+A during partition
6. **ACID** - Atomicity, Consistency, Isolation, Durability
7. **Isolation Levels** - Read committed, snapshot, serializable
8. **Leader Election** - Raft/Paxos, ZooKeeper
9. **Quorum** - Majority agreement (W + R > N)
10. **Eventual Consistency** - Will converge, but when?
11. **Write-Ahead Log (WAL)** - Durability before acknowledgment
12. **LSM-Tree vs B-Tree** - Write vs read optimization
13. **Bloom Filter** - Probably yes, definitely no
14. **Idempotency** - Same operation, same result
15. **Exactly-Once Semantics** - At-least-once + deduplication
16. **Event Sourcing** - Store events, derive state
17. **CQRS** - Separate read and write models
18. **CDC (Change Data Capture)** - Stream database changes
19. **Saga Pattern** - Distributed transactions via compensation
20. **Circuit Breaker** - Fail fast when downstream is down

---

## 📊 Common System Design Patterns

### Modern Data Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        CLIENTS                                   │
│                    (Web, Mobile, API)                           │
└─────────────────────────┬───────────────────────────────────────┘
                          │
┌─────────────────────────▼───────────────────────────────────────┐
│                    API GATEWAY                                   │
│              (Rate limiting, Auth, Routing)                     │
└─────────────────────────┬───────────────────────────────────────┘
                          │
┌─────────────────────────▼───────────────────────────────────────┐
│                    SERVICES                                      │
│         (Microservices or Modular Monolith)                    │
└───────┬─────────────────┬─────────────────┬─────────────────────┘
        │                 │                 │
        ▼                 ▼                 ▼
   ┌─────────┐      ┌─────────┐      ┌─────────┐
   │  Cache  │      │   DB    │      │  Queue  │
   │ (Redis) │      │(Postgres│      │ (Kafka) │
   └─────────┘      └─────────┘      └────┬────┘
                          │               │
                          ▼               ▼
                    ┌─────────┐    ┌─────────────┐
                    │ Replica │    │   Stream    │
                    └─────────┘    │  Processor  │
                                   └──────┬──────┘
                                          │
                    ┌─────────────────────┼────────────────────┐
                    ▼                     ▼                    ▼
              ┌──────────┐         ┌──────────┐         ┌──────────┐
              │  Search  │         │Analytics │         │   ML     │
              │(Elastic) │         │(BigQuery)│         │ Models   │
              └──────────┘         └──────────┘         └──────────┘
```

### Event-Driven Architecture

```
Services ──► Kafka ──┬──► Service A (processes events)
                     ├──► Service B (builds search index)
                     ├──► Service C (sends notifications)
                     └──► Analytics (data warehouse)
```

---

## 🛠 Practical Recommendations

1. **Start simple** - Single PostgreSQL handles more than you think (Instagram scaled to 30M users on one DB)
2. **Measure first** - Don't optimize without data; premature optimization is the root of all evil
3. **Design for failure** - Everything fails eventually; plan for it
4. **Make operations idempotent** - Operations will be retried
5. **Use boring technology** - Battle-tested > cutting-edge for production
6. **Log everything** - Immutable event logs enable debugging, replay, and audit

---

## 📖 How to Use These Notes

### For Learning
Read chapters 1-4 first (foundations), then 5-9 (distributed systems), finally 10-12 (data processing).

### For Interview Prep
1. Review the "Key Interview Concepts" in the table of contents
2. Read the "System Design Interview Tips" section at the end of each chapter
3. Practice explaining each concept in 2-3 sentences
4. Know the trade-offs (when to use what)

### For Quick Reference
Jump to specific chapters when solving design problems. Use the decision frameworks and comparison tables.

---

## 🎤 Sample Interview Questions by Chapter

| Chapter | Sample Question |
|---------|-----------------|
| 1 | "How would you design a system for 99.99% availability?" |
| 2 | "SQL vs NoSQL—when would you use each?" |
| 3 | "Why might you use a column database for analytics?" |
| 4 | "How do you handle schema changes without downtime?" |
| 5 | "How does database replication work? What can go wrong?" |
| 6 | "How would you shard a database? What's a hot spot?" |
| 7 | "Explain ACID. What isolation level would you choose?" |
| 8 | "What problems occur in distributed systems?" |
| 9 | "What is the CAP theorem? What is Raft?" |
| 10 | "How does MapReduce work? Why use Spark instead?" |
| 11 | "Kafka vs RabbitMQ—when would you use each?" |
| 12 | "What is event sourcing? When would you use it?" |

---

*These notes are for study purposes. For complete understanding, read the original book by Martin Kleppmann.*
