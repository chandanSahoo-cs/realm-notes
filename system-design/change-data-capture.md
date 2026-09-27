# Change Data Capture (CDC)

Change Data Capture (CDC) is an architectural pattern that detects and streams row-level changes (inserts, updates, deletes) from a source database to downstream consumers in real time. It serves as the primary mechanism for synchronizing databases with search indexes, distributed caches, event-driven microservices, and analytics data lakes.

---

## The Dual-Write Hazard: Why CDC Is Necessary

When an application must update a database and notify an external system (e.g., updating an inventory table and publishing an event to Kafka), a direct **dual-write** approach introduces critical failure modes:

```text
Application
  │
  ├── 1. Write to Primary Database  --> [SUCCESS]
  │
  └── 2. Publish Event to Kafka     --> [NETWORK TIMEOUT / PROCESS CRASH]
```

### Critical Dual-Write Failure Scenarios
1. **Partial Writes:** If Step 1 succeeds and the process crashes before Step 2, external search indexes or caches become permanently inconsistent with the database.
2. **Concurrent Race Conditions:** If Request A sets an order status to `PAID` and Request B sets it to `CANCELLED`, they may commit in order `A -> B` in the database, but network latency may cause their Kafka messages to arrive in reverse order `B -> A`. The downstream system ends up with corrupted state.
3. **Two-Phase Commit (2PC) Overhead:** Running a distributed transaction across both the database and the message broker adds significant latency and couples their availability.

CDC resolves this by using the database's own append-only transaction log as the single source of truth for all downstream propagation.

---

## CDC Ingestion Strategies

```text
                                  CDC Strategies
        ┌───────────────────────────────┼───────────────────────────────┐
        ▼                               ▼                               ▼
 Transaction Log-Based             Database Triggers            Timestamp Polling
(WAL, Binlog, Redo Log)          (Synchronous Shadow Table)     (SELECT ... WHERE updated_at)
- Zero application query load    - Simple to configure          - Simplest to implement
- Captures all intermediate updates - Adds write latency penalty  - Misses hard deletes
- Preserves commit order         - High table maintenance       - Significant polling I/O
```

### 1. Transaction-Log-Based CDC (Recommended Standard)
Relational databases write all state modifications to an append-only sequential log (**WAL** in PostgreSQL, **Binlog** in MySQL, **Redo Log** in Oracle) before writing to table data pages on disk.

```text
Application  -->  Primary DB Engine  -->  Write-Ahead Log (WAL on Disk)
                                                 │
                                           CDC Connector (e.g., Debezium)
                                                 │ Reads Log Sequence Numbers (LSN)
                                                 ▼
                                           Event Stream (Apache Kafka)
                                                 │
                     ┌───────────────────────────┼───────────────────────────┐
                     ▼                           ▼                           ▼
               Search Index                 Redis Cache                 Data Lake
             (Elasticsearch)              (Invalidation)               (Snowflake)
```

- **Mechanism:** A connector (e.g., Debezium) connects to the database as a replication replica, reading sequential Log Sequence Numbers (LSNs) and streaming committed row changes to a message broker.
- **Advantages:** Zero performance impact on active user queries; naturally captures hard deletes; preserves exact commit ordering.
- **Trade-offs:** Requires database replication permissions; log retention policies must be sized properly to handle downstream consumer downtime.

### 2. Trigger-Based CDC
- **Mechanism:** Custom database triggers execute synchronously `AFTER INSERT/UPDATE/DELETE`, writing row copies to an internal audit shadow table. A separate process tails this audit table.
- **Trade-offs:** Every application write pays a synchronous performance penalty to execute the trigger and write to the shadow table. Triggers can fail and abort parent transactions, and shadow tables require continuous maintenance.

### 3. Timestamp Polling
- **Mechanism:** A background process queries tables at regular intervals:
  ```sql
  SELECT * FROM orders WHERE updated_at > :last_poll_time ORDER BY updated_at ASC;
  ```
- **Trade-offs:** Hard deletes leave no trace unless soft-deletes (`is_deleted = true`) are enforced; rapid updates between polling windows overwrite intermediate states; continuous table scans consume database CPU and I/O.

---

## Log-Based CDC Internals: Transaction Boundaries

CDC engines do not emit events for individual log writes as they occur. They explicitly buffer events and wait for **transaction commit boundaries**.

```text
Transaction A (Committed):
  BEGIN
  INSERT INTO users (id, name) VALUES (101, 'Alex')   --> Written to WAL
  UPDATE accounts SET balance = 500 WHERE id = 101     --> Written to WAL
  COMMIT                                              --> Written to WAL
                                                             │
                                                             ▼
                                                    CDC Emits Both Events

Transaction B (Rolled Back):
  BEGIN
  INSERT INTO orders (id, total) VALUES (501, 75.00)  --> Written to WAL
  [PROCESS CRASH / DEADLOCK DETECTED]
  ROLLBACK                                            --> Written to WAL
                                                             │
                                                             ▼
                                                    CDC Discards Event
```

### Why Commit Boundaries Matter
- **Write-Ahead Logging:** Databases write changes to disk logs *before* validating all constraints or committing to ensure durability across power failures.
- **Preventing Phantom States:** If CDC emitted events as soon as they hit the WAL, a rolled-back transaction would cause phantom records to contaminate downstream caches, search indexes, and event consumers.
- **Engine Behavior:** Connectors buffer changes in memory keyed by transaction ID. When the engine encounters a `COMMIT` record in the log, it emits the batch downstream. When it encounters a `ROLLBACK`, the buffered changes are discarded.

---

## CDC + The Transactional Outbox Pattern

When an application needs to publish domain events with rich business metadata (which may not exist as columns in the primary table), pair CDC with the **Transactional Outbox Pattern**:

```text
Application Service
  │
  ├── Single ACID Transaction:
  │     ├── 1. INSERT INTO orders (...)
  │     └── 2. INSERT INTO outbox_events (id, aggregate_type, payload, created_at)
  │
  └── Database WAL logs both inserts atomically to disk
        │
    Debezium CDC Connector (Filters strictly for outbox_events table)
        │
        ▼
    Apache Kafka Topic ("order-domain-events")
        │
        ▼
    Downstream Microservices
```

This guarantees that domain events are persisted in the same ACID transaction as the business entity, eliminating partial failure risks while avoiding application-level polling.

---

## Operational Considerations

1. **Initial Snapshot vs. Streaming:** When attaching CDC to an existing database, the connector executes an initial consistent snapshot using read-views. Once the dump completes, it seamlessly transitions to streaming WAL changes from the recorded snapshot LSN.
2. **At-Least-Once Delivery & Idempotency:** Connectors guarantee at-least-once delivery. Network interruptions, connector restarts, or consumer rebalances can cause duplicate events to be emitted. Downstream consumers must be idempotent (e.g., tracking processed event IDs or entity version numbers).
3. **Log Retention Limits:** Relational databases prune WAL/binlog files when disk thresholds are reached. If a downstream consumer goes offline longer than the log retention window, the log files are deleted, requiring a full snapshot resync.
