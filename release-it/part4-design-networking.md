# Release It! - Part 4: General Design Issues & Networking

> Architectural decisions and network considerations for production systems

---

## 1. Networking Fundamentals

### The Fallacies of Distributed Computing
```
┌─────────────────────────────────────────────────────────────┐
│         THE 8 FALLACIES OF DISTRIBUTED COMPUTING            │
│                                                             │
│  ❌ FALLACY 1: The network is reliable                      │
│     Reality: Packets get lost, connections drop             │
│                                                             │
│  ❌ FALLACY 2: Latency is zero                              │
│     Reality: Every call has overhead (often 1-100ms)        │
│                                                             │
│  ❌ FALLACY 3: Bandwidth is infinite                        │
│     Reality: Networks get congested                         │
│                                                             │
│  ❌ FALLACY 4: The network is secure                        │
│     Reality: Data can be intercepted/modified               │
│                                                             │
│  ❌ FALLACY 5: Topology doesn't change                      │
│     Reality: Routes, IPs, and servers change                │
│                                                             │
│  ❌ FALLACY 6: There is one administrator                   │
│     Reality: Multiple teams, vendors, ISPs involved         │
│                                                             │
│  ❌ FALLACY 7: Transport cost is zero                       │
│     Reality: Serialization, bandwidth cost money/time       │
│                                                             │
│  ❌ FALLACY 8: The network is homogeneous                   │
│     Reality: Different protocols, versions, platforms       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Network Failure Modes
```
┌─────────────────────────────────────────────────────────────┐
│               NETWORK FAILURE MODES                         │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  COMPLETE FAILURE                                    │   │
│  │  ┌─────┐     ✗ ✗ ✗     ┌─────┐                     │   │
│  │  │  A  │ ──────────── │  B  │                      │   │
│  │  └─────┘               └─────┘                      │   │
│  │  Connection refused, network unreachable            │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  PARTIAL FAILURE (Split Brain)                       │   │
│  │  ┌─────┐             ┌─────┐                        │   │
│  │  │  A  │ ◄─────────► │  B  │                        │   │
│  │  └─────┘      ▲      └─────┘                        │   │
│  │               │                                      │   │
│  │               ✗  Partition                          │   │
│  │               │                                      │   │
│  │  ┌─────┐      ▼      ┌─────┐                        │   │
│  │  │  C  │ ◄─────────► │  D  │                        │   │
│  │  └─────┘             └─────┘                        │   │
│  │  A↔B works, C↔D works, but A↔C is broken           │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  ASYMMETRIC FAILURE                                  │   │
│  │  ┌─────┐  ──────────► ┌─────┐                       │   │
│  │  │  A  │              │  B  │                        │   │
│  │  └─────┘  ◄─── ✗ ───  └─────┘                       │   │
│  │  A can send to B, but B's replies don't reach A     │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  DEGRADED PERFORMANCE                                │   │
│  │  ┌─────┐  ═══════════ ┌─────┐                       │   │
│  │  │  A  │  slow/lossy  │  B  │                        │   │
│  │  └─────┘  ═══════════ └─────┘                       │   │
│  │  High latency, packet loss, but not complete failure│   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### TCP Connection States
```
┌─────────────────────────────────────────────────────────────┐
│              TCP CONNECTION LIFECYCLE                       │
│                                                             │
│  CLIENT                                    SERVER           │
│    │                                          │             │
│    │ ──────── SYN ─────────────────────────► │             │
│    │                                          │             │
│    │ ◄─────── SYN-ACK ─────────────────────  │             │
│    │                                          │             │
│    │ ──────── ACK ─────────────────────────► │             │
│    │                                          │             │
│    │ ════════ ESTABLISHED ═══════════════════│             │
│    │          (data transfer)                 │             │
│    │                                          │             │
│    │ ──────── FIN ─────────────────────────► │             │
│    │                                          │             │
│    │ ◄─────── ACK ─────────────────────────  │             │
│    │                                          │             │
│    │ ◄─────── FIN ─────────────────────────  │             │
│    │                                          │             │
│    │ ──────── ACK ─────────────────────────► │             │
│    │                                          │             │
│    │ ════════ TIME_WAIT (2*MSL) ═════════════│             │
│    │                                          │             │
│    ▼                                          ▼             │
│                                                             │
│  ⚠️ TIME_WAIT can exhaust ports if connections churn fast  │
│  ⚠️ SYN_RECV queue can fill during SYN flood attacks       │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. DNS Considerations

### DNS Failure Scenarios
```
┌─────────────────────────────────────────────────────────────┐
│                 DNS FAILURE MODES                           │
│                                                             │
│  1. DNS SERVER DOWN                                         │
│     Application → DNS → ✗ TIMEOUT                          │
│     Result: All service discovery fails                    │
│                                                             │
│  2. STALE DNS CACHE                                        │
│     Old IP: 10.0.1.5  →  Server moved to: 10.0.2.10       │
│     Result: Requests go to wrong/dead server               │
│                                                             │
│  3. DNS PROPAGATION DELAY                                  │
│     Change made → Takes 0-48 hours to propagate            │
│     Result: Different clients see different IPs            │
│                                                             │
│  4. DNS TTL TOO HIGH                                       │
│     TTL: 86400 (24 hours)                                  │
│     Result: Can't quickly failover to new servers          │
│                                                             │
│  5. DNS TTL TOO LOW                                        │
│     TTL: 30 seconds                                        │
│     Result: Excessive DNS queries, added latency           │
└─────────────────────────────────────────────────────────────┘
```

### DNS Best Practices
```
┌─────────────────────────────────────────────────────────────┐
│               DNS BEST PRACTICES                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  TTL RECOMMENDATIONS:                                       │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  Production services:     60-300 seconds            │   │
│  │  During migration:        30-60 seconds             │   │
│  │  Static content/CDN:      3600+ seconds             │   │
│  │  Internal services:       30-60 seconds             │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  RESILIENCE:                                                │
│  • Use multiple DNS providers                              │
│  • Cache DNS locally with fallback                         │
│  • Monitor DNS resolution time                             │
│  • Have IP-based fallback for critical services            │
│                                                             │
│  JVM DNS CACHING:                                          │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  # JVM caches DNS forever by default!               │   │
│  │  # Set in java.security or programmatically:        │   │
│  │  networkaddress.cache.ttl=60                        │   │
│  │  networkaddress.cache.negative.ttl=10               │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 3. Load Balancing Strategies

### Load Balancer Types
```
┌─────────────────────────────────────────────────────────────┐
│              LOAD BALANCER TYPES                            │
│                                                             │
│  LAYER 4 (Transport)                                        │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  Client ──► LB ──► Server                           │   │
│  │                                                      │   │
│  │  • Routes based on IP + Port                        │   │
│  │  • Very fast (no payload inspection)                │   │
│  │  • Can't route based on URL/headers                 │   │
│  │  • Examples: AWS NLB, HAProxy (TCP mode)            │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  LAYER 7 (Application)                                      │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  Client ──► LB ──► Server                           │   │
│  │                                                      │   │
│  │  • Routes based on URL, headers, cookies            │   │
│  │  • Can do SSL termination                           │   │
│  │  • Can modify requests/responses                    │   │
│  │  • More CPU intensive                               │   │
│  │  • Examples: AWS ALB, Nginx, HAProxy (HTTP mode)    │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Load Balancing Algorithms
```
┌─────────────────────────────────────────────────────────────┐
│           LOAD BALANCING ALGORITHMS                         │
│                                                             │
│  ROUND ROBIN                                                │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  Request 1 → Server A                               │   │
│  │  Request 2 → Server B                               │   │
│  │  Request 3 → Server C                               │   │
│  │  Request 4 → Server A (cycle repeats)               │   │
│  │                                                      │   │
│  │  ✓ Simple    ✗ Ignores server load                 │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  WEIGHTED ROUND ROBIN                                       │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  Server A (weight 3): Gets 3x traffic              │   │
│  │  Server B (weight 1): Gets 1x traffic              │   │
│  │                                                      │   │
│  │  Pattern: A, A, A, B, A, A, A, B...                 │   │
│  │                                                      │   │
│  │  ✓ Handles different server capacities             │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  LEAST CONNECTIONS                                          │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  Server A: 10 connections                           │   │
│  │  Server B: 5 connections  ◄── Next request goes here│   │
│  │  Server C: 8 connections                            │   │
│  │                                                      │   │
│  │  ✓ Better for varying request times                │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  IP HASH (Sticky Sessions)                                  │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  hash(client_ip) % num_servers = target             │   │
│  │                                                      │   │
│  │  Client 1.2.3.4 → Always Server A                   │   │
│  │  Client 5.6.7.8 → Always Server B                   │   │
│  │                                                      │   │
│  │  ✓ Session affinity    ✗ Uneven distribution       │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  LEAST RESPONSE TIME                                        │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  Routes to server with lowest response time         │   │
│  │                                                      │   │
│  │  Server A: avg 50ms                                 │   │
│  │  Server B: avg 30ms  ◄── Preferred                  │   │
│  │  Server C: avg 45ms                                 │   │
│  │                                                      │   │
│  │  ✓ Optimizes for latency                           │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### Health Checks
```
┌─────────────────────────────────────────────────────────────┐
│                  HEALTH CHECK TYPES                         │
│                                                             │
│  TCP HEALTH CHECK                                           │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  LB ──► Open TCP connection to port 8080            │   │
│  │  If connection succeeds → Server healthy            │   │
│  │                                                      │   │
│  │  ✓ Fast    ✗ Process may be up but broken          │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  HTTP HEALTH CHECK                                          │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  LB ──► GET /health                                 │   │
│  │  If HTTP 200 → Server healthy                       │   │
│  │                                                      │   │
│  │  ✓ Verifies app responding    ✗ Slower             │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  DEEP HEALTH CHECK                                          │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  GET /health/deep                                   │   │
│  │  {                                                   │   │
│  │    "status": "healthy",                             │   │
│  │    "database": "connected",                         │   │
│  │    "cache": "connected",                            │   │
│  │    "disk_space": "ok",                              │   │
│  │    "memory": "ok"                                   │   │
│  │  }                                                   │   │
│  │                                                      │   │
│  │  ✓ Comprehensive    ✗ Can cause cascading failures │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  ⚠️ WARNING: Deep health checks on dependencies can cause │
│     your service to go unhealthy when dependency is down  │
│     Consider: shallow checks for LB, deep checks for alerts│
└─────────────────────────────────────────────────────────────┘
```

---

## 4. Service Discovery

### Service Discovery Patterns
```
┌─────────────────────────────────────────────────────────────┐
│            SERVICE DISCOVERY PATTERNS                       │
│                                                             │
│  CLIENT-SIDE DISCOVERY                                      │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                                                      │   │
│  │   Client ──► Registry ──► Get service locations     │   │
│  │     │                                                │   │
│  │     │ ──────────────────────► Service Instance A    │   │
│  │     │        (direct call)                          │   │
│  │                                                      │   │
│  │   ✓ Client has full control                        │   │
│  │   ✓ No extra hop                                   │   │
│  │   ✗ Client complexity                              │   │
│  │   Examples: Netflix Eureka, Consul                  │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  SERVER-SIDE DISCOVERY                                      │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                                                      │   │
│  │   Client ──► Load Balancer ──► Service Instance     │   │
│  │                   │                                  │   │
│  │                   └──► Registry (for routing)       │   │
│  │                                                      │   │
│  │   ✓ Simple client                                  │   │
│  │   ✓ Language agnostic                              │   │
│  │   ✗ Extra network hop                              │   │
│  │   Examples: AWS ALB, Kubernetes Service             │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  SERVICE MESH                                               │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                                                      │   │
│  │   ┌─────────┐        ┌─────────┐                   │   │
│  │   │ Service │        │ Service │                   │   │
│  │   │    A    │        │    B    │                   │   │
│  │   └────┬────┘        └────┬────┘                   │   │
│  │        │                  │                         │   │
│  │   ┌────┴────┐        ┌────┴────┐                   │   │
│  │   │ Sidecar │◄──────►│ Sidecar │                   │   │
│  │   │ Proxy   │        │ Proxy   │                   │   │
│  │   └─────────┘        └─────────┘                   │   │
│  │                                                      │   │
│  │   ✓ Transparent to application                     │   │
│  │   ✓ Advanced features (mTLS, tracing)              │   │
│  │   ✗ Operational complexity                         │   │
│  │   Examples: Istio, Linkerd, Consul Connect          │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### Service Registry
```
┌─────────────────────────────────────────────────────────────┐
│              SERVICE REGISTRY FLOW                          │
│                                                             │
│  1. REGISTRATION                                            │
│     ┌──────────┐    "I'm alive at 10.0.1.5:8080"          │
│     │ Service  │ ─────────────────────────────►            │
│     │ Instance │                           ┌──────────┐    │
│     └──────────┘                           │ Registry │    │
│                                            │          │    │
│  2. HEARTBEAT                              │ Services:│    │
│     ┌──────────┐    "Still alive" (every 30s)│ - A: 10.│    │
│     │ Service  │ ─────────────────────────────│ - B: 10.│    │
│     │ Instance │                              └──────────┘    │
│     └──────────┘                                            │
│                                                             │
│  3. DISCOVERY                                               │
│     ┌──────────┐    "Where is Service A?"                  │
│     │  Client  │ ─────────────────────────────►            │
│     └──────────┘                           ┌──────────┐    │
│                    ◄───────────────────────│ Registry │    │
│                    "10.0.1.5:8080, 10.0.1.6:8080"         │
│                                            └──────────┘    │
│                                                             │
│  4. DEREGISTRATION (on shutdown or missed heartbeats)      │
│     Registry removes service after N missed heartbeats     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 5. API Design for Resilience

### Idempotency
```
┌─────────────────────────────────────────────────────────────┐
│                   IDEMPOTENCY                               │
│                                                             │
│  Definition: Operation can be applied multiple times        │
│              without changing result beyond first apply     │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  IDEMPOTENT OPERATIONS                              │   │
│  │                                                      │   │
│  │  GET /users/123        → Same user every time       │   │
│  │  PUT /users/123        → Same final state           │   │
│  │  DELETE /users/123     → User deleted (or already)  │   │
│  │  SET balance = 100     → Balance is 100             │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  NON-IDEMPOTENT OPERATIONS                          │   │
│  │                                                      │   │
│  │  POST /users           → Creates new user each time │   │
│  │  POST /orders          → Creates duplicate orders!  │   │
│  │  balance += 100        → Keeps incrementing!        │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  MAKING NON-IDEMPOTENT OPERATIONS SAFE:                    │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                                                      │   │
│  │  Use idempotency key:                               │   │
│  │                                                      │   │
│  │  POST /orders                                       │   │
│  │  Idempotency-Key: abc-123-xyz                       │   │
│  │  {order data}                                       │   │
│  │                                                      │   │
│  │  Server checks: "Have I seen abc-123-xyz before?"   │   │
│  │  If yes → Return cached response                    │   │
│  │  If no  → Process and store result                  │   │
│  │                                                      │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### Example: Idempotent Payment
```python
class PaymentService:
    def __init__(self):
        self.processed_keys = {}  # In production: Redis/DB
    
    def process_payment(self, idempotency_key, payment):
        # Check if already processed
        if idempotency_key in self.processed_keys:
            return self.processed_keys[idempotency_key]
        
        # Process payment
        result = self._charge_card(payment)
        
        # Store result for future duplicate requests
        self.processed_keys[idempotency_key] = result
        
        return result
    
    # Client can safely retry on timeout:
    # POST /payments
    # Idempotency-Key: order-12345-attempt-1
    # 
    # If timeout, client retries with SAME key
    # Server returns same result without double-charging
```

### Retry Strategies
```
┌─────────────────────────────────────────────────────────────┐
│                 RETRY STRATEGIES                            │
│                                                             │
│  EXPONENTIAL BACKOFF                                        │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                                                      │   │
│  │  Attempt 1: Wait 1s                                 │   │
│  │  Attempt 2: Wait 2s                                 │   │
│  │  Attempt 3: Wait 4s                                 │   │
│  │  Attempt 4: Wait 8s                                 │   │
│  │  Attempt 5: Wait 16s (max)                          │   │
│  │                                                      │   │
│  │  wait_time = min(base * 2^attempt, max_wait)        │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  EXPONENTIAL BACKOFF + JITTER                              │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                                                      │   │
│  │  Without jitter: All clients retry at same time!    │   │
│  │  ████████████████████  (thundering herd)            │   │
│  │                                                      │   │
│  │  With jitter: Retries spread out                    │   │
│  │  █  ██   █  ████  ██   ███  █  (distributed)       │   │
│  │                                                      │   │
│  │  wait_time = random(0, min(cap, base * 2^attempt))  │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  RETRY BUDGET                                               │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                                                      │   │
│  │  Limit retries to % of total requests               │   │
│  │                                                      │   │
│  │  If retry_count / total_requests > 10%:             │   │
│  │      Stop retrying (system is overloaded)           │   │
│  │                                                      │   │
│  │  Prevents retry storms from overwhelming system     │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

```python
import random
import time

def retry_with_backoff(func, max_retries=5, base_delay=1, max_delay=60):
    for attempt in range(max_retries):
        try:
            return func()
        except RetryableError:
            if attempt == max_retries - 1:
                raise
            
            # Exponential backoff with jitter
            delay = min(base_delay * (2 ** attempt), max_delay)
            jitter = random.uniform(0, delay)
            
            time.sleep(jitter)
    
    raise MaxRetriesExceeded()
```

---

## 6. Versioning & Compatibility

### API Versioning Strategies
```
┌─────────────────────────────────────────────────────────────┐
│              API VERSIONING STRATEGIES                      │
│                                                             │
│  URL PATH VERSIONING                                        │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  GET /api/v1/users/123                              │   │
│  │  GET /api/v2/users/123                              │   │
│  │                                                      │   │
│  │  ✓ Clear, explicit                                 │   │
│  │  ✓ Easy to route                                   │   │
│  │  ✗ URL pollution                                   │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  HEADER VERSIONING                                          │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  GET /api/users/123                                 │   │
│  │  Accept: application/vnd.myapp.v2+json              │   │
│  │  # or                                               │   │
│  │  API-Version: 2                                     │   │
│  │                                                      │   │
│  │  ✓ Clean URLs                                      │   │
│  │  ✗ Hidden version, harder to test                  │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  QUERY PARAMETER                                            │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  GET /api/users/123?version=2                       │   │
│  │                                                      │   │
│  │  ✓ Optional, can default                           │   │
│  │  ✗ Can be forgotten                                │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Backward Compatibility Rules
```
┌─────────────────────────────────────────────────────────────┐
│          BACKWARD COMPATIBILITY RULES                       │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ✅ SAFE CHANGES (Backward Compatible)                     │
│  ├── Adding new optional fields                            │
│  ├── Adding new endpoints                                  │
│  ├── Adding new optional query parameters                  │
│  ├── Widening accepted value ranges                        │
│  └── Adding new enum values (if client ignores unknown)    │
│                                                             │
│  ❌ BREAKING CHANGES (Need New Version)                    │
│  ├── Removing fields                                       │
│  ├── Renaming fields                                       │
│  ├── Changing field types                                  │
│  ├── Making optional fields required                       │
│  ├── Changing endpoint URLs                                │
│  ├── Changing authentication                               │
│  └── Removing enum values                                  │
│                                                             │
│  ROBUSTNESS PRINCIPLE (Postel's Law):                      │
│  "Be conservative in what you send,                        │
│   be liberal in what you accept"                           │
│                                                             │
│  • Send well-formed, minimal responses                     │
│  • Accept and ignore unexpected fields                     │
│  • Handle missing optional fields gracefully               │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 7. Configuration Management

### Configuration Sources
```
┌─────────────────────────────────────────────────────────────┐
│            CONFIGURATION HIERARCHY                          │
│                                                             │
│  Priority (highest to lowest):                              │
│                                                             │
│  1. ┌─────────────────────────────────────────────────┐    │
│     │ COMMAND LINE ARGUMENTS                          │    │
│     │ --server.port=8080                              │    │
│     └─────────────────────────────────────────────────┘    │
│                           │                                 │
│  2. ┌─────────────────────┴───────────────────────────┐    │
│     │ ENVIRONMENT VARIABLES                           │    │
│     │ SERVER_PORT=8080                                │    │
│     └─────────────────────────────────────────────────┘    │
│                           │                                 │
│  3. ┌─────────────────────┴───────────────────────────┐    │
│     │ EXTERNAL CONFIG (Consul, etcd, Vault)           │    │
│     │ Dynamic, can change at runtime                  │    │
│     └─────────────────────────────────────────────────┘    │
│                           │                                 │
│  4. ┌─────────────────────┴───────────────────────────┐    │
│     │ CONFIG FILES (YAML, Properties)                 │    │
│     │ application-prod.yml                            │    │
│     └─────────────────────────────────────────────────┘    │
│                           │                                 │
│  5. ┌─────────────────────┴───────────────────────────┐    │
│     │ DEFAULT VALUES (in code)                        │    │
│     │ @Value("${server.port:8080}")                   │    │
│     └─────────────────────────────────────────────────┘    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Feature Flags
```
┌─────────────────────────────────────────────────────────────┐
│                  FEATURE FLAGS                              │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                                                      │   │
│  │  if feature_flags.is_enabled("new_checkout"):       │   │
│  │      return new_checkout_flow(cart)                 │   │
│  │  else:                                              │   │
│  │      return legacy_checkout_flow(cart)              │   │
│  │                                                      │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  USE CASES:                                                 │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                                                      │   │
│  │  RELEASE TOGGLE                                     │   │
│  │  • Deploy code disabled, enable when ready          │   │
│  │  • Quick rollback by toggling off                   │   │
│  │                                                      │   │
│  │  EXPERIMENT TOGGLE                                  │   │
│  │  • A/B testing                                      │   │
│  │  • Enable for % of users                            │   │
│  │                                                      │   │
│  │  OPS TOGGLE                                         │   │
│  │  • Disable expensive features under load            │   │
│  │  • Kill switch for problematic features             │   │
│  │                                                      │   │
│  │  PERMISSION TOGGLE                                  │   │
│  │  • Enable for premium users only                    │   │
│  │  • Beta features for specific users                 │   │
│  │                                                      │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  ⚠️ Clean up old flags! Technical debt accumulates fast   │
└─────────────────────────────────────────────────────────────┘
```

---

## Summary: Design Principles

```
┌────────────────────────────────────────────────────────────┐
│           DESIGN PRINCIPLES FOR PRODUCTION                 │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  NETWORK                                                   │
│  • Assume network will fail in every possible way         │
│  • Set timeouts on every network call                     │
│  • Cache DNS appropriately                                │
│  • Use multiple availability zones                        │
│                                                            │
│  API DESIGN                                                │
│  • Make operations idempotent where possible              │
│  • Use exponential backoff with jitter for retries        │
│  • Version APIs for backward compatibility                │
│  • Be liberal in what you accept                          │
│                                                            │
│  CONFIGURATION                                             │
│  • Externalize all configuration                          │
│  • Use feature flags for safe deployments                 │
│  • Support configuration changes without restart          │
│  • Never hardcode secrets                                 │
│                                                            │
│  SERVICE DISCOVERY                                         │
│  • Don't hardcode service locations                       │
│  • Implement health checks properly                       │
│  • Handle service unavailability gracefully               │
│                                                            │
└────────────────────────────────────────────────────────────┘
```
