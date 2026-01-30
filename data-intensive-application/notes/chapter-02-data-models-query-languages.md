# Chapter 2: Data Models and Query Languages

## Core Concept

> "Data models are the most important part of developing software."

**Why this matters**: The data model you choose affects everything—how you write code, what queries are easy or hard, and how the system scales. Choosing wrong is expensive to fix later.

**The Layered Thinking**:

Each layer models data in terms of the layer below it. As developers, we mostly work at the top two layers:

```
Application Objects (User, Order, Product)
        ↓ You decide how to represent these...
Data Model (JSON, Tables, Graphs)
        ↓ Database decides how to store this...
Storage Format (Bytes on disk)
        ↓ OS/Hardware handles this...
Hardware (Electrical signals)
```

**Interview tip**: When designing a system, one of your first decisions is: "What data model fits this problem?" This chapter helps you answer that.

---

## Relational vs Document vs Graph Models

### Quick Comparison

| Aspect | Relational (SQL) | Document (NoSQL) | Graph |
|--------|------------------|------------------|-------|
| **Structure** | Tables with rows | JSON/BSON documents | Nodes + Edges |
| **Schema** | Strict, predefined | Flexible, schema-on-read | Flexible |
| **Relationships** | JOINs via foreign keys | Embedded or references | Native edges |
| **Best for** | Structured data, complex queries | Hierarchical, varied structure | Highly connected data |
| **Scaling** | Primarily vertical | Horizontal (sharding) | Depends on implementation |
| **Examples** | PostgreSQL, MySQL | MongoDB, CouchDB | Neo4j, DGraph |

**Simple way to think about it**:
- **Relational**: Data fits neatly into spreadsheets/tables
- **Document**: Data is like JSON files, each record can be different
- **Graph**: Data is about connections between things

---

## Relational Model (SQL)

### What It Is

The relational model organizes data into **tables** (relations), where each table has **rows** (records) and **columns** (fields). Relationships between tables are expressed through **foreign keys** and **JOINs**.

**Think of it like**: A collection of spreadsheets that can reference each other.

### When to Use Relational Databases

1. **Your schema is stable and well-defined**: You know what fields you need upfront
2. **You need complex queries with JOINs**: "Find all orders from users who signed up last month"
3. **ACID transactions are required**: Banking, inventory, anything where consistency is critical
4. **You have reporting/analytics needs**: SQL is great for slicing and dicing data

### Example: E-commerce Schema

```sql
-- Normalized relational design
-- Each piece of data lives in ONE place

CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    email VARCHAR(255) UNIQUE
);

CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    user_id INT REFERENCES users(id),  -- Foreign key relationship
    total DECIMAL(10,2),
    created_at TIMESTAMP
);

CREATE TABLE order_items (
    order_id INT REFERENCES orders(id),
    product_id INT REFERENCES products(id),
    quantity INT,
    PRIMARY KEY (order_id, product_id)
);

-- Query with JOINs - the power of relational
SELECT u.name, COUNT(o.id) as order_count, SUM(o.total) as lifetime_value
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
GROUP BY u.id
HAVING SUM(o.total) > 1000
ORDER BY lifetime_value DESC;
```

**Why this design?**
- User data isn't duplicated in every order
- If user changes email, update once in users table
- Complex queries are natural with JOINs

### The Object-Relational Impedance Mismatch

**The problem**: In application code, you think in objects. In SQL, you think in tables. There's a translation needed.

```
Application Code          Database
┌──────────────┐         ┌─────────────┐
│ User Object  │  ←──→   │ users table │
│ - id         │         │ orders table│
│ - name       │         │ addresses   │
│ - orders[]   │         │ order_items │
│ - addresses[]│         │             │
└──────────────┘         └─────────────┘
     Rich object           Flat tables
     with nested data      with relationships
```

**Solutions**:
- **ORMs** (Hibernate, ActiveRecord, SQLAlchemy): Map objects to tables automatically
- **Accept the mismatch**: Write SQL, map results manually
- **Use documents**: If your data is naturally hierarchical

### Normalization vs Denormalization

**Normalized** (traditional relational): Each fact stored once
- Pro: No data duplication, easy updates
- Con: Requires JOINs to reconstruct full picture

**Denormalized**: Data duplicated for faster reads
- Pro: No JOINs needed, faster reads
- Con: Updates must change multiple places, risk of inconsistency

```sql
-- Normalized: user name in one place
SELECT o.id, u.name 
FROM orders o 
JOIN users u ON o.user_id = u.id;

-- Denormalized: user name copied into orders
-- Faster to read, but if user changes name, must update all orders
SELECT id, user_name FROM orders;
```

---

## Document Model (NoSQL)

### What It Is

Document databases store data as **documents** (usually JSON or BSON). Each document is self-contained and can have a different structure from other documents.

**Think of it like**: A folder full of JSON files, where each file can have different fields.

### When to Use Document Databases

1. **Schema varies between records**: Different users have different profile fields
2. **Data is naturally hierarchical**: A blog post with comments and replies
3. **You need horizontal scaling**: Sharding is built-in
4. **Rapid development**: Schema can evolve without migrations

### Example: Same E-commerce in MongoDB

```javascript
// Document-oriented design - data that's accessed together is stored together
{
  "_id": "user_123",
  "name": "John Doe",
  "email": "john@example.com",
  // Orders are embedded - no JOIN needed!
  "orders": [
    {
      "order_id": "ord_456",
      "total": 99.99,
      "created_at": "2024-01-15",
      "items": [
        { "product": "Widget", "quantity": 2, "price": 29.99 },
        { "product": "Gadget", "quantity": 1, "price": 40.01 }
      ]
    }
  ],
  // Addresses embedded too
  "addresses": [
    { "type": "home", "city": "Seattle", "zip": "98101" },
    { "type": "work", "city": "Bellevue", "zip": "98004" }
  ]
}
```

**Why this design?**
- Fetch user with all their orders in ONE query (no JOINs)
- Data that belongs together is stored together
- Great for: "Show me everything about user X"

### Schema-on-Read vs Schema-on-Write

**Schema-on-Write** (Relational):
- Define schema upfront (CREATE TABLE)
- Database enforces schema on every write
- Changing schema requires migration (ALTER TABLE)

**Schema-on-Read** (Document):
- No upfront schema definition
- Application interprets structure when reading
- Different documents can have different fields

```javascript
// Schema-on-read: Handle variations in application code
function getFullName(user) {
  // Some old documents have separate fields
  if (user.first_name && user.last_name) {
    return `${user.first_name} ${user.last_name}`;
  }
  // Newer documents have combined field
  return user.full_name || "Unknown";
}

// vs SQL schema-on-write: Must migrate all data
ALTER TABLE users ADD COLUMN full_name VARCHAR(200);
UPDATE users SET full_name = first_name || ' ' || last_name;
ALTER TABLE users DROP COLUMN first_name, DROP COLUMN last_name;
-- Hope nothing breaks during migration!
```

### Document Model Limitations

**The many-to-many problem**:

```
                    ┌─────────────┐
              ┌────►│   Author    │
              │     └─────────────┘
┌─────────┐   │     
│  Book   ├───┤     A book can have multiple authors
└─────────┘   │     An author can write multiple books
              │     ┌─────────────┐
              └────►│   Author    │
                    └─────────────┘
```

**Options (all have downsides)**:
1. **Embed authors in book**: If author updates bio, must update in ALL their books
2. **Store author IDs, fetch separately**: Now you're doing JOINs in application code
3. **Accept denormalization**: Duplicate author data, accept inconsistency risk

**Rule of thumb**: Documents work great for one-to-many (user → orders). They struggle with many-to-many (books ↔ authors).

### When Documents Become Awkward

Your document model might be wrong if you find yourself:
- Doing multiple queries to reconstruct data (manual JOINs)
- Updating the same data in many documents
- Documents growing unboundedly large
- Needing transactions across multiple documents

---

## Graph Model

### What It Is

Graph databases store **nodes** (entities) and **edges** (relationships). Both nodes and edges can have properties. Traversing relationships is a first-class operation.

**Think of it like**: A social network map where you can easily find "friends of friends."

### When to Use Graph Databases

1. **Social networks**: Users, follows, friendships
2. **Fraud detection**: Finding suspicious patterns in transactions
3. **Recommendation engines**: "People who liked X also liked Y"
4. **Knowledge graphs**: Wikipedia-style interconnected information
5. **Network/IT infrastructure**: Dependencies between services

**Key insight**: If you find yourself writing recursive SQL queries or multiple hops of JOINs, you probably want a graph database.

### Property Graph Model

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│   (Alice)──────FRIEND──────►(Bob)                          │
│   {age:25}     {since:2020}  {age:30}                      │
│      │                         │                            │
│      │                         │                            │
│   WORKS_AT                  WORKS_AT                        │
│   {role:SWE}                {role:PM}                       │
│      │                         │                            │
│      ▼                         ▼                            │
│   (Google)◄────ACQUIRED────(YouTube)                       │
│   {founded:1998}  {year:2006}  {founded:2005}             │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Key concepts**:
- **Nodes**: Entities (Alice, Bob, Google, YouTube)
- **Edges**: Relationships (FRIEND, WORKS_AT, ACQUIRED)
- **Properties**: Key-value pairs on both nodes and edges

### Cypher Query Language (Neo4j)

Cypher is designed for graph patterns. It reads almost like English:

```cypher
// Find friends of friends who work at the same company
MATCH (person:User {name: 'Alice'})-[:FRIEND]->(friend)-[:FRIEND]->(foaf),
      (person)-[:WORKS_AT]->(company)<-[:WORKS_AT]-(foaf)
WHERE NOT (person)-[:FRIEND]->(foaf)
RETURN foaf.name, company.name

// Translation:
// 1. Start with Alice
// 2. Find her friends
// 3. Find their friends (foaf = friend of a friend)
// 4. Filter to those who work at Alice's company
// 5. Exclude people Alice is already friends with
```

**Compare to SQL** (painful!):

```sql
-- Same query requires recursive CTEs
WITH RECURSIVE friends AS (
    SELECT friend_id, 1 as depth FROM friendships WHERE user_id = 'alice'
    UNION ALL
    SELECT f.friend_id, fr.depth + 1
    FROM friendships f JOIN friends fr ON f.user_id = fr.friend_id
    WHERE fr.depth < 2
)
SELECT DISTINCT u.name, c.name
FROM friends fr
JOIN users u ON fr.friend_id = u.id
JOIN works_at w1 ON u.id = w1.user_id
JOIN works_at w2 ON w2.company_id = w1.company_id AND w2.user_id = 'alice'
WHERE fr.depth = 2
AND fr.friend_id NOT IN (SELECT friend_id FROM friendships WHERE user_id = 'alice');
```

**The graph query is clearer AND often faster** because graph DBs optimize for traversals.

---

## Query Languages

### Declarative vs Imperative

**Imperative** (how to get results): You specify the steps
**Declarative** (what results you want): You describe the outcome, system figures out how

```javascript
// Imperative: HOW to get results (JavaScript)
function getAdultUsers(users) {
    const results = [];
    for (const user of users) {
        if (user.age >= 18) {
            results.push(user);
        }
    }
    return results.sort((a, b) => a.name.localeCompare(b.name));
}

// Declarative: WHAT results you want (SQL)
SELECT * FROM users WHERE age >= 18 ORDER BY name;
```

**Why declarative wins**:
1. **Optimization**: Database can choose best execution plan
2. **Parallelization**: Database can use multiple cores automatically
3. **Readability**: Shorter, more expressive
4. **Flexibility**: Change indexes, query works the same

### MapReduce: Historical Context

MapReduce was popular in the Hadoop era but has been largely replaced by better abstractions.

```javascript
// MongoDB MapReduce (legacy - don't use in new code)
db.orders.mapReduce(
    function() { emit(this.product_id, this.quantity); },  // Map
    function(key, values) { return Array.sum(values); },   // Reduce
    { out: "product_totals" }
);

// Modern equivalent: Aggregation Pipeline (use this!)
db.orders.aggregate([
    { $group: { _id: "$product_id", total: { $sum: "$quantity" } } }
]);
```

**Why aggregation pipelines won**: More declarative, easier to optimize, easier to read.

---

## Practical Decision Framework

```
┌─────────────────────────────────────────────────────────────┐
│                   Choose Your Data Model                     │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Is your data highly interconnected?                        │
│  (social networks, recommendations, fraud detection)        │
│       │                                                     │
│       ├── YES → Graph Database (Neo4j, DGraph)             │
│       │         Great for traversing relationships          │
│       │                                                     │
│       └── NO ↓                                              │
│                                                             │
│  Is your schema stable and well-defined?                   │
│  (you know all the fields upfront)                         │
│       │                                                     │
│       ├── YES → Relational (PostgreSQL)                    │
│       │         Great for JOINs, ACID, complex queries     │
│       │                                                     │
│       └── NO → Document (MongoDB)                          │
│               Great for flexible schemas, nested data       │
│                                                             │
│  Do you need ACID transactions across multiple records?    │
│       │                                                     │
│       ├── YES → Strongly favor Relational                  │
│       │                                                     │
│       └── NO → Document or Graph can work                  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Real-World Polyglot Persistence

**Big insight**: Modern systems often use MULTIPLE databases, each for what it's best at.

```
┌─────────────────────────────────────────────────────────────┐
│                    E-commerce Platform                       │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  PostgreSQL          MongoDB           Neo4j               │
│  ───────────         ────────          ─────               │
│  • Orders            • Product         • Recommendations   │
│    (ACID needed)       catalog         • "Customers who    │
│  • Payments            (varied attrs)    bought X..."      │
│    (transactions)    • Reviews         • Fraud detection   │
│  • Inventory           (flexible)                          │
│    (consistency)     • User sessions                       │
│                                                             │
│  ──────────────────────────────────────────────────────────│
│                    Elasticsearch                            │
│                    ─────────────                            │
│                    • Full-text search                       │
│                    • Faceted navigation                     │
│                    • Autocomplete                           │
│                                                             │
│  ──────────────────────────────────────────────────────────│
│                    Redis                                    │
│                    ─────                                    │
│                    • Session cache                          │
│                    • Rate limiting                          │
│                    • Real-time leaderboards                 │
└─────────────────────────────────────────────────────────────┘
```

**The trade-off**: More databases = more operational complexity. Start simple (PostgreSQL can do a LOT), add specialized DBs when you have a specific need.

---

## System Design Interview Tips

### Common Questions and Model Choices

| System | Likely Choice | Why |
|--------|---------------|-----|
| E-commerce | PostgreSQL + Elasticsearch | Transactions for orders, search for products |
| Social Network | Graph + PostgreSQL | Relationships are core, need some transactions |
| Content Management | Document (MongoDB) | Flexible content structures |
| Banking | PostgreSQL | ACID is non-negotiable |
| Real-time Analytics | Time-series DB | Optimized for time-ordered data |

### Red Flags in Interviews

❌ "We'll use NoSQL because it scales" (wrong reason—SQL scales too)
❌ "We'll use SQL for everything" (ignoring specialized needs)
❌ Choosing database before understanding access patterns

### Good Signals

✅ "What are the main access patterns?"
✅ "What consistency guarantees do we need?"
✅ "How will the data grow?"
✅ "Let's start with X, but we can add Y later if needed"

---

## Key Takeaways

1. **No one-size-fits-all**: Different data models for different use cases
2. **Relational** still dominates for structured, transactional data—don't dismiss SQL
3. **Document** model great for flexibility, but watch for many-to-many relationships
4. **Graph** model natural for connected data—painful to replicate in SQL
5. **Declarative queries** (SQL, Cypher) let database optimize; prefer over imperative
6. **Polyglot persistence**: Use multiple databases for different needs, but add complexity thoughtfully

---

## Quick Reference

| Need | Model | Example DB | Why |
|------|-------|------------|-----|
| ACID transactions | Relational | PostgreSQL | Built-in transaction support |
| Flexible schema | Document | MongoDB | Schema-on-read |
| Social/recommendation | Graph | Neo4j | Fast relationship traversal |
| Full-text search | Inverted Index | Elasticsearch | Optimized for text matching |
| Time series | Columnar | InfluxDB, TimescaleDB | Optimized for time-ordered writes |
| Caching | Key-Value | Redis | In-memory speed |
