# Time to Live (TTL)

TTL acts as an expiration mechanism for network packets to prevent them from circulating indefinitely.

- It exists as a specific field within an IP packet's header.
- Every time a router processes and forwards the packet (a "hop"), it decrements the TTL counter by 1.
- If the TTL counter reaches 0 before reaching its destination, the router discards the packet.

- Primary use case: Stopping infinite routing loops caused by misconfigured network paths.

## Practical Example

- A packet is sent with a starting TTL of 64.
- It traverses 5 intermediate routers, so the TTL drops to 59.
- If the packet gets caught in a loop and the TTL hits 0, it is automatically dropped by the network to free up bandwidth.
