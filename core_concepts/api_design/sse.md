# Server-Sent Events (SSE) — Detailed System Design Notes

---

## 1. Problem Context and Motivation

Traditional HTTP request–response interactions are initiated by the client and completed by the server. This model is not well-suited for use cases where the server needs to continuously push updates to the client (for example: live scores, notifications, dashboards).

Common HTTP-based workarounds include:

* **Polling**: client repeatedly sends requests at intervals. This leads to unnecessary load and latency.
* **Long polling**: client sends a request that the server holds open until data is available. This reduces waste but introduces complexity and still requires repeated re-requests.

SSE addresses this by allowing a **single long-lived HTTP response** through which the server can continuously send updates.

---

## 2. Definition (Precise)

Server-Sent Events (SSE) is a mechanism where:

* The client initiates an HTTP request.
* The server responds with a special content type and keeps the connection open.
* The server continuously writes data to the response stream.
* The client receives updates incrementally without closing the connection.

This is strictly **unidirectional communication (server → client)**.

---

## 3. Connection Lifecycle

### Step 1 — Client initiates request

```http
GET /events HTTP/1.1
Host: example.com
Accept: text/event-stream
```

Important aspects:

* This is a standard HTTP request.
* No protocol upgrade is involved.

---

### Step 2 — Server responds

```http
HTTP/1.1 200 OK
Content-Type: text/event-stream
Cache-Control: no-cache
Connection: keep-alive
```

Key points:

* The server explicitly declares a streaming response.
* The connection is kept open indefinitely.

---

### Step 3 — Continuous streaming

The server writes data in chunks:

```text
data: {"score": "1-0"}


data: {"score": "2-0"}

```

Notes:

* Each event ends with a double newline (`\n\n`).
* The response is never finalized unless explicitly closed.

---

## 4. Network-Level Behavior

SSE does not introduce a new protocol. It remains within HTTP over TCP.

### Packet structure

```
[IP Header][TCP Header][HTTP Response Stream (chunked)]
```

Differences from standard HTTP:

* The response is **streamed over time**.
* HTTP headers are sent only once.
* Data is appended incrementally to the same response.

---

## 5. Event Format Specification

SSE defines a simple text-based format.

### Basic event

```text
data: Hello world

```

---

### Structured event

```text
id: 123
event: message
data: {"text": "hello"}

```

Fields:

* `data`: payload (required)
* `event`: event type (optional)
* `id`: event identifier (optional)
* `retry`: reconnection delay hint (optional)

---

## 6. Client Implementation (Browser)

Example:

```javascript
const es = new EventSource("/events");

es.onmessage = (event) => {
  console.log(event.data);
};

es.onerror = (err) => {
  console.log("connection lost or closed");
};
```

Key behaviors:

* Browser automatically manages connection.
* Automatic reconnection is built-in.

---

## 7. Server Implementation (Example)

```javascript
app.get("/events", (req, res) => {
  res.setHeader("Content-Type", "text/event-stream");
  res.setHeader("Cache-Control", "no-cache");
  res.setHeader("Connection", "keep-alive");

  const interval = setInterval(() => {
    res.write(`data: ${JSON.stringify({ time: Date.now() })}\n\n`);
  }, 1000);

  req.on("close", () => {
    clearInterval(interval);
  });
});
```

---

## 8. Connection Termination

### Server-side termination

* Server calls `res.end()`.
* Underlying TCP connection is closed.

### Client-side detection

* Browser triggers `onerror`.
* Automatic reconnection begins unless prevented.

---

## 9. Reconnection Behavior

By default, browsers automatically reconnect when:

* Network drops
* Server closes connection

### Control mechanisms

#### Server suggests retry interval

```text
retry: 5000

```

#### Stop reconnection

Server can respond with:

```http
HTTP/1.1 204 No Content
```

This instructs the browser to stop reconnecting.

---

## 10. Event Replay and Reliability

SSE supports resuming streams using event IDs.

### Server sends ID

```text
id: 123
data: update

```

### Client reconnects with header

```http
Last-Event-ID: 123
```

### Server responsibility

* Maintain event history
* Resume from last ID

---

## 11. Heartbeats and Keep-Alive

### Problem

Intermediate systems (load balancers, proxies) may close idle connections.

### Solution

Server periodically sends comments or empty events:

```text
: heartbeat

```

or

```text
data: {}

```

Purpose:

* Prevent idle timeout
* Keep TCP connection active

---

## 12. Resource Utilization

Each SSE connection consumes:

* One open TCP socket
* Memory for buffers
* File descriptor

### Characteristics

* Lightweight compared to WebSockets
* Still significant at scale (hundreds of thousands of connections)

---

## 13. Scaling Considerations

### Challenges

* Large number of concurrent open connections
* Load balancer timeouts
* Connection distribution across servers

### Typical architecture

```
Client → Load Balancer → SSE Servers → Event Source (DB / Queue)
```

### Fan-out mechanism

* Use pub-sub system
* Each server pushes updates to its connected clients

---

## 14. Limitations

* Unidirectional communication only
* Text-based (no binary)
* Limited control over connection (browser-managed)
* Not suitable for high-frequency bidirectional systems

---

## 15. Security Considerations

### Authentication

* Done during initial HTTP request
* Typically via cookies or Authorization header

### Transport security

* Use HTTPS
* Prevents interception

### Trust model

* Server must not trust client-provided data

---

## 16. Comparison with WebSockets

| Aspect          | SSE                | WebSocket            |
| --------------- | ------------------ | -------------------- |
| Direction       | Server → Client    | Bidirectional        |
| Protocol        | HTTP               | Custom after upgrade |
| Complexity      | Low                | Higher               |
| Browser support | Native EventSource | Native WebSocket     |
| Reconnection    | Automatic          | Manual               |

---

## 17. When to Use SSE

Use SSE when:

* Server pushes updates
* Client does not need real-time upstream communication
* Simplicity is preferred

Examples:

* Live scoreboards
* Notification feeds
* Monitoring dashboards

---

## 18. When Not to Use SSE

Avoid SSE when:

* Bidirectional communication required
* Binary data required
* Very high-frequency interaction needed

---

## 19. Key Mental Model

SSE is best understood as:

* A single HTTP request
* Whose response is continuously extended over time
* Allowing the server to push updates without closing the connection

---

## End
