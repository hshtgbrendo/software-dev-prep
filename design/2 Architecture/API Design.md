---
tags: [design, architecture, api]
aliases: [REST, GraphQL, gRPC, API, WebSockets]
---
# API Design

> [!summary] What you'll learn
> An **API (Application Programming Interface)** is a contract that lets one piece of software use another: your React app calling your backend, one microservice calling another, or a partner calling your public API. Good APIs are predictable, hard to misuse, and can evolve without breaking clients.
>
> This note covers **REST** in depth (resources, methods, status codes, pagination, idempotency, versioning, caching, errors), then **GraphQL**, **gRPC**, and **real-time options** (polling, SSE, WebSockets, webhooks), with when to use each.
>
> **Interview angle:** every system design interview has an "API design" step, and API design is reviewed in nearly every pull request that adds an endpoint.

Back to [[Design Index]] · Related: [[Architectural Styles]] · [[Networking Essentials]] · [[Security Fundamentals]]

---

## What makes an API good?
1. **Predictable**: consistent naming, structure, and behavior across endpoints. Once you learn one endpoint, you can guess the others.
2. **Hard to misuse**: clear errors, validation, safe retries.
3. **Evolvable**: can add features without breaking existing clients (mobile apps in the wild can't be force-updated).
4. **Efficient**: doesn't force clients into many round trips or huge payloads.
5. **Secure by default**: authentication and authorization on every endpoint.
6. **Documented**: clients shouldn't need to read your source code.

---

## REST (Representational State Transfer)

### What it is
An **architectural style** described by Roy Fielding in his 2000 dissertation, built on HTTP's semantics. The core idea: model your domain as **resources** (nouns, like users, orders, products), identify each with a **URL**, and manipulate them through a small, uniform set of **HTTP methods**.

### The REST constraints (and why they matter)
| Constraint | Meaning | Why it matters |
|---|---|---|
| **Client–server** | UI and data storage are separated | Each can evolve independently |
| **Stateless** | Each request contains everything needed to process it; the server keeps no client session state between requests | **Any server can handle any request**, so you can add servers behind a load balancer freely (horizontal scaling) |
| **Cacheable** | Responses declare whether they can be cached | Clients, CDNs, and proxies can reuse responses |
| **Uniform interface** | Standard methods, resource URLs, self-descriptive messages | Generic tooling works (browsers, proxies, caches) |
| **Layered system** | Clients can't tell if they're talking to the server or an intermediary | Load balancers, CDNs, and gateways can be inserted transparently |

"Stateless" doesn't mean the app has no state. It means **session state isn't stored in the server's memory**. Authentication travels with each request (a token or session cookie), and data lives in databases.

### Resource naming
```
GET    /users                     list users (paginated)
POST   /users                     create a user
GET    /users/123                 get user 123
PUT    /users/123                 replace user 123 entirely
PATCH  /users/123                 partially update user 123
DELETE /users/123                 delete user 123

GET    /users/123/orders          orders belonging to user 123 (sub-resource)
GET    /orders?status=shipped&sort=-createdAt&limit=20     filter, sort, paginate with query params
POST   /orders/456/cancel         an action that doesn't map cleanly to CRUD (acceptable)
```

**Rules of thumb:**
- Use **nouns, not verbs**: `/users`, not `/getUsers` or `/createUser`. The HTTP method is the verb.
- **Plural** collection names, lowercase, hyphen-separated (`/order-items`).
- Nest at most **one or two levels**. `/users/123/orders/456/items/7/reviews` is too deep; use `/reviews?itemId=7`.
- For actions that aren't CRUD ("cancel," "refund," "resend"), a sub-resource like `POST /orders/456/cancel` is pragmatic and widely used (Stripe and GitHub do this).
- Use query parameters for filtering, sorting, searching, field selection, and pagination.

### HTTP methods, safety, and idempotency
| Method | Purpose | Safe? | Idempotent? | Has a body? |
|---|---|---|---|---|
| **GET** | Read a resource or collection | ✅ | ✅ | No |
| **HEAD** | Like GET, headers only | ✅ | ✅ | No |
| **OPTIONS** | Which methods are allowed (used by CORS preflight) | ✅ | ✅ | No |
| **POST** | Create a resource, or trigger an action | ❌ | ❌ | Yes |
| **PUT** | **Replace** the entire resource at a known URL (or create it there) | ❌ | ✅ | Yes |
| **PATCH** | **Partially** update a resource | ❌ | ❌ (not guaranteed) | Yes |
| **DELETE** | Delete a resource | ❌ | ✅ | Usually no |

- **Safe** = no side effects on the server (just reading). Crawlers, prefetchers, and caches assume GET is safe, so **never change data on GET**.
- **Idempotent** = making the same request N times has the same effect as making it once. `PUT /users/1 {name: "Bo"}` twice leaves the same state. `DELETE /users/1` twice: the user is gone either way (the second call may return 404, but the *state* is the same). `POST /orders` twice creates **two** orders, so POST is not idempotent.
- Why idempotency matters: networks fail, and clients, proxies, and load balancers retry. Idempotent operations are safe to retry automatically. For POST, use idempotency keys (below).

**PUT vs PATCH:**
```http
PUT /users/123            → body must be the FULL user; missing fields are cleared/defaulted
{ "name": "Brendan", "email": "b@x.com", "bio": "" }

PATCH /users/123          → only the fields you want to change
{ "bio": "Fullstack dev" }
```

### Status codes (know these cold)
| Code | Name | When to use |
|---|---|---|
| **200** | OK | Successful GET, PUT, PATCH (with a body) |
| **201** | Created | Successful POST that created something. Include a `Location: /orders/789` header and/or the created object |
| **202** | Accepted | Request accepted for **asynchronous** processing (a report generation job); return a job ID to poll |
| **204** | No Content | Success with no body (often DELETE or PUT) |
| **301 / 308** | Moved Permanently | Resource has a new permanent URL (308 preserves the method) |
| **302 / 307** | Found / Temporary Redirect | Temporary redirect (307 preserves the method) |
| **304** | Not Modified | The client's cached copy is still valid (conditional GET with ETag) |
| **400** | Bad Request | Malformed request or validation failure |
| **401** | Unauthorized | **Not authenticated**: missing, invalid, or expired credentials ("who are you?") |
| **403** | Forbidden | **Authenticated but not allowed** ("I know who you are, and you can't do this") |
| **404** | Not Found | Resource doesn't exist (also used to hide existence from unauthorized users) |
| **405** | Method Not Allowed | e.g., DELETE on a read-only resource |
| **409** | Conflict | Request conflicts with current state: duplicate email, version mismatch, invalid state transition |
| **410** | Gone | Resource intentionally and permanently removed |
| **412** | Precondition Failed | `If-Match` ETag didn't match (someone else changed it). Used for optimistic concurrency |
| **415** | Unsupported Media Type | Wrong `Content-Type` |
| **422** | Unprocessable Content | Well-formed but semantically invalid (some APIs use this for validation errors instead of 400) |
| **429** | Too Many Requests | Rate limited. Include a `Retry-After` header |
| **500** | Internal Server Error | Unhandled server bug. Never intentionally return this |
| **502** | Bad Gateway | A proxy/gateway got an invalid response from upstream |
| **503** | Service Unavailable | Overloaded or down for maintenance (temporary; may include `Retry-After`) |
| **504** | Gateway Timeout | A proxy/gateway timed out waiting for upstream |

**Rule:** 4xx means the **client** must change something; 5xx means the **server** failed and the request might succeed on retry. Retry logic depends on this distinction.

### Error responses
Return a **consistent, machine-readable** error body so clients can handle errors programmatically. A common standard is **RFC 9457 "Problem Details for HTTP APIs"**:
```json
HTTP/1.1 409 Conflict
Content-Type: application/problem+json

{
  "type": "https://api.example.com/errors/insufficient-funds",
  "title": "Insufficient funds",
  "status": 409,
  "detail": "Account balance is $30.00 but the transfer requires $50.00.",
  "instance": "/transfers/abc123",
  "traceId": "4bf92f3577b34da6"
}
```
For validation errors, include per-field details: `"errors": [{ "field": "email", "message": "must be a valid email" }]`. **Never leak stack traces or internal details** in production errors (that's a security risk).

### Pagination
Any endpoint returning a list **must** be paginated. Otherwise one day it returns a million rows and takes down your server.

**Offset/limit pagination:**
```
GET /products?limit=20&offset=40        → items 41–60
SELECT * FROM products ORDER BY id LIMIT 20 OFFSET 40;
```
- ✅ Simple; lets users jump to page N; easy to show "page 3 of 50."
- ❌ **Slow on deep pages**: the database still reads and discards the first `offset` rows (`OFFSET 1000000` scans a million rows).
- ❌ **Unstable under changes**: if an item is inserted while you're paging, items shift, so you see duplicates or miss some.

**Cursor (keyset) pagination:**
```
GET /products?limit=20                         → { items: [...], nextCursor: "eyJpZCI6MTIzfQ" }
GET /products?limit=20&cursor=eyJpZCI6MTIzfQ    → next page

-- cursor decodes to { id: 123 }: "give me items after the last one I saw"
SELECT * FROM products WHERE id > 123 ORDER BY id LIMIT 20;
```
- ✅ **Fast at any depth** (uses the index to jump straight to the position).
- ✅ **Stable** when items are added or removed.
- ❌ No jumping to arbitrary pages; harder to implement with complex sort orders (the cursor must encode all sort keys, e.g., `(createdAt, id)`).
- The cursor is **opaque** (base64-encoded) so clients don't depend on its format.

> **Use cursor pagination** for feeds, timelines, infinite scroll, large datasets, and real-time data. Twitter/X, Facebook, Slack, and Stripe use it. Offset is fine for small admin tables with page numbers.

### Idempotency
**The problem:** a client sends `POST /payments` to charge $50. The server charges the card, but the response is lost (network timeout). The client doesn't know if it worked, so it retries, and the customer is **charged twice**.

**The solution: idempotency keys.** The client generates a unique key (UUID) per logical operation and sends it with the request. The server records the key with the result. If the same key arrives again, the server returns the stored result instead of performing the operation again.
```http
POST /v1/payments
Idempotency-Key: 9b2c5f1e-8d3a-4c7b-a1f2-6e8d9c0b1a2f
Content-Type: application/json

{ "amountCents": 5000, "currency": "USD", "customerId": "cus_42" }
```

```javascript
// Server-side sketch
app.post("/v1/payments", async (req, res) => {
  const key = req.get("Idempotency-Key");
  if (!key) return res.status(400).json({ error: "Idempotency-Key header required" });

  const existing = await idempotencyStore.get(key);           // e.g., Redis or a DB table with a unique key
  if (existing?.status === "completed") return res.status(existing.code).json(existing.body);
  if (existing?.status === "in_progress") return res.status(409).json({ error: "Request in progress" });

  await idempotencyStore.create(key, { status: "in_progress" }, { ttlHours: 24 });  // atomic insert-if-absent
  const charge = await payments.charge(req.body);
  const body = { id: charge.id, status: charge.status };
  await idempotencyStore.complete(key, { code: 201, body });
  res.status(201).json(body);
});
```
Stripe popularized this pattern. More on idempotency in [[Distributed Systems Patterns#Idempotency]].

### Versioning
Once clients depend on your API, **breaking changes** break them. Mobile apps especially can't be updated instantly.

**Breaking vs non-breaking changes:**
| ✅ Non-breaking (safe) | ❌ Breaking |
|---|---|
| Adding a new endpoint | Removing or renaming an endpoint or field |
| Adding an **optional** request field | Making an optional field required |
| Adding a response field (clients should ignore unknown fields) | Changing a field's type or format (`"price": "9.99"` → `999`) |
| Adding a new enum value (*if* clients handle unknowns) | Changing the meaning of a field or status code |

**Versioning strategies:**
- **URL path**: `/v1/users`, `/v2/users`. Most common, most explicit, easy to route and cache.
- **Header**: `Accept: application/vnd.myapi.v2+json`, or a custom `Api-Version: 2026-10-01` (Stripe uses date-based versions pinned per account).
- **Query param**: `?version=2`. Less common.

**Deprecation:** announce in advance, mark deprecated fields in docs, return `Deprecation` and `Sunset` headers, monitor who still calls old versions, then remove.

### Caching headers
HTTP has caching built in, which is one of REST's big advantages:
- `Cache-Control: public, max-age=3600`: any cache (browser, CDN) may store it for an hour.
- `Cache-Control: private, max-age=60`: only the user's browser may cache it (user-specific data).
- `Cache-Control: no-store`: never cache (sensitive data).
- **ETag** (a version fingerprint of the resource) + `If-None-Match`: the client asks "has it changed since version X?" If not, the server returns **304 Not Modified** with no body, which saves bandwidth.
- **ETag + `If-Match`** on writes: "only update if it's still version X," otherwise **412**. This prevents lost updates when two people edit the same thing (optimistic concurrency).

### HATEOAS and the Richardson Maturity Model
**HATEOAS** (Hypermedia as the Engine of Application State): responses include links to related resources and available actions, so clients navigate the API like a website.
```json
{ "id": 456, "status": "paid", "_links": { "self": "/orders/456", "cancel": "/orders/456/cancel", "customer": "/users/123" } }
```
The **Richardson Maturity Model** ranks APIs: Level 0 (one endpoint, RPC over HTTP), Level 1 (resources), Level 2 (HTTP verbs + status codes, where most "REST" APIs live), Level 3 (hypermedia). Rarely implemented fully in practice; just know the terms.

### Documentation: OpenAPI
**OpenAPI** (formerly Swagger) is a YAML/JSON spec describing endpoints, parameters, request/response schemas, and auth. From one spec you can generate interactive docs (Swagger UI, Redoc), client SDKs, server stubs, and request validation. Many large companies practice **API-first (contract-first) design**: write and review the spec before implementing.

---

## GraphQL

### The problem it solves
With REST, clients often **over-fetch** (the endpoint returns 40 fields, the mobile screen needs 3) or **under-fetch** (rendering a profile page requires `/users/1`, then `/users/1/posts`, then `/posts/*/comments`: many round trips). Each new screen tends to need a new or modified endpoint.

### How it works
GraphQL (open-sourced by Facebook in 2015) exposes **one endpoint** (`POST /graphql`) and a **strongly typed schema**. Clients send a **query describing exactly the data shape they want**, and the server returns exactly that.

```graphql
# Schema (server side)
type User  { id: ID!  name: String!  avatarUrl: String  posts(last: Int): [Post!]! }
type Post  { id: ID!  title: String!  likes: Int!  comments: [Comment!]! }
type Query { user(id: ID!): User }
type Mutation { createPost(title: String!, body: String!): Post! }
```
```graphql
# Query (client side): one round trip, exactly these fields
query ProfilePage {
  user(id: "123") {
    name
    avatarUrl
    posts(last: 3) { title likes }
  }
}
```
```json
{ "data": { "user": { "name": "Brendan", "avatarUrl": "...", "posts": [{ "title": "Keycaps 101", "likes": 12 }] } } }
```
- **Queries** read, **mutations** write, **subscriptions** push real-time updates (usually over WebSockets).
- The server implements **resolvers**: one function per field that knows how to fetch it.

### The N+1 problem and DataLoader
A naive resolver for `posts → author` runs one query for the posts, then **one query per post** for its author (N+1 queries). The standard fix is **DataLoader**: it collects all the author IDs requested during one tick and fetches them in a single batched query (`WHERE id IN (...)`), with per-request caching.

### Pros and cons
| ✅ Pros | ❌ Cons |
|---|---|
| Clients get exactly what they need, in one round trip | **HTTP caching is harder**: everything is POST to one URL (use persisted queries + GET, or client caches like Apollo) |
| Strong typing, introspection, excellent tooling (autocomplete, codegen) | **N+1 queries** unless you use DataLoader |
| Great when many clients (web, iOS, Android) need different shapes | Malicious or accidental **expensive queries** (deeply nested). Need depth/complexity limits and timeouts |
| Evolve without versions: add fields, deprecate old ones | Rate limiting by "request" doesn't work well; need cost-based limiting |
| Aggregates many backend services behind one schema (federation) | More server complexity; error handling always returns 200 with an `errors` array |

### When to use it
Product APIs with **many different clients and screens**, rapidly evolving frontends, and aggregating data from many services (GitHub, Shopify, and Meta use it). Less compelling for simple CRUD, server-to-server calls, or public APIs where HTTP caching matters.

---

## gRPC

### The problem it solves
Service-to-service calls inside a company happen **millions of times per second**. JSON over HTTP/1.1 is verbose to parse, loosely typed (typos in field names fail at runtime), and lacks built-in streaming.

### How it works
gRPC (open-sourced by Google, 2015) is a **Remote Procedure Call** framework. You define services and messages in **Protocol Buffers** (`.proto` files), and code generators produce typed client and server code in many languages. Messages are serialized in a compact **binary** format and sent over **HTTP/2**.

```protobuf
syntax = "proto3";

service OrderService {
  rpc GetOrder (GetOrderRequest) returns (Order);                  // unary
  rpc WatchOrderStatus (GetOrderRequest) returns (stream OrderStatus);  // server streaming
}

message GetOrderRequest { string order_id = 1; }
message Order {
  string id = 1;
  string user_id = 2;
  int64 total_cents = 3;
  repeated LineItem items = 4;
}
```
```javascript
// Generated client: looks like a local function call (a "remote proxy")
const order = await orderClient.getOrder({ orderId: "o_123" });
```
- The numbers (`= 1`, `= 2`) are **field tags** used in the binary encoding. Never reuse or change them; that's how backward compatibility works.
- Four call types: **unary**, **server streaming**, **client streaming**, **bidirectional streaming**.
- Built-in deadlines/timeouts, cancellation, metadata (headers), and status codes.

### Pros and cons
| ✅ Pros | ❌ Cons |
|---|---|
| Fast and compact (binary, HTTP/2 multiplexing) | Not human-readable; you need tools (grpcurl) to debug |
| Strong contracts and generated clients in many languages | **Browsers can't call it directly**: needs gRPC-Web plus a proxy (Envoy) |
| Streaming in both directions | Steeper learning curve; .proto files must be shared and versioned |
| Deadlines and cancellation propagate across services | Less useful for public APIs |

### When to use it
**Internal microservice-to-microservice communication**, polyglot environments, low-latency or high-throughput paths, and streaming. Google, Netflix, Square, and Uber use it heavily internally.

---

## Real-time & push options

HTTP's request/response model means the **client** always initiates. When the **server** has new data (a chat message, a price change), you need a technique to get it to the client.

### Short polling
The client asks every N seconds: `GET /notifications?since=...`.
- ✅ Trivial to implement; works everywhere.
- ❌ Wasteful (most responses are "nothing new"); latency up to N seconds; heavy load at scale.
- **Use for:** low-frequency updates where a few seconds of delay is fine (checking job status).

### Long polling
The client sends a request; the server **holds it open** until there's new data or a timeout (e.g., 30 s), then responds; the client immediately sends another.
- ✅ Near-real-time over plain HTTP; works through most proxies.
- ❌ Each waiting client holds a connection; reconnect overhead; awkward ordering.
- **Use for:** a fallback when WebSockets aren't available; simpler real-time needs.

### Server-Sent Events (SSE)
The client opens one long-lived HTTP connection (`EventSource` in the browser) and the server **streams** text events down it.
```javascript
const events = new EventSource("/api/orders/123/status");
events.onmessage = (e) => updateUI(JSON.parse(e.data));   // auto-reconnects
```
- ✅ Simple, built into browsers, automatic reconnection with last-event ID, works over normal HTTP (HTTP/2 friendly).
- ❌ **One-way** (server → client only); text only.
- **Use for:** notifications, live scores, dashboards, progress updates, **streaming LLM responses token by token** (ChatGPT-style UIs use SSE).

### WebSockets
After an HTTP "upgrade" handshake, the client and server keep a **persistent, full-duplex** TCP connection where **either side can send messages at any time** with very low overhead.
```javascript
const ws = new WebSocket("wss://chat.example.com/rooms/42");
ws.onmessage = (e) => renderMessage(JSON.parse(e.data));
ws.send(JSON.stringify({ type: "message", text: "hello" }));
```
- ✅ True bidirectional, low-latency messaging.
- ❌ **Stateful connections** make scaling harder: each server holds many open connections, load balancers must support long-lived connections, and you need a way to route a message to whichever server holds the recipient's connection. Reconnection logic is your job. Some corporate proxies interfere.
- **Use for:** chat, multiplayer games, collaborative editing (Google Docs, Figma), live trading, Twitch-style chat. See [[Design a Chat System]].

### Webhooks
**Server-to-server push**: when an event happens in service A (Stripe: "payment succeeded"; GitHub: "push received"), A sends an HTTP POST to a URL that service B registered in advance.
```http
POST https://yourapp.com/webhooks/stripe
Stripe-Signature: t=1700000000,v1=5257a869e7...
{ "id": "evt_123", "type": "payment_intent.succeeded", "data": { ... } }
```
**Webhook best practices** (both sending and receiving):
- **Verify signatures** (HMAC of the payload with a shared secret) so attackers can't forge events.
- **Respond 2xx quickly**, then process asynchronously (enqueue the event). Slow handlers cause timeouts and retries.
- **Expect duplicates and out-of-order delivery**: senders retry on failure. Dedupe by event ID (make handlers idempotent).
- Senders should retry with exponential backoff and offer a way to replay events.

### Comparison
| Technique | Direction | Latency | Complexity | Best for |
|---|---|---|---|---|
| Short polling | Client pulls | Up to the poll interval | Lowest | Infrequent status checks |
| Long polling | Client pulls (held) | Low | Low–medium | Fallback real-time |
| SSE | Server → client | Low | Low | Notifications, feeds, LLM streaming |
| WebSockets | Both ways | Lowest | Highest | Chat, games, collaboration |
| Webhooks | Server → another server | Low | Medium | Third-party integrations |

---

## REST vs GraphQL vs gRPC: choosing

| | REST | GraphQL | gRPC |
|---|---|---|---|
| Data format | JSON (text) | JSON | Protocol Buffers (binary) |
| Transport | HTTP/1.1 or HTTP/2 | HTTP | HTTP/2 |
| Contract | Optional (OpenAPI) | Required schema | Required .proto |
| Fetching | Server decides the response shape | Client decides the shape | Server decides (typed messages) |
| Caching | Excellent (HTTP caching, CDNs) | Harder | Custom |
| Browser support | Native | Native | Needs gRPC-Web proxy |
| Streaming | Not built in (use SSE/WebSockets) | Subscriptions | Built-in, bidirectional |
| Best for | Public APIs, CRUD, cacheable resources | Many clients with varied data needs; aggregation | Internal microservices, low latency, streaming |

A common real-world setup: **REST or GraphQL at the edge** (for browsers, mobile, partners) and **gRPC internally** between services.

---

## API design in a system design interview

After clarifying requirements, define 3–5 core endpoints **before** drawing architecture. It forces clarity about what the system actually does.

```
POST /v1/tweets                     body: { text, mediaIds[] }       → 201 { tweetId, createdAt }
GET  /v1/feed?cursor=&limit=20                                       → 200 { tweets[], nextCursor }
POST /v1/users/{userId}/follow                                       → 204
POST /v1/media/upload-url           body: { contentType, size }      → 200 { uploadUrl, mediaId }
```

Things to mention briefly: **authentication** (a bearer token identifies the user, so `userId` isn't a body param for "my" actions), **cursor pagination** for lists, **idempotency keys** for creates and payments, **rate limiting**, and versioning (`/v1`).

---

## API design checklist (for work and code reviews)
- [ ] Resource-oriented URLs; consistent naming and casing (`camelCase` JSON is most common in JS ecosystems)
- [ ] Correct methods and status codes; never mutate on GET
- [ ] Input validation with clear, consistent error bodies
- [ ] **Every list endpoint is paginated** (prefer cursors for large or changing data)
- [ ] Idempotency for non-idempotent operations that clients may retry (payments, orders)
- [ ] Authentication on every endpoint; authorization checks per resource (can *this* user access *this* order?)
- [ ] Rate limiting ([[Design a Rate Limiter]])
- [ ] Versioning strategy; no breaking changes to existing versions
- [ ] Timestamps as ISO 8601 in UTC (`2026-11-01T08:00:00Z`)
- [ ] **Money as integer minor units** (cents) plus a currency code, never floats (`0.1 + 0.2 !== 0.3`)
- [ ] IDs as opaque strings (lets you change ID formats later)
- [ ] Documented in OpenAPI with examples
- [ ] Don't leak internal fields (password hashes, internal flags). Map to response DTOs

---

## Quick self-test
- [ ] What does "stateless" mean in REST, and why does it enable horizontal scaling?
- [ ] PUT vs PATCH vs POST, and which are idempotent.
- [ ] 401 vs 403; 400 vs 409 vs 422; 502 vs 503 vs 504.
- [ ] Why is offset pagination slow on deep pages, and how does cursor pagination fix it?
- [ ] Walk through how an idempotency key prevents a double charge.
- [ ] List three breaking and three non-breaking API changes.
- [ ] What problem does GraphQL solve? What's the N+1 problem, and how does DataLoader fix it?
- [ ] When would you choose gRPC over REST?
- [ ] SSE vs WebSockets: when would you pick each? What makes WebSockets harder to scale?
- [ ] How do you secure and make a webhook receiver reliable?

Next: [[Domain-Driven Design]]
