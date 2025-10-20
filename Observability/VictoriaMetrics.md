

<img width="1736" height="579" alt="image" src="https://github.com/user-attachments/assets/712b4ff9-0b9b-4ca1-8d35-c0057323dcd2" />

Understanding VictoriaMetrics: Architecture, Performance, and Why It’s Faster Than Prometheus

## 1. Why VictoriaMetrics Was Created
Prometheus is excellent for local metrics collection but becomes inefficient as the number of time-series grows (“high cardinality”). The founders of VictoriaMetrics—Aliaksandr Valialkin and team—designed a new system around 2018 to solve three core problems:
- Memory explosion caused by too many unique label combinations.
- Slow startup and high RAM consumption from Prometheus’s in-memory index.
- Limited scalability for large-scale, long-term metrics.

## 2. What Is High Cardinality?
Every unique combination of metric labels defines a separate **time-series**.
Example:
```
http_requests_total{user_id="1"}
http_requests_total{user_id="2"}
```
Two series. If `user_id` has one million values, that’s one million series.

Each unique series consumes:
- **Memory:** to keep active metadata and buffers.
- **CPU:** for ingestion and queries.
- **Storage:** to retain historical samples.

High cardinality (too many label combinations) multiplies resource use.

## 3. What Are Active Series?
An **active series** is one currently receiving new samples.
- Active = data still coming in.
- Idle = recently stopped but still in cache.
- Archived = fully flushed to disk.

Prometheus and VictoriaMetrics both track active series in memory to handle ingestion efficiently.

## 4. Why Active Series Must Stay in Memory
They must:
1. Track the **latest sample** and timestamp.
2. Maintain **write buffers** before compressing to disk.
3. Keep **label-to-ID lookups** for fast ingestion.

Hence more series → more memory.

## 5. How VictoriaMetrics Solves High Cardinality

### 5.1 Compact Label Storage
- Stores metric names and labels once in **deduplicated string tables**.
- Each unique label set gets a numeric **TSID** (Time-Series ID).
- Samples are stored as `(TSID, timestamp, value)` instead of full text labels.

### 5.2 Shared, Disk-Backed Index
- Instead of giant in-RAM hash maps, VM uses **memory-mapped files** for its inverted index (label → TSID).
- Only hot parts are cached; cold ones stay on disk.

### 5.3 Minimal In-Memory State
- Keeps only metadata for truly active series in RAM.
- Old data remains compressed and disk-resident.
- Typical footprint: ~850 MB per million active series (vs several GB in Prometheus).

Result → orders-of-magnitude lower RAM per series.

## 5A. Data Flow Model — Pull and Push Architecture

VictoriaMetrics supports **two ingestion modes**, aligning with Prometheus and modern observability setups:

### 1. Pull Model (Prometheus-Compatible)
- Prometheus or another collector **scrapes metrics** from targets (applications, exporters) and then **pushes them to VictoriaMetrics** via **`remote_write`** API.
- VM acts as the long-term backend; it doesn’t scrape endpoints directly.
- This preserves Prometheus’s pull-based semantics but offloads storage and querying to VM.

### 2. Push Model (via vmagent or other senders)
- The **`vmagent`** component allows metrics to be **pushed directly** to VictoriaMetrics.
- vmagent can also **pull from targets** (like Prometheus does) or **receive pushed data** (via Influx, Graphite, OpenTSDB, etc.) and forward to VM.
- This hybrid approach lets VictoriaMetrics integrate into any telemetry pipeline — whether pull-based (Prometheus scrape) or push-based (IoT devices, edge systems, etc.).

**Result:**
- Pull = Prometheus scrapes → pushes to VM (`remote_write`).
- Push = Applications → vmagent → VM.
- VM focuses on **storage, query, and performance**, while ingestion flexibility is handled by vmagent.

## 5B. vmagent — The Ingestion Engine of VictoriaMetrics
`vmagent` is part of the official VictoriaMetrics ecosystem — developed and maintained by the same team.

### Role of vmagent
Acts as a **metrics ingestion and forwarding layer**, similar to Prometheus’s scraper but far more flexible.

### What vmagent Does
- **Scrapes metrics** from Prometheus-style endpoints (`/metrics`).
- **Receives pushed metrics** via multiple protocols (Prometheus `remote_write`, InfluxDB, Graphite, OpenTSDB, JSON, CSV).
- **Relabels, filters, and aggregates** data before forwarding.
- **Forwards metrics** to one or more remote storage systems (VictoriaMetrics, Thanos, Cortex, Mimir, etc.).

### Why It Exists
Prometheus can only “pull and store.” At scale, that approach causes load and duplication. `vmagent` solves this by:
- Acting as a lightweight, stateless forwarder.
- Reducing Prometheus’s workload.
- Allowing direct metric push into VictoriaMetrics for centralized, scalable storage.

### VictoriaMetrics Ecosystem
| Component | Purpose |
|------------|----------|
| **vmagent** | Collects and pushes metrics to VictoriaMetrics. |
| **vmalert** | Evaluates alerting rules (Prometheus-compatible). |
| **vmstorage / vmselect** | Core storage and query engines (in cluster mode). |
| **vmauth** | Handles multi-tenant authentication and routing. |

`vmagent` effectively replaces Prometheus scrapers, push gateways, and remote writers — unifying all ingestion modes.


## 6. TSID Explained
A **TSID** (Time-Series ID) uniquely identifies each metric + label set.
When a sample arrives:
1. VictoriaMetrics checks if that label set already has a TSID.
2. If yes, it appends the sample to that ID.
3. If no, it creates a new TSID entry and updates the index.

Because data is stored sorted by TSID, labels and names are **not repeated per sample**, drastically cutting storage overhead.

## 7. Prometheus vs. VictoriaMetrics Internals

| Aspect | **Prometheus** | **VictoriaMetrics** |
|---------|----------------|---------------------|
| **ID System** | In-memory, transient series IDs. | Persistent TSIDs reused across restarts. |
| **Label Storage** | Full label maps per active series in RAM. | Deduplicated labels stored once globally. |
| **Write Model** | 2-hour block rotation; frequent compaction. | Continuous append-only storage sorted by TSID. |
| **Index** | Inverted index rebuilt in RAM at startup. | Persistent, memory-mapped index on disk. |
| **Restart** | Scans all blocks to rebuild index (slow). | Opens existing mmap files instantly (fast). |
| **Memory Use** | Grows sharply with cardinality. | Predictable, small footprint. |

## 8. Why Prometheus Rebuilds Indexes on Restart
Prometheus stores data in small 2-hour **blocks**, each with its own index.
When restarted, it must:
1. Open every block’s index.
2. Merge them to form one big in-memory lookup table.
3. Load the latest “head block” into memory for ingestion.

That process touches millions of label entries and uses heavy RAM before queries can run.

## 9. Why VictoriaMetrics Starts Instantly
VictoriaMetrics maintains one global, persistent index on disk.
At restart:
- It **memory-maps** the existing files.
- The OS loads pages only when accessed.
- No rebuild, no merge, no delay.

## 10. Does One Huge Index Slow It Down?
No.
VictoriaMetrics doesn’t keep one monolithic file. It splits data into **daily partitions** and many compact binary files.
Through memory mapping, only the relevant parts are read. The rest stay on disk.

## 11. What Lives in RAM in VictoriaMetrics
- Active series metadata.
- In-RAM write buffers (recent samples).
- Hot cache of label and query results.
- OS page cache for frequently read index pages.

Everything else — historical blocks and cold index pages — stays on disk.

## 12. Why VictoriaMetrics Is Faster Than Prometheus
1. **Append-only architecture** → no in-place updates.
2. **Memory-mapped, persistent index** → zero rebuild time.
3. **TSID-sorted data** → efficient reads and compression.
4. **Streaming query engine (MetricsQL)** → low CPU, low memory per query.
5. **Less locking and garbage creation** → higher write throughput.

At its core:
**Prometheus = in-memory, write-optimized.**
**VictoriaMetrics = append-only + mmap-optimized + globally indexed.**

## 13. “Index on Disk—Won’t That Be Slower?”
No.
Memory mapping makes disk files behave like memory:
- The **OS page cache** keeps hot pages in RAM.
- Reads on those pages are as fast as memory access.
- Cold data stays on disk and is fetched only if queried.

The OS automatically tracks page usage using LRU (Least Recently Used) lists.
Frequently accessed pages remain cached; rarely touched ones get evicted when memory is low.

## 14. “So mmap Is Inefficient for Writes—Doesn’t That Hurt VictoriaMetrics?”
No. VictoriaMetrics sidesteps mmap’s write inefficiency by splitting responsibilities:

**Write path (RAM):**
- New samples land in in-memory buffers.
- Lock-free append pipeline.
- Periodic flush to disk as new immutable files.

**Read path (mmap):**
- Immutable files mapped read-only.
- No mutation, so no page-fault storms.
- Background compaction merges older data asynchronously.

**Result:**
- Writes → pure sequential I/O, high throughput.
- Reads → mmap, fast and memory-efficient.
- No conflict between write speed and mmap design.

## 15. How the Kernel Knows Which Parts to Keep in Memory
The **Linux page cache** tracks every file page (usually 4 KB).
- Each access updates its recency counters.
- Frequently touched pages move to the “active” list.
- Rarely used pages fall to “inactive” and are evicted under pressure.
- Sequential reads trigger **readahead**, preloading upcoming pages.

So the kernel automatically keeps VictoriaMetrics’ most-used index and data pages hot in RAM.

## 16. Why Prometheus Doesn’t Use mmap
- mmap suits **read-mostly** data; Prometheus’s head block is **write-heavy**.
- Thousands of small rotating blocks would fragment mmap usage.
- The project prioritizes simplicity, portability, and small-scale local monitoring.
- Redesigning the TSDB around mmap would break compatibility.

VictoriaMetrics, built later, optimized for **massive scale and long retention**, so it designed mmap and TSID-based indexing from day one.

## 17. Summary
VictoriaMetrics is fundamentally faster and more scalable than Prometheus because it:
- Stores data append-only, sorted by TSID.
- Keeps its global index on disk via mmap (not in RAM).
- Writes new samples to in-RAM buffers and flushes sequentially.
- Uses the kernel’s page cache to keep hot pages in memory automatically.
- Avoids per-restart index rebuilds and memory bloat.
- Cleanly separates read-optimized (mmap) and write-optimized (buffers) paths.

**Outcome:**
- Fast writes, fast queries, low RAM, instant restarts.
- Handles massive high-cardinality workloads that choke Prometheus.

VictoriaMetrics succeeds because it lets the OS manage what’s hot, keeps its writes simple and sequential, and reads via mmap—combining the best of RAM speed and disk scale in one design.
