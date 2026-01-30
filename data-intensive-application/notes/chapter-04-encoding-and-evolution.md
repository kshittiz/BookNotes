# Chapter 4: Encoding and Evolution

## Core Concept

> "Applications change over time. Old and new code, old and new data formats must coexist."

**What it is:** This chapter covers how to serialize data (encoding) and how to evolve your data formats without breaking things.

**Why it matters for interviews:** Every system design involves data flowing between services. Understanding encoding helps you:
- Choose between JSON, Protobuf, Avro (and explain WHY)
- Design APIs that can evolve without breaking clients
- Handle rolling deployments where multiple versions coexist

**The Challenge:**
```
┌─────────────────────────────────────────────────────────────┐
│                Rolling Deployment Reality                    │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   Server 1: v2.0  ◄─── writes new format                   │
│   Server 2: v1.9  ◄─── reads new format? (forward compat)  │
│   Server 3: v2.0  ◄─── reads old format? (backward compat) │
│                                                             │
│   Mobile App: v1.8 ◄─── May NEVER update!                  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

> 💡 **Interview Tip:** When designing any distributed system, immediately think about versioning. "How will we deploy updates without downtime?"

---

## Compatibility Types

**What they mean:**

| Type | Definition | Real-World Example |
|------|------------|---------|
| **Backward** | New code can read old data | New server reads database records written by old server |
| **Forward** | Old code can read new data | Old mobile app handles new API response fields |

**Why both matter:** During a rolling deployment, you have BOTH old and new code running simultaneously. You need both types of compatibility!

**The golden rule:** New code must handle old data (backward), and old code must gracefully ignore new fields (forward).

> 💡 **Interview Tip:** When asked about API versioning, mention both compatibility types. "We need backward compatibility for old data in the database, and forward compatibility for old clients still calling our API."

---

## Encoding Formats: When to Use Each

### JSON (The Default Choice)

**What it is:** Human-readable text format. The lingua franca of web APIs.

```json
{
  "userName": "Alice",
  "favoriteNumber": 42,
  "interests": ["music", "coding"]
}
```

**When to use JSON:**
- ✅ Public APIs (everyone can read it, debug it, curl it)
- ✅ Config files humans need to edit
- ✅ Prototyping and early-stage projects
- ✅ Browser/JavaScript clients (native support)

**When NOT to use JSON:**
- ❌ High-throughput internal services (too verbose)
- ❌ When you need schema enforcement
- ❌ Large numeric IDs (precision issues!)

```javascript
// JavaScript's famous number precision problem
JSON.parse('{"id": 9007199254740993}')
// Returns: {id: 9007199254740992}  ← SILENTLY WRONG!

// Solution: Use strings for large IDs
{"id": "9007199254740993"}
```

> 💡 **Interview Tip:** Twitter and many companies use string IDs in their JSON APIs specifically because of JavaScript's number precision limit (2^53).

### Protocol Buffers (Google's Choice)

**What it is:** Binary format with a schema. Smaller and faster than JSON.

```protobuf
// schema.proto - the schema is separate from the data
message Person {
  required string user_name = 1;     // field tag = 1
  optional int64 favorite_number = 2; // field tag = 2
  repeated string interests = 3;      // field tag = 3
}
```

**The key insight:** Field TAGS (1, 2, 3) are stored in the binary data, not field NAMES. This is why it's compact.

**Binary encoding:**
```
┌─────────────────────────────────────────────────────────────┐
│  Tag 1 (string): 0A 05 41 6C 69 63 65  ("Alice")          │
│  Tag 2 (int64):  10 2A                  (42)               │
│  Tag 3 (string): 1A 05 6D 75 73 69 63  ("music")          │
│  Tag 3 (string): 1A 06 63 6F 64 69 6E 67 ("coding")       │
│                                                             │
│  Total: 27 bytes (vs ~70 bytes JSON) = 60% smaller        │
└─────────────────────────────────────────────────────────────┘
```

**When to use Protobuf:**
- ✅ Internal microservices (gRPC)
- ✅ Performance-critical paths
- ✅ Mobile apps (smaller payloads = faster)
- ✅ When you want type safety and code generation

**Used by:** Google (everything), Uber, Netflix, Square

### Avro (The Data Pipeline Choice)

**What it is:** Binary format where schema is stored WITH the data (or in a registry). No field tags in the data itself.

```json
// Schema (JSON format, but data is binary)
{
  "type": "record",
  "name": "Person",
  "fields": [
    {"name": "userName", "type": "string"},
    {"name": "favoriteNumber", "type": ["null", "long"], "default": null},
    {"name": "interests", "type": {"type": "array", "items": "string"}}
  ]
}
```

**Key insight:** Unlike Protobuf, Avro doesn't store field identifiers in the data. The reader's schema and writer's schema are compared at read time.

**Why Avro for data pipelines?**
- Schema is stored with the data file (self-describing)
- Schema evolution is more flexible
- Better for dynamically typed languages (no code generation needed)
- Hadoop ecosystem uses it extensively

**When to use Avro:**
- ✅ Kafka message queues (with Schema Registry)
- ✅ Data warehousing and Hadoop pipelines
- ✅ When schemas change frequently
- ✅ When you need to generate schemas dynamically

**Used by:** LinkedIn, Confluent Kafka, most data engineering teams

### Format Comparison Summary

| Format | Size | Schema | Human Readable | Best For |
|--------|------|--------|----------------|----------|
| JSON | Large | No | Yes | Public APIs, debugging |
| Protobuf | Small | Yes (.proto) | No | gRPC, mobile apps |
| Avro | Smallest | Yes (JSON) | No | Kafka, data pipelines |
| Thrift | Small | Yes (.thrift) | No | Facebook services |

> 💡 **Interview Tip:** "For internal services, I'd use Protobuf/gRPC for type safety and performance. For public APIs, JSON for debuggability. For Kafka, Avro with Schema Registry for schema evolution."

---

## Schema Evolution: The Rules

**What it is:** How to change your schema without breaking existing readers/writers.

### Protobuf/Thrift Evolution Rules

```protobuf
// Version 1
message Person {
  required string name = 1;
  optional string email = 2;
}

// Version 2 - Adding field (SAFE ✓)
message Person {
  required string name = 1;
  optional string email = 2;
  optional string phone = 3;  // NEW - old code ignores unknown tags
}

// Version 3 - Removing field (CAREFUL!)
message Person {
  required string name = 1;
  // email removed - OK because it was optional
  optional string phone = 3;
  // CRITICAL: NEVER reuse tag number 2!
}
```

**Why never reuse tags?** Old data still has tag 2 with email values. If you reuse tag 2 for a new field, old data will be misinterpreted!

### The Golden Rules

| Change | Backward Safe? | Forward Safe? |
|--------|---------------|--------------|
| Add optional field | ✓ Yes | ✓ Yes |
| Add required field | ✗ No (old data lacks it) | ✓ Yes |
| Remove optional field | ✓ Yes | ✓ Yes |
| Remove required field | ✓ Yes | ✗ No (new code expects it) |
| Rename field | ✓ (same tag) | ✓ (same tag) |
| Change field type | ⚠️ Depends | ⚠️ Depends |

**Simple rule:** Make new fields optional with defaults. Never remove required fields. Never reuse tag numbers.

### Avro Schema Evolution

**Avro is different:** It uses schema resolution at read time. The writer's schema and reader's schema don't need to be identical—Avro figures out how to translate.

```json
// Writer schema (v1) - what was written to disk
{
  "type": "record",
  "name": "User",
  "fields": [
    {"name": "name", "type": "string"}
  ]
}

// Reader schema (v2) - what current code expects
{
  "type": "record", 
  "name": "User",
  "fields": [
    {"name": "name", "type": "string"},
    {"name": "email", "type": "string", "default": "unknown@example.com"}
  ]
}
```

**How it works:** When reading old data:
1. Avro sees writer had only `name`
2. Reader wants `name` and `email`
3. Avro copies `name`, fills `email` with default value

**Why this is powerful:** You can add fields without coordinating deployments. Old data automatically gets default values.

> 💡 **Interview Tip:** "Avro's reader/writer schema resolution is why it's popular for Kafka—producers and consumers can evolve schemas independently as long as they're compatible."

---

## Data Flow Patterns

Data flows between systems in three main ways. Each has different compatibility considerations.

### 1. Through Databases

**The challenge:** Data written today might be read by code 5 years from now!

```
┌─────────────────────────────────────────────────────────────┐
│                   Database Evolution                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   App v1 writes → Database ← App v2 reads (backward)       │
│   App v2 writes → Database ← App v1 reads (forward)        │
│                                                             │
│   Challenge: Data outlives code!                           │
│   - Row written in 2015 may be read by 2024 code          │
│   - You can't just "update all the data"                   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Solutions:**
- **Schema migrations:** ALTER TABLE (but careful with large tables—can lock for hours!)
- **Dual-write:** Write to old AND new columns during transition
- **Lazy migration:** Convert data format when rows are read/updated

### 2. Through Services (REST/RPC)

**What it is:** APIs between services or between server and clients.

```
┌─────────────────────────────────────────────────────────────┐
│                    API Versioning Strategies                 │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   URL versioning (most common):                            │
│   GET /api/v1/users/123                                    │
│   GET /api/v2/users/123                                    │
│                                                             │
│   Header versioning (cleaner URLs):                        │
│   Accept: application/vnd.myapi.v2+json                    │
│                                                             │
│   Query param versioning:                                  │
│   GET /api/users/123?version=2                             │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Best practice:** Support N-1 versions (current and previous). When v3 releases, deprecate v1, keep v2 working.

> 💡 **Interview Tip:** "URL versioning is most common because it's explicit and cacheable, but header versioning is theoretically cleaner. The key is picking one and being consistent."

### REST vs RPC (gRPC)

**What's the difference?**

```
┌────────────────────────┬────────────────────────────────────┐
│         REST           │           gRPC (RPC)               │
├────────────────────────┼────────────────────────────────────┤
│ Resources + HTTP verbs │ Function calls                     │
│ GET /users/123         │ userService.GetUser(123)           │
│                        │                                    │
│ JSON (usually)         │ Protobuf (binary)                  │
│ Human-debuggable       │ Needs special tooling              │
│ Cacheable (GET)        │ Not cacheable                      │
│ Browser-friendly       │ Needs grpc-web                     │
│                        │                                    │
│ Loose coupling         │ Tighter coupling (shared .proto)   │
└────────────────────────┴────────────────────────────────────┘
```

**When to use REST:**
- Public APIs
- Browser/mobile clients
- When debuggability matters
- CRUD operations on resources

**When to use gRPC:**
- Internal microservices
- High-throughput, low-latency needs
- Streaming (bidirectional!)
- When type safety is important

### gRPC Example

```protobuf
// user.proto - defines the contract
service UserService {
  rpc GetUser(GetUserRequest) returns (User);
  rpc CreateUser(CreateUserRequest) returns (User);
  rpc ListUsers(ListUsersRequest) returns (stream User);  // streaming!
}

message User {
  int64 id = 1;
  string name = 2;
  string email = 3;
}
```

```python
# Client code (generated from .proto)
channel = grpc.insecure_channel('localhost:50051')
stub = user_pb2_grpc.UserServiceStub(channel)
user = stub.GetUser(GetUserRequest(user_id=123))
print(user.name)  # Type-safe!
```

> 💡 **Interview Tip:** "gRPC is great for internal services because of performance and type safety, but I'd put a REST gateway in front for public clients. This gives you the best of both worlds."

### 3. Through Message Queues (Kafka, RabbitMQ)

**The challenge:** Producers and consumers deploy independently. A message written today might be consumed by code deployed weeks later.

```
┌─────────────────────────────────────────────────────────────┐
│              Async Message Passing                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   Producer (v2) ──► Kafka Topic ──► Consumer (v1)          │
│                                                             │
│   Problem: Consumer doesn't know producer's schema!         │
│                                                             │
│   Solution: Schema Registry                                 │
│                                                             │
│   ┌──────────┐    ┌─────────┐    ┌──────────┐             │
│   │ Producer │───►│  Kafka  │───►│ Consumer │             │
│   └────┬─────┘    └─────────┘    └────┬─────┘             │
│        │                              │                     │
│        │    ┌─────────────────┐       │                     │
│        └───►│ Schema Registry │◄──────┘                     │
│             │  (Confluent)    │                             │
│             └─────────────────┘                             │
│                                                             │
│   How it works:                                             │
│   1. Producer registers schema, gets schema ID             │
│   2. Message contains [schema_id][data] (ID is 4 bytes)   │
│   3. Consumer fetches schema by ID, deserializes          │
│   4. Registry validates compatibility on registration      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Why Schema Registry matters:**
- Validates compatibility BEFORE bad messages get published
- Stores all schema versions (audit trail)
- Smaller messages (ID instead of full schema)
- Consumers can evolve independently

> 💡 **Interview Tip:** "For Kafka, I'd use Avro with Confluent Schema Registry. It enforces compatibility at the registry level, so we catch breaking changes before they hit production."

---

## Practical Patterns

### The Expand-Contract Pattern (Three-Phase Migration)

**What it is:** A safe way to make breaking schema changes in production.

```
Phase 1: EXPAND (add new, keep old)
┌────────────────────────────────────┐
│ Old fields: first_name, last_name  │
│ New field: full_name (added)       │
│ Code writes: ALL THREE fields      │
│ Code reads: old fields (safe)      │
└────────────────────────────────────┘

Phase 2: MIGRATE (update all code and data)
┌────────────────────────────────────┐
│ Deploy code that reads full_name   │
│ Backfill old data with full_name   │
│ Verify all consumers updated       │
└────────────────────────────────────┘

Phase 3: CONTRACT (remove old)
┌────────────────────────────────────┐
│ Remove first_name, last_name       │
│ Only full_name remains             │
│ Clean up writing code              │
└────────────────────────────────────┘
```

**Why three phases?** 
- Phase 1→2 is backward compatible (new code reads old data)
- Phase 2→3 is forward compatible (old code handles new data)
- At no point do you break running code!

> 💡 **Interview Tip:** "For any breaking change, I'd use the expand-contract pattern. It takes three deployments but guarantees zero downtime."

### Handling Unknown Fields (Forward Compatibility)

**The problem:** Old code receives a response with new fields it doesn't know about. What should it do?

```python
# BAD: Discards unknown fields
class User:
    def __init__(self, data):
        self.name = data['name']
        self.email = data['email']
        # New fields silently discarded!
    
    def to_dict(self):
        return {'name': self.name, 'email': self.email}

# GOOD: Preserves unknown fields
class User:
    def __init__(self, data):
        self.name = data.get('name')
        self.email = data.get('email')
        self._extra = {k: v for k, v in data.items() 
                       if k not in ('name', 'email')}
    
    def to_dict(self):
        result = {'name': self.name, 'email': self.email}
        result.update(self._extra)  # Preserve unknown fields!
        return result
```

**Why this matters:** If old code reads data, modifies one field, and writes it back, you don't want to lose the new fields that were added!

---

## Real-World Implementation Examples

### Kafka + Avro + Schema Registry

```python
from confluent_kafka import SerializingProducer
from confluent_kafka.schema_registry import SchemaRegistryClient
from confluent_kafka.schema_registry.avro import AvroSerializer

# Connect to Schema Registry
schema_registry = SchemaRegistryClient({'url': 'http://localhost:8081'})

# Schema will be registered automatically
schema_str = """
{
  "type": "record",
  "name": "User",
  "fields": [
    {"name": "id", "type": "long"},
    {"name": "name", "type": "string"},
    {"name": "email", "type": ["null", "string"], "default": null}
  ]
}
"""

avro_serializer = AvroSerializer(schema_registry, schema_str)

producer = SerializingProducer({
    'bootstrap.servers': 'localhost:9092',
    'value.serializer': avro_serializer
})

# If this schema is incompatible with previous versions,
# the registry will REJECT it before any messages are sent!
producer.produce(
    topic='users',
    value={'id': 123, 'name': 'Alice', 'email': 'alice@example.com'}
)
```

### Database Migration with Zero Downtime

```sql
-- Phase 1: Add new column (nullable, no default = instant on modern DBs)
ALTER TABLE users ADD COLUMN full_name VARCHAR(200);

-- Phase 2: Backfill (in batches to avoid locking!)
-- Run this in application code, not one big UPDATE
UPDATE users 
SET full_name = CONCAT(first_name, ' ', last_name)
WHERE full_name IS NULL
  AND id BETWEEN 1 AND 10000;
-- Repeat for next batch...

-- Phase 3: Deploy code that writes to both columns

-- Phase 4: Deploy code that reads from new column

-- Phase 5: Make new column required (after ALL data migrated)
ALTER TABLE users ALTER COLUMN full_name SET NOT NULL;

-- Phase 6: Remove old columns (after all code updated)
ALTER TABLE users DROP COLUMN first_name, DROP COLUMN last_name;
```

**Why batches?** Large UPDATE statements can lock the table and cause downtime. Small batches keep the database responsive.

---

## Key Takeaways

1. **JSON for humans, binary for machines:** Use JSON for public APIs and debugging, Protobuf/Avro for internal high-throughput services
2. **Never reuse field tags/IDs:** Old data still references them
3. **New fields must be optional** with defaults—this ensures backward compatibility
4. **Schema Registry is essential** for message queues—catch breaking changes early
5. **Expand-Contract pattern** for safe migrations—takes 3 deploys but zero downtime
6. **Test compatibility** before deploying—automate this in CI/CD

---

## Quick Reference

| Use Case | Recommended Format | Why |
|----------|-------------------|-----|
| Public API | JSON + OpenAPI | Human-readable, universal tooling |
| Internal microservices | gRPC + Protobuf | Type safety, performance, streaming |
| Kafka messages | Avro + Schema Registry | Schema evolution, compatibility checks |
| Config files | JSON or YAML | Human-editable |
| Data warehouse | Parquet (columnar) | Analytics-optimized, compressed |
| Browser/Mobile | JSON (or Protobuf) | Browser-native, or protobuf.js for perf |

---

## System Design Interview Tips

1. **Always discuss compatibility:** "During rolling deployments, we'll have both v1 and v2 servers. We need backward compatibility (new reads old) and forward compatibility (old ignores new fields)."

2. **Know the trade-offs:**
   - JSON: Easy to debug, but verbose and no schema enforcement
   - Protobuf: Fast and compact, but need to compile schemas
   - Avro: Most flexible evolution, but more complex setup

3. **Mention Schema Registry for Kafka:** "For Kafka, I'd use Avro with Confluent Schema Registry. It validates compatibility at registration time, so we never publish incompatible messages."

4. **Explain API versioning strategy:** "I'd use URL versioning (/v1/, /v2/) because it's explicit and cacheable. We'd support N-1 versions and give clients deprecation warnings."

5. **Database migrations:** "For schema changes, I'd use the expand-contract pattern: add new column, backfill in batches, migrate code, then remove old column. Never big-bang migrations."

---

## Common Interview Questions

**Q: "How do you choose between JSON and Protobuf?"**
A: JSON for public APIs (human-readable, universal), Protobuf for internal services (smaller, faster, type-safe). The overhead of Protobuf setup isn't worth it for low-traffic endpoints, but for high-throughput internal services, the 10x size reduction matters.

**Q: "How do you handle API versioning?"**
A: URL versioning (/api/v1/users) is most common—it's explicit and works well with caches. Support the current version plus the previous version. Deprecate with headers (Sunset, Deprecation) and give clients time to migrate. For breaking changes, bump the major version.

**Q: "What's backward vs forward compatibility?"**
A: Backward = new code can read old data (e.g., new server reads old database rows). Forward = old code can handle new data (e.g., old mobile app receives new API fields). You need BOTH for zero-downtime deployments because during rollout, both versions run simultaneously.

**Q: "How would you migrate a database column safely?"**
A: Expand-contract pattern: (1) Add new nullable column, (2) Deploy code writing to both, (3) Backfill old rows in batches, (4) Deploy code reading from new column, (5) Make new column NOT NULL, (6) Remove old column. Never do it in one step—that risks downtime.

**Q: "Why use Schema Registry with Kafka?"**
A: Producers and consumers deploy independently, so messages in Kafka might be read by code deployed weeks later. Schema Registry validates compatibility at publish time—if a producer tries to register an incompatible schema, it fails BEFORE any bad messages get published. It also stores all versions for audit and makes messages smaller (schema ID instead of full schema).
