# Chapter 1: Reliable, Scalable, and Maintainable Applications

## Core Concept

Modern applications are **data-intensive** not compute-intensive. This means the hard problems aren't about CPU cycles—they're about:
- **Amount of data**: Storing and processing terabytes/petabytes
- **Complexity of data**: Relationships, different formats, changing requirements
- **Speed of change**: Data arriving rapidly, users expecting real-time updates

**Why this matters in interviews**: When designing any system, you'll be evaluated on how well you address these three concerns. They form the foundation of every system design discussion.

## Building Blocks of Data Systems

Almost every application you'll design in an interview uses some combination of these components:

```
┌─────────────────────────────────────────────────────────────┐
│                    Application Code                          │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────────────┐│
│  │Database │  │  Cache  │  │ Search  │  │Message Queue    ││
│  │(storage)│  │(speed)  │  │ Index   │  │(async processing││
│  └─────────┘  └─────────┘  └─────────┘  └─────────────────┘│
└─────────────────────────────────────────────────────────────┘
```

**What each component does:**
- **Database**: Stores your data durably. When the server restarts, data is still there. Examples: PostgreSQL, MySQL, MongoDB
- **Cache**: Stores frequently accessed data in memory for fast retrieval. Much faster than database but data can be lost. Examples: Redis, Memcached
- **Search Index**: Allows searching text by keywords. Databases are bad at "find all products containing 'wireless headphones'". Examples: Elasticsearch, Algolia
- **Message Queue**: Lets services communicate asynchronously. Service A puts a message, Service B processes it later. Examples: Kafka, RabbitMQ, SQS

**Real-world example**: A typical web app combines:
- PostgreSQL/MySQL for primary data (users, orders, products)
- Redis for caching (session data, frequently accessed queries)
- Elasticsearch for search (product search, log analysis)
- Kafka for event streaming (order events, notifications)

---

## 1. Reliability

> "The system should continue to work correctly even when things go wrong."

**Simple definition**: Reliability means the system does what users expect, even when things fail. It doesn't mean "never fails"—it means "handles failures gracefully."

**Interview tip**: When asked "how would you make this reliable?", think about what can fail and how to handle each failure.

### Types of Faults

| Fault Type | Examples | Mitigation |
|------------|----------|------------|
| **Hardware** | Disk failure, RAM errors, power outage | Redundancy (RAID, multi-node) |
| **Software** | Bugs, resource leaks, cascading failures | Testing, monitoring, graceful degradation |
| **Human** | Config errors, wrong deployments | Good UX, sandbox environments, rollback |

**Hardware faults explained**: 
- Hard disks have a Mean Time To Failure (MTTF) of 10-50 years. With 10,000 disks, expect one to die per day on average.
- Solution: Redundancy. Use RAID for disks, have multiple servers, spread across data centers.

**Software faults explained**:
- A bug that crashes the application when input is unusual
- A service that slows down, causing timeouts everywhere
- Memory leaks that crash the server after a few days
- Solution: Testing, monitoring, circuit breakers, graceful degradation

**Human faults explained**:
- Someone pushes bad config to production
- Database migration script deletes data
- Solution: Good tooling, staging environments, easy rollback, gradual rollouts

### Chaos Engineering

**What it is**: Intentionally breaking things in production to find weaknesses before they cause real outages.

Netflix's **Chaos Monkey** randomly kills production instances to ensure systems handle failures gracefully. If your system can't handle one server dying randomly, you'll find out during a Chaos Monkey test rather than during Black Friday.

```bash
# The idea: What happens if we kill this right now?
kill -9 <primary_db_pid>
# A reliable system: Traffic automatically routes to replica
# An unreliable system: 500 errors until someone wakes up and fixes it
```

### Key Metrics

- **MTTF** (Mean Time To Failure): Average time until something breaks
- **MTTR** (Mean Time To Recovery): Average time to fix it after it breaks

**MTTR is often more important!** You can't prevent all failures, but you can recover quickly. A system that fails once a month but recovers in 5 minutes is better than one that fails once a year but takes 4 hours to fix.

**Interview insight**: When discussing reliability, mention:
1. What can fail (list the components)
2. How you detect failures (monitoring, health checks)
3. How you recover (failover, retries, backups)

---

## 2. Scalability

> "As the system grows, there should be reasonable ways to deal with that growth."

**Simple definition**: Scalability is NOT about handling current load—it's about having a plan for when load increases 10x or 100x. 

**Interview tip**: Always ask "what's the expected scale?" before designing. A system for 1000 users/day looks completely different from one for 1 billion users/day.

### Describing Load: Load Parameters

Before you can scale, you need to describe your current load. Different systems have different "load parameters":
- **Web server**: Requests per second
- **Database**: Read/write ratio, number of concurrent connections
- **Cache**: Hit rate, memory usage
- **Message queue**: Messages per second, queue depth

**The Twitter Example** (classic interview question):

Twitter's challenge (2012 data):
- 12,000 tweets/sec (posting tweets)
- 300,000 reads/sec (viewing home timeline)

**The problem**: When you open Twitter, you see tweets from everyone you follow. How do you build that?

**Approach 1: Read-heavy (fan-out on read)**
```sql
-- When user opens home timeline, query in real-time:
SELECT * FROM tweets 
WHERE author_id IN (SELECT followed_id FROM follows WHERE follower_id = me)
ORDER BY created_at DESC LIMIT 100;
```
- **Pro**: Writing a tweet is instant (just insert one row)
- **Con**: Reading is expensive (join across millions of rows)

**Approach 2: Write-heavy (fan-out on write)**
```
When user posts tweet:
1. Insert tweet into tweets table
2. Look up all followers (say 1000 people)
3. Insert tweet into each follower's cached timeline
```
- **Pro**: Reading is instant (just fetch pre-computed list)
- **Con**: Celebrities with 50M followers = 50M writes per tweet!

```
Approach 1: Read-heavy join
┌──────────┐        ┌──────────┐
│  Tweets  │◄──────►│Followers │  = JOIN at read time
└──────────┘        └──────────┘

Approach 2: Write-time fan-out  
Tweet ──► Cache timelines for all followers
```

**Twitter's actual solution**: Hybrid approach
- For most users: fan-out on write (pre-compute timelines)
- For celebrities (>1M followers): fan-out on read (merge their tweets at read time)

**Interview insight**: There's rarely one "correct" answer. The right approach depends on your specific load patterns.

### Describing Performance

**Latency vs Response Time** (people often confuse these):
- **Latency**: Time the request spends waiting (in queues, for processing to start)
- **Response time**: Total time from client sending request to receiving response

**Why percentiles matter more than averages**:

Imagine 1000 requests with these response times:
- Average: 200ms
- But actually: 950 requests at 100ms, 50 requests at 2000ms

The average looks fine (200ms), but 5% of users are having a terrible experience!

```
Response times for 1000 requests:
p50 (median):  200ms  ← Half of requests faster than this
p95:           500ms  ← 95% of requests faster than this
p99:           1.5s   ← 99% of requests faster than this
p99.9:         3s     ← Tail latency (often VIP customers!)
```

**Why p99 matters**: Your most engaged users make the most requests, so they're most likely to hit the slow p99 cases. If Amazon's p99 is 3 seconds, that's affecting your best customers!

**SLA Example**: "99% of requests complete in < 200ms"
- This means you're measuring p99, not average
- If you promise average < 200ms, half your users might be unhappy

### Scaling Approaches

| Approach | What it means | Pros | Cons |
|----------|---------------|------|------|
| **Vertical (Scale Up)** | Bigger machine (more CPU, RAM) | Simple, no code changes | Has limits, expensive at top end |
| **Horizontal (Scale Out)** | More machines | No theoretical limit | Complex (distributed systems) |
| **Elastic** | Automatically add/remove machines | Handles spikes, cost-efficient | Complex to set up properly |

**Vertical scaling explained**:
- Your server has 8GB RAM and is struggling
- Buy a server with 64GB RAM
- Works great until you need 1TB RAM (very expensive) or 10TB (doesn't exist)

**Horizontal scaling explained**:
- Your server handles 1000 req/sec
- Add 9 more servers behind a load balancer
- Now you handle 10,000 req/sec
- But now you need to think about: Which server has the data? How do they coordinate?

**Modern approach**: Start vertical (it's simpler), go horizontal when you hit limits.

**Interview insight**: Don't immediately jump to horizontal scaling. Many systems can run on a single powerful server much longer than people think. Instagram had 30 million users on a single PostgreSQL server!

---

## 3. Maintainability

> "Over time, many people will work on the system. They should be productive."

**Simple definition**: Maintainability is about the humans who will work on this system in the future. Can they understand it? Can they change it safely? Can they operate it without constant firefighting?

**The reality**: Most software cost is NOT in initial development—it's in ongoing maintenance. Bug fixes, keeping systems running, adapting to new requirements, paying down technical debt.

### Three Design Principles

#### Operability (Easy to Operate)

**What it means**: Operations teams can keep the system running smoothly without heroics.

**Signs of good operability**:
- System health is visible at a glance
- Common tasks are automated
- Documentation exists and is accurate
- System behavior is predictable

```yaml
# Good: Observable system with health endpoints
GET /health
{
  "status": "healthy",
  "db_connection": "ok",
  "cache_hit_ratio": 0.94,
  "queue_depth": 12,
  "uptime_seconds": 86400
}

# Good: Runbooks for common issues
# "If queue_depth > 100, scale up workers"
# "If cache_hit_ratio < 0.5, check for cache eviction issues"

# Bad: "Only Dave knows how this works"
# Bad: "We restart it every night because it gets slow"
```

**Operability checklist:**
- [ ] Monitoring & alerting (know when things break)
- [ ] Deployment automation (deploy without fear)
- [ ] Documentation (onboard new team members)
- [ ] Predictable behavior (no surprises)
- [ ] Easy rollback (undo mistakes quickly)

#### Simplicity (Manage Complexity)

**What it means**: The system is as simple as possible (but no simpler). New engineers can understand it.

**Two types of complexity**:
- **Essential complexity**: Inherent to the problem (a banking system MUST handle money correctly)
- **Accidental complexity**: We added it ourselves (convoluted code, unnecessary abstractions)

**The goal**: Minimize accidental complexity through good abstractions.

```python
# Bad: Leaky abstraction - database details everywhere
def save_user(user):
    conn = psycopg2.connect(
        host="db.example.com", 
        database="prod",
        user="app", 
        password="secret123"
    )
    cursor = conn.cursor()
    cursor.execute(
        "INSERT INTO users (name, email) VALUES (%s, %s)",
        (user.name, user.email)
    )
    conn.commit()
    conn.close()
    
# Good: Clean abstraction - implementation hidden
def save_user(user):
    user_repository.save(user)
    
# The repository handles all the database details
# If you switch from PostgreSQL to MongoDB, only change the repository
```

**Signs of accidental complexity**:
- "You need to understand X, Y, and Z before you can change this"
- Code that does the same thing in multiple places
- Workarounds for previous workarounds
- "We'd refactor this, but we're afraid to touch it"

#### Evolvability (Easy to Change)

**What it means**: You can modify the system as requirements change without massive rewrites.

**Why it matters**: Requirements ALWAYS change. New features, new regulations, new scale, new integrations.

**Design for change**:
- **Agile practices** at the process level (small iterations, feedback)
- **TDD** at the code level (tests give confidence to change)
- **Schema evolution** at the data level (see Chapter 4)
- **API versioning** at the service level

**Interview insight**: When designing systems, ask yourself: "What if requirements change?" Good designs make common changes easy.

---

## Real-World Architecture Example

Here's how these concepts come together in a typical web application:

```
                    ┌─────────────┐
                    │   Users     │
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │ Load Balancer│  ← Reliability: No single point of failure
                    └──────┬──────┘
                           │
         ┌─────────────────┼─────────────────┐
         │                 │                 │
   ┌─────▼─────┐    ┌─────▼─────┐    ┌─────▼─────┐
   │ App Server│    │ App Server│    │ App Server│  ← Scalability: Horizontal
   └─────┬─────┘    └─────┬─────┘    └─────┬─────┘
         │                │                │
         └────────────────┼────────────────┘
                          │
    ┌─────────────────────┼─────────────────────┐
    │                     │                     │
┌───▼───┐          ┌─────▼─────┐         ┌─────▼─────┐
│ Cache │          │  Primary  │         │  Search   │
│(Redis)│          │    DB     │         │(Elastic)  │
└───────┘          │(Postgres) │         └───────────┘
                   └─────┬─────┘
                         │ Replication    ← Reliability: Data redundancy
                   ┌─────▼─────┐
                   │  Replica  │
                   │    DB     │
                   └───────────┘
```

**How this addresses each concern**:
- **Reliability**: Load balancer + multiple app servers + database replica = no single point of failure
- **Scalability**: Add more app servers for more traffic, add read replicas for more database reads
- **Maintainability**: Each component has a clear purpose, standard technologies

---

## System Design Interview Tips

When answering system design questions, structure your answer around these three pillars:

1. **Start with requirements**: "Let me clarify the scale and requirements"
2. **Address reliability**: "Here's how we handle failures..."
3. **Address scalability**: "As we grow, we can scale by..."
4. **Address maintainability**: "This design is simple because..."

## Key Takeaways

1. **Reliability** ≠ "never fails" → means "handles failures gracefully"
2. **Scalability** requires understanding YOUR specific load patterns first
3. **Maintainability** is about people, not just code—think about the next engineer
4. **Percentiles** (p95, p99) matter more than averages for SLAs
5. **Simplicity** is a feature, not a shortcut—fight accidental complexity

---

## Quick Reference

| Concept | Key Question to Ask | Interview Red Flags |
|---------|---------------------|---------------------|
| Reliability | What happens when X fails? | "We assume nothing fails" |
| Scalability | What if traffic grows 10x? | "We'll figure it out later" |
| Maintainability | Can a new hire understand this? | "Only I know how this works" |

---

## Common Interview Questions

1. **"How would you design a system that's highly available?"**
   - Redundancy at every layer
   - Automatic failover
   - Health checks and monitoring
   - Graceful degradation

2. **"How would you handle 10x traffic increase?"**
   - First: Can we optimize? (caching, queries)
   - Then: Scale vertically if headroom exists
   - Finally: Scale horizontally (more servers, sharding)

3. **"How do you measure system performance?"**
   - Not averages—use percentiles (p50, p95, p99)
   - Set SLOs based on user experience
   - Monitor continuously, alert on anomalies
