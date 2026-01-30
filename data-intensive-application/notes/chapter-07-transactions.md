# Chapter 7: Transactions

## Core Concept

> "Transactions are an abstraction layer that allows an application to pretend that certain concurrency problems and certain kinds of hardware/software faults don't exist."

**What it is:** A transaction groups multiple database operations into one logical unit—all operations succeed together or fail together.

**Why it matters:** Without transactions, if your app crashes halfway through transferring money between accounts, you might deduct from one account but never credit the other. Transactions guarantee you won't end up in these inconsistent states.

> **💡 Interview Tip:** When asked "why use transactions?", mention: (1) fault tolerance - partial failures become total failures that can be safely retried, (2) concurrency safety - multiple users don't corrupt each other's data.

---

## ACID Properties (What They Really Mean)

**What it is:** ACID is a set of properties that database transactions provide. But here's the key insight—the letters don't all mean what most people think!

```
┌─────────────────────────────────────────────────────────────┐
│                    ACID Explained                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   A - Atomicity (really means "Abortability")              │
│       NOT about concurrency - that's Isolation!            │
│       If transaction fails halfway, all changes undone     │
│       "All or nothing" - you can safely retry              │
│                                                             │
│   C - Consistency                                          │
│       Application-level invariants remain true             │
│       (e.g., "total money in system stays constant")       │
│       ⚠️ This is YOUR code's responsibility, not the DB!  │
│                                                             │
│   I - Isolation                                            │
│       Concurrent transactions don't see each other's       │
│       uncommitted changes. Each transaction runs as if     │
│       it's the only one in the system.                     │
│                                                             │
│   D - Durability                                           │
│       Once committed, data survives crashes                │
│       Achieved via: WAL, replication, fsync to disk        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

> **💡 Interview Tip:** The "C" in ACID is often called "the weakest letter" because consistency is really the application's job, not the database's. The DB just gives you tools (transactions) to maintain consistency. This is a great thing to mention to show deeper understanding.

**Why it works this way:** Atomicity uses a write-ahead log (WAL)—before changing data, the DB writes intentions to a log. If it crashes, it replays the log. Durability uses the same log plus fsync() to force data to disk.

### BASE vs ACID (NoSQL Trade-off)

| ACID | BASE |
|------|------|
| Atomicity | **B**asically **A**vailable |
| Consistency | **S**oft state |
| Isolation | **E**ventual consistency |
| Durability | |

**When to use ACID:** Financial systems, inventory management, anything where "close enough" isn't good enough.

**When BASE is OK:** Social media feeds, analytics, caching—where temporary inconsistency is acceptable for better availability/performance.

**Reality:** Most systems are somewhere in between! Even "NoSQL" databases like MongoDB now support multi-document transactions.

---

## Isolation Levels

**What it is:** Isolation levels are different "strengths" of protection against concurrency bugs. Higher isolation = more safety but lower performance.

**Why it matters:** Choosing the right isolation level is a core trade-off in system design. Too weak and you get data corruption bugs. Too strong and your system becomes slow or gets deadlocks.

### The Hierarchy

```
┌─────────────────────────────────────────────────────────────┐
│             Isolation Levels (Weakest → Strongest)           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   Read Uncommitted   ← Almost never used (can see garbage) │
│         ↓                                                   │
│   Read Committed     ← PostgreSQL/Oracle default (safe     │
│         ↓              for most apps)                       │
│   Repeatable Read    ← MySQL default                       │
│         ↓                                                   │
│   Snapshot Isolation ← Good balance - PostgreSQL calls     │
│         ↓              this "Repeatable Read"              │
│   Serializable       ← Strongest, can be slow              │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

> **💡 Interview Tip:** Know the default for major databases: PostgreSQL = Read Committed, MySQL = Repeatable Read. When asked "what isolation level would you use?", the answer is usually: "Read Committed for most operations, Serializable for critical sections like payment processing."

---

### Read Committed

**What it is:** You only see data that has been committed. You never see "in-progress" writes from other transactions.

**Why it matters:** This prevents you from reading garbage data that might get rolled back.

**Guarantees:**
1. **No dirty reads** - only see committed data
2. **No dirty writes** - only overwrite committed data

```
┌─────────────────────────────────────────────────────────────┐
│                   Dirty Read (Prevented)                     │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   Transaction 1              Transaction 2                  │
│   ─────────────              ─────────────                  │
│   UPDATE x = 3                                              │
│                              SELECT x  → Would see 3        │
│   ROLLBACK                   (dirty read!)                  │
│                                                             │
│   With Read Committed: T2 sees original value until commit │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**What it doesn't prevent:** Non-repeatable reads—you might read the same row twice and get different values because another transaction committed in between.

**When to use:** Most web applications. It's a good default unless you have specific requirements.

### Snapshot Isolation (Repeatable Read)

**What it is:** Each transaction sees a consistent "snapshot" of the database as it was at the start of the transaction. Other transactions' commits during your transaction are invisible to you.

**Why it matters:** Great for long-running read queries (reports, analytics) that need consistent data, and for preventing many concurrency bugs.

```
┌─────────────────────────────────────────────────────────────┐
│              Snapshot Isolation with MVCC                    │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   Transaction 1 (txid=100)   Transaction 2 (txid=101)      │
│   ────────────────────────   ────────────────────────      │
│   BEGIN                      BEGIN                          │
│   SELECT balance → $500      SELECT balance → $500         │
│                              UPDATE balance = $400          │
│                              COMMIT                         │
│   SELECT balance → $500      (T1 still sees $500!)         │
│   COMMIT                                                    │
│                                                             │
│   T1 sees consistent snapshot from when it started         │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**How it works (MVCC):** The database keeps multiple versions of each row. Instead of overwriting data, it creates new versions. Each transaction sees only versions that existed when it started.

```
┌─────────────────────────────────────────────────────────────┐
│                     MVCC Versioning                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   Row versions stored in database:                         │
│                                                             │
│   │ id │ balance │ created_by │ deleted_by │               │
│   ├────┼─────────┼────────────┼────────────┤               │
│   │ 1  │ $500    │ txid=50    │ txid=101   │  (old)       │
│   │ 1  │ $400    │ txid=101   │ NULL       │  (current)   │
│                                                             │
│   Transaction 100 visibility rules:                        │
│   - Skip rows where created_by > 100 (created after me)   │
│   - Skip rows where deleted_by <= 100 (deleted before me) │
│   - Result: Sees balance=$500                              │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

> **💡 Interview Tip:** MVCC is how databases achieve snapshot isolation without heavy locking. Readers never block writers, writers never block readers. This is why PostgreSQL can handle concurrent read-heavy workloads so well.

**When to use:** Analytics queries, reports, any read-heavy workload where you need point-in-time consistency.

---

## Concurrency Bugs (Race Conditions)

These are the bugs that isolation levels protect against. Understanding them is crucial for interviews.

### 1. Lost Update

**What it is:** Two transactions read the same value, modify it, and write back. The second write overwrites the first, losing that update.

**Why it happens:** Read-modify-write patterns without proper protection. Classic example: incrementing a counter.

```
┌─────────────────────────────────────────────────────────────┐
│                      Lost Update                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   Counter increment: Both transactions read, then write    │
│                                                             │
│   T1                         T2                            │
│   ──                         ──                            │
│   READ counter → 42          READ counter → 42             │
│   counter = 42 + 1           counter = 42 + 1              │
│   WRITE counter = 43         WRITE counter = 43            │
│                                                             │
│   Expected: 44   Actual: 43  (T1's update lost!)          │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Real-world examples:**
- Two users updating the same wiki page
- Incrementing "like" counts
- Reducing inventory during concurrent purchases

**Solutions (in order of preference):**

```sql
-- 1. Atomic operations (BEST - let the database handle it)
UPDATE counters SET value = value + 1 WHERE id = 'x';

-- 2. Explicit locking (SELECT FOR UPDATE)
SELECT * FROM counters WHERE id = 'x' FOR UPDATE;
UPDATE counters SET value = 43 WHERE id = 'x';

-- 3. Compare-and-set (CAS) - optimistic locking
UPDATE counters SET value = 43 
WHERE id = 'x' AND value = 42;  -- Fails if value changed

-- 4. Application-level version check
UPDATE products SET stock = 5, version = 2
WHERE id = 'x' AND version = 1;
```

> **💡 Interview Tip:** Always prefer atomic operations when possible. If asked "how would you implement a counter?", the answer is `UPDATE x SET count = count + 1`, NOT "read, increment in code, write back."

### 2. Write Skew

**What it is:** Two transactions each read the same data, make decisions based on what they read, and write different records. Individually valid, but combined they violate an invariant.

**Why it's tricky:** Each transaction is correct in isolation. The bug only appears when you look at both together. Snapshot isolation doesn't prevent this!

```
┌─────────────────────────────────────────────────────────────┐
│                     Write Skew                               │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   Business rule: At least 1 doctor must be on call         │
│   Current state: Alice and Bob both on call                │
│                                                             │
│   T1 (Alice wants off)         T2 (Bob wants off)          │
│   ────────────────────         ─────────────────           │
│   SELECT COUNT(*) WHERE        SELECT COUNT(*) WHERE       │
│     on_call = true → 2         on_call = true → 2          │
│   "2 on call, OK if I leave"   "2 on call, OK if I leave"  │
│   UPDATE Alice.on_call=false   UPDATE Bob.on_call=false   │
│   COMMIT                       COMMIT                      │
│                                                             │
│   Result: Nobody on call! Business rule violated.          │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Why snapshot isolation doesn't help:** Both transactions see a consistent snapshot where 2 doctors are on call. They write to different rows, so there's no write conflict.

**Other examples of write skew:**
- Meeting room double-booking
- Username uniqueness across shards
- Multiplayer game moves that conflict

**Solutions:**

```sql
-- 1. Materialize the conflict (create a row to lock)
SELECT * FROM on_call_slots WHERE shift = 'tonight' FOR UPDATE;
-- Now both transactions try to lock the SAME row

-- 2. Use serializable isolation
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;

-- 3. Application-level check with explicit lock
BEGIN;
SELECT * FROM doctors WHERE on_call = true FOR UPDATE;
-- Check count > 1, then proceed
UPDATE doctors SET on_call = false WHERE id = 'alice';
COMMIT;
```

> **💡 Interview Tip:** Write skew is a great example to bring up when discussing why "serializable is sometimes necessary." It shows you understand the limits of weaker isolation levels.

### 3. Phantom Reads

**What it is:** A transaction reads a set of rows matching a condition. Another transaction inserts a new row matching that condition. The first transaction's subsequent operations are based on stale information.

**Why it happens:** You can lock existing rows, but you can't lock rows that don't exist yet!

```
┌─────────────────────────────────────────────────────────────┐
│                    Phantom Reads                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   Meeting room booking: Check if slot is free, then book   │
│                                                             │
│   T1                              T2                       │
│   ──                              ──                       │
│   SELECT * FROM bookings                                   │
│   WHERE room=123 AND time='12:00'                          │
│   → Empty (room is free!)                                  │
│                                                             │
│                                   INSERT booking           │
│                                   room=123, time='12:00'   │
│                                   COMMIT                   │
│                                                             │
│   INSERT booking                                           │
│   room=123, time='12:00'                                   │
│   COMMIT                                                   │
│                                                             │
│   Result: Double booking! A "phantom" row appeared.        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Solutions:**

```sql
-- 1. Predicate locks (lock the CONDITION, not just rows)
-- "Lock all rows and future rows where room=123 AND time='12:00'"

-- 2. Index-range locks (practical approximation)
-- Lock the entire index entry for room 123

-- 3. Serializable isolation (database handles it)

-- 4. Materialize: pre-create all possible slots
INSERT INTO time_slots (room, time, booked) 
VALUES (123, '12:00', false);
-- Now you can SELECT FOR UPDATE on the slot
```

> **💡 Interview Tip:** Phantoms are why booking systems are tricky. If someone asks how to prevent double-booking, mention: (1) unique constraints in DB, (2) serializable isolation, or (3) pre-created time slots that you lock.

---

## Serializability Implementations

Three ways databases achieve true serializable isolation. Know the trade-offs!

### 1. Actual Serial Execution

**What it is:** Run transactions one at a time on a single thread. No concurrency = no concurrency bugs.

**Why it works:** If there's literally only one transaction running at a time, there can't be conflicts. Simple!

**Why it's viable now:** Modern RAM is cheap. If your data fits in memory, there's no disk I/O wait, so single-threaded can be surprisingly fast.

```
┌─────────────────────────────────────────────────────────────┐
│              Serial Execution (Single Thread)                │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   Queue: [T1] [T2] [T3] [T4]                              │
│                  │                                          │
│                  ▼                                          │
│          ┌──────────────┐                                  │
│          │ Single Thread│ ← Execute one at a time         │
│          │   Executor   │                                  │
│          └──────────────┘                                  │
│                                                             │
│   Requirements:                                            │
│   ✓ All data fits in RAM (no disk I/O waits)              │
│   ✓ Transactions are SHORT (use stored procedures)        │
│   ✓ Write throughput fits single CPU core                  │
│                                                             │
│   Used by: VoltDB, Redis, Datomic                          │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Critical requirement - stored procedures:**
```sql
-- BAD: Multiple network round trips (slow!)
BEGIN;
SELECT balance FROM accounts WHERE id = 1;  -- round trip 1
-- application thinks for 100ms
UPDATE accounts SET balance = balance - 100 WHERE id = 1;  -- round trip 2
UPDATE accounts SET balance = balance + 100 WHERE id = 2;  -- round trip 3
COMMIT;

-- GOOD: Single round trip with stored procedure
CREATE PROCEDURE transfer(from_id, to_id, amount) AS
BEGIN
    UPDATE accounts SET balance = balance - amount WHERE id = from_id;
    UPDATE accounts SET balance = balance + amount WHERE id = to_id;
END;

CALL transfer(1, 2, 100);  -- One round trip, microseconds
```

**Trade-offs:**
- ✅ Simple, no concurrency bugs possible
- ✅ Very fast for in-memory workloads
- ❌ Single CPU = limited throughput
- ❌ All transactions must be fast (can't have long-running queries)
- ❌ Can partition by key for parallelism, but cross-partition transactions are slow

### 2. Two-Phase Locking (2PL)

**What it is:** Transactions acquire locks as they access data. Reads get shared locks, writes get exclusive locks. All locks released at commit.

**Why "two-phase":** Phase 1 = acquire locks (growing), Phase 2 = release locks (shrinking). Once you release any lock, you can't acquire more.

```
┌─────────────────────────────────────────────────────────────┐
│               Two-Phase Locking (2PL)                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   Transaction lifecycle:                                   │
│                                                             │
│   BEGIN                                                    │
│   │ READ x  → acquire SHARED lock on x (others can read)  │
│   │ READ y  → acquire SHARED lock on y                    │
│   │ WRITE x → upgrade to EXCLUSIVE lock (blocks everyone) │
│   │ (cannot acquire new locks after first release)        │
│   COMMIT → release all locks at once                       │
│                                                             │
│   Lock compatibility:                                      │
│   ┌──────────┬────────┬───────────┐                       │
│   │          │ Shared │ Exclusive │                       │
│   ├──────────┼────────┼───────────┤                       │
│   │ Shared   │   ✅   │    ❌     │                       │
│   │ Exclusive│   ❌   │    ❌     │                       │
│   └──────────┴────────┴───────────┘                       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**The deadlock problem:**
```
T1: holds lock on A, wants lock on B
T2: holds lock on B, wants lock on A
→ Both wait forever (DEADLOCK)

Solution: Deadlock detector aborts one transaction
```

**Trade-offs:**
- ✅ Provides true serializability
- ✅ Works with disk-based data
- ❌ Readers block writers, writers block readers
- ❌ Deadlocks require detection and transaction abort
- ❌ Can have very high latency under contention

> **💡 Interview Tip:** 2PL is rarely used in modern systems for general workloads because the blocking significantly reduces throughput. It's mentioned for completeness and legacy systems.

### 3. Serializable Snapshot Isolation (SSI)

**What it is:** Optimistic concurrency control. Run transactions on snapshots without blocking, then check for conflicts at commit time. If conflict detected, abort and retry.

**Why it's popular:** Combines the performance benefits of snapshot isolation (readers never block) with true serializability. Best of both worlds.

```
┌─────────────────────────────────────────────────────────────┐
│        Serializable Snapshot Isolation (SSI)                 │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   Optimistic approach:                                     │
│   1. Execute transaction on snapshot (no locks!)           │
│   2. Track what was read and written                       │
│   3. At commit: check if any reads are now stale          │
│   4. If conflict → ABORT and retry                         │
│                                                             │
│   Example:                                                 │
│   T1: reads X (version 1), does computation               │
│   T2: writes X (creates version 2), commits               │
│   T1: tries to commit                                     │
│       → ABORT! (T1's read of X is stale)                  │
│                                                             │
│   The database asks: "Would executing these transactions  │
│   serially give the same result?" If not, abort one.      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**How conflict detection works:**
- Track reads: "T1 read row X at version V1"
- When T2 writes X: "T1's read is potentially stale"
- When T1 commits: Check if any of T1's reads were modified
- If yes: abort T1

**Trade-offs:**
- ✅ Readers never block writers (like snapshot isolation)
- ✅ Better throughput than 2PL for read-heavy workloads
- ✅ No deadlocks (only aborts)
- ❌ Transactions may need to retry if aborted
- ❌ Not suitable for high-contention workloads (too many aborts)

**Used by:** PostgreSQL (SERIALIZABLE mode), FoundationDB, CockroachDB

> **💡 Interview Tip:** SSI is the modern approach to serializability. If asked "how would you implement serializable isolation?", describe SSI's optimistic approach—it shows you understand modern database internals.

---

## Isolation Level Comparison

| Issue | Read Uncommitted | Read Committed | Snapshot | Serializable |
|-------|------------------|----------------|----------|--------------|
| Dirty Read | ❌ | ✅ | ✅ | ✅ |
| Dirty Write | ❌ | ✅ | ✅ | ✅ |
| Non-repeatable Read | ❌ | ❌ | ✅ | ✅ |
| Lost Update | ❌ | ❌ | Partial* | ✅ |
| Write Skew | ❌ | ❌ | ❌ | ✅ |
| Phantom | ❌ | ❌ | ❌ | ✅ |

*Snapshot isolation prevents lost updates in some databases (PostgreSQL), but not all.

> **💡 Interview Tip:** When asked "which isolation level should I use?", the answer depends on your use case:
> - **Most web apps:** Read Committed (fast, prevents obvious bugs)
> - **Reports/analytics:** Snapshot Isolation (consistent reads, no blocking)  
> - **Financial/booking systems:** Serializable (prevent all anomalies)

---

## Practical Patterns

### Idempotent Operations

**What it is:** An operation that produces the same result no matter how many times you execute it.

**Why it matters:** Networks are unreliable. Requests get retried. If your operation isn't idempotent, retries cause duplicate charges, double orders, etc.

```python
# ❌ Non-idempotent (dangerous with retries)
def charge_card(user_id, amount):
    create_charge(user_id, amount)  # Duplicate charges on retry!

# ✅ Idempotent (safe to retry)
def charge_card(user_id, amount, idempotency_key):
    if charge_exists(idempotency_key):
        return get_charge(idempotency_key)  # Return existing
    charge = create_charge(user_id, amount, idempotency_key)
    return charge

# Client generates unique key per logical operation
charge_card(user_id=123, amount=100, 
            idempotency_key="order_456_payment")
```

> **💡 Interview Tip:** Stripe, AWS, and most APIs use idempotency keys. Always mention this when discussing payment systems or operations that might be retried.

### Outbox Pattern (Transactional Messaging)

**What it is:** Write your business data AND a message record in the same DB transaction. A separate process publishes to Kafka/RabbitMQ.

**Why it matters:** You can't have an atomic transaction across a database and a message queue. This pattern solves it.

```
┌─────────────────────────────────────────────────────────────┐
│                    Outbox Pattern                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   BEGIN TRANSACTION;                                        │
│     INSERT INTO orders (...);                              │
│     INSERT INTO outbox (event_type, payload, sent=false); │
│   COMMIT;  -- Both or neither!                             │
│                                                             │
│   Separate process:                                        │
│     SELECT * FROM outbox WHERE sent = false;               │
│     → Publish to Kafka                                     │
│     UPDATE outbox SET sent = true;                         │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Saga Pattern (Distributed Transactions)

**What it is:** A sequence of local transactions. Each step triggers the next. Failures trigger compensating transactions.

**Why it matters:** True distributed transactions (2PC) don't scale. Sagas give eventual consistency with better availability.

```
┌─────────────────────────────────────────────────────────────┐
│                      Saga Pattern                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   1. Create order  →  2. Charge card  →  3. Reserve stock  │
│                                                             │
│   If step 3 fails, COMPENSATE backwards:                   │
│   3. (failed)  →  2. Refund card  →  1. Cancel order       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

> **💡 Interview Tip:** When asked about distributed transactions, say: (1) avoid them if possible, (2) use Sagas for cross-service workflows, (3) ensure idempotency so steps can be safely retried.

---

## Real-World Examples

### PostgreSQL Transaction

```sql
-- Serializable isolation
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;

-- Check balance (creates read dependency)
SELECT balance FROM accounts WHERE id = 1;

-- Transfer money
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;

-- If any serialization conflict, this will fail
COMMIT;
```

### MongoDB Multi-Document Transaction

```javascript
const session = client.startSession();
session.startTransaction({
    readConcern: { level: 'snapshot' },
    writeConcern: { w: 'majority' }
});

try {
    await accounts.updateOne(
        { _id: 'account1' },
        { $inc: { balance: -100 } },
        { session }
    );
    await accounts.updateOne(
        { _id: 'account2' },
        { $inc: { balance: 100 } },
        { session }
    );
    await session.commitTransaction();
} catch (error) {
    await session.abortTransaction();
}
```

---

## Key Takeaways

1. **ACID** is not binary - levels of isolation exist
2. **Read Committed** is usually the default (and often sufficient)
3. **Snapshot Isolation** prevents most anomalies except write skew
4. **Serializable** is safest but has performance cost
5. **Choose isolation level** based on data correctness requirements
6. **Design for idempotency** - operations may be retried
7. **Use atomic operations** over read-modify-write patterns

---

## Quick Reference

| Need | Use |
|------|-----|
| Counter increment | `UPDATE x SET val = val + 1` |
| Prevent double-spending | Serializable isolation |
| Read consistency in report | Snapshot isolation |
| Simple web app | Read Committed + app-level checks |
| Distributed transactions | Saga pattern |
| Reliable messaging | Outbox pattern |

---

## System Design Interview Tips

**When to bring up transactions in an interview:**
- Any time data consistency matters (payments, inventory, bookings)
- When discussing how to prevent race conditions
- When explaining trade-offs between consistency and performance

**Red flags interviewers look for:**
- Not considering concurrent access to shared data
- Using read-modify-write without protection
- Not understanding that "eventually consistent" means temporary wrong answers

**What sets great candidates apart:**
- Understanding that most databases default to Read Committed
- Knowing when Serializable is truly necessary vs. overkill
- Mentioning idempotency for retry-safety
- Understanding the limits of snapshot isolation (write skew)

---

## Common Interview Questions

**Q: How would you implement a money transfer between accounts?**
A: Use a single database transaction. Either use Serializable isolation, OR use `SELECT FOR UPDATE` on both accounts before updating. Always use atomic operations (`balance = balance - amount`), never read-modify-write in application code.

**Q: Two users are trying to book the same seat. How do you prevent double-booking?**
A: (1) Unique constraint in database on (event_id, seat_id), (2) Use SELECT FOR UPDATE when checking availability, or (3) Use Serializable isolation. For high-traffic systems, pre-create seat records and lock them with SELECT FOR UPDATE.

**Q: What's the difference between optimistic and pessimistic locking?**
A: Pessimistic (2PL): Acquire locks upfront, block others until done. Safe but slow under contention. Optimistic (SSI): Proceed without locks, check for conflicts at commit, retry if conflict. Better throughput when conflicts are rare.

**Q: How do you handle distributed transactions across microservices?**
A: Avoid if possible (restructure service boundaries). If necessary, use Sagas with compensating transactions. Ensure each step is idempotent. Accept eventual consistency rather than blocking distributed locks.

**Q: What isolation level would you use for a banking application?**
A: Serializable for actual money movements (transfers, payments). Snapshot isolation for balance queries and reports (consistent point-in-time view). Read Committed for non-critical reads (transaction history display).
