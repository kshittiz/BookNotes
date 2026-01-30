# Clean Architecture Notes
## A Craftsman's Guide to Software Structure and Design
### By Robert C. Martin (Uncle Bob)

---

# Part IV: Component Principles

## Overview

If SOLID principles are about arranging bricks into walls, component principles are about arranging rooms into buildings.

**Components** = Units of deployment (JAR files, DLLs, Gems, etc.)

---

## Chapter 12: What Are Components?

### Definition

Components are the **smallest entities that can be deployed** as part of a system.

```
Language        │ Component
────────────────┼───────────────
Java            │ .jar files
Ruby            │ .gem files
.NET            │ .dll files
Python          │ packages
JavaScript      │ npm modules
```

### Historical Evolution

```
1950s-60s: Programs = one big chunk
    ↓
1970s: Linking at compile time (slow!)
    ↓
1980s: Separate linking step
    ↓
1990s+: Dynamic linking at load time
    ↓
Today: Plugin architectures are easy!
```

**Modern reality:** We can now easily create plugin architectures where components are loaded dynamically at runtime.

---

## Chapter 13: Component Cohesion

Three principles guide which classes belong in which components:

### 1. REP: Reuse/Release Equivalence Principle

> **"The granule of reuse is the granule of release."**

Classes grouped into a component should be **releasable together**. They should share a common theme or purpose.

```
GOOD Component:                 BAD Component:
┌─────────────────────┐        ┌─────────────────────┐
│   StringUtils       │        │   MiscUtils         │
│   ├─ StringBuilder  │        │   ├─ DateParser     │
│   ├─ StringParser   │        │   ├─ ImageResizer   │
│   └─ StringFormatter│        │   └─ AudioEncoder   │
└─────────────────────┘        └─────────────────────┘
  All related to strings!        Random hodgepodge!
```

### 2. CCP: Common Closure Principle

> **"Gather together those classes that change for the same reasons and at the same times."**

This is SRP for components. If code must change, let it all be in one component.

```
When "payment processing" changes:

GOOD:                           BAD:
┌─────────────────────┐        ┌─────────────────────┐
│  PaymentComponent   │        │  Various Components │
│  ├─ PaymentGateway  │        │  ├─ Component A     │
│  ├─ PaymentValidator│ ←      │  │    └─ Gateway    │
│  └─ PaymentReceipt  │  All   │  ├─ Component B     │
└─────────────────────┘  here! │  │    └─ Validator  │
                               │  └─ Component C     │
                               │       └─ Receipt    │
                               └─────────────────────┘
                                  Scattered!
```

### 3. CRP: Common Reuse Principle

> **"Don't force users of a component to depend on things they don't need."**

This is ISP for components. Classes that aren't tightly coupled shouldn't be in the same component.

```
┌─────────────────────┐          ┌─────────────────────┐
│    Component A      │          │    Component A      │
│   ┌───────────┐     │   uses   │   ┌───────────┐     │
│   │  Class X  │ ◄───┼──────────┼───│  User     │     │
│   └───────────┘     │          │   └───────────┘     │
│   ┌───────────┐     │          └─────────────────────┘
│   │  Class Y  │     │          
│   └───────────┘     │ If Y changes, User must redeploy
└─────────────────────┘ even though it only uses X!
```

### The Tension Triangle

These three principles are in tension:

```
                        REP
                     (Reusers)
                        /\
                       /  \
                      /    \
   Too many          /      \        Too many
   unneeded         /        \       components
   releases        /          \      affected by
                  /            \     changes
                 /              \
                /________________\
              CCP              CRP
          (Maintainers)    (Minimizers)
```

**Where to be on the triangle depends on project maturity:**
- Early project: Lean toward CCP (development speed)
- Mature project: Lean toward REP (reusability)

---

## Chapter 14: Component Coupling

Three principles guide relationships between components:

### 1. ADP: Acyclic Dependencies Principle

> **"Allow no cycles in the component dependency graph."**

### The Problem: Dependency Cycles

```
GOOD (Acyclic - DAG):            BAD (Has Cycle):
                                 
┌─────────┐                      ┌─────────┐
│  Main   │                      │  Main   │
└────┬────┘                      └────┬────┘
     │                                │
     ▼                                ▼
┌─────────┐                      ┌─────────┐
│Presenter│                      │Presenter│
└────┬────┘                      └────┬────┘
     │                                │
     ▼                                ▼
┌─────────┐                      ┌─────────┐
│Interactor│                     │Interactor│───┐
└────┬────┘                      └────┬────┘    │
     │                                │         │
     ▼                                ▼         │
┌─────────┐                      ┌─────────┐    │
│Entities │                      │Entities │◄───┘
└─────────┘                      └─────────┘
                                 
                                 CYCLE! Entities depends
                                 on Interactor which
                                 depends on Entities!
```

**Why cycles are bad:**
- "Morning after syndrome" - your code breaks due to others' changes
- Can't build or test components in isolation
- Can't determine build order

### Breaking Cycles

**Method 1: Dependency Inversion (add interface)**

```
BEFORE:                          AFTER:
┌─────────┐                      ┌─────────┐
│Entities │───────┐              │Entities │
└─────────┘       │              └────┬────┘
     ▲            │                   │
     │            │                   ▼
┌────┴────┐       │              ┌─────────┐
│Authorizer│◄─────┘              │«IPerms» │ ◄── New interface
└─────────┘                      └────┬────┘
                                      ▲
                                      │ implements
                                 ┌────┴────┐
                                 │Authorizer│
                                 └─────────┘
```

**Method 2: Create new component**

```
BEFORE:                          AFTER:
┌─────────┐    ┌─────────┐      ┌─────────┐    ┌─────────┐
│Entities │◄───│Authorizer│      │Entities │    │Authorizer│
└────┬────┘    └────┬────┘      └────┬────┘    └────┬────┘
     │              ▲                │              │
     └──────────────┘                ▼              ▼
         CYCLE!                  ┌─────────────────────┐
                                 │   Permissions       │ ◄── New!
                                 │  (shared classes)   │
                                 └─────────────────────┘
```

### 2. SDP: Stable Dependencies Principle

> **"Depend in the direction of stability."**

### What is Stability?

Stability = How hard something is to change (not frequency of change)

```
STABLE component X:              UNSTABLE component Y:
(hard to change)                 (easy to change)

    A ───→                           ┌───→ A
    B ───→  ┌───┐                    │
    C ───→  │ X │                ┌───┤ Y ├───→ B
            └───┘                │   └───┘
                                 │         ───→ C
Many dependents                  │
No dependencies                  No dependents
                                 Many dependencies
```

### Stability Metrics

```
Fan-in  = Incoming dependencies (classes depending on this component)
Fan-out = Outgoing dependencies (classes this component depends on)

I (Instability) = Fan-out / (Fan-in + Fan-out)

I = 0: Maximally stable (many dependents, no dependencies)
I = 1: Maximally unstable (no dependents, many dependencies)
```

**Rule:** I should decrease as you follow dependencies.

### 3. SAP: Stable Abstractions Principle

> **"A component should be as abstract as it is stable."**

### The Relationship

```
Abstractness (A) = Abstract classes / Total classes

Stable components (low I) → Should be abstract (high A)
Unstable components (high I) → Should be concrete (low A)
```

### The Main Sequence

Plot components on A vs I graph:

```
        1 ┌─────────────────────────────┐
          │ Zone of          ╲          │
    A     │ Uselessness       ╲         │
          │ (abstract but      ╲        │
    b     │  no one uses it)    ╲       │
    s     │                      ╲      │
    t     │                       ╲     │
    r     │                        ╲    │  ← Main Sequence
    a     │                         ╲   │    (ideal line)
    c     │                          ╲  │
    t     │                           ╲ │
    n     │                            ╲│
    e     │                             │
    s     │          Zone of           ╲│
    s     │          Pain               │
          │       (concrete but        ╲│
        0 │        many dependents)     │
          └─────────────────────────────┘
          0      Instability (I)        1
```

**Zones to Avoid:**
- **Zone of Pain (0,0):** Stable but concrete - hard to extend
- **Zone of Uselessness (1,1):** Abstract but no one depends on it

**Goal:** Keep components on or near the Main Sequence!

---

## Component Principles Summary

```
┌────────────────────────────────────────────────────────────────────┐
│  COHESION (what goes inside)                                       │
├────────────┬───────────────────────────────────────────────────────┤
│    REP     │ Group classes releasable together                     │
├────────────┼───────────────────────────────────────────────────────┤
│    CCP     │ Group classes that change together                    │
├────────────┼───────────────────────────────────────────────────────┤
│    CRP     │ Don't group classes that aren't used together         │
├────────────┴───────────────────────────────────────────────────────┤
│  COUPLING (relationships between)                                  │
├────────────┬───────────────────────────────────────────────────────┤
│    ADP     │ No cycles in dependency graph                         │
├────────────┼───────────────────────────────────────────────────────┤
│    SDP     │ Depend toward stability                               │
├────────────┼───────────────────────────────────────────────────────┤
│    SAP     │ Stable components should be abstract                  │
└────────────┴───────────────────────────────────────────────────────┘
```

---

*Notes created from "Clean Architecture" by Robert C. Martin*
