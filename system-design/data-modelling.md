# Data Modeling & Schema Design

Data modeling translates application requirements into database schemas that optimize for **query performance**, **write throughput**, and **data integrity**. The choice of data model dictates how the system scales, how transactions execute, and where latency bottlenecks will emerge.

---

## Storage Engine Paradigms

```text
                               Storage Engine Paradigms
        ┌───────────────────────┬───────────────────────┬───────────────────────┬───────────────────────┐
        ▼                       ▼                       ▼                       ▼                       ▼
 Relational (SQL)           Document               Key-Value              Wide-Column                 Graph
(PostgreSQL, MySQL)    (MongoDB, Couchbase)     (Redis, DynamoDB)     (Cassandra, ScyllaDB)     (Neo4j, Neptune)
- Rigid schema          - Flexible schema       - Fast exact-key lookup - Massive write rate    - Index-free adjacency
- Strong ACID           - Embedded objects      - O(1) access latency   - Query-driven schemas  - Deep graph traversals
- Multi-table joins     - Single-doc atomicity  - Cache/session stores  - Append-only SSTables  - Fraud, social networks
```

### 1. Relational Databases (SQL)
- **Model:** Tabular rows and columns enforcing foreign keys and relational constraints.
- **Strengths:** Strong ACID transactions, expressive joins, mature B+ Tree indexing, data integrity.
- **Limitations:** Horizontal sharding is complex; heavy joins across multi-million-row tables cause significant I/O overhead under high concurrency.
- **Best Suited For:** Financial transactions, e-commerce order processing, user management, and systems requiring strict consistency.

### 2. Document Stores (NoSQL)
- **Model:** Hierarchical, semi-structured documents (JSON/BSON) grouped into collections.
- **Strengths:** Schema agility, nesting related entities together to eliminate joins, intuitive mapping to application objects.
- **Limitations:** Multi-document ACID transactions incur latency penalties; deeply nested queries degrade index efficiency.
- **Best Suited For:** Content management systems, product catalog metadata with variable attributes, dynamic user profile schemas.

### 3. Key-Value Stores
- **Model:** Schema-agnostic binary blobs or JSON strings indexed strictly by an arbitrary primary key.
- **Strengths:** Sub-millisecond reads and writes (`O(1)` complexity), horizontal partitioning.
- **Limitations:** Cannot query or filter by internal attributes without full keyspace scans.
- **Best Suited For:** User sessions, authentication tokens, rate-limiting state, shopping cart scratchpads.

### 4. Wide-Column / Partitioned LSM Stores
- **Model:** Rows partitioned by a **Partition Key** (physical node routing) and ordered by a **Clustering Key** (on-disk sorting).
- **Strengths:** High write throughput via Log-Structured Merge (LSM) trees, linear horizontal scaling, fast contiguous range scans within a partition.
- **Limitations:** Strict query-first design—schemas must be designed around exact queries; joins and arbitrary aggregations are not supported.
- **Best Suited For:** Time-series telemetry, IoT sensor streams, audit trails, real-time messaging histories.

### 5. Graph Databases
- **Model:** Property graph consisting of **Nodes** (entities), **Edges** (relationships), and **Properties** (key-value metadata on nodes and edges).
- **Strengths:** **Index-free adjacency** allows traversing relationships in constant time `O(1)` per hop regardless of the total size of the graph, making deep recursive queries (e.g., 4th-degree connections) efficient.
- **Limitations:** Very difficult to shard across multiple machines without expensive distributed graph-partitioning algorithms.
- **Best Suited For:** Fraud detection networks (detecting cyclical transactions), identity and access management (IAM permission graphs), knowledge engines, recommendation graphs.
- **Architectural Reality:** Unless your queries frequently traverse relationships deeper than 2 or 3 degrees of separation, relational or document databases with indexed foreign keys are simpler to operate.

---

## Paradigm Evaluation Matrix

| Requirement | Relational (SQL) | Document | Key-Value | Wide-Column | Graph |
|---|---|---|---|---|---|
| **ACID Transactions** | Multi-table (Native) | Single-document | Single-key | Single-partition | Single-node graph |
| **Join Support** | Full SQL joins | Application-level | None | None | Edge traversals |
| **Horizontal Scalability**| Challenging (Sharding) | Native (Replica sets) | Native (Partitioning)| Linear (Hash ring) | Challenging |
| **Schema Flexibility** | Schema migrations needed| Dynamic / Flexible | Schema-agnostic | Fixed column families| Flexible graph |
| **Query Flexibility** | High (Ad-hoc queries)| Moderate | Low (Key-only) | Strict (Query-driven)| Traversal-driven |

---

## Schema Engineering: Normalization vs. Deliberate Denormalization

```text
Normalized Schema (3NF) - Write-Optimized:
Users Table:        [ user_id (PK), name, email ]
Orders Table:       [ order_id (PK), user_id (FK), total_amount, created_at ]
Order_Items Table:  [ item_id (PK), order_id (FK), product_id, quantity, unit_price ]

- Pros: Zero redundancy; updating a user's email requires modifying exactly one row.
- Cons: Reading an order invoice requires a 3-table join across millions of records.

Denormalized Schema - Read-Optimized:
Orders Table:       [ order_id (PK), user_id, user_name, user_email, items_json, total, created_at ]

- Pros: A single point lookup retrieves the entire order invoice with zero joins.
- Cons: If user_email changes, older orders display stale data or require expensive batch updates.
```

### Constraints & The Foreign Key Dilemma at Scale
Database constraints protect data integrity, but incur significant operational trade-offs under high write throughput:

- **Database-Enforced Foreign Keys:** In relational databases, inserting a child record locks the corresponding parent record in shared mode to verify existence. Under high write concurrency, this generates severe row-lock contention and deadlocks. Furthermore, foreign keys cannot cross database shards.
- **Application-Layer Referential Integrity:** At hyper-scale (e.g., platforms operating partitioned MySQL or Postgres), engineering teams often **drop database-level foreign key constraints**. Existence checks are enforced in the application layer, and eventual consistency is maintained via asynchronous background reconciliation jobs.
- **Column Constraints (`NOT NULL`, `CHECK`, `UNIQUE`):** While `NOT NULL` and `CHECK` add minimal CPU validation overhead, `UNIQUE` constraints require an underlying index and serialize concurrent writes to guarantee uniqueness.

---

## Document Modeling: Embedding vs. Referencing

In document databases, deciding whether to embed child data inside a parent document or store it in a separate collection with an ID reference is a fundamental design decision:

```text
Embedding (1-to-Few)                          Referencing (1-to-Many / Unbounded)
┌───────────────────────────────────────┐      ┌───────────────────────────────────────┐
│ Order Document                        │      │ User Document                         │
│ - order_id: 1042                      │      │ - user_id: 881                        │
│ - customer_id: 44                     │      │ - name: "Sarah"                       │
│ - items: [                            │      └──────────────────┬────────────────────┘
│     { sku: "X1", qty: 2, price: 15 }, │                         │ 1:N Reference
│     { sku: "Y2", qty: 1, price: 30 }  │                         ▼
│   ]                                   │      ┌───────────────────────────────────────┐
└───────────────────────────────────────┘      │ User Audit Logs Collection            │
- Atomic single-disk read/write                │ - log_id: 99120                       │
- Best when child records are bounded (< 100)  │ - user_id: 881                        │
- Avoids exceeding 16MB document limits        │ - action: "PASSWORD_RESET"            │
                                               └───────────────────────────────────────┘
```

- **Embed when:** Child items are tightly coupled to the parent, always read and updated together, and bounded in size (e.g., items within an order, addresses for a user).
- **Reference when:** The child collection grows unbounded (e.g., user activity logs, comments on a post), child entities are updated frequently by background workers, or records are shared across multiple parents (many-to-many).

---

## Indexing Mechanics & Query Optimization

Indexes trade disk space and write throughput for logarithmic (`O(log N)`) read speed.

```text
Table Heap (Unsorted Data Pages)         B+ Tree Index on (tenant_id, created_at)
┌────┬───────────┬────────────┐                       [Root Node]
│ id │ tenant_id │ created_at │                       /         \
├────┼───────────┼────────────┤              [Internal]         [Internal]
│ 1  │ 5         │ 2026-01-01 │               /     \             /     \
│ 2  │ 2         │ 2026-01-03 │            [Leaf]  [Leaf]      [Leaf]  [Leaf]
│ 3  │ 5         │ 2026-01-02 │               │       │           │       │
└────┴───────────┴────────────┘               ▼       ▼           ▼       ▼
                                        Pointers to matching table heap rows
```

### 1. Clustered vs. Secondary Indexes
- **Clustered Index:** Defines the physical, on-disk sequential order of table rows (typically the Primary Key). A table can contain only **one** clustered index.
- **Secondary Index:** A separate B+ Tree where leaf nodes contain the indexed column values along with pointers back to the primary clustered index key.

### 2. Composite Indexes and the Leftmost Prefix Rule
When a query filters or sorts by multiple columns, create a composite index:

```sql
CREATE INDEX idx_orders_tenant_status_date ON orders (tenant_id, status, created_at);
```

The database query planner can use this index if the `WHERE` clause filters by:
- `(tenant_id)`
- `(tenant_id, status)`
- `(tenant_id, status, created_at)`

The index **cannot** be used efficiently if the query filters only by `(status)` or `(created_at)`, because the B+ Tree is sorted primarily by `tenant_id`.

### 3. Covering Indexes (Index-Only Scans)
If an index contains every column requested by a query, the engine satisfies the request directly from the B+ Tree leaf nodes without fetching the main table heap:

```sql
CREATE INDEX idx_users_lookup ON users (email) INCLUDE (user_id, account_status);

-- Runs entirely in memory from index leaf pages:
SELECT user_id, account_status FROM users WHERE email = 'dev@example.com';
```

---

## Sharded Schema Design Patterns

When designing schemas for distributed or partitioned databases:
- **Avoid Monotonic Shard Keys:** Sharding by `created_at` or auto-incrementing IDs directs 100% of new write traffic to the most recent shard.
- **Use Compound Partition Keys:** Combine a high-cardinality grouping identifier with an ordering key:
  - In Cassandra: `PRIMARY KEY ((tenant_id), created_at)` — hashes `tenant_id` to locate the node, and orders records by `created_at` on that node.
  - In distributed SQL: Shard by `hash(account_id)` to distribute writes, while keeping each account's transactions co-located on a single shard.
