# Release It! - Part 1: Stability Patterns

> Key patterns to build resilient, production-ready systems

---

## 1. Circuit Breaker Pattern

### Concept
Prevents cascade failures by stopping requests to a failing service. Like an electrical circuit breaker that trips to prevent damage.

### States
```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│    ┌────────┐      failures > threshold    ┌────────┐      │
│    │ CLOSED │ ──────────────────────────► │  OPEN  │      │
│    └────────┘                              └────────┘      │
│         ▲                                       │          │
│         │                                       │          │
│         │      success                    timeout          │
│         │                                       │          │
│         │                                       ▼          │
│         │                              ┌─────────────┐     │
│         └───────────────────────────── │ HALF-OPEN   │     │
│                                        └─────────────┘     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### States Explained
| State | Behavior |
|-------|----------|
| **Closed** | Normal operation, requests pass through |
| **Open** | All requests fail immediately (fail fast) |
| **Half-Open** | Allow limited requests to test recovery |

### Example Implementation (Java)
```java
public class CircuitBreaker {
    private State state = State.CLOSED;
    private int failureCount = 0;
    private int threshold = 5;
    private long timeout = 30000; // 30 seconds
    private long lastFailureTime;

    public Response call(Request request) {
        if (state == State.OPEN) {
            if (System.currentTimeMillis() - lastFailureTime > timeout) {
                state = State.HALF_OPEN;
            } else {
                throw new CircuitOpenException();
            }
        }
        
        try {
            Response response = makeRequest(request);
            onSuccess();
            return response;
        } catch (Exception e) {
            onFailure();
            throw e;
        }
    }
    
    private void onSuccess() {
        failureCount = 0;
        state = State.CLOSED;
    }
    
    private void onFailure() {
        failureCount++;
        lastFailureTime = System.currentTimeMillis();
        if (failureCount >= threshold) {
            state = State.OPEN;
        }
    }
}
```

### Key Configuration
- **Failure threshold**: Number of failures before opening
- **Timeout**: Time before attempting recovery
- **Success threshold**: Successes needed to close again

---

## 2. Timeout Pattern

### Concept
Never wait forever. Every external call should have a timeout to prevent threads from blocking indefinitely.

### Diagram
```
┌────────────────────────────────────────────────────────────┐
│                     WITHOUT TIMEOUT                        │
│                                                            │
│  Service A ──────────────────────────────► Service B       │
│     │                                         │            │
│     │              waiting...                 │ (hung)     │
│     │              waiting...                 │            │
│     │              waiting forever...         │            │
│     ▼                                         ▼            │
│  Thread blocked indefinitely = Resource exhaustion         │
└────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────┐
│                      WITH TIMEOUT                          │
│                                                            │
│  Service A ──────────────────────────────► Service B       │
│     │                                         │            │
│     │              waiting... (2s)            │ (hung)     │
│     │              TIMEOUT!                   │            │
│     ▼                                                      │
│  Thread released, error handled gracefully                 │
└────────────────────────────────────────────────────────────┘
```

### Types of Timeouts
```
┌─────────────────────────────────────────────────────────┐
│                    TIMEOUT TYPES                        │
├─────────────────────────────────────────────────────────┤
│  Connection Timeout  │ Time to establish connection    │
│  Read Timeout        │ Time to wait for response data  │
│  Write Timeout       │ Time to send request data       │
│  Total Timeout       │ Overall operation time limit    │
└─────────────────────────────────────────────────────────┘
```

### Example (Python)
```python
import requests
from requests.exceptions import Timeout

def call_service(url, data):
    try:
        response = requests.post(
            url,
            json=data,
            timeout=(3.0, 10.0)  # (connect_timeout, read_timeout)
        )
        return response.json()
    except Timeout:
        # Fallback or fail gracefully
        return get_cached_response()
```

### Best Practices
- Set timeouts shorter than user patience (~3-5s for web)
- Use different timeouts for different operations
- Always have a timeout, even if very long

---

## 3. Bulkhead Pattern

### Concept
Isolate components so failure in one doesn't sink the entire ship (like compartments in a ship's hull).

### Diagram
```
┌─────────────────────────────────────────────────────────────┐
│                   WITHOUT BULKHEADS                         │
│                                                             │
│   ┌─────────────────────────────────────────────────────┐   │
│   │              Shared Thread Pool (100)               │   │
│   │  ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐   │   │
│   │  │ Req │ │ Req │ │ Req │ │ Req │ │ Req │ │ ... │   │   │
│   │  └─────┘ └─────┘ └─────┘ └─────┘ └─────┘ └─────┘   │   │
│   └─────────────────────────────────────────────────────┘   │
│                                                             │
│   If Service C is slow → All 100 threads blocked →          │
│   ENTIRE APPLICATION DIES                                   │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                    WITH BULKHEADS                           │
│                                                             │
│   ┌─────────────┐  ┌─────────────┐  ┌─────────────┐        │
│   │ Pool A (30) │  │ Pool B (40) │  │ Pool C (30) │        │
│   │  Service A  │  │  Service B  │  │  Service C  │        │
│   │  ┌──┐ ┌──┐  │  │  ┌──┐ ┌──┐  │  │  ┌──┐ ┌──┐  │        │
│   │  └──┘ └──┘  │  │  └──┘ └──┘  │  │  └──┘ └──┘  │        │
│   └─────────────┘  └─────────────┘  └─────────────┘        │
│                                                             │
│   If Service C is slow → Only Pool C affected →             │
│   Services A & B continue working!                          │
└─────────────────────────────────────────────────────────────┘
```

### Implementation Strategies

| Strategy | Description | Use Case |
|----------|-------------|----------|
| **Thread Pool** | Separate pools per service | Blocking I/O operations |
| **Semaphore** | Limit concurrent calls | Lightweight isolation |
| **Process** | Separate processes | Complete isolation |
| **Container** | Separate containers | Microservices |

### Example (Java with Resilience4j)
```java
// Thread pool bulkhead
ThreadPoolBulkheadConfig config = ThreadPoolBulkheadConfig.custom()
    .maxThreadPoolSize(10)
    .coreThreadPoolSize(5)
    .queueCapacity(20)
    .build();

ThreadPoolBulkhead bulkhead = ThreadPoolBulkhead.of("paymentService", config);

Supplier<Payment> supplier = ThreadPoolBulkhead
    .decorateSupplier(bulkhead, () -> paymentService.process(order));

CompletableFuture<Payment> future = supplier.get().toCompletableFuture();
```

---

## 4. Steady State Pattern

### Concept
System should run indefinitely without manual intervention. Resources should be self-cleaning.

### Problem
```
┌─────────────────────────────────────────────────────────────┐
│                    RESOURCE LEAKS                           │
│                                                             │
│   Time ──────────────────────────────────────►              │
│                                                             │
│   Disk Space        ████████████████████████████ FULL!      │
│   Memory            ████████████████████████████ OOM!       │
│   Log Files         ████████████████████████████ 100GB!     │
│   DB Connections    ████████████████████████████ Exhausted! │
│   Temp Files        ████████████████████████████ Full!      │
└─────────────────────────────────────────────────────────────┘
```

### Solution: Auto-cleanup Mechanisms
```
┌─────────────────────────────────────────────────────────────┐
│                   STEADY STATE SYSTEM                       │
│                                                             │
│   ┌─────────────┐    ┌─────────────┐    ┌─────────────┐    │
│   │ Log Rotate  │    │ Cache TTL   │    │ Temp Clean  │    │
│   │  Daily      │    │  1 hour     │    │  Hourly     │    │
│   └──────┬──────┘    └──────┬──────┘    └──────┬──────┘    │
│          │                  │                  │            │
│          ▼                  ▼                  ▼            │
│   ┌─────────────────────────────────────────────────────┐  │
│   │            Resources stay bounded                    │  │
│   │                                                      │  │
│   │  Disk: ████░░░░░░  Memory: ███░░░░░░  Conn: ██░░░░  │  │
│   │        (stable)           (stable)        (stable)   │  │
│   └─────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

### Checklist
- [ ] Log rotation configured
- [ ] Cache eviction policies set
- [ ] Database connection pool limits
- [ ] Session cleanup scheduled
- [ ] Temp file purging automated
- [ ] Old data archived/deleted

---

## 5. Fail Fast Pattern

### Concept
If a request is going to fail, fail immediately. Don't waste resources on doomed transactions.

### Diagram
```
┌────────────────────────────────────────────────────────────┐
│                     FAIL SLOW (BAD)                        │
│                                                            │
│  Request ──► Auth ──► Validate ──► DB Query ──► FAIL!     │
│                                        │                   │
│              Wasted: 5 seconds + DB connection             │
└────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────┐
│                     FAIL FAST (GOOD)                       │
│                                                            │
│  Request ──► Check Circuit ──► FAIL! (Circuit Open)       │
│                    │                                       │
│              Wasted: 0.001 seconds, no resources           │
└────────────────────────────────────────────────────────────┘
```

### When to Fail Fast
```
┌─────────────────────────────────────────────────────────────┐
│              FAIL FAST CHECKPOINTS                          │
├─────────────────────────────────────────────────────────────┤
│  1. Circuit breaker is OPEN         → Fail immediately     │
│  2. Required service unavailable    → Fail immediately     │
│  3. Invalid input parameters        → Fail immediately     │
│  4. Missing authentication          → Fail immediately     │
│  5. Rate limit exceeded             → Fail immediately     │
│  6. Feature flag disabled           → Fail immediately     │
└─────────────────────────────────────────────────────────────┘
```

### Example
```python
def process_order(order):
    # Fail fast checks at the start
    if not inventory_service.is_available():
        raise ServiceUnavailableError("Inventory service down")
    
    if not payment_circuit.is_closed():
        raise ServiceUnavailableError("Payment service unavailable")
    
    if order.total > user.credit_limit:
        raise ValidationError("Exceeds credit limit")
    
    # Only proceed if all checks pass
    return execute_order(order)
```

---

## 6. Handshaking Pattern

### Concept
Server tells client whether it can accept work before client sends full request. Prevents wasted effort.

### Diagram
```
┌─────────────────────────────────────────────────────────────┐
│                    HANDSHAKE FLOW                           │
│                                                             │
│   Client                                    Server          │
│     │                                          │            │
│     │  ──── "Can you handle request?" ────►   │            │
│     │                                          │            │
│     │  ◄──── "Yes, I'm ready" ─────────────   │            │
│     │         or                               │            │
│     │  ◄──── "No, try later" (503) ────────   │            │
│     │                                          │            │
│     │  ──── Full request (if accepted) ────►  │            │
│     │                                          │            │
└─────────────────────────────────────────────────────────────┘
```

### HTTP Example: Expect Header
```http
POST /upload HTTP/1.1
Host: api.example.com
Content-Length: 1073741824
Expect: 100-continue

# Server responds:
# HTTP/1.1 100 Continue  (proceed with upload)
# or
# HTTP/1.1 503 Service Unavailable  (don't send data)
```

### Health Check Handshake
```
┌────────────────────────────────────────────────────────────┐
│                 LOAD BALANCER HANDSHAKE                    │
│                                                            │
│  Load Balancer                                             │
│       │                                                    │
│       │  GET /health ──────────► Server 1: 200 OK ✓       │
│       │  GET /health ──────────► Server 2: 503 Busy ✗     │
│       │  GET /health ──────────► Server 3: 200 OK ✓       │
│       │                                                    │
│       │  Route traffic to Server 1 and 3 only              │
└────────────────────────────────────────────────────────────┘
```

---

## 7. Test Harness Pattern

### Concept
Create test implementations that simulate failures to verify resilience patterns work correctly.

### Failure Modes to Test
```
┌─────────────────────────────────────────────────────────────┐
│               TEST HARNESS FAILURE MODES                    │
├─────────────────────────────────────────────────────────────┤
│  Network Failures                                           │
│  ├── Connection refused                                     │
│  ├── Connection timeout                                     │
│  ├── Read timeout (accept but don't respond)               │
│  └── Partial response (disconnect mid-stream)              │
│                                                             │
│  Application Failures                                       │
│  ├── HTTP 500 errors                                       │
│  ├── HTTP 503 Service Unavailable                          │
│  ├── Malformed responses                                   │
│  └── Slow responses (variable latency)                     │
│                                                             │
│  Resource Failures                                          │
│  ├── Memory exhaustion                                     │
│  ├── CPU saturation                                        │
│  └── Disk full                                             │
└─────────────────────────────────────────────────────────────┘
```

### Example: Chaos Test Harness
```python
class ChaosServiceMock:
    def __init__(self, failure_rate=0.1, latency_ms=0):
        self.failure_rate = failure_rate
        self.latency_ms = latency_ms
    
    def call(self, request):
        # Simulate latency
        time.sleep(self.latency_ms / 1000)
        
        # Simulate random failures
        if random.random() < self.failure_rate:
            failure_type = random.choice([
                'timeout',
                'connection_refused',
                'http_500',
                'malformed_response'
            ])
            raise SimulatedFailure(failure_type)
        
        return MockResponse(status=200)
```

---

## Summary: Pattern Selection Guide

```
┌────────────────────────────────────────────────────────────┐
│              WHEN TO USE EACH PATTERN                      │
├────────────────────────────────────────────────────────────┤
│ Problem                        │ Pattern                   │
├────────────────────────────────┼───────────────────────────┤
│ Cascading failures             │ Circuit Breaker           │
│ Blocked threads                │ Timeout                   │
│ One service kills all          │ Bulkhead                  │
│ Resource exhaustion over time  │ Steady State              │
│ Wasting resources on failures  │ Fail Fast                 │
│ Server overwhelmed             │ Handshaking               │
│ Verifying resilience           │ Test Harness              │
└────────────────────────────────┴───────────────────────────┘
```

---

## Key Takeaways

1. **Expect failures** - Design for when things go wrong, not just when they work
2. **Contain failures** - Don't let one component bring down the whole system
3. **Recover quickly** - Minimize blast radius and recovery time
4. **Fail gracefully** - Degrade functionality rather than crash completely
5. **Test for failure** - Use chaos engineering to verify resilience
