# System Design: Distributed Log Collection + API Debugging
> Senior / Principal Engineer Level — Three interview questions unified under one observability framework.
> Modeled on the Ticket Master pattern: Requirements → Entities → API → HLD → Deep Dives.

---

## The Three Questions This Covers

| Question | Type | What They're Testing |
|---|---|---|
| **Design a tool that collects logs from all servers in a region** | System Design | Distributed systems, data pipelines, scale |
| **An instance suddenly went missing — how do you figure out the issue?** | Debugging / Investigation | Operational maturity, observability instincts |
| **After 200 API calls you start getting 5xx/4xx errors — how do you debug this?** | Debugging under load | Systematic thinking, knowing what to look at first |

> **Principal-level framing:** These are not three separate questions. They are all expressions of the same underlying problem — **you cannot operate a distributed system you cannot observe**. A log collection system is the infrastructure that makes the debugging questions answerable. Walk into any of these questions with that framing and you immediately signal seniority.

---

## Interview Roadmap (Same Pattern as Ticket Master)

| Phase | Time | Goal |
|---|---|---|
| Requirements | 5–8 min | Functional + non-functional; scope the problem |
| Core Entities | 2–3 min | What data flows through the system |
| APIs | 3–5 min | Ingestion, query, alerting interfaces |
| High-Level Design | 15 min | Simple pipeline satisfying all FRs |
| Deep Dives | 15 min | Scale, fault tolerance, missing instance, API debugging |

---

## Part 1: Design a Distributed Log Collection System

---

### 1. Requirements

#### Functional Requirements

| # | Requirement |
|---|---|
| 1 | The system should **collect logs from all servers** in a region in real time |
| 2 | Logs should be **queryable** — filter by server, service, time range, severity, keyword |
| 3 | The system should **alert** when error rates cross a threshold |
| 4 | Logs should be **durable** — not lost if a collection agent crashes |
| 5 | The system should **detect missing/dead instances** and alert on them |

**Out of scope:** Multi-region replication, log-based billing, RBAC per log stream, compliance redaction (acknowledge, don't design).

> **Clarify early:** Are we collecting structured logs (JSON) or unstructured (raw text)? Both? This affects parsing complexity. Assume both — structured preferred, unstructured handled.

---

#### Non-Functional Requirements

| Requirement | Why It Matters Here |
|---|---|
| **High availability for ingestion** | If the log pipeline goes down during an incident, you're blind exactly when you need visibility most. Ingestion must never be the single point of failure. |
| **Eventual consistency for queries** | A log appearing 2–5 seconds late in search is acceptable. Logs are append-only; strict ordering within a query window matters more than global ordering. |
| **Durability — no log loss** | Logs are the audit trail. Losing them during a crash is unacceptable. At-least-once delivery is required. |
| **Low ingestion latency** | Alert pipelines depend on logs arriving quickly. Target: logs visible in query within 5–10 seconds of emission. |
| **High write throughput** | A fleet of 10,000 servers each emitting 1,000 log lines/second = 10M lines/second. The pipeline must handle this without backpressure. |
| **Cost-efficient storage** | Logs are high volume, mostly cold after 24 hours. Hot/warm/cold tiering is essential — not everything lives in expensive fast storage. |
| **Scalability for fleet size changes** | Auto-scaling events mean instances appear and disappear. The collection system must handle dynamic fleet topology without manual reconfiguration. |

> **Senior/Principal signal:** Consistency split — same pattern as Ticket Master. Ingestion path needs durability (strong guarantee); query path is fine with eventual consistency. Frame it this way explicitly.

---

### 2. Core Entities

```
┌──────────────────────────────────┐
│           LogEntry                │  ← atomic unit of data
│  id (UUID)                        │
│  server_id → Server.id (FK)       │
│  service_name                     │
│  severity (INFO/WARN/ERROR/FATAL) │
│  message (TEXT)                   │
│  structured_data (JSONB)          │  ← parsed fields if structured
│  timestamp                        │
│  ingested_at                      │
│  region                           │
└──────────────────────────────────┘

┌──────────────────────────────────┐
│             Server                │  ← emitting instance
│  id (UUID)                        │
│  hostname                         │
│  ip_address                       │
│  region                           │
│  tags (string[])                  │  ← e.g. {api-server, prod, us-east-1}
│  status (alive / missing / dead)  │
│  last_seen_at                     │  ← updated on every log received
│  registered_at                    │
└──────────────────────────────────┘

┌──────────────────────────────────┐
│          AlertRule                │
│  id (UUID)                        │
│  name                             │
│  condition (JSONB)                │  ← e.g. error_rate > 5% over 1 min
│  severity                         │
│  notification_channel             │  ← PagerDuty, Slack, email
│  enabled (bool)                   │
└──────────────────────────────────┘

┌──────────────────────────────────┐
│          AlertEvent               │
│  id (UUID)                        │
│  rule_id → AlertRule.id (FK)      │
│  server_id → Server.id (FK)       │
│  triggered_at                     │
│  resolved_at (nullable)           │
│  message                          │
└──────────────────────────────────┘
```

### Relationship Summary

| From | Field | Points To | Cardinality | Notes |
|---|---|---|---|---|
| `LogEntry` | `server_id` | `Server.id` | N:1 | Many log lines from one server |
| `AlertEvent` | `rule_id` | `AlertRule.id` | N:1 | Many firings of one rule |
| `AlertEvent` | `server_id` | `Server.id` | N:1 | Which server triggered the alert |

---

### 3. APIs

#### Log Ingestion (Agent → Collector) — Internal
```
POST /internal/ingest
Body: {
  server_id: string,
  service_name: string,
  logs: [
    {
      severity: "ERROR" | "WARN" | "INFO" | "DEBUG" | "FATAL",
      message: string,
      structured_data: object | null,
      timestamp: ISO8601
    }
  ]
}

Response: { accepted: number, dropped: number }
```
> Batch ingestion — agents buffer locally and flush every 1–5 seconds. Never one log line per HTTP call.

#### Query Logs
```
GET /logs/search
Query params:
  server_id?: string
  service_name?: string
  severity?: string[]          // ["ERROR", "FATAL"]
  keyword?: string             // full-text search
  from: ISO8601
  to: ISO8601
  limit?: number               // default 100, max 1000
  cursor?: string              // pagination

Response: {
  logs: LogEntry[],
  next_cursor: string | null,
  total_matched: number
}
```

#### Get Server Status
```
GET /servers/{serverId}

Response: {
  server: Server,
  last_seen_at: ISO8601,
  status: "alive" | "missing" | "dead",
  log_rate_per_second: number
}
```

#### List All Servers in Region
```
GET /servers?region=us-east-1&status=missing

Response: Server[]
```
> This is the endpoint that answers "which instances are missing?" — key for the dead instance investigation.

#### Alert Rules
```
POST /alerts/rules          — create a rule
GET  /alerts/rules          — list rules
GET  /alerts/events         — list triggered alerts (filterable by server, rule, time)
```

---

### 4. High-Level Design

#### Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                    SERVER FLEET (per region)                       │
│                                                                    │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐            │
│  │   Server A   │  │   Server B   │  │   Server C   │  ...       │
│  │ Log Agent    │  │ Log Agent    │  │ Log Agent    │            │
│  │ (sidecar)    │  │ (sidecar)    │  │ (sidecar)    │            │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘            │
└─────────┼────────────────┼────────────────┼───────────────────────┘
          │                │                │
          │  batch POST /internal/ingest every 1-5s
          ▼                ▼                ▼
┌──────────────────────────────────────────────────────────────────┐
│               Collector Service (horizontally scaled)             │
│  - Validates + parses log batches                                 │
│  - Updates Server.last_seen_at                                    │
│  - Publishes to Kafka                                             │
└──────────────────────────────────┬───────────────────────────────┘
                                   │
                                   ▼
                    ┌──────────────────────────┐
                    │   Kafka (Log Stream)      │
                    │   topic: logs.{region}    │
                    │   topic: servers.heartbeat│
                    └────────────┬─────────────┘
                                 │
              ┌──────────────────┼──────────────────┐
              │                  │                  │
              ▼                  ▼                  ▼
   ┌──────────────────┐ ┌──────────────┐  ┌──────────────────┐
   │  Indexer Service │ │ Alert Engine │  │ Instance Monitor │
   │                  │ │              │  │                  │
   │  writes to       │ │ evaluates    │  │ detects missing  │
   │  Elasticsearch   │ │ alert rules  │  │ servers          │
   └────────┬─────────┘ └──────┬───────┘  └──────┬───────────┘
            │                  │                  │
            ▼                  ▼                  ▼
   ┌──────────────┐   ┌──────────────┐   ┌──────────────────┐
   │Elasticsearch │   │  PagerDuty   │   │   PostgreSQL      │
   │(hot storage) │   │  Slack etc.  │   │  (server registry)│
   └──────┬───────┘   └──────────────┘   └──────────────────┘
          │
          │  after 7 days
          ▼
   ┌──────────────┐
   │  S3 (cold    │
   │   storage)   │
   └──────────────┘

┌──────────────────────────────────────────────────────────────────┐
│                      Query Service                                 │
│  serves GET /logs/search → reads from Elasticsearch               │
│  serves GET /servers → reads from PostgreSQL                      │
└──────────────────────────────────────────────────────────────────┘
```

---

#### Walking Through Each Functional Requirement

**FR1: Collect logs from all servers**

Every server runs a **Log Agent** as a sidecar process. The agent:
- Tails log files (e.g. `/var/log/app/*.log`) or reads from stdout/stderr
- Buffers log lines in memory (with a local disk fallback — see fault tolerance)
- Flushes a batch every 1–5 seconds to the **Collector Service** via `POST /internal/ingest`
- On each flush, updates `Server.last_seen_at` in the Collector (heartbeat piggybacks on log delivery)

The Collector validates, parses (JSON detection), and **publishes to Kafka** topic `logs.{region}`. It never writes directly to the storage layer — Kafka is the durable buffer.

**FR2: Queryable logs**

The **Indexer Service** consumes from Kafka and writes to **Elasticsearch**. Elasticsearch builds an inverted index on `message`, `service_name`, `severity`, and parsed structured fields — enabling fast full-text search and filtered queries.

The **Query Service** translates `GET /logs/search` parameters into Elasticsearch DSL queries and returns paginated results.

**FR3: Alerting**

The **Alert Engine** consumes from the same Kafka topic (different consumer group — fan-out). It evaluates alert rules in a sliding window: "more than 5% of log lines from service X in the last 60 seconds have severity ERROR." When a rule fires, it writes an `AlertEvent` and calls the notification channel (PagerDuty, Slack webhook, etc.).

**FR4: Durability**

Three layers:
1. **Agent local buffer** — if the Collector is unreachable, the agent buffers to disk. Flushes when connection resumes.
2. **Kafka retention** — messages are retained on disk for 24–48 hours. If the Indexer crashes and restarts, it replays from the last committed offset. No logs lost.
3. **S3 archival** — after 7 days in Elasticsearch (hot), logs are exported to S3 (cold). Cheap, queryable with Athena.

**FR5: Detect missing instances**

The **Instance Monitor** service:
- Queries PostgreSQL for servers where `last_seen_at < NOW() - threshold` (e.g. 60 seconds)
- Marks them `status = missing`
- Fires an alert via the Alert Engine
- If still missing after 5 minutes → marks `status = dead`

The `last_seen_at` is updated on every log batch received by the Collector — log delivery is the heartbeat. No separate heartbeat endpoint needed.

---

### 5. Deep Dives

#### Deep Dive 1: Scale — 10M Log Lines Per Second

**Problem:** 10,000 servers × 1,000 lines/second = 10M lines/second. That's ~1–5 GB/second of raw data depending on line size.

**Collector scaling:**
- Collector is stateless — horizontally scale behind a load balancer
- Each Collector instance handles batches from N agents
- Agents are configured with multiple Collector endpoints and round-robin

**Kafka scaling:**
- Partition the `logs.{region}` topic by `server_id` or `service_name`
- More partitions = more parallelism for consumers (Indexer, Alert Engine)
- Kafka can handle millions of messages/second with proper partition count

**Elasticsearch scaling:**
- Shard by time (daily indices: `logs-2026-08-26`)
- Hot nodes for recent data (fast SSDs); warm nodes for older data (cheaper HDDs)
- Index rollover: when an index hits a size threshold, roll to a new one
- Curator or ILM (Index Lifecycle Management) handles hot → warm → cold → delete transitions automatically

**Agent backpressure:**
- If Kafka is slow or the Collector is overloaded, agents accumulate in their local buffer
- Buffer has a max size (e.g. 100MB on disk). If exceeded, oldest logs are dropped (with a counter metric on the drop rate)
- This is a deliberate trade-off — prefer dropping old verbose logs over crashing the agent

---

#### Deep Dive 2: Fault Tolerance — Agent Crashes, Collector Crashes, Kafka Goes Down

| Failure | Impact | Recovery |
|---|---|---|
| **Agent crashes** | Logs from that server stop flowing | Agent restarts automatically (systemd/k8s); local buffer survives on disk; resumes from last flush position |
| **Collector instance crashes** | Agents retry with exponential backoff; other Collector instances absorb traffic | Stateless — new instance spins up immediately |
| **Kafka partition leader fails** | Brief pause (~15–30s) while Kafka elects new leader | Kafka replication (RF=3) makes this transparent to producers after leader election |
| **Indexer crashes** | Log ingestion continues; Kafka retains messages | Indexer restarts, replays from committed offset — no data loss, just indexing lag |
| **Elasticsearch goes down** | Querying fails; ingestion continues into Kafka | Kafka acts as buffer; Indexer resumes when ES recovers; alerts still work via Alert Engine consuming Kafka directly |

> **Key insight:** Kafka is the durability layer, not Elasticsearch. Elasticsearch is the query layer. If ES goes down, you lose query ability temporarily but zero logs. This is the right trade-off.

---

#### Deep Dive 3: Storage Tiering and Cost

```
Age          Storage         Cost        Query Speed
─────────────────────────────────────────────────────
0–7 days     Elasticsearch   $$$         Sub-second
7–30 days    Elasticsearch   $$          Sub-second (warm nodes)
30–90 days   S3 + Parquet    $           Minutes (Athena)
90+ days     S3 Glacier      cents       Hours (restore first)
```

**Implementation:**
- Elasticsearch ILM policy: hot (0–7d) → warm (7–30d) → delete from ES
- Before deletion, the Archiver Service exports to S3 as gzipped Parquet files partitioned by `region/date/service_name`
- AWS Athena or Spark can query S3 logs when historical analysis is needed
- This keeps Elasticsearch lean and fast for the recent hot window that matters for ops

---

## Part 2: If an Instance Suddenly Went Missing — Investigation Playbook

> This is an operational debugging question. The interviewer wants to see a **systematic, layered approach** — not guessing. Same instinct as the API debugging question: metrics → traces → logs, never the reverse.

---

### What "Missing" Means (Clarify First)

Before investigating, nail down what "missing" actually means — the answer changes the investigation:

| Signal | What It Means |
|---|---|
| Instance no longer sending logs | Agent crashed, network partition, or instance terminated |
| Instance removed from load balancer | Health check failing, graceful shutdown, or autoscaler terminated it |
| Instance not responding to SSH | OS-level crash, kernel panic, hardware failure, or security group changed |
| Instance missing from cloud console | Terminated by autoscaler, spot instance reclaimed, or accidental deletion |

---

### Investigation Playbook (Systematic, Layer by Layer)

```
START: Instance X is missing
        │
        ▼
Step 1: CHECK THE CLOUD CONSOLE FIRST
        │
        ├── Is the instance still listed?
        │     No → Was it terminated? Check autoscaling activity log
        │           Was it a spot instance? Check spot interruption notice
        │           Was it manually terminated? Check CloudTrail
        │
        ├── Is it listed but in a bad state?
        │     "stopped" → check stop reason
        │     "terminated" → check termination reason + who/what did it
        │
        ▼
Step 2: CHECK THE LOG COLLECTION SYSTEM
        │
        ├── When did last_seen_at stop updating?
        │     (query: SELECT last_seen_at FROM servers WHERE id = :id)
        │
        ├── What were the last log lines before it disappeared?
        │     (query: GET /logs/search?server_id=X&severity=ERROR&limit=50)
        │     Look for: OOM killed, segfault, unhandled exception, disk full
        │
        ├── Was there a spike in ERROR/FATAL logs just before?
        │     Signals: application crash, dependency failure
        │
        ▼
Step 3: CHECK INFRASTRUCTURE METRICS (CloudWatch / Prometheus)
        │
        ├── CPU — was it pegged at 100%? → runaway process, fork bomb
        ├── Memory — did it hit the limit? → OOM killer terminated a process
        ├── Disk — was it full? → writes failing, agent couldn't buffer logs
        ├── Network — did traffic drop to zero? → NIC failure, security group change
        ├── Status checks — did AWS EC2 status checks fail? → hardware issue
        │
        ▼
Step 4: CHECK THE LOAD BALANCER
        │
        ├── Was this instance deregistered from the target group?
        ├── Was health check failing? What endpoint? What response code?
        ├── When was the last successful health check?
        │
        ▼
Step 5: CHECK AUTOSCALING ACTIVITY
        │
        ├── Did the autoscaler terminate it intentionally?
        │     (scale-in event, scheduled action, reactive scale-down)
        ├── Did it terminate due to a failed health check?
        ├── Was it replaced by a new instance?
        │
        ▼
Step 6: CHECK CLOUDTRAIL / AUDIT LOGS
        │
        ├── Who or what called TerminateInstances?
        ├── Was there an IAM role or automation that triggered it?
        ├── Was there a deployment pipeline that cycled instances?
        │
        ▼
CONCLUSION: Root cause falls into one of:
  A. Intentional termination (autoscaler, deployment, manual)
  B. Application crash (OOM, unhandled exception, disk full)
  C. Infrastructure failure (hardware, network, hypervisor)
  D. External action (security group change, spot reclaim, accidental delete)
```

---

### What the Log Collection System Enables Here

This is the payoff of the design in Part 1. Without it:
- You cannot answer "what were the last log lines?"
- You cannot answer "when did it go missing?"
- You cannot answer "was there an error spike before it died?"

With it, steps 2 and 3 of the playbook above are answered in seconds via `GET /logs/search` and `GET /servers/{id}`.

**This is the answer to why the interviewer asks both questions in the same session.** The log collection system IS the tool that makes instance investigation possible.

---

## Part 3: After 200 API Calls You Get 5xx/4xx Errors — Debug It

> This is a **stateful failure** — the system works fine initially then degrades. That pattern is a strong signal. A senior engineer immediately asks: "what changed between call 1 and call 200?" The answer is almost always resource exhaustion.

---

### The Pattern Recognition First

The fact that it fails *after* 200 calls and not immediately tells you a lot:

```
Fails immediately → config wrong, dependency unreachable, code bug
Fails after N calls → resource exhaustion, leak, rate limit, connection pool
Fails after N *time* → TTL expiry, token expiry, cache eviction
Fails under load only → thread pool saturation, DB connection pool, cascade
```

"After 200 API calls" → **resource exhaustion** is the most likely class of problem. Your investigation is looking for what runs out.

---

### Systematic Debug Playbook (Observability First, Always)

```
NEVER start by looking at code.
ALWAYS start by looking at signals.
```

#### Step 1: Classify the Error (4xx vs 5xx)

This is the first fork in the road:

| Error Class | What It Means | Where to Look First |
|---|---|---|
| **4xx (client errors)** | The server is up; the request is wrong | Rate limiting, auth token expiry, malformed request after state change |
| **5xx (server errors)** | The server is failing | Thread pool, DB, downstream dependency, OOM |
| **Mix of both** | Different failure modes for different request types | Investigate separately by endpoint |

```
→ If 429 Too Many Requests: you're being rate limited
→ If 401/403: token expired or auth state changed after N calls
→ If 500: server-side failure — look at server metrics + logs
→ If 502/503/504: upstream timeout or gateway failure — look at downstream deps
```

---

#### Step 2: Metrics First — Establish the Baseline

Before touching code or logs, pull these metrics at the moment errors started:

```
┌─────────────────────────────────────────────────────────┐
│  WHAT TO CHECK                    WHAT YOU'RE LOOKING FOR│
├─────────────────────────────────────────────────────────┤
│  Request rate (RPS)               Did traffic spike?     │
│  Error rate (%)                   When exactly did it    │
│                                   cross the threshold?   │
│  Latency p50 / p95 / p99          Did latency climb      │
│                                   before errors started? │
│  CPU utilization                  Runaway process?       │
│  Memory utilization               Growing leak?          │
│  JVM heap (if Java)               GC pressure?           │
│  Thread pool active/queued        Saturation?            │
│  DB connection pool active/idle   Pool exhausted?        │
│  DB query latency                 Slow queries?          │
│  Downstream service latency       Dependency degraded?   │
└─────────────────────────────────────────────────────────┘
```

> **What you're building:** a timeline. "At call 150, latency started climbing. At call 180, thread pool hit 100%. At call 200, requests started failing." That timeline tells you causality.

---

#### Step 3: Traces — Find the Slow Path

With distributed tracing (OpenTelemetry / Jaeger / X-Ray), pull a trace from a failed request and compare it to a successful one:

```
Successful request trace (call #10):
  API handler          2ms
  DB query             5ms
  Cache read           1ms
  Total:               8ms

Failed request trace (call #210):
  API handler          2ms
  DB query             [waiting for connection... 29,998ms]
  Total:               30,000ms → timeout → 500
```

The trace tells you **exactly where time is spent**. You don't need to guess which component is the problem — the trace shows you.

---

#### Step 4: Logs — Confirm the Root Cause

Now look at logs around the time errors started. With the log collection system from Part 1:

```
GET /logs/search?service_name=api&severity=ERROR&from=T-5min&to=T+1min
```

Common patterns you're looking for:

```
"too many connections" → DB connection pool exhausted
"connection refused" → downstream service down
"timeout waiting for connection from pool" → thread pool or DB pool full
"OutOfMemoryError" → heap exhausted
"Circuit breaker OPEN" → downstream dependency tripped the breaker
"rate limit exceeded" → external API rate limit hit
"Token expired" → auth token not being refreshed
```

---

#### Step 5: The Six Most Common Root Causes (and Fixes)

---

##### Root Cause 1: DB Connection Pool Exhausted

**What happens:**
- App starts fine. After N requests, all DB connections are checked out.
- New requests wait for a connection. Wait exceeds timeout. → 500.

**How to confirm:**
```sql
-- PostgreSQL: see active connections
SELECT count(*), state FROM pg_stat_activity GROUP BY state;
-- If max_connections hit → pool exhausted
```
```
Metric: db.pool.active == db.pool.max → confirmed
```

**Fix:**
```
Short term: increase pool size (but this is a band-aid)
Real fix:   find the connection leak — transactions not closed,
            connections not returned to pool on exception paths
            Use try-with-resources / connection pool timeouts
```

---

##### Root Cause 2: Thread Pool Saturation

**What happens:**
- Each request occupies a thread while waiting for slow I/O (DB, HTTP call)
- After N requests, all threads are occupied waiting
- New requests queue up. Queue fills. → 503 / timeout

**How to confirm:**
```
Metric: thread_pool.active == thread_pool.max
Metric: thread_pool.queue_size growing
Log: "Request rejected from queue" or "Thread pool exhausted"
```

**Fix:**
```
Short term: increase thread pool size (again, band-aid)
Real fix:   switch blocking I/O to async/reactive (WebFlux, async handlers)
            OR reduce I/O latency (fix the slow downstream call)
            Use circuit breakers so slow dependencies don't hold threads
```

---

##### Root Cause 3: Memory Leak → OOM → Crash → 5xx

**What happens:**
- Memory grows slowly with each request (leaked objects, growing caches)
- After N requests, heap exhausted → GC thrash → OOM → process dies → 502/503

**How to confirm:**
```
Metric: jvm.memory.heap.used growing monotonically (never coming down)
Metric: jvm.gc.pause increasing (GC working harder and harder)
Log: "java.lang.OutOfMemoryError: Java heap space"
```

**Fix:**
```
Take a heap dump: jmap -dump:format=b,file=heap.hprof <pid>
Analyze with: Eclipse MAT, JProfiler, VisualVM
Look for: objects accumulating in static collections,
          unclosed streams, ThreadLocal values not cleaned up
```

---

##### Root Cause 4: Cascading Failure from Downstream Dependency

**What happens:**
- A downstream service (DB, external API, cache) becomes slow
- Your service waits → threads held → pool exhausts → your service fails too
- You get 5xx even though YOUR code is fine

**How to confirm:**
```
Trace: downstream call latency spiked at the same time errors started
Metric: downstream_service.latency p99 climbing
Log: "timeout calling payment-service" / "connection refused to cache"
```

**Fix:**
```
Immediate: circuit breaker trips → fast-fail instead of waiting
           (Resilience4j, Hystrix, or manual implementation)
           Timeout + retry with exponential backoff on the call
Real fix:  fix the downstream service
           Add fallback behavior (cached response, degraded mode)
```

Circuit breaker state machine:
```
CLOSED (normal)
  → too many failures →
OPEN (fast-fail all requests, don't call downstream)
  → after timeout →
HALF-OPEN (allow one test request)
  → success → CLOSED
  → failure → OPEN again
```

---

##### Root Cause 5: Rate Limiting (429) — External or Internal

**What happens:**
- External API (Stripe, Twilio, etc.) has a rate limit of e.g. 200 calls/minute
- Your app hits it → 429 from external → your app returns 500 to caller

**How to confirm:**
```
Log: "429 Too Many Requests from stripe.com"
Metric: external_api.response_code{code="429"} non-zero
```

**Fix:**
```
Implement token bucket / leaky bucket rate limiter on your outbound calls
Queue + batch requests to stay under the limit
Cache responses where possible (don't call stripe for the same data twice)
```

---

##### Root Cause 6: Auth Token Expiry

**What happens:**
- App starts with a valid token (OAuth, API key, JWT)
- Token expires after N minutes / N calls
- App doesn't refresh → subsequent calls get 401/403

**How to confirm:**
```
Error pattern: first N calls succeed, then all fail with 401
Log: "Token expired" or "401 Unauthorized"
Time since startup: matches token TTL
```

**Fix:**
```
Implement token refresh before expiry (refresh at 80% of TTL)
Use a credentials provider that auto-refreshes (AWS SDK does this natively)
Never hardcode tokens in config — use a secrets manager with rotation
```

---

#### Step 6: EXPLAIN ANALYZE — Database Query Investigation

If the trace pointed at the DB as the slow component:

```sql
EXPLAIN ANALYZE
SELECT * FROM orders
WHERE user_id = 12345
AND status = 'pending'
ORDER BY created_at DESC;
```

**What to look for in the output:**

```
Seq Scan on orders (rows=1000000)   ← FULL TABLE SCAN — needs an index
Index Scan using idx_user_id        ← good, using index
Nested Loop (rows=50000)            ← might be N+1 query problem
Hash Join (rows=1000)               ← usually fine
Sort (rows=500000)                  ← expensive if no index on sort column
```

**Most common fixes:**
```
Missing index          → CREATE INDEX CONCURRENTLY idx_orders_user_status
                         ON orders(user_id, status);
N+1 query             → Batch with IN clause or JOIN instead of loop
Missing LIMIT          → Never SELECT * without LIMIT on user-facing queries
Lock contention        → Check pg_locks; long transactions blocking others
```

---

### The Full Debug Decision Tree

```
5xx/4xx errors start after N calls
        │
        ▼
Is it 4xx or 5xx?
        │
        ├── 429 → Rate limited → implement outbound rate limiter
        ├── 401/403 → Token expired → implement token refresh
        ├── 400 → Bad request after state change → check input validation
        │
        └── 5xx → Server failure
                    │
                    ▼
            Did latency climb before errors?
                    │
                    ├── Yes → Resource exhaustion
                    │           │
                    │           ├── Thread pool full? → async I/O or reduce I/O latency
                    │           ├── DB pool full?     → fix connection leak
                    │           ├── Memory growing?   → heap dump → fix leak
                    │           └── Downstream slow?  → circuit breaker + fix dep
                    │
                    └── No → Sudden failure
                                │
                                ├── Instance crashed?  → check logs, OOM, segfault
                                ├── Deployment rolled? → check deploy timeline
                                └── Config changed?    → check audit trail
```

---

## How to Present This in a System Design Interview

When the interviewer asks ANY of these three questions, use this opening:

> *"Before I jump into the design / before I start debugging — I want to establish what observability we have, because everything else flows from that. The three things I always want are: metrics (what's happening in aggregate), distributed traces (what's happening per request), and logs (what did the system say). Let me design the system that provides those first, then use that foundation to answer the specific question."*

That framing alone signals principal-level thinking. You're not reacting to symptoms — you're establishing the foundation that makes the symptoms interpretable.

---

## Key Design Decisions Summary

| Decision | Choice | Rationale |
|---|---|---|
| Log delivery model | Agent batches every 1–5s | Balances latency vs. overhead; one HTTP call per N lines not per line |
| Durability layer | Kafka (not Elasticsearch) | ES is queryable but not durable; Kafka absorbs crashes and replays |
| Heartbeat mechanism | Piggybacked on log delivery | No separate heartbeat endpoint; `last_seen_at` updated per batch |
| Query layer | Elasticsearch | Inverted index for full-text search; geospatial + time-range native |
| Cold storage | S3 + Parquet | Dirt cheap; queryable with Athena; Glacier for long-term archive |
| Alert engine | Separate consumer group on Kafka | Decoupled from indexing; can alert even if ES is down |
| Instance detection | `last_seen_at` threshold query | Simple, reliable, no separate ping mechanism needed |
| API error debugging order | Metrics → Traces → Logs | Never guess; establish the timeline first |
| DB debugging | EXPLAIN ANALYZE | Ground truth for query plan; always run before adding indexes |
| Cascading failure defense | Circuit breaker | Fast-fail prevents thread pool exhaustion from slow dependencies |

---

## Interview Tips

| Tip | Detail |
|---|---|
| **Observability first, always** | For any debugging question, start with "what signals do we have?" before proposing a fix |
| **The error pattern tells you the class** | Fails immediately = config/code. Fails after N calls = exhaustion. Fails under load only = concurrency. |
| **4xx vs 5xx is the first fork** | 4xx means the server is healthy but the request is wrong. 5xx means the server is failing. Different investigations. |
| **Kafka is not a queue** | Clarify: Kafka is an event log with retention and replay. SQS is a queue. Use Kafka where fan-out or replay matters; SQS where competing consumers is enough. |
| **Heartbeat = log delivery** | Don't design a separate heartbeat mechanism. Piggyback on log delivery — if we're getting logs, the instance is alive. |
| **Circuit breaker is the cascade answer** | Whenever asked "what if a downstream is slow?" — circuit breaker is the answer. Know the three states: CLOSED, OPEN, HALF-OPEN. |
| **EXPLAIN ANALYZE before any index** | Never add an index by instinct. Run EXPLAIN ANALYZE, read the plan, confirm the index helps. |
| **Math with purpose** | 10,000 servers × 1,000 lines/sec × 200 bytes/line = 2 GB/sec. Do this math only to justify Kafka partitioning and multi-shard Elasticsearch — not as a warmup exercise. |
| **Mention ILM (Index Lifecycle Management)** | Shows you've operated Elasticsearch in production. Hot → warm → cold → delete, automated. |

---

## Glossary

| Term | Definition |
|---|---|
| **Alert Engine** | Service that evaluates alert rules against the log stream and fires notifications when thresholds are crossed |
| **At-Least-Once Delivery** | Messaging guarantee: a message is delivered one or more times; idempotent consumers required |
| **Circuit Breaker** | Pattern that fast-fails calls to a degraded dependency, preventing thread pool exhaustion from cascading |
| **CloudTrail** | AWS audit log of all API calls made in an account; used to determine who terminated an instance |
| **Cold Storage** | Cheap, slow storage tier (S3, Glacier) for logs older than 30–90 days |
| **Connection Pool** | Pre-allocated set of reusable DB connections; exhaustion causes requests to wait and eventually time out |
| **Consumer Group (Kafka)** | A named group of consumers sharing a topic; each partition goes to one member; multiple groups = fan-out |
| **EXPLAIN ANALYZE** | PostgreSQL command that executes a query and shows the actual query plan and timing per step |
| **Elasticsearch** | Search-optimized store with inverted indexes; used here for full-text log search and filtering |
| **Fan-out** | One message being consumed by multiple independent consumer groups (e.g. Indexer + Alert Engine both read logs) |
| **Heap Dump** | Snapshot of JVM heap memory; analyzed to find memory leaks by identifying accumulating object types |
| **Hot Storage** | Fast, expensive storage tier (Elasticsearch on SSD) for recent logs actively queried by operators |
| **ILM (Index Lifecycle Management)** | Elasticsearch feature that automatically transitions indices through hot → warm → cold → delete phases |
| **Instance Monitor** | Background service that detects missing servers by querying `last_seen_at` against a staleness threshold |
| **Inverted Index** | Data structure mapping terms to documents containing them; foundation of Elasticsearch full-text search |
| **Kafka** | Distributed event log; durable, replayable, supports multiple consumer groups; used here as the ingestion buffer |
| **last_seen_at** | Timestamp updated on every log batch received; the heartbeat signal for instance liveness detection |
| **Log Agent** | Sidecar process on each server that tails log files, buffers, and ships batches to the Collector |
| **Memory Leak** | Objects accumulating in heap that are never garbage collected; leads to OOM over time |
| **N+1 Query** | Anti-pattern where code issues one query per item in a list instead of one batched query |
| **OOM (Out of Memory)** | Condition where a process exceeds its memory limit; JVM throws OutOfMemoryError; Linux OOM killer terminates the process |
| **OpenTelemetry** | Vendor-neutral standard for distributed tracing, metrics, and logs; instruments services end-to-end |
| **Parquet** | Columnar file format used in S3 cold storage; efficient for analytical queries with Athena/Spark |
| **Rate Limiting** | Throttling outbound or inbound calls to stay within an API provider's limits |
| **Seq Scan** | PostgreSQL full table scan; appears in EXPLAIN output when no index is used; often indicates a missing index |
| **Sidecar** | A helper process running alongside the main application process on the same host |
| **Spot Instance** | AWS EC2 instance available at discount but subject to reclamation with 2-minute notice |
| **Thread Pool Saturation** | Condition where all worker threads are occupied (usually waiting on I/O); new requests queue then fail |
| **Token Bucket** | Rate limiting algorithm that allows bursts up to a capacity, then enforces a steady refill rate |
| **Trace** | End-to-end record of a single request across all services and components; shows exactly where time is spent |
| **Warm Storage** | Medium-cost storage tier (Elasticsearch on HDD) for logs 7–30 days old; slower than hot but cheaper |
