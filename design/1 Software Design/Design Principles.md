---
tags: [design, software-design, principles]
aliases: [DRY, KISS, YAGNI, Clean Code, Code Smells]
---
# Design Principles (Beyond SOLID)

> [!summary] What you'll learn
> The everyday rules of thumb that experienced engineers apply in code review: DRY, KISS, YAGNI, separation of concerns, the Law of Demeter, and more. You'll also learn what "clean code" means in practice, how to recognize **code smells** and fix them with named **refactorings**, and how to think about **technical debt**.
>
> Each principle is presented as **what it says → why it exists → example → when it's wrong to apply it**. Many of these principles pull against each other, and knowing *when* to apply one is the actual skill.

Back to [[Design Index]] · Prev: [[SOLID Principles]] · Next: [[Creational Patterns]]

---

## DRY: Don't Repeat Yourself

### What it says
> "Every piece of **knowledge** must have a single, unambiguous, authoritative representation within a system." (Andy Hunt & Dave Thomas, *The Pragmatic Programmer*, 1999)

### Why it exists
If the same business rule (e.g., "orders over $50 ship free") is written in five places, changing the rule means finding and updating all five. Miss one and the system becomes inconsistent: the cart says free shipping, the invoice charges $5.

### Example
```javascript
// ❌ The rule is duplicated in the cart UI, checkout API, and invoice generator
if (cart.total > 50) shipping = 0;        // cart.js
if (order.subtotal >= 50) shipping = 0;   // checkout.js  (≥ vs > — already drifted!)
if (inv.amount > 50) inv.shipping = 0;    // invoice.js

// ✅ One authoritative source
// shipping-policy.js
export const FREE_SHIPPING_THRESHOLD_CENTS = 5000;
export const qualifiesForFreeShipping = (subtotalCents) => subtotalCents >= FREE_SHIPPING_THRESHOLD_CENTS;
```

### When DRY goes wrong
DRY is about **knowledge**, not **text**. Two pieces of code that *look* identical but represent **different rules that happen to match today** are not duplication. If you merge them, then the day one rule changes, you'll need flags and special cases to pull them apart again.

```javascript
// These look the same today...
const validateUsername = (s) => s.length >= 3 && s.length <= 20;
const validateTeamName = (s) => s.length >= 3 && s.length <= 20;
// ...but they're different business rules. Product will change one without the other.
// Merging them into validateName(s) couples two unrelated concepts.
```

This leads to the counter-principles below.

### WET, AHA, and the Rule of Three
- **WET** ("Write Everything Twice"): it's OK to duplicate once while the pattern is still unclear.
- **AHA** ("Avoid Hasty Abstractions", Kent C. Dodds): prefer duplication over the *wrong* abstraction.
- **Rule of Three**: wait until you've seen the same thing **three times** before extracting it. By then you understand what's truly common.

> "Duplication is far cheaper than the wrong abstraction." — Sandi Metz

The "wrong abstraction" failure mode looks like this: someone extracts a shared function, then each new caller needs a slightly different behavior, so they add a parameter, then another, until `formatThing(x, true, false, null, "legacy")` is unreadable and every change risks breaking every caller.

---

## KISS: Keep It Simple, Stupid

### What it says
Prefer the **simplest solution that solves the actual problem**. (Originally attributed to Lockheed engineer Kelly Johnson: jet parts had to be repairable by an average mechanic in the field with basic tools.)

### Why it exists
Complexity is the main cost in software. Every clever trick, extra layer, or unnecessary technology must be understood, debugged, and maintained by everyone who touches the code later, including you in six months. Code is read far more often than it's written.

### Examples
```javascript
// ❌ clever
const isEven = n => !(n & 1);
const uniq = a => [...a.reduce((m, x) => m.set(JSON.stringify(x), x), new Map()).values()];

// ✅ obvious (unless profiling shows this exact line matters)
const isEven = n => n % 2 === 0;
const uniqueIds = ids => [...new Set(ids)];
```

At the **system** level, KISS means:
- One Postgres database before introducing Kafka + Cassandra + Elasticsearch.
- A cron job before a distributed workflow engine.
- A monolith before microservices (see [[Architectural Styles#Monolith]]).

> [!tip] In system design interviews
> Start with the simplest design that meets the functional requirements, then add complexity **only to address a stated non-functional requirement** ("we need caching because reads are 50K/s"). Interviewers explicitly penalize unjustified complexity.

### When it's wrong
"Simple" isn't "naive." Ignoring real requirements (security, concurrency, error handling) isn't simplicity, it's incompleteness. The goal is the least complexity that is *still correct*.

---

## YAGNI: You Aren't Gonna Need It

### What it says
Don't build features, options, or extension points until you **actually need** them. (From Extreme Programming, late 1990s.)

### Why it exists
Speculative code has real costs even if it's never used:
1. **Cost of building** it now (time not spent on what's needed).
2. **Cost of carrying** it: more code to read, test, and keep compatible.
3. **Cost of being wrong**: when the real need arrives, it's usually different from what you guessed, so you rewrite it anyway.

### Example
```javascript
// ❌ "We might support multiple currencies, databases, and themes someday"
class PriceFormatter {
  constructor(locale, currency, roundingStrategy, pluginRegistry, themeProvider) { ... }
}

// ✅ The product is US-only today
const formatUSD = (cents) => `$${(cents / 100).toFixed(2)}`;
```

### YAGNI vs Open/Closed: resolving the tension
[[SOLID Principles#O — Open/Closed Principle (OCP)|OCP]] says "design for extension"; YAGNI says "don't build what you don't need." They reconcile like this:
- Keep code **easy to change** (small functions, clear boundaries, tests). That's always cheap and always worth it.
- Add **actual extension mechanisms** (interfaces, plugin systems, config options) only when a second variant exists or is concretely scheduled.

YAGNI is about **features and flexibility**, not about skipping quality. It never justifies skipping tests, error handling, or security.

---

## Separation of Concerns (SoC)

### What it says
Divide a program into distinct sections, each handling **one concern**: one area of functionality or one kind of knowledge. (Edsger Dijkstra, 1974.)

### Why it exists
When concerns are tangled (e.g., SQL inside React components, or HTML strings built inside business logic), you can't change one without understanding all of them, and you can't reuse or test them independently.

### Where you see it
| Scale | Separation |
|---|---|
| Web page | HTML (structure) / CSS (presentation) / JS (behavior) |
| Frontend | Components (UI) / hooks or state management (logic) / API client (data fetching) |
| Backend | Routes/controllers (HTTP) / services (business rules) / repositories (data access) |
| Architecture | [[Architectural Styles#Layered (N-tier) architecture\|Layers]], MVC, microservices |

SRP is SoC applied to individual classes.

---

## Law of Demeter (Principle of Least Knowledge)

### What it says
A method should only call methods on:
1. its own object (`this`),
2. its parameters,
3. objects it creates,
4. its direct fields/components.

Informally: **"only talk to your immediate friends; don't talk to strangers."**

### Why it exists
When code navigates deep object chains, it becomes coupled to the **entire structure** of other objects. If `Customer` stops having an `Address` object (say addresses move to a separate service), every chain through it breaks.

```javascript
// ❌ "Train wreck": this code knows Order → Customer → Address → City → zip
const zip = order.getCustomer().getAddress().getCity().getZip();

// ✅ Ask the nearest object for what you need; it handles the navigation
const zip = order.shippingZip();
```

### When it's wrong
It doesn't apply to **plain data structures and fluent APIs**. `response.data.items[0].name` on a JSON payload, or a builder chain like `query.where(...).orderBy(...).limit(10)`, is fine. The principle targets objects with behavior whose internals you shouldn't depend on.

---

## Tell, Don't Ask

### What it says
**Tell objects what to do** instead of asking for their data and making decisions on their behalf.

```javascript
// ❌ Ask: the caller pulls data out and applies the account's rules itself
if (account.getBalance() >= amount && !account.isFrozen()) {
  account.setBalance(account.getBalance() - amount);
}

// ✅ Tell: the account enforces its own rules (encapsulation)
account.withdraw(amount);   // throws if frozen or insufficient funds
```

### Why it exists
The "ask" version spreads the account's rules across every caller. The "tell" version keeps the rules with the data, which is the whole point of [[OOP Fundamentals#Pillar 1: Encapsulation|encapsulation]].

---

## Program to an interface, not an implementation

Depend on **what** something does (its contract), not on **which class** does it. From the *Design Patterns* book. Example: a function that accepts any `PaymentProcessor` rather than specifically a `StripeClient`. This is the foundation of [[SOLID Principles#D — Dependency Inversion Principle (DIP)|DIP]], DI, testability, and most patterns.

---

## Principle of Least Astonishment (Least Surprise)

### What it says
A component should behave the way most users (or developers) would **expect**.

### Examples of violations
- `getUser(id)` that also updates `lastSeen` in the database (a hidden side effect).
- An `isValid()` method that throws instead of returning `false`.
- A `DELETE` endpoint that returns 200 with `{ "success": false }`.
- `array.sort()` in JS sorting numbers as strings (`[10, 9, 1].sort()` → `[1, 10, 9]`). A famous real-world astonishment.

### Why it matters
Surprising behavior causes bugs in every caller who reasonably assumed otherwise. Good naming and consistent conventions are the main tools.

---

## Fail Fast

### What it says
Detect problems **as early as possible** and fail **loudly**, close to the cause.

```javascript
function createUser({ email, age }) {
  if (!email?.includes("@")) throw new ValidationError("Invalid email");   // fail at the boundary
  if (!Number.isInteger(age) || age < 0) throw new ValidationError("Invalid age");
  // ...rest of the logic can trust the data
}
```

### Why it exists
If bad data slips through, it fails much later in some unrelated place (or worse, silently corrupts the database), and the error message gives no clue where the problem started. Validate at **trust boundaries**: API inputs, config loading, messages from queues, third-party responses.

**Related:** don't swallow exceptions (`catch (e) {}`). If you can't handle an error meaningfully, let it propagate or log it with context and rethrow.

---

## Command–Query Separation (CQS)

### What it says
Every method should either be a **command** (changes state, returns nothing) or a **query** (returns data, changes nothing), **not both**. (Bertrand Meyer.)

```javascript
// ❌ Mixed: does calling it twice pop two items? Can I call it "just to check"?
stack.popAndReturnSize();

// ✅ Separate
stack.pop();      // command
stack.size();     // query (safe to call any number of times)
```

### Why it exists
Queries become safe to call anywhere (in logs, debuggers, tests) without side effects. Reasoning about state changes becomes easier.
**Pragmatic exceptions:** `array.pop()` and atomic operations like `INCR` that return the new value are deliberate violations for convenience or atomicity.

Scaled up to architecture, CQS becomes [[Architectural Styles#CQRS|CQRS]].

---

## Convention over Configuration

Frameworks assume sensible defaults so you write less boilerplate: Rails maps the `User` model to the `users` table; Next.js maps `app/about/page.tsx` to `/about`; Spring Boot auto-configures a datasource if it sees a DB driver. You only configure what deviates from the convention. **Benefit:** less code, consistency across projects. **Cost:** "magic" that's confusing until you learn the conventions.

---

## Idempotency (preview)

An operation is **idempotent** if doing it multiple times has the same effect as doing it once (`SET x = 5` is idempotent; `x = x + 1` isn't). It's critical once networks and retries are involved. Covered in depth in [[API Design#Idempotency]] and [[Distributed Systems Patterns#Idempotency]].

---

## Clean Code essentials

What reviewers at large companies actually comment on. (Mostly from Robert Martin's *Clean Code* and the Google style guides.)

### Naming
- **Intention-revealing**: `elapsedTimeInDays`, not `d` or `data2`.
- **Functions are verbs** (`calculateTax`, `sendInvite`); **classes are nouns** (`InvoiceRenderer`).
- **Booleans read as yes/no questions**: `isActive`, `hasPermission`, `shouldRetry`.
- **Avoid noise words**: `UserData`, `UserInfo`, `UserManager` say nothing extra about `User`.
- **Use domain vocabulary** consistently (see "ubiquitous language" in [[Domain-Driven Design]]).

### Functions
- **Small**, doing one thing at one level of abstraction. If you need comments to separate "sections" of a function, those sections want to be functions.
- **Few parameters**: more than ~3 is a smell; group them into an options object or value object.
- **No hidden side effects**: `checkPassword()` must not also start a session.
- **No flag arguments**: `render(true)` is unreadable at the call site. Split into `renderForPrint()` and `renderForScreen()`.
- **Guard clauses / early returns** instead of deep nesting:

```javascript
// ❌ Arrow-shaped code
function ship(order) {
  if (order) {
    if (order.isPaid) {
      if (order.items.length > 0) {
        // actual logic buried 3 levels deep
      }
    }
  }
}

// ✅ Guard clauses: handle the edge cases first, then the main path reads straight down
function ship(order) {
  if (!order) throw new Error("No order");
  if (!order.isPaid) throw new Error("Order not paid");
  if (order.items.length === 0) return;
  // main logic at top level
}
```

### Comments
- Comments explain **why** (a business reason, a workaround for a bug, a non-obvious constraint), not **what**. The code should say what.
- Outdated comments are worse than none.
- Good: `// Stripe retries webhooks for 3 days, so dedupe by event ID`. Bad: `// increment i`.

### Error handling
- Use exceptions (or result types) consistently. Don't mix "return null on failure" and "throw on failure" in the same API.
- Don't return error codes that callers can ignore.
- Include context in errors (`Order o_123: payment declined (insufficient_funds)`).

### Other habits
- **Magic numbers → named constants** (`MAX_LOGIN_ATTEMPTS = 5`).
- **Boy Scout Rule**: leave the code a little cleaner than you found it (rename a confusing variable, extract a function) in the area you're already touching.
- **Consistent formatting** via tools (Prettier, ESLint, gofmt), so reviews never argue about style.

---

## Code smells and refactorings

A **code smell** (Kent Beck & Martin Fowler, *Refactoring*) is a surface symptom that *usually* indicates a deeper design problem. A **refactoring** is a small, behavior-preserving change that improves structure. Refactor in tiny steps with tests passing between each step.

| Smell | What it looks like | Why it's a problem | Typical refactoring |
|---|---|---|---|
| **Long Method** | 100+ line function | Hard to understand, test, reuse | **Extract Function** |
| **Large Class / God Object** | One class knows and does everything | Violates SRP; every change touches it | **Extract Class** |
| **Long Parameter List** | `create(a, b, c, d, e, f)` | Easy to pass args in the wrong order | **Introduce Parameter Object** |
| **Duplicated Code** | Same logic in several places | Rules drift apart | **Extract Function**, Pull Up Method |
| **Feature Envy** | A method uses another object's data more than its own | Logic is in the wrong place | **Move Method** to the data's owner |
| **Data Clumps** | The same fields always travel together (`lat, lng`; `start, end`) | A missing concept | **Introduce Value Object** (`Coordinates`, `DateRange`) |
| **Primitive Obsession** | Strings and numbers for domain concepts (`"USD"`, `price: 1999`) | No validation, easy to mix units | **Replace Primitive with Object** (`Money`) |
| **Switch Statements** | The same `switch(type)` in many places | Violates OCP | **Replace Conditional with Polymorphism** |
| **Shotgun Surgery** | One logical change requires edits in many classes | Responsibility is scattered | **Move Method/Field** to consolidate |
| **Divergent Change** | One class is edited for many unrelated reasons | Violates SRP | **Extract Class** |
| **Speculative Generality** | Abstract classes, hooks, or params "for the future" with one user | YAGNI | **Collapse Hierarchy**, **Inline** |
| **Dead Code** | Unused functions, unreachable branches, commented-out code | Noise; misleads readers | **Delete it** (git remembers) |
| **Comments as deodorant** | Long comments explaining confusing code | The code itself is unclear | **Rename**, **Extract Function** |
| **Temporary Field** | A field only set in some situations | Confusing object state | **Extract Class** |
| **Message Chains** | `a.b().c().d()` | Violates Law of Demeter | **Hide Delegate** |

---

## Technical debt

### What it is
The **implied future cost of choosing an expedient solution now** instead of a better one that would take longer. (Ward Cunningham's metaphor: like financial debt, you get something now, and you pay **interest**, meaning slower development and more bugs, until you pay down the **principal** by refactoring.)

### Kinds of debt (Martin Fowler's quadrant)
| | Reckless | Prudent |
|---|---|---|
| **Deliberate** | "We don't have time for design." | "We'll ship now and refactor after the launch." ✅ often a valid business decision |
| **Inadvertent** | "What's layering?" | "Now we know how we should have built it." (Unavoidable learning) |

### How to manage it in practice
- **Make it visible**: tickets, `// TODO(JIRA-123): ...` with a ticket rather than a bare TODO.
- **Explain it in business terms** to PMs: "This module causes ~30% of our incidents and slows every feature in it by about a week."
- **Pay it down incrementally**: Boy Scout Rule; allocate a percentage of each sprint; refactor the area before building a feature there.
- **Don't rewrite from scratch** unless absolutely necessary. Rewrites usually take far longer than expected and lose hidden knowledge embedded in the old code. Prefer incremental replacement (the "strangler fig" approach in [[Architectural Styles]]).

---

## Quick self-test
- [ ] DRY is about knowledge, not text. Give an example of two identical-looking functions that should *not* be merged.
- [ ] What's the Rule of Three, and what problem does it prevent?
- [ ] How do YAGNI and Open/Closed conflict, and how do you reconcile them?
- [ ] Rewrite a "train wreck" call chain to satisfy the Law of Demeter.
- [ ] Explain Command–Query Separation and one pragmatic exception.
- [ ] Name five code smells and the refactoring for each.
- [ ] How would you explain the cost of tech debt to a product manager?

Next: [[Creational Patterns]]
