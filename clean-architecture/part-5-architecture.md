# Clean Architecture Notes
## A Craftsman's Guide to Software Structure and Design
### By Robert C. Martin (Uncle Bob)

---

# Part V: Architecture

## Chapter 15: What is Architecture?

### The Definition

> **"Architecture is the shape given to a system by those who build it."**

- Division into components
- Arrangement of components
- Communication between components

### The Purpose

> **"The goal is to minimize human resources required to build and maintain the system."**

**Primary purpose:** Support the life cycle of the system
- Easy to understand
- Easy to develop
- Easy to maintain  
- Easy to deploy

### The Strategy

> **"Leave as many options open as possible, for as long as possible."**

### Architecture Must Support:

```
┌─────────────────────────────────────────────────────────────────┐
│                    GOOD ARCHITECTURE                            │
├────────────────┬────────────────────────────────────────────────┤
│  Development   │ Easy for teams to work independently           │
├────────────────┼────────────────────────────────────────────────┤
│  Deployment    │ System can be deployed with a single action    │
├────────────────┼────────────────────────────────────────────────┤
│  Operation     │ Reveals operational needs to developers        │
├────────────────┼────────────────────────────────────────────────┤
│  Maintenance   │ Easy to find where to make changes (low risk)  │
└────────────────┴────────────────────────────────────────────────┘
```

### Policy vs Details

Every system can be decomposed into:

```
┌─────────────────────────────────────────┐
│              POLICY                      │
│                                          │
│   Business rules and procedures          │
│   The TRUE VALUE of the system           │
│                                          │
│   Example: "Calculate interest at 5%"    │
└─────────────────────────────────────────┘
                    │
    Policy should NOT know about details!
                    │
┌─────────────────────────────────────────┐
│              DETAILS                     │
│                                          │
│   IO devices, databases, web systems,    │
│   frameworks, protocols                  │
│                                          │
│   Example: MySQL, REST API, React        │
└─────────────────────────────────────────┘
```

### Decisions That Can Be Delayed

| Detail | When to Decide |
|--------|---------------|
| Database type | Not early - high-level policy shouldn't care if it's SQL/NoSQL/files |
| Web server | Not early - policy shouldn't know it's being delivered over web |
| REST vs SOAP | Not early - policy should be agnostic about interface |
| Dependency injection framework | Not early - policy shouldn't care how dependencies are resolved |

> **"A good architect maximizes the number of decisions NOT made."**

---

## Chapter 16: Independence

### Decoupling Layers (Horizontal)

Separate things that change for different reasons:

```
┌─────────────────────────────────────────────────────────────────┐
│                         UI Layer                                │
│   Changes for: User experience reasons                          │
├─────────────────────────────────────────────────────────────────┤
│                Application Business Rules                       │
│   Changes for: Application-specific policy changes              │
├─────────────────────────────────────────────────────────────────┤
│                Domain Business Rules                            │
│   Changes for: Core business policy changes                     │
├─────────────────────────────────────────────────────────────────┤
│                      Database Layer                             │
│   Changes for: Data storage/query reasons                       │
└─────────────────────────────────────────────────────────────────┘
```

### Decoupling Use Cases (Vertical)

Use cases are narrow vertical slices:

```
        UI Layer
    ┌────┬────┬────┐
    │    │    │    │
    │ UC │ UC │ UC │  ← Each use case has its own
    │ 1  │ 2  │ 3  │    UI, business rules, and
    │    │    │    │    database components
    ├────┼────┼────┤
    │    │    │    │
    │ UC │ UC │ UC │
    │ 1  │ 2  │ 3  │
    │    │    │    │
    ├────┼────┼────┤
    │    │    │    │
    │ UC │ UC │ UC │
    │ 1  │ 2  │ 3  │
    └────┴────┴────┘
      Database Layer
```

### Decoupling Modes

Three levels of decoupling:

```
┌────────────────────────────────────────────────────────────────┐
│ Source Level                                                    │
│                                                                 │
│ • Control dependencies between source modules                   │
│ • Components execute in same address space                      │
│ • Communication via function calls                              │
│ • Example: Monolithic application                               │
├────────────────────────────────────────────────────────────────┤
│ Deployment Level (Binary/JAR/DLL)                               │
│                                                                 │
│ • Control dependencies between deployable units                 │
│ • May share address space or use IPC                            │
│ • Example: Plugins, dynamically loaded libraries                │
├────────────────────────────────────────────────────────────────┤
│ Service Level                                                   │
│                                                                 │
│ • Communicate only through network packets                      │
│ • Completely independent of source/binary changes               │
│ • Example: Microservices                                        │
└────────────────────────────────────────────────────────────────┘
```

**Best approach:** Start with source-level decoupling, graduate to services only when needed.

### Beware False Duplication!

```
Two screens look similar...

Screen A (User Profile)     Screen B (Admin Panel)
┌─────────────────────┐    ┌─────────────────────┐
│  Name: ________     │    │  Name: ________     │
│  Email: _______     │    │  Email: _______     │
│  [Save]             │    │  [Save]             │
└─────────────────────┘    └─────────────────────┘

Temptation: Share the code!

BUT... in a year:

Screen A (User Profile)     Screen B (Admin Panel)  
┌─────────────────────┐    ┌─────────────────────────────┐
│  Name: ________     │    │  Name: ________ Status: ___  │
│  Email: _______     │    │  Email: _______ Role: ______ │
│  [Save]             │    │  Permissions: [___________]  │
└─────────────────────┘    │  Audit Log: [View]           │
                           │  [Save] [Deactivate] [Delete]│
                           └─────────────────────────────┘

They evolved differently! Separating them now is painful.
```

**Rule:** Only unify TRUE duplication (changes together for same reasons)

---

## Chapter 17-22: Boundaries and Clean Architecture

### Drawing Boundaries

Boundaries separate components that change for different reasons:

```
┌──────────────────────────────────────────────────────────────┐
│                                                              │
│  ┌─────────────┐        BOUNDARY        ┌─────────────┐     │
│  │  Business   │ ◄─────────────────────│     GUI     │     │
│  │   Rules     │                        └─────────────┘     │
│  │             │ ◄───────────────────── ┌─────────────┐     │
│  │             │        BOUNDARY        │  Database   │     │
│  └─────────────┘                        └─────────────┘     │
│                                                              │
│  The business rules don't know about GUI or Database!        │
│  GUI and Database are PLUGINS to the business rules.         │
└──────────────────────────────────────────────────────────────┘
```

### The Clean Architecture

The famous "onion" architecture:

```
                    ┌───────────────────────────────────────┐
                    │           External Interfaces         │
                    │  (Web, Devices, DB, External APIs)    │
                    │  ┌───────────────────────────────────┐│
                    │  │      Interface Adapters           ││
                    │  │  (Controllers, Presenters,        ││
                    │  │   Gateways)                       ││
                    │  │  ┌───────────────────────────────┐││
                    │  │  │     Application Business      │││
                    │  │  │        Rules (Use Cases)      │││
                    │  │  │  ┌───────────────────────────┐│││
                    │  │  │  │    Enterprise Business    ││││
                    │  │  │  │    Rules (Entities)       ││││
                    │  │  │  │                           ││││
                    │  │  │  └───────────────────────────┘│││
                    │  │  └───────────────────────────────┘││
                    │  └───────────────────────────────────┘│
                    └───────────────────────────────────────┘
                    
    ════════════════════════════════════════════════════════
              ALL DEPENDENCIES POINT INWARD →
    ════════════════════════════════════════════════════════
```

### The Dependency Rule

> **"Source code dependencies must point only inward, toward higher-level policies."**

**Nothing in an inner circle can know about anything in an outer circle.**

### Crossing Boundaries

When control flows outward but dependencies must point inward, use interfaces:

```
┌─────────────────────────────────────────────────────────────────┐
│ USE CASE                           │ INTERFACE ADAPTER          │
│                                    │                            │
│  ┌──────────────────┐              │                            │
│  │   Use Case       │              │   ┌──────────────────┐     │
│  │   Interactor     │──────────────│──▶│    Presenter     │     │
│  └────────┬─────────┘              │   └──────────────────┘     │
│           │                        │                            │
│           │ implements             │                            │
│           ▼                        │                            │
│  ┌──────────────────┐              │                            │
│  │ «Output Port»    │◀─────────────│───────── (implements)      │
│  │   Interface      │              │                            │
│  └──────────────────┘              │                            │
│                                    │                            │
│  Dependencies point IN even though │                            │
│  control flows OUT!                │                            │
└─────────────────────────────────────────────────────────────────┘
```

### Typical Data Flow

```
HTTP Request
     │
     ▼
┌─────────────────┐
│   Controller    │  Converts HTTP to Input DTO
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Input Port     │  «interface»
│  (Use Case      │
│   Interface)    │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Use Case       │  Application business rules
│  Interactor     │  
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│   Entities      │  Enterprise business rules
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Output Port    │  «interface»
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│   Presenter     │  Converts to View Model
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│     View        │  Renders response
└─────────────────┘
```

---

## Screaming Architecture

Your architecture should "scream" what the system does:

```
BAD - Architecture screams "Rails app!":     GOOD - Architecture screams "Healthcare system!":

├── app/                                     ├── patients/
│   ├── controllers/                         │   ├── admission/
│   ├── models/                              │   ├── discharge/
│   ├── views/                               │   └── transfer/
│   └── helpers/                             ├── billing/
├── config/                                  │   ├── insurance_claims/
├── db/                                      │   └── patient_invoicing/
└── lib/                                     ├── scheduling/
                                             │   ├── appointments/
                                             │   └── operating_rooms/
                                             └── medical_records/
```

**When you look at the top-level structure, you should immediately know what the system DOES, not what framework it uses.**

---

## Part V Summary: Architecture Checklist

```
✓ Does the architecture support the use cases?
  (Are they visible in the structure?)

✓ Does it support development by multiple teams?
  (Can teams work independently?)

✓ Does it support deployment?
  (Can we deploy easily and frequently?)

✓ Does it support operation?
  (Can we scale and monitor?)

✓ Does it defer decisions?
  (Are databases/frameworks/UI pluggable?)

✓ Does it follow the Dependency Rule?
  (All dependencies point inward?)

✓ Does it scream the domain?
  (Can you tell what it does from the structure?)
```

---

# Quick Reference Card

## The Golden Rules

| Principle | Remember This |
|-----------|---------------|
| **SRP** | One module, one actor, one reason to change |
| **OCP** | Add new code, don't modify existing code |
| **LSP** | Subtypes must honor the base type's contract |
| **ISP** | Small, specific interfaces > Large, general ones |
| **DIP** | High-level modules shouldn't depend on low-level modules |
| **ADP** | No cycles in your dependency graph |
| **SDP** | Depend on stable things |
| **SAP** | Stable things should be abstract |

## The Clean Architecture Mantra

> **"The center is your business logic. Everything else is a detail. Details depend on the center, never the reverse."**

## The Ultimate Goal

> **"Good architecture makes the system easy to understand, develop, maintain, and deploy. The strategy is to leave options open as long as possible."**

---

*Notes created from "Clean Architecture" by Robert C. Martin*
