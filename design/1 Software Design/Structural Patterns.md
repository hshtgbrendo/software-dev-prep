---
tags: [design, design-patterns, structural]
aliases: [Adapter, Decorator, Facade, Proxy, Composite, Bridge, Flyweight]
---
# Structural Design Patterns

> [!summary]
> Structural patterns are about **how classes and objects are composed** into larger structures while keeping them flexible. Several of these look alike (Adapter, Decorator, Proxy all "wrap" something), so learn the **intent** that distinguishes them.

Back to [[Design Index]] · Prev: [[Creational Patterns]]

---

## Adapter

**Intent:** Convert one interface into another that clients expect. A "translator" or plug adapter.

```javascript
// Our app expects: paymentGateway.pay(amountCents, currency)
// The new vendor SDK has: vendor.makeCharge({ value: dollars, cur })
class VendorPaymentAdapter {
  constructor(vendorSdk) { this.vendor = vendorSdk; }
  pay(amountCents, currency) {
    return this.vendor.makeCharge({ value: amountCents / 100, cur: currency });
  }
}
```

**Real uses:** wrapping third-party SDKs, legacy system integration, the "adapters" in [[Architectural Styles#Hexagonal / Ports & Adapters|Hexagonal Architecture]].

---

## Decorator

**Intent:** Attach **additional behavior to an object dynamically** by wrapping it, without changing its class. Wrappers share the same interface, so they can be stacked.

```javascript
const baseFetch = (url) => fetch(url);

const withLogging = (fn) => async (url) => {
  console.time(url); const r = await fn(url); console.timeEnd(url); return r;
};
const withRetry = (fn, n = 3) => async (url) => {
  for (let i = 0; i < n; i++) { try { return await fn(url); } catch (e) { if (i === n - 1) throw e; } }
};

const robustFetch = withLogging(withRetry(baseFetch));   // stack behaviors
```

**Real uses:** Express/Koa middleware, Java I/O streams (`new BufferedReader(new FileReader(f))`), Python `@decorators`, React higher-order components, TypeScript/NestJS decorators.

**Decorator vs inheritance:** decorators combine at runtime in any order, avoiding a subclass explosion (`LoggingRetryingCachingClient`).

---

## Facade

**Intent:** Provide a **simple, unified interface** to a complex subsystem.

```javascript
class CheckoutFacade {
  constructor(inventory, payments, shipping, email) { Object.assign(this, { inventory, payments, shipping, email }); }
  async placeOrder(cart, user) {
    await this.inventory.reserve(cart.items);
    const charge = await this.payments.charge(user, cart.total);
    const label = await this.shipping.createLabel(user.address, cart.items);
    await this.email.sendConfirmation(user, charge, label);
    return { chargeId: charge.id, tracking: label.tracking };
  }
}
```

**Real uses:** service layer methods, SDKs over raw REST APIs, jQuery over the DOM, an **API Gateway** is a facade at system level.

---

## Proxy

**Intent:** Provide a **stand-in** that controls access to another object. Same interface as the real object.

Flavors:
- **Virtual proxy**: lazy-load an expensive object on first use.
- **Protection proxy**: access control / auth checks.
- **Remote proxy**: local object representing something over the network (gRPC stubs).
- **Caching proxy**: return cached results.

```javascript
// JS has a built-in Proxy!
const user = { name: "Brendan", role: "user" };
const guarded = new Proxy(user, {
  set(target, prop, value) {
    if (prop === "role") throw new Error("role is read-only");
    target[prop] = value; return true;
  }
});
```

**Real uses:** ORM lazy-loading (Hibernate), Vue 3 reactivity (built on JS `Proxy`), reverse proxies (Nginx) at system scale.

> [!question] Adapter vs Decorator vs Proxy vs Facade
> - **Adapter**: *different* interface → make it match.
> - **Decorator**: *same* interface → *add* behavior.
> - **Proxy**: *same* interface → *control access* to the real object.
> - **Facade**: *new, simpler* interface over *many* objects.

---

## Composite

**Intent:** Compose objects into **tree structures** and let clients treat individual objects and groups the same way.

```javascript
class File   { constructor(n, size){ this.n=n; this.s=size; } size(){ return this.s; } }
class Folder {
  constructor(n){ this.n=n; this.children=[]; }
  add(c){ this.children.push(c); return this; }
  size(){ return this.children.reduce((t, c) => t + c.size(), 0); }  // same method!
}
const root = new Folder("root").add(new File("a.txt", 10)).add(new Folder("img").add(new File("b.png", 200)));
root.size(); // 210
```

**Real uses:** file systems, the DOM, React component trees, org charts, menus, expression trees.

---

## Bridge

**Intent:** Split a large class into two separate hierarchies, **abstraction** and **implementation**, that vary independently.

Example: `Shape` (Circle, Square) × `Renderer` (SVG, Canvas). Without Bridge: `SvgCircle`, `CanvasCircle`, `SvgSquare`, ... (N×M classes). With Bridge: `Shape` *has a* `Renderer` (N+M classes).

**Real uses:** JDBC drivers (API abstraction vs vendor driver), cross-platform UI.

---

## Flyweight

**Intent:** Share common state between many objects to **save memory**.

Split state into **intrinsic** (shared, immutable: a tree's mesh/texture) and **extrinsic** (unique: x, y position). Store intrinsic once.

**Real uses:** characters/glyphs in text editors, particles/trees in games, string interning in Java/JS.

---

## Summary table

| Pattern | Key idea | Think of |
|---|---|---|
| Adapter | Convert interface | Travel plug adapter |
| Decorator | Add behavior, stackable | Middleware |
| Facade | Simplify a subsystem | `placeOrder()` |
| Proxy | Control access | Bouncer / lazy loader |
| Composite | Tree of uniform nodes | File system |
| Bridge | Two independent hierarchies | Shape × Renderer |
| Flyweight | Share state to save memory | Glyphs |

## Self-test
- [ ] What distinguishes Decorator from Proxy if both wrap with the same interface?
- [ ] Where have you used Adapter in your own projects? (Hint: wrapping a third-party API like Twitch or Stripe.)
- [ ] Why is the DOM an example of Composite?

Next: [[Behavioral Patterns]]
