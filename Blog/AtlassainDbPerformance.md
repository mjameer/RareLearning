# Read-Your-Write Consistency in a Java Spring Boot Application

## **Overview**
One of the key challenges in scaling databases is ensuring **Read-Your-Write (RYW) consistency**, where a user sees their latest writes on subsequent reads. This is particularly challenging in **master-replica architectures**, where reads are often directed to replicas to reduce load on the primary database. However, **replication lag** can cause a scenario where a user reads stale data from a replica before the latest writes have been propagated.

### **Bitbucket’s Approach to Ensuring Read-Your-Write Consistency**
Bitbucket, which uses **PostgreSQL** as its database, employs a **replication lag-aware routing strategy** to ensure **RYW consistency** while offloading as much read traffic as possible to replicas.

#### **1. Master-Replica Setup**
- Writes go to the **master** database.
- Reads typically go to **replicas**, except when a user expects to read their most recent writes.

#### **2. Replication Mechanism**
- Replicas **pull** data from the master using the **Write-Ahead Log (WAL)**.
- Each write entry in WAL is assigned a **Log Sequence Number (LSN)**, which helps determine replication progress.

#### **3. Tracking the Latest User Write LSN**
- After a user **writes** to the master, the system **stores the latest LSN** for that user in **Redis** cache, mapping userID → LSN.
- The stored LSN serves as a checkpoint indicating the latest write this user expects to see.

#### **4. Determining the Correct Replica for Reads**
- When a user issues a read request, the middleware:
  - Retrieves the **latest LSN** for that user from Redis.
  - Queries all **replicas** for their current **LSN** progress.
  - Directs the read to a **replica that has caught up** with the latest user write, ensuring consistency.


##### . Get the Current LSN
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

##### b. Check a Replica’s Last Applied LSN
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

##### c. Find Replication Lag
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

##### d. Monitor LSN Across All Replicas
To get LSN details for all connected replicas, use:

```sql
SELECT application_name, client_addr, state, sent_lsn, write_lsn, flush_lsn, replay_lsn 
FROM pg_stat_replication;
```

**Output Example:**
```
 application_name | client_addr | state  | sent_lsn | write_lsn | flush_lsn | replay_lsn 
------------------+------------+--------+----------+-----------+-----------+------------
 replica1        | 192.168.1.2 | streaming | 0/16B2C50 | 0/16B2A40 | 0/16B2A40 | 0/16B2A40
 replica2        | 192.168.1.3 | streaming | 0/16B2C50 | 0/16B2B00 | 0/16B2B00 | 0/16B2B00
(2 rows)
```

This query helps monitor replication status and lag across all read replicas.

---

##### e. Ensuring Read-Your-Write Consistency
To ensure **read-your-write consistency**, route read queries only to replicas **where the `replay_lsn` is greater than or equal to the user’s last written LSN**.

```sql
SELECT pg_wal_lsn_diff('user_write_lsn', pg_last_wal_replay_lsn()) <= 0;
```

If the result is `TRUE`, the replica has caught up; otherwise, fallback to the master for consistency.

---

## Summary
| Query | Purpose |
|--------|---------|
| `SELECT pg_current_wal_lsn();` | Get the current LSN on master |
| `SELECT pg_last_wal_replay_lsn();` | Get the last applied LSN on a replica |
| `SELECT pg_wal_lsn_diff(pg_current_wal_lsn(), pg_last_wal_replay_lsn());` | Check replication lag |
| `SELECT * FROM pg_stat_replication;` | Monitor all replicas |

Using these queries, you can **optimize replication strategies**, detect **lagging replicas**, and ensure **consistent reads across distributed databases**.


#### **5. Performance Impact**
- The system reduces **read queries hitting the master** by **50%**, significantly improving scalability.
- The overhead for checking LSNs (~10ms) is **tolerable**, ensuring fast and scalable performance.


### **Summary of Implementation**
- **Writes** update PostgreSQL and store the latest **LSN** in Redis.
- **Reads** check **replication lag** by comparing the **replica LSN** with the user's latest **LSN**.
- If the **replica is caught up**, read from it. Otherwise, read from the **master**.

This approach **reduces master database load**, ensuring **Read-Your-Write consistency** while maintaining performance.

