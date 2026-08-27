# System Design: Ticket Booking Service (Ticket Master)
> Based on a breakdown by a former Meta Staff Engineer / Interviewer. Asked frequently at Meta and other top companies.

---

## Interview Roadmap

Follow this structure for any user-facing product system design:

1. **Requirements** — Functional + Non-Functional
2. **Core Entities** — What data is persisted and exchanged
3. **APIs** — User-facing endpoints mapping to functional requirements
4. **High-Level Design** — Simple design satisfying all functional requirements
5. **Deep Dives** — Satisfy non-functional requirements; show senior-level depth

---

## 1. Requirements

### Functional Requirements

| # | Requirement |
|---|---|
| 1 | Users should be able to **search for events** |
| 2 | Users should be able to **view an event** (details, seat map) |
| 3 | Users should be able to **book a ticket** (two-phase: reserve → confirm) |

> **Tip:** Walk backwards through the user flow. Booking requires viewing; viewing requires discovery via search.

### Non-Functional Requirements

> **Biggest mistake candidates make:** Writing generic terms like "scalability" and "availability" without context. Always frame each requirement *specifically* to this problem.

| Requirement | Detail |
|---|---|
| **Strong Consistency — Booking** | No double booking. If a user in Germany books a seat, a user in the US must see it as unavailable immediately. |
| **High Availability — Search & Viewing** | It's acceptable if a newly added event doesn't appear in search results for a few seconds. |
| **High Read:Write Ratio** | ~100:1 reads to writes (many users browse; ~1% convert to a purchase). Impacts caching and scaling decisions. |
| **Scalability for Traffic Surges** | Not linear traffic — massive spikes when popular events (Taylor Swift, Super Bowl) go on sale. Design must handle this. |
| **Low Latency Search** | Full table scans on free-text queries are unacceptable at scale. |

**Out of Scope (below the line):** GDPR compliance, fault tolerance, email notifications, admin event creation.

> **Interview Tip:** After listing requirements, ask the interviewer: *"Would you like me to reprioritize anything, or move anything in or out of scope?"* Shows product thinking and alignment.

---

## 2. Core Entities

| Entity | Key Fields |
|---|---|
| **Event** | `id`, `venue_id` (FK), `performer_id` (FK), `name`, `description`, `date`, metadata |
| **Venue** | `id`, `location`, `seat_map`, metadata |
| **Performer** | `id`, `name`, metadata |
| **Ticket** | `id`, `event_id` (FK), `seat`, `price`, `status` (`available` / `booked`) |

> **Tip:** Don't exhaustively enumerate all fields up front. Identify the entities now; fields will evolve naturally as you design. Fill them in when you reach the database section.

---

## 3. APIs

> One API (sometimes more) per functional requirement. Inputs/outputs should use the core entities.

### View an Event
```
GET /events/{eventId}

Response: {
  event: Event,
  venue: Venue,
  performer: Performer,
  tickets: Ticket[]   // for rendering the seat map
}
```

### Search for Events
```
GET /events/search?term=<string>&location=<string>&type=<string>&date=<range>...

Response: PartialEvent[]   // only fields needed for search results (name, date, performer, etc.)
```

### Book a Ticket (Two-Phase)

**Phase 1 — Reserve**
```
POST /booking/reserve
Body: { ticketId: string }
Header: JWT (user identity — never pass userId in body; it's a security risk)

Response: 200 OK (ticket reserved for 10 minutes)
```

**Phase 2 — Confirm**
```
PUT /booking/confirm
Body: { ticketId: string, paymentDetails: object }
Header: JWT

Response: 200 OK (ticket booked)
```

> **Why two phases?** Mirrors real-world UX: user selects a seat → goes to payment page with a 10-minute countdown → confirms purchase. If countdown expires, ticket returns to available.

> **Security note:** Never put `userId` in the request body — it can be manipulated. Use JWT or session token in the header.

---

## 4. High-Level Design

### Architecture

- **Microservices** — Default choice for these interviews unless there's a clear reason otherwise.
- **API Gateway** — Handles routing, authentication, and rate limiting.

### Services

#### Event CRUD Service
- Handles `GET /events/{eventId}`
- Reads from **PostgreSQL DB** via joins across Event, Venue, Performer tables
- Returns event details + ticket list for the seat map

#### Search Service
- Handles `GET /events/search`
- Initial (naive) implementation: SQL `LIKE` query with wildcards
- **Problem:** Full table scan — extremely slow at scale
- Flagged for optimization in the Deep Dive

#### Booking Service
- Handles both reserve and confirm endpoints
- On **reserve:** locks the ticket
- On **confirm:** calls Stripe (third-party payment processor) → Stripe calls back via webhook → update ticket status to `booked` + associate `userId`

### Database Schema (PostgreSQL)

**Events Table**
| Column | Type |
|---|---|
| id | UUID PK |
| venue_id | UUID FK |
| performer_id | UUID FK |
| name | VARCHAR |
| description | TEXT |
| date | TIMESTAMP |
| ... | ... |

**Tickets Table**
| Column | Type |
|---|---|
| id | UUID PK |
| event_id | UUID FK |
| seat | VARCHAR |
| price | DECIMAL |
| status | ENUM (`available`, `booked`) |
| user_id | UUID FK (nullable) |

> **SQL vs NoSQL:** This debate is largely outdated. Most NoSQL databases (e.g., DynamoDB) support ACID properties and transactions today. What matters is identifying the *qualities* you need: ACID compliance, relational joins, consistency. PostgreSQL is chosen here because it clearly satisfies all of these. Either would work — just justify your choice based on requirements.

### Handling Ticket Expiry (Reservation TTL)

**Problem:** If a user reserves a ticket but never completes payment, the ticket stays "reserved" forever.

#### Option 1 — Reserved Timestamp + Query Filter
- Add `reserved_at` column to Tickets table
- Query: `WHERE status = 'available' OR (status = 'reserved' AND reserved_at < NOW() - INTERVAL '10 minutes')`
- **Downside:** Confusing data model; status doesn't reflect reality

#### Option 2 — Cron Job (Mid-Level Passing Answer)
- Cron runs every ~10 minutes
- Finds tickets with `status = 'reserved'` and expired `reserved_at`
- Resets them to `status = 'available'`
- **Downside:** Introduces lag delta `n` — a ticket might stay reserved up to 19 minutes instead of 10

#### Option 3 — Distributed Lock with TTL (Optimal / Senior+ Answer) ✅

Use **Redis** as a distributed lock store.

- On reserve: `SET ticket:{ticketId} true EX 600` (10-minute TTL)
- On expiry: Redis automatically deletes the key — no cleanup job needed
- On seat map query: Check DB for `status = 'available'`, then cross-reference Redis — if key exists, ticket is reserved; exclude it from results
- On confirm: Remove Redis key + update DB `status → booked`

**Why distributed (Redis), not in-memory in the service?**
The Booking Service scales horizontally across many instances. Each instance needs the same consistent view of the lock — an in-process cache wouldn't be shared. Redis provides a single source of truth.

**What if Redis goes down?**
- A new instance spins up immediately (detected + auto-restarted)
- Users who reserved in the last 10 minutes lose their reservation
- Multiple users could reach the payment page for the same ticket simultaneously
- PostgreSQL's ACID guarantees ensure only one write wins — others get an error
- Bad UX for a brief window, but consistency is preserved
- Acceptable risk: discuss with product team

---

## 5. Deep Dives

> For **mid-level**: The high-level design above (especially with the Redis lock) is likely sufficient.
> For **senior/staff**: Pick 1–3 areas to go deep. Lead the conversation from your non-functional requirements.

### Deep Dive 1: Low Latency Search

**Problem:** SQL `LIKE '%term%'` = full table scan. Unacceptable.

**Solution: Elasticsearch (or AWS OpenSearch)**

Elasticsearch builds an **inverted index**:
- Events are tokenized into terms: "Philadelphia Eagles Playoff Wild Card"
- Terms map to event IDs in a hash-map-like structure
- Search for "playoff" → instantly returns all events containing that term
- Also supports **geospatial queries** (quad trees + geohashing) for location-based search

**Keeping Elasticsearch in Sync with PostgreSQL**

Two approaches:

| Approach | Description | Trade-offs |
|---|---|---|
| **Dual-write (Application Code)** | Write to PostgreSQL AND Elasticsearch on every event mutation | Simple but adds complexity; must handle partial failures (one write succeeds, one fails) |
| **Change Data Capture (CDC)** | Changes to PostgreSQL are streamed (e.g., via Debezium/Kafka); a consumer updates Elasticsearch | Decoupled, resilient, but adds infrastructure; good abstraction for interviews |

> **Note:** Events/venues/performers change very rarely → write volume to Elasticsearch is low → no need for queue/batching.

> **Important:** Do NOT use Elasticsearch as your primary data store. It lacks robust transaction management and has durability concerns.

**Caching Popular Search Queries**

| Option | How | Pros | Cons |
|---|---|---|---|
| OpenSearch Node Query Cache | Built-in; caches top 10K queries per shard in LRU | No extra infra, config-only | Limited control |
| Redis / Memcached | Cache `searchTerm → results` with explicit invalidation | Flexible | Need to manage invalidation |
| **CDN API Caching** ✅ | Cache search API responses for 30–60s at edge | Fastest; geographically close to user | Less effective with many query param permutations or personalized results |

**CDN Caveats:**
- Loses effectiveness as query param permutations increase (lat/long, date combos)
- Cannot be used if search results are personalized per user

---

### Deep Dive 2: Scalability for Popular Event Surges

**Problem:** 100K–1M users simultaneously trying to book tickets to a single event. The seat map loads, then instantly goes black as every seat is taken.

#### Step 1 — Real-Time Seat Map Updates

Instead of a static snapshot, push seat status changes to clients in real time.

| Mechanism | Description | Best For |
|---|---|---|
| **Long Polling** | Client holds HTTP connection open (~30–60s); server responds when state changes; client immediately re-polls | Short sessions (<5 min); no extra infra needed |
| **Server-Sent Events (SSE)** | Persistent unidirectional connection; server pushes updates as they happen | Longer sessions; lower overhead than WebSockets |
| **WebSockets** | Bidirectional persistent connection | Overkill here — we only need server→client updates |

**Recommendation:** Start with long polling; upgrade to SSE if analytics show users staying on the page for extended periods.

#### Step 2 — Virtual Waiting Queue

**Problem:** Even with real-time updates, 1M users hitting the booking service simultaneously overwhelms backends and creates a terrible UX ("all seats gone in 200ms").

**Solution:** Gate access to the event page with a virtual waiting queue.

- Admin-configurable (enabled per event)
- Users attempting to access a high-demand event are placed in a queue
- Queue implementation: **Redis Sorted Set** (sorted by arrival time for FIFO, or randomized for fairness)
- As seats are booked, users are dequeued in batches and notified (via SSE connection) that they can proceed
- Protects backend services by throttling the inflow of active sessions

> **Key insight:** The best solution isn't always the most complex. The virtual waiting queue is simple in concept, uses existing infrastructure (Redis + SSE), and elegantly solves both the UX and the scalability problem.

---

### Deep Dive 3: Scaling for High Read:Write Ratio

#### Horizontal Scaling
- API Gateway (managed, e.g., AWS API Gateway) has built-in load balancing
- Microservices scale horizontally behind load balancers based on CPU/memory

#### Database Sharding
- Shard on **event_id** (most queries are scoped to a single event)
- If geographically distributing shards: consider **venue_id** (strong geographic correlation between users and nearby venues)
- Do the math to justify: estimate storage requirements → determine if a single PostgreSQL instance suffices or sharding is needed

#### Read Caching with Redis
- Events, venues, and performers rarely change → ideal cache candidates
- Cache: `event_id → { event, venue, performer }`
- Only tickets remain dynamic (not cached — status changes on every reserve/confirm)
- Dramatically reduces DB read load; makes `GET /events/{eventId}` very fast

---

## Interview Tips Summary

| Tip | Detail |
|---|---|
| **Don't just list buzzwords** | All systems need availability and scalability. Explain *why* each matters *for this system*. |
| **Walk backwards through the user flow** | Booking → viewing → searching. Helps derive functional requirements naturally. |
| **Flag known weaknesses early** | "This naive SQL search won't scale — I'll optimize it in the deep dive." Shows awareness. |
| **Math with purpose** | Skip back-of-envelope estimations unless the result changes your design (e.g., to determine if sharding is needed). |
| **SQL vs NoSQL is a stale debate** | Frame it as: "Here are the qualities I need. Either would work; I'll use Postgres because it clearly satisfies them." |
| **User ID in body = security risk** | Always use JWT / session tokens in headers for identity. |
| **Check your requirements at the end** | Before concluding, verify: does the design satisfy all functional AND non-functional requirements? |
| **Amend requirements as you go** | It's fine — expected even — to go back and add non-functional requirements you discover mid-design. |

---

## Glossary

| Term | Definition |
|---|---|
| **ACID** | Atomicity, Consistency, Isolation, Durability — properties guaranteeing reliable database transactions |
| **API Gateway** | Entry point that routes incoming requests to the correct microservice; handles auth and rate limiting |
| **CAP Theorem** | A distributed system can guarantee at most two of: Consistency, Availability, Partition Tolerance |
| **CDC (Change Data Capture)** | Pattern for streaming database mutations (inserts/updates/deletes) to downstream consumers in real time |
| **CDN (Content Delivery Network)** | Geographically distributed servers that cache and serve content close to the end user |
| **Cron Job** | Scheduled background task that runs at fixed time intervals |
| **Distributed Lock** | A lock stored in a shared system (e.g., Redis) so multiple instances of a service share the same concurrency control |
| **Elasticsearch / OpenSearch** | Search-optimized data store that builds inverted indexes for fast full-text and geospatial queries |
| **Geohashing** | Technique to encode lat/long coordinates into a short string for efficient proximity-based indexing |
| **Horizontal Scaling** | Adding more instances of a service (as opposed to vertical scaling = bigger machines) |
| **Inverted Index** | Data structure mapping terms to the documents containing them; foundation of search engines |
| **JWT (JSON Web Token)** | Signed token passed in HTTP headers to authenticate and identify users |
| **Long Polling** | Client keeps an HTTP connection open; server responds when new data is available; client re-polls immediately |
| **Microservices** | Architectural pattern where a system is composed of small, independently deployable services |
| **PostgreSQL** | Open-source relational database with full ACID compliance and rich query capabilities |
| **Redis** | In-memory key-value store; supports TTLs, sorted sets, pub/sub; used here for distributed locks and caching |
| **Redis Sorted Set** | Redis data structure with elements ordered by a score; used here to implement a priority queue |
| **Server-Sent Events (SSE)** | Unidirectional persistent HTTP connection allowing the server to push updates to the client |
| **Sharding** | Partitioning a database horizontally across multiple nodes, each holding a subset of the data |
| **Stripe** | Third-party payment processor; handles payment intent creation, credit card charging, and webhook callbacks |
| **TTL (Time To Live)** | Expiry duration after which a record (e.g., a Redis key) is automatically deleted |
| **Two-Phase Booking** | UX pattern: Phase 1 = reserve (locks ticket for N minutes); Phase 2 = confirm (completes payment) |
| **Virtual Waiting Queue** | Queue placed in front of a high-demand resource (event page) to meter access and protect backend services |
| **WebSocket** | Bidirectional persistent TCP connection between client and server; overkill for unidirectional push use cases |
| **Webhook** | HTTP callback registered with a third party (e.g., Stripe) that is triggered when an async event occurs |
