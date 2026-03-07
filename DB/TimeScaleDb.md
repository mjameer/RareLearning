# TimescaleDB — Junior Engineer's Complete Guide

> Built from first principles. Every concept explained from scratch.
> Covers every "why", "what", and "how" a junior would ask.

---

## Table of Contents

1. [The Problem TimescaleDB Solves](#1-the-problem-timescaledb-solves)
2. [What Is TimescaleDB](#2-what-is-timescaledb)
3. [What Is Time Series Data](#3-what-is-time-series-data)
4. [What Is a Hypertable](#4-what-is-a-hypertable)
5. [What Is a Chunk](#5-what-is-a-chunk)
6. [Is TimescaleDB Only Active During Queries](#6-is-timescaledb-only-active-during-queries)
7. [What Is Compression](#7-what-is-compression)
8. [Row Format vs Columnar Format](#8-row-format-vs-columnar-format)
9. [How Queries Still Work After Compression](#9-how-queries-still-work-after-compression)
10. [The Three Compression Settings](#10-the-three-compression-settings)
11. [When Does Compression Actually Run](#11-when-does-compression-actually-run)
12. [What Are Continuous Aggregates](#12-what-are-continuous-aggregates)
13. [Docker Image](#13-docker-image)
14. [Putting It All Together — Full Setup](#14-putting-it-all-together--full-setup)
15. [The Complete Day-by-Day Lifecycle](#15-the-complete-day-by-day-lifecycle)
16. [When TimescaleDB Is the Wrong Tool](#16-when-timescaledb-is-the-wrong-tool)
17. [Summary — Three Settings You Must Never Confuse](#17-summary--three-settings-you-must-never-confuse)

---

## 1. The Problem TimescaleDB Solves

Imagine you join a stock trading company. On day 1 your manager says:

> "We store every price tick for 500 stocks. That's 10 million rows per day.
> After 6 months Grafana dashboards are timing out. Fix it."

You look at the database. It is vanilla PostgreSQL. One giant `trades` table.
1.8 billion rows. Every dashboard query scans the whole thing.

This is exactly the problem TimescaleDB was built for.

```
Vanilla PostgreSQL after 1 year:

SELECT avg(price) FROM trades WHERE time > now() - INTERVAL '7 days';

PostgreSQL must:
  → Open the ENTIRE trades table (1.8 billion rows)
  → Read every single row from disk
  → Filter out 99.9% of rows after reading them
  → Return result

Like searching for last week's emails by reading
every email you ever sent since college.
```

---

## 2. What Is TimescaleDB

Not a replacement for PostgreSQL. Not a separate database.

```
TimescaleDB = PostgreSQL + an extension installed on top
```

Everything you know about Postgres still works:

| Works as before        | New capabilities added                        |
|------------------------|-----------------------------------------------|
| Same SQL               | Hypertables (auto partitioning by time)       |
| Same `psql`            | Compression (columnar storage for old data)   |
| Same `pg_dump`         | Continuous Aggregates (auto-refreshing summaries) |
| Same Grafana datasource| Retention Policies (auto-delete old data)     |
| Same indexes, JOINs    | Background job scheduler                      |

Think of it like installing an app on your phone. The phone (PostgreSQL) works
exactly the same. The app (TimescaleDB) adds new capabilities on top.

---

## 3. What Is Time Series Data

Time series data has three characteristics:

```
1. Every row has a timestamp
   trades: (2025-03-07 09:30:00, AAPL, 195.0, 1000)
                ↑ always present

2. Rows arrive in time order — mostly append-only
   You insert new prices. You rarely update old prices.
   "What was AAPL at 09:30 yesterday" never changes.

3. You almost always query by time range
   "Give me last 7 days"
   "Give me March data"
   "Give me last 1 hour"
```

Examples:

| Domain        | Data              | Frequency          |
|---------------|-------------------|--------------------|
| Stock trading | Price ticks       | Every millisecond  |
| IoT           | Sensor temperature| Every 5 seconds    |
| DevOps        | CPU/memory metrics| Every minute       |
| Web analytics | User click events | Continuous         |
| Weather       | Station readings  | Every hour         |

---

## 4. What Is a Hypertable

A hypertable looks like one normal table to you. Behind the scenes TimescaleDB
splits it into many small tables called **chunks**, each covering a specific
time window.

```
What YOU see:
┌────────────────────────────────────────┐
│          trades  (hypertable)          │
│       all your data, all time          │
└────────────────────────────────────────┘

What TimescaleDB actually stores on disk:
┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐
│ chunk_1  │  │ chunk_2  │  │ chunk_3  │  │ chunk_N  │
│  Jan 1   │  │  Jan 2   │  │  Jan 3   │  │  Today   │
│  (raw)   │  │  (raw)   │  │  (raw)   │  │  (raw)   │
└──────────┘  └──────────┘  └──────────┘  └──────────┘

You always query trades — TimescaleDB routes to the right chunks invisibly.
```

### Chunk Exclusion — Why This Matters

```
Query: WHERE time > now() - INTERVAL '7 days'

                ┌──────────────────────────────────┐
                │    TimescaleDB Query Planner      │
                └──────────────┬───────────────────┘
                               │
          ┌────────────────────┼────────────────────┐
          │                    │                    │
          ▼                    ▼                    ▼
   ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
   │  chunk_1    │     │  chunk_2    │     │  chunk_358  │
   │  Jan        │     │  Feb        │     │  ...        │
   │  ✗ SKIP     │     │  ✗ SKIP     │     │  ✗ SKIP     │
   └─────────────┘     └─────────────┘     └─────────────┘

          ┌────────────────────┬────────────────────┐
          ▼                    ▼                    ▼
   ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
   │  chunk_359  │     │  chunk_360  │     │  chunk_361  │
   │  Mar 1      │     │  Mar 2      │     │  Mar 3      │
   │  ✓ READ     │     │  ✓ READ     │     │  ✓ READ     │
   └─────────────┘     └─────────────┘     └─────────────┘

358 chunks never touched. Only last 7 opened. This is CHUNK EXCLUSION.
```

---

## 5. What Is a Chunk

A chunk is a real, physical PostgreSQL table. You just never interact with it
directly.

```sql
-- After create_hypertable, TimescaleDB creates under the hood:
_timescaledb_internal._hyper_1_1_chunk  -- chunk for day 1
_timescaledb_internal._hyper_1_2_chunk  -- chunk for day 2
_timescaledb_internal._hyper_1_3_chunk  -- chunk for day 3
-- ... one per day (or week, or hour — whatever you configured)
```

You can see your chunks at any time:

```sql
SELECT chunk_name,
       range_start,
       range_end,
       is_compressed
FROM timescaledb_information.chunks
WHERE hypertable_name = 'trades';

-- Output:
-- _hyper_1_1_chunk | 2025-03-01 | 2025-03-02 | false
-- _hyper_1_2_chunk | 2025-03-02 | 2025-03-03 | false
-- _hyper_1_3_chunk | 2025-03-03 | 2025-03-04 | true
```

**What does `chunk_time_interval => INTERVAL '1 day'` mean?**

Only one thing: **how wide is each chunk in time**.

```
'1 day'  → each chunk holds exactly 1 day of data
'1 week' → each chunk holds exactly 1 week of data
'1 hour' → each chunk holds exactly 1 hour of data

Has NOTHING to do with compression.
Has NOTHING to do with how often jobs run.
Only controls: how wide is each time window per chunk.
```

---

## 6. Is TimescaleDB Only Active During Queries?

**No. TimescaleDB is always working — not just during queries.**

```
ON EVERY INSERT:

  INSERT INTO trades VALUES ('2025-03-07 09:30', 'AAPL', 195.0, 1000)
                                       │
                        TimescaleDB reads the timestamp
                                       │
                        "March 7 belongs to chunk_47"
                                       │
                        Writes row to chunk_47
                        You never see this happening

ON EVERY QUERY:

  WITH time filter    → skips irrelevant chunks (fast)
  WITHOUT time filter → scans all chunks (same as vanilla Postgres, slow)

BACKGROUND — always running on a schedule:

  ┌─────────────────────────────────────────────────┐
  │  Compression Job    → compresses old chunks     │
  │  Aggregate Refresh  → updates pre-computed views│
  │  Retention Job      → drops expired chunks      │
  └─────────────────────────────────────────────────┘
```

| Activity              | When               | What TimescaleDB Does                        |
|-----------------------|--------------------|----------------------------------------------|
| Every INSERT          | Always             | Routes row to correct chunk by timestamp     |
| SELECT with time filter | Query time       | Skips chunks outside time range (fast)       |
| SELECT without filter | Query time         | Scans all chunks (same as vanilla Postgres)  |
| Compression job       | Background schedule| Compresses old chunks                        |
| Aggregate refresh     | Background schedule| Updates continuous aggregate summaries       |
| Retention job         | Background schedule| Drops chunks older than threshold            |

---

## 7. What Is Compression

Old chunks stop receiving new data. TimescaleDB can compress them to save space
and speed up reads.

```
Raw row format after 1 year:    15 GB
After compression on old data:   1.5 GB
                                ──────
Savings:                         90%
```

Compression does two things:

```
1. Changes storage format: row → columnar
   Similar values stored together → much smaller on disk

2. Changes access pattern: read all columns → read only needed columns
   Query for price only → only price bytes read from disk
   Volume and symbol bytes never touched at all
```

---

## 8. Row Format vs Columnar Format

### Row Format — How All Databases Default

Data is stored row by row on disk. To compute `avg(price)`, every field of
every row must be read:

```
DISK (row format):

[09:30][AAPL][195.0][1000]  [09:31][AAPL][195.2][900]  [09:32][TSLA][250.0][800]
 ←──────── row 1 ──────────→ ←──────── row 2 ──────────→ ←──────── row 3 ──────────→

Query: SELECT avg(price)

  [09:30] → need? NO  (time)
  [AAPL]  → need? NO  (symbol)
  [195.0] → need? YES ✓
  [1000]  → need? NO  (volume)
  [09:31] → need? NO
  [AAPL]  → need? NO
  [195.2] → need? YES ✓
  [900]   → need? NO
  ...

Result: read 12 values, only 3 were useful. 75% of disk reads wasted.
```

### Columnar Format — After TimescaleDB Compression

Same column values stored together on disk. Only the relevant column block
is read:

```
DISK (columnar format):

[09:30][09:31][09:32]  [AAPL][AAPL][TSLA]  [195.0][195.2][250.0]  [1000][900][800]
 ←── all times ───────  ←── all symbols ──  ←──── all prices ─────  ←─ all volumes

Query: SELECT avg(price)

  time block    → SKIP entirely, never opened
  symbol block  → SKIP entirely, never opened
  price block   → READ all values  ✓ ✓ ✓
  volume block  → SKIP entirely, never opened

Result: read 3 values, all 3 useful. 0% waste.
```

### Why 90% Smaller

Similar values stored next to each other compress extremely well:

```
AAPL prices stored together:  195.0,  195.2,  195.1,  195.3,  195.0

Compression algorithm sees:
  Base:   195.0
  Deltas: +0.2, -0.1, +0.2, -0.3   ← tiny numbers, compress hugely

Stores: one base number + four tiny deltas
Instead of: five full decimal numbers

In row format, 195.0 neighbors on disk are [AAPL] and [1000]
— completely different types, nothing to compress.
```

---

## 9. How Queries Still Work After Compression

When data is spread across column blocks, how does TimescaleDB know which
price belongs to which row?

**The answer: position index.**

Every column block stores values in the same order. Position 1 in the time
block always corresponds to position 1 in the price block — always.

```
Position:      [0]      [1]      [2]

time block:  [09:30]  [09:31]  [09:32]
symbol block:[AAPL]   [AAPL]   [TSLA]
price block: [195.0]  [195.2]  [250.0]
volume block:[1000]   [900]    [800]

Position 0 = complete row: 09:30, AAPL, 195.0, 1000
Position 1 = complete row: 09:31, AAPL, 195.2, 900
Position 2 = complete row: 09:32, TSLA, 250.0, 800

Position index is the invisible thread tying all columns together.
```

### Query walkthrough

```sql
SELECT price FROM trades
WHERE symbol = 'AAPL' AND time BETWEEN '09:30' AND '09:32';
```

```
Step 1 — Read time block, find matching positions:

  time block: [09:30]  [09:31]  [09:32]
               pos 0    pos 1    pos 2

  09:30 → outside range → pos 0 = NO
  09:31 → inside range  → pos 1 = YES ✓
  09:32 → outside range → pos 2 = NO

  Bitmask: [ 0, 1, 0 ]

Step 2 — Check symbol block at surviving positions only:

  Only check pos 1 (where bitmask = 1)
  pos 1 → AAPL → matches ✓
  Bitmask still: [ 0, 1, 0 ]

Step 3 — Fetch price block at surviving positions only:

  Only read pos 1
  pos 1 → 195.2  ← answer

  Volume block: never opened
  pos 0 and pos 2: never fetched from any block
```

---

## 10. The Three Compression Settings

```sql
ALTER TABLE trades SET (
    timescaledb.compress,                        -- Setting 1
    timescaledb.compress_segmentby = 'symbol',   -- Setting 2
    timescaledb.compress_orderby   = 'time DESC' -- Setting 3
);
```

### Setting 1 — `timescaledb.compress`

Just a flag. "Compression is allowed on this table." Does not compress
anything yet. Without this TimescaleDB refuses to compress at all.

### Setting 2 — `compress_segmentby = 'symbol'`

Before compressing, group rows by symbol first. Each symbol gets its own
compressed block:

```
WITHOUT segmentby — one mixed block:

  pos: [0]      [1]      [2]      [3]      [4]
  sym: [AAPL]   [TSLA]   [AAPL]   [NVDA]   [AAPL]
  pri: [195.0]  [250.0]  [195.2]  [880.0]  [195.1]

  AAPL query must scan all 5 positions to find AAPL rows.


WITH segmentby = 'symbol' — separate block per symbol:

  AAPL block:                TSLA block:            NVDA block:
  pos: [0]    [1]    [2]     pos: [0]    [1]        pos: [0]
  pri: [195.0][195.2][195.1] pri: [250.0][250.5]    pri: [880.0]

  AAPL query opens AAPL block only.
  TSLA and NVDA blocks never touched, never decompressed.
```

### Setting 3 — `compress_orderby = 'time DESC'`

Within each symbol group, sort rows newest first before compressing:

```
AAPL block sorted time DESC:

  pos 0 → 09:36  (newest)
  pos 1 → 09:33
  pos 2 → 09:31  (oldest)

Query: WHERE time > 09:33

  pos 0: 09:36 ✓ keep
  pos 1: 09:33 ✗ stop — sorted DESC, everything below is older

Finds result without scanning the full block.
```

---

## 11. When Does Compression Actually Run

**Compression does NOT run continuously. It runs on a schedule like a cron job.**

```
Timeline of one day:

00:00 ─────────────────────────────────────────────── 24:00
  │
  │   Normal inserts all day
  │   No compression happening
  │   Active chunk fully available for reads/writes
  │
02:00 ──► background job wakes up
          "Any chunks older than 7 days?"
           YES → compress them (takes minutes)
           NO  → go back to sleep

Next day 02:00 ──► wakes up again
```

It is NOT:
- Compressing every insert
- Compressing every minute
- Running continuously

It IS:
- A scheduled background job inside TimescaleDB's own scheduler
- Runs periodically (default every 1 day)
- Only touches chunks that qualify (older than threshold)
- Leaves active/recent chunks completely alone

```sql
-- See all scheduled jobs TimescaleDB created
SELECT job_id,
       application_name,
       schedule_interval,
       next_start,
       last_run_status
FROM timescaledb_information.jobs;

-- Output:
-- 1000 | Compression Policy [trades] | 1 day | 2025-03-08 02:00:00 | Success

-- Change when it runs
SELECT alter_job(1000,
    next_start => '2025-03-08 02:00:00'
);
```

### Three completely separate settings — people always confuse these

```
┌──────────────────────────────────────────────────────────────────┐
│  chunk_time_interval = '1 day'                                   │
│                                                                  │
│  Question: How WIDE is each chunk?                               │
│  Answer:   Each chunk covers exactly 1 calendar day of data      │
│  Has NOTHING to do with compression                              │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│  compress_after = '7 days'                                       │
│                                                                  │
│  Question: How OLD must a chunk be before compressing?           │
│  Answer:   Only compress chunks older than 7 days                │
│  Has NOTHING to do with chunk width                              │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│  background job schedule_interval                                │
│                                                                  │
│  Question: How OFTEN does the job wake up and check?             │
│  Answer:   Default every 1 day — checks at scheduled time        │
│  Has NOTHING to do with chunk width or compression threshold     │
└──────────────────────────────────────────────────────────────────┘
```

---

## 12. What Are Continuous Aggregates

**The problem:**

Your Grafana dashboard shows hourly OHLCV charts. Every page load runs:

```sql
SELECT time_bucket('1 hour', time), avg(price)
FROM trades WHERE symbol = 'AAPL'
GROUP BY 1;
```

After 1 year this query scans millions of raw rows every single time. Slow.

**The solution — pre-compute the summary once, refresh automatically:**

```sql
CREATE MATERIALIZED VIEW hourly_ohlcv
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 hour', time) AS bucket,
    symbol,
    first(price, time) AS open,
    max(price)         AS high,
    min(price)         AS low,
    last(price, time)  AS close,
    sum(volume)        AS volume
FROM trades
GROUP BY bucket, symbol;

-- Auto-refresh every hour
SELECT add_continuous_aggregate_policy('hourly_ohlcv',
    start_offset      => INTERVAL '2 hours',
    end_offset        => INTERVAL '1 hour',
    schedule_interval => INTERVAL '1 hour'
);
```

### Why better than a regular Postgres materialized view

```
POSTGRES MATERIALIZED VIEW — gets slower every day:

  Monday    REFRESH → scans ALL 365 days → rebuilds from scratch
  Tuesday   REFRESH → scans ALL 365 days → rebuilds from scratch
  Wednesday REFRESH → scans ALL 365 days → rebuilds from scratch
  ...
  Gets slower as history grows. A full year = full year scanned every refresh.


TIMESCALEDB CONTINUOUS AGGREGATE — always fast:

  Monday    → compute everything → watermark set to Mon 23:59
                                          │
  Tuesday   → only process Tue data ──────┘ watermark moves to Tue 23:59
                                          │
  Wednesday → only process Wed data ──────┘ watermark moves to Wed 23:59

  Always fast regardless of how much historical data exists.
  Only new data since the last watermark is ever processed.
```

Grafana queries the aggregate (24 rows/day) instead of raw data (millions/day).

---

## 13. Docker Image

```bash
# Vanilla PostgreSQL — no TimescaleDB
docker pull postgres:16

# TimescaleDB — PostgreSQL 16 with extension pre-installed
docker pull timescale/timescaledb-ha:pg16

# Run with Podman
podman run -d --name timescaledb \
  -p 5432:5432 \
  -e POSTGRES_PASSWORD=secret \
  timescale/timescaledb-ha:pg16
```

The `-ha` image is PostgreSQL 16 with TimescaleDB already installed inside it.
Nothing is removed. All Postgres tools (`psql`, `pg_dump`, Grafana, etc.) work
normally.

After connecting, activate the extension per database — **once only:**

```sql
-- Run once per database.
-- IF NOT EXISTS = safe to run again, no error if already active.
CREATE EXTENSION IF NOT EXISTS timescaledb;
```

Think of it like installing an app vs opening it:
- The Docker image = app installed on your phone
- `CREATE EXTENSION` = you open/activate it in that specific database

---

## 14. Putting It All Together — Full Setup

```sql
-- Step 1: Create normal Postgres table
CREATE TABLE trades (
    time    TIMESTAMPTZ NOT NULL,
    symbol  TEXT        NOT NULL,
    price   NUMERIC     NOT NULL,
    volume  BIGINT
);

-- Step 2: Convert to hypertable
-- Each chunk = 1 day of data
SELECT create_hypertable('trades', 'time',
    chunk_time_interval => INTERVAL '1 day'
);

-- Step 3: Tell TimescaleDB HOW to compress (the recipe)
-- This does NOT compress anything yet — just configuration
ALTER TABLE trades SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'symbol',
    timescaledb.compress_orderby   = 'time DESC'
);

-- Step 4: Schedule the compression job
-- Compress chunks older than 7 days
-- Background job checks every 1 day automatically
SELECT add_compression_policy('trades',
    compress_after => INTERVAL '7 days'
);

-- Step 5: Create hourly summary for dashboards
CREATE MATERIALIZED VIEW hourly_ohlcv
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 hour', time) AS bucket,
    symbol,
    first(price, time) AS open,
    max(price)         AS high,
    min(price)         AS low,
    last(price, time)  AS close,
    sum(volume)        AS volume
FROM trades
GROUP BY bucket, symbol;

SELECT add_continuous_aggregate_policy('hourly_ohlcv',
    start_offset      => INTERVAL '2 hours',
    end_offset        => INTERVAL '1 hour',
    schedule_interval => INTERVAL '1 hour'
);

-- Step 6: Auto-delete data older than 1 year
SELECT add_retention_policy('trades',
    drop_after => INTERVAL '1 year'
);
```

After this setup — **you just use it like a normal Postgres table forever:**

```sql
-- Normal insert — TimescaleDB routes to correct chunk automatically
INSERT INTO trades VALUES (now(), 'AAPL', 195.0, 1000);

-- Normal query — chunk exclusion skips old chunks automatically
SELECT avg(price) FROM trades
WHERE symbol = 'AAPL'
AND time > now() - INTERVAL '30 days';

-- Dashboard query — hits pre-computed aggregate, not raw data
SELECT bucket, open, high, low, close, volume
FROM hourly_ohlcv
WHERE symbol = 'AAPL'
AND bucket > now() - INTERVAL '7 days';
```

---

## 15. The Complete Day-by-Day Lifecycle

### Days 1–7: Raw chunks filling up

```
Mar 1      Mar 2      Mar 3      Mar 4      Mar 5      Mar 6      Mar 7
┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐
│chunk_1 │ │chunk_2 │ │chunk_3 │ │chunk_4 │ │chunk_5 │ │chunk_6 │ │chunk_7 │
│  raw   │ │  raw   │ │  raw   │ │  raw   │ │  raw   │ │  raw   │ │  raw   │
└────────┘ └────────┘ └────────┘ └────────┘ └────────┘ └────────┘ └────────┘
                                                                   ↑ active
```

### Day 8: First compression run at 02:00

```
Background job wakes up:
  chunk_1 = 8 days old  →  YES → compress ✓
  chunk_2 = 7 days old  →  borderline → skip
  chunk_3 to chunk_8    →  too recent → skip

Mar 1      Mar 2      Mar 3      ...        Mar 8
┌────────┐ ┌────────┐ ┌────────┐            ┌────────┐
│chunk_1 │ │chunk_2 │ │chunk_3 │            │chunk_8 │
│  COMP  │ │  raw   │ │  raw   │            │  raw   │
└────────┘ └────────┘ └────────┘            └────────┘
↑ 90% smaller                               ↑ active
```

### Day 9 onwards: One chunk compressed per day

```
Day 9:   chunk_2 compressed
Day 10:  chunk_3 compressed
...
Day 14:  chunk_7 compressed

Mar 1      Mar 2      Mar 3      Mar 4      Mar 5      Mar 6      Mar 7
┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐
│chunk_1 │ │chunk_2 │ │chunk_3 │ │chunk_4 │ │chunk_5 │ │chunk_6 │ │chunk_7 │
│  COMP  │ │  COMP  │ │  COMP  │ │  COMP  │ │  COMP  │ │  COMP  │ │  COMP  │
└────────┘ └────────┘ └────────┘ └────────┘ └────────┘ └────────┘ └────────┘
```

### Day 365: Retention job drops oldest chunks

```
chunk_1 = 365 days old → DROP instantly
Dropping a chunk = dropping one physical table = near-instant
No row-by-row DELETE scan needed
```

### Full lifecycle in one view

```
INSERT arrives
     │
     ▼
TimescaleDB routes to correct chunk by timestamp
     │
     ▼
┌─────────────────────┐
│   Raw chunk         │  ← fast inserts, recent queries
│   row format        │
└─────────────────────┘
     │
     │  7 days pass
     │  background compression job runs
     ▼
┌─────────────────────┐
│   Compressed chunk  │  ← 90% smaller, fast analytical reads
│   columnar format   │
└─────────────────────┘
     │                 ├──► Continuous aggregate refreshes hourly
     │                 │         │
     │                 │         ▼
     │                 │    Grafana queries aggregate
     │                 │    (24 rows/day not millions)
     │  1 year passes
     │  retention job runs
     ▼
  DROPPED instantly
  storage stays bounded forever
```

---

## 16. When TimescaleDB Is the Wrong Tool

```
USE TimescaleDB when:                  AVOID when:
──────────────────────────────────     ──────────────────────────────────
Data always has a timestamp            Data has no time component
Mostly INSERT, rarely UPDATE           Frequently update historical rows
Query mostly by time range             Random access patterns
Data grows forever                     Fixed or small dataset
Need storage savings on old data       Storage is not a concern
Grafana dashboards on metrics          General purpose OLTP application
IoT / trading / monitoring             Transactional app (orders, users)
```

**The critical limitation — updates on compressed data:**

```sql
-- Vanilla Postgres: instant
UPDATE trades SET price = 195.5 WHERE time = '09:31' AND symbol = 'AAPL';

-- TimescaleDB compressed chunk: expensive
-- 1. Decompress the entire AAPL segment
-- 2. Update the one row
-- 3. Recompress the entire AAPL segment
-- 4. Write back to disk
-- → Much slower, much more disk I/O
```

TimescaleDB is designed for **append-only** workloads. Historical data does not
change. New data keeps arriving. This matches perfectly with: stock ticks,
sensor readings, server metrics, event logs.

---

## 17. Summary — Three Settings You Must Never Confuse

```
chunk_time_interval = '1 day'
  → How WIDE is each chunk
  → 1 day of data per chunk
  → Nothing to do with compression

compress_after = '7 days'
  → How OLD must a chunk be before compressing
  → Nothing to do with chunk width

schedule_interval = '1 day'  (background job)
  → How OFTEN the compression job wakes up and checks
  → Nothing to do with chunk width or compression threshold
```

These three independent settings work together:

```
Recent data   → raw chunks     → fast inserts, fast recent queries
Old data      → compressed     → 90% smaller, fast analytical reads
Very old data → dropped        → storage stays bounded forever
Dashboards    → aggregates     → pre-computed, instant response
```
