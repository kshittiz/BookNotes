# Release It! - Part 5: Operations & Deployment

> Running, monitoring, and deploying production systems safely

---

## 1. Deployment Strategies

### Deployment Patterns Comparison
```
┌─────────────────────────────────────────────────────────────┐
│              DEPLOYMENT STRATEGIES                          │
│                                                             │
│  BIG BANG (All at Once)                                     │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  Before:  [v1] [v1] [v1] [v1]                       │   │
│  │                    │                                 │   │
│  │                    ▼ Deploy                         │   │
│  │  After:   [v2] [v2] [v2] [v2]                       │   │
│  │                                                      │   │
│  │  ✓ Simple          ✗ High risk                     │   │
│  │  ✓ Fast            ✗ Full outage on failure        │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  ROLLING DEPLOYMENT                                         │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  Step 1: [v2] [v1] [v1] [v1]                        │   │
│  │  Step 2: [v2] [v2] [v1] [v1]                        │   │
│  │  Step 3: [v2] [v2] [v2] [v1]                        │   │
│  │  Step 4: [v2] [v2] [v2] [v2]                        │   │
│  │                                                      │   │
│  │  ✓ Zero downtime   ✗ Mixed versions during deploy  │   │
│  │  ✓ Gradual         ✗ Slower rollback               │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  BLUE-GREEN DEPLOYMENT                                      │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                                                      │   │
│  │  ┌───────────────┐     ┌───────────────┐           │   │
│  │  │  BLUE (v1)    │     │  GREEN (v2)   │           │   │
│  │  │  [v1] [v1]    │     │  [v2] [v2]    │           │   │
│  │  └───────┬───────┘     └───────┬───────┘           │   │
│  │          │                     │                    │   │
│  │          └────────┬────────────┘                    │   │
│  │                   │                                 │   │
│  │            ┌──────┴──────┐                          │   │
│  │            │ Load Balancer│ ← Switch traffic       │   │
│  │            └─────────────┘                          │   │
│  │                                                      │   │
│  │  ✓ Instant rollback    ✗ Double infrastructure    │   │
│  │  ✓ Full testing        ✗ Database migrations hard │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  CANARY DEPLOYMENT                                          │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                                                      │   │
│  │  Traffic: ═══════════════════════════════►          │   │
│  │           │                                          │   │
│  │           ├──95%──► [v1] [v1] [v1] [v1]             │   │
│  │           │                                          │   │
│  │           └──5%───► [v2]  ← Canary                  │   │
│  │                                                      │   │
│  │  Monitor canary → If healthy → Increase traffic     │   │
│  │                 → If errors  → Rollback canary      │   │
│  │                                                      │   │
│  │  ✓ Low risk        ✗ Complex routing               │   │
│  │  ✓ Real production ✗ Needs good monitoring         │   │
│  │    testing                                          │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### Canary Deployment Flow
```
┌─────────────────────────────────────────────────────────────┐
│               CANARY DEPLOYMENT FLOW                        │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ 1. Deploy canary (1 instance)                       │   │
│  │    [v1] [v1] [v1] [v1] [v2]                        │   │
│  │                         └── 1% traffic             │   │
│  └─────────────────────────────────────────────────────┘   │
│                    │                                        │
│                    ▼                                        │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ 2. Monitor metrics (5-10 minutes)                   │   │
│  │    • Error rate                                     │   │
│  │    • Latency (p50, p95, p99)                       │   │
│  │    • CPU/Memory                                     │   │
│  │    • Business metrics                               │   │
│  └─────────────────────────────────────────────────────┘   │
│                    │                                        │
│         ┌─────────┴─────────┐                              │
│         ▼                   ▼                              │
│  ┌─────────────┐    ┌─────────────┐                       │
│  │  Healthy?   │    │  Problems?  │                       │
│  │  Continue   │    │  Rollback   │                       │
│  └──────┬──────┘    └─────────────┘                       │
│         │                                                   │
│         ▼                                                   │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ 3. Gradually increase traffic                       │   │
│  │    1% → 5% → 10% → 25% → 50% → 100%                │   │
│  │    (with monitoring at each step)                   │   │
│  └─────────────────────────────────────────────────────┘   │
│                    │                                        │
│                    ▼                                        │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ 4. Complete rollout                                 │   │
│  │    [v2] [v2] [v2] [v2] [v2]                        │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### Database Migration Strategies
```
┌─────────────────────────────────────────────────────────────┐
│            DATABASE MIGRATION STRATEGIES                    │
│                                                             │
│  EXPAND-CONTRACT PATTERN (Zero Downtime)                   │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                                                      │   │
│  │  Phase 1: EXPAND                                    │   │
│  │  ┌─────────────────────────────────────────────┐   │   │
│  │  │  Add new column (nullable)                   │   │   │
│  │  │  ALTER TABLE users ADD email_new VARCHAR(255)│   │   │
│  │  │                                              │   │   │
│  │  │  Table: [id] [name] [email] [email_new]     │   │   │
│  │  └─────────────────────────────────────────────┘   │   │
│  │                                                      │   │
│  │  Phase 2: MIGRATE                                   │   │
│  │  ┌─────────────────────────────────────────────┐   │   │
│  │  │  App writes to both columns                  │   │   │
│  │  │  Backfill existing data                      │   │   │
│  │  │  UPDATE users SET email_new = LOWER(email)   │   │   │
│  │  └─────────────────────────────────────────────┘   │   │
│  │                                                      │   │
│  │  Phase 3: CONTRACT                                  │   │
│  │  ┌─────────────────────────────────────────────┐   │   │
│  │  │  App reads only from new column              │   │   │
│  │  │  Drop old column                             │   │   │
│  │  │  ALTER TABLE users DROP email                │   │   │
│  │  │  ALTER TABLE users RENAME email_new TO email │   │   │
│  │  └─────────────────────────────────────────────┘   │   │
│  │                                                      │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  KEY PRINCIPLES:                                           │
│  • Never make breaking changes in single deploy           │
│  • App must work with old AND new schema during transition │
│  • Use feature flags to control which code path is active │
│  • Migrations must be reversible                          │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. Monitoring & Observability

### The Three Pillars
```
┌─────────────────────────────────────────────────────────────┐
│           THREE PILLARS OF OBSERVABILITY                    │
│                                                             │
│  ┌─────────────────┐ ┌─────────────────┐ ┌───────────────┐ │
│  │     METRICS     │ │      LOGS       │ │    TRACES     │ │
│  │                 │ │                 │ │               │ │
│  │  What is        │ │  What           │ │  How requests │ │
│  │  happening?     │ │  happened?      │ │  flow?        │ │
│  │                 │ │                 │ │               │ │
│  │  • CPU: 75%     │ │  Error at       │ │  A → B → C    │ │
│  │  • RPS: 1000    │ │  10:30:05:      │ │  ↓   ↓   ↓    │ │
│  │  • Latency: 50ms│ │  "Connection    │ │  5ms 10ms 3ms │ │
│  │  • Errors: 0.1% │ │   refused"      │ │               │ │
│  │                 │ │                 │ │               │ │
│  │  Quantitative   │ │  Qualitative    │ │  Causal       │ │
│  │  Aggregated     │ │  Detailed       │ │  Distributed  │ │
│  └─────────────────┘ └─────────────────┘ └───────────────┘ │
│         │                   │                   │          │
│         └───────────────────┼───────────────────┘          │
│                             │                               │
│                             ▼                               │
│                   COMPLETE PICTURE                          │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Key Metrics (RED & USE)
```
┌─────────────────────────────────────────────────────────────┐
│              KEY METRICS FRAMEWORKS                         │
│                                                             │
│  RED METHOD (For Services)                                  │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                                                      │   │
│  │  R - Rate:      Requests per second                 │   │
│  │                 "How busy is the service?"          │   │
│  │                                                      │   │
│  │  E - Errors:    Failed requests per second          │   │
│  │                 "How often does it fail?"           │   │
│  │                                                      │   │
│  │  D - Duration:  Time per request (latency)          │   │
│  │                 "How long do requests take?"        │   │
│  │                                                      │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  USE METHOD (For Resources)                                 │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                                                      │   │
│  │  U - Utilization: % of resource being used          │   │
│  │                   "CPU at 80%"                      │   │
│  │                                                      │   │
│  │  S - Saturation:  Work queued/waiting               │   │
│  │                   "10 requests waiting"             │   │
│  │                                                      │   │
│  │  E - Errors:      Error count for resource          │   │
│  │                   "5 disk errors"                   │   │
│  │                                                      │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  GOLDEN SIGNALS (Google SRE)                               │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  • Latency      - Time to serve requests            │   │
│  │  • Traffic      - Demand on system                  │   │
│  │  • Errors       - Rate of failed requests           │   │
│  │  • Saturation   - How "full" the system is         │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### Latency Percentiles
```
┌─────────────────────────────────────────────────────────────┐
│               LATENCY PERCENTILES                           │
│                                                             │
│  Why percentiles matter:                                    │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                                                      │   │
│  │  Average latency: 100ms  ← MISLEADING!              │   │
│  │                                                      │   │
│  │  Reality:                                           │   │
│  │  ────────────────────────────────────────────────   │   │
│  │  │██████████████████████████████              │   │   │
│  │  │ Most requests: 50ms                        │   │   │
│  │  │                          │████████████████ │   │   │
│  │  │                          │ Some: 500ms+    │   │   │
│  │  ────────────────────────────────────────────────   │   │
│  │  50ms  100ms  200ms  300ms  400ms  500ms+           │   │
│  │                                                      │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  PERCENTILE MEANINGS:                                       │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                                                      │   │
│  │  P50 (Median):  50% of requests faster than this   │   │
│  │                 "Typical user experience"           │   │
│  │                                                      │   │
│  │  P95:           95% of requests faster than this   │   │
│  │                 "Most users' worst experience"      │   │
│  │                                                      │   │
│  │  P99:           99% of requests faster than this   │   │
│  │                 "Edge cases, often your biggest    │   │
│  │                  customers (most requests)"         │   │
│  │                                                      │   │
│  │  P99.9:         Extreme outliers                   │   │
│  │                 "1 in 1000 requests"               │   │
│  │                                                      │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  SLO EXAMPLE:                                              │
│  • P50 latency < 100ms                                     │
│  • P95 latency < 300ms                                     │
│  • P99 latency < 1000ms                                    │
└─────────────────────────────────────────────────────────────┘
```

### Distributed Tracing
```
┌─────────────────────────────────────────────────────────────┐
│              DISTRIBUTED TRACING                            │
│                                                             │
│  Request Flow (Single Trace):                               │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  Trace ID: abc-123                                   │   │
│  │                                                      │   │
│  │  ┌──────────────────────────────────────────────┐   │   │
│  │  │ API Gateway (Span 1)                          │   │   │
│  │  │ ████████████████████████████████  200ms      │   │   │
│  │  └──────────────────────────────────────────────┘   │   │
│  │       │                                              │   │
│  │       ├──► ┌────────────────────────────────┐       │   │
│  │       │    │ User Service (Span 2)          │       │   │
│  │       │    │ █████████████████  80ms        │       │   │
│  │       │    └────────────────────────────────┘       │   │
│  │       │         │                                    │   │
│  │       │         └──► ┌──────────────────────┐       │   │
│  │       │              │ User DB (Span 3)     │       │   │
│  │       │              │ ██████████  40ms     │       │   │
│  │       │              └──────────────────────┘       │   │
│  │       │                                              │   │
│  │       └──► ┌────────────────────────────────┐       │   │
│  │            │ Order Service (Span 4)         │       │   │
│  │            │ ███████████████████████  100ms │       │   │
│  │            └────────────────────────────────┘       │   │
│  │                 │                                    │   │
│  │                 └──► ┌──────────────────────┐       │   │
│  │                      │ Order DB (Span 5)    │       │   │
│  │                      │ ████████████  50ms   │       │   │
│  │                      └──────────────────────┘       │   │
│  │                                                      │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  Context Propagation:                                       │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  HTTP Header: traceparent: 00-abc123-def456-01      │   │
│  │  Contains: version-trace_id-parent_span_id-flags    │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

---

## 3. Alerting

### Alert Design Principles
```
┌─────────────────────────────────────────────────────────────┐
│               ALERTING PRINCIPLES                           │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                                                      │   │
│  │  ✅ GOOD ALERTS                                     │   │
│  │  • Actionable - Someone needs to do something       │   │
│  │  • Urgent - Needs attention now                     │   │
│  │  • Clear - What's wrong is obvious                  │   │
│  │  • Unique - Each alert represents distinct issue    │   │
│  │                                                      │   │
│  │  ❌ BAD ALERTS                                      │   │
│  │  • "CPU high" - So what? Is it affecting users?    │   │
│  │  • "Disk 80%" - Not urgent, use ticket instead     │   │
│  │  • Flapping - Up/down/up/down repeatedly           │   │
│  │  • Duplicates - Same issue, multiple alerts        │   │
│  │                                                      │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  ALERT ON SYMPTOMS, NOT CAUSES                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                                                      │   │
│  │  ❌ Alert: "Database connection pool at 90%"        │   │
│  │                                                      │   │
│  │  ✅ Alert: "API error rate > 1% for 5 minutes"     │   │
│  │     (This is what users experience)                 │   │
│  │                                                      │   │
│  │  Then investigate: Why is error rate high?          │   │
│  │  → Database connection pool exhausted               │   │
│  │                                                      │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### Alert Severity Levels
```
┌─────────────────────────────────────────────────────────────┐
│                ALERT SEVERITY LEVELS                        │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  CRITICAL (Page immediately)                        │   │
│  │  ─────────────────────────────                      │   │
│  │  • Service completely down                          │   │
│  │  • Data loss occurring                              │   │
│  │  • Security breach detected                         │   │
│  │  • Revenue-impacting outage                         │   │
│  │                                                      │   │
│  │  Response: Wake someone up at 3 AM                  │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  HIGH (Page during business hours)                  │   │
│  │  ────────────────────────────────                   │   │
│  │  • Significant degradation                          │   │
│  │  • Error rate elevated but service up               │   │
│  │  • Capacity approaching limits                      │   │
│  │                                                      │   │
│  │  Response: Address within 1 hour                    │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  MEDIUM (Ticket)                                    │   │
│  │  ───────────────                                    │   │
│  │  • Minor degradation                                │   │
│  │  • Approaching thresholds                           │   │
│  │  • Non-critical service affected                    │   │
│  │                                                      │   │
│  │  Response: Address within 1 day                     │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  LOW (Dashboard/Report)                             │   │
│  │  ──────────────────────                             │   │
│  │  • Informational                                    │   │
│  │  • Trends to watch                                  │   │
│  │  • Optimization opportunities                       │   │
│  │                                                      │   │
│  │  Response: Review in next planning cycle            │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### SLOs and Error Budgets
```
┌─────────────────────────────────────────────────────────────┐
│             SLOs AND ERROR BUDGETS                          │
│                                                             │
│  SLI (Service Level Indicator):                             │
│  "What we measure"                                          │
│  Example: Successful requests / Total requests              │
│                                                             │
│  SLO (Service Level Objective):                             │
│  "What we aim for"                                          │
│  Example: 99.9% availability over 30 days                   │
│                                                             │
│  ERROR BUDGET:                                              │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                                                      │   │
│  │  SLO: 99.9% availability                            │   │
│  │  Error Budget: 0.1% = 43 minutes/month downtime     │   │
│  │                                                      │   │
│  │  ════════════════════════════════════════════════   │   │
│  │  █████████████████████████████████████░░░░░░░░░░   │   │
│  │  └──────── SLO (99.9%) ──────────┘└─ Budget ─┘     │   │
│  │                                                      │   │
│  │  Month progress: Day 15                              │   │
│  │  Budget used:    15 minutes (35%)                   │   │
│  │  Budget remaining: 28 minutes                       │   │
│  │                                                      │   │
│  │  If budget exhausted → Freeze deployments           │   │
│  │                     → Focus on reliability          │   │
│  │                                                      │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  COMMON SLOs:                                               │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  Availability: 99.9% (Three nines)                  │   │
│  │  → 8.76 hours downtime/year                        │   │
│  │  → 43.8 minutes/month                              │   │
│  │                                                      │   │
│  │  Availability: 99.99% (Four nines)                  │   │
│  │  → 52.6 minutes downtime/year                      │   │
│  │  → 4.38 minutes/month                              │   │
│  │                                                      │   │
│  │  Latency: P99 < 500ms                              │   │
│  │  Error Rate: < 0.1%                                 │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

---

## 4. Incident Response

### Incident Workflow
```
┌─────────────────────────────────────────────────────────────┐
│               INCIDENT RESPONSE FLOW                        │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ 1. DETECT                                           │   │
│  │    • Alert fires                                    │   │
│  │    • Customer report                                │   │
│  │    • Monitoring shows anomaly                       │   │
│  └─────────────────────────────────────────────────────┘   │
│                    │                                        │
│                    ▼                                        │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ 2. TRIAGE                                           │   │
│  │    • Assess severity                                │   │
│  │    • Determine impact                               │   │
│  │    • Assign incident commander                      │   │
│  └─────────────────────────────────────────────────────┘   │
│                    │                                        │
│                    ▼                                        │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ 3. MITIGATE                                         │   │
│  │    • STOP THE BLEEDING FIRST                        │   │
│  │    • Rollback if recent deploy                      │   │
│  │    • Scale up if capacity issue                     │   │
│  │    • Redirect traffic if one region bad            │   │
│  │    • Don't debug in production during outage!      │   │
│  └─────────────────────────────────────────────────────┘   │
│                    │                                        │
│                    ▼                                        │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ 4. RESOLVE                                          │   │
│  │    • Fix root cause (after stable)                  │   │
│  │    • Verify fix in staging                          │   │
│  │    • Deploy carefully                               │   │
│  └─────────────────────────────────────────────────────┘   │
│                    │                                        │
│                    ▼                                        │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ 5. POST-MORTEM                                      │   │
│  │    • Timeline of events                             │   │
│  │    • Root cause analysis                            │   │
│  │    • Action items to prevent recurrence             │   │
│  │    • Blameless!                                     │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### Incident Communication Template
```
┌─────────────────────────────────────────────────────────────┐
│            INCIDENT COMMUNICATION                           │
│                                                             │
│  INITIAL NOTIFICATION:                                      │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  [INCIDENT] Payment processing degraded             │   │
│  │                                                      │   │
│  │  Status: Investigating                              │   │
│  │  Impact: ~5% of payments failing                    │   │
│  │  Started: 14:30 UTC                                 │   │
│  │  Incident Commander: @alice                         │   │
│  │  War Room: #incident-2024-01-15                     │   │
│  │                                                      │   │
│  │  Next update in 15 minutes                          │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  UPDATE:                                                    │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  [UPDATE] Payment processing                        │   │
│  │                                                      │   │
│  │  Status: Mitigating                                 │   │
│  │  Root Cause: Database connection pool exhausted     │   │
│  │  Action: Rolling back deploy from 14:15             │   │
│  │  ETA to resolution: 15 minutes                      │   │
│  │                                                      │   │
│  │  Next update in 10 minutes                          │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  RESOLUTION:                                                │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  [RESOLVED] Payment processing restored             │   │
│  │                                                      │   │
│  │  Duration: 45 minutes                               │   │
│  │  Impact: ~2,000 failed transactions                 │   │
│  │  Root Cause: Connection leak in new code            │   │
│  │  Resolution: Rolled back to previous version        │   │
│  │                                                      │   │
│  │  Post-mortem scheduled for tomorrow 10:00 UTC       │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

---

## 5. Runbooks & Automation

### Runbook Structure
```
┌─────────────────────────────────────────────────────────────┐
│                  RUNBOOK TEMPLATE                           │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  RUNBOOK: High API Error Rate                       │   │
│  │  ════════════════════════════                       │   │
│  │                                                      │   │
│  │  TRIGGER:                                           │   │
│  │  Alert "api-error-rate-high" fires when             │   │
│  │  error rate > 1% for 5 minutes                      │   │
│  │                                                      │   │
│  │  IMPACT:                                            │   │
│  │  Users experiencing failed requests                 │   │
│  │                                                      │   │
│  │  DIAGNOSIS STEPS:                                   │   │
│  │  1. Check recent deployments                        │   │
│  │     $ kubectl rollout history deployment/api        │   │
│  │                                                      │   │
│  │  2. Check error logs                                │   │
│  │     $ kubectl logs -l app=api --since=10m | grep ERR│   │
│  │                                                      │   │
│  │  3. Check downstream services                       │   │
│  │     Dashboard: grafana.com/d/dependencies           │   │
│  │                                                      │   │
│  │  REMEDIATION:                                       │   │
│  │  If recent deploy:                                  │   │
│  │     $ kubectl rollout undo deployment/api           │   │
│  │                                                      │   │
│  │  If database issue:                                 │   │
│  │     See runbook: database-connection-issues         │   │
│  │                                                      │   │
│  │  If dependency down:                                │   │
│  │     Enable circuit breaker:                         │   │
│  │     $ curl -X POST admin/circuit/payment/open       │   │
│  │                                                      │   │
│  │  ESCALATION:                                        │   │
│  │  If not resolved in 15 min → Page @senior-oncall    │   │
│  │                                                      │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### Common Operations Automation
```
┌─────────────────────────────────────────────────────────────┐
│            OPERATIONS AUTOMATION                            │
│                                                             │
│  AUTOMATED RESPONSES:                                       │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                                                      │   │
│  │  Auto-scaling                                       │   │
│  │  ┌──────────────────────────────────────────────┐  │   │
│  │  │  IF cpu > 70% for 5 min                      │  │   │
│  │  │  THEN add 2 instances                        │  │   │
│  │  │  MAX 20 instances                            │  │   │
│  │  └──────────────────────────────────────────────┘  │   │
│  │                                                      │   │
│  │  Auto-restart                                       │   │
│  │  ┌──────────────────────────────────────────────┐  │   │
│  │  │  IF health check fails 3 times               │  │   │
│  │  │  THEN restart container                      │  │   │
│  │  │  IF restart fails 3 times                    │  │   │
│  │  │  THEN page oncall                            │  │   │
│  │  └──────────────────────────────────────────────┘  │   │
│  │                                                      │   │
│  │  Auto-rollback                                      │   │
│  │  ┌──────────────────────────────────────────────┐  │   │
│  │  │  IF error rate > 5% within 10 min of deploy  │  │   │
│  │  │  THEN rollback automatically                 │  │   │
│  │  │  AND notify team                             │  │   │
│  │  └──────────────────────────────────────────────┘  │   │
│  │                                                      │   │
│  │  Circuit breaker activation                         │   │
│  │  ┌──────────────────────────────────────────────┐  │   │
│  │  │  IF payment-service error > 50%              │  │   │
│  │  │  THEN open circuit breaker                   │  │   │
│  │  │  AND return cached/fallback response         │  │   │
│  │  └──────────────────────────────────────────────┘  │   │
│  │                                                      │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

---

## 6. Chaos Engineering

### Chaos Engineering Principles
```
┌─────────────────────────────────────────────────────────────┐
│            CHAOS ENGINEERING                                │
│                                                             │
│  "The discipline of experimenting on a system to build     │
│   confidence in the system's capability to withstand       │
│   turbulent conditions in production."                     │
│                                                             │
│  PRINCIPLES:                                                │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                                                      │   │
│  │  1. Build hypothesis around steady state            │   │
│  │     "System should maintain <1% error rate"         │   │
│  │                                                      │   │
│  │  2. Vary real-world events                          │   │
│  │     Server crash, network partition, disk full      │   │
│  │                                                      │   │
│  │  3. Run experiments in production                   │   │
│  │     Staging doesn't reveal real issues              │   │
│  │                                                      │   │
│  │  4. Automate to run continuously                    │   │
│  │     Not a one-time thing                            │   │
│  │                                                      │   │
│  │  5. Minimize blast radius                           │   │
│  │     Start small, contain failures                   │   │
│  │                                                      │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### Chaos Experiments
```
┌─────────────────────────────────────────────────────────────┐
│              CHAOS EXPERIMENTS                              │
│                                                             │
│  INFRASTRUCTURE CHAOS:                                      │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  • Kill random instance                             │   │
│  │  • Fill disk to 100%                               │   │
│  │  • Consume all CPU                                  │   │
│  │  • Exhaust memory                                   │   │
│  │  • Kill entire availability zone                    │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  NETWORK CHAOS:                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  • Add latency (100ms, 500ms, 2000ms)              │   │
│  │  • Drop packets (1%, 5%, 20%)                      │   │
│  │  • Network partition between services               │   │
│  │  • DNS failure                                      │   │
│  │  • Corrupt packets                                  │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  APPLICATION CHAOS:                                         │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  • Return errors from dependencies                  │   │
│  │  • Slow down responses                              │   │
│  │  • Return malformed data                            │   │
│  │  • Exhaust connection pools                         │   │
│  │  • Trigger garbage collection                       │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  EXAMPLE: Netflix Chaos Monkey                              │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                                                      │   │
│  │  During business hours:                             │   │
│  │  - Randomly terminate EC2 instances                 │   │
│  │  - Verify service continues working                 │   │
│  │  - Teams learn to build resilient services          │   │
│  │                                                      │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### Game Days
```
┌─────────────────────────────────────────────────────────────┐
│                    GAME DAYS                                │
│                                                             │
│  Structured disaster recovery exercises                     │
│                                                             │
│  BEFORE:                                                    │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  □ Define scenarios to test                         │   │
│  │  □ Notify stakeholders                              │   │
│  │  □ Prepare rollback plan                            │   │
│  │  □ Set up monitoring dashboards                     │   │
│  │  □ Brief all participants                           │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  DURING:                                                    │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  □ Execute failure scenario                         │   │
│  │  □ Observe system behavior                          │   │
│  │  □ Time response and recovery                       │   │
│  │  □ Document observations                            │   │
│  │  □ Be ready to abort if needed                      │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  AFTER:                                                     │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  □ Review what happened                             │   │
│  │  □ Identify gaps in resilience                      │   │
│  │  □ Update runbooks                                  │   │
│  │  □ Create action items                              │   │
│  │  □ Schedule follow-up game day                      │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  EXAMPLE SCENARIOS:                                         │
│  • "Primary database fails over"                           │
│  • "Entire AWS region goes down"                           │
│  • "Third-party payment provider unavailable"              │
│  • "DNS provider has outage"                               │
└─────────────────────────────────────────────────────────────┘
```

---

## Summary: Operations Checklist

```
┌────────────────────────────────────────────────────────────┐
│           PRODUCTION READINESS CHECKLIST                   │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  DEPLOYMENT                                                │
│  □ Automated deployment pipeline                          │
│  □ Rollback procedure tested                              │
│  □ Canary/Blue-green deployment ready                     │
│  □ Database migration strategy                            │
│  □ Feature flags for risky changes                        │
│                                                            │
│  MONITORING                                                │
│  □ Key metrics dashboards                                 │
│  □ Latency percentiles tracked (p50, p95, p99)           │
│  □ Distributed tracing enabled                            │
│  □ Log aggregation configured                             │
│  □ SLOs defined and measured                              │
│                                                            │
│  ALERTING                                                  │
│  □ Alerts on symptoms, not causes                         │
│  □ Severity levels defined                                │
│  □ On-call rotation established                           │
│  □ Escalation paths documented                            │
│  □ Alert fatigue minimized                                │
│                                                            │
│  INCIDENT RESPONSE                                         │
│  □ Incident response process documented                   │
│  □ Communication templates ready                          │
│  □ War room / incident channel process                    │
│  □ Post-mortem process established                        │
│                                                            │
│  OPERATIONS                                                │
│  □ Runbooks for common issues                             │
│  □ Automation for repetitive tasks                        │
│  □ Chaos experiments scheduled                            │
│  □ Game days conducted regularly                          │
│  □ Capacity planning reviewed quarterly                   │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

---

## Key Takeaways

1. **Deploy safely** - Use canary/blue-green, automate rollbacks
2. **Measure everything** - Metrics, logs, traces (the three pillars)
3. **Alert on symptoms** - User impact, not internal metrics
4. **Respond quickly** - Mitigate first, debug later
5. **Learn from failures** - Blameless post-mortems
6. **Practice failure** - Chaos engineering, game days
7. **Automate operations** - Reduce toil, increase reliability
