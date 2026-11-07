---
tags: [design, design-patterns, creational]
aliases: [Singleton, Factory, Builder, Abstract Factory, Prototype]
---
# Creational Design Patterns

> [!summary]
> Creational patterns deal with **how objects get created**: decoupling *what* you create from *how* you create it. From the "Gang of Four" (GoF) book, *Design Patterns* (1994), which has 23 patterns in 3 groups: Creational (5), [[Structural Patterns|Structural]] (7), [[Behavioral Patterns|Behavioral]] (11).

Back to [[Design Index]]

> [!tip] How to study patterns
> For each pattern know: **the problem it solves → the structure (1 sentence) → a real example you've seen → a downside**. Interviewers care more that you can *recognize when to apply one* than that you recite UML.

---

## Singleton

**Intent:** Ensure a class has only **one instance** and provide a global access point.

**Real uses:** config objects, logger, connection pool, cache client.

```javascript
class Config {
  static #instance;
  constructor() {
    if (Config.#instance) return Config.#instance;
    this.settings = { env: process.env.NODE_ENV };
    Config.#instance = this;
  }
  static getInstance() { return Config.#instance ?? new Config(); }
}

// In Node, the module system already gives you a singleton:
// db.js
//   const pool = new Pool(); module.exports = pool;   // cached after first require
```

**Downsides (frequently asked!):**
- It's **global state** in disguise: hidden dependencies, hard to test (can't easily swap in a mock).
- Thread-safety issues in multithreaded languages (Java needs double-checked locking or an enum singleton).
- Violates SRP (manages its own lifecycle + does its job).

> [!note] Modern take
> Prefer letting a **DI container** manage a single instance ("singleton scope") and inject it. You get one instance *without* global access.

---

## Factory Method

**Intent:** Define an interface for creating an object, but let subclasses/logic **decide which class to instantiate**.

```javascript
class NotificationFactory {
  static create(channel) {
    switch (channel) {
      case "email": return new EmailNotification();
      case "sms":   return new SmsNotification();
      case "push":  return new PushNotification();
      default: throw new Error(`Unknown channel ${channel}`);
    }
  }
}
const n = NotificationFactory.create(user.preferredChannel);
n.send(user, "Your order shipped");
```

(Strictly speaking, that's a **Simple Factory**. The GoF *Factory Method* uses subclassing: `Dialog.createButton()` overridden by `WindowsDialog` and `WebDialog`. In interviews, "factory" usually means either.)

**When:** creation logic is complex or conditional, or callers shouldn't know concrete classes.
**Downside:** another layer of indirection; a switch-based factory still has to change when you add types (you can use a registry map instead).

---

## Abstract Factory

**Intent:** Create **families of related objects** without specifying concrete classes.

```javascript
// A UI kit factory produces matching components
const lightTheme = { button: () => new LightButton(), modal: () => new LightModal() };
const darkTheme  = { button: () => new DarkButton(),  modal: () => new DarkModal()  };

function renderPage(factory) {
  const btn = factory.button();
  const modal = factory.modal();    // guaranteed to match the button
}
```

**Real uses:** cross-platform UI toolkits, swapping a family of cloud clients (AWS vs GCP storage + queue + DB), test vs prod object families.

---

## Builder

**Intent:** Construct a complex object **step by step**, separating construction from representation. Avoids "telescoping constructors" with 10 params.

```javascript
class HttpRequestBuilder {
  #req = { method: "GET", headers: {}, query: {} };
  url(u)            { this.#req.url = u; return this; }
  method(m)         { this.#req.method = m; return this; }
  header(k, v)      { this.#req.headers[k] = v; return this; }
  body(b)           { this.#req.body = JSON.stringify(b); return this; }
  build() {
    if (!this.#req.url) throw new Error("url required");   // validate at the end
    return Object.freeze({ ...this.#req });                  // immutable result
  }
}

const req = new HttpRequestBuilder()
  .url("/api/orders").method("POST")
  .header("Authorization", "Bearer x")
  .body({ sku: "KC-01", qty: 2 })
  .build();
```

**Real uses:** query builders (Knex, SQLAlchemy), Java's `StringBuilder`, Lombok `@Builder`, test data builders (`aUser().withEmail(...).build()`).

---

## Prototype

**Intent:** Create new objects by **cloning** an existing instance instead of constructing from scratch.

```javascript
const baseEnemy = { hp: 100, speed: 5, attack() { /* ... */ } };
const fastEnemy = { ...structuredClone({ hp: baseEnemy.hp, speed: 10 }), attack: baseEnemy.attack };
// JS itself is prototype-based: Object.create(proto)
const goblin = Object.create(baseEnemy);
```

**When:** object creation is expensive (loaded from DB/config) and you need many similar copies. Watch out for **shallow vs deep copy** bugs.

---

## Related: Object Pool

Not GoF, but commonly discussed. Reuse expensive objects (DB connections, threads) instead of creating and destroying them. **DB connection pools** are everywhere in backend work. A pool size that's too small causes queueing; too large overloads the DB.

---

## Comparison

| Pattern | Use when you need… | Example |
|---|---|---|
| Singleton | exactly one shared instance | Logger, config |
| Factory Method | to choose a concrete class at runtime | `NotificationFactory.create("sms")` |
| Abstract Factory | families of matching objects | Light/Dark UI kits |
| Builder | many optional params / step-wise construction | HTTP request, query builder |
| Prototype | cheap copies of a preconfigured object | Game entities, document templates |

## Self-test
- [ ] Why is Singleton often called an anti-pattern? What's the alternative?
- [ ] Factory vs Abstract Factory: what's the difference?
- [ ] When would you choose Builder over a constructor with an options object (in JS, the options object often suffices)?

Next: [[Structural Patterns]]
