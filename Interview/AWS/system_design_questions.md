60+ interviews. These System Design topics came up the most.
Last week I shared the top 10 Java questions. This is the System Design batch.
If you're prepping for senior backend roles, these are non-negotiable:

1. "Design a URL Shortener"
→ Access pattern: write once, read millions of times. Base62 encoding. DynamoDB + Redis cache. 301 vs 302 redirect.
Asked in: 4 interviews

2. "Design a Notification System"
→ SNS fan-out → per-channel SQS queues (email, SMS, push). Different latency requirements per channel.
Asked in: 3 interviews

3. "Design a Rate Limiter"
→ Token Bucket vs Sliding Window. API Gateway vs app layer. Distributed rate limiting with Redis INCR + EXPIRE.
Asked in: 3 interviews

4. "Your API is slow under load. Debug it."
→ Not a design question — a debugging question. Observability first. EXPLAIN ANALYZE. Thread pool analysis. Circuit breakers.
Asked in: 5 interviews (most frequent!)

5. "Monolith to Microservices — how?"
→ Strangler Fig. Bounded contexts. Database-per-service. Event-driven communication. Never a big bang rewrite.
Asked in: 3 interviews

6. "How do you handle data consistency across microservices?"
→ Saga Pattern. Choreography vs Orchestration. Transactional Outbox. Idempotent consumers.
Asked in: 4 interviews

7. "Design a caching strategy for [X]"
→ Never say "add Redis." Say: what you're caching, TTL, eviction policy, cache-aside vs write-through, thundering herd prevention.
Asked in: every SD round

The pattern: they don't want textbook diagrams. They want YOUR decisions and YOUR trade-offs.

"I chose DynamoDB because my access pattern is key-value with sub-ms latency" beats "I used DynamoDB" every single time.

More batches coming — React, DevOps, Behavioral.
