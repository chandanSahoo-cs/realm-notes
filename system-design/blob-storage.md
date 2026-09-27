# Blob & Object Storage Architecture

Blob (Binary Large Object) storage is a flat storage architecture designed to store massive volumes of unstructured binary data—such as video files, images, PDFs, application binaries, and database backups—that do not fit neatly into traditional database tables.

---

## Why Databases Fail for Binary Blobs

Storing large binary files (e.g., 50 MB images or 2 GB video files) directly in relational (PostgreSQL, MySQL) or document (MongoDB) databases causes severe architectural degradation:

```text
Database Engine (NVMe Storage)                     Object Storage (AWS S3 / GCS)
┌──────────────────────────────────────┐           ┌──────────────────────────────────────┐
│  Row Data Pages & B+ Tree Indexes    │           │  Flat Key-Value Object Namespace     │
│  ┌────────────────────────────────┐  │           │  - Bucket: "media-assets"            │
│  │ User Row (id: 42, name: Alex)  │  │           │  - Key: "videos/user_42/stream.mp4"  │
│  ├────────────────────────────────┤  │           │                                      │
│  │ 500 MB Raw Video Blob (BLOB)   │  │           │  - Distributed Erasure Coding        │
│  └────────────────────────────────┘  │           │  - Pay-as-you-go capacity            │
│ Buffer Pool Memory Thrashing         │           │  - $0.023 / GB per month             │
│ $0.20 - $0.30 / GB per month         │           └──────────────────────────────────────┘
└──────────────────────────────────────┘
```

### The 4 Major Failure Modes of In-Database Blobs
1. **Buffer Pool Thrashing:** Relational databases cache active table pages in RAM (e.g., InnoDB buffer pool). Reading large blobs loads hundreds of megabytes of raw bytes into memory, aggressively evicting frequently queried B+ Tree indexes and hot relational rows.
2. **Replication & Backup Saturation:** Replicating a database with multi-terabyte blobs over write-ahead logs (WAL/Binlog) saturates inter-region network links and turns daily database backups into multi-hour operational bottlenecks.
3. **Storage Cost Disparity:** High-IOPS cloud block storage (e.g., AWS EBS gp3/io2) costs between `$0.10` and `$0.30` per GB-month. Object storage (S3 Standard) costs roughly `$0.023` per GB-month, and archive tiers drop below `$0.004` per GB-month.
4. **Connection Pool Starvation:** Clients downloading 1 GB media files directly from an API server tie up application threads and database connection sockets for minutes.

---

## Cloud Object Storage Architecture (AWS S3, Cloudflare R2)

Object storage discards hierarchical file directory trees in favor of a flat address space:

```text
https://my-bucket.s3.amazonaws.com/uploads/2026/avatar.png
       └─── Bucket ───┘           └──────── Key ────────┘
```

- **Buckets:** Top-level logical containers configured with access control, regional location, and lifecycle rules.
- **Keys:** Unique string identifiers identifying individual objects within a bucket (prefixes like `uploads/2026/` simulate virtual directories).
- **Metadata:** System metadata (Content-Type, ETag, Content-Length) and custom user-defined headers stored alongside the payload.
- **Extreme Durability (11 9s):** Services like S3 provide 99.999999999% durability by splitting objects into data and parity chunks using **Reed-Solomon erasure coding**, distributed across a minimum of three physically separated Availability Zones.

---

## Direct Client Uploads via Pre-Signed URLs

Application servers should never act as proxies for large file transfers. Instead, use **Pre-Signed URLs** to offload upload and download bandwidth entirely to object storage:

```text
Client (Web / Mobile)               Application Server / Gateway             Object Storage (AWS S3)
        │                                         │                                      │
        │─── 1. POST /api/upload-request ────────▶│                                      │
        │    { "filename": "video.mp4" }          │                                      │
        │                                         │─── 2. Validate auth & permissions    │
        │                                         │    Generate pre-signed PUT URL       │
        │                                         │    (Cryptographic signature, TTL 5m) │
        │◀── 3. Return Pre-Signed URL ────────────│                                      │
        │                                                                                │
        │─── 4. PUT https://s3.amazonaws.com/my-bucket/video.mp4?signature=... ────────▶│
        │       (Binary stream uploads directly to S3)                                   │
        │                                                                                │
        │◀── 5. 200 OK (Upload Complete) ────────────────────────────────────────────────│
        │                                         │                                      │
        │─── 6. POST /api/upload-confirm ────────▶│                                      │
        │    (Saves S3 object key in Postgres DB) │                                      │
```

### Architectural Benefits
- **Zero Server Memory Footprint:** The application server never handles or buffers incoming binary streams; its RAM and network interfaces remain available for lightweight JSON APIs.
- **Direct S3 Multipart Uploads:** For files larger than 100 MB, clients can split the file into 5 MB chunks and upload them in parallel directly to S3.
- **Security:** Pre-signed URLs grant temporary, fine-grained access (e.g., 5-minute validity) strictly for a specific object key without exposing master AWS credentials to client devices.

---

## Storage Lifecycle Management & Tiering

To optimize infrastructure spend, configure automated lifecycle transition rules that move objects through progressively cheaper storage tiers:

| Tier | Availability | Retrieval Latency | Storage Cost / GB | Typical Workload |
|---|---|---|---|---|
| **S3 Standard** | 99.99% | Milliseconds | ~$0.023 / mo | Active user uploads, frequently accessed media |
| **S3 Infrequent Access (IA)**| 99.9% | Milliseconds | ~$0.0125 / mo | Files accessed a few times a month (e.g., invoices) |
| **S3 Glacier Flexible** | 99.9% | Minutes to Hours | ~$0.0036 / mo | Compliance archives, historical audit logs |
| **S3 Glacier Deep Archive** | 99.9% | 12 to 48 Hours | ~$0.00099 / mo | Legal records retained for 7+ years |
