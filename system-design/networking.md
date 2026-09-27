# Distributed Systems Networking

Network infrastructure is the physical and logical foundation of distributed architectures. Designing reliable, low-latency backends requires an understanding of how data flows across **transport layers**, the operational profiles of **application protocols**, **traffic routing mechanics**, and **fault mitigation patterns**.

---

## 1. Network Layering & Execution Contexts

In backend engineering, the OSI model matters primarily because of where and how packets are processed by the operating system:

```text
Layer               Primary Protocols        Processing Context   Core Responsibility
─────────────────   ──────────────────────   ──────────────────   ─────────────────────────────
L7: Application     HTTP/1-3, gRPC, WS, SSE  User Space           Domain logic, JSON/Protobuf, routes
L4: Transport       TCP, UDP, QUIC           Kernel Space         Port addressing, connection state
L3: Network         IP, ICMP, BGP            Kernel Space         Packet routing, IP addresses
L1–L2: Link/Phys    Ethernet, Fiber          Hardware / Drivers   Physical framing and signaling
```

- **Layers 3 and 4 (Kernel Space):** Processed inside the OS kernel network stack or accelerated via eBPF/DPDK. Packets are evaluated at wire speed with minimal CPU overhead.
- **Layer 7 (User Space):** Involves copying data from kernel socket buffers to user application memory, decrypting TLS cryptographic layers, and parsing application text or binary payloads. This adds significant CPU utilization and latency per hop.

### Layer 3: Network Protocol (IP) Fundamentals
The Internet Protocol (IP) provides host-to-host packet routing across heterogeneous networks:
- **Public IP Addresses:** Globally unique addresses routable across the public internet, allocated by Regional Internet Registries (RIRs like ARIN, RIPE). Core BGP routers maintain global routing tables to forward traffic to these blocks.
- **Private IP Addresses (RFC 1918):** Non-routable address spaces reserved for internal isolated networks (`10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`). Network Address Translation (NAT) maps internal instances to public egress IPs.
- **Dynamic Host Configuration (DHCP):** Automatically assigns local IP addresses, subnet masks, default gateways, and DNS resolvers to machines joining a network.

### The End-to-End Web Request Lifecycle
When a client requests a resource (e.g., `https://api.example.com/orders`), multiple distinct networking layers coordinate sequentially:

```text
1. DNS Resolution:
   Client queries local recursive resolver -> Root -> TLD (.com) -> Authoritative Nameserver
   Resolves "api.example.com" to IP "203.0.113.45" (cached locally based on DNS TTL).

2. Transport Handshake (TCP):
   Client sends SYN -> Server responds SYN-ACK -> Client completes with ACK (1 RTT).

3. Cryptographic Handshake (TLS 1.3):
   ClientHello with key share -> ServerHello with certificate and cipher selection (1 RTT).

4. Application Request (HTTP):
   Client transmits HTTP payload (headers, path, cookies, request body).

5. Server Processing & Upstream Dispatch:
   Reverse proxy / gateway terminates TLS, routes request to microservices, queries DB.

6. Application Response:
   Server transmits status code, content-negotiated payload, and caching headers.

7. Connection Management:
   HTTP Keep-Alive keeps TCP connection open for subsequent requests.
   When terminating, 4-way handshake executes: FIN -> ACK -> FIN -> ACK.
```

---

## 2. Transport Protocols: TCP, UDP, and QUIC (Layer 4)

```text
                               Transport Layer Selection
                                           │
           ┌───────────────────────────────┼───────────────────────────────┐
           ▼                               ▼                               ▼
          TCP                             UDP                            QUIC
Guaranteed, ordered stream         Fire-and-forget datagrams       UDP-based reliable transport
- 3-Way Handshake (1-RTT)          - Zero handshake latency        - 0-RTT / 1-RTT TLS 1.3
- Congestion / Flow control        - Unordered, loss-tolerant      - Stream multiplexing
- Transport Head-of-Line Blocking  - Video, VoIP, gaming, metrics  - No cross-stream HoL blocking
```

### TCP (Transmission Control Protocol)
TCP provides a connection-oriented, reliable, full-duplex byte stream.
- **Three-Way Handshake:** Establishes connection state (`SYN -> SYN-ACK -> ACK`) before transmitting application data.
- **Flow Control (Sliding Window):** The receiving host advertises its available buffer capacity, preventing the sender from overwhelming the receiver's memory.
- **Congestion Control (CUBIC, BBR):** Dynamically adjusts sending rates based on network packet drop or round-trip time variance.
- **Transport Head-of-Line (HoL) Blocking:** If an individual packet drops, all subsequent received packets on that TCP connection must wait in the OS buffer until the dropped packet is retransmitted and acknowledged.

### UDP (User Datagram Protocol)
UDP transmits stateless datagrams with an 8-byte header and zero delivery guarantees.
- Packets can arrive out of order, duplicate, or drop silently.
- **Use Cases:** Live media streaming (WebRTC), voice calls, fast-paced game state synchronization, and high-frequency metrics collection (StatsD).

### QUIC (HTTP/3 Transport)
QUIC runs on top of UDP in user space to overcome TCP's architectural constraints:
- **Integrated Encryption:** Combines transport and cryptographic handshakes (TLS 1.3), establishing connections in 1-RTT (or 0-RTT for repeat connections).
- **Independent Multiplexed Streams:** Streams run independently inside a single UDP connection. A dropped packet on Stream 1 does not block data delivery on Stream 2, eliminating transport-level Head-of-Line blocking.
- **Connection Migration:** Connections are identified by a 64-bit Connection ID rather than the network 4-tuple (IP and port). A mobile client switching from Wi-Fi to cellular keeps active transfers alive without renegotiation.

---

## 3. Application Protocols: Communication Models (Layer 7)

### HTTP Headers & Content Negotiation
HTTP headers allow clients and servers to exchange metadata and negotiate data representations dynamically:
- **Content Negotiation:** The client sends `Accept-Encoding: gzip, br` indicating supported decompression algorithms; the server compresses the payload and responds with `Content-Encoding: br`.
- **Cache Directives:** `Cache-Control: max-age=3600, stale-while-revalidate=60` dictates how CDNs and browsers store and refresh cached representations.

### gRPC & Protocol Buffers Schema Definition
gRPC defines interfaces and binary payloads using `.proto` IDL files:

```protobuf
syntax = "proto3";

package orders.v1;

service OrderService {
  // Unary RPC (Single request, single response)
  rpc GetOrder (GetOrderRequest) returns (OrderResponse);

  // Server Streaming RPC (Live shipment updates)
  rpc StreamTracking (TrackingRequest) returns (stream TrackingEvent);
}

message GetOrderRequest {
  string order_id = 1;
}

message OrderResponse {
  string order_id = 1;
  int64 user_id = 2;
  double total_amount = 3;
  string status = 4;
}
```

### WebRTC & NAT Traversal (STUN vs. TURN)
WebRTC establishes direct peer-to-peer (P2P) UDP media channels between web browsers:
- **STUN (Session Traversal Utilities for NAT):** Lightweight server that reflects the client's public IP address and port back to it. Allows peers behind regular NATs to punch direct UDP holes through firewalls for direct media streaming.
- **TURN (Traversal Using Relays around NAT):** Fallback relay server. When symmetric NATs or restrictive corporate firewalls block direct P2P connections, all encrypted media streams are relayed through the TURN server, adding bandwidth and server infrastructure costs.

### Protocol Comparison Matrix

| Protocol | Underlying Transport | Connection State | Communication Pattern | Primary Architectural Use Case |
|---|---|---|---|---|
| **REST / HTTP** | TCP / TLS | Stateless | Unidirectional (Req/Resp) | Public APIs, CRUD endpoints, Web/Mobile |
| **gRPC** | HTTP/2 (TCP) | Stateless / Stream | Unary, Streaming RPC | Low-latency internal microservice communication |
| **WebSocket** | TCP | Stateful | Full-Duplex (Bidirectional)| Interactive chat, collaborative whiteboards, trading |
| **SSE** | HTTP | Stateful connection | Server-to-Client Stream | Live sports scores, stock tickers, progress feeds |
| **WebRTC** | UDP | Peer-to-Peer | Bidirectional (Audio/Video)| Direct browser-to-browser audio/video calls |

---

## 4. Traffic Routing & Load Balancing Topologies

Load balancers distribute incoming traffic across pools of servers to maximize availability and throughput.

```text
                             Load Balancing Tiers
           ┌───────────────────────────────────┴───────────────────────────────────┐
           ▼                                                                       ▼
 Layer 4 Load Balancing                                                  Layer 7 Load Balancing
 (AWS NLB, IPVS, HAProxy TCP)                                            (AWS ALB, Envoy, NGINX)
 - Routes on IP and Port only                                            - Terminates TCP and Decrypts TLS
 - Does not inspect application payloads                                 - Inspects HTTP Paths, Headers, Cookies
 - Extremely high throughput (Millions RPS)                              - Enables path routing, canaries, rate limits
 - Ideal for WebSockets and raw TCP traffic                              - Standard for microservice ingress
```

### Load Balancing Implementations: Hardware vs. Software vs. Client-Side
- **Hardware Appliances (F5 BIG-IP, NetScaler):** Dedicated rack-mounted appliances with specialized ASICs. Expensive, highly rigid, but capable of processing tens of millions of packets per second with sub-millisecond switching latency.
- **Software Proxies (NGINX, Envoy, HAProxy, AWS ALB):** Run on commodity servers or containerized clusters. Highly programmable, support dynamic service discovery, TLS termination, and distributed tracing.
- **Client-Side Load Balancing:** The client library maintains a list of healthy backend instances (via a service discovery registry like Consul or Eureka) and directly picks a target node:
  - Eliminates the load balancer network hop.
  - Used in **Redis Cluster** (clients cache hash slot topology maps and handle `MOVED` redirects internally) and **gRPC client-side balancing**.

### Load Balancing Algorithms
1. **Round Robin / Weighted Round Robin:** Sequential distribution across instances. Ideal for stateless applications where request processing cost is relatively uniform.
2. **Least Connections:** Routes traffic to the server with the lowest count of active TCP connections. **Mandatory for stateful connections (WebSockets, SSE)** to avoid connection accumulation on older nodes.
3. **Consistent Hashing / IP Hash:** Deterministically maps client IPs to specific backends for cache warmth or local session persistence.

### Health Checking Mechanisms
- **L4 Health Checks:** Periodically sends TCP SYN packets. Verifies that the port is listening, but cannot detect application deadlocks or HTTP 500 errors.
- **L7 Deep Health Checks:** Sends an HTTP request (e.g., `GET /healthz`). The application validates internal database and cache connectivity before returning `200 OK`.

---

## 5. Latency, Anycast, and Geographic Distribution

Physical distance imposes a hard constraint on network round-trip time:

```text
Anycast BGP Edge Routing:
User in Frankfurt  -->  Closest POP Router  -->  Datacenter (Frankfurt) [IP: 198.51.100.1]
User in Tokyo      -->  Closest POP Router  -->  Datacenter (Tokyo)     [IP: 198.51.100.1]
```

### Anycast vs. Unicast
- **Unicast:** A given IP address resolves to exactly one physical server location globally.
- **Anycast:** Multiple edge locations announce the **identical IP address** via Border Gateway Protocol (BGP). Core internet routers naturally route a user's packets to the topologically closest facility.
  - Used by **Cloudflare**, **Google Public DNS (8.8.8.8)**, and **AWS Global Accelerator** to terminate TCP and TLS handshakes at the edge, reducing connection setup latency before tunneling traffic over private backbones.

---

## 6. Network Resilience & Failure Mitigation

Distributed networks fail routinely. Robust architectures implement defense-in-depth failure patterns:

### 1. Timeouts and Deadlines
Every network call must enforce an explicit timeout:
- **Connection Timeout:** Maximum duration to establish a TCP/TLS handshake.
- **Socket / Read Timeout:** Maximum idle duration waiting for incoming data bytes.
- Failing to set timeouts causes thread pool exhaustion when downstream dependencies hang.

### 2. Exponential Backoff with Full Jitter
Retrying failed network requests immediately causes **thundering herd storms** that prevent struggling downstream services from recovering.

Always add uniform random jitter to exponential backoff delays:

```text
Retry Delay = random(0, min(max_delay, base_delay * (2 ^ attempt)))
```

```text
Client 1:  ──[ 0.8s ]──> Retry 1  ────[ 3.2s ]────> Retry 2
Client 2:  ─[ 1.4s ]─> Retry 1  ──────[ 2.1s ]──────> Retry 2
Client 3:  ───[ 0.3s ]───> Retry 1  ──[ 4.5s ]──> Retry 2

Random jitter desynchronizes clients, spreading traffic evenly over time.
```

### 3. The Circuit Breaker Pattern
Prevents a service from continuously executing calls that are guaranteed to fail:

```text
                   ┌──────────────────────────────┐
                   │            CLOSED            │
                   │     (Normal Operations)      │
                   └──────────────┬───────────────┘
                                  │
                     Failure threshold exceeded
                     (e.g., 50% errors over 10s)
                                  ▼
                   ┌──────────────────────────────┐
                   │             OPEN             │
                   │  (Fail fast: reject calls,   │
                   │   return fallback data)      │
                   └──────────────┬───────────────┘
                                  │
                      Sleep window expires (e.g., 30s)
                                  ▼
                   ┌──────────────────────────────┐
                   │          HALF-OPEN           │
                   │  (Send 1 trial canary call)  │
                   └──────────┬─────────────┬─────┘
                              │             │
                      Trial Succeeds   Trial Fails
                              │             │
                              ▼             ▼
                            CLOSED         OPEN
```

- **Fail-Fast:** Immediately rejects requests without waiting for timeouts, conserving worker threads and connection pools.
- **Graceful Degradation:** Allows applications to return cached stale data or fallback defaults while downstream dependencies recover.
