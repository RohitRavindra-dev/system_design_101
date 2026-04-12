# 🚀 API Design for High-Level Design (HLD) Interviews
*Strategic notes for clearing SE II / Staff-level design rounds.*

## 1. The "Contract First" Mindset
In an HLD interview, you have ~5 minutes to define your API. The goal is to establish a clear contract before discussing the internal "black box" (scaling, load balancing, databases).

### The "5-Minute" Checklist:
1.  **Endpoint Name:** Plural nouns (e.g., `/bookings`).
2.  **HTTP Method:** Clear mapping to intent (POST, GET, etc.).
3.  **Authentication:** How is the identity verified (Headers, not Body)?
4.  **Request/Response Shape:** Key fields and types.
5.  **Status Codes:** Success (2xx) and failure (4xx/5xx) scenarios.

---

## 2. Choosing the Communication Protocol


### REST (The Default)
* **Best for:** Public APIs, Web/Mobile to Backend.
* **Analogy:** **The Restaurant Menu.** You get a fixed list of items. If you want the "Burger" and the "Fries," you have to order them from their specific sections.
* **Idempotency Note:** `POST` is NOT idempotent (creates a new resource every time). `PUT` and `DELETE` are idempotent (repeated calls don't change the state further).

### GraphQL (The Efficiency Expert)
* **Best for:** Complex frontends with varying data needs.
* **Fixes:** Over-fetching (getting 50 fields when you need 2) and Under-fetching (making 3 calls to show 1 screen).
* **Analogy:** **The Grocery List.** You tell the store exactly which items you want, and they deliver them in one bag.
* **Interview Deep-Dive: The N+1 Problem.** Naive GraphQL queries can hammer your DB with 100 individual calls for related data. **Solution:** Use **Data Loaders / Batching** to group IDs into a single `SELECT` query.

### gRPC / RPC (The Internal Speed Demon)
* **Best for:** Internal Microservice communication.
* **Why?** Uses **Protocol Buffers (Binary)** instead of JSON (Text). It is 5–10x faster due to reduced serialization overhead.
* **Analogy:** **The Secret Handshake.** Both services must know the exact ritual (the `.proto` file) to communicate, but once they do, it's instantaneous.

---

## 3. Advanced Pagination: Scale Matters
If you are designing for millions of users, standard pagination will fail.



| Technique | Example | Pros/Cons |
| :--- | :--- | :--- |
| **Offset-Based** | `/events?page=2&limit=10` | Simple, but slow for deep pages. Vulnerable to "skipping" or "duplicating" items if new data is inserted while scrolling. |
| **Cursor-Based** | `/events?cursor=ID123&limit=10` | **Gold Standard.** DB query starts exactly after the last seen item. Extremely fast and consistent even with high-frequency writes. |

---

## 4. Security & Identity Management


### The "Golden Rule" of API Security:
**Never put `userId` in the Request Body.**
* **Why?** An attacker could change the body to someone else's ID.
* **Solution:** Identity must be extracted from the **Authorization Header** (using a JWT or Session Token).

### JWT (JSON Web Token) Analogy:
A JWT is like a **Digital Passport**. It contains your name, roles (e.g., `admin`), and an expiration date. It is signed by the server, so the client cannot modify it. The server trusts the passport's signature without needing to check a central database every single time.

---

## 5. Pro-Tips for "Senior" Signaling
To differentiate yourself, proactively mention these trade-offs:

1.  **Rate Limiting:** Briefly mention you'd return a `429 Too Many Requests` to protect the service from DDoS or bad actors.
2.  **Backward Compatibility:** If you change an API, how do you handle old clients? (Versioning: `/v1/events` vs `/v2/events`).
3.  **Real-time requirements:** If the system needs sub-second updates (like a ride-sharing map), shift from REST to **WebSockets** or **SSE (Server-Sent Events)**.
4.  **Bulk APIs:** If the client needs to update 100 items, don't make them call the API 100 times. Propose a `/bulk-update` endpoint to save on network round-trips.

---