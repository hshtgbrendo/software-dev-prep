---
tags: [design, architecture, ddd]
aliases: [DDD, Bounded Context]
---
# Domain-Driven Design (DDD) Essentials

> [!summary]
> DDD (Eric Evans, 2003) is an approach to modeling complex business software around the **business domain**. You don't need to be an expert as an entry/mid engineer, but enterprise teams use its vocabulary constantly, especially **bounded context**, **aggregate**, **entity vs value object**, and **ubiquitous language**. It's also the main tool for **deciding microservice boundaries**.

Back to [[Design Index]] · Related: [[Architectural Styles]] · [[OOP Fundamentals]]

---

## Strategic DDD (the big picture)

### Ubiquitous Language
Engineers and domain experts use the **same terms**, in conversation *and* in code. If the business says "policyholder," the class isn't `Customer`.

### Domain & Subdomains
- **Core domain**: what makes the business special (Uber's matching/pricing). Invest the most here.
- **Supporting subdomain**: needed, specific to you, but not a differentiator.
- **Generic subdomain**: solved problems (auth, billing, email). **Buy or use off-the-shelf** (Auth0, Stripe).

### Bounded Context
An explicit boundary within which a model and its language are consistent. **The same word can mean different things in different contexts.**

> "Product" in the **Catalog** context = title, images, description.
> "Product" in the **Inventory** context = SKU, warehouse location, quantity.
> "Product" in the **Shipping** context = weight, dimensions.

Rather than one giant shared `Product` class, each context has its own model. Bounded contexts are **natural microservice (or module) boundaries**.

### Context Map
How bounded contexts relate: shared kernel, customer/supplier, conformist, **anti-corruption layer (ACL)** (a translation layer protecting your model from a messy external or legacy model, essentially the [[Structural Patterns#Adapter|Adapter]] pattern at system scale), open host service, published language.

---

## Tactical DDD (building blocks in code)

| Concept | Definition | Example |
|---|---|---|
| **Entity** | Has a unique **identity** that persists over time; mutable | `User(id=42)`, `Order(id=...)` |
| **Value Object** | Defined only by its **attributes**; **immutable**; no identity | `Money(500, "USD")`, `Address`, `DateRange` |
| **Aggregate** | A cluster of entities/value objects treated as **one consistency unit** | `Order` + its `OrderLines` |
| **Aggregate Root** | The single entry point to the aggregate; outside code only references the root | `Order` (you never modify an `OrderLine` directly) |
| **Repository** | Persistence abstraction for aggregates | `OrderRepository.save(order)` |
| **Domain Service** | Business logic that doesn't naturally belong to one entity | `TransferService.transfer(from, to, amt)` |
| **Domain Event** | Something meaningful that happened in the domain | `OrderPlaced`, `PaymentCaptured` |
| **Factory** | Encapsulates complex aggregate creation | `OrderFactory.fromCart(cart)` |

### Aggregate rules (useful in system design too)
1. Protect **invariants** inside the aggregate boundary ("order total = sum of lines", "can't add lines after shipping").
2. **One transaction = one aggregate.** Changes across aggregates are **eventually consistent** via domain events.
3. Reference other aggregates **by ID**, not by object reference.
4. Keep aggregates **small**.

```javascript
class Money {                       // Value Object
  constructor(cents, currency) { this.cents = cents; this.currency = currency; Object.freeze(this); }
  add(other) {
    if (other.currency !== this.currency) throw new Error("Currency mismatch");
    return new Money(this.cents + other.cents, this.currency);   // returns new, never mutates
  }
  equals(o) { return o.cents === this.cents && o.currency === this.currency; }
}

class Order {                       // Aggregate root
  #lines = []; #status = "DRAFT";
  constructor(id, customerId) { this.id = id; this.customerId = customerId; }  // customer by ID
  addLine(productId, qty, unitPrice) {
    if (this.#status !== "DRAFT") throw new Error("Cannot modify a placed order");  // invariant
    this.#lines.push({ productId, qty, unitPrice });
  }
  total() { return this.#lines.reduce((t, l) => t.add(new Money(l.unitPrice.cents * l.qty, l.unitPrice.currency)), new Money(0, "USD")); }
  place() {
    if (this.#lines.length === 0) throw new Error("Empty order");
    this.#status = "PLACED";
    return { type: "OrderPlaced", orderId: this.id, total: this.total() };   // domain event
  }
}
```

### Anemic vs Rich domain model
- **Anemic**: entities are just data bags with getters/setters; all logic lives in services. Common in CRUD apps; Fowler calls it an anti-pattern for complex domains.
- **Rich**: entities enforce their own rules (like `Order` above). Better when business logic is complex.
- Simple CRUD apps don't need DDD. Use it where the domain is genuinely complex.

---

## Event Storming
A workshop technique: domain experts and engineers place sticky notes for **domain events** (orange), commands (blue), actors, and policies on a timeline to discover bounded contexts and aggregates. You may join one at a big company.

## Self-test
- [ ] Entity vs value object: give 2 examples each.
- [ ] What's a bounded context, and how does it relate to microservices?
- [ ] Why should a transaction only modify one aggregate?
- [ ] What is an anti-corruption layer?

Next: [[System Design Interview Framework]]
