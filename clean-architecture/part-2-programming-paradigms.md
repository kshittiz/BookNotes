# Clean Architecture Notes
## A Craftsman's Guide to Software Structure and Design
### By Robert C. Martin (Uncle Bob)

---

# Part II: Programming Paradigms - The Building Blocks

## Overview

Software is built using three fundamental programming paradigms. Interestingly, **each paradigm restricts what programmers can do**, rather than adding new capabilities. They tell us what *not* to do.

```
┌─────────────────────────────────────────────────────────────────┐
│                    THREE PARADIGMS                              │
├─────────────────────────────────────────────────────────────────┤
│  Structured Programming  →  Restricts: goto statements         │
│  Object-Oriented         →  Restricts: function pointers       │
│  Functional Programming  →  Restricts: variable assignment     │
└─────────────────────────────────────────────────────────────────┘
```

**Key Insight**: All three paradigms were discovered between 1958-1968. No new paradigms have emerged since!

---

## Chapter 3 & 4: Structured Programming

### The Core Idea

**Structured programming imposes discipline on direct transfer of control.**

Before structured programming, code used `goto` statements everywhere, making programs hard to reason about and prove correct.

### The Three Control Structures

Dijkstra proved that **all programs can be built from just three structures**:

```
┌──────────────────┬──────────────────┬──────────────────┐
│    SEQUENCE      │    SELECTION     │    ITERATION     │
├──────────────────┼──────────────────┼──────────────────┤
│   Do A           │   if (x)         │   while (x)      │
│   Then B         │     do A         │     do A         │
│   Then C         │   else           │                  │
│                  │     do B         │                  │
└──────────────────┴──────────────────┴──────────────────┘
```

### Why It Matters for Architecture

> "Software is like a science. We show correctness by failing to prove incorrectness."

- Programs can be **recursively decomposed** into smaller, testable functions
- Each small function can be tested (falsified) independently
- This is why **functional decomposition** remains a best practice

**Example:**
```
Large System
    └── Module A
    │       └── Function A1
    │       └── Function A2
    └── Module B
            └── Function B1
            └── Function B2
```

Each level can be tested independently!

---

## Chapter 5: Object-Oriented Programming

### The Core Idea

**OO imposes discipline on indirect transfer of control (function pointers).**

OO is NOT just "data + functions" or "modeling the real world." From an architect's perspective, OO is about one thing: **POLYMORPHISM**.

### The Three Pillars (With Caveats)

| Pillar | What It Really Means |
|--------|---------------------|
| **Encapsulation** | OO languages actually *weakened* encapsulation compared to C |
| **Inheritance** | Convenient, but existed before OO (just less convenient) |
| **Polymorphism** | The real power - OO made it safe and convenient |

### The Real Power: Dependency Inversion

Before OO, source code dependencies followed the flow of control:

```
BEFORE OO:
┌──────────┐      ┌──────────┐      ┌──────────┐
│   Main   │ ──→  │ High-Lvl │ ──→  │  Low-Lvl │
└──────────┘      └──────────┘      └──────────┘
     │                  │                 │
     ▼                  ▼                 ▼
  Flow of Control AND Source Code Dependencies
  go in the SAME direction
```

With OO and polymorphism, we can **invert** dependencies:

```
AFTER OO (with interfaces):
┌──────────────────┐      ┌─────────────────────┐
│  Business Rules  │ ←──  │  «interface»        │
│  (High-Level)    │      │  DataGateway        │
└──────────────────┘      └─────────────────────┘
                                   △
                                   │ implements
                          ┌───────────────────┐
                          │  Database         │
                          │  (Low-Level)      │
                          └───────────────────┘

Source Code Dependency points OPPOSITE to flow of control!
```

### The Plugin Architecture

This inversion allows us to make low-level details (database, UI) into **plugins** to high-level business rules:

```
                    ┌─────────────────┐
                    │ Business Rules  │
                    │   (Core)        │
                    └────────┬────────┘
                             │
              ┌──────────────┼──────────────┐
              ▼              ▼              ▼
        ┌──────────┐  ┌──────────┐  ┌──────────┐
        │    UI    │  │ Database │  │   Web    │
        │ (Plugin) │  │ (Plugin) │  │ (Plugin) │
        └──────────┘  └──────────┘  └──────────┘
```

**Benefits:**
- Business rules never mention UI or database
- Components can be deployed independently
- Teams can develop independently

---

## Chapter 6: Functional Programming

### The Core Idea

**Functional programming imposes discipline on variable assignment.**

In functional languages, **variables don't change** (immutability).

### Why Immutability Matters

All concurrency problems stem from mutable variables:

| Problem | Caused By |
|---------|-----------|
| Race conditions | Multiple threads changing same variable |
| Deadlocks | Locking mutable resources |
| Concurrent update issues | Simultaneous writes |

**No mutable variables = No concurrency problems!**

### Practical Approach: Segregation of Mutability

Since pure immutability isn't always practical, separate your application:

```
┌─────────────────────────────────────────────────────┐
│                    APPLICATION                       │
├──────────────────────┬──────────────────────────────┤
│   IMMUTABLE          │         MUTABLE              │
│   COMPONENTS         │         COMPONENTS           │
│                      │                              │
│   Pure functions     │   Transactional memory       │
│   No side effects    │   Protected by locks         │
│   Easy to test       │   Minimized scope            │
│   Easy to parallelize│                              │
└──────────────────────┴──────────────────────────────┘

                ↑ Push as much code here as possible
```

### Event Sourcing Pattern

Instead of storing current state, store all transactions:

```
Traditional:                    Event Sourcing:
┌────────────────┐              ┌────────────────┐
│ Balance: $500  │              │ +$1000 (deposit)│
└────────────────┘              │ -$200 (withdraw)│
                                │ -$300 (purchase)│
                                │ = $500 (computed)│
                                └────────────────┘
```

**Benefits:**
- Nothing ever deleted or updated
- Complete audit trail
- No concurrent update issues
- (Your source control works this way!)

---

## Part II Summary

```
┌───────────────────────────────────────────────────────────────┐
│ Paradigm          │ Restriction        │ Architectural Use    │
├───────────────────┼────────────────────┼──────────────────────┤
│ Structured        │ goto statements    │ Algorithmic          │
│                   │                    │ decomposition        │
├───────────────────┼────────────────────┼──────────────────────┤
│ Object-Oriented   │ Function pointers  │ Cross architectural  │
│                   │                    │ boundaries           │
├───────────────────┼────────────────────┼──────────────────────┤
│ Functional        │ Assignment         │ Data management &    │
│                   │                    │ concurrency          │
└───────────────────┴────────────────────┴──────────────────────┘
```

---

*Notes created from "Clean Architecture" by Robert C. Martin*
