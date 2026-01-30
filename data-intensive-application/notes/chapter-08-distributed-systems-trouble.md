# Chapter 8: The Trouble with Distributed Systems

## Core Concept

> "Anything that can go wrong, will go wrong." - Murphy's Law, amplified for distributed systems.

**What it is:** This chapter covers all the ways distributed systems can fail—and why those failures are fundamentally different from single-machine failures.

**Why it matters:** In interviews, understanding failure modes separates candidates who've "read about" distributed systems from those who've actually built them. These problems show up constantly in real systems.

**Key Insight:** In distributed systems, partial failures are the norm, not the exception. Some nodes work while others don't. You can't tell the difference between a slow network and a dead node.

```
┌─────────────────────────────────────────────────────────────┐
│        Single Machine vs Distributed System                  │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   Single Machine:                                          │
│   • Works or doesn't work (binary failure)                 │
│   • Deterministic - same input → same output               │
│   • Shared memory, single clock                            │
│                                                             │
│   Distributed System:                                      │
│   • Partial failures common (some nodes up, some down)     │
│   • Non-deterministic (network delays vary randomly)       │
│   • No shared state, clocks drift apart                    │
│   • Messages can be lost, delayed, or duplicated           │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

> **💡 Interview Tip:** When discussing distributed systems, always acknowledge that "anything can fail partially." This shows maturity. Junior engineers often assume the happy path; senior engineers design for failures.

---

## Unreliable Networks

**What it is:** Networks between machines are fundamentally unreliable. Packets get lost, delayed, duplicated, or delivered out of order.

**Why it matters:** Most distributed system bugs come from assuming the network is reliable. Understanding network failures helps you design resilient systems.

### Network Problems

```
┌─────────────────────────────────────────────────────────────┐
│            When You Send a Request...                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   Client ───────► ???                                      │
│                                                             │
│   What might have happened:                                │
│   1. Request lost in network (never arrived)               │
│   2. Request queued somewhere, will arrive later           │
│   3. Remote node received it but crashed before responding │
│   4. Remote node is alive but too busy to respond          │
│   5. Remote responded, but response was lost               │
│   6. Remote responded, but response is delayed             │
│                                                             │
│   ⚠️ YOU CAN'T TELL WHICH ONE FROM THE CLIENT SIDE!       │
│                                                             │
│   All you know: "I sent a request and got no response"     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Real-world implication:** If you don't get a response, you don't know if your operation succeeded. This is why idempotency matters—you might retry an operation that already succeeded.

### The Timeout Dilemma

**What it is:** How long should you wait before assuming a request failed? There's no good answer.

**Why it's hard:** You're choosing between two types of errors:
- **False positive (timeout too short):** Node was fine, just slow. You trigger unnecessary failover, which causes load spikes and cascading failures.
- **False negative (timeout too long):** Node is actually dead. Users wait forever for errors. Other services wait for you.

```
┌─────────────────────────────────────────────────────────────┐
│                 Timeout Trade-offs                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   Short timeout (e.g., 1 second):                          │
│   ✓ Fast failure detection                                 │
│   ✗ False positives - node was just slow (GC pause?)      │
│   ✗ Unnecessary failovers cause load spikes               │
│   ✗ Thundering herd: all clients retry at once            │
│                                                             │
│   Long timeout (e.g., 30 seconds):                         │
│   ✓ Fewer false positives                                  │
│   ✗ Slow failure detection                                 │
│   ✗ Users/services wait too long                          │
│   ✗ Problems compound while waiting                        │
│                                                             │
│   Real-world approaches:                                   │
│   • Adaptive timeouts based on observed latency (P99)      │
│   • Phi Accrual Failure Detector (Cassandra, Akka)         │
│   • Exponential backoff for retries (don't hammer!)        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

> **💡 Interview Tip:** When asked "how would you set timeouts?", don't give a fixed number. Say: "I'd measure the P99 latency under normal conditions and set the timeout higher than that, maybe 2-3x. Then use exponential backoff for retries to avoid thundering herd."

### Network Partitions Are Real

**What it is:** A network partition occurs when some nodes can talk to each other but not to other nodes. The network is "split."

**Why it matters:** Partitions force you to choose between consistency and availability (CAP theorem). You can't have both during a partition.

```
            Data Center 1              Data Center 2
           ┌─────────────┐            ┌─────────────┐
           │   Node A    │            │   Node C    │
           │   Node B    │     X      │   Node D    │
           └─────────────┘   (link    └─────────────┘
                             down!)
                             
   A and B can talk to each other ✓
   C and D can talk to each other ✓
   But A/B cannot reach C/D! ✗
   
   Which side has the "correct" data? Neither knows!
   
   Options during partition:
   1. Refuse writes (choose consistency - CP)
   2. Accept writes, reconcile later (choose availability - AP)
```

---

## Unreliable Clocks

**What it is:** Each machine has its own clock, and these clocks don't agree perfectly. They drift apart over time (milliseconds to seconds).

**Why it matters:** If you use timestamps to order events or determine "which write was last," clock skew can cause data loss. This is one of the most common bugs in distributed systems.

### Two Types of Clocks

```
┌─────────────────────────────────────────────────────────────┐
│              Time-of-Day vs Monotonic Clocks                 │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   Time-of-Day Clock (wall clock):                          │
│   • Returns current time: "2024-01-15 10:30:00.123"        │
│   • ⚠️ CAN JUMP BACKWARDS! (NTP sync, leap seconds)       │
│   • ❌ DON'T use for measuring elapsed time               │
│   • ❌ DON'T use for ordering events across machines      │
│                                                             │
│   Monotonic Clock:                                         │
│   • Returns arbitrary counter (nanoseconds since boot)     │
│   • ✓ ALWAYS moves forward on a single machine            │
│   • ✓ Perfect for measuring duration locally              │
│   • ❌ Values meaningless across different machines       │
│                                                             │
│   Rule: Use monotonic for "how long did this take?"        │
│         Use wall clock only for human display              │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Clock Skew and Last-Write-Wins

**What it is:** Last-Write-Wins (LWW) uses timestamps to resolve conflicts: the write with the latest timestamp wins. Sounds reasonable, but clock skew breaks it.

**Why it's dangerous:** If Node A's clock is fast, its writes will "win" even if they happened before Node B's writes in real time. This silently loses data.

```
┌─────────────────────────────────────────────────────────────┐
│                  Last-Write-Wins Bug                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   Real-world timeline:                                     │
│   ──────────────────────────────────────────────────►      │
│                                                             │
│   Node A (clock is 5 seconds FAST):                        │
│   Clock shows: 10:00:05                                    │
│   Action: write x = 1                                      │
│                                                             │
│   Node B (clock is correct):                               │
│   Clock shows: 10:00:00                                    │
│   Action: write x = 2 (happens AFTER Node A in real time) │
│                                                             │
│   With LWW: x = 1 wins (10:00:05 > 10:00:00)              │
│   Reality: x = 2 was the LATER write!                     │
│                                                             │
│   ⚠️ 5 seconds of clock skew caused SILENT DATA LOSS      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Solutions:**

| Approach | How it works | Used by |
|----------|--------------|---------|
| Logical clocks (Lamport) | Counters that track causality, not time | Many systems |
| Vector clocks | Track per-node counters to detect conflicts | Riak, Dynamo |
| TrueTime | Bounded uncertainty intervals | Google Spanner |
| Hybrid Logical Clocks | Combine physical and logical time | CockroachDB |

> **💡 Interview Tip:** When someone mentions LWW, ask "how do you handle clock skew?" If they don't have an answer, that's a red flag. Good answers: logical clocks, vector clocks, or accepting that LWW can lose data (fine for some use cases like caching).

### Google TrueTime

**What it is:** Instead of returning a single timestamp, TrueTime returns an interval: "the current time is somewhere between T1 and T2."

**Why it works:** By explicitly acknowledging uncertainty, Spanner can wait until intervals don't overlap, guaranteeing correct ordering.

```
┌─────────────────────────────────────────────────────────────┐
│                    TrueTime API                              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   TT.now() returns interval [earliest, latest]             │
│                                                             │
│   Example: TT.now() = [10:00:00.000, 10:00:00.007]         │
│   Meaning: "Current time is within this 7ms window"        │
│                                                             │
│   To guarantee ordering:                                   │
│   • Commit T1 at time t1                                   │
│   • Before committing T2, wait until TT.now().earliest    │
│     > t1 (intervals don't overlap)                         │
│   • Now T2 is guaranteed to be "after" T1                  │
│                                                             │
│   Google achieves ε ≈ 7ms using:                           │
│   • GPS receivers in every data center                     │
│   • Atomic clocks as backup                                │
│   • Sophisticated time sync protocols                      │
│                                                             │
│   Trade-off: Commits have added latency (wait for ε)      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

> **💡 Interview Tip:** TrueTime is impressive but requires specialized hardware (GPS, atomic clocks). For most companies, use logical clocks or accept eventual consistency. Only Google-scale systems justify TrueTime's complexity.

---

## Process Pauses

**What it is:** A process can pause for an arbitrary amount of time and have no idea it was paused. When it wakes up, it continues as if nothing happened.

**Why it matters:** A process holding a lock or lease might pause, have the lock/lease expire, and another process might acquire it. When the first process wakes up, it thinks it still has the lock. Two processes now think they're the leader → split brain!

### Why Processes Pause

```
┌─────────────────────────────────────────────────────────────┐
│              Sources of Process Pauses                       │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   1. Garbage Collection (GC)                               │
│      • Java/Go can pause for seconds during GC!           │
│      • "Stop-the-world" pauses freeze the entire process  │
│      • The process has NO IDEA it was paused              │
│                                                             │
│   2. Virtual Machine Suspension                            │
│      • VM live migration between physical hosts            │
│      • Hypervisor can pause your VM to run others         │
│      • Cloud instances can be "stolen" temporarily        │
│                                                             │
│   3. Operating System                                      │
│      • Context switches when CPU is overloaded            │
│      • Swapping to disk (thrashing)                       │
│      • Disk I/O waits                                      │
│                                                             │
│   4. Resource Contention                                   │
│      • Another process hogging CPU                         │
│      • Network socket buffers full                        │
│                                                             │
│   Key insight: The paused process CANNOT TELL it paused!  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### The Lease Problem

**What it is:** A lease is like a lock with an expiration time. If you don't renew it, someone else can take it. But what if you pause?

```
┌─────────────────────────────────────────────────────────────┐
│              Lease Expiry During GC Pause                    │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   Timeline:                                                 │
│   ────────────────────────────────────────────────────►    │
│                                                             │
│   0s:  Node A acquires lease (expires at 10s)              │
│   5s:  Node A starts GC pause... (frozen)                  │
│   10s: Lease expires (A still paused, doesn't know!)       │
│   11s: Node B acquires lease (becomes new leader)          │
│   12s: Node A wakes up, thinks it's still the leader!      │
│   12s: Node A writes to storage (CORRUPTS DATA!)           │
│                                                             │
│   Problem: A had no idea 7 seconds passed!                 │
│   A checked the lease at 5s, it was valid.                 │
│   A can't tell it was paused.                              │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

> **💡 Interview Tip:** This is a classic distributed systems gotcha. If someone proposes using "a lock with a timeout," ask "what happens if the lock holder pauses due to GC?" The answer is fencing tokens.

### Fencing Tokens

**What it is:** A monotonically increasing token issued with each lock/lease acquisition. Storage servers reject writes from stale tokens.

**Why it works:** Even if Node A wakes up thinking it has the lock, its token (33) is older than Node B's token (34). Storage rejects A's writes.

```
┌─────────────────────────────────────────────────────────────┐
│                    Fencing Token                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   Lock service returns monotonically increasing token      │
│                                                             │
│   Timeline:                                                │
│   1. Node A acquires lock → gets token 33                  │
│   2. Node A pauses (GC)                                    │
│   3. Lock expires                                          │
│   4. Node B acquires lock → gets token 34                  │
│   5. Node A wakes up with token 33                         │
│                                                             │
│   Storage server behavior:                                 │
│   • Tracks highest token seen (currently: 34)              │
│   • Node A sends write with token 33                       │
│   • Storage: "33 < 34? REJECTED!"                          │
│   • Node B's writes with token 34: ACCEPTED                │
│                                                             │
│       Node A                  Node B                       │
│       token=33                token=34                     │
│          │                       │                         │
│          │    Storage Server     │                         │
│          │  ┌───────────────┐    │                         │
│          └─►│ max_token=34  │◄───┘                         │
│     REJECT  │               │  ACCEPT                      │
│             └───────────────┘                              │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Implementation requirements:**
1. Lock service must return monotonically increasing tokens
2. Storage must track the highest token it's seen
3. Storage must reject writes with tokens ≤ the highest seen

> **💡 Interview Tip:** Fencing tokens are the solution to "lock expiration during pause." If designing a distributed lock, always mention: (1) locks should have TTL, (2) use fencing tokens to prevent stale holders from causing damage.

---

## Knowledge in Distributed Systems

### The Two Generals Problem

**What it is:** A thought experiment proving that you can never achieve 100% certainty of agreement over an unreliable network.

**Why it matters:** It sets realistic expectations. We can't have perfect consensus, but we can get arbitrarily close with enough redundancy.

```
┌─────────────────────────────────────────────────────────────┐
│               Two Generals Problem                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   Two armies must attack together or they lose.            │
│   They can only communicate via messengers who might       │
│   be captured (messages lost).                             │
│                                                             │
│   Army A                    Army B                         │
│   ┌─────┐                   ┌─────┐                        │
│   │     │ ──"Attack at───►  │     │                        │
│   │     │    dawn"          │     │  ← might be lost!     │
│   │     │ ◄──"ACK"───────   │     │  ← might be lost!     │
│   │     │ ──"ACK of ACK"─►  │     │  ← might be lost!     │
│   │     │ ◄──"ACK of..."─   │     │  ...infinite acks     │
│   └─────┘                   └─────┘                        │
│                                                             │
│   No finite number of messages can guarantee agreement!    │
│                                                             │
│   Real-world implication:                                  │
│   Perfect consensus is impossible over unreliable network. │
│   But we CAN get "good enough" with timeouts + retries.   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Byzantine Faults

**What it is:** When a node doesn't just fail, but actively misbehaves—sending incorrect or conflicting messages.

**Why it (usually) doesn't matter:** In most systems, we trust our own nodes. Byzantine fault tolerance is expensive (need 3f+1 nodes to tolerate f failures) and only matters for: blockchains, aerospace, or when you don't trust other participants.

```
┌─────────────────────────────────────────────────────────────┐
│           Types of Node Failures                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   Crash-stop (most common assumption):                     │
│   • Node works correctly, then stops forever               │
│   • Simple to handle: detect failure, failover             │
│   • Most consensus algorithms assume this                  │
│                                                             │
│   Crash-recovery:                                          │
│   • Node may crash but can restart                         │
│   • May lose in-memory state, must recover from disk       │
│   • Practical for real systems                             │
│                                                             │
│   Byzantine (rarely needed):                               │
│   • Node may behave ARBITRARILY (even maliciously)        │
│   • Could lie, send conflicting messages                   │
│   • Need 3f+1 nodes to tolerate f Byzantine failures      │
│   • Relevant for: Blockchain, adversarial environments    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

> **💡 Interview Tip:** Most interview problems assume crash-stop failures. Don't over-engineer for Byzantine faults unless specifically asked about blockchain or adversarial scenarios.

---

## System Models

**What it is:** Formal assumptions about how the system behaves, used to reason about algorithm correctness.

### Timing Assumptions

```
┌─────────────────────────────────────────────────────────────┐
│                  System Models                               │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   Synchronous (unrealistic):                               │
│   • Messages delivered within known bounded time           │
│   • Processes execute at known bounded speed               │
│   • Clocks have known bounded drift                        │
│   • Easy to build algorithms, but doesn't match reality    │
│                                                             │
│   Asynchronous (too pessimistic):                          │
│   • No timing assumptions whatsoever                       │
│   • Can't use timeouts for failure detection!             │
│   • Consensus is impossible (FLP result)                   │
│                                                             │
│   Partially Synchronous (practical - what we use):        │
│   • System behaves synchronously MOST of the time          │
│   • Occasionally has unbounded delays                      │
│   • Algorithms use timeouts but handle false positives    │
│   • This matches real networks                             │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

> **💡 Interview Tip:** Real systems assume "partial synchrony"—usually fast, occasionally slow. Design for the common case (use timeouts) but handle the uncommon case (retries, idempotency).

---

## Practical Patterns

### Circuit Breaker

**What it is:** A pattern that "trips" after repeated failures, preventing cascade failures by fast-failing requests.

**Why it matters:** Without circuit breakers, a failing service can take down everything that depends on it (cascade failure).

```python
class CircuitBreaker:
    """
    States: CLOSED (normal) → OPEN (failing) → HALF_OPEN (testing)
    """
    def __init__(self, failure_threshold=5, reset_timeout=60):
        self.failures = 0
        self.threshold = failure_threshold
        self.reset_timeout = reset_timeout
        self.state = "CLOSED"
        self.last_failure_time = None
    
    def call(self, func, *args):
        if self.state == "OPEN":
            # Check if enough time passed to try again
            if time.time() - self.last_failure_time > self.reset_timeout:
                self.state = "HALF_OPEN"
            else:
                raise CircuitOpenError("Failing fast - service down")
        
        try:
            result = func(*args)
            if self.state == "HALF_OPEN":
                self.state = "CLOSED"  # Service recovered!
                self.failures = 0
            return result
        except Exception as e:
            self.failures += 1
            self.last_failure_time = time.time()
            if self.failures >= self.threshold:
                self.state = "OPEN"  # Trip the circuit
            raise
```

### Retry with Exponential Backoff

**What it is:** Wait longer between each retry attempt to avoid overwhelming a recovering service.

**Why the jitter:** Without jitter, all clients retry at the same time (thundering herd).

```python
def retry_with_backoff(func, max_retries=5, base_delay=1):
    for attempt in range(max_retries):
        try:
            return func()
        except RetriableError as e:
            if attempt == max_retries - 1:
                raise  # Final attempt failed
            
            # Exponential backoff: 1s, 2s, 4s, 8s, 16s
            delay = base_delay * (2 ** attempt)
            # Add jitter to prevent thundering herd
            jitter = random.uniform(0, delay * 0.1)
            time.sleep(delay + jitter)
```

> **💡 Interview Tip:** Always mention exponential backoff + jitter when discussing retries. It's a signal that you understand real-world distributed systems issues.

### Health Checks

**What it is:** Endpoints that report whether a service is alive and ready to handle traffic.

```yaml
# Kubernetes-style health probes
livenessProbe:           # Is the process alive? (restart if not)
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 5
  failureThreshold: 3    # 3 failures → restart container

readinessProbe:          # Can it handle traffic? (remove from LB if not)
  httpGet:
    path: /ready
    port: 8080
  periodSeconds: 5
```

---

## The Split-Brain Problem

**What it is:** During a network partition, both sides think they're the "real" cluster and accept writes independently. When the partition heals, you have conflicting data.

**Why it's dangerous:** Two leaders accepting writes = data divergence = data loss or corruption.

```
┌─────────────────────────────────────────────────────────────┐
│                Split-Brain Scenario                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   Before partition:                                         │
│   ┌─────────────────────────────────────┐                  │
│   │  [Leader A]  [Follower B]  [Follower C]  │             │
│   └─────────────────────────────────────┘                  │
│                                                             │
│   After network partition:                                 │
│   ┌───────────────┐    X    ┌───────────────┐             │
│   │  [Leader A]   │  (net)  │ [New Leader B]│             │
│   │               │  down   │  [Follower C] │             │
│   └───────────────┘         └───────────────┘             │
│                                                             │
│   Both A and B accept writes → DATA DIVERGENCE!           │
│                                                             │
│   Prevention strategies:                                   │
│   • Quorum: Need majority to accept writes                │
│     (A alone = 1/3 → can't write; B+C = 2/3 → can write)  │
│   • Fencing: Cut off old leader's storage access          │
│   • STONITH: "Shoot The Other Node In The Head"           │
│     (Physically power off the old leader)                 │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

> **💡 Interview Tip:** When designing leader election, always address split-brain. Say: "We need a quorum (majority) to elect a new leader. A single node can't declare itself leader." This shows you understand consensus.

---

## Key Takeaways

1. **Networks are unreliable** - design for packet loss, delays, partitions
2. **Clocks are unreliable** - use logical clocks for ordering, not wall clocks
3. **Processes can pause** - GC, VM migration, OS scheduling (use fencing tokens!)
4. **Partial failures are normal** - some nodes work while others don't
5. **Use timeouts** but understand they're imperfect (false positives happen)
6. **Fencing tokens** prevent stale leaders from causing damage
7. **Design for failure** - circuit breakers, retries with backoff, graceful degradation

---

## Quick Reference

| Problem | Detection | Mitigation |
|---------|-----------|------------|
| Network partition | Timeout | Quorum, graceful degradation |
| Node failure | Heartbeat timeout | Failover, replication |
| Clock skew | NTP monitoring | Logical clocks, TrueTime |
| Process pause | Lease expiry | Fencing tokens |
| Split brain | Quorum check | STONITH, fencing |
| Message loss | ACK timeout | Retry with idempotency |

---

## System Design Interview Tips

**When to bring up these problems in an interview:**
- Any distributed database design (mention clock skew, partitions)
- Leader election (mention split-brain, fencing)
- Microservices communication (mention timeouts, circuit breakers)
- Cache invalidation (mention clock skew, eventual consistency)

**Red flags interviewers look for:**
- Assuming the network is reliable
- Using wall-clock timestamps for ordering
- Not considering what happens during leader failover
- Ignoring GC pauses when discussing locks/leases

**What sets great candidates apart:**
- Proactively mentioning failure modes before being asked
- Understanding the trade-offs (timeouts: too short vs too long)
- Knowing practical solutions (fencing tokens, idempotency)
- Acknowledging uncertainty ("we can't distinguish slow from dead")

---

## Common Interview Questions

**Q: How would you detect if a node is dead?**
A: Use heartbeats with timeouts. But acknowledge that you can't distinguish "dead" from "very slow" or "network partition." Set timeout based on observed P99 latency (e.g., 2-3x P99). Use adaptive timeouts that adjust to network conditions.

**Q: How do you handle a request timeout—do you retry?**
A: It depends. If the operation is idempotent (safe to repeat), retry with exponential backoff + jitter. If not idempotent, you need an idempotency key or to check if the operation succeeded before retrying. Never hammer a failing service.

**Q: How would you implement a distributed lock?**
A: Use a lock service with TTL (auto-expiry). Issue fencing tokens with each lock grant. Storage servers must validate tokens and reject stale ones. Consider using ZooKeeper or etcd. Acknowledge that GC pauses can cause a node to hold an expired lock—that's why fencing tokens matter.

**Q: Why can't you use timestamps to order events in a distributed system?**
A: Clock skew. Different machines have different clock values. A write that happened "later" in real time might have an earlier timestamp if that machine's clock is slow. Use logical clocks (Lamport, Vector) or TrueTime for ordering.

**Q: What happens if your leader fails mid-operation?**
A: Followers detect missing heartbeats and elect a new leader (quorum required). The old leader might still think it's leader (GC pause, network partition). Use fencing tokens so storage rejects writes from the stale leader. Design operations to be idempotent so they can safely retry on the new leader.
