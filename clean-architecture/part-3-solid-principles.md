# Clean Architecture Notes
## A Craftsman's Guide to Software Structure and Design
### By Robert C. Martin (Uncle Bob)

---

# Part III: SOLID Design Principles

## Overview

The SOLID principles tell us how to arrange functions and data structures into classes, and how those classes should be interconnected.

```
S - Single Responsibility Principle
O - Open-Closed Principle
L - Liskov Substitution Principle
I - Interface Segregation Principle
D - Dependency Inversion Principle
```

**Goal:** Create mid-level software structures that:
- Tolerate change
- Are easy to understand
- Can be reused across systems

---

## Chapter 7: SRP - Single Responsibility Principle

### Common Misconception

❌ "A module should do one thing" (This is about functions, not SRP)

✅ SRP is about **who** we are serving, not **what** the code does

### The Real Definition

> **"A module should have one, and only one, reason to change"**

Or better: **"A module should be responsible to one, and only one, actor."**

**What is an "Actor"?** An actor is a group of people (stakeholders) who want the system to change in a particular way. Examples:
- CFO and accounting team → want payroll calculations
- COO and HR team → want hour tracking reports  
- CTO and tech team → want data persistence

### The Problem: Accidental Coupling

**Example:** An Employee class serving multiple departments:

```
                    ┌───────────────────────────────────────┐
    CFO ──────────→ │            Employee                   │ ←──────── COO
   (Accounting)     │                                       │           (HR)
                    │  + calculatePay()    ──┐              │
                    │  + reportHours()     ──┼──→ BOTH call │
                    │  + save()              │    regularHours()
                    │                        │              │
                    │  - regularHours() ◄────┘  (private)   │
    CTO ──────────→ │                                       │
   (Tech)           └───────────────────────────────────────┘
```

**The Hidden Shared Method Problem:**

```java
public class Employee {
    
    // Called by ACCOUNTING (CFO's team)
    public Money calculatePay() {
        int hours = regularHours();  // ← Uses shared method
        return hours * payRate;
    }
    
    // Called by HR (COO's team)  
    public Hours reportHours() {
        int hours = regularHours();  // ← Uses SAME shared method
        return new Hours(hours);
    }
    
    // DANGER: This private method serves TWO different actors!
    private int regularHours() {
        // Algorithm to calculate non-overtime hours
        // What if CFO and COO need DIFFERENT calculations?
    }
    
    // Called by TECH (CTO's team)
    public void save() {
        database.save(this);
    }
}
```

**What Goes Wrong - A Realistic Scenario:**

```
Step 1: CFO asks: "Change how we calculate regular hours for payroll"
        (They want to exclude lunch breaks from pay calculation)
        
Step 2: Developer modifies regularHours() to exclude lunch breaks
        
Step 3: calculatePay() now works correctly for CFO ✓

Step 4: BUT... reportHours() also uses regularHours()!
        COO's HR reports now show WRONG hours! 💥
        
        HR needed lunch breaks INCLUDED for compliance reporting!
```

**Why This Happened:**
- One class had TWO reasons to change (two actors)
- A change requested by Actor A (CFO) broke functionality for Actor B (COO)
- The developers didn't realize the shared dependency

### The Solution: Separate by Actor

**Option 1: Separate Classes with Shared Data**

Each actor gets their own class with their own logic:

```
┌─────────────────────┐   ┌─────────────────────┐   ┌─────────────────────┐
│   PayCalculator     │   │    HourReporter     │   │   EmployeeSaver     │
│   (for CFO)         │   │    (for COO)        │   │   (for CTO)         │
│                     │   │                     │   │                     │
│ + calculatePay()    │   │ + reportHours()     │   │ + save()            │
│ - calcRegularHours()│   │ - calcRegularHours()│   │                     │
│   (own algorithm)   │   │   (own algorithm)   │   │                     │
└──────────┬──────────┘   └──────────┬──────────┘   └──────────┬──────────┘
           │                         │                         │
           └─────────────────────────┼─────────────────────────┘
                                     │
                                     ▼
                          ┌─────────────────────┐
                          │    EmployeeData     │
                          │    (shared data)    │
                          │                     │
                          │  - name             │
                          │  - hoursWorked[]    │
                          │  - payRate          │
                          └─────────────────────┘

Now CFO's changes to PayCalculator can't break COO's HourReporter!
```

**Option 2: Facade Pattern** (when you need a single entry point)

```
┌─────────────────────────────────────┐
│          EmployeeFacade             │
│     (simple delegating class)       │
│                                     │
│  + calculatePay() → payCalc.calc()  │
│  + reportHours() → hourRpt.report() │
│  + save() → saver.save()            │
└──────────────────┬──────────────────┘
                   │
        ┌──────────┼──────────┐
        ▼          ▼          ▼
   ┌─────────┐ ┌─────────┐ ┌─────────┐
   │  Pay    │ │  Hour   │ │Employee │
   │  Calc   │ │ Reporter│ │  Saver  │
   └─────────┘ └─────────┘ └─────────┘
   
The Facade has no logic - just delegates.
Each specialist class has ONE actor.
```

### SRP Symptoms Checklist

Watch for these warning signs:

| Symptom | What It Means |
|---------|---------------|
| Class has methods used by different departments | Multiple actors |
| "When I change X, Y breaks" | Accidental coupling |
| Merge conflicts between teams in same file | Competing actors |
| Class name includes "And" or "Manager" | Probably doing too much |
| Hard to name the class clearly | Unclear single purpose |

### Key Takeaway

> **SRP is NOT about code doing one thing. It's about code serving ONE STAKEHOLDER GROUP.**

A class can have many methods, as long as they all serve the same actor and change for the same reasons.

---

## Chapter 8: OCP - Open-Closed Principle

### The Definition

> **"Software entities should be open for extension, but closed for modification."**

You should be able to add new behavior without changing existing code.

### Example: Adding a New Report Format

**Bad approach:** Modify existing code every time you need a new format

**Good approach:** Use proper component organization

```
                          ┌──────────────────┐
                          │   «Interactor»   │
                          │  (Business Rules)│
                          └────────┬─────────┘
                                   │
           Protected from          │        Changes flow
           changes below           ▼        INWARD
                          ┌──────────────────┐
               ┌──────────│   «Controller»   │──────────┐
               │          └──────────────────┘          │
               ▼                                        ▼
    ┌──────────────────┐                     ┌──────────────────┐
    │ «Web Presenter»  │                     │«Print Presenter» │
    └────────┬─────────┘                     └────────┬─────────┘
             │                                        │
             ▼                                        ▼
    ┌──────────────────┐                     ┌──────────────────┐
    │   «Web View»     │                     │  «Print View»    │
    └──────────────────┘                     └──────────────────┘
    
    Adding PDF Presenter? Just add new component - don't modify existing!
```

### The Hierarchy of Protection

```
Most Protected (highest level)
         ↑
    ┌────────────────┐
    │   Interactor   │  ← Contains business rules
    └────────────────┘
         ↑
    ┌────────────────┐
    │   Controller   │
    └────────────────┘
         ↑
    ┌────────────────┐
    │   Presenters   │
    └────────────────┘
         ↑
    ┌────────────────┐
    │     Views      │  ← Most likely to change
    └────────────────┘
         ↓
Least Protected (lowest level)
```

**Rule:** Dependencies point toward more stable, higher-level components.

---

## Chapter 9: LSP - Liskov Substitution Principle

### The Definition

> **"Objects of a supertype should be replaceable with objects of its subtypes without breaking the program."**

### The Classic Violation: Square/Rectangle

```
┌─────────────────┐
│   Rectangle     │
│                 │
│  setWidth(w)    │
│  setHeight(h)   │
│  area()         │
└────────┬────────┘
         △
         │
┌────────┴────────┐
│     Square      │  ← PROBLEM!
└─────────────────┘

// This code works for Rectangle but FAILS for Square:
Rectangle r = getShape();  // might return Square
r.setWidth(5);
r.setHeight(2);
assert(r.area() == 10);    // FAILS if r is Square!
```

**Why it violates LSP:**
- Rectangle: width and height are independent
- Square: width and height must be equal
- Square cannot substitute for Rectangle without breaking expectations

### LSP in Architecture: Real World Example

**Scenario:** Taxi dispatch aggregator with multiple taxi companies

```
Expected REST interface:
/driver/Bob/pickupAddress/24 Maple St./pickupTime/153/destination/ORD

Company A follows spec ✓
Company B follows spec ✓
Company C uses "dest" instead of "destination" ✗
```

**The Violation:** Company C isn't substitutable!

**Result:** Architecture gets polluted with special cases:

```java
// BAD - LSP violation forces ugly workarounds
if (driver.getDispatchUri().startsWith("acme.com")) {
    // Special handling for non-conforming company
}
```

**Lesson:** LSP applies to interfaces at all levels, not just classes!

---

## Chapter 10: ISP - Interface Segregation Principle

### The Definition

> **"Clients should not be forced to depend on interfaces they don't use."**

### The Problem: Fat Interfaces

```
                    ┌────────────────────┐
    User1 ────────→ │        OPS         │ ←──────── User2
   (uses op1)       │                    │         (uses op2)
                    │  op1()             │
                    │  op2()             │
    User3 ────────→ │  op3()             │
   (uses op3)       └────────────────────┘

Problem: Change to op2() forces User1 and User3 to recompile!
```

### The Solution: Segregated Interfaces

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   User1     │     │    User2    │     │   User3     │
└──────┬──────┘     └──────┬──────┘     └──────┬──────┘
       │                   │                   │
       ▼                   ▼                   ▼
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  «U1Ops»    │     │  «U2Ops»    │     │  «U3Ops»    │
│   op1()     │     │   op2()     │     │   op3()     │
└──────┬──────┘     └──────┬──────┘     └──────┬──────┘
       │                   │                   │
       └───────────────────┼───────────────────┘
                           │
                    ┌──────┴──────┐
                    │     OPS     │
                    │             │
                    │  op1()      │
                    │  op2()      │
                    │  op3()      │
                    └─────────────┘
```

Now changes to op2() only affect User2!

### ISP at the Architecture Level

```
┌─────────────┐         ┌─────────────┐         ┌─────────────┐
│   System    │ ──────→ │  Framework  │ ──────→ │  Database   │
│      S      │         │      F      │         │      D      │
└─────────────┘         └─────────────┘         └─────────────┘

If D has features F doesn't use, and D changes...
F may need redeploying, which means S needs redeploying!

LESSON: Don't depend on things you don't need!
```

---

## Chapter 11: DIP - Dependency Inversion Principle

### The Definition

> **"Depend on abstractions, not concretions."**

High-level modules should not depend on low-level modules. Both should depend on abstractions.

### The Coding Practices

| Do | Don't |
|----|-------|
| Refer to abstract interfaces | Refer to volatile concrete classes |
| Derive from abstract classes | Derive from volatile concrete classes |
| Create abstract functions | Override concrete functions |
| Mention abstract names | Mention concrete, volatile names |

### The Factory Pattern Solution

**Problem:** Creating objects requires knowing concrete classes

**Solution:** Use Abstract Factory

```
┌──────────────────────────────────────────────────────────────┐
│                    ABSTRACT SIDE                             │
│  ┌──────────────┐       ┌───────────────────┐                │
│  │ Application  │ ────→ │ «interface»       │                │
│  │              │       │   Service         │                │
│  │              │       └───────────────────┘                │
│  │              │                △                           │
│  │              │                │                           │
│  │              │       ┌───────────────────┐                │
│  │              │ ────→ │ «interface»       │                │
│  └──────────────┘       │ ServiceFactory    │                │
│                         │                   │                │
│                         │ makeSvc()         │                │
│                         └───────────────────┘                │
├──────────────────────────────────────────────────────────────┤
│                    CONCRETE SIDE                             │
│                                 △                            │
│                                 │ implements                 │
│                   ┌─────────────┴─────────────┐              │
│                   │  ServiceFactoryImpl       │              │
│                   │                           │              │
│                   │  makeSvc() {              │              │
│                   │    return new ConcreteImpl│              │
│                   │  }                        │              │
│                   └───────────────────────────┘              │
│                                 │                            │
│                                 │ creates                    │
│                                 ▼                            │
│                   ┌─────────────────────────┐                │
│                   │     ConcreteImpl        │                │
│                   └─────────────────────────┘                │
└──────────────────────────────────────────────────────────────┘

All source code dependencies cross the boundary pointing UP (toward abstractions)
```

### The Architectural Boundary

The curved line in DIP diagrams represents an **architectural boundary**:
- All dependencies point toward the abstract side
- Flow of control may go the opposite direction
- This is why it's called "inversion"

---

## SOLID Principles Summary

```
┌────────────────────────────────────────────────────────────────────┐
│ Principle │ One-Liner                                              │
├───────────┼────────────────────────────────────────────────────────┤
│    SRP    │ A module has one reason to change (one actor)          │
├───────────┼────────────────────────────────────────────────────────┤
│    OCP    │ Open for extension, closed for modification            │
├───────────┼────────────────────────────────────────────────────────┤
│    LSP    │ Subtypes must be substitutable for their base types    │
├───────────┼────────────────────────────────────────────────────────┤
│    ISP    │ Don't depend on things you don't use                   │
├───────────┼────────────────────────────────────────────────────────┤
│    DIP    │ Depend on abstractions, not concretions                │
└────────────────────────────────────────────────────────────────────┘
```

---

*Notes created from "Clean Architecture" by Robert C. Martin*
