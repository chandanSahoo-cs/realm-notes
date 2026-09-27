# Event-Driven Architecture (EDA)

Event-Driven Architecture (EDA) is a software design pattern where decoupled services communicate asynchronously by emitting and consuming state changes (events). Rather than invoking remote endpoints directly, services publish records of past occurrences, allowing downstream systems to react independently.

---

## Why Event-Driven Architecture?

```text
Direct Synchronous Coupling (RPC / REST):
[ Order Service ] ──▶ [ Inventory Service ]
                  ──▶ [ Notification Service ]
                  ──▶ [ Analytics Service ]
- Tight Coupling: Order Service must know the network address and API contract of every downstream system.
- Cascading Latency: Total checkout latency is the sum of all downstream calls.
- Fragility: If the Notification Service times out, the order creation flow risks failing.

Event-Driven Architecture:
[ Order Service ] ──▶ Publishes "OrderPlaced" ──▶ [ Event Broker (Kafka) ]
                                                            │
                                ┌───────────────────────────┼───────────────────────────┐
                                ▼                           ▼                           ▼
                      [ Inventory Service ]       [ Notification Service ]    [ Analytics Service ]
```

### Core Architectural Benefits
1. **Producer-Consumer Decoupling:** The producing service has zero awareness of which services consume its events, or how many consumers exist. New consumer services can be added without modifying producer code.
2. **Failure Isolation (Temporal Decoupling):** If the notification service is experiencing downtime or network issues, events safely queue in the broker. When the service recovers, it resumes consumption without data loss.
3. **Independent Scalability:** Compute-heavy consumers (e.g., analytics processing or image resizing) can scale horizontally without affecting the throughput of the transaction-processing producer.

---

## Core Event-Driven Patterns

Distributed systems implement EDA using four primary event propagation patterns:

```text
1. Simple Event Notification (Thin Events)
   Producer emits minimal event:
   {
     "event_type": "ORDER_CREATED",
     "order_id": 9921,
     "timestamp": 1710000000
   }
   Downstream Consumer Workflow:
   1. Consumer receives event.
   2. Consumer calls Order Service DB: SELECT * FROM orders WHERE id = 9921;
   - Pros: Low network payload size; zero stale data in the event broker.
   - Cons: Thundering herd against producer database when multiple consumers query for details.

2. Event-Carried State Transfer (Fat Events)
   Producer emits the complete entity payload:
   {
     "event_type": "ORDER_CREATED",
     "order_id": 9921,
     "customer_id": 44,
     "total_amount": 129.50,
     "items": [ { "sku": "A1", "qty": 2, "price": 40.00 } ],
     "shipping_address": { "city": "Seattle", "zip": "98101" }
   }
   Downstream Consumer Workflow:
   - Consumer extracts all required attributes directly from the message payload.
   - Pros: Completely eliminates round-trip queries back to the producer database.
   - Cons: Larger message sizes increase broker memory, disk storage, and network serialization costs.

3. Event Sourcing
   Instead of storing only the current mutable state of an entity (e.g., balance = $150),
   the system persists the entire immutable sequence of state-changing events:
   - Event 1: AccountOpened (Initial: $0)
   - Event 2: MoneyDeposited (+$200)
   - Event 3: MoneyWithdrawn (-$50)
   - Current State: Derived by replaying events from genesis.
   - Strengths: Complete audit trail, time-travel debugging, and native event replay.

4. Command Query Responsibility Segregation (CQRS)
   Separates the model that writes data (Commands) from the model that reads data (Queries):
   - Command Model (Write): Highly normalized relational database optimized for ACID validation.
   - Query Model (Read): Denormalized Elasticsearch or Redis view updated asynchronously via events.
```

---

## Pattern Comparison: Simple Notification vs. State Transfer

| Dimension | Simple Event Notification (Thin) | Event-Carried State Transfer (Fat) |
|---|---|---|
| **Event Size** | Minimal (IDs and timestamps only) | Large (Full entity snapshot) |
| **Broker Storage Overhead**| Low | High |
| **Producer DB Load** | High (Every consumer queries producer DB) | Zero (Consumers are self-sufficient) |
| **Data Freshness** | Always fresh (Fetched live from source DB) | Snapshot in time (May drift if reordered) |
| **Coupling** | Tightly couples consumers to producer API/DB | Fully decoupled from producer infrastructure |
| **Best Suited For** | Internal workflows with few consumers | Enterprise event buses with many consumers |
