# System Design: Top Topics from 60+ Interviews

> If you're prepping for senior backend roles, these are non-negotiable.
> *(Last week: Top 10 Java questions. This week: System Design batch.)*

---

## 🏆 Frequency Breakdown

| # | Topic | Times Asked |
|---|-------|-------------|
| 4 | Your API is slow under load. Debug it. | ⭐ Most Frequent |
| 1 | Design a URL Shortener | 4 interviews |
| 6 | Data Consistency Across Microservices | 4 interviews |
| 2 | Design a Notification System | 3 interviews |
| 3 | Design a Rate Limiter | 3 interviews |
| 5 | Monolith to Microservices | 3 interviews |
| 7 | Design a Caching Strategy | Every SD round |

---

## 1. Design a URL Shortener
**Asked in:** 4 interviews

- **Access pattern:** Write once, read millions of times
- **Encoding:** Base62 for short key generation
- **Stack:** DynamoDB + Redis cache
- **Key decision:** 301 (permanent) vs 302 (temporary) redirect — and why it matters for analytics

---

## 2. Design a Notification System
**Asked in:** 3 interviews

- **Fan-out:** SNS → per-channel SQS queues (email, SMS, push)
- **Key insight:** Different latency requirements per channel — push can be near real-time, email can tolerate delay
- **Trade-off to discuss:** At-least-once delivery vs deduplication at the consumer

---

## 3. Design a Rate Limiter
**Asked in:** 3 interviews

- **Algorithms:** Token Bucket vs Sliding Window — know when to pick each
- **Placement:** API Gateway vs app layer — different failure modes
- **Distributed:** Redis `INCR` + `EXPIRE` for shared state across instances

---

## 4. ⭐ Your API Is Slow Under Load — Debug It
**Asked in:** 5 interviews — Most Frequent

> Not a design question. A debugging question. Observability first, always.

- **Step 1:** Metrics, traces, logs — establish a baseline before guessing
- **Step 2:** `EXPLAIN ANALYZE` on slow queries
- **Step 3:** Thread pool saturation — are you blocking on I/O?
- **Step 4:** Circuit breakers — are downstream dependencies cascading?

---

## 5. Monolith to Microservices — How?
**Asked in:** 3 interviews

- **Pattern:** Strangler Fig — never a big bang rewrite
- **Foundation:** Identify bounded contexts before splitting anything
- **Rule:** Database-per-service — shared DB defeats the purpose
- **Communication:** Event-driven over synchronous calls where possible

---

## 6. Data Consistency Across Microservices
**Asked in:** 4 interviews

- **Pattern:** Saga — either Choreography or Orchestration
- **Reliability:** Transactional Outbox to guarantee event delivery
- **Consumers:** Idempotent by design — retries must be safe
- **Key trade-off:** Eventual consistency vs distributed transactions (avoid the latter)

---

## 7. Design a Caching Strategy for [X]
**Asked in:** Every SD round

> ❌ Never just say "add Redis."
> ✅ Always say: **what**, **TTL**, **eviction policy**, **pattern**, and **failure mode**.

- **What to cache:** Hot reads, computed aggregates, session state
- **Pattern:** Cache-aside vs write-through — depends on consistency tolerance
- **TTL:** Justify it — too short = cache miss storm, too long = stale data
- **Thundering herd:** Mutex/lock on cache miss, or probabilistic early expiration

---

## The Core Pattern

> They don't want textbook diagrams.
> They want **your decisions** and **your trade-offs**.

| Weak Answer | Strong Answer |
|-------------|---------------|
| "I used DynamoDB" | "I chose DynamoDB because my access pattern is key-value with sub-ms latency requirements and no relational joins" |
| "Add Redis" | "I'd use Redis with cache-aside, 5-min TTL, LRU eviction, and a mutex to prevent thundering herd on cold start" |

---

*More batches coming — React, DevOps, Behavioral.*
