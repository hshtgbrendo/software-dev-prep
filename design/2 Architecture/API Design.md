---
tags: [design, architecture, api]
aliases: [REST, GraphQL, gRPC, API]
---
# API Design

> [!summary]
> APIs are the contracts between your services, your frontend, and the outside world. Every system design interview includes an **"API design" step**. At work you'll write and review API specs constantly. Know REST deeply, and know when GraphQL, gRPC, WebSockets, or webhooks fit better.

Back to [[Design Index]] · Related: [[Architectural Styles]] · [[Networking Essentials]] · [[Security Fundamentals]]

---

## REST (Representational State Transfer)

An architectural style (Roy Fielding, 2000) using HTTP semantics around **resources**.

**Constraints:** client–server, **stateless** (each request carries all the info needed, so any server can handle it → easy horizontal scaling), cacheable, uniform interface, layered system, (optional) code-on-demand.

### Resource naming
```
GET    /users                 list users
POST   /users                 create user
GET    /users/123             get one
PUT    /users/123             replace (full update)
PATCH  /users/123             partial update
DELETE /users/123             delete
GET    /users/123/orders      sub-resource: orders of user 123
POST   /orders/456/cancel     action that doesn't map to CRUD (acceptable "controller" resource)
```
- **Nouns, plural**, lowercase, hyphenated (`/order-items`), no verbs (`/getUser` ❌).
- Nest at most 1–2 levels.
- Filtering/sorting via query params: `GET /orders?status=shipped&sort=-createdAt&limit=20`.

### HTTP methods & properties

| Method | Purpose | Safe? | Idempotent? |
|---|---|---|---|
| GET | Read | ✅ | ✅ |
| HEAD | Headers only | ✅ | ✅ |
| OPTIONS | Capabilities / CORS preflight | ✅ | ✅ |
| POST | Create / non-idempotent action | ❌ | ❌ |
| PUT | Replace entire resource | ❌ | ✅ |
| PATCH | Partial update | ❌ | ❌ (not guaranteed) |
| DELETE | Remove | ❌ | ✅ |

- **Safe** = no side effects. **Idempotent** = repeating gives the same result/state.

### Status codes (know these cold)

| Code | Meaning | When |
|---|---|---|
| **200** OK | Success | GET/PUT/PATCH success |
| **201** Created | Resource created | POST success (+ `Location` header) |
| **202** Accepted | Queued for async processing | Long jobs |
| **204** No Content | Success, empty body | DELETE |
| **301/302** | Moved permanently/temporarily | Redirects (see [[Design a URL Shortener]]) |
| **304** Not Modified | Use your cache | ETag / If-None-Match |
| **400** Bad Request | Malformed / validation error | |
| **401** Unauthorized | **Not authenticated** (who are you?) | Missing/invalid token |
| **403** Forbidden | **Authenticated but not allowed** | Wrong role |
| **404** Not Found | | |
| **405** Method Not Allowed | | |
| **409** Conflict | State conflict | Duplicate, version mismatch |
| **412** Precondition Failed | `If-Match` ETag mismatch | Optimistic locking |
| **422** Unprocessable Entity | Semantically invalid | Validation (some APIs use 400) |
| **429** Too Many Requests | Rate limited | + `Retry-After` header |
| **500** Internal Server Error | Bug | |
| **502** Bad Gateway | Upstream returned garbage | Proxy/LB |
| **503** Service Unavailable | Overloaded / maintenance | |
| **504** Gateway Timeout | Upstream too slow | |

### Error format
Use a consistent body (e.g., RFC 9457 "Problem Details"):
```json
{ "type": "https://api.example.com/errors/insufficient-funds",
  "title": "Insufficient funds", "status": 409,
  "detail": "Balance 30.00 is less than 50.00", "traceId": "abc123" }
```

### Pagination

| Type | How | Pros | Cons |
|---|---|---|---|
| **Offset/limit** | `?offset=40&limit=20` | Simple; jump to page N | Slow at large offsets (DB scans + skips); duplicates/missed rows when data changes |
| **Cursor/keyset** | `?after=eyJpZCI6MTIzfQ&limit=20` (opaque cursor encodes last seen sort key) | Fast (`WHERE id > ?` uses index); stable under inserts | No random page jumps |

> Use **cursor pagination** for feeds and large or real-time lists. Facebook, Twitter/X, Slack, and Stripe APIs all use cursors.

### Idempotency
Networks fail. A client may retry `POST /payments` after a timeout without knowing if the first one succeeded. Solution: an **idempotency key**.
```
POST /payments
Idempotency-Key: 7f9c2b1e-...      (client-generated UUID)
```
The server stores `key → response` (with a TTL). On a repeat, it returns the stored response instead of charging twice. Stripe popularized this. See also [[Distributed Systems Patterns#Idempotency]].

### Versioning
- **URI**: `/v1/users`. Most common and explicit.
- **Header**: `Accept: application/vnd.example.v2+json`.
- **Query**: `?version=2`.
- Rules: **never break existing clients**. Adding optional fields is safe; removing or renaming fields or changing types is breaking. Deprecate with notice and `Sunset` headers.

### Caching headers
- `Cache-Control: max-age=3600, public` / `private` / `no-store`
- `ETag` + `If-None-Match` → 304
- `Last-Modified` + `If-Modified-Since`

### HATEOAS
"Hypermedia as the engine of application state": responses include links to related actions. The highest level of the **Richardson Maturity Model** (L0 RPC over HTTP → L1 resources → L2 HTTP verbs → L3 hypermedia). Rarely fully implemented in practice; know the term.

### Documentation
**OpenAPI (Swagger)** spec: machine-readable API description → docs, client SDKs, validation. Contract-first design is common at large companies.

---

## GraphQL

A query language where the **client specifies exactly what data it needs** from a single endpoint (`POST /graphql`). Created at Facebook.

```graphql
query {
  user(id: "123") {
    name
    orders(last: 3) { id total items { name } }
  }
}
```

| ✅ Pros | ❌ Cons |
|---|---|
| No **over-fetching** (extra fields) or **under-fetching** (multiple round trips) | HTTP caching is harder (everything is POST to one URL) |
| Strongly typed schema, introspection, great tooling | **N+1 query problem** in resolvers → use **DataLoader** (batching) |
| Great for multiple clients (web/mobile) with different needs | Complex queries can be expensive → need depth/complexity limits |
| Evolve without versions (deprecate fields) | Harder rate limiting; file uploads awkward |

Operations: **query** (read), **mutation** (write), **subscription** (real-time, usually over WebSockets).

---

## gRPC

Google's high-performance RPC framework: **Protocol Buffers** (binary, schema-defined) over **HTTP/2**.

```protobuf
service OrderService {
  rpc GetOrder (GetOrderRequest) returns (Order);
  rpc StreamUpdates (OrderId) returns (stream OrderUpdate);
}
message GetOrderRequest { string id = 1; }
```

| ✅ | ❌ |
|---|---|
| Fast, compact binary; strong contracts; code generation | Not human-readable; browsers need gRPC-Web proxy |
| Streaming: unary, server-, client-, bidirectional | Steeper learning curve |
| Ideal for **internal service-to-service** calls | |

---

## Real-time & push options

| Tech | Direction | Use when |
|---|---|---|
| **Short polling** | Client asks every N sec | Simple, low-frequency updates; wasteful |
| **Long polling** | Client asks; server holds request open until data or timeout | Fallback for real-time; simpler infra |
| **Server-Sent Events (SSE)** | Server → client over one HTTP connection | One-way streams: notifications, live scores, LLM token streaming |
| **WebSockets** | **Full-duplex** persistent TCP connection | Chat, multiplayer games, collaborative editing, live trading. See [[Design a Chat System]] |
| **Webhooks** | Server → *another server* via HTTP callback | Third-party integrations (Stripe payment succeeded, GitHub push) |

> [!tip] Webhook best practices
> Sign payloads (HMAC) so receivers can verify them, retry with backoff, make receivers idempotent (events can arrive twice), and return 2xx fast and process async.

---

## REST vs GraphQL vs gRPC

| | REST | GraphQL | gRPC |
|---|---|---|---|
| Format | JSON (text) | JSON | Protobuf (binary) |
| Transport | HTTP/1.1 or 2 | HTTP | HTTP/2 |
| Best for | Public APIs, CRUD, cacheable resources | Client-driven data needs, aggregating many sources | Internal microservices, low latency, streaming |
| Caching | Excellent (HTTP) | Harder | Custom |
| Browser support | Native | Native | Needs proxy |

---

## API design in a system design interview

After requirements, define the core endpoints **before** drawing boxes:

```
POST /v1/tweets            { text, mediaIds[] }           → 201 { tweetId }
GET  /v1/feed?cursor=&limit=20                            → 200 { tweets[], nextCursor }
POST /v1/users/{id}/follow                                → 204
```

Mention: auth (bearer token), pagination, idempotency for creates/payments, rate limiting.

---

## API design checklist (for work and reviews)
- [ ] Resource-oriented, consistent naming and casing (`camelCase` JSON is common)
- [ ] Correct methods and status codes
- [ ] Validation with clear error bodies
- [ ] Pagination on every list endpoint
- [ ] Idempotency for non-idempotent writes that clients retry
- [ ] AuthN/AuthZ on every endpoint. See [[Security Fundamentals]]
- [ ] Rate limiting. See [[Design a Rate Limiter]]
- [ ] Versioning strategy; no breaking changes
- [ ] Timestamps in ISO 8601 UTC; money as integer minor units (cents) + currency
- [ ] Documented with OpenAPI
- [ ] Backward compatibility with existing clients

## Self-test
- [ ] PUT vs PATCH vs POST and idempotency of each
- [ ] 401 vs 403
- [ ] Offset vs cursor pagination tradeoffs
- [ ] How does an idempotency key prevent double charges?
- [ ] When would you choose gRPC over REST? GraphQL over REST?

Next: [[Domain-Driven Design]]
