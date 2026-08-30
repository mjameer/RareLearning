# Read-Your-Write Consistency in a Java Spring Boot Application

## **Overview**
One of the key challenges in scaling databases is ensuring **Read-Your-Write (RYW) consistency**, where a user sees their latest writes on subsequent reads. This is particularly challenging in **master-replica architectures**, where reads are often directed to replicas to reduce load on the primary database. However, **replication lag** can cause a scenario where a user reads stale data from a replica before the latest writes have been propagated.

### **Bitbucket's Approach to Ensuring Read-Your-Write Consistency**
Bitbucket, which uses **PostgreSQL** as its database, employs a **replication lag-aware routing strategy** to ensure **RYW consistency** while offloading as much read traffic as possible to replicas.

#### **1. Master-Replica Setup**
- Writes go to the **master** database.
- Reads typically go to **replicas**, except when a user expects to read their most recent writes.

#### **2. Replication Mechanism**
- Replicas **pull** data from the master using the **Write-Ahead Log (WAL)**.
- Each write entry in WAL is assigned a **Log Sequence Number (LSN)**, which helps determine replication progress.

#### **3. LSN exists on both Primary and Replica — but means different things**

| Database | Query | Meaning |
|---|---|---|
| **Primary (Write)** | `pg_current_wal_lsn()` | "I have written up to this point" |
| **Replica (Read)** | `pg_last_wal_replay_lsn()` | "I have applied/replayed up to this point" |

- The **primary's LSN moves forward** with every new committed write.
- The **replica's LSN moves forward** as it applies WAL entries received from the primary.
- The gap between them is the **replication lag**.

#### **4. Tracking the Latest User Write LSN**
- After a user **writes** to the master, the system **stores the latest LSN** for that user in **Redis** cache, mapping `userID → LSN`.
- The stored LSN serves as a checkpoint indicating the latest write this user expects to see.
- LSN is stored with a **TTL (e.g. 24 hours)** to avoid stale entries accumulating for inactive users.

#### **5. Redis TTL and Null Handling**

| Redis State | Meaning | Action |
|---|---|---|
| Key exists, LSN is recent | User wrote recently, replica might lag | Check replica LSN, route carefully |
| Key exists, LSN is ancient | All replicas will have passed it | Any replica is safe |
| Key expired / missing | No recent write to worry about | Route to any replica freely |

```java
String userLsn = redisTemplate.opsForValue().get("user:lsn:" + userId);

if (userLsn == null) {
    return anyReplica(); // safe — no recent write to be consistent about
}
```

#### **6. Determining the Correct Replica for Reads**
- When a user issues a read request, the middleware:
  - Retrieves the **latest LSN** for that user from Redis.
  - Queries each **replica individually** for its current `replay_lsn`.
  - Directs the read to a **replica that has caught up** with the latest user write, ensuring consistency.

> **Note:** `'user_write_lsn'` in the query below is not a real SQL value — it is the LSN string fetched from Redis at runtime (e.g. `'0/16B2C50'`) and substituted dynamically by the app middleware.

```sql
-- Run this on each replica individually
SELECT pg_wal_lsn_diff('0/16B2C50', pg_last_wal_replay_lsn()) <= 0;
-- TRUE  → replica has caught up → safe to route here
-- FALSE → replica is lagging   → skip, try next replica
```

#### **7. Where Each Query Runs**

| Query | Run On | Purpose |
|---|---|---|
| `pg_current_wal_lsn()` | **Primary** | Capture LSN after a user write |
| `pg_last_wal_replay_lsn()` | Each **Replica** | Per-request routing decision |
| `pg_stat_replication` | **Primary** | Monitoring/ops only — not used for routing |

> `pg_stat_replication` gives an overview of all replicas from the primary but is **slightly stale** (async feedback) and requires a master round-trip — too expensive for per-request routing. Use it for alerting on lagging replicas, not for RYW checks.

#### **8. Performance Impact**
- The system reduces **read queries hitting the master** by **50%**, significantly improving scalability.
- The overhead for checking LSNs (~10ms) is **tolerable**, ensuring fast and scalable performance.

---

### **End-to-End Flow**

```
User writes
    → hits Primary
    → Primary commits, returns LSN "0/16B2C50"
    → App stores in Redis: { userId → "0/16B2C50" } with TTL

User reads
    → App fetches from Redis: userLSN = "0/16B2C50"
    → Ask Replica1: caught up to "0/16B2C50"? → NO  (skip)
    → Ask Replica2: caught up to "0/16B2C50"? → YES (route here)
    → If no replica qualifies → fallback to Primary
```

---

### **Summary of Implementation**
- **Writes** update PostgreSQL and store the latest **LSN** in Redis (with TTL).
- **Reads** fetch the user's LSN from Redis and check each **replica's** `replay_lsn` individually.
- If a **replica has caught up** (`<= 0`), read from it. Otherwise, fall back to the **master**.
- If **Redis has no LSN** for the user, route to any replica freely.

### Reference
[scaling-bitbuckets-database](https://www.atlassian.com/blog/atlassian-engineering/scaling-bitbuckets-database)

---

### PostgreSQL Log Sequence Number (LSN) Queries

#### a. Get the Current LSN
To retrieve the **current LSN** of the master node, use:

```sql
SELECT pg_current_wal_lsn();
```

**Output Example:**
```
 pg_current_wal_lsn 
--------------------
 0/16B2C50
(1 row)
```

This query provides the **current WAL position** of the master database.

---

#### b. Check a Replica's Last Applied LSN
To check the **latest LSN applied** on a read replica, run:

```sql
SELECT pg_last_wal_replay_lsn();
```

**Output Example:**
```
 pg_last_wal_replay_lsn 
-------------------------
 0/16B2A40
(1 row)
```

This LSN represents the **last committed transaction** on the replica.

---

#### c. Find Replication Lag
To calculate the **difference** between the master and the replica (replication lag), use:

```sql
SELECT pg_wal_lsn_diff(pg_current_wal_lsn(), pg_last_wal_replay_lsn());
```

**Output Example:**
```
 pg_wal_lsn_diff 
------------------
            512
(1 row)
```

A higher difference means the **replica is lagging** behind the master.

---

#### d. Monitor LSN Across All Replicas
To get LSN details for all connected replicas, use:

```sql
SELECT application_name, client_addr, state, sent_lsn, write_lsn, flush_lsn, replay_lsn 
FROM pg_stat_replication;
```

**Output Example:**
```
 application_name | client_addr | state     | sent_lsn  | write_lsn | flush_lsn | replay_lsn 
------------------+-------------+-----------+-----------+-----------+-----------+------------
 replica1         | 192.168.1.2 | streaming | 0/16B2C50 | 0/16B2A40 | 0/16B2A40 | 0/16B2A40
 replica2         | 192.168.1.3 | streaming | 0/16B2C50 | 0/16B2B00 | 0/16B2B00 | 0/16B2B00
(2 rows)
```

This query helps monitor replication status and lag across all read replicas. **Run on Primary only — for monitoring, not routing.**

---

#### e. Ensuring Read-Your-Write Consistency
To ensure **read-your-write consistency**, route read queries only to replicas **where the `replay_lsn` is greater than or equal to the user's last written LSN**.

```sql
-- '0/16B2C50' is fetched from Redis at runtime — not a hardcoded value
SELECT pg_wal_lsn_diff('0/16B2C50', pg_last_wal_replay_lsn()) <= 0;
```

If the result is `TRUE`, the replica has caught up; otherwise, fallback to the master for consistency.

---

### Summary

| Query | Run On | Purpose |
|--------|--------|---------|
| `SELECT pg_current_wal_lsn();` | Primary | Get the current LSN after a write |
| `SELECT pg_last_wal_replay_lsn();` | Each Replica | Get the last applied LSN for routing |
| `SELECT pg_wal_lsn_diff(...);` | Each Replica | Check if replica caught up to user's LSN |
| `SELECT * FROM pg_stat_replication;` | Primary | Monitor all replicas (ops/alerting only) |

Using these queries, you can **optimize replication strategies**, detect **lagging replicas**, and ensure **consistent reads across distributed databases**.

---

### How to say it in an interview

> *"We use a replication-lag aware routing strategy. After every write, we capture the WAL log sequence number and store it in Redis for that user. On reads, we compare it against each replica's current replay position. If a replica has caught up, we route there. Otherwise we fall back to the primary. If Redis has no entry for the user — either expired or they never wrote — we route to any replica freely since there's no consistency risk."*
