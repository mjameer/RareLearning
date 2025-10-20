# Understanding VictoriaMetrics: Architecture, Performance, and Why It’s Faster Than Prometheus

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
