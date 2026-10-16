---
tags: [design, software-design, principles]
aliases: [DRY, KISS, YAGNI, Clean Code]
---
# Design Principles (Beyond SOLID)

> [!summary]
> The everyday rules of thumb senior engineers use in code review. You'll hear these acronyms constantly at work. Know what each means **and when it's wrong to apply it**.

Back to [[Design Index]] · Prev: [[SOLID Principles]]

---

## The core acronyms

### DRY: Don't Repeat Yourself
"Every piece of knowledge must have a single, unambiguous, authoritative representation." (*The Pragmatic Programmer*)
- It's about **knowledge/business rules**, not identical-looking text. Two functions that happen to look similar but change for different reasons are *not* duplication.
- **Over-DRYing** creates wrong abstractions. See **AHA**, **WET**, and the **Rule of Three** below.

### WET / AHA / Rule of Three
- **WET** = "Write Everything Twice": tolerate duplication until a pattern is clear.
- **AHA** = "Avoid Hasty Abstractions" (Kent C. Dodds).
- **Rule of Three**: abstract on the *third* occurrence, not the second.
> "Duplication is far cheaper than the wrong abstraction." — Sandi Metz

### KISS: Keep It Simple, Stupid
Choose the simplest solution that works. Clever code is a maintenance cost. In system design interviews, **start simple** (single server + DB) and scale only when requirements demand it.

### YAGNI: You Aren't Gonna Need It
Don't build features or extension points for hypothetical futures. Every abstraction has a cost: more code, indirection, onboarding time.

> [!tip] Balancing YAGNI with OCP
> OCP says "design for extension"; YAGNI says "don't build what you don't need." Resolve it by adding extension points **where change is likely or already happening** (you have 2 payment providers today), not everywhere.

### Separation of Concerns (SoC)
Divide a program into distinct sections, each addressing one concern: UI vs business logic vs data access, or HTML/CSS/JS. Generalizes SRP to the whole system. MVC, layered architecture, and microservices are all SoC at different scales.

### Law of Demeter (Principle of Least Knowledge)
"Only talk to your immediate friends." A method should call methods on: itself, its params, objects it creates, its direct fields.
```javascript
// ❌ train wreck: knows the whole object graph
order.getCustomer().getAddress().getCity().getZip();
// ✅ ask, don't dig
order.shippingZip();
```

### Tell, Don't Ask
Tell objects what to do instead of pulling their data out and deciding for them.
```javascript
// ❌ ask
if (account.balance >= amt) account.balance -= amt;
// ✅ tell
account.withdraw(amt);  // the account enforces its own rules
```

### Composition over Inheritance
See [[OOP Fundamentals#Composition over inheritance]].

### Program to an interface, not an implementation
Depend on the contract (`PaymentProcessor`), not the concrete class (`StripeClient`). Core idea of [[SOLID Principles#D — Dependency Inversion Principle (DIP)|DIP]] and the GoF book.

### Principle of Least Astonishment (Surprise)
APIs, functions, and UIs should behave the way users expect. `getUser()` should not also send an email.

### Fail Fast
Detect errors as early as possible and loudly: validate inputs at boundaries, throw on invalid state, use assertions. Easier debugging, less corrupted data.

### Command–Query Separation (CQS)
A method should either **change state (command)** or **return data (query)**, not both. Scaled up to architecture, this becomes [[Architectural Styles#CQRS|CQRS]].

### Convention over Configuration
Frameworks (Rails, Next.js file routing, Spring Boot) assume sensible defaults so you write less config.

### Idempotency
Doing an operation twice has the same effect as doing it once. **Critical** in distributed systems and APIs. See [[API Design#Idempotency]] and [[Distributed Systems Patterns#Idempotency]].

---

## Clean Code essentials

What reviewers at big companies actually comment on:

- **Naming**: intention-revealing names. `elapsedTimeInDays` > `d`. Functions are verbs (`calculateTax`), booleans read as questions (`isActive`, `hasPermission`).
- **Small functions** that do one thing at one level of abstraction.
- **Few parameters**: more than 3 is a smell; use an options object.
- **No side effects hidden in names**: `checkPassword()` shouldn't start a session.
- **Avoid flag arguments**: `render(true)`. Split into `renderForPrint()` / `renderForScreen()`.
- **Early returns / guard clauses** instead of deep nesting.
- **Comments explain *why*, not *what***. Code should explain what.
- **Consistent error handling**: don't swallow exceptions; don't return `null` *and* throw in the same API.
- **Magic numbers → named constants**.
- **Boy Scout Rule**: leave code a little cleaner than you found it.

```javascript
// ❌ nested
function process(user) {
  if (user) {
    if (user.isActive) {
      if (user.orders.length > 0) { /* ... */ }
    }
  }
}
// ✅ guard clauses
function process(user) {
  if (!user || !user.isActive) return;
  if (user.orders.length === 0) return;
  /* ... */
}
```

---

## Code smells (Martin Fowler, *Refactoring*)

| Smell | Description | Typical refactoring |
|---|---|---|
| Long Method | Function does too much | Extract Function |
| Large Class / God Object | Class knows/does everything | Extract Class |
| Long Parameter List | 5+ params | Introduce Parameter Object |
| Duplicated Code | Same logic in many places | Extract Function / Pull Up Method |
| Feature Envy | Method uses another class's data more than its own | Move Method |
| Data Clumps | Same group of fields travel together (`lat, lng`) | Introduce Value Object |
| Primitive Obsession | Strings/ints for domain concepts (`"USD"`, cents as number) | Value Object (`Money`) |
| Switch Statements | Type-based switching repeated | Replace Conditional with Polymorphism |
| Shotgun Surgery | One change requires edits in many classes | Move/Inline to consolidate |
| Divergent Change | One class changes for many reasons | Split (SRP) |
| Speculative Generality | Unused abstractions "for later" | Remove (YAGNI) |
| Dead Code | Unreachable / unused | Delete it |

---

## Technical debt

- The implied cost of rework from choosing a quick solution now.
- **Not always bad**: deliberate, documented debt to hit a deadline is a valid business choice.
- Track it (tickets, `TODO(JIRA-123)`), pay it down incrementally, and explain it to PMs in terms of velocity/risk.

## Self-test
- [ ] When is DRY harmful?
- [ ] How do YAGNI and Open/Closed conflict, and how do you balance them?
- [ ] Name 5 code smells and their refactorings.
- [ ] Explain Law of Demeter with an example.

Next: [[Creational Patterns]]
