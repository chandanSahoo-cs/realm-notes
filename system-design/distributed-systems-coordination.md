# Distributed Coordination, Consensus & Quorums

A distributed system coordinates a cluster of independent physical machines over a network, presenting them to clients as a single unified service. The central challenge of distributed computing is managing state, detecting node failures, and achieving consensus across unreliable networks.

---

## 1. Leader-Follower Coordination & Auto-Recovery

Most stateful distributed systems assign distinct roles to nodes to serialize writes and maintain cluster order:

```text
                                  Client Request
                                        │
                                        ▼
                             ┌─────────────────────┐
                             │   Leader / Master   │
                             │  (State Coordinator)│
                             └──────────┬──────────┘
                                        │
                 ┌──────────────────────┼──────────────────────┐
                 ▼                      ▼                      ▼
        ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
        │   Follower 1    │    │   Follower 2    │    │   Follower 3    │
        │ (Read Replica)  │    │ (Read Replica)  │    │ (Read Replica)  │
        └─────────────────┘    └─────────────────┘    └─────────────────┘
```

- **Leader Role:** Accepts client writes, orders transactions, coordinates replication, and assigns work partitions to followers.
- **Follower Role:** Replicates state, serves read traffic, and participates in heartbeats.
- **The Orchestrator / Controller Pattern:** Dedicated supervisors monitor the health of worker instances. To prevent the supervisor itself from becoming a single point of failure, orchestrators run in a clustered quorum (e.g., Kubernetes Control Plane, ZooKeeper ensembles), electing an active leader among themselves.

---

## 2. Leader Election Algorithms

When an active leader crashes or partitions from the cluster, the remaining nodes must elect a new leader automatically:

```text
The Bully Algorithm (Highest ID Wins):
1. Node 2 detects Leader 5 is dead (Heartbeat timeout).
2. Node 2 sends ELECTION messages to all nodes with higher IDs (Nodes 3, 4, 5).
3. If Node 3 and Node 4 respond "OK", Node 2 steps down.
4. Node 4, having the highest active ID, bullies lower nodes and broadcasts "COORDINATOR".
```

| Algorithm | Topology | Message Complexity | Operational Profile |
|---|---|---|---|
| **Bully Algorithm** | Fully connected | `O(N^2)` worst-case, `O(N)` best | Nodes with higher IDs take priority; simple but chatty. |
| **Ring / LCR Algorithm**| Logical ring | `O(N^2)` messages | IDs pass clockwise; node with highest ID wins. |
| **HS Algorithm** | Logical ring | `O(N log N)` messages | Bidirectional token passing in doubling phases (`2^k`). |
| **Gossip Protocol** | Mesh / Decentralized | `O(log N)` rounds | Nodes exchange periodic heartbeats (every 1–3s) with random peers to detect failures probabilistically. Used in Cassandra and Consul. |
| **Raft Consensus** | Quorum / Majority | `O(N)` per election | Randomized election timeouts; candidates request votes; winner requires majority quorum `(N/2) + 1`. Modern standard (etcd, CockroachDB). |

---

## 3. Quorum-Based Consistency Mathematics

Leaderless and multi-master databases (e.g., Apache Cassandra, Amazon DynamoDB) use quorum intersection to control consistency without a single leader:

```text
Parameters:
N = Replication Factor (Total number of replicas storing a key)
W = Write Quorum (Number of replicas that must acknowledge a write before success)
R = Read Quorum (Number of replicas that must respond to a read request)
```

```text
Strong Consistency (Quorum Intersection):
                  W + R > N

Example (N = 5, W = 3, R = 3):
W + R = 6 (which is > 5)

Replicas:    [ Node 1 ]   [ Node 2 ]   [ Node 3 ]   [ Node 4 ]   [ Node 5 ]
Write hits:  [ WRITE  ]   [ WRITE  ]   [ WRITE  ]
Read hits:                             [ READ   ]   [ READ   ]   [ READ   ]
                                            ▲
                                            │
               Node 3 is guaranteed to overlap between write and read sets!
```

- **Strong Consistency (`W + R > N`):** The write quorum and read quorum are mathematically guaranteed to overlap on at least one replica. That replica returns the latest version (timestamp), guaranteeing linearizability.
- **Eventual Consistency (`W + R <= N`):** Writes and reads require fewer acknowledgments (e.g., `W = 1, R = 1`), reducing network latency. However, a read might query only replicas that have not yet received the latest write.

---

## 4. Distributed Batch & Stream Processing (Big Data Engines)

When a dataset exceeds the memory, disk, and CPU limits of a single machine, computation must be distributed across a cluster:

```text
                                  Client Job Submission
                                            │
                                            ▼
                              ┌───────────────────────────┐
                              │    Coordinator / Driver   │
                              │ (DAG Planning & Scheduler)│
                              └─────────────┬─────────────┘
                                            │
                 ┌──────────────────────────┼──────────────────────────┐
                 ▼                          ▼                          ▼
        ┌──────────────────┐       ┌──────────────────┐       ┌──────────────────┐
        │ Worker Node A    │       │ Worker Node B    │       │ Worker Node C    │
        │ - Computes Part 1│       │ - Computes Part 2│       │ - Computes Part 3│
        └──────────────────┘       └──────────────────┘       └──────────────────┘
```

- **Coordinator Responsibilities:** Deconstructs a high-level job (e.g., SQL query or ML training pipeline) into a Directed Acyclic Graph (DAG) of discrete tasks, partitions input data, and assigns tasks to workers based on data locality.
- **Fault Recovery:** If a worker node crashes mid-computation, the coordinator reschedules only the lost partition's task onto another healthy worker.
- **Engines:** **Apache Spark** (in-memory Resilient Distributed Datasets for batch processing) and **Apache Flink** (low-latency stateful stream processing).
