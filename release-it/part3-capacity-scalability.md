# Release It! - Part 3: Capacity & Scalability

> Designing systems that handle growth and traffic variations

---

## 1. Capacity Planning

### Concept
Capacity planning is about understanding current usage, predicting future growth, and ensuring your system can handle peak loads.

### Capacity Model
```
┌─────────────────────────────────────────────────────────────┐
│                 CAPACITY PLANNING MODEL                     │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                  TRAFFIC PATTERNS                    │   │
│  │                                                      │   │
│  │   Peak │            ╱╲                              │   │
│  │        │           ╱  ╲                             │   │
│  │        │     ╱╲   ╱    ╲   ╱╲                      │   │
│  │ Average│────╱──╲─╱──────╲─╱──╲────── Baseline     │   │
│  │        │                                            │   │
│  │   Low  │                                            │   │
│  │        └────────────────────────────────────►      │   │
│  │              Mon  Tue  Wed  Thu  Fri  Sat  Sun      │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  KEY METRICS:                                               │
│  • Average Load: What you handle normally                  │
│  • Peak Load: Maximum sustained traffic                    │
│  • Burst Load: Short spikes (often 10x peak)               │
│  • Headroom: Buffer for unexpected growth                  │
└─────────────────────────────────────────────────────────────┘
```

### Capacity Metrics to Track
```
┌─────────────────────────────────────────────────────────────┐
│                 KEY CAPACITY METRICS                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  THROUGHPUT METRICS                                         │
│  ├── Requests per second (RPS)                             │
│  ├── Transactions per second (TPS)                         │
│  ├── Messages per second                                   │
│  └── Data processed per second (MB/s)                      │
│                                                             │
│  LATENCY METRICS                                            │
│  ├── P50 (median) response time                            │
│  ├── P95 response time                                     │
│  ├── P99 response time                                     │
│  └── Max response time                                     │
│                                                             │
│  RESOURCE METRICS                                           │
│  ├── CPU utilization                                       │
│  ├── Memory usage                                          │
│  ├── Disk I/O                                              │
│  ├── Network bandwidth                                     │
│  └── Connection pool usage                                 │
│                                                             │
│  QUEUE METRICS                                              │
│  ├── Queue depth                                           │
│  ├── Queue wait time                                       │
│  └── Queue rejection rate                                  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Little's Law
```
┌─────────────────────────────────────────────────────────────┐
│                    LITTLE'S LAW                             │
│                                                             │
│          L = λ × W                                          │
│                                                             │
│   L = Average number of items in system                     │
│   λ = Average arrival rate                                  │
│   W = Average time in system                                │
│                                                             │
│  EXAMPLE:                                                   │
│  ┌──────────────────────────────────────────────────┐      │
│  │  If 100 requests/sec arrive (λ)                  │      │
│  │  And each takes 0.5 seconds to process (W)       │      │
│  │  Then 50 requests are "in flight" at any time (L)│      │
│  │                                                   │      │
│  │  You need at least 50 threads/connections!       │      │
│  └──────────────────────────────────────────────────┘      │
│                                                             │
│  IMPLICATIONS:                                              │
│  • To handle more traffic, reduce processing time          │
│  • Slow responses = more concurrent connections needed     │
│  • Double latency = double resources needed                │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. Scaling Strategies

### Vertical vs Horizontal Scaling
```
┌─────────────────────────────────────────────────────────────┐
│                 SCALING APPROACHES                          │
│                                                             │
│  VERTICAL SCALING (Scale Up)                                │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                                                      │   │
│  │   Before          After                             │   │
│  │   ┌────┐          ┌────────┐                        │   │
│  │   │    │          │        │                        │   │
│  │   │ 4  │    →     │   16   │                        │   │
│  │   │ GB │          │   GB   │                        │   │
│  │   │    │          │        │                        │   │
│  │   └────┘          │        │                        │   │
│  │                   └────────┘                        │   │
│  │                                                      │   │
│  │  ✓ Simple           ✗ Has limits                   │   │
│  │  ✓ No code changes  ✗ Expensive at high end        │   │
│  │  ✓ Less complexity  ✗ Single point of failure      │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  HORIZONTAL SCALING (Scale Out)                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                                                      │   │
│  │   Before          After                             │   │
│  │   ┌────┐          ┌────┐ ┌────┐ ┌────┐ ┌────┐     │   │
│  │   │    │          │    │ │    │ │    │ │    │     │   │
│  │   │ 4  │    →     │ 4  │ │ 4  │ │ 4  │ │ 4  │     │   │
│  │   │ GB │          │ GB │ │ GB │ │ GB │ │ GB │     │   │
│  │   │    │          │    │ │    │ │    │ │    │     │   │
│  │   └────┘          └────┘ └────┘ └────┘ └────┘     │   │
│  │                                                      │   │
│  │  ✓ Near-infinite scale  ✗ Requires stateless design │   │
│  │  ✓ Fault tolerant       ✗ More complex operations   │   │
│  │  ✓ Cost efficient       ✗ Data consistency harder   │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### Auto-Scaling Architecture
```
┌─────────────────────────────────────────────────────────────┐
│                  AUTO-SCALING SYSTEM                        │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                   METRICS                            │   │
│  │   CPU: 75%    Memory: 60%    RPS: 1000              │   │
│  └──────────────────────┬──────────────────────────────┘   │
│                         │                                   │
│                         ▼                                   │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              AUTO-SCALER RULES                       │   │
│  │                                                      │   │
│  │  IF CPU > 70% for 5 min → Add 2 instances           │   │
│  │  IF CPU < 30% for 10 min → Remove 1 instance        │   │
│  │  IF RPS > 800 per instance → Add 1 instance         │   │
│  │  MIN instances: 3                                    │   │
│  │  MAX instances: 20                                   │   │
│  └──────────────────────┬──────────────────────────────┘   │
│                         │                                   │
│           ┌─────────────┴─────────────┐                    │
│           ▼                           ▼                    │
│      Scale Out                   Scale In                   │
│    ┌───┐ ┌───┐ ┌───┐           ┌───┐ ┌───┐               │
│    │ + │ │ + │ │   │           │   │ │ - │               │
│    └───┘ └───┘ └───┘           └───┘ └───┘               │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Scaling Checklist
```
┌─────────────────────────────────────────────────────────────┐
│            SCALING READINESS CHECKLIST                      │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  STATELESS APPLICATION                                      │
│  ☐ No in-memory session state                              │
│  ☐ External session storage (Redis, DB)                    │
│  ☐ No local file dependencies                              │
│  ☐ Externalized configuration                              │
│                                                             │
│  DATABASE                                                   │
│  ☐ Read replicas configured                                │
│  ☐ Connection pooling tuned                                │
│  ☐ Sharding strategy defined                               │
│  ☐ Query performance optimized                             │
│                                                             │
│  CACHING                                                    │
│  ☐ Distributed cache (Redis/Memcached)                     │
│  ☐ Cache invalidation strategy                             │
│  ☐ Cache hit rate monitored                                │
│                                                             │
│  LOAD BALANCING                                             │
│  ☐ Health checks configured                                │
│  ☐ Session affinity if needed                              │
│  ☐ Graceful drain on scale-down                            │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 3. Caching Strategies

### Cache Architecture
```
┌─────────────────────────────────────────────────────────────┐
│                 CACHING LAYERS                              │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                    CLIENT                            │   │
│  └───────────────────────┬─────────────────────────────┘   │
│                          │                                  │
│                          ▼                                  │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              BROWSER CACHE                           │   │
│  │          (HTTP Cache-Control headers)                │   │
│  └───────────────────────┬─────────────────────────────┘   │
│                          │                                  │
│                          ▼                                  │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                   CDN CACHE                          │   │
│  │              (Static assets, API responses)          │   │
│  └───────────────────────┬─────────────────────────────┘   │
│                          │                                  │
│                          ▼                                  │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              APPLICATION CACHE                       │   │
│  │              (Redis, Memcached)                      │   │
│  └───────────────────────┬─────────────────────────────┘   │
│                          │                                  │
│                          ▼                                  │
│  ┌─────────────────────────────────────────────────────┐   │
│  │               DATABASE CACHE                         │   │
│  │           (Query cache, Buffer pool)                 │   │
│  └───────────────────────┬─────────────────────────────┘   │
│                          │                                  │
│                          ▼                                  │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                  DATABASE                            │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### Cache Patterns

#### Cache-Aside (Lazy Loading)
```
┌─────────────────────────────────────────────────────────────┐
│                 CACHE-ASIDE PATTERN                         │
│                                                             │
│   Application                                               │
│       │                                                     │
│       │ 1. Check cache                                      │
│       ▼                                                     │
│   ┌───────┐                                                │
│   │ Cache │ ──► HIT? ──► Return data                       │
│   └───────┘                                                │
│       │                                                     │
│       │ MISS                                                │
│       ▼                                                     │
│   ┌───────┐                                                │
│   │  DB   │ 2. Fetch from database                         │
│   └───────┘                                                │
│       │                                                     │
│       │ 3. Store in cache                                   │
│       ▼                                                     │
│   ┌───────┐                                                │
│   │ Cache │                                                │
│   └───────┘                                                │
│       │                                                     │
│       │ 4. Return data                                      │
│       ▼                                                     │
│   Application                                               │
└─────────────────────────────────────────────────────────────┘
```

```python
def get_user(user_id):
    # Try cache first
    cached = cache.get(f"user:{user_id}")
    if cached:
        return cached
    
    # Cache miss - fetch from DB
    user = db.query("SELECT * FROM users WHERE id = ?", user_id)
    
    # Store in cache for next time
    cache.set(f"user:{user_id}", user, ttl=300)
    
    return user
```

#### Write-Through Cache
```
┌─────────────────────────────────────────────────────────────┐
│               WRITE-THROUGH PATTERN                         │
│                                                             │
│   Application                                               │
│       │                                                     │
│       │ 1. Write to cache                                   │
│       ▼                                                     │
│   ┌───────┐                                                │
│   │ Cache │                                                │
│   └───┬───┘                                                │
│       │                                                     │
│       │ 2. Cache writes to DB (synchronous)                │
│       ▼                                                     │
│   ┌───────┐                                                │
│   │  DB   │                                                │
│   └───────┘                                                │
│       │                                                     │
│       │ 3. Confirm write complete                          │
│       ▼                                                     │
│   Application                                               │
│                                                             │
│  ✓ Data consistency guaranteed                             │
│  ✗ Higher write latency                                    │
└─────────────────────────────────────────────────────────────┘
```

#### Write-Behind (Write-Back) Cache
```
┌─────────────────────────────────────────────────────────────┐
│               WRITE-BEHIND PATTERN                          │
│                                                             │
│   Application                                               │
│       │                                                     │
│       │ 1. Write to cache (returns immediately)            │
│       ▼                                                     │
│   ┌───────┐                                                │
│   │ Cache │                                                │
│   └───┬───┘                                                │
│       │                                                     │
│       │ 2. Async batch write to DB (later)                 │
│       ▼                                                     │
│   ┌───────┐                                                │
│   │  DB   │                                                │
│   └───────┘                                                │
│                                                             │
│  ✓ Very fast writes                                        │
│  ✗ Risk of data loss if cache fails                       │
│  ✗ Eventual consistency only                               │
└─────────────────────────────────────────────────────────────┘
```

### Cache Invalidation Strategies
```
┌─────────────────────────────────────────────────────────────┐
│            CACHE INVALIDATION STRATEGIES                    │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  TTL (Time-To-Live)                                        │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  cache.set("key", value, ttl=300)  # Expires in 5min│   │
│  │                                                      │   │
│  │  ✓ Simple to implement                              │   │
│  │  ✗ Stale data until expiry                          │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  Event-Based Invalidation                                   │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  def update_user(user):                              │   │
│  │      db.update(user)                                 │   │
│  │      cache.delete(f"user:{user.id}")  # Invalidate  │   │
│  │                                                      │   │
│  │  ✓ Immediate consistency                            │   │
│  │  ✗ Must track all cache keys                        │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  Version-Based Invalidation                                 │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  cache_key = f"user:{user.id}:v{user.version}"      │   │
│  │                                                      │   │
│  │  ✓ Old entries naturally expire                     │   │
│  │  ✗ Storage overhead for old versions                │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 4. Connection Pooling

### Why Connection Pooling?
```
┌─────────────────────────────────────────────────────────────┐
│              CONNECTION COST                                │
│                                                             │
│  WITHOUT POOLING:                                           │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  Request 1 ──► Create Conn ──► Query ──► Close Conn │   │
│  │                    │                        │        │   │
│  │               ~50-100ms               wasted!        │   │
│  │                                                      │   │
│  │  Request 2 ──► Create Conn ──► Query ──► Close Conn │   │
│  │                    │                        │        │   │
│  │               ~50-100ms               wasted!        │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  WITH POOLING:                                              │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  Pool: [Conn1] [Conn2] [Conn3] [Conn4] [Conn5]      │   │
│  │                                                      │   │
│  │  Request 1 ──► Borrow Conn1 ──► Query ──► Return    │   │
│  │                    │                                 │   │
│  │                  <1ms                                │   │
│  │                                                      │   │
│  │  Request 2 ──► Borrow Conn2 ──► Query ──► Return    │   │
│  │                    │                                 │   │
│  │                  <1ms                                │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### Pool Configuration
```
┌─────────────────────────────────────────────────────────────┐
│              POOL SIZING FORMULA                            │
│                                                             │
│   Pool Size = (Core Count × 2) + Disk Spindles              │
│                                                             │
│   For SSDs/NVMe: Pool Size ≈ Core Count × 2                │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  EXAMPLE CONFIGURATION (HikariCP)                    │   │
│  │                                                      │   │
│  │  minimumIdle: 5          # Keep 5 connections warm  │   │
│  │  maximumPoolSize: 20     # Never exceed 20          │   │
│  │  connectionTimeout: 30s  # Wait max 30s for conn    │   │
│  │  idleTimeout: 600s       # Close idle after 10min   │   │
│  │  maxLifetime: 1800s      # Recycle every 30min      │   │
│  │  leakDetectionThreshold: 60s  # Warn if held >60s   │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  WARNING: More connections ≠ better performance!           │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                                                      │   │
│  │  Throughput                                          │   │
│  │      │        ╱──── Optimal                         │   │
│  │      │       ╱                                       │   │
│  │      │      ╱       ────────── Diminishing returns  │   │
│  │      │     ╱                 ╲                       │   │
│  │      │    ╱                   ╲── Contention        │   │
│  │      │   ╱                                          │   │
│  │      └──────────────────────────────────►           │   │
│  │                Pool Size                             │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### Connection Leak Detection
```java
// HikariCP leak detection
HikariConfig config = new HikariConfig();
config.setLeakDetectionThreshold(60000); // 60 seconds

// If connection held longer than threshold, logs warning:
// "Connection leak detection triggered for connection"
// with stack trace of where connection was obtained

// Common leak causes:
// 1. Missing finally block to close connection
// 2. Exception before connection.close()
// 3. Long-running transactions
```

---

## 5. Load Shedding

### Concept
When system is overloaded, reject some requests to protect the system and serve remaining requests well.

### Load Shedding Strategies
```
┌─────────────────────────────────────────────────────────────┐
│               LOAD SHEDDING APPROACHES                      │
│                                                             │
│  1. QUEUE-BASED SHEDDING                                   │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                                                      │   │
│  │  Incoming ──► [Queue: max 1000] ──► Processing      │   │
│  │     │                                                │   │
│  │     │ Queue full? → Reject immediately (503)        │   │
│  │     ▼                                                │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  2. DEADLINE-BASED SHEDDING                                │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                                                      │   │
│  │  Request arrives with deadline (e.g., 5s)           │   │
│  │       │                                              │   │
│  │       │ Time in queue > deadline?                   │   │
│  │       │ → Drop request (too late to matter)         │   │
│  │       ▼                                              │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  3. PRIORITY-BASED SHEDDING                                │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                                                      │   │
│  │  High Priority: Premium users, purchases            │   │
│  │  ───────────────────────────► Always processed     │   │
│  │                                                      │   │
│  │  Low Priority: Analytics, recommendations           │   │
│  │  ───────────────────────────► Shed first           │   │
│  │                                                      │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  4. RANDOM SHEDDING                                        │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                                                      │   │
│  │  At 80% capacity: reject 10% randomly               │   │
│  │  At 90% capacity: reject 30% randomly               │   │
│  │  At 95% capacity: reject 50% randomly               │   │
│  │                                                      │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### Implementation Example
```python
import time
from threading import Semaphore

class LoadShedder:
    def __init__(self, max_concurrent=100, queue_timeout=5.0):
        self.semaphore = Semaphore(max_concurrent)
        self.queue_timeout = queue_timeout
    
    def process(self, request):
        # Try to acquire with timeout
        acquired = self.semaphore.acquire(timeout=self.queue_timeout)
        
        if not acquired:
            # System overloaded - shed this request
            raise ServiceOverloadedException(
                "System busy, please retry later",
                retry_after=30
            )
        
        try:
            return handle_request(request)
        finally:
            self.semaphore.release()

# HTTP response for shed requests
# HTTP 503 Service Unavailable
# Retry-After: 30
```

### Graceful Degradation
```
┌─────────────────────────────────────────────────────────────┐
│              GRACEFUL DEGRADATION LEVELS                    │
│                                                             │
│  LEVEL 0: Normal Operation                                  │
│  ├── All features enabled                                  │
│  └── Full functionality                                    │
│                                                             │
│  LEVEL 1: Light Load (70% capacity)                        │
│  ├── Disable non-essential analytics                       │
│  └── Reduce recommendation quality                         │
│                                                             │
│  LEVEL 2: Moderate Load (85% capacity)                     │
│  ├── Serve cached data even if stale                       │
│  ├── Disable real-time features                            │
│  └── Simplify search results                               │
│                                                             │
│  LEVEL 3: Heavy Load (95% capacity)                        │
│  ├── Static fallback pages                                 │
│  ├── Queue-only for writes                                 │
│  └── Read-only mode                                        │
│                                                             │
│  LEVEL 4: Emergency (99% capacity)                         │
│  ├── Maintenance page for non-essential traffic           │
│  └── Only critical paths operational                       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 6. Backpressure

### Concept
Backpressure allows downstream systems to signal upstream when they can't keep up, preventing overload.

### Backpressure Mechanisms
```
┌─────────────────────────────────────────────────────────────┐
│              BACKPRESSURE FLOW                              │
│                                                             │
│  WITHOUT BACKPRESSURE:                                      │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                                                      │   │
│  │  Producer ═══════════════════════════► Consumer     │   │
│  │  (fast)   ████████████████████████████   (slow)     │   │
│  │           ↑                                          │   │
│  │           └── Unbounded buffer grows forever        │   │
│  │               → Memory exhaustion → Crash           │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  WITH BACKPRESSURE:                                         │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                                                      │   │
│  │  Producer ══════[FULL]═══════════════► Consumer     │   │
│  │  (waits)  ◄───── "slow down!" ────────  (slow)      │   │
│  │                                                      │   │
│  │  Producer adjusts rate to match consumer capacity   │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Implementation Strategies
```
┌─────────────────────────────────────────────────────────────┐
│           BACKPRESSURE STRATEGIES                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. BLOCKING                                               │
│     Producer blocks when buffer is full                    │
│     ✓ Simple    ✗ Can cause thread starvation             │
│                                                             │
│  2. DROP                                                   │
│     Discard new items when buffer is full                  │
│     ✓ Protects memory    ✗ Data loss                      │
│                                                             │
│  3. SAMPLE                                                 │
│     Keep only every Nth item when overloaded               │
│     ✓ Reduces load    ✗ Loses data                        │
│                                                             │
│  4. BUFFER WITH TIMEOUT                                    │
│     Queue items with expiry, drop if too old               │
│     ✓ Balanced    ✗ Complex                               │
│                                                             │
│  5. ADAPTIVE RATE                                          │
│     Producer adjusts rate based on feedback                │
│     ✓ Optimal throughput    ✗ Requires coordination       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Example: Reactive Streams
```java
// Java Flow API with backpressure
public class BackpressureProcessor implements Flow.Processor<Data, Data> {
    private Flow.Subscription subscription;
    private Flow.Subscriber<? super Data> subscriber;
    
    @Override
    public void onSubscribe(Flow.Subscription subscription) {
        this.subscription = subscription;
        // Request only what we can handle
        subscription.request(10);  // Start with 10 items
    }
    
    @Override
    public void onNext(Data item) {
        // Process item
        Data processed = process(item);
        subscriber.onNext(processed);
        
        // Request more only when ready
        if (canHandleMore()) {
            subscription.request(1);  // Pull-based backpressure
        }
    }
}
```

---

## 7. Database Scaling Patterns

### Read Replicas
```
┌─────────────────────────────────────────────────────────────┐
│                READ REPLICA PATTERN                         │
│                                                             │
│                    ┌─────────────┐                         │
│                    │   Primary   │                         │
│                    │   (Write)   │                         │
│                    └──────┬──────┘                         │
│                           │                                 │
│              ┌────────────┼────────────┐                   │
│              │ Replication│            │                   │
│              ▼            ▼            ▼                   │
│        ┌──────────┐ ┌──────────┐ ┌──────────┐            │
│        │ Replica  │ │ Replica  │ │ Replica  │            │
│        │  (Read)  │ │  (Read)  │ │  (Read)  │            │
│        └──────────┘ └──────────┘ └──────────┘            │
│              ▲            ▲            ▲                   │
│              └────────────┼────────────┘                   │
│                           │                                 │
│                    ┌──────────────┐                        │
│                    │ Application  │                        │
│                    │ Write→Primary│                        │
│                    │ Read→Replica │                        │
│                    └──────────────┘                        │
│                                                             │
│  ⚠️ Eventual consistency - reads may be slightly stale    │
└─────────────────────────────────────────────────────────────┘
```

### Sharding
```
┌─────────────────────────────────────────────────────────────┐
│                 DATABASE SHARDING                           │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │               SHARDING STRATEGIES                    │   │
│  │                                                      │   │
│  │  1. HASH-BASED:  shard = hash(user_id) % num_shards│   │
│  │                                                      │   │
│  │  2. RANGE-BASED: shard = user_id / 1000000         │   │
│  │                                                      │   │
│  │  3. GEOGRAPHY:   shard = user.region               │   │
│  │                                                      │   │
│  │  4. TENANT:      shard = company_id                │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                                                      │   │
│  │    Request: GET user 12345                          │   │
│  │         │                                            │   │
│  │         ▼                                            │   │
│  │    Shard Router: hash(12345) % 4 = 1                │   │
│  │         │                                            │   │
│  │         ▼                                            │   │
│  │  ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐                   │   │
│  │  │ S0  │ │ S1 ◄│ │ S2  │ │ S3  │                   │   │
│  │  │     │ │     │ │     │ │     │                   │   │
│  │  │0-25%│ │25-50│ │50-75│ │75-100                   │   │
│  │  └─────┘ └─────┘ └─────┘ └─────┘                   │   │
│  │                                                      │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  CHALLENGES:                                                │
│  • Cross-shard queries are expensive                       │
│  • Rebalancing when adding shards                          │
│  • Maintaining referential integrity                       │
└─────────────────────────────────────────────────────────────┘
```

---

## Summary: Capacity & Scalability Principles

```
┌────────────────────────────────────────────────────────────┐
│           CAPACITY PLANNING PRINCIPLES                     │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  1. MEASURE FIRST                                          │
│     • Know your current capacity                           │
│     • Understand traffic patterns                          │
│     • Identify bottlenecks                                 │
│                                                            │
│  2. PLAN FOR PEAKS                                         │
│     • Design for 2-3x expected peak                        │
│     • Have auto-scaling ready                              │
│     • Test at scale                                        │
│                                                            │
│  3. DEGRADE GRACEFULLY                                     │
│     • Have multiple degradation levels                     │
│     • Protect core functionality                           │
│     • Shed load intelligently                              │
│                                                            │
│  4. CACHE STRATEGICALLY                                    │
│     • Cache at multiple levels                             │
│     • Have invalidation strategy                           │
│     • Monitor cache hit rates                              │
│                                                            │
│  5. SCALE HORIZONTALLY                                     │
│     • Design for statelessness                             │
│     • Use distributed systems                              │
│     • Avoid single points of failure                       │
│                                                            │
└────────────────────────────────────────────────────────────┘
```
