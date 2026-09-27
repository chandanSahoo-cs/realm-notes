# Database Sharding

Database sharding is the practice of horizontally partitioning a single logical dataset across multiple independent database instances. Unlike single-node scaling, each shard runs as an autonomous server with dedicated CPU, memory, and disk.

---

## Single-Instance Partitioning vs. Horizontal Sharding

Before distributing data across multiple servers, understand the boundary between local table partitioning and true distributed sharding:

```text
Table Partitioning (Single Host)             Horizontal Sharding (Multi-Host)
┌──────────────────────────────────────┐     ┌──────────┐  ┌──────────┐  ┌──────────┐
│           Database Server            │     │ Shard 1  │  │ Shard 2  │  │ Shard 3  │
│  ┌────────────────────────────────┐  │     │ (Node A) │  │ (Node B) │  │ (Node C) │
│  │ Partition 1: Jan - Mar 2026    │  │     │          │  │          │  │          │
│  ├────────────────────────────────┤  │     │ Users    │  │ Users    │  │ Users    │
│  │ Partition 2: Apr - Jun 2026    │  │     │ 0 - 10M  │  │ 10M - 20M│  │ 20M - 30M│
│  └────────────────────────────────┘  │     └──────────┘  └──────────┘  └──────────┘
│ Shared CPU, RAM, and Disk I/O        │     Independent Hardware & Connections
└──────────────────────────────────────┘
```

- **Table Partitioning (In-Instance):** Divides a large table into smaller physical segments within the same database engine. It speeds up queries via partition pruning and simplifies table maintenance (e.g., dropping an old month's partition instantly). However, all partitions still compete for the same server CPU, memory, and disk bandwidth:
  - **Horizontal Partitioning (by row):** Divides records into row subsets (e.g., one partition per calendar month) while maintaining identical schema columns.
  - **Vertical Partitioning (by column):** Divides wide tables by columns, moving rarely accessed large text blobs or JSON payloads to an auxiliary table to keep frequently queried core data pages compact in memory.
- **Horizontal Sharding:** Distributes segments across physically distinct database servers. It scales storage capacity, connection limits, and read/write throughput linearly by adding machines.

---

## Sharding Architecture & Query Routing

A sharded database requires an intermediate layer to route queries to the correct shard based on the **shard key**:

```text
                                Client Request
                                      │
                                      ▼
┌───────────────────────────────────────────────────────────────────────────┐
│                       Routing / Coordinator Layer                         │
│             (App Client Library, Vitess, Citus, or Proxy)                 │
└─────────────────────────────────────┬─────────────────────────────────────┘
                                      │
          ┌───────────────────────────┼───────────────────────────┐
          ▼                           ▼                           ▼
   ┌─────────────┐             ┌─────────────┐             ┌─────────────┐
   │   Shard 1   │             │   Shard 2   │             │   Shard 3   │
   │  (Primary)  │             │  (Primary)  │             │  (Primary)  │
   │      │      │             │      │      │             │      │      │
   │      ▼      │             │      ▼      │             │      ▼      │
   │   Replica   │             │   Replica   │             │   Replica   │
   └─────────────┘             └─────────────┘             └─────────────┘
```

### Routing Mechanisms
1. **Application-Level Routing:** The application code hashes the key and connects directly to the appropriate database connection pool. Highly performant with zero extra network hops, but tightly couples database infrastructure to application logic.
2. **Proxy / Middleware Routing:** A dedicated proxy layer (e.g., Vitess for MySQL, Citus for PostgreSQL, or `mongos` in MongoDB) presents itself as a single unified database. It inspects incoming SQL/NoSQL statements and transparently routes them to the backing shards.

---

## Shard Key Selection Principles

The shard key dictates how data is distributed and how queries execute. Selecting a poor shard key is difficult to reverse once petabytes of data have been written.

### The Three Core Requirements
1. **High Cardinality:** The key must have millions of distinct values. Sharding by an enum (e.g., `status` with 4 states) restricts you to at most 4 shards.
2. **Uniform Write Distribution:** The key must distribute writes evenly across time. Monotonically increasing values (e.g., auto-incrementing IDs or timestamps) funnel 100% of new writes into the newest shard.
3. **Query Locality:** The key must match the primary filter of your highest-frequency queries. If queries filter by `user_id`, sharding by `user_id` allows requests to hit a single shard rather than querying every node.

### Candidate Shard Key Evaluation

| Key Candidate | Cardinality | Write Distribution | Query Locality | Suitability |
|---|---|---|---|---|
| `user_id` / `account_id` | High (Millions) | Uniform (hashed) | Excellent for user-scoped queries | **Best default** for consumer products |
| `tenant_id` | Variable | Risk of skew from enterprise tenants | Excellent for B2B SaaS isolation | Strong for multi-tenant SaaS; requires VIP shard routing |
| `order_id` | High (Millions) | Uniform | Good for single order lookups; bad for customer order histories | Good for logistics; poor for user dashboards |
| `created_at` (Timestamp) | Infinite | **Severe hot spotting** (all writes hit current shard) | Good for time-series range scans | **Anti-pattern** for transactional writes |

---

## Distribution Strategies

```text
1. Hash-Based Sharding (Standard Default)
   Shard ID = hash(shard_key) % num_shards
   - Pros: Completely randomizes key placement; eliminates write hot spots.
   - Cons: Range queries cannot target a single shard and must fan out to all nodes.

2. Range-Based Sharding
   Shard 1: IDs 0 - 1,000,000 | Shard 2: IDs 1,000,001 - 2,000,000
   - Pros: Efficient range scans; adjacent data sits on the same disk.
   - Cons: Monotonically increasing keys cause all new writes to hit the highest shard.

3. Directory-Based / Lookup Sharding
   Central mapping service: Key -> Specific Shard ID
   - Pros: Maximum operational flexibility; oversized "whale" tenants can be 
           dynamically relocated to dedicated high-performance hardware.
   - Cons: Every query requires a lookup in the directory table, creating a single 
           point of failure and adding network latency (requires aggressive caching).
```

---

## Distributed Complications & Failure Modes

Sharding solves storage and write throughput limits, but it breaks the assumptions of single-node relational databases:

### 1. Scatter-Gather Queries (The P99 Latency Problem)
When a query does not include the shard key (e.g., searching for products by name or generating global analytical reports), the coordinator must send the query to **every shard in parallel**, collect all responses, and merge/sort the results in memory.
- The overall query latency is bound by the **slowest responding shard**. If 1 out of 64 shards experiences an I/O freeze, the entire user request blocks.
- **Mitigation:** Maintain secondary search indexes in a specialized search cluster (e.g., Elasticsearch/OpenSearch) or use asynchronous pipelines to populate pre-aggregated reporting tables.

### 2. Cross-Shard Transactions
Single-node ACID transactions cannot span multiple independent databases.
- **Two-Phase Commit (2PC):** Uses a coordinator to prepare and commit writes across shards. It guarantees strong consistency but is slow, holds locks across network round-trips, and blocks indefinitely if the coordinator dies during the commit phase.
- **The Saga Pattern:** Decompose multi-shard operations into a sequence of local transactions. If Step 2 fails on Shard B, execute a compensating transaction on Shard A to revert Step 1.

### 3. Cross-Shard Joins
Relational SQL engines cannot execute joins across different network nodes.
- **Mitigation 1 (Denormalization):** Duplicate parent data into the child record so that queries are self-contained on a single shard.
- **Mitigation 2 (Global Table Replication):** Small, slowly changing reference tables (e.g., countries, tax rates, currency codes) are replicated in full to every shard, allowing local joins.

---

## Online Resharding Workflow

As data volume grows, existing shards inevitably run out of storage or IOPS capacity. Adding new shards without taking downtime requires a staged migration pipeline:

```text
Phase 1: Provision new target shard instances.
Phase 2: Enable dual-writing (or stream CDC changes from old shards to new shards).
Phase 3: Run a background batch worker to backfill historical data up to a snapshot timestamp.
Phase 4: Monitor replication lag until the new shards are fully caught up with live writes.
Phase 5: Update the routing coordinator's configuration to point traffic to the new shard boundaries.
Phase 6: Deprecate and safely prune data on the old shards.
```

---

## Database-Specific Sharding Implementations

Different database engines approach horizontal scaling through distinct architectural trade-offs:

| System | Sharding Strategy | Routing / Rebalancing Architecture |
|---|---|---|
| **Apache Cassandra** | Consistent Hash Ring (`Murmur3Partitioner`) | Fully decentralized peer-to-peer. The partition key maps to a 64-bit integer token; tokens are interleaved across virtual nodes (vnodes). |
| **Amazon DynamoDB** | Managed Hash Partitioning | Fully automated. AWS automatically provisions and splits storage partitions when partition size exceeds 10 GB or throughput exceeds 1,000 WCU / 3,000 RCU. |
| **MongoDB** | Range or Hashed Sharding | Coordinated cluster: `mongos` query routers act as stateless proxies; Config Servers maintain chunk metadata; background balancer automatically migrates chunks across shards. |
| **Vitess (MySQL) / Citus (Postgres)** | Declarative Sharded Relational SQL | Middleware / extension proxies that parse standard SQL queries, coordinate two-phase commits across relational shards, and automate online resharding cutovers. |

