# API Design for System Design & HLD Interviews

These notes distill and expand on the content from "API Design in System Design Interviews w/ Meta Staff Engineer" by Hello Interview (Evan King). Use this as a revision sheet before system design / HLD rounds so you do not need to rewatch the full video.

---

## 1. Where API Design Fits in a System Design Interview

- Typical flow (Hello Interview delivery framework):
  - Requirements → Core entities (nouns) → API design → Data flow → High-level design → Deep dives.
  - API design is just one small step; do not spend more than ~5 minutes here in a 45–60 minute interview.
- Goal of API section:
  - Show that you understand how clients talk to your service and how services talk to each other.
  - Be clear and structured, not exhaustive.

**Interview tip:** If you are unsure which protocol to choose for client → backend APIs, default to REST. There is a very high chance that is the expected answer.

---

## 2. Quick Refresher: What Is an API?

- API = Application Programming Interface.
- It is a contract that lets two software components communicate using defined rules and protocols.
- In a simple 3‑tier system:
  - Client (browser/mobile app) → Backend service → Database.
  - The client calls APIs on the backend; the backend may call other services or the database via internal APIs.

### Ticketing Example

Imagine a simplified Ticketmaster‑like app:

- The browser calls your backend: "Give me all events in Bengaluru this weekend." → `GET /events?city=Bengaluru&date=2026-04-18`.
- Your backend calls the database: `SELECT * FROM events WHERE city='Bengaluru' AND date='2026-04-18';`.
- Backend formats the results (JSON) and returns them to the client.

In system design interviews you usually focus on **external APIs** (client ↔ backend), with occasional discussion of internal service‑to‑service APIs.

---

## 3. Choosing the Right API Style

For system / HLD interviews, you mainly need to know:

- REST (default for client ↔ backend)
- GraphQL (flexible client‑driven queries)
- RPC / gRPC (internal service‑to‑service)
- Real‑time protocols (only briefly mention: WebSocket, Server‑Sent Events, WebRTC)

A good mental model:

- **REST**: Simple, resource‑based, human‑readable, works great across diverse clients.
- **GraphQL**: Single endpoint, client chooses fields, reduces over‑fetching / under‑fetching.
- **RPC / gRPC**: High‑performance binary communication between internal microservices.

In most interviews, you can:

- Use **REST** for client ↔ backend.
- Mention **RPC/gRPC** for internal microservice calls (e.g., `TicketService`, `PaymentService`).
- Name‑drop real‑time protocols if the product obviously needs streaming updates (chat, live scores, collaborative editing) but keep it brief unless asked.

---

## 4. REST API Design Essentials

REST is built around:

- Standard HTTP methods (GET, POST, PUT, PATCH, DELETE).
- Resources represented as nouns in the URL path.
- JSON request/response bodies in most modern web systems.

### 4.1 Resources and URL Design

**Core idea:** URLs represent **resources** (your entities / nouns), not actions.

From the Ticketing example, typical entities:

- `Event`
- `Venue`
- `Ticket`
- `Booking`

Map them to plural, noun‑based REST resources:

```text
/events
/events/{eventId}
/venues
/venues/{venueId}
/events/{eventId}/tickets
/bookings
/bookings/{bookingId}
```

**Rules of thumb:**

- Use **plural nouns** for resource paths: `/events`, not `/event`, not `/getEvents`, not `/createEvent`.
- The **HTTP method** expresses the action (GET, POST…), not the URL.

**Bad vs good examples:**

```text
# Bad
POST /createEvent
GET  /getEvents

# Good
POST /events        # create an event
GET  /events        # list events
GET  /events/{id}   # get one event
```

In interviews, most people will not nitpick perfect REST purism, but it is very easy to get plural nouns and verbs right, so you might as well.

### 4.2 HTTP Methods (Verbs)

You only really need 5:

- **GET** – Retrieve data (no side effects; safe, idempotent).
- **POST** – Create a new resource (not idempotent: re‑sending creates duplicates).
- **PUT** – Replace an entire resource (idempotent: same request → same final state).
- **PATCH** – Partially update a resource (often idempotent in practice; updates selected fields).
- **DELETE** – Remove a resource (idempotent: deleting twice usually has same final state).

**Examples for `Event`:**

```http
GET    /events                       # list all events (usually paginated)
GET    /events/{eventId}             # get one event
POST   /events                       # create a new event
PATCH  /events/{eventId}             # update certain fields of an event
DELETE /events/{eventId}             # delete an event
```

**Interview guidance:**

- Do not over‑optimize the PUT vs PATCH vs POST discussion; mention the distinction once and move on.
- In many designs you will use mostly GET and POST; mention PATCH if you show an update API.

### 4.3 Request Inputs: Path, Query, Body

REST requests take inputs in three common places:

1. **Path parameters** – required identifiers that are part of the resource path.
2. **Query parameters** – optional filters, sorting, pagination, search terms.
3. **Request body** – JSON payload used to create or update resources.

**Path parameter example (required ID):**

```http
GET /events/123
```

- `123` is the **eventId**.
- Use path params when the identifier is strictly required to fulfill the request.

**Query parameter example (optional filters):**

```http
GET /events?city=Bengaluru&date=2026-04-18
```

- Use the `?` to start query parameters, `&` to add more.
- Use for optional filters, ordering, pagination:

```http
GET /events?city=Bengaluru&date=2026-04-18&sort=startTime&order=asc
```

**Request body example (create or update):**

```http
POST /events
Content-Type: application/json

{
  "title": "Tech Conference Bengaluru",
  "description": "Annual dev conference",
  "location": "Bengaluru",
  "startTime": "2026-05-02T10:00:00Z",
  "endTime": "2026-05-02T18:00:00Z",
  "venueId": "v_42"
}
```

**Rule of thumb:**

- Required resource identifier → **path param**.
- Optional filters / pagination → **query params**.
- Data to create/update → **request body**.

### 4.4 Responses and Status Codes

Each API response has:

- A **status code** (2xx, 4xx, 5xx).
- An optional **JSON body** with data or error details.

For interviews, you can usually group codes into ranges:

- `2xx` – Success (200 OK, 201 Created).
- `4xx` – Client error (bad request, unauthorized, not found).
- `5xx` – Server error (internal error, timeout, etc.).

**Example:**

```http
GET /events/123

200 OK
Content-Type: application/json

{
  "id": "123",
  "title": "Rock Concert",
  "location": "Bengaluru",
  "startTime": "2026-04-20T19:00:00Z",
  "venueId": "v_10"
}
```

In the interview, it is often enough to say:

> "On success we return 2xx with a list of `Event` objects; on client errors 4xx; on server errors 5xx."

You do **not** need to list every field of every response unless the interviewer pushes in that direction.

---

## 5. GraphQL Basics

GraphQL is an alternative API style initially created at Facebook to solve REST inefficiencies, especially for mobile clients on slow networks.

### 5.1 When REST Struggles

With REST, different clients (web, iOS, Android) might need slightly different shapes of data for the same screen:

- Option 1: Create multiple REST endpoints tailored to each client (sprawl of endpoints).
- Option 2: Create one "fat" endpoint that returns everything, and clients ignore what they don’t need (over‑fetching, larger payloads).

Both options are messy.

### 5.2 GraphQL Idea

GraphQL lets the client specify **exactly** what fields it needs in a single request.[page:1]

- Typically a single HTTP endpoint, e.g. `POST /graphql`.
- The request body contains a **query** written in GraphQL syntax.
- The response returns a JSON object that mirrors the query’s shape.

**Example GraphQL query for Ticketing:**

```http
POST /graphql
Content-Type: application/json

{
  "query": "query GetEvent($id: ID!) {\n    event(id: $id) {\n      name\n      date\n      venue {\n        name\n        address\n      }\n      tickets {\n        section\n        price\n        availability\n      }\n    }\n  }",
  "variables": { "id": "123" }
}
```

The response might look like:

```json
{
  "data": {
    "event": {
      "name": "Rock Concert",
      "date": "2026-04-20T19:00:00Z",
      "venue": {
        "name": "Bengaluru Arena",
        "address": "MG Road, Bengaluru"
      },
      "tickets": [
        { "section": "A", "price": 1500, "availability": "AVAILABLE" },
        { "section": "B", "price": 900,  "availability": "SOLD_OUT" }
      ]
    }
  }
}
```

With REST you might have needed multiple calls:

- `GET /events/123`
- `GET /events/123/venue`
- `GET /events/123/tickets`

GraphQL collapses this into **one** network request.

### 5.3 N+1 Problem in GraphQL

A classic performance pitfall:[page:1]

- Suppose a query asks for the **top 100 events** and, for each event, its venue and tickets.
- Naïve resolver implementation might:
  - Run 1 query for top 100 events.
  - For each of the 100 events, run 1 query to fetch the venue and 1 query for tickets.
  - That’s O(N) extra queries → the **N+1 problem**.

In practice the fix is to **batch** these queries:

- Use a **data loader** or batching layer that:
  - Collects all venue IDs needed and makes **one** query: `SELECT * FROM venues WHERE id IN (...)`.
  - Similarly batches ticket queries.

For an interview, if you mention GraphQL and the interviewer asks about performance:

> "We’d avoid N+1 by using a batching layer / data loader to fetch related entities in bulk."

### 5.4 Field‑Level Authorization

With REST, you typically secure whole endpoints.

With GraphQL:

- You can apply authorization at the **field** level.
- Example: Everyone can see event name and date, but only admins can see revenue fields.[page:1]
- Implemented inside resolvers by checking user roles before resolving certain fields.

In system design interviews, you usually only need to mention this at a high level.

---

## 6. RPC and gRPC for Service‑to‑Service APIs

Most of your API design discussion will be about external APIs, but some interviews focus heavily on backend internals.[page:1]

Example: designing post search infrastructure at a large social network.

- There may be **no direct client‑facing** endpoint in scope.
- Instead, you care about how services like `IndexService`, `RankingService`, `LoggingService` talk to each other.

### 6.1 What Is RPC?

RPC (Remote Procedure Call) is like calling a function on a remote machine.[page:1]

- Instead of thinking in terms of resources/URLs, you think in terms of **methods**.
- Example methods on a ticketing service:

```proto
service TicketService {
  rpc GetEvent(GetEventRequest) returns (Event) {}
  rpc CreateBooking(CreateBookingRequest) returns (Booking) {}
  rpc GetAvailableTickets(GetAvailableTicketsRequest) returns (AvailableTicketsResponse) {}
}
```

- Each `rpc` takes a request message and returns a response message.

### 6.2 Why gRPC / Protobuf Are Fast

gRPC commonly uses **Protocol Buffers** (Protobuf):[page:1]

- Data is encoded as compact binary instead of text (JSON).
- Messages are strongly typed and schema‑driven.
- Smaller payloads → fewer bytes on the wire → significantly lower latency and bandwidth usage (the video notes 5–10x faster in practice).[page:1]

### 6.3 Why Not Use RPC From Clients?

If RPC is so fast, why not use it between clients and backend services?[page:1]

- Public clients are diverse (browsers, iOS, Android, third‑party integrations).
- REST over HTTP:
  - Works through firewalls and proxies.
  - Uses standard tooling (curl, Postman, browser dev tools).
  - Is human readable (easy to debug JSON).
- gRPC / binary protocols require extra tooling and sometimes special gateways for browsers.

Pattern in many modern systems:

- External interface: REST (or GraphQL) over HTTP.
- Internal microservices: RPC / gRPC.

Mention this explicitly in the interview when you break down your design.

---

## 7. Pagination

Pagination is almost always a follow‑up topic for list APIs that could return many resources.[page:1]

### 7.1 Why Pagination?

Without pagination, something like:

```http
GET /events
```

could theoretically return **1 million events**:

- Huge JSON payload → high latency and memory use.
- Wasteful because the user typically only needs the first page.

So we paginate.

### 7.2 Offset / Page‑Based Pagination

Simple, common, and usually acceptable in interviews.[page:1]

**API shape:**

```http
GET /events?page=1&pageSize=25
```

Interpretation:

- `page=1, pageSize=25` → return items 1–25.
- `page=2, pageSize=25` → return items 26–50.

Typical SQL:

```sql
SELECT *
FROM events
ORDER BY created_at DESC
LIMIT 25 OFFSET 25 * (page - 1);
```

**Pros:**

- Very simple to explain and implement.
- Good for relatively static lists or when write‑rate is low.

**Cons:**

- If there are frequent insertions/deletions at the top, pages can **shift** between requests.
- Can cause duplicates or missing items when users go from page 1 to page 2.

### 7.3 Cursor‑Based Pagination

Better for high‑write or real‑time feeds (timelines, comment streams).[page:1]

**API shape:**

- First page (no cursor):

```http
GET /events?pageSize=25
```

- Server response includes a `nextCursor` (e.g., last event’s ID or timestamp):

```json
{
  "items": [ ... 25 events ... ],
  "nextCursor": "event_1452"
}
```

- Next page request:

```http
GET /events?cursor=event_1452&pageSize=25
```

On the backend, this becomes:

```sql
SELECT *
FROM events
WHERE id > 'event_1452'
ORDER BY id
LIMIT 25;
```

Because you resume **after** the last seen item, new inserts at the top do not disturb the ordering of pages you have already seen.

**Interview guidance:**

- For simple lists, say: "We’ll use page‑based pagination with `page` and `pageSize` query params."
- For feeds with high write volume, add: "To avoid duplicates and shifting pages, we can use cursor‑based pagination where the cursor is the last seen item’s ID or timestamp."

---

## 8. Security: Authentication & Authorization in APIs

Security follow‑ups often focus on:[page:1]

- Which endpoints require authentication.
- How you model roles and permissions.
- How you identify the caller (user or service).

### 8.1 Marking Authenticated Endpoints

Example in a review platform like Yelp:

- Anyone can **view** businesses.
- Only authenticated users can **write** reviews.

You can annotate this in your notes or during the interview:

```text
GET  /businesses            # public
GET  /businesses/{id}       # public
POST /businesses/{id}/reviews   [auth required]
```

For stricter roles (e.g., admin‑only operations in Ticketing):

```text
POST   /events            [auth: admin]
DELETE /events/{eventId}  [auth: admin]
```

It is okay to use such informal annotations in your diagram/whiteboard; they are not meant to be literal REST syntax.

### 8.2 How Clients Prove Who They Are

Two common approaches for web/mobile APIs:

1. **JSON Web Token (JWT)** in the `Authorization` header.
2. **Session token / cookie** that maps to server‑side session data.[page:1]

**JWT‑based example:**

```http
POST /events
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
Content-Type: application/json

{
  "title": "Private Concert",
  "location": "Secret Venue",
  "startTime": "2026-06-01T20:00:00Z"
}
```

- JWT contains claims like `userId`, `roles` (e.g., `admin`), `exp` (expiry time).
- Token is **signed**; the server verifies the signature to ensure claims have not been tampered with.

**Session‑based example:**

- Client sends a cookie or header like `Session-Id: abc123`.
- Server looks up `abc123` in a database/cache (Redis) to find user, roles, expiry, etc.

### 8.3 Common Security Pitfall in API Design

Bad design:

```http
POST /tweets
Content-Type: application/json

{
  "userId": "rohit",
  "text": "Hello world!"
}
```

If the server trusts `userId` from the body, then any caller can craft a request to post tweets on behalf of **any** user (impersonation).[page:1]

Safer design:

- Derive the acting user from the **token**, not the request body.

```http
POST /tweets
Authorization: Bearer <jwt-for-user-rohit>
Content-Type: application/json

{
  "text": "Hello world!"
}
```

On the server:

- Decode & verify JWT.
- Extract `userId` from token (e.g., `rohit`).
- Create tweet for that user; ignore any `userId` the client tries to send.

In an interview, say something like:

> "We won’t take `userId` from the request body for actions like creating tweets or bookings; we’ll derive it from the authenticated token instead."

This shows you understand practical security concerns.

---

## 9. Example: Putting It All Together (Ticketing System)

Here is a compact example section you can almost reuse verbatim in interviews.

### 9.1 Core Entities

- `User`
- `Event`
- `Venue`
- `Ticket`
- `Booking`

### 9.2 External REST APIs (Client → Backend)

```http
# Events
GET  /events                                # list events (supports filters + pagination)
GET  /events/{eventId}                      # event details

# Authenticated user actions
POST /bookings                              # create a booking for the current user

# Admin‑only operations
POST   /events                              # create event    [auth: admin]
PATCH  /events/{eventId}                    # update event    [auth: admin]
DELETE /events/{eventId}                    # delete event    [auth: admin]

# Pagination example
GET /events?city=Bengaluru&date=2026-05-01&page=1&pageSize=20
```

**Request/response example (create booking):**

```http
POST /bookings
Authorization: Bearer <jwt>
Content-Type: application/json

{
  "eventId": "e_123",
  "ticketType": "VIP",
  "quantity": 2
}
```

Possible responses:

```http
201 Created

{
  "id": "b_789",
  "userId": "u_456",
  "eventId": "e_123",
  "ticketType": "VIP",
  "quantity": 2,
  "status": "CONFIRMED"
}
```

or an error:

```http
4xx  # invalid input, sold out, unauthorized, etc.
5xx  # internal server error
```

### 9.3 Internal RPC APIs (Service ↔ Service)

Suppose you have separate services:

- `BookingService`
- `PaymentService`
- `NotificationService`

You might model gRPC contracts like:

```proto
service PaymentService {
  rpc AuthorizePayment(AuthorizePaymentRequest) returns (AuthorizePaymentResponse);
}

service NotificationService {
  rpc SendBookingConfirmation(SendBookingConfirmationRequest)
      returns (SendBookingConfirmationResponse);
}
```

The `BookingService` will:

1. Call `AuthorizePayment` via RPC.
2. Persist the `Booking` if payment succeeds.
3. Call `SendBookingConfirmation` to send email/SMS.

RPC details usually go into the "deep dives" portion if the interviewer wants more backend focus.

---

## 10. How to Use These Notes During HLD/System Design Interviews

- Spend **1–2 minutes** on the API section, not 10–15.
- Mention:
  - Protocol choice (REST / GraphQL / RPC) and why.
  - A few key endpoints with path, method, and high‑level request/response shape.
  - Pagination and auth/security at a high level.
- Only dive deeper (e.g., exact JSON schemas, GraphQL schema, RPC methods) if the interviewer explicitly asks.[page:1]

A typical spoken outline you can adapt:

> "For external APIs we’ll expose REST endpoints like `GET /events` and `POST /bookings`. We support filters and pagination via query params and derive the acting user from a JWT in the `Authorization` header. Internally, services such as Booking and Payment communicate over gRPC with protobuf contracts so we can keep service‑to‑service calls fast and strongly typed. For high‑volume listing endpoints we use page‑based pagination, and we can switch to cursor‑based pagination for feeds with heavy write traffic."

These notes should be enough to refresh you quickly before interviews without rewatching the entire video, while still being deep enough to adapt to slightly harder follow‑up questions.