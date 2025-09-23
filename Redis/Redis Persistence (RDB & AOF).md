# Redis Persistence (RDB & AOF)

This document explains how Redis persistence works, including **RDB**, **AOF**, and how both can be used together.

---

## 1. RDB (Redis Database Backup / Snapshotting)

- **What it does**: Creates point-in-time snapshots of the dataset at specified intervals.
- **How it works**:
  - Redis forks a child process.
  - Child writes the in-memory data to a temporary RDB file.
  - File is renamed atomically to replace the old RDB file.

**Pros:**
- Compact file (great for backups).
- Faster restarts (just load one file).
- Low runtime overhead (I/O only during snapshot).

**Cons:**
- Data since the last snapshot may be lost if Redis crashes.
- Not ideal for real-time durability.

---

## 2. AOF (Append-Only File)

- **What it does**: Logs every write operation Redis receives. On restart, Redis replays the log to rebuild the dataset.
- **How it works**:
  - Every modifying command is appended to the AOF file.
  - Flush policies (`appendfsync`):
    - `always` → fsync every write (safest, slowest).
    - `everysec` → fsync once per second (default, good tradeoff).
    - `no` → OS decides when to flush (fastest, riskiest).
  - Redis can rewrite/compact the AOF in the background.

**Pros:**
- Much more durable (lose at most 1 second with `everysec`).
- Human-readable log of operations.

**Cons:**
- Slower than RDB on write-heavy workloads.
- Larger file size.
- Slower restart (must replay log).

---

## 3. Using Both Together

Redis can use **both RDB and AOF** for balance:

- **RDB** → faster restarts & backups.
- **AOF** → real-time durability between snapshots.

### Runtime
- Redis keeps data in memory.
- **RDB** takes periodic full snapshots of the in-memory dataset.
- **AOF** logs every write after the last snapshot, ensuring you don’t lose recent changes between snapshots (`appendonly.aof`).

### Restart / Recovery Rules
1. **If AOF is enabled and valid → Redis loads AOF.**
2. **If AOF is missing/invalid → Redis loads RDB.**

So: **AOF wins** if available.

### Special Case: `aof-use-rdb-preamble yes`
- During AOF rewrite, Redis can start the new AOF with an **RDB snapshot**, then append commands after it.
- On restart, Redis loads:
  - **Step 1:** The RDB preamble (fast bulk load).  
  - **Step 2:** The AOF tail (recent ops after snapshot).

---

## 4. Timeline Diagrams

### AOF Enabled (no preamble)
```
RUNTIME
┌───────────────┬───────────────────────────────┐
│ In-memory ops │  AOF appends each write       │
│ continue      │  (fsync everysec / always)    │
└───────────────┴───────────────────────────────┘
          └─(periodic)─► BGSAVE writes RDB snapshots in background

CRASH / RESTART
┌───────────────────────────────────────────────────────────────┐
│ Redis checks files:                                           │
│   1) AOF present & valid?  YES → Load dataset from AOF.       │
│                           (RDB is ignored)                    │
│   2) Else                  NO  → Load dataset from latest RDB.│
└───────────────────────────────────────────────────────────────┘
```

### AOF with `aof-use-rdb-preamble yes`
```
RUNTIME
┌─────────┬──────────────────────┬───────────────────────────────┐
│ Writes  │ AOF appends commands │ BGREWRITEAOF (periodic/size)  │
│         │                      │ creates NEW AOF as:           │
│         │                      │   [RDB SNAPSHOT] + [AOF TAIL] │
└─────────┴──────────────────────┴───────────────────────────────┘

CRASH / RESTART
┌────────────────────────────────────────────────────────────────┐
│ AOF present & valid → Redis loads AOF:                         │
│   Step A: Read RDB preamble (fast bulk load).                  │
│   Step B: Replay AOF tail (recent ops after snapshot).         │
│ RDB file is *not* used directly here—AOF still “wins.”         │
└────────────────────────────────────────────────────────────────┘
```

---

## 5. Example Config (Balanced Setup)

```conf
# --- RDB ---
save 900 1
save 300 10
save 60 10000
rdbcompression yes
stop-writes-on-bgsave-error yes
dbfilename dump.rdb

# --- AOF ---
appendonly yes
appendfilename appendonly.aof
appendfsync everysec
no-appendfsync-on-rewrite no
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
aof-use-rdb-preamble yes

# --- Storage location ---
dir /var/lib/redis
```

---

## 6. Key Takeaways

- **RDB** = snapshots (fast restart, backups, may lose recent data).  
- **AOF** = command log (durability, larger, slower restart).  
- **Both together** = best of both worlds: fast restarts, real-time durability.
