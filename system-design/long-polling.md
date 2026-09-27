# Long Polling

Long polling is a communication pattern where a client (like a web browser or mobile app) requests data, and the server intentionally delays the response until new information is available.

## Core Mechanism
- The client initiates an HTTP request asking for updates.
- Server behavior:
  - Has data ready: Responds back to the client right away.
  - No data ready: Suspends the response, keeping the HTTP connection alive.
  - Once new data is generated, the server completes the response.
  - Upon receiving the response, the client instantly opens a new long-polling request to wait for the next update.
