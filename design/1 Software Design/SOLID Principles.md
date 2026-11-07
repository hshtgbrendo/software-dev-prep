---
tags: [design, software-design, solid]
aliases: [SOLID]
---
# SOLID Principles

> [!summary]
> Five principles (popularized by Robert C. "Uncle Bob" Martin) for writing classes/modules that are **easy to change**. Interviewers love asking "explain SOLID" or "which SOLID principle does this violate?" Know each with a **violation → fix** example.

Back to [[Design Index]] · Prev: [[OOP Fundamentals]]

| Letter | Principle | One-liner |
|---|---|---|
| **S** | Single Responsibility | A class should have one reason to change |
| **O** | Open/Closed | Open for extension, closed for modification |
| **L** | Liskov Substitution | Subtypes must be usable anywhere their base type is |
| **I** | Interface Segregation | Don't force clients to depend on methods they don't use |
| **D** | Dependency Inversion | Depend on abstractions, not concretions |

---

## S — Single Responsibility Principle (SRP)

**"A module should have one, and only one, reason to change"** (i.e., it serves one actor/stakeholder).

```javascript
// ❌ Violation: business logic, persistence, and formatting in one class
class Invoice {
  calculateTotal() { /* tax rules */ }
  saveToDatabase() { /* SQL */ }
  printAsPDF()     { /* layout */ }
}
// Finance changes tax rules → edit Invoice. DBA changes schema → edit Invoice. Design changes PDF → edit Invoice.

// ✅ Fix: split by reason-to-change
class Invoice           { calculateTotal() {} }
class InvoiceRepository { save(invoice) {} }
class InvoicePdfRenderer{ render(invoice) {} }
```

> [!warning] Common misread
> SRP ≠ "a class should do one thing / have one method." It's about **one axis of change**. Splitting into dozens of one-method classes is over-engineering.

---

## O — Open/Closed Principle (OCP)

**Software entities should be open for extension but closed for modification.** Add new behavior by adding new code, not editing tested code.

```javascript
// ❌ Every new shape = edit this function
function area(shape) {
  switch (shape.type) {
    case "circle": return Math.PI * shape.r ** 2;
    case "square": return shape.side ** 2;
    // add "triangle" here... and in perimeter(), and in draw()...
  }
}

// ✅ Polymorphism: new shapes extend; existing code untouched
class Circle { constructor(r){this.r=r} area(){ return Math.PI * this.r ** 2; } }
class Square { constructor(s){this.s=s} area(){ return this.s ** 2; } }
const totalArea = shapes => shapes.reduce((a, s) => a + s.area(), 0);
```

Patterns that implement OCP: [[Behavioral Patterns#Strategy|Strategy]], [[Structural Patterns#Decorator|Decorator]], [[Behavioral Patterns#Observer|Observer]], plugins.

---

## L — Liskov Substitution Principle (LSP)

**If S is a subtype of T, objects of T can be replaced with objects of S without breaking the program.** Subclasses must honor the parent's contract: no stronger preconditions, no weaker postconditions, no surprising exceptions.

Classic violation, **Square extends Rectangle**:

```javascript
class Rectangle {
  setWidth(w)  { this.w = w; }
  setHeight(h) { this.h = h; }
  area() { return this.w * this.h; }
}
class Square extends Rectangle {
  setWidth(w)  { this.w = w; this.h = w; }   // surprise!
  setHeight(h) { this.w = h; this.h = h; }
}

function resize(rect) {
  rect.setWidth(5);
  rect.setHeight(4);
  console.assert(rect.area() === 20);  // fails for Square (16)
}
```

Another: `Bird.fly()` and `Penguin extends Bird` that throws `CannotFlyError`. Fix by modeling capabilities separately (`FlyingBird`, or a `Flyable` interface).

> [!tip] Smell
> A subclass that overrides a method to throw `NotImplemented`/`UnsupportedOperation`, or code that does `if (x instanceof Penguin)`, usually means LSP is broken.

---

## I — Interface Segregation Principle (ISP)

**Clients should not be forced to depend on interfaces they don't use.** Prefer several small, role-specific interfaces over one fat one.

```typescript
// ❌ Fat interface
interface Machine { print(): void; scan(): void; fax(): void; }
class BasicPrinter implements Machine {
  print() {}
  scan() { throw new Error("no scanner"); }   // forced to implement, also breaks LSP
  fax()  { throw new Error("no fax"); }
}

// ✅ Segregated
interface Printer { print(): void; }
interface Scanner { scan(): void; }
class BasicPrinter implements Printer { print() {} }
class OfficeMachine implements Printer, Scanner { print() {} scan() {} }
```

The same idea applies at the API/service level: don't make a mobile client call a 50-field endpoint when it needs 3 fields (motivation for [[API Design#GraphQL|GraphQL]] and BFFs).

---

## D — Dependency Inversion Principle (DIP)

1. **High-level modules should not depend on low-level modules. Both should depend on abstractions.**
2. **Abstractions should not depend on details. Details depend on abstractions.**

```javascript
// ❌ High-level policy hard-wired to a low-level detail
class NotificationService {
  constructor() { this.sender = new TwilioSmsClient(); }
  notify(user, msg) { this.sender.sendSms(user.phone, msg); }
}

// ✅ Both depend on an abstraction ("something with send(to, msg)")
class NotificationService {
  constructor(sender) { this.sender = sender; }      // injected
  notify(user, msg) { this.sender.send(user, msg); }
}
const svc = new NotificationService(new TwilioSender());   // prod
const testSvc = new NotificationService({ send: jest.fn() }); // test
```

- **DIP** is the principle; **Dependency Injection** is a technique to achieve it; **IoC containers** (Spring, .NET DI, NestJS) automate the wiring.
- This is the basis of [[Architectural Styles#Hexagonal / Ports & Adapters|Hexagonal]] and [[Architectural Styles#Clean Architecture|Clean Architecture]]: business logic defines interfaces ("ports"); infrastructure implements them ("adapters").

---

## Putting it together: spotting violations

| Code smell | Likely violated |
|---|---|
| God class with 2,000 lines touching DB, HTTP, email | SRP |
| Giant `switch(type)` that grows every sprint | OCP |
| Subclass throws `NotSupported` / `instanceof` checks | LSP (and often ISP) |
| Implementations with empty/throwing methods | ISP |
| `new ConcreteThing()` deep inside business logic; can't unit test | DIP |

> [!quote] Pragmatism
> SOLID is guidance, not law. In interviews, show you know it **and** that you'd avoid over-abstracting a 50-line script. "I'd introduce an interface here because we already have two payment providers and expect more" beats "I always add interfaces."

## Self-test
- [ ] Give a real-world example of SRP violation from a web app (e.g., Express route handler doing validation + SQL + email).
- [ ] Explain why Square/Rectangle violates LSP.
- [ ] What's the difference between DIP and dependency injection?
- [ ] Which design patterns help achieve OCP?

Next: [[Design Principles]]
