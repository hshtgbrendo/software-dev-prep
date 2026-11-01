---
tags: [design, design-patterns, structural]
aliases: [Adapter, Decorator, Facade, Proxy, Composite, Bridge, Flyweight]
---
# Structural Design Patterns

> [!summary] What you'll learn
> Structural patterns are about **how classes and objects are combined into larger structures** while keeping those structures flexible and loosely coupled. Several of them look alike in code (Adapter, Decorator, and Proxy all "wrap" another object), so the key skill is recognizing the **intent**: *why* you're wrapping.
>
> Patterns covered: **Adapter, Decorator, Facade, Proxy, Composite, Bridge, Flyweight**.

Back to [[Design Index]] · Prev: [[Creational Patterns]] · Next: [[Behavioral Patterns]]

---

## Adapter

### The problem
You have code that expects one interface, and a component (a third-party SDK, a legacy module, a different service) that provides **the right functionality with the wrong interface**. You can't (or shouldn't) change either side.

Examples:
- Your app calls `paymentGateway.pay(amountCents, currency)`, but the new vendor's SDK has `vendor.createCharge({ value: dollars, cur })`.
- Your logging code expects `logger.info(msg)`, but a library only supports `console.log`-style or Winston-style calls.
- An old XML-based internal service needs to work with your JSON-based code.

### How it works
Write a small **adapter** class that implements the interface your code expects and **translates** each call into the other component's interface. Like a travel plug adapter: same electricity, different shape.

```mermaid
flowchart LR
  Client[Checkout code] -->|"pay(cents, currency)"| A[VendorPaymentAdapter]
  A -->|"createCharge({value, cur})"| V[Vendor SDK]
```

```javascript
// What our application expects (the "target" interface):
//   pay(amountCents, currency) → Promise<{ id, status }>

class AcmePayAdapter {
  constructor(acmeSdk) { this.acme = acmeSdk; }

  async pay(amountCents, currency) {
    // translate inputs: cents → dollars, our param names → theirs
    const res = await this.acme.createCharge({ value: amountCents / 100, cur: currency.toLowerCase() });
    // translate outputs: their shape → ours
    return { id: res.charge_ref, status: res.ok ? "succeeded" : "failed" };
  }
}

// Swapping providers is now a one-line change at wiring time
const gateway = new AcmePayAdapter(new AcmeSDK(apiKey));
await checkout(cart, gateway);
```

### When to use it
- Integrating **third-party libraries/APIs** whose interface you don't control. (Wrapping a vendor SDK in your own adapter is excellent practice: it isolates vendor-specific details and makes switching or faking easy.)
- **Legacy integration**: making old code work with a new interface without rewriting it.
- Normalizing several similar services (Stripe, PayPal, Square) behind one interface.

### Where you've seen it
- The **"adapters"** in [[Architectural Styles#Hexagonal / Ports & Adapters|Hexagonal Architecture]].
- ORMs adapting different databases (Postgres, MySQL, SQLite) to one API.
- Axios adapters (XHR in browsers, `http` in Node).
- At system scale: the **Anti-Corruption Layer** in [[Domain-Driven Design]].

---

## Decorator

### The problem
You want to **add responsibilities to an object** (logging, caching, retries, authorization, compression) **without modifying its class** and without creating a subclass for every combination.

With inheritance, combinations explode: `LoggingHttpClient`, `RetryingHttpClient`, `LoggingRetryingHttpClient`, `CachingLoggingRetryingHttpClient`... And you can't add or remove behaviors at runtime.

### How it works
A **decorator** wraps an object and implements **the same interface**. It forwards calls to the wrapped object, adding behavior before and/or after. Because decorators share the interface, they can be **stacked** in any order.

```mermaid
flowchart LR
  Caller --> L[Logging decorator] --> R[Retry decorator] --> C[Cache decorator] --> Real[Real HTTP client]
```

Class-based version:
```javascript
class HttpClient {
  async get(url) { const r = await fetch(url); return r.json(); }
}

class LoggingClient {
  constructor(inner) { this.inner = inner; }
  async get(url) {
    const start = Date.now();
    try { return await this.inner.get(url); }
    finally { console.log(`GET ${url} took ${Date.now() - start}ms`); }
  }
}

class CachingClient {
  constructor(inner, ttlMs = 60_000) { this.inner = inner; this.cache = new Map(); this.ttl = ttlMs; }
  async get(url) {
    const hit = this.cache.get(url);
    if (hit && hit.expires > Date.now()) return hit.value;
    const value = await this.inner.get(url);
    this.cache.set(url, { value, expires: Date.now() + this.ttl });
    return value;
  }
}

// Compose behaviors at runtime, in any order
const client = new LoggingClient(new CachingClient(new HttpClient()));
```

Function-based version (very idiomatic in JS, called **higher-order functions**):
```javascript
const withRetry = (fn, attempts = 3) => async (...args) => {
  for (let i = 1; ; i++) {
    try { return await fn(...args); }
    catch (err) { if (i >= attempts) throw err; }
  }
};
const withTiming = (fn, label) => async (...args) => {
  console.time(label);
  try { return await fn(...args); } finally { console.timeEnd(label); }
};

const fetchUser = withTiming(withRetry((id) => api.get(`/users/${id}`)), "fetchUser");
```

### When to use it
- Adding **cross-cutting concerns** (logging, metrics, caching, retries, auth checks, validation) to existing objects.
- You need **combinations** of optional behaviors.
- You want to add or remove behavior **at runtime** or via configuration.

### Where you've seen it
- **Express/Koa middleware** (each layer wraps the next).
- **React Higher-Order Components** (`withRouter(Component)`), and conceptually hooks.
- **Python `@decorator`** and **TypeScript/NestJS decorators** (`@Get()`, `@UseGuards()`).
- Java I/O: `new BufferedReader(new InputStreamReader(new FileInputStream(file)))`.

### Downsides
- Many small wrappers can make debugging stack traces harder.
- **Order matters**: caching outside retry vs inside retry behaves differently.
- Identity checks break: the decorated object isn't `===` the original.

---

## Facade

### The problem
A task requires coordinating **many classes or subsystems** in the right order with the right details. If every caller does that coordination itself, the knowledge is duplicated, callers are coupled to every subsystem, and changing one subsystem breaks many callers.

### How it works
Provide **one simple, high-level interface** that hides the complex subsystem behind it. The facade doesn't add new functionality; it **simplifies and orchestrates** existing functionality.

```javascript
class CheckoutFacade {
  constructor({ inventory, payments, shipping, email, orders }) {
    Object.assign(this, { inventory, payments, shipping, email, orders });
  }

  // One call instead of five subsystems that must be called in the right order
  async placeOrder(user, cart) {
    const reservation = await this.inventory.reserve(cart.items);
    try {
      const charge = await this.payments.charge(user.paymentMethodId, cart.totalCents);
      const order = await this.orders.create({ userId: user.id, items: cart.items, chargeId: charge.id });
      const label = await this.shipping.createLabel(user.address, cart.items);
      await this.email.sendConfirmation(user.email, order, label.trackingNumber);
      return order;
    } catch (err) {
      await this.inventory.release(reservation);   // the facade also owns the error handling
      throw err;
    }
  }
}

// Controller code stays tiny
app.post("/checkout", async (req, res) => res.json(await checkout.placeOrder(req.user, req.body.cart)));
```

### When to use it
- Callers only need a **common subset** of a complex system's capabilities.
- You want to **layer** a system: higher layers talk to facades of lower layers.
- Wrapping a **messy legacy system** or verbose API behind a clean interface.

### Where you've seen it
- A **service layer** method in a backend (as above).
- SDKs that wrap raw REST APIs (`stripe.customers.create()`).
- jQuery as a facade over inconsistent browser DOM APIs.
- At system scale, an **API Gateway** or **Backend-for-Frontend** is a facade over many microservices.

### Downside
A facade can grow into a **God object** if everything gets routed through it. Keep facades focused on specific use cases.

---

## Proxy

### The problem
You need to **control access** to an object: delay its creation, check permissions, cache its results, count calls, or represent something that lives elsewhere (on another server). And you want to do this **without callers knowing**: they should use the proxy exactly as they'd use the real object.

### How it works
A proxy implements **the same interface** as the real object ("subject") and holds a reference to it. It intercepts calls and decides whether, when, and how to forward them.

### Kinds of proxy
| Kind | Purpose | Example |
|---|---|---|
| **Virtual proxy** | **Lazy-load** an expensive object only when first used | Image placeholder that loads the full image on scroll; ORM lazy-loaded relations |
| **Protection proxy** | **Access control** | Check the user's role before allowing `delete()` |
| **Remote proxy** | Local stand-in for an object on **another machine** | gRPC/RPC client stubs: `orderService.getOrder(id)` looks local but makes a network call |
| **Caching proxy** | Return cached results for repeated requests | Memoizing expensive API calls |
| **Logging/metrics proxy** | Record usage | Count calls per method |

```javascript
// Protection + logging proxy using JavaScript's built-in Proxy object
function protect(repo, user) {
  return new Proxy(repo, {
    get(target, prop) {
      const original = target[prop];
      if (typeof original !== "function") return original;
      return (...args) => {
        if (prop.startsWith("delete") && user.role !== "admin") {
          throw new Error(`Forbidden: ${user.id} cannot call ${prop}`);
        }
        console.log(`[audit] ${user.id} called ${prop}`);
        return original.apply(target, args);
      };
    },
  });
}

const repo = protect(new DocumentRepository(), currentUser);
await repo.findById(42);     // allowed + logged
await repo.deleteById(42);   // throws for non-admins
```

### Where you've seen it
- **Vue 3's reactivity system** is built on JS `Proxy` (intercepting property reads/writes to trigger re-renders). MobX and Immer use proxies too.
- ORM **lazy loading** (Hibernate, TypeORM relations).
- At system scale: **reverse proxies** (Nginx), CDNs, and service mesh sidecars control access to backend servers. See [[Networking Essentials]].

### Adapter vs Decorator vs Proxy vs Facade (a classic question)
They all wrap something. The **intent** distinguishes them:

| Pattern | Interface relative to the wrapped object | Purpose |
|---|---|---|
| **Adapter** | **Different**: converts to the interface the client expects | Make incompatible things work together |
| **Decorator** | **Same** | **Add** behavior; typically stacked; client chooses the wrapping |
| **Proxy** | **Same** | **Control access** (lazy, protect, remote, cache); often manages the real object's lifecycle |
| **Facade** | **New, simpler** interface over **many** objects | Simplify a complex subsystem |

> Decorator vs Proxy is the subtle one. Implementation can be nearly identical. A decorator *enhances* what the object does, and the client typically composes it. A proxy *controls whether/how* you reach the object, and often creates or manages the real object itself (e.g., a lazy proxy).

---

## Composite

### The problem
You have **tree-shaped data** where individual items ("leaves") and groups of items ("composites") should be treated the **same way**. Without the pattern, every operation must check "is this a single item or a group?" and recurse manually:

```javascript
function totalSize(node) {
  if (node.type === "file") return node.size;
  if (node.type === "folder") return node.children.reduce((t, c) => t + totalSize(c), 0);
}
// ...and the same branching in render(), search(), delete(), permissions()...
```

### How it works
Leaves and composites implement the **same interface**. A composite holds children (which may be leaves *or* other composites) and implements each operation by delegating to its children. Clients treat the whole tree uniformly.

```javascript
class File {
  constructor(name, size) { this.name = name; this.size = size; }
  getSize() { return this.size; }
  print(indent = "") { console.log(`${indent}📄 ${this.name} (${this.size}B)`); }
}

class Folder {
  constructor(name) { this.name = name; this.children = []; }
  add(child) { this.children.push(child); return this; }
  getSize() { return this.children.reduce((t, c) => t + c.getSize(), 0); }   // same method, recursive
  print(indent = "") {
    console.log(`${indent}📁 ${this.name}`);
    this.children.forEach(c => c.print(indent + "  "));
  }
}

const root = new Folder("root")
  .add(new File("readme.md", 120))
  .add(new Folder("img").add(new File("logo.png", 2048)).add(new File("bg.jpg", 4096)));

root.getSize();  // 6264: the client doesn't care whether root is a file or folder
```

### When to use it
Any **part–whole hierarchy**: file systems, UI component trees, org charts, nested menus/comments, product bundles (a bundle contains products or other bundles, and each has a price), arithmetic expression trees, permission groups containing users or other groups.

### Where you've seen it
The **DOM** (every node is a `Node`; elements contain nodes), **React's component tree**, JSON itself, scene graphs in games, ASTs in compilers.

---

## Bridge

### The problem
A class varies along **two independent dimensions**, and inheritance forces you to create a subclass for every combination.

Example: shapes (`Circle`, `Square`, `Triangle`) × rendering backends (`SVG`, `Canvas`, `WebGL`). With inheritance: `SvgCircle`, `CanvasCircle`, `WebGLCircle`, `SvgSquare`, ... = **N × M classes**. Adding one renderer means adding N new classes.

### How it works
Split the dimensions into **two separate hierarchies** and connect them with composition: the **abstraction** (Shape) holds a reference to an **implementation** (Renderer). Now you need **N + M classes**, and each side can evolve independently.

```javascript
// Implementation hierarchy
class SvgRenderer    { drawCircle(x, y, r) { return `<circle cx="${x}" cy="${y}" r="${r}"/>`; } }
class CanvasRenderer { drawCircle(x, y, r) { /* ctx.arc(...) */ } }

// Abstraction hierarchy
class Shape  { constructor(renderer) { this.renderer = renderer; } }   // the "bridge"
class Circle extends Shape {
  constructor(renderer, x, y, r) { super(renderer); Object.assign(this, { x, y, r }); }
  draw() { return this.renderer.drawCircle(this.x, this.y, this.r); }
}

new Circle(new SvgRenderer(), 10, 10, 5).draw();
new Circle(new CanvasRenderer(), 10, 10, 5).draw();
```

### When to use it
Two (or more) independent axes of variation: notification **type** × delivery **channel**; report **format** × data **source**; device **remote** × device **type**.

### Where you've seen it
**JDBC/database drivers**: your code uses the JDBC API (abstraction) while vendor drivers (implementation) vary independently. Logging facades (SLF4J) over logging backends (Logback, Log4j).

> Bridge is designed **up front** to let two dimensions vary. Adapter is applied **after the fact** to make two existing, incompatible things work together.

---

## Flyweight

### The problem
You need a **huge number of similar objects** (millions of characters in a document, trees or particles in a game, map markers), and storing full data in each exhausts memory.

### How it works
Split each object's state into:
- **Intrinsic state**: shared and immutable (a tree's 3D mesh and texture, a font glyph's shape). Stored **once** in a shared flyweight.
- **Extrinsic state**: unique per object (position, size). Stored per object, or passed in when needed.

A factory returns the **same** flyweight instance for the same intrinsic data.

```javascript
class TreeType {                         // flyweight (intrinsic, shared)
  constructor(species, texture, mesh) { Object.assign(this, { species, texture, mesh }); }
}

const treeTypes = new Map();
function getTreeType(species, texture, mesh) {
  const key = `${species}|${texture}`;
  if (!treeTypes.has(key)) treeTypes.set(key, new TreeType(species, texture, mesh));
  return treeTypes.get(key);             // reuse
}

// 1,000,000 trees but only a handful of TreeType objects
const forest = Array.from({ length: 1_000_000 }, (_, i) => ({
  x: Math.random() * 1000, y: Math.random() * 1000,           // extrinsic
  type: getTreeType("oak", "oak.png", "oak.obj"),
}));
```

### When to use it
Only when memory is a **measured** problem with very many fine-grained objects. Otherwise it adds complexity for nothing.

### Where you've seen it
String interning (Java's string pool, JS engines internally), text editors' glyph rendering, game engines (instanced rendering), icon sprites.

---

## Summary table

| Pattern | Problem it solves | Key idea | Think of |
|---|---|---|---|
| **Adapter** | Incompatible interfaces | Translate one interface to another | Travel plug adapter; wrapping a vendor SDK |
| **Decorator** | Adding behavior without subclass explosion | Wrap with the same interface; stackable | Express middleware |
| **Facade** | Complex subsystem, many callers | One simple entry point | `placeOrder()` service method |
| **Proxy** | Controlling access to an object | Same interface; intercept calls | Lazy loader, auth guard, RPC stub, Vue reactivity |
| **Composite** | Treating trees of items uniformly | Leaves and groups share an interface | File system, DOM |
| **Bridge** | N × M subclass explosion across two dimensions | Separate hierarchies joined by composition | Shape × Renderer; JDBC |
| **Flyweight** | Memory blow-up from millions of similar objects | Share intrinsic state | Glyphs, game trees |

## Quick self-test
- [ ] Explain the difference between Adapter, Decorator, Proxy, and Facade in terms of intent.
- [ ] Why does Decorator beat inheritance for adding optional behaviors? What problem does order cause?
- [ ] Wrap a third-party API from one of your projects (Stripe, Twitch) in an Adapter. What do you gain?
- [ ] Why is the DOM an example of Composite? Write `getSize()` for a folder tree.
- [ ] When would you use Bridge instead of inheritance? Count the classes saved for 4 shapes × 3 renderers.
- [ ] Name three kinds of Proxy and a real-world example of each.

Next: [[Behavioral Patterns]]
