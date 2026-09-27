# Back-of-the-Envelope Estimation & Capacity Planning

Back-of-the-envelope estimations calculate orders of magnitude for **throughput (QPS)**, **storage capacity**, **network bandwidth**, and **compute resources** before designing a system. These numbers dictate database selection, caching requirements, and cluster sizing.

---

## 1. Numbers Every Engineer Must Know

### Powers of 2 vs. Powers of 10 Approximation

```text
Power of 2    Exact Bytes             Power of 10    Approximation       Unit
─────────────────────────────────────────────────────────────────────────────
2^10          1,024 bytes             10^3           1 Thousand          1 KB
2^20          1,048,576 bytes         10^6           1 Million           1 MB
2^30          1,073,741,824 bytes     10^9           1 Billion           1 GB
2^40          1,099,511,627,776 bytes 10^12          1 Trillion          1 TB
2^50          1,125,899,906,842,624   10^15          1 Quadrillion       1 PB
```

### Essential Time Constants
- **1 day** = 24 hours * 3,600 seconds = **86,400 seconds** ~ **100,000 seconds** (for mental math)
- **1 million requests/day** ~ **12 requests/second**
- **100 million requests/day** ~ **1,200 requests/second**
- **1 billion requests/day** ~ **12,000 requests/second**

---

## 2. Standard Capacity Estimation Framework

When sizing a distributed architecture, calculate four primary dimensions:

```text
                    Capacity Planning Dimensions
┌───────────────────────┬───────────────────────┬───────────────────────┐
│     Throughput (QPS)  │   Storage Capacity    │   Cache & RAM Sizing  │
│  - Read vs Write QPS  │  - Daily generation   │  - 80/20 Pareto rule  │
│  - Peak multiplier    │  - 3-5 year retention │  - In-memory footprint│
└───────────────────────┴───────────────────────┴───────────────────────┘
```

### Step 1: Throughput (Queries Per Second)
Distinguish clearly between read operations and write operations:

```text
Given:
- Daily Active Users (DAU) = 100 Million
- Average writes per user/day = 5 posts
- Average reads per user/day = 100 posts viewed

Calculations:
Daily Write Volume = 100M * 5 = 500 Million writes/day
Average Write QPS  = 500,000,000 / 86,400 ≈ 5,800 writes/sec
Peak Write QPS     = 5,800 * 2 (Peak traffic multiplier) ≈ 11,600 writes/sec

Daily Read Volume  = 100M * 100 = 10 Billion reads/day
Average Read QPS   = 10,000,000,000 / 86,400 ≈ 116,000 reads/sec
Peak Read QPS      = 116,000 * 2 ≈ 232,000 reads/sec

Architectural Takeaway:
- Read-to-Write Ratio = 20:1 (Read-heavy workload -> heavily benefits from Redis/CDN caching).
```

### Step 2: Storage Capacity
Calculate uncompressed raw record size multiplied by write volume and retention window:

```text
Given:
- 500 Million writes per day
- Metadata size per record = 500 bytes
- 20% of writes include an image (average size = 2 MB)

Daily Metadata Storage:
= 500,000,000 * 500 bytes = 250 GB / day
5-Year Metadata Storage:
= 250 GB * 365 days * 5 years ≈ 456 TB

Daily Media Storage (Blob):
= (500,000,000 * 0.20) * 2 MB = 100,000,000 * 2 MB = 200 TB / day
5-Year Media Storage:
= 200 TB * 365 * 5 ≈ 365 PB

Architectural Takeaway:
- Metadata (456 TB) requires a sharded distributed database (Cassandra / Sharded PostgreSQL).
- Media (365 PB) must be stored in Object Storage (S3) with lifecycle archiving to Glacier.
```

### Step 3: Cache Sizing (The 80/20 Pareto Principle)
According to the Pareto distribution, **20% of keys generate 80% of daily read traffic**. Caches should be sized to hold this active working set in RAM:

```text
Daily Active Read Data Volume:
= 100 Million active users * 100 posts * 500 bytes ≈ 5 TB read data / day

Target Working Set (20%):
= 5 TB * 0.20 = 1 TB of RAM required for the cache cluster

Hardware Provisioning:
- 1 TB RAM can be hosted across 8 Redis nodes with 128 GB RAM each (or 16 nodes with 64 GB).
```

### Step 4: Network Bandwidth (Ingress & Egress)
```text
Ingress Bandwidth (Incoming writes):
= 5,800 writes/sec * 500 bytes ≈ 2.9 MB/s (Metadata)
+ (1,160 image writes/sec * 2 MB) ≈ 2.32 GB/s (Media Ingress)

Egress Bandwidth (Outgoing reads):
= 116,000 reads/sec * 500 bytes ≈ 58 MB/s (Metadata Egress)
+ Image views routed through CDN edge points.
```

---

## 3. Compute & Worker Sizing

To estimate the number of application server instances needed to handle high concurrency:

```text
Given:
- Peak QPS = 50,000 requests/sec
- Average CPU processing time per request = 20 ms
- Target server hardware: 8 CPU Cores

Total CPU Time Required per Second:
= 50,000 requests/sec * 20 ms = 1,000,000 ms of CPU time per second

Cores Required:
= 1,000,000 ms / 1,000 ms (1 core-second) = 1,000 CPU cores

Total Servers Required:
= 1,000 cores / 8 cores per server = 125 server instances

Provision with 30% Headroom for Spikes:
= 125 * 1.30 ≈ 163 instances (configured in an Auto Scaling Group behind an ALB)
```
