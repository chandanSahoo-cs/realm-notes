# API Design & Architecture

An Application Programming Interface (API) establishes the formal operational contract between client applications and backend systems, or across internal microservices. Designing scalable, resilient APIs requires careful evaluation of **architectural paradigms**, **safe retry semantics**, **pagination performance**, and **gateway-level security**.

---

## 1. API Architectural Paradigms

Modern systems typically combine multiple API styles across different system boundaries:

```text
External Clients (Web / Mobile / Third-Party)
                    │
                    ▼
           [ API Gateway Layer ]
                    │
        ┌───────────┴───────────┐
        ▼                       ▼
Public REST API          GraphQL BFF Layer
(Resource-oriented)     (Dynamic Graph Querying)
        │                       │
        └───────────┬───────────┘
                    │
                    ▼
       Internal Microservices
       (gRPC / Protocol Buffers)
```

### Resource-Oriented HTTP (REST)
REST organizes endpoints around business domain **resources (nouns)** rather than RPC procedures (verbs):

```http
# Resource-Oriented CRUD Conventions:
GET    /orders              # Retrieve paginated list of orders
POST   /orders              # Create a new order (server assigns ID)
GET    /orders/{id}         # Fetch specific order by ID
PUT    /orders/{id}         # Replace the entire order resource
PATCH  /orders/{id}         # Partial update (e.g., update shipping address)
DELETE /orders/{id}         # Remove the order resource
GET    /orders/{id}/items   # Sub-resource: Line items belonging to order
```

#### HTTP Method Semantics & Idempotency
An operation is **idempotent** if executing it multiple times produces the identical server state as executing it once. This is critical for network retry safety:

| HTTP Verb | Operation | Safe? | Idempotent? | Expected Success Code |
|---|---|---|---|---|
| **GET** | Read resource | Yes | Yes | `200 OK` |
| **POST** | Create resource / command | No | No | `201 Created` / `202 Accepted` |
| **PUT** | Full replace / upsert | No | Yes | `200 OK` / `204 No Content` |
| **PATCH** | Partial field update | No | Conditionally | `200 OK` |
| **DELETE** | Remove resource | No | Yes | `204 No Content` / `200 OK` |

- **Safe:** Does not modify server state.
- **Idempotent:** Safe to retry automatically over unreliable networks without generating duplicate side effects.

#### Data Passing & Resource Nesting
Data is passed to REST endpoints through three distinct mechanisms based on structural intent:

1. **Path Parameters:** Mandatory resource identifiers defining identity hierarchy (`/orders/{id}`).
2. **Query Parameters:** Optional modifiers used for filtering, sorting, projection, and pagination (`/orders?status=shipped&sort=desc&limit=50`).
3. **Request Body:** Complex data payloads representing structured entity states (`POST /orders` or `PATCH /orders/{id}`).

```text
Nested vs. Flat Resource Modeling:
- Use Nested Paths (/customers/{id}/invoices): When the child entity's lifecycle is strictly
  subordinate to the parent and never accessed independently.
- Use Flat Paths (/invoices?customer_id=44): When the entity can be queried across multiple 
  dimensions (e.g., searching invoices by status or date range regardless of customer).
```

#### HTTP Status Codes & Error Contracts
Status codes communicate operational outcomes reliably across heterogeneous clients:

| Status Code | Meaning | Category | Architectural Implication |
|---|---|---|---|
| **200 OK** | Success | 2xx Success | Standard response for successful GET, PUT, or PATCH. |
| **201 Created** | Created | 2xx Success | Returned on POST; includes `Location` header to new URI. |
| **204 No Content** | No Body | 2xx Success | Returned on successful DELETE or PUT where no body is returned. |
| **400 Bad Request** | Invalid Input | 4xx Client Error | Malformed JSON, missing fields, or validation failures. |
| **401 Unauthorized**| No Auth | 4xx Client Error | Authentication token is missing, expired, or cryptographically invalid. |
| **403 Forbidden** | No Permission | 4xx Client Error | Principal is authenticated, but RBAC policies deny permission. |
| **404 Not Found** | Missing | 4xx Client Error | Resource URI does not correspond to an existing record. |
| **429 Too Many Req**| Rate Limited | 4xx Client Error | Client exceeded quota; response should include `Retry-After`. |
| **500 Internal Err**| Server Crash | 5xx Server Error | Unhandled backend exception; alerts on-call engineering. |
| **502 Bad Gateway** | Upstream Error | 5xx Server Error | Gateway or reverse proxy received an invalid response from upstream. |
| **503 Unavailable** | Overloaded | 5xx Server Error | Service circuit breaker is open or database pool is exhausted. |

> **4xx vs. 5xx Retry Semantics:**
> **4xx errors** indicate client faults; repeating the identical request will fail again without modifying client parameters.
> **5xx errors** indicate server or network infrastructure faults; clients can safely initiate retries using exponential backoff.

---

### Remote Procedure Calls (gRPC & Protobuf)
gRPC operates as an action-oriented RPC framework built on HTTP/2 and Protocol Buffers:
- **Binary Serialization:** Protobuf encodes data into compact binary payloads, consuming 60–80% less network bandwidth and CPU parsing overhead than JSON.
- **Strict Interface Definition Language (IDL):** Client and server stubs are compiled from `.proto` schema definitions, catching contract mismatches at compile time across polyglot microservices.
- **Multiplexed Streaming:** Supports Unary, Client-streaming, Server-streaming, and Bidirectional streaming over a single TCP connection.
- **Apache Thrift Alternative:** Originally developed by Facebook, Apache Thrift is a mature alternative RPC framework that also provides code generation and binary serialization across multiple languages.
- **Standard Use Case:** High-throughput, low-latency inter-service communication behind the API gateway.

---

### Graph-Based Querying (GraphQL)
GraphQL provides a single endpoint where clients declare the exact shape of the data they need:
- **Solves Over-fetching & Under-fetching:** A mobile client can query only `id` and `title`, while a desktop dashboard queries the complete nested graph including author, comments, and permissions in a single round-trip.
- **The N+1 Query Trap:** If a query requests 50 orders and each order's customer details, a naive resolver executes 1 query for orders plus 50 separate database queries for customers.
  - **Mitigation:** Use the **DataLoader pattern** to batch and deduplicate keys across an event loop tick (`SELECT * FROM customers WHERE id IN (...)`).
- **Standard Use Case:** Backend-For-Frontend (BFF) layers serving diverse client devices with varying bandwidth and UI layout requirements.

---

### Paradigm Comparison Matrix

| Dimension | REST | gRPC | GraphQL |
|---|---|---|---|
| **Underlying Model** | Domain Entities (Nouns) | Procedures / Functions | Graph of Types |
| **Transport / Format**| HTTP/1.1 or HTTP/2 + JSON | HTTP/2 + Protobuf Binary | HTTP + JSON (Single POST) |
| **Type Safety** | Optional (OpenAPI/Swagger) | Strict (Compile-time code-gen)| Strict (Schema-validated) |
| **Network Overhead** | Moderate (Verbose JSON keys)| Minimal (Compressed binary) | Low over wire; high server parse |
| **Client Support** | Universal (All browsers/tools)| Requires gRPC-Web proxy in browser| Universal (Standard HTTP clients) |
| **Primary Domain** | Public APIs, Partner integrations | Internal Microservices | Web/Mobile Frontend Aggregation |

---

## 2. Pagination & High-Volume Query Optimization

APIs must never return unpaginated database collections. Two primary pagination strategies exist:

```text
Offset Pagination:
GET /v1/orders?offset=100000&limit=20
SQL: SELECT * FROM orders ORDER BY created_at DESC LIMIT 20 OFFSET 100000;
- Problem: The database must scan and discard 100,000 rows before returning 20.
- Page Drift: If records are inserted between page requests, users see duplicate items.

Keyset / Cursor Pagination:
GET /v1/orders?cursor=eyJpZCI6OTkyMSwidHMiOjE3MTAwMDAwfQ==&limit=20
SQL: SELECT * FROM orders 
     WHERE (created_at, order_id) < (:cursor_ts, :cursor_id) 
     ORDER BY created_at DESC, order_id DESC LIMIT 20;
- Performance: O(1) B+ Tree index seek. Constant latency regardless of pagination depth.
- Concurrency Safe: Stable under continuous real-time inserts and deletes.
- Limitation: Sequential traversal only (cannot jump to arbitrary page 50).
```

| Criterion | Offset Pagination | Keyset / Cursor Pagination |
|---|---|---|
| **Database Complexity** | Simple (`OFFSET n LIMIT m`) | Requires composite cursor encoding (e.g., base64) |
| **Query Performance** | `O(N)` — degrades significantly at deep offsets | `O(1)` — constant seek time using composite index |
| **Data Consistency** | Prone to duplicate or skipped items | Stable across live concurrent writes |
| **Random Page Access** | Supported (Jump to page N) | Unsupported (Next / Previous only) |

---

## 3. Resilience & Safe Retry Patterns

### Distributed Idempotency Keys
Non-idempotent endpoints (e.g., `POST /payments`, `POST /orders`) must handle client retries safely without executing duplicate transactions:

```text
Client                                  API Gateway / Service                     Distributed Cache (Redis)
  │                                               │                                          │
  │─── POST /v1/payments ────────────────────────▶│                                          │
  │    Idempotency-Key: a8f1-4c22-9e10            │─── 1. Check if key exists in Redis ─────▶│
  │                                               │                                          │
  │                                               │◀── 2. Key NOT found ─────────────────────│
  │                                               │                                          │
  │                                               │─── 3. Set atomic lock on key ───────────▶│
  │                                               │       (status: "IN_FLIGHT", TTL: 120s)   │
  │                                               │                                          │
  │                                               │─── 4. Execute payment transaction        │
  │                                               │                                          │
  │                                               │─── 5. Store response body & code ───────▶│
  │                                               │       (status: "COMPLETED", TTL: 24h)    │
  │                                               │                                          │
  │◀── 6. Return 201 Created ─────────────────────│                                          │
  │                                               │                                          │
  │─── [RETRY AFTER TIMEOUT] ────────────────────▶│                                          │
  │    POST /v1/payments                          │─── 7. Check key in Redis ────────────────▶│
  │    Idempotency-Key: a8f1-4c22-9e10            │                                          │
  │                                               │◀── 8. Key found! status: "COMPLETED" ────│
  │◀── 9. Return cached 201 response immediately ─│                                          │
```

1. **Client Generation:** The client generates a unique UUID before sending the mutation request.
2. **Locking & Execution:** The gateway checks Redis. If absent, it records an `IN_FLIGHT` state with a short lock TTL (e.g., 2 minutes) and proceeds with the business logic.
3. **Response Caching:** Once the transaction completes, the server writes the final status code and response body to Redis with an expiration window (e.g., 24 hours).
4. **Duplicate Interception:** Retries with the same key bypass business logic entirely and return the cached response.

---

### API Versioning Strategies

When introducing breaking changes (removing fields, altering data types, changing required headers):

1. **URI Path Versioning (Recommended):**
   ```http
   GET /v1/customers
   GET /v2/customers
   ```
   - **Pros:** Explicit, easy to route at the load balancer or API gateway, transparent in application logs and browser testing.
   - **Cons:** Binds the URL identifier to schema versions.

2. **Header / Content Negotiation Versioning:**
   ```http
   GET /customers
   Accept: application/vnd.company.v2+json
   # Or:
   X-API-Version: 2026-03-01
   ```
   - **Pros:** Keeps resource URIs clean and stable across iterations.
   - **Cons:** Harder to debug manually; complicates edge-cache routing rules.

---

## 4. Perimeter Security & Traffic Management

```text
Incoming Request
       │
       ▼
┌──────────────────┐
│  Authentication  │ ──> WHO are you?
│ (Verify JWT/Key) │     - Validates cryptographic signature
└────────┬─────────┘     - Extracts user identity & tenant claims
         │
         ▼
┌──────────────────┐
│  Authorization   │ ──> WHAT are you permitted to do?
│  (RBAC / ABAC)   │     - Role checks (e.g., "admin" vs "viewer")
└────────┬─────────┘     - Resource ownership (Can user A view invoice B?)
         │
         ▼
┌──────────────────┐
│  Rate Limiting   │ ──> HOW OFTEN can you call?
│  (Token Bucket)  │     - Protects downstream services from overload
└────────┬─────────┘     - Returns HTTP 429 Too Many Requests
         │
         ▼
[ Service Handlers ]
```

### Authentication Architecture: Stateless JWTs vs. API Keys
- **API Keys:** Opaque, randomly generated strings stored in a database. Ideal for machine-to-machine integrations or third-party developer platforms. Require a database or cache lookup on every request to resolve identity.
- **JSON Web Tokens (JWT):** Signed, self-contained identity tokens containing user claims (`sub`, `tenant_id`, `roles`, `exp`). The API gateway verifies the token using the authentication service's public key (e.g., RS256) without performing a centralized database lookup.

### Rate Limiting Algorithms
- **Token Bucket:** Tokens are added to a bucket at a fixed rate `r` up to capacity `b`. Each request consumes 1 token. Handles bursty traffic up to limit `b` while enforcing an average rate `r`.
- **Sliding Window Counter:** Divides time into rolling windows tracked in an in-memory sorted set (e.g., Redis `ZADD` + `ZREMRANGEBYSCORE`). Prevents boundary bursts associated with fixed-window counters.
