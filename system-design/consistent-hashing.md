# Consistent Hashing

Consistent hashing is a partitioning strategy designed for distributed caches and databases where the number of nodes in the cluster changes dynamically. It minimizes the proportion of keys that must be remapped when nodes are added or removed.

---

## The Dynamic Cluster Problem: Modulo Partitioning Failure

The simplest way to route data across `N` storage nodes is modulo hashing:

```python
target_node = hash(key) % N
```

While this distributes keys evenly when `N` is fixed, it fails in dynamic clusters:

```text
Cluster Resizing (N = 3 to N = 4):
Key     Hash    Hash % 3    Hash % 4    Data Movement
------------------------------------------------------
Key 1   14      Node 2      Node 2      No movement
Key 2   27      Node 0      Node 3      MOVES: Node 0 -> Node 3
Key 3   55      Node 1      Node 3      MOVES: Node 1 -> Node 3
Key 4   82      Node 1      Node 2      MOVES: Node 1 -> Node 2
Key 5   99      Node 0      Node 3      MOVES: Node 0 -> Node 3
```

When resizing from `N` to `N + 1`, the proportion of keys that must be migrated across the network is:

```text
Rehash Churn Rate = N / (N + 1)
```

In a 10-node cluster scaled to 11, **roughly 91% of all cached or stored data must move across the network**. 
- In distributed caches (e.g., Memcached), this causes near-total cache invalidation and a thundering herd against primary databases.
- In distributed databases, migrating 90% of terabytes of disk data saturates network interfaces and locks resources.

---

## Consistent Hashing Architecture

Consistent hashing resolves this by mapping both **storage nodes** and **data keys** to the same continuous coordinate space: a logical circular ring.

```text
                           0 / (2^32 - 1)
                           .------------.
                       .-'       |        '-.
                    .-'     [Key 1]          '-.
                  .'             |              '.
                 /               v                \
               .'           Walk Clockwise         '.
              /                                      \
             ;                                        ;
   [Node C] ●                                          ● [Node A]
            ;                                         ;
             ;                                        ;
              \                                      /
               '.                                  .'
                 \               ●                /
                  '.          [Node B]          .'
                    '-.                        .-'
                       '-.                  .-'
                          '----------------'
```

### Routing Algorithm
1. **Define the coordinate space:** A deterministic hash function (e.g., MurmurHash3, MD5) outputs an integer in the range `[0, 2^32 - 1]`. The maximum value wraps around to 0.
2. **Place nodes on the ring:** Compute `hash(node_ip)` or `hash(node_id)` to assign each physical machine a coordinate on the circle.
3. **Map keys to nodes:** Compute `hash(key)`. Traverse the ring **clockwise** from that point until encountering the first node. That node owns the key.
4. **Lookup complexity:** Store node positions in a sorted array or balanced binary search tree (e.g., `TreeMap` or `std::map`). Locating the owning node takes `O(log N)` via binary search.

---

## Node Addition and Removal Dynamics

Because routing relies on walking clockwise to the next node, membership changes only affect immediate neighbors:

```text
Adding Node D between Node A and Node B:

Before:  Node A =================================> Node B
                   (All keys in this arc map to Node B)

After:   Node A =============> Node D ============> Node B
                 (Arc maps to D)     (Arc maps to B)
```

- **Adding a Node:** The new node only absorbs keys in the arc between itself and its immediate counter-clockwise predecessor. Keys outside this range remain completely undisturbed.
- **Removing a Node:** If a node fails, its keys shift entirely to its immediate clockwise successor. No other nodes in the cluster participate in data movement.
- **Bounded Migration:** On average, adding or removing a node migrates only `1 / (N + 1)` of the total keys—a dramatic reduction compared to modulo hashing's `N / (N + 1)`.

---

## The Variance Problem & Virtual Nodes (vnodes)

Basic consistent hashing has two critical failure modes in production:

1. **Statistical Skew:** With a small number of physical nodes, random hash placement creates unequal arcs. One machine might own 60% of the ring while another owns 10%.
2. **Cascading Failure:** If Node B fails, 100% of its workload falls onto its immediate clockwise neighbor, Node C. If Node C is already running near capacity, the doubled load crashes it, cascading Node B + Node C's traffic onto Node D.

```text
Without Virtual Nodes:
Node B fails  -->  100% of Node B's load shifts directly to Node C (High risk of cascade)

With Virtual Nodes:
Node B fails  -->  Tokens are interleaved across the ring; load distributes 
                   proportionately across Node A, Node C, Node D, etc.
```

### How Virtual Nodes Work
Instead of assigning a physical server a single position, assign it `V` distinct tokens across the ring (typically 128 to 256):

```text
hash("server-01#token-0")
hash("server-01#token-1")
...
hash("server-01#token-255")
```

### Key Engineering Benefits:
- **Uniform Distribution:** By the Law of Large Numbers, hundreds of interleaved tokens distribute keys evenly across physical machines within a 1–3% variance margin.
- **Graceful Failover:** When a physical node goes offline, its tokens disappear across the entire ring. Its traffic spreads evenly across all remaining cluster members rather than a single victim.
- **Heterogeneous Hardware:** Powerful servers (e.g., 64 cores, 256 GB RAM) can be assigned 500 virtual nodes, while smaller edge nodes receive 100 virtual nodes.

---

## Traffic Skew vs. Key Distribution (The Hot-Key Problem)

Consistent hashing guarantees an even distribution of **keys**, but not an even distribution of **request traffic**. If a single key receives 100,000 requests per second (e.g., a viral product during a flash sale), the single node owning that key will be overwhelmed.

### Architectural Mitigations
- **Key-Space Salting (Write Heavy):** Append random suffixes to hot keys (`order_item_441#0` through `order_item_441#9`). Writes scatter across multiple ring nodes, and read queries fan out and aggregate.
- **Local In-Process Caching (Read Heavy):** Place an in-memory cache (e.g., Caffeine, Guava) with a short TTL (2–5 seconds) directly on the API gateway instances to absorb repetitive reads before they reach the hash ring.
- **Clockwise Read Replication:** Replicate keys to the next `R` physical nodes clockwise on the ring, allowing clients to load-balance read requests across all replicas.

---

## Data Movement in Practice: Routing vs. Migration

A common misconception is that when a node crashes, the cluster must immediately transfer terabytes of data across the network to the new owner. In reality, distributed systems separate **logical routing** from **physical data movement**:

- **Failures do not trigger reactive data movement:** Production systems (like Cassandra and DynamoDB) pre-replicate data to `R` consecutive nodes on the ring. When a primary node dies, a surviving replica immediately serves the traffic. Zero bytes are moved across the network during the failure.
- **Data movement only occurs during planned topology changes:** When you intentionally add new nodes to expand cluster capacity, or bootstrap a replacement node to restore the replication factor, background streaming processes migrate only the affected key ranges (roughly `1 / (N + 1)` of the dataset).

---

## Architectural Topologies: Ring vs. Fixed Slots

Distributed systems implement key distribution in two primary ways:

| Feature | Consistent Hash Ring (Cassandra, Ketama) | Fixed Hash Slots (Redis Cluster) |
|---|---|---|
| **Keyspace** | Continuous circle (`0` to `2^32 - 1`) | Fixed 16,384 discrete slots |
| **Routing Logic** | Hash key -> Walk clockwise to first node | `CRC16(key) % 16384` -> Slot-to-Node map |
| **Coordination** | Fully decentralized (gossip protocol) | Master node consensus / cluster state map |
| **Resharding** | Automatic based on ring arc shifts | Manual or orchestrated slot migration |
| **Best Suited For** | Highly dynamic, multi-master databases | In-memory key-value clusters with coordinated failover |
