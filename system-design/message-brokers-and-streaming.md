# Message Brokers & Event Streaming

Distributed systems use asynchronous messaging to decouple services, buffer traffic spikes, and execute long-running background tasks without blocking client request-response cycles.

---

## Synchronous vs. Asynchronous Communication

```text
Synchronous (Request-Response):
Client  ──[ HTTP POST /checkout ]──▶  API Gateway  ──[ RPC ]──▶  Order Service  ──[ RPC ]──▶  Payment Service
Client  ◀──[ 200 OK Order Placed ]──  API Gateway  ◀────────────  Order Service  ◀────────────  Payment Service
- Blocking: Client connection stays open until the slowest upstream dependency completes.
- Cascading Failure: If Payment Service hangs or crashes, the entire checkout request fails.

Asynchronous (Event-Driven / Queue-Based):
Client  ──[ HTTP POST /checkout ]──▶  API Gateway  ──▶  Order DB (Write Order "PENDING")
                                                │
                                                ├──▶  Publish Event to Message Broker
                                                │
Client  ◀──[ 202 Accepted ]─────────────────────┘
                                                │
                                                ▼
                                         [ Message Broker ]
                                                │
                                    ┌───────────┴───────────┐
                                    ▼                       ▼
                           Payment Consumer         Inventory Consumer
                           (Processes charge)       (Deducts stock)
```

### When to Use Asynchronous Messaging
- **Long-Running Computations:** Video transcoding, PDF generation, ML model inference, and report generation that exceed standard HTTP timeout thresholds (e.g., > 2–5 seconds).
- **Non-Critical Path Operations:** Sending verification emails, updating search indexes, dispatching push notifications, and logging analytics events.
- **Traffic Smoothing / Load Leveling:** Buffering sudden bursts of writes (e.g., telemetry pings, flash sales) so downstream databases and workers can ingest data at a controlled, sustainable rate.

---

## Message Queues vs. Message Streams

```text
Message Queue (Point-to-Point / Task Queue)
[ Producer ] ──▶ [ Queue: Task 1, Task 2, Task 3 ]
                           │          │          │
                           ▼          ▼          ▼
                      [Worker 1] [Worker 2] [Worker 3]
- Destructive Read: Once Worker 1 processes Task 1, it is permanently deleted from the queue.
- Competing Consumers: Multiple workers pull from the same queue to distribute the workload.
- Single Consumer Purpose: Designed for task dispatch where each job is executed exactly once.

Message Stream (Append-Only Distributed Log)
[ Producer ] ──▶ [ Partitioned Log: Event 1, Event 2, Event 3 ... (Retained on Disk) ]
                           │                             │
                           ▼                             ▼
              [ Consumer Group A: Analytics ]   [ Consumer Group B: Search Indexer ]
              (Offset: 2)                       (Offset: 1)
- Non-Destructive Read: Events remain stored on disk according to a retention policy (e.g., 7 days).
- Independent Consumer Groups: Multiple distinct systems read the same log at their own pace.
- "Write Once, Read Many": Ideal for event broadcasting and event sourcing.
```

### Comparison Matrix

| Dimension | Message Queues (RabbitMQ, AWS SQS) | Message Streams (Apache Kafka, AWS Kinesis) |
|---|---|---|
| **Underlying Model** | Ephemeral Task Queue | Append-Only Distributed Commit Log |
| **Message Consumption** | Destructive (Deleted after ACK) | Non-destructive (Tracked via consumer offsets) |
| **Consumer Model** | Competing consumers for task division | Multiple independent consumer groups |
| **Message Ordering** | FIFO per queue (lost under concurrent workers) | Strictly ordered per partition |
| **Data Retention** | Removed once processed | Retained by time (e.g., 7 days) or log compaction |
| **Replayability** | No (Cannot rewind processed tasks) | Yes (Consumers can reset offsets to replay history) |
| **Primary Use Case** | Async task execution, worker pools | High-throughput streaming, CDC, event buses |

---

## Apache Kafka Architecture & Partition Mechanics

Apache Kafka is a distributed event store and stream-processing platform designed for horizontal scalability, high write throughput, and fault tolerance.

```text
                                  Kafka Cluster
┌─────────────────────────────────────────────────────────────────────────────────┐
│                                Topic: "orders"                                  │
│                                                                                 │
│   Partition 0: [ Msg 0 ] [ Msg 1 ] [ Msg 2 ] [ Msg 3 ] ──▶ Consumer 1 (Group A) │
│                                                                                 │
│   Partition 1: [ Msg 0 ] [ Msg 1 ] [ Msg 2 ] ────────────▶ Consumer 2 (Group A) │
│                                                                                 │
│   Partition 2: [ Msg 0 ] [ Msg 1 ] ─────────────────────▶ Consumer 3 (Group A) │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### Core Components
- **Broker:** An individual Kafka server node that stores partitions and serves client requests.
- **Topic:** A logical channel or category to which records are published (e.g., `user_signups`, `payment_events`).
- **Partition:** The fundamental unit of parallelism and storage in Kafka. Each topic is split across one or more partitions distributed across brokers.
- **Consumer Group:** A set of cooperating consumers that divide the partitions of a topic among themselves.

### Partition Assignment & Consumer Scaling Rules
Kafka enforces a strict relationship between topic partitions and consumer group members:

1. **One Consumer Per Partition:** Within a single consumer group, each partition is assigned to **at most one** consumer thread.
2. **Strict Ordering Guarantee:** Messages within a partition are strictly ordered by sequential 64-bit offsets. Because only one consumer reads a partition, Kafka guarantees strictly ordered processing per partition.
3. **The Consumer Scaling Limit:**
   - If a topic has **4 partitions** and a consumer group has **4 consumers**, each consumer reads 1 partition.
   - If you scale the consumer group to **6 consumers**, **2 consumers will sit completely idle** with zero assigned work.
   - **Rule:** To horizontally scale consumer processing capacity, you must provision at least as many topic partitions as concurrent consumer instances.

```text
Topic with 3 Partitions:
[ Partition 0 ] ──────▶ Consumer 1
[ Partition 1 ] ──────▶ Consumer 2
[ Partition 2 ] ──────▶ Consumer 3
                        Consumer 4 (IDLE - No partition available)
```

---

## Real-Time Pub/Sub vs. Message Brokers

```text
Message Broker (Pull-Based):
Producer  ──▶  [ Broker Queue ]  ◀── Poll / Pull (API / SDK) ──  Consumer Worker
- Consumer actively pulls batches of messages when ready.
- Messages wait in the broker until a consumer is available.

Real-Time Pub/Sub (Push-Based):
Publisher ──▶  [ Pub/Sub Channel (Redis) ]  ─── Immediate Push ──▶  Subscriber 1
                                            ─── Immediate Push ──▶  Subscriber 2
- Ephemeral delivery: The broker immediately forwards the message to all active TCP connections.
- Zero persistence: If no subscriber is connected at the millisecond the message arrives, it is lost forever.
```

### Scaling Stateful WebSockets Using Redis Pub/Sub
A classic distributed architecture challenge is routing messages between users connected to different WebSocket servers:

```text
Client A (User 1)               Client B (User 2)
      │                               │
  WebSocket                       WebSocket
      ▼                               ▼
[ WS Server 1 ]                 [ WS Server 2 ]
      │                               ▲
      │ 1. Publish to "chat:room_42"  │ 2. Delivers message to Client B
      ▼                               │
┌─────────────────────────────────────┴─────────────────────────────────────┐
│                       Redis Pub/Sub Message Bus                           │
│                   Channel: "chat:room_42" (In-Memory)                     │
└───────────────────────────────────────────────────────────────────────────┘
```

1. **The Problem:** Client A connects to WebSocket Server 1, while Client B connects to WebSocket Server 2. Server 1 cannot directly communicate with Server 2's memory.
2. **The Solution:** Both servers subscribe to the Redis Pub/Sub channel `chat:room_42`. When Client A sends a chat message, Server 1 publishes it to Redis. Redis pushes the payload to Server 2, which emits it down Client B's active WebSocket connection.
