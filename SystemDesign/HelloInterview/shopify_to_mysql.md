# Shopify: Replacing Redis with MySQL for Inventory Reservations

> **Two lenses** — every section annotated for:
> - 🎓 **1st Year CS Student** — plain concepts, analogies
> - 🏗️ **Principal Engineer** — tradeoffs, architecture decisions

---

## The Problem

When a buyer purchases on Shopify, Shopify must guarantee the item is actually available. Getting this wrong fails in two directions:

| Failure | Impact |
|---|---|
| Two buyers claim the same last unit | One gets a cancellation + refund + apology email |
| Buyer thinks item is sold out (but it isn't) | Merchant loses a sale |

---

## The Solution: Reservation System

1. Buyer puts item in cart → item is **reserved** for a time window
2. Buyer completes payment → unit **permanently deducted** from inventory
3. Payment fails / timer expires → unit goes **back into the pool**

---

## Part 1 — The Old System: Redis

### How It Worked

- One Redis key per inventory item (e.g. `blue-hoodie`)
- Value of that key = units currently available to sell
- Reserve → `DECREMENT` the key
- Abandon → `INCREMENT` the key back
- **Source of truth**: MySQL. Redis was just the fast layer.

```
Initial:   Redis=10, MySQL=10
Reserve:   Redis=9,  MySQL=10   ← user in checkout
Purchase:  Redis=9,  MySQL=9    ← synced on success
Abandon:   Redis must go back to 10 somehow...
```

---

### 🎓 Why Redis?

Redis is like a **whiteboard counter** — super fast to update (in-memory). MySQL is the filing cabinet — reliable but slower. During a flash sale with thousands of concurrent checkouts, you don't want everyone waiting on the filing cabinet.

### 🏗️ Why Redis?

Redis's in-memory O(1) operations on a single key handle high concurrent read/write throughput without row-lock contention. A counter in MySQL would serialize all checkouts behind a row lock on the same row — one transaction at a time.

---

### Handling Expired Reservations (Video's Best Guess)

The blog doesn't explain this. Likely design in Redis:

1. **Counter key** — raw count of available units (increment/decrement)
2. **Sorted Set** — one entry per outstanding reservation, scored by expiry time. Top of the set = soonest to expire.

**Cron job** periodically peeks at the top of the sorted set, removes expired entries, increments the counter back.

**Problem with cron**: if it runs every 10s, a flash sale could show "sold out" for up to 10 seconds after reservations expired — lost revenue.

**Better fix — Lua script (atomic)**: Before taking a reservation, inline-check the top of the sorted set for anything already expired, remove it, increment the counter, then run your own decrement. All in one Lua script = atomic, like a DB transaction. Expired inventory reclaimed the moment the next buyer needs it. Cron stays as a backstop.

---

### 🎓 Lua Script = Atomic

Like a bank teller with a sealed instruction card: "Do ALL these steps before helping the next person." Nobody interrupts in the middle.

### 🏗️ Lua Atomicity

Redis's single-threaded model guarantees Lua runs without interleaving — ACID-like atomicity on read-modify-write with no CAS loops. Tradeoff: long scripts block all other Redis commands, so keep them short.

---

## Part 2 — The Real Problem: Two Systems, No Atomic Commit

When payment **succeeds**, two writes must happen:
1. MySQL: deduct inventory
2. Redis: remove the reservation from the sorted set

**No way to wrap these in one atomic transaction** — two different systems.

If the process crashes between the two writes:
- MySQL deducted ✅
- Redis reservation still sitting there ❌ → eventually reclaimed → inventory count inflated → overselling

---

### 🎓 Why This Is Hard

Cashier takes your money (MySQL) but forgets to pull your reserved ticket off the board (Redis). Someone else grabs that ticket and claims the same item.

### 🏗️ Two-Phase Commit Isn't Practical

Redis doesn't participate in distributed transactions. Fixes like outbox pattern, saga, or compensating transactions all add latency and complexity. Cleaner answer: eliminate the dual-write by putting everything in one store.

---

## Part 3 — The New System: MySQL Only + SKIP LOCKED

### Why MySQL-Only Had Failed Before

**Attempt 1 — Single row:**
```sql
-- one row for the item
item_id | available_qty
hoodie  | 10

UPDATE inventory SET available_qty = available_qty - 1 WHERE item_id = 'hoodie';
```
MySQL gives a row lock to one transaction at a time. All checkouts queue up in a single-file line. Throughput collapses.

**Attempt 2 — One row per unit:**
```sql
id | item_id | status
1  | hoodie  | available
2  | hoodie  | available
...
10 | hoodie  | available

SELECT * FROM reservation_pool WHERE item_id='hoodie' AND status='available' FOR UPDATE LIMIT 1;
```
Logical idea — split the one contested row into 10. But `FOR UPDATE` without `SKIP LOCKED` stops and waits at the **first locked row it hits**. Buyer 2 queues behind Buyer 1 even though rows 2–10 are free. Still a single-file line.

---

### The Breakthrough: MySQL 8 (2018) — `SKIP LOCKED`

```sql
SELECT * FROM reservation_pool
WHERE item_id = 'hoodie' AND status = 'available'
FOR UPDATE SKIP LOCKED
LIMIT 1;
```

When the scan hits a locked row, instead of waiting — it **skips it and keeps looking** for the next free one. Buyer 1 locks row 1, Buyer 2 skips row 1 and locks row 2, both commit at the same time.

---

### 🎓 SKIP LOCKED

10 checkout lanes at a supermarket. Old rule: everyone must try lane 1 first — if it's busy, wait, even if lanes 2–10 are empty. `SKIP LOCKED` = "lane busy? move to the next open one." All 10 customers go simultaneously.

### 🏗️ SKIP LOCKED as a Queue Primitive

`SELECT ... FOR UPDATE SKIP LOCKED` is a non-blocking dequeue on a DB-backed queue — used for job queues, rate limiters, this exact pattern. No external lock manager needed. The DB is both the queue and the lock server.

---

### Pessimistic Row-Level Locking (concept used above)

**Pessimistic locking** = assume conflict will happen, lock the row upfront before anyone else touches it.

🎓 **Student**: Bathroom with one key — you grab it before entering, everyone waits outside until you're done.
🏗️ **Principal**: InnoDB acquires an exclusive row lock on first write, holds it until `COMMIT`/`ROLLBACK`. Contrast: **optimistic locking** skips the lock and does a version-check at write time, retrying on conflict. Optimistic wins when collisions are rare; pessimistic wins when correctness on every write is non-negotiable.

```sql
BEGIN;
  UPDATE inventory SET available_qty = 9 WHERE item_id = 'hoodie';
  -- ← exclusive row lock held here. T2 trying same UPDATE is BLOCKED.
COMMIT; -- lock released, T2 unblocks
```

---

### Does Every UPDATE Auto-Lock? Yes.

MySQL/PostgreSQL acquire an exclusive row lock on every `UPDATE`/`DELETE` — no `LOCK` keyword needed.

🎓 **Student**: UPDATE is secretly three steps: READ → COMPUTE → WRITE. Without a lock, two people read `10` simultaneously, both compute `9`, both write `9` — correct answer is `8`. One decrement vanished. Engine locks automatically to prevent this.
🏗️ **Principal**: Plain `SELECT` never blocks (MVCC serves a committed snapshot). Only writers contend. `SELECT ... FOR UPDATE` is for when you need the lock *before* writing — read-then-decide-then-write pattern.

```sql
-- Implicit (no keyword needed):
UPDATE inventory SET available_qty = available_qty - 1 WHERE item_id = 'hoodie';
-- ↑ exclusive lock auto-acquired, held until COMMIT

-- Explicit (lock now, write later):
SELECT available_qty FROM inventory WHERE item_id = 'hoodie' FOR UPDATE;
UPDATE inventory SET available_qty = 9 WHERE item_id = 'hoodie';
COMMIT;
```

| Query | Lock | Blocks writers? | Blocks plain SELECTs? |
|---|---|---|---|
| `SELECT` | None (MVCC) | No | No |
| `UPDATE` / `DELETE` | Exclusive (auto) | Yes | No |
| `SELECT ... FOR UPDATE` | Exclusive (explicit) | Yes | No |

---

### The New Reservation Flow

```
Pool (5 hoodies → 5 rows):
[row-1: available] [row-2: available] [row-3: available] [row-4: available] [row-5: available]

Buyer checks out:
  BEGIN
  SELECT id FROM reservation_pool WHERE item_id='hoodie'
    AND status='available' FOR UPDATE SKIP LOCKED LIMIT 1;
  → grabs row-1, writes a hold (buyer_id, expires_at)
  COMMIT

Payment succeeds:
  BEGIN
  UPDATE inventory SET qty = qty - 1 WHERE item_id = 'hoodie';  -- real inventory deducted
  DELETE hold;                                                    -- hold retired
  COMMIT  ← ONE transaction, ONE database ✅

Payment fails:
  DELETE hold → row back in pool ← also one transaction ✅
```

Both the inventory deduction and the hold retirement commit atomically. Dual-write problem is gone.

---

## Part 4 — The Row Explosion Problem

One row per unit sounds fine for 10 hoodies. At Shopify's scale:

| | |
|---|---|
| Merchants | Millions |
| Items per merchant | Thousands |
| Units per item | Tens of thousands |
| Total rows | Millions of millions of millions |

Worse: a merchant with 10,000 hoodies will never have 10,000 simultaneous checkouts. Almost all rows sit **idle** — still replicated, backed up, and paid for.

---

### Shopify's Fix: Cap the Pool at 1,000 Rows

No matter how much inventory a merchant has, the reservation pool for any item is **capped at 1,000 rows**. Real inventory count still lives in the inventory table. The pool just gets refilled as it drains.

- 1,000 = how many concurrent reservations the system can absorb at once
- Number chosen empirically: peak observed rate was a few hundred → add buffer → 1,000
- During a flash sale the pool drains to 0 → the next checkout request triggers a refill itself before retrying

---

### 🎓 Pool as a Buffer

Vending machine tray: warehouse has 50,000 items but the tray holds 20. When it empties, it gets refilled. No need for a tray with 50,000 slots.

### 🏗️ Capacity Planning

Pool size = peak concurrency + headroom (Little's Law: concurrency = arrival rate × hold time). This decouples storage cost from inventory scale. Refill-on-demand is a lazy inline refill — no separate background process needed for steady-state.

---

## Part 5 — Handling Expired Reservations in the New System

When replenishment runs, it answers one question:

```
reservable_now = unsold_inventory - live_unexpired_reservations
reservable_now = min(reservable_now, 1000)

rows_to_add = reservable_now - current_pool_size
INSERT that many rows
```

Expired reservations are **automatically excluded** from the formula — nothing has to go clean them up. When 50 reservations expire, the next replenishment sees 50 fewer live ones, inserts 50 more rows. Pool self-corrects.

Same inline trick as the Redis scenario: the buyer who needs inventory is the one who reclaims it, not an external cron.

---

### 🎓 Self-Healing Pool

Parking lot attendant who doesn't track which cars left. New car arrives, lot looks full — attendant just counts occupied spots right now and opens the difference. No chasing expired reservations.

### 🏗️ State Reconciliation vs Event-Driven Cleanup

Instead of reacting to individual expiry events (timers, sorted sets, cron), the system recomputes desired state from ground truth on the refill path. Fewer moving parts, handles clock skew and crash recovery naturally.

---

## Key Takeaways

1. **Atomicity requires co-location** — if two writes must be atomic, they must be in the same store. Dual-write between Redis and MySQL was the root cause, not throughput.
2. **MySQL/Postgres handle more than you think** — if Shopify can do this volume in MySQL, your app almost certainly can too.
3. **SKIP LOCKED is powerful** — turns a relational DB into a concurrent non-blocking queue. Available MySQL 8+ and PostgreSQL.
4. **Decouple pool size from inventory size** — size the pool to peak concurrency, not stock count.
5. **Revisit old decisions** — Shopify correctly ruled out MySQL when SKIP LOCKED didn't exist. The DB changed; the old decision deserved a second look.

---

## Architecture Comparison

| Dimension | Old (Redis + MySQL) | New (MySQL Only) |
|---|---|---|
| Reservation store | Redis | MySQL |
| Source of truth | MySQL | MySQL |
| Atomicity on success | ❌ Dual-write | ✅ Single transaction |
| Concurrency mechanism | Redis counter + Lua | `SELECT ... FOR UPDATE SKIP LOCKED` |
| Expiry handling | Sorted set + cron / inline Lua | Implicit via pool recomputation |
| Operational complexity | Two systems to sync | One system |
| Row count | 1 row per item | Capped at 1,000 per item |

---

*Source: Shopify Engineering Blog — Inventory Reservation System*
