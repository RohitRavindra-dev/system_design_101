# WebSockets — Detailed System Design Notes

---

## 1. Why WebSockets exist (Problem first, not solution)

In traditional HTTP (REST-style communication):

- Client initiates every interaction.
- Server responds and the interaction ends.
- Even with connection reuse (HTTP keep-alive), the logical model is still request → response.

### Problem in real systems

Consider systems like chat, live tracking, or notifications:

- Messages need to reach users immediately.
- Server must sometimes initiate communication.

### Workarounds with HTTP

1. **Polling**
   - Client repeatedly calls API every few seconds.
   - Wasteful: many empty responses.

2. **Long Polling**
   - Client request is held open until server has data.
   - Still inefficient and complex to scale.

### Core limitation

HTTP is fundamentally **client-driven**.

---

## 2. What WebSocket actually is (No buzzwords)

A WebSocket is:

> A long-lived TCP connection between client and server, established via an HTTP request, after which both sides can send messages independently at any time.

---

## 3. Connection Lifecycle (Step-by-step)

### Step 1 — Client sends HTTP request

```http
GET /ws HTTP/1.1
Host: example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: abc123
```

This is a normal HTTP request with special headers.

---

### Step 2 — Server validates and responds

```http
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: xyz456
```

Meaning:

- Server agrees to upgrade protocol.
- Both sides agree to keep the TCP connection open.

---

### Step 3 — Protocol switch

After this:

- HTTP is no longer used.
- Communication happens via WebSocket frames.

---

## 4. What actually flows over the network

### REST (HTTP)

```
[IP Header][TCP Header][HTTP Headers][Body]
```

### WebSocket (after upgrade)

```
[IP Header][TCP Header][WebSocket Frame][Payload]
```

### Key differences

- TCP/IP headers still exist.
- HTTP headers are NOT sent repeatedly.
- WebSocket frames are lightweight.

---

## 5. What is a “connection” technically

When a client connects:

- Server creates a **socket object**.
- This socket represents the open TCP connection.

Example (Node.js style):

```javascript
wss.on("connection", (socket, request) => {
  console.log("client connected");
});
```

### Important

The `socket` is NOT just an ID.

It is:

- A live communication channel
- A handle to send data to that specific client

---

## 6. What is connectionId / registry

Servers must track active connections.

Example structure:

```javascript
const userConnections = {
  userA: socketA,
  userB: socketB,
};
```

### Why needed?

If user A sends message to user B:

```javascript
userConnections["userB"].send("hello");
```

This allows direct delivery without another HTTP request.

---

## 7. Message flow (Single server case)

1. User A sends message via WebSocket
2. Server receives it
3. Server looks up user B’s socket
4. Server writes message into that socket

---

## 8. Multi-server problem (Horizontal scaling)

### Scenario

```
User A → Server 1
User B → Server 3
```

Server 1 does not know about sockets in Server 3.

---

### Solution: Pub-Sub

1. Server 1 publishes event:

```json
{ "to": "userB", "message": "hello" }
```

2. All servers consume the event.
3. Only Server 3 finds userB in its registry.
4. Server 3 sends message via socket.

---

## 9. API design for WebSockets

You do NOT design multiple endpoints like REST.

### Instead define:

#### 1. Connection endpoint

```
GET /ws
```

#### 2. Message format

```json
{
  "type": "SEND_MESSAGE",
  "to": "userB",
  "content": "hello"
}
```

#### 3. Server push format

```json
{
  "type": "NEW_MESSAGE",
  "from": "userA",
  "content": "hello"
}
```

---

## 10. Resource usage (Critical for scaling)

Each connection consumes:

- Memory (socket buffers, metadata)
- File descriptor

### Approximate memory usage

- 1KB–10KB per connection

### Example

| Users | Memory |
| ----- | ------ |
| 100K  | ~500MB |
| 1M    | ~5GB   |

---

### CPU usage

- Idle connections → low cost
- Active messaging → higher cost

---

## 11. Keeping connection alive

Connections can drop due to:

- Network issues
- Idle timeouts
- Server restarts

### Solution: Heartbeats

Client or server sends periodic ping:

```json
{ "type": "PING" }
```

Response:

```json
{ "type": "PONG" }
```

---

## 12. Failure scenarios

### Server crash

- All connections on that server are lost
- Clients must reconnect

### Reconnection strategy

- Exponential backoff
- Resume session if possible

---

## 13. Security considerations

### 1. Authentication

Done during handshake:

```http
Authorization: Bearer <JWT>
```

Server maps:

```
socket → userId
```

---

### 2. Do not trust client data

Bad:

```json
{ "from": "userA" }
```

Correct:

- Use authenticated identity from socket

---

### 3. Use TLS

- Use `wss://`
- Prevents packet sniffing

---

### 4. DoS attacks

- Too many open connections

Mitigation:

- Rate limiting
- Connection limits per IP

---

## 14. Mental model summary

### REST

- Stateless
- Request-response
- Client-driven

### WebSocket

- Stateful
- Persistent connection
- Bidirectional communication

---

## 15. When to use WebSockets

Use when:

- Real-time updates required
- Server needs to push data
- High-frequency interactions

Avoid when:

- Simple CRUD operations
- Low-frequency updates

---

## 16. Key trade-offs

Advantages:

- Low latency
- Efficient for continuous communication

Disadvantages:

- Harder to scale
- Stateful complexity
- Resource heavy

---

## 17. Final conceptual model

Think of WebSocket as:

- A continuously open communication channel
- Where both client and server can send messages anytime
- Without re-establishing context for every interaction

---

## End

These notes cover foundational + system design depth required for interviews.
