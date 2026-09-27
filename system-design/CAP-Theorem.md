# CAP Theorem & Distributed Consistency

In distributed systems, physical networks are inherently unreliable. The CAP theorem—originally formulated by Eric Brewer—describes the core trade-offs systems must make when network communication between nodes fails.

---

## The Core Trade-Off

The theorem evaluates three fundamental system guarantees:

- **Consistency (Linearizability):** Every read receives the most recent write. All clients observe the same state simultaneously, behaving as if there is only a single centralized copy of the data.
- **Availability:** Every non-failing node returns a successful (non-error) response to every request, though it cannot guarantee the data is up to date.
- **Partition Tolerance:** The system continues operating despite arbitrary message loss, delay, or network cuts between nodes.

> **ACID Consistency vs. CAP Consistency**
> These two terms mean different things. In ACID, Consistency means preserving database integrity rules and constraints (e.g., unique keys, foreign keys). In CAP, Consistency specifically refers to single-copy serializability (linearizability) across multiple physical nodes.

---

## Why "CA" Does Not Exist in Physical Systems

A common misconception is that distributed architectures can choose any two out of the three properties. In practice, **Partition Tolerance is mandatory**.

Physical networks will drop packets, cut fiber links, or experience latency spikes indistinguishable from hardware failure. When two nodes cannot communicate, you have only two choices when a write arrives at one side:

```text
                     Network Partition Occurs
                                │
        ┌───────────────────────┴───────────────────────┐
        ▼                                               ▼
   Prioritize Consistency (CP)                 Prioritize Availability (AP)
   Reject writes on isolated nodes             Accept writes on isolated nodes
   - Prevents stale reads or split-brain       - Keeps system responsive 
   - Requests fail with errors or timeouts     - Data diverges across partitions
```

A theoretical "CA" system can only exist if network partitions are physically impossible—which means running on a single machine, not a distributed system.

---

## CP Systems: Enforcing Linearizability

Choose a CP architecture when stale reads or conflicting concurrent writes cause irreversible operational or financial errors.

### When Consistency Is Mandatory
- **Financial Ledgers & Balances:** Reading a stale balance during an authorization check can allow double-spending.
- **Inventory Allocation:** During flash sales or limited stock events, allowing writes on isolated nodes leads to overselling physical goods.
- **Distributed Coordination:** Services like etcd, Consul, and ZooKeeper maintain cluster leadership and locks; a split-brain condition would crash downstream workloads.

### Architectural Implications of CP
- **Consensus Protocols:** Nodes rely on majority quorums (Raft, Paxos) requiring `(N/2) + 1` nodes to agree before committing writes.
- **Synchronous Replication:** Writes wait for acknowledgments across multiple nodes before returning success to the client, increasing write latency.
- **Distributed Transactions (2PC):** Multi-node atomicity requires Two-Phase Commit protocols to ensure all partitions either commit or abort together.
- **Failure Behavior:** If a node ends up in a minority network partition that cannot reach a quorum, it actively refuses reads and writes to avoid serving stale data.

---

## AP Systems: Maximizing Uptime & Throughput

Choose an AP architecture when continuous availability and low latency outweigh the need for immediate global synchronization.

### When Availability Is Preferred
- **Activity Streams & Feed Items:** Showing a user profile update or post a few seconds late across geographic regions does not degrade the core user experience.
- **Telemetry & Location Tracking:** In high-volume tracking (e.g., vehicle GPS, IoT metrics), individual stale or dropped coordinates are quickly superseded by the next update.
- **Product Browsing Catalogs:** An outdated product description or review score is far better than an HTTP 500 error page.

### Architectural Implications of AP
- **Asynchronous Replication:** The primary node acknowledges the write immediately, propagating changes to read replicas in the background.
- **Multi-Leader & Leaderless Storage:** Systems like Apache Cassandra or Amazon DynamoDB allow writes on any node.
- **Conflict Resolution:** Since partitions allow divergent writes on different nodes, systems use Last-Write-Wins (LWW) or Conflict-Free Replicated Data Types (CRDTs) to reconcile state after partitions heal.

---

## The Consistency Spectrum

Distributed consistency is not merely binary (Strong vs. Eventual). Real-world databases and architectures operate across a spectrum of consistency guarantees:

```text
Strongest Guarantee                                                     Weakest Guarantee
Linearizable  ───▶  Causal Consistency  ───▶  Read-Your-Own-Writes  ───▶  Eventual Consistency
(Single global clock) (Preserves cause-effect) (Client sees own writes)   (Converges over time)
```

| Consistency Model | Guarantee | Typical Implementation / Example |
|---|---|---|
| **Linearizable (Strict)** | Real-time global ordering; all reads reflect the absolute latest write globally. | Google Cloud Spanner, etcd, distributed locks |
| **Causal Consistency** | Operations that are causally related are seen in the same order by all nodes; concurrent unrelated writes can be reordered. | Message thread replies, comment chains |
| **Read-Your-Own-Writes** | A client immediately sees its own updates, even if other users see stale data for a few seconds. | User updating profile settings or shipping address |
| **Eventual Consistency** | If no new updates are made, all replicas eventually converge to the same value; temporary read staleness is tolerated. | DNS propagation, social media view counts, product reviews |

---

## Architectural Decomposition: Feature-Level Boundaries

Production platforms are rarely 100% CP or 100% AP. Scalable systems decouple services according to their individual consistency requirements:

```text
                               Platform Architecture
                                         │
        ┌────────────────────────────────┴────────────────────────────────┐
        ▼                                                                 ▼
CP Domain (Strong Consistency)                                     AP Domain (Eventual Consistency)
- Payment processing                                               - Catalog search
- Order placement & stock reservation                              - Customer reviews
- User authentication & permissions                                - Product recommendations
- Wallet balance updates                                           - Activity streams
```

By isolating mission-critical transactional paths from high-volume read paths, systems preserve data correctness where money is involved while maintaining global low latency for general browsing.
