# System Design: Your API Is Slow Under Load — Debug It
> Senior / Principal Engineer Level
> Most frequently asked debugging question across system design interviews.
> Treat it like a system design: Requirements → Understand the System → Hypotheses → Investigation → Fix → Prevent

---

## Why This Question Is Asked

This is not a trivia question. The interviewer is testing:

| What They Want to See | What They're Listening For |
|---|---|
| Do you panic and guess? | Systematic, signal-driven approach |
| Do you know your observability tools? | Metrics → Traces → Logs — in that order |
| Do you know distributed systems failure modes? | Thread pools, connection pools, GC, cascading |
| Do you understand the difference between cause and symptom? | Latency is a symptom. What caused it? |
| Can you prioritize under pressure? | Triage the blast radius first, debug second |

> **Principal-level framing:** "Before I touch any code, I want to understand what signals we have and build a timeline. A slow API is a symptom, not a root cause. My job is to find the cause — and I do that by reading what the system is already telling me."

---

## Interview Roadmap

| Phase | Goal |
|---|---|
| **Step 0: Triage** | Is this an incident? Restore service first, debug second |
| **Step 1: Understand the system** | What does the request path look like end to end? |
| **Step 2: Establish a baseline** | Metrics — when did it start, what changed? |
| **Step 3: Find the slow component** | Distributed traces — where is time being spent? |
| **Step 4: Read what the system said** | Logs — what did the system tell us? |
| **Step 5: Go deep on the suspect** | Component-specific investigation |
| **Step 6: Fix + verify** | Change one thing, measure, confirm |
| **Step 7: Prevent recurrence** | Alerts, load tests, capacity planning |

---

## Step 0: Triage First — Are Users Being Impacted Right Now?

Before any debugging, answer this:

```
Is this happening right now in production?
        │
        ├── YES → Triage first
        │         │
        │         ├── Can you restart / scale out? → do it NOW, debug later
        │         ├── Can you roll back a recent deploy? → do it
        │         ├── Can you shed load (rate limit, queue)? → do it
        │         └── THEN debug with the incident timeline
        │
        └── NO → Reproduced in staging / load test → full debug mode
```

> **This is the answer that separates senior from junior.** A junior engineer starts debugging. A senior engineer asks "are users impacted right now?" and restores service before investigating cause.

---

## Step 1: Understand the System — Draw the Request Path

Before any tool, draw what a single API request touches. You cannot find a bottleneck in a system you haven't mapped.

```
Client
  │
  ▼
Load Balancer / API Gateway
  │
  ▼
API Server (your service)
  │
  ├──► Cache (Redis / Memcached)        ← miss? → hits DB
  │
  ├──► Database (PostgreSQL / MySQL)    ← slow query? lock contention?
  │
  ├──► Downstream Service A             ← slow? timing out?
  │      │
  │      └──► Its own DB / cache
  │
  └──► Message Queue (async path)       ← backlog building?
```

Every hop in this path is a candidate for the slowness. Your investigation is systematically ruling each one out.

**Ask the interviewer (or yourself):**
- Is this all endpoints slow, or one specific endpoint?
- Is it slow for all users or a subset?
- Did it start suddenly or degrade gradually?
- Was there a recent deploy, config change, or traffic spike?

These answers cut the search space in half before you look at a single metric.

---

## Step 2: Metrics — Build the Timeline

**Golden rule: never hypothesize before you have a timeline.**

The timeline answers: *what changed, when did it change, and what else changed at the same time?*

### The Four Golden Signals (Always Start Here)

```
┌─────────────────────────────────────────────────────────────────┐
│  Signal         What to Look At          What It Tells You      │
├─────────────────────────────────────────────────────────────────┤
│  Latency        p50, p95, p99            p99 spiking but p50     │
│                 (NOT average)            fine = tail latency,    │
│                                          not uniform slowness    │
│                                                                   │
│  Traffic        Requests per second      Did load increase?      │
│                 by endpoint              Which endpoint?         │
│                                                                   │
│  Error Rate     4xx rate, 5xx rate       Errors or just slow?    │
│                 by endpoint + code       What type of failure?   │
│                                                                   │
│  Saturation     CPU, memory, threads,    What is running out?    │
│                 DB connections, queue    This is the bottleneck  │
└─────────────────────────────────────────────────────────────────┘
```

> **Never use average latency.** An average of 50ms hides the reality that 1% of requests are taking 30 seconds. Always look at p95 and p99 — those are the requests your users are actually experiencing as broken.

### Resource Metrics Checklist

Pull all of these at the moment slowness started:

```
COMPUTE
  ├── CPU utilization (per core, not just average)
  ├── Memory utilization + swap usage
  ├── JVM heap used / max (if Java/Go with GC)
  ├── GC pause duration + frequency
  └── Process count / file descriptors

THREADS
  ├── Thread pool: active threads
  ├── Thread pool: queued requests
  ├── Thread pool: rejected requests        ← this is the smoking gun
  └── Thread pool: max size

DATABASE
  ├── Connection pool: active connections
  ├── Connection pool: idle connections
  ├── Connection pool: wait time            ← requests waiting for a connection
  ├── Query latency p50/p95/p99
  ├── Active queries count
  ├── Lock wait time
  └── Replication lag (if reads hit replica)

NETWORK
  ├── Bytes in/out
  ├── TCP connection count
  ├── Connection errors
  └── Retries

DOWNSTREAM DEPENDENCIES
  ├── Latency to each dependency (p50/p95/p99)
  ├── Error rate from each dependency
  └── Circuit breaker state (CLOSED / OPEN / HALF-OPEN)
```

### What the Metrics Pattern Tells You

```
Pattern                                 → Likely Cause
────────────────────────────────────────────────────────────
p99 latency high, p50 fine              → Tail latency issue
                                          (GC pause, lock contention,
                                           slow DB query on some data)

ALL latency climbing gradually          → Resource leak
                                          (memory leak, connection leak,
                                           growing queue backlog)

Latency spike at exact deploy time      → Code regression
                                          (new slow query, removed cache,
                                           sync call added to hot path)

Latency spike correlates with traffic   → Capacity issue
                                          (not enough threads/instances
                                           for the load)

One endpoint slow, others fine          → Endpoint-specific bug
                                          (bad query for that data,
                                           missing index, N+1 query)

All endpoints slow at same time         → Shared resource exhausted
                                          (DB connection pool, thread pool,
                                           shared downstream dependency)

Downstream latency spiked first         → Cascading failure
                                          (their problem became your problem)
```

---

## Step 3: Distributed Traces — Find Exactly Where Time Goes

Metrics tell you *something is slow*. Traces tell you *what exactly is slow*.

### What a Trace Looks Like

A trace captures one request's entire journey through the system, with timing for each component:

```
Trace: POST /api/checkout  (total: 8,450ms)  ← this is what the user sees
│
├── API Gateway                    12ms
│
├── Auth middleware                 8ms
│
├── CheckoutService handler         3ms
│
├── Redis cache lookup              2ms   (miss)
│
├── DB query: SELECT orders         4,200ms   ◄── HERE. This is the problem.
│   WHERE user_id = 12345
│   AND status = 'active'
│
├── PaymentService HTTP call        3,800ms   ◄── AND HERE.
│   POST /payment/validate
│
└── Response serialization          5ms

Healthy trace (same endpoint, call #5):
├── DB query: SELECT orders         12ms
└── PaymentService HTTP call        45ms
```

Now you know:
- The DB query is 350x slower than normal
- The payment service call is 84x slower than normal

Without the trace you'd be guessing. With the trace you have two confirmed suspects.

### What to Look For in Traces

```
Long span on DB call        → slow query, lock wait, missing index, pool wait
Long span on HTTP call      → downstream service degraded, network issue
Long gap between spans      → thread context switching, queue wait, GC pause
Span missing entirely       → circuit breaker open, timeout before call made
Many identical short spans  → N+1 query (loop calling DB once per item)
```

### N+1 Query — The Silent Killer

This is one of the most common causes of API slowness that looks fine at low load:

```
BAD (N+1):
  GET /orders → fetch 100 orders (1 query)
             → for each order, fetch user details (100 queries)
             → TOTAL: 101 queries
             
  At 1 RPS: 101 queries × 5ms each = 505ms (acceptable)
  At 100 RPS: 10,100 queries/second → DB overwhelmed → latency explodes

GOOD (1 query):
  SELECT orders.*, users.*
  FROM orders
  JOIN users ON orders.user_id = users.id
  WHERE orders.status = 'active'
  → 1 query regardless of result size
```

In the trace, N+1 looks like this:
```
DB: SELECT * FROM orders          5ms
DB: SELECT * FROM users WHERE id=1  5ms
DB: SELECT * FROM users WHERE id=2  5ms
DB: SELECT * FROM users WHERE id=3  5ms
... × 100
```
100 nearly identical spans in a loop = N+1. Fix with a JOIN or batch `WHERE id IN (...)`.

---

## Step 4: Logs — Read What the System Already Told You

After metrics give you the timeline and traces give you the slow component, logs give you the exact failure message.

Search logs at the moment slowness started. What you're looking for:

```
ERROR MESSAGE                                        WHAT IT MEANS
──────────────────────────────────────────────────────────────────
"Timeout waiting for connection from pool"     → DB/HTTP connection pool full
"too many connections"                         → DB max_connections hit
"Connection refused"                           → downstream service down
"Circuit breaker OPEN for payment-service"     → CB tripped, fast-failing
"java.lang.OutOfMemoryError: Java heap space"  → OOM, likely GC thrash before
"GC overhead limit exceeded"                   → JVM spending >98% time in GC
"Thread pool is full, rejecting request"       → thread saturation
"Slow query: 4523ms"                           → DB query exceeded threshold
"Lock wait timeout exceeded"                   → DB lock contention
"429 Too Many Requests from stripe.com"        → external rate limit hit
"Read timeout after 30000ms"                   → downstream took too long
```

> **Log search order:** Start broad (all ERROR/FATAL for the service) → narrow by time (2 minutes around when slowness started) → narrow by component (the one traces pointed at).

---

## Step 5: Deep Dive — The Six Root Causes

Now you've confirmed a suspect via traces + logs. Here is a complete investigation + fix for each root cause.

---

### Root Cause 1: Thread Pool Saturation

**The mechanism:**

```
Requests arrive → each grabs a thread → thread makes a DB call (blocking)
Thread is held while waiting for DB response
New requests arrive → grab more threads
DB is slow → all threads are waiting
Thread pool full → new requests queue
Queue full → requests rejected → 503 / timeout
```

**The shape of this problem:**

```
Normal load (10 RPS):            High load (100 RPS):
  10 threads in use               100 threads in use
  DB latency: 50ms                DB latency: 500ms (pool contention)
  Threads released quickly        Threads held longer
  Pool never saturates            Pool saturates → cascade
```

**How to confirm:**

```
Metric: thread_pool.active == thread_pool.max_size  (pool full)
Metric: thread_pool.queue_size > 0 and growing
Metric: thread_pool.rejected_count > 0              (the smoking gun)
Log:    "Thread pool is full" / "Request rejected from queue"
Trace:  Long gap BEFORE the first span (waiting for a thread)
```

**Fix — short term:**
```
Increase thread pool size temporarily
  BUT: this is a band-aid — more threads = more memory + more DB connections needed
  If DB is the bottleneck, more threads just means more threads waiting on DB
```

**Fix — real:**

```
Option A: Make I/O non-blocking (reactive / async)
  Java:   Spring WebFlux + R2DBC (reactive DB driver)
  Go:     goroutines (already non-blocking by design)
  Node:   already async; ensure no blocking calls in event loop
  
  Effect: One thread handles thousands of concurrent requests
          Thread is released while waiting for I/O, reused immediately
          Thread pool size becomes irrelevant

Option B: Reduce I/O latency
  Faster DB queries → threads released sooner → pool never saturates
  Add caching → eliminate DB calls entirely for read-heavy paths
  Fix downstream slowness → threads not held waiting

Option C: Backpressure + queue
  Don't reject requests — queue them with a bounded queue
  Requests wait in queue instead of failing immediately
  Set a queue timeout so requests don't wait forever
```

---

### Root Cause 2: Database Connection Pool Exhausted

**The mechanism:**

```
Connection pool: 20 connections configured
20 concurrent requests → each checks out one connection
21st request → waits for a connection to be returned
If requests are slow (slow queries) → connections held longer
→ pool wait time climbs → requests queue → timeout → 500
```

**How to confirm:**

```sql
-- PostgreSQL: see active vs idle connections right now
SELECT state, count(*)
FROM pg_stat_activity
WHERE datname = 'your_db'
GROUP BY state;

-- state = 'active' → running a query
-- state = 'idle in transaction' → DANGER: transaction open, connection held
-- state = 'idle' → connection available
```

```
Metric: db.pool.connections.active == db.pool.max_size  (pool full)
Metric: db.pool.connections.wait_time > 0 and climbing
Log:    "Timeout waiting for connection from pool after Xms"
```

**Fix:**

```
Immediate: increase pool size
  (hikari: maximumPoolSize, pg: max_connections aware of this)
  BUT: DB has its own max_connections limit — you can't increase pool
  size past what the DB supports

Real fix 1: Find and fix connection leaks
  Connections not returned to pool on exception paths
  Transactions opened but not committed/rolled back
  Use try-with-resources or connection pool leak detection
  Hikari: leakDetectionThreshold = 2000ms (logs if connection held >2s)

Real fix 2: Fix slow queries so connections are released faster
  A 5ms query holds connection for 5ms
  A 5000ms query holds connection for 5000ms
  At same RPS, pool needs 1000x more connections for slow queries

Real fix 3: PgBouncer (connection pooler in front of Postgres)
  Postgres has a hard limit on connections (default 100)
  PgBouncer sits in front, multiplexes thousands of app connections
  into a smaller number of real Postgres connections
  Transaction pooling mode: connection returned to pool after each transaction
```

**The "idle in transaction" trap:**

```sql
-- Find transactions open for more than 5 minutes (stuck)
SELECT pid, now() - pg_stat_activity.query_start AS duration, query, state
FROM pg_stat_activity
WHERE state = 'idle in transaction'
AND (now() - pg_stat_activity.query_start) > interval '5 minutes';
```

These are connections being held by transactions that started but never committed or rolled back — often due to an exception that bypassed the finally block.

---

### Root Cause 3: Slow Database Query

**The mechanism:**

Queries that run fine with 1,000 rows in dev blow up with 10,000,000 rows in production. The query plan the DB chose with a small dataset is wrong for a large one.

**How to confirm:**

```
Trace: DB span is 3,000ms instead of 3ms
Log:   "Slow query logged: 3241ms — SELECT * FROM orders WHERE..."
```

**Investigation: EXPLAIN ANALYZE**

```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT o.*, u.name, u.email
FROM orders o
JOIN users u ON o.user_id = u.id
WHERE o.status = 'pending'
AND o.created_at > NOW() - INTERVAL '7 days'
ORDER BY o.created_at DESC
LIMIT 100;
```

**Reading the output — what to look for:**

```
Seq Scan on orders  (cost=0.00..485234.00 rows=1000000)
  ↑ FULL TABLE SCAN — reading every row in the table
  ↑ This is O(N) where N = total rows
  ↑ Fine at 1K rows, catastrophic at 10M rows

Index Scan using idx_orders_status on orders
  ↑ Using an index — O(log N) lookup
  ↑ This is what you want

Nested Loop  (rows=50000 width=200)
  ↑ For each row in outer, scan inner
  ↑ Can be O(N²) if inner has no index

Hash Join  (rows=1000)
  ↑ Usually efficient — build hash table from smaller side
  ↑ But can spill to disk if rows estimate is wrong

Sort  (cost=... rows=1000000)
  ↑ Sorting 1M rows in memory is expensive
  ↑ If it says "external sort" → spilling to disk → very slow
  ↑ Fix: index on the sort column, or LIMIT before sort

actual rows=1523456  vs  rows=1000  (in the estimate)
  ↑ HUGE MISMATCH — planner chose wrong plan because statistics are stale
  ↑ Fix: ANALYZE orders; (refresh table statistics)
```

**The most common fixes:**

```sql
-- Missing index on WHERE clause columns
CREATE INDEX CONCURRENTLY idx_orders_status_created
ON orders(status, created_at DESC)
WHERE status = 'pending';   -- partial index: only indexes pending rows
                            -- much smaller, much faster

-- Missing index on JOIN column
CREATE INDEX CONCURRENTLY idx_orders_user_id ON orders(user_id);

-- Stale statistics causing wrong query plan
ANALYZE orders;
ANALYZE users;

-- Query returning too many columns
SELECT o.id, o.total, u.name   -- only what you need
FROM orders o JOIN users u ...  -- NOT SELECT *

-- Missing LIMIT causing full result set materialization
SELECT ... FROM orders LIMIT 100 OFFSET 0;  -- paginate, never load all rows
```

**Lock contention — a separate DB problem:**

```sql
-- Find blocking locks
SELECT
  blocked.pid,
  blocked.query AS blocked_query,
  blocking.pid AS blocking_pid,
  blocking.query AS blocking_query
FROM pg_stat_activity AS blocked
JOIN pg_stat_activity AS blocking
  ON blocking.pid = ANY(pg_blocking_pids(blocked.pid))
WHERE cardinality(pg_blocking_pids(blocked.pid)) > 0;
```

Long-running transactions (e.g. a batch job) can hold row-level locks that block your API queries. The API query waits → thread is held → pool exhausts → 503.

**Fix:** Kill the blocking query (`SELECT pg_terminate_backend(pid)`), then fix the root cause — batch jobs should use smaller transactions, or run during off-peak with `NOWAIT` / `SKIP LOCKED`.

---

### Root Cause 4: Memory Leak → GC Thrash → OOM

**The mechanism:**

```
Request 1  → allocates objects → response sent → GC collects → memory freed
Request 100 → some objects escape (not collected — references held)
Request 1000 → more objects escaped → heap slowly filling
Request 10000 → heap 90% full → GC runs constantly
                JVM spending 80% of time in GC, 20% doing work
                Latency climbs because GC stop-the-world pauses
                Eventually: OutOfMemoryError → process dies → 502
```

**The signature of this problem:**

```
Metric: jvm.memory.heap.used — grows monotonically, never stabilizes
Metric: jvm.gc.pause.duration — increasing over time
Metric: jvm.gc.collections_per_minute — increasing over time
Metric: process.cpu — high even with moderate load (GC is CPU-heavy)
Log:    "java.lang.OutOfMemoryError: Java heap space"
        (or: "GC overhead limit exceeded")
```

**GC pause pattern in latency:**

```
Normal:          ──────────────────────── p99: 20ms
After 1hr leak:  ─────────────┬──────────────────────┬─── p99: 2000ms
                              │                      │
                           GC pause              GC pause
                           (stop-the-world)
```

**Investigation: Heap Dump**

```bash
# Capture heap dump while the process is alive (before OOM)
jmap -dump:format=b,file=/tmp/heap-$(date +%s).hprof <pid>

# Or configure JVM to auto-dump on OOM:
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/tmp/heapdumps/

# Analyze with Eclipse MAT or VisualVM
# Look for: "Leak Suspects" report
# Look for: objects with many instances that shouldn't accumulate
```

**Common memory leak patterns:**

```
Static collections that grow unboundedly:
  private static Map<String, Object> cache = new HashMap<>();
  // cache.put() called on every request, never evicted
  // Fix: use Guava Cache / Caffeine with size/time eviction

ThreadLocal values not cleaned up:
  ThreadLocal<Context> ctx = new ThreadLocal<>();
  // Thread pool reuses threads — old ThreadLocal value survives
  // Fix: always call ctx.remove() in finally block

Event listeners never unregistered:
  eventBus.subscribe(this);
  // Object subscribed but never unsubscribed
  // Object cannot be GC'd because eventBus holds a reference
  // Fix: unsubscribe in cleanup/destroy

Unclosed streams / connections:
  InputStream is = url.openStream();
  // Exception thrown before is.close()
  // Fix: try-with-resources
```

---

### Root Cause 5: Cascading Failure from Downstream Dependency

**The mechanism:**

```
Your API calls Payment Service for every checkout request
Payment Service becomes slow (their DB is overloaded)
Your threads now wait 30s for Payment Service instead of 100ms
Your thread pool fills up waiting on Payment Service
Your API is now "slow" — but YOUR code is fine
You are a victim of their problem
```

**How cascades propagate:**

```
Payment Service: 100ms → 30,000ms
        │
        ▼
Your threads: held for 30s instead of 100ms
→ Need 300x more threads to handle same RPS
→ Thread pool exhausts in seconds
→ Your API returns 503
        │
        ▼
Caller of your API:
→ Your API is now slow for them
→ THEIR thread pool exhausts
→ THEIR API returns 503
        │
        ▼
(cascade continues up the call chain)
```

**How to confirm:**

```
Trace: HTTP span to payment-service is 30,000ms (was 100ms)
Metric: payment_service.latency.p99 → 30,000ms (was 100ms)
Metric: payment_service.error_rate → spiking
Log:    "Read timeout after 30000ms calling payment-service"
```

**Fix: Circuit Breaker**

The circuit breaker stops you from holding threads waiting on a broken dependency.

```
State machine:

  CLOSED (normal operation)
    │  count failures over sliding window
    │  if failure_rate > threshold (e.g. 50%)
    ▼
  OPEN (fast-fail)
    │  all calls immediately return error (no network call)
    │  threads not held — pool stays available
    │  after wait_duration (e.g. 30s)
    ▼
  HALF-OPEN (probe)
    │  allow 1 test call through
    │  success → CLOSED
    │  failure → OPEN again
```

```java
// Resilience4j example
CircuitBreaker cb = CircuitBreaker.ofDefaults("payment-service");

String result = cb.executeSupplier(() ->
    paymentClient.validate(request)  // if CB OPEN → throws exception immediately
);
```

**Without circuit breaker:** Payment Service slow → your threads held → your pool exhausts → your API slow → callers affected.

**With circuit breaker:** Payment Service slow → CB trips OPEN → your API fast-fails with "payment unavailable" → threads not held → your API stays fast for other endpoints.

**Additional defenses:**

```
Timeout:              always set a timeout on outbound calls
                      default: no timeout = thread held indefinitely
                      
Retry with backoff:   retry on transient failures
                      exponential backoff + jitter to avoid thundering herd
                      BUT: don't retry on timeout — you'll amplify the problem
                      
Bulkhead:             separate thread pool per downstream dependency
                      payment service slowness can't exhaust the main pool
                      
Fallback:             if payment fails, return cached result / degrade gracefully
                      "payment service temporarily unavailable, try again in 30s"
```

---

### Root Cause 6: Tail Latency — p99 Slow, p50 Fine

**The mechanism:**

p99 slow but p50 fine means: 99% of requests are fast, 1% are very slow. This is often:

- **GC stop-the-world pauses** — 99% of requests finish between pauses, 1% unlucky enough to hit a pause
- **Lock contention** — 99% of requests don't hit a hot lock, 1% do and wait
- **Hot data / cold data** — 99% of requests hit cached data, 1% hit uncached cold rows triggering a full DB read
- **Noisy neighbor** — other processes on the same machine occasionally steal CPU

**How to confirm:**

```
Metric: p50 latency stable (20ms)
Metric: p99 latency spiking (2000ms)
→ This rules out "everything is slow" — only some requests are slow

Metric: GC pause duration → correlates with p99 spikes?
Metric: Lock wait time → correlates with p99?
Metric: Cache hit rate → dropped? → cold requests going to DB
```

**Fixes per cause:**

```
GC pauses:
  → Switch to low-pause GC: G1GC (Java 9+), ZGC (Java 15+), Shenandoah
  → Reduce allocation rate (object pooling, reduce temp object creation)
  → Increase heap size to reduce GC frequency
  -XX:+UseZGC -Xmx8g

Lock contention:
  → Identify hot lock: Java: jstack, async-profiler
  → Replace synchronized with ReadWriteLock (multiple readers, one writer)
  → Replace locks with lock-free data structures (ConcurrentHashMap, AtomicLong)
  → Shard the data to reduce contention

Cache cold start:
  → Pre-warm cache on startup
  → Implement cache stampede protection (probabilistic early expiration)
  → For new deploys: warm cache before shifting traffic
```

---

## Step 6: Fix, Verify, and Confirm

After identifying the root cause:

```
1. Make ONE change at a time
   (if you change three things, you don't know which one fixed it)

2. Measure after each change
   Compare p50/p95/p99 before and after
   Compare error rate before and after

3. Load test before promoting to production
   Reproduce the load that caused the problem
   Confirm it no longer reproduces

4. Confirm in production
   Monitor for 30 minutes after deploy
   Watch the same metrics that showed the problem
```

---

## Step 7: Prevent Recurrence

**Alerting:**

```
Alert on: p99 latency > 500ms for 3 consecutive minutes
Alert on: error rate > 1% for 2 consecutive minutes
Alert on: thread pool rejected count > 0
Alert on: DB connection pool wait time > 100ms
Alert on: heap usage > 80%
Alert on: circuit breaker state = OPEN
```

**Load testing:**

```
Run load tests in staging before every major deploy
Gradually ramp load until you find the breaking point
Know your system's capacity before production finds it
Tools: k6, Gatling, Locust, wrk
```

**Capacity planning:**

```
Know these numbers for your system:
  Max sustainable RPS before latency climbs
  Memory per request × max concurrent requests = minimum heap needed
  DB connections needed at peak load
  Thread pool size needed at peak load

Review these after every major feature that changes the request path
```

---

## The Full Decision Tree

```
API is slow under load
        │
        ▼
Step 0: Are users impacted NOW?
        YES → restore service first (scale out / roll back)
        NO  → investigate safely
        │
        ▼
Step 1: Is it ALL endpoints or ONE endpoint?
        ONE  → endpoint-specific bug (bad query, missing index, N+1)
        ALL  → shared resource exhausted (thread pool, DB pool, memory)
        │
        ▼
Step 2: Build the timeline from metrics
        When did latency start climbing?
        What else changed at that time? (deploy, traffic spike, config)
        │
        ├── Correlates with deploy → code regression
        ├── Correlates with traffic spike → capacity issue
        └── Gradual climb over hours → leak (memory, connection)
        │
        ▼
Step 3: Pull a trace from a slow request
        Which span is the slow one?
        │
        ├── DB span slow → go to Step 5A (DB investigation)
        ├── HTTP span slow → go to Step 5B (downstream / circuit breaker)
        ├── Gap before first span → thread pool waiting
        └── Everything slow uniformly → GC or CPU starvation
        │
        ▼
Step 4: Read logs around the start time
        Confirm the error message matches the hypothesis
        │
        ▼
Step 5: Deep dive on the confirmed suspect
        Thread pool → async I/O or reduce I/O latency
        DB pool → fix connection leak, add PgBouncer
        Slow query → EXPLAIN ANALYZE → add index
        Memory → heap dump → fix leak
        Downstream → circuit breaker + timeout + fallback
        GC → low-pause collector, reduce allocation
        │
        ▼
Step 6: Fix one thing, measure, confirm
        │
        ▼
Step 7: Add alerts, run load tests, update capacity plan
```

---

## Key Principles (What Separates Principal from Mid-Level)

| Mid-Level Approach | Principal Approach |
|---|---|
| "Let me add logging and see what happens" | "Let me pull the trace for a failed request first" |
| Looks at average latency | Looks at p99 latency — average hides the problem |
| Increases thread pool size as the fix | Understands thread pool increase is a band-aid; fixes I/O |
| Restarts the service | Asks "why did it need restarting?" |
| Adds an index based on intuition | Runs EXPLAIN ANALYZE first, then adds the index the plan tells you to |
| Fixes the symptom | Finds the cause and fixes that |
| Works alone silently | Communicates status to stakeholders during an incident |
| Fixes this incident | Adds alerts and load tests so the next one is caught earlier |

---

## Glossary

| Term | Definition |
|---|---|
| **p50 / p95 / p99** | Percentile latency — p99 means 99% of requests were faster than this value. Always use percentiles, never averages. |
| **Golden Signals** | The four fundamental metrics for any service: latency, traffic, errors, saturation |
| **Thread Pool** | A fixed set of worker threads that handle incoming requests; saturation causes requests to queue then fail |
| **Connection Pool** | Pre-allocated set of DB connections reused across requests; exhaustion causes requests to wait then timeout |
| **EXPLAIN ANALYZE** | PostgreSQL command that executes a query and shows the actual plan, row counts, and timing per step |
| **Seq Scan** | Full table scan in PostgreSQL; appears when no index is available; becomes catastrophically slow at scale |
| **N+1 Query** | Anti-pattern: one query per item in a loop instead of one batch query; catastrophic under load |
| **Circuit Breaker** | Pattern that fast-fails calls to a degraded dependency; has three states: CLOSED, OPEN, HALF-OPEN |
| **Cascading Failure** | A downstream dependency's slowness propagates upward, causing independent services to fail |
| **Bulkhead** | Pattern that isolates thread pools per dependency, preventing one slow dependency from exhausting all threads |
| **GC Pause** | Stop-the-world garbage collection event; all application threads paused; causes p99 latency spikes |
| **Heap Dump** | Snapshot of JVM heap; analyzed to identify leaked objects and growing collections |
| **Tail Latency** | High latency at high percentiles (p99, p99.9) while low percentiles remain fast; indicates intermittent issues |
| **Distributed Trace** | End-to-end record of one request through all services and components with per-span timing |
| **Lock Contention** | Multiple threads competing for the same lock; causes some threads to wait while one proceeds |
| **PgBouncer** | Connection pooler for PostgreSQL; multiplexes many application connections into fewer real DB connections |
| **Thundering Herd** | Many clients retry simultaneously after a failure, causing a traffic spike that re-triggers the failure |
| **Backpressure** | Signaling to upstream systems to slow down when a downstream component is overwhelmed |
| **Idempotency** | Property where repeating an operation produces the same result; important for safe retries |
| **ILM (Index Lifecycle Management)** | Elasticsearch policy that automatically moves indices through hot → warm → cold → delete phases |
| **Reactive / Async I/O** | Non-blocking I/O model where threads are not held while waiting for network or disk; enables higher concurrency with fewer threads |
