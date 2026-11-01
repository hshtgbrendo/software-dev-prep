---
tags: [design, design-patterns, creational]
aliases: [Singleton, Factory, Builder, Abstract Factory, Prototype, Design Patterns]
---
# Creational Design Patterns

> [!summary] What you'll learn
> A **design pattern** is a named, reusable solution to a problem that keeps coming up in software design. Patterns give teams a shared vocabulary ("let's put a factory here") and capture decades of trial and error. The classic catalog is *Design Patterns: Elements of Reusable Object-Oriented Software* (1994) by the "Gang of Four" (GoF): 23 patterns in three groups:
> - **Creational** (this note): *how objects get created*
> - **[[Structural Patterns|Structural]]**: *how objects are composed into larger structures*
> - **[[Behavioral Patterns|Behavioral]]**: *how objects communicate and share responsibilities*
>
> **Creational patterns** separate *what* gets created from *how and when* it's created, so the rest of your code doesn't depend on concrete classes or messy construction logic.

Back to [[Design Index]] · Prev: [[Design Principles]] · Next: [[Structural Patterns]]

> [!tip] How to study each pattern
> For every pattern, be able to answer:
> 1. **What problem does it solve?** (the pain without it)
> 2. **How does it work?** (one sentence + a sketch)
> 3. **When would I use it?** (concrete signals)
> 4. **Where have I seen it?** (frameworks, libraries)
> 5. **What's the downside?**
>
> Interviewers care far more that you **recognize when a pattern fits** than that you can draw the exact UML from the book.

> [!note] Patterns in JavaScript
> The GoF book targeted C++ and Smalltalk, where classes are the only tool. In JavaScript, first-class functions, closures, object literals, and modules make several patterns much lighter (sometimes a single function). The *problem* each pattern solves stays the same.

---

## Singleton

### The problem
Some things should exist **exactly once** in a running program, and many parts of the code need access to that single instance:
- A **configuration** object loaded from files and environment variables.
- A **database connection pool** (creating several pools would open far too many connections).
- A **logger** writing to one destination.
- An in-process **cache**.

If any code can call `new ConnectionPool()`, you can accidentally end up with ten pools and exhaust the database's connection limit.

### How it works
The class controls its own instantiation: the constructor is private (or guarded), and a static method returns the one shared instance, creating it lazily on first use.

```javascript
class Config {
  static #instance = null;

  constructor() {
    if (Config.#instance) throw new Error("Use Config.getInstance()");
    this.values = { env: process.env.NODE_ENV ?? "development", port: Number(process.env.PORT ?? 3000) };
  }

  static getInstance() {
    if (!Config.#instance) Config.#instance = new Config();   // lazy initialization
    return Config.#instance;
  }
}

Config.getInstance() === Config.getInstance(); // true
```

**In Node.js you get singletons for free from the module system.** A module is evaluated once and cached, so every `import` receives the same object:

```javascript
// db.js
import { Pool } from "pg";
export const pool = new Pool({ connectionString: process.env.DATABASE_URL, max: 10 });

// anywhere else
import { pool } from "./db.js";   // same pool everywhere
```

### When to use it
- The resource is genuinely **one-per-process** by nature (pool, config, hardware device handle).
- Creating more than one would be **wrong or wasteful**, not merely unnecessary.

### Downsides (this is one of the most-asked pattern questions)
1. **It's global state in disguise.** Any code can reach it, so dependencies become hidden: you can't tell from a function's signature that it uses the database.
2. **Hard to test.** Tests can't easily substitute a fake, and state leaks between tests because the instance persists.
3. **Violates SRP**: the class manages its own lifecycle *and* does its real job.
4. **Concurrency issues** in multithreaded languages: two threads can both see "no instance yet" and create two. Java solutions include eager initialization, double-checked locking with `volatile`, or an `enum` singleton.

### The modern alternative
Let your **composition root** or **DI container** create one instance and **inject it** wherever it's needed ("singleton scope" in Spring/NestJS/.NET). You keep the "only one instance" benefit without the global access and the testing problems.

---

## Factory Method (and the "Simple Factory")

### The problem
Code often needs to create one of several related classes depending on runtime information:

```javascript
// Scattered across the codebase...
let notifier;
if (user.pref === "email") notifier = new EmailNotifier(smtpConfig);
else if (user.pref === "sms") notifier = new SmsNotifier(twilioKey);
else if (user.pref === "push") notifier = new PushNotifier(fcmCreds);
```
Problems:
- Every place that creates notifiers **knows every concrete class** and its constructor arguments.
- Adding a new type means finding and editing every such `if/else`.
- Construction details (config, credentials) leak into business code.

### How it works
**Put creation logic in one place** that returns objects through a common interface. Callers ask for "a notifier for this channel" and get back something with `.send()`.

**Simple Factory** (the most common form in practice):
```javascript
const notifierFactories = {
  email: () => new EmailNotifier(config.smtp),
  sms:   () => new SmsNotifier(config.twilio),
  push:  () => new PushNotifier(config.fcm),
};

function createNotifier(channel) {
  const make = notifierFactories[channel];
  if (!make) throw new Error(`Unsupported channel: ${channel}`);
  return make();
}

// Caller code: knows nothing about concrete classes or credentials
createNotifier(user.pref).send(user, "Your order shipped!");
```
Using a **registry map** instead of a `switch` means a new channel only requires adding one entry (Open/Closed).

**GoF Factory Method** (the formal version) uses subclassing. A base class defines an abstract "create" method, and each subclass decides which concrete product to create:

```javascript
class Dialog {
  render() {
    const button = this.createButton();   // factory method: subclasses decide
    button.onClick(() => this.close());
    button.draw();
  }
  createButton() { throw new Error("subclass must implement"); }
}
class WebDialog extends Dialog     { createButton() { return new HtmlButton(); } }
class WindowsDialog extends Dialog { createButton() { return new WindowsButton(); } }
```
The base class contains shared logic (`render`) and lets subclasses plug in the specific product.

### When to use it
- Which class to create depends on **runtime data** (config, user input, file type).
- Construction is **complex** (needs config, credentials, several steps) and shouldn't be repeated.
- You want callers to depend on an **interface**, not concrete classes.
- **Static factory methods** with descriptive names improve readability: `User.fromJSON(json)`, `Money.ofCents(500)`, `Date.parse()`.

### Where you've seen it
`document.createElement("div")`, `React.createElement`, `Promise.resolve()`, `Array.from()`, `express()`, ORMs creating model instances from rows, Java's `Calendar.getInstance()`.

### Downsides
An extra layer of indirection. Don't create a factory for a class with one implementation and a trivial constructor; `new Thing()` is fine.

---

## Abstract Factory

### The problem
Sometimes you create **families of related objects that must be used together**, and you want to switch the entire family at once:
- A UI toolkit: `Button`, `Checkbox`, `Modal`, all in **Light** theme or all in **Dark** theme (never a mix).
- A cloud abstraction: `Storage`, `Queue`, `Secrets`, all on AWS or all on GCP.
- Testing: all real implementations, or all in-memory fakes.

If each part of the code chooses components individually, it's easy to accidentally mix families (a dark button in a light modal, or an AWS queue with a GCP storage client).

### How it works
Define a factory **interface** with one creation method per product. Each concrete factory produces a consistent family. The client receives one factory and creates everything through it.

```javascript
// Two concrete factories implementing the same "interface"
const awsFactory = {
  createStorage: () => new S3Storage(),
  createQueue:   () => new SqsQueue(),
  createSecrets: () => new AwsSecretsManager(),
};

const inMemoryFactory = {           // for local dev and tests
  createStorage: () => new InMemoryStorage(),
  createQueue:   () => new InMemoryQueue(),
  createSecrets: () => new EnvSecrets(),
};

// Application code is written once and works with either family
function bootstrapApp(infra) {
  const storage = infra.createStorage();
  const queue = infra.createQueue();
  return new UploadService(storage, queue);
}

const app = bootstrapApp(process.env.NODE_ENV === "test" ? inMemoryFactory : awsFactory);
```

### Factory Method vs Abstract Factory
| | Factory Method | Abstract Factory |
|---|---|---|
| Creates | **One** product | A **family** of related products |
| Mechanism | One method (often overridden by subclasses) | An object with several creation methods |
| Use when | "Which class do I instantiate?" | "Which *set* of classes do I instantiate, consistently?" |

### Downside
Adding a new **kind** of product (say `createCache()`) means updating every factory. Use it only when you truly have multiple families.

---

## Builder

### The problem
Some objects have **many parameters, most of them optional**, or must be assembled in **several steps**:

```javascript
// "Telescoping constructor": which argument is which?
new HttpRequest("POST", "/orders", null, { "Content-Type": "application/json" }, body, 5000, true, 3, false);
```
- Call sites are unreadable and error-prone (arguments easy to swap).
- You can't validate the *combination* of options until everything is set.
- Partially built objects might be used by mistake.

### How it works
A separate **builder** object collects the configuration through readable, chainable methods, then a final `build()` validates everything and returns the finished (often **immutable**) object.

```javascript
class HttpRequestBuilder {
  #req = { method: "GET", headers: {}, timeoutMs: 10_000, retries: 0 };

  url(u)          { this.#req.url = u; return this; }       // return this → chaining
  method(m)       { this.#req.method = m; return this; }
  header(k, v)    { this.#req.headers[k] = v; return this; }
  json(body)      { this.#req.body = JSON.stringify(body); return this.header("Content-Type", "application/json"); }
  timeout(ms)     { this.#req.timeoutMs = ms; return this; }
  retries(n)      { this.#req.retries = n; return this; }

  build() {
    const r = this.#req;
    if (!r.url) throw new Error("url is required");
    if (r.method === "GET" && r.body) throw new Error("GET requests cannot have a body");  // cross-field validation
    return Object.freeze({ ...r, headers: { ...r.headers } });                              // immutable result
  }
}

const request = new HttpRequestBuilder()
  .url("/api/orders")
  .method("POST")
  .header("Authorization", "Bearer token")
  .json({ sku: "KC-01", qty: 2 })
  .retries(3)
  .build();
```

### In JavaScript: often an options object is enough
```javascript
createRequest({ url: "/api/orders", method: "POST", body, retries: 3 });
```
Named options already solve the readability problem. Use a real builder when you need **step-by-step construction**, **cross-field validation**, **immutability of the result**, or a **fluent DSL** (query builders).

### When to use it
- Many optional parameters, or construction in several steps.
- Constructing complex things like SQL queries, HTTP requests, documents, test data.
- You want the final object to be immutable and always valid.

### Where you've seen it
Query builders (Knex: `knex("users").where("age", ">", 18).orderBy("name").limit(10)`), Mongoose queries, Java's `StringBuilder` and Lombok's `@Builder`, Protocol Buffers builders, and **test data builders** (`aUser().withEmail("x@y.com").admin().build()`), which are very common in enterprise test suites.

---

## Prototype

### The problem
Creating an object from scratch is sometimes **expensive** (loaded from a DB, parsed from a large config, computed) or **complicated** (many fields to set), and you need many similar copies with small variations.

### How it works
Keep a pre-configured **prototype** object and **clone** it, then tweak the copy.

```javascript
const baseInvoice = {
  currency: "USD",
  terms: "Net 30",
  company: { name: "KeycapAlley", address: "Seattle, WA" },
  lineItems: [],
};

// Deep clone so copies don't share nested objects
const invoiceA = structuredClone(baseInvoice);
invoiceA.lineItems.push({ sku: "KC-01", qty: 2 });

// Shallow copy pitfall:
const invoiceB = { ...baseInvoice };
invoiceB.company.name = "Oops";   // ⚠️ also changes baseInvoice.company.name (shared reference)
```

JavaScript itself is a **prototype-based** language: objects inherit directly from other objects (`Object.create(proto)`), and `class` syntax is built on top of that.

### When to use it
- Object creation is costly and copies are cheap.
- You have a few "preset" configurations (document templates, game entity archetypes like "goblin," "orc").
- You want to create objects without coupling to their concrete classes.

### Downside
**Shallow vs deep copy bugs** (shown above). Cloning objects with circular references, functions, or class instances needs care (`structuredClone` doesn't copy functions or class prototypes).

---

## Related: Object Pool (not GoF, very common in backend work)

### The problem
Some objects are **expensive to create and destroy**: database connections (TCP + TLS handshake + authentication can take tens of milliseconds), threads, large buffers. Creating one per request would add latency and could overwhelm the server (Postgres handles each connection with a separate OS process).

### How it works
Keep a pool of ready-made objects. A request **borrows** one, uses it, and **returns** it. If none are available, the request waits (up to a timeout) or the pool grows to a maximum size.

```javascript
const pool = new Pool({ max: 20, idleTimeoutMillis: 30_000 });
const client = await pool.connect();   // borrow
try {
  await client.query("SELECT ...");
} finally {
  client.release();                     // return (always, even on error)
}
```

### Tuning matters
- **Too small**: requests queue waiting for a connection → latency spikes.
- **Too large**: the database is overwhelmed by connections (each uses memory) → everything slows down.
- With many app instances, total connections = instances × pool size. That's why tools like **PgBouncer** (a connection pooler in front of Postgres) exist.

---

## Comparison: which creational pattern?

| Pattern | Solves | Signal you need it | Example |
|---|---|---|---|
| **Singleton** | Exactly one shared instance | Duplicates would be wrong (pool, config) | `db.js` exporting one pool |
| **Factory Method / Simple Factory** | Choosing which class to create at runtime; hiding construction details | `if/else` on a type to decide what to `new` | `createNotifier("sms")` |
| **Abstract Factory** | Creating consistent families of objects | Need to switch a whole set of implementations together | AWS vs in-memory infra |
| **Builder** | Readable, validated construction of complex objects | Many optional params; multi-step construction | Query builders, test data builders |
| **Prototype** | Cheap copies of preconfigured objects | Expensive creation, many similar objects | Document templates |
| **Object Pool** | Reusing expensive resources | Costly setup per use | DB connection pools |

---

## Quick self-test
- [ ] Why is Singleton often called an anti-pattern? What would you use instead, and how does it fix testing?
- [ ] How does Node's module system give you singleton behavior?
- [ ] Rewrite an `if/else` of `new X()` calls as a registry-based factory. Which SOLID principle improves?
- [ ] Factory Method vs Abstract Factory: what's the difference, with an example of each?
- [ ] When is a plain options object enough in JS, and when would you still use a Builder?
- [ ] What's the shallow-copy bug with Prototype?
- [ ] Why does connection pool size matter, both too small and too large?

Next: [[Structural Patterns]]
