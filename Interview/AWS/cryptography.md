# Security & Architecture Principles — AWS SysDE Interview Cheat Sheet

> Flagged twice on Glassdoor as a recurring surprise. Candidates over-index on system design diagrams and get caught on first principles. These are the fundamentals Amazon actually tests.

---

## 1. Hashing

### What it is

A **one-way mathematical function** that maps any input (any size) to a fixed-size output called a **digest** or **hash**.

```
Input: "hello"        → SHA-256 → a665a45920422f9d417e4867efdc4fb8a04a1f3fff1fa07e998e86f7f7a27ae3
Input: "hello!"       → SHA-256 → 4c1aa493d9cad48c3c5f7f5be5d571fe8a0f3ce9bc5ecc9c1c70a2a6c3c0e21a
                                   ↑ completely different — avalanche effect
```

You cannot reverse the digest to get "hello" back. That's the point.

### The 4 Core Properties

| Property | What it means | Why it matters |
|---|---|---|
| **One-way** | Cannot reverse digest → original input | Safe to store hashes of passwords |
| **Deterministic** | Same input always → same hash | Verification works reliably |
| **Avalanche effect** | Tiny input change → completely different hash | Tampered data is detectable |
| **Collision resistant** | Hard to find two inputs with same hash | Cannot fake a matching hash |

### Algorithms — Know Which to Use When

| Algorithm | Status | Use |
|---|---|---|
| MD5 | ❌ Broken (collisions found) | Legacy checksums only, never security |
| SHA-1 | ❌ Deprecated | Don't use |
| SHA-256 | ✅ Secure | Data integrity, digital signatures, TLS |
| SHA-3 | ✅ Secure | Modern alternative to SHA-2 |
| bcrypt | ✅ Password hashing | Slow by design + salting built in |
| Argon2 | ✅ Password hashing | Winner of Password Hashing Competition, preferred over bcrypt |

### Why bcrypt/Argon2 for Passwords, Not SHA-256?

SHA-256 is **too fast** for passwords. An attacker with a GPU can compute billions of SHA-256 hashes per second — brute-forcing a weak password takes milliseconds.

bcrypt/Argon2 are **intentionally slow** (cost factor configurable) and add a **salt** (random value mixed in before hashing):

```
bcrypt("password123", salt="x7kQ") → $2b$12$x7kQ...longhashedstring
bcrypt("password123", salt="m2pR") → $2b$12$m2pR...completelydifferent
```

Same password → different hashes per user. **Rainbow table attacks fail.**

### Use Cases

```
Password storage          → store bcrypt(password + salt), never plaintext
Data integrity (files)    → compare SHA-256 of downloaded file vs published hash
Digital signatures        → sign hash(message) with private key
Git object IDs            → SHA-1/SHA-256 of file content = content address
Message authentication    → HMAC (keyed hash, proves message wasn't tampered)
```

### The Hashing vs Encryption Distinction (Amazon loves this)

| | Hashing | Encryption |
|---|---|---|
| Reversible? | ✗ No | ✓ Yes (with key) |
| Purpose | Integrity / fingerprint | Confidentiality |
| Key required? | No | Yes |
| Example | Password storage | HTTPS traffic |

> **One-liner:** *"Hashing is one-way, deterministic, and used for integrity and password storage — not confidentiality. You cannot decrypt a hash."*

---

## 2. Symmetric vs. Asymmetric Cryptography

### The Core Problem Both Solve

You want to send a secret message over an untrusted network. How do you ensure only the intended recipient can read it?

### Symmetric Cryptography

**One key** — same key encrypts and decrypts.

```
Alice encrypts:  plaintext + KEY → ciphertext
Bob decrypts:    ciphertext + KEY → plaintext
                            ↑ same key
```

**The fundamental problem:** How does Alice get the key to Bob securely in the first place? If the network is untrusted, sending the key over it defeats the purpose. This is the **key distribution problem**.

| Aspect | Detail |
|---|---|
| Keys | 1 shared secret key |
| Speed | Fast — simple math (XOR, bit shifts) |
| Scales to large data? | Yes — bulk encryption |
| Main weakness | Key distribution problem |
| Algorithms | AES-128, AES-256, ChaCha20 |
| Use cases | Encrypting data at rest, disk encryption, session data |

**AES-256 in practice:**
```
Encrypt a file:   file + AES_KEY → encrypted_file
Decrypt a file:   encrypted_file + AES_KEY → file

S3 server-side encryption = AES-256 under the hood
Full disk encryption (FileVault, BitLocker) = AES-256
```

### Asymmetric Cryptography

**Two mathematically linked keys** — what one key encrypts, only the other can decrypt.

```
Key pair:
  Public key  → share with everyone, post it publicly
  Private key → never leaves your machine

Alice wants to send Bob a secret:
  1. Alice gets Bob's PUBLIC key (it's public, anyone can have it)
  2. Alice encrypts with Bob's PUBLIC key → ciphertext
  3. Only Bob's PRIVATE key can decrypt → Bob reads it
```

The magic: knowing the public key doesn't let you derive the private key (mathematically hard — factoring huge primes, discrete logarithm).

| Aspect | Detail |
|---|---|
| Keys | Key pair: public + private |
| Speed | Slow — heavy math (modular exponentiation) |
| Scales to large data? | No — expensive, used for small payloads |
| Main strength | Solves key distribution problem |
| Algorithms | RSA, ECC (Elliptic Curve), Diffie-Hellman |
| Use cases | TLS handshake, digital signatures, key exchange |

### Digital Signatures — Asymmetric in Reverse

For signatures, the private key **signs**, and the public key **verifies**:

```
Alice signs a message:
  1. hash(message) → digest
  2. encrypt(digest, Alice's PRIVATE key) → signature

Bob verifies:
  1. decrypt(signature, Alice's PUBLIC key) → digest
  2. hash(message) → compute digest independently
  3. compare — if equal, message is from Alice and unmodified
```

Only Alice has her private key → only Alice could have produced that signature. This proves **authenticity** and **integrity**.

### How TLS Uses Both (The Hybrid Approach)

This is the single most important thing to understand — TLS/HTTPS uses **both** symmetric and asymmetric, exploiting the strengths of each:

```
Phase 1 — Handshake (Asymmetric):
  Client → Server: "Hello, here are the cipher suites I support"
  Server → Client: Certificate (contains server's PUBLIC key)
  Client: generates a random session key
  Client → Server: session key encrypted with server's PUBLIC key
  Server: decrypts with its PRIVATE key → now both have the session key

Phase 2 — Data transfer (Symmetric):
  All subsequent traffic encrypted with AES using that session key
  Fast, efficient, handles gigabytes of data
```

Why not just use asymmetric for everything? **Too slow.** RSA on bulk data is ~1000x slower than AES. The handshake negotiates a key; AES does the heavy lifting.

```
Asymmetric → solves key distribution (safely exchange session key)
Symmetric  → solves performance (encrypt actual data fast)
TLS        → uses both
```

### Side-by-side Comparison

| | Symmetric | Asymmetric |
|---|---|---|
| Keys | 1 shared secret | Key pair: public + private |
| Speed | Fast | Slow |
| Key distribution | Problem | Solved |
| Bulk data? | Yes | No |
| Examples | AES-256, ChaCha20 | RSA, ECC, Diffie-Hellman |
| Used for | Data at rest, session data | Key exchange, digital signatures, TLS handshake |

> **One-liner:** *"Symmetric is fast but has the key distribution problem. Asymmetric solves key exchange but is too slow for bulk data. TLS combines both — asymmetric to negotiate a session key, symmetric to encrypt everything after."*

---

## 3. Three-Tier Architecture

### The Core Idea

Separate concerns into **three independent layers**, each with its own responsibility, security boundary, and scalability axis.

```
┌─────────────────────────────────┐
│      PRESENTATION TIER          │  ← User faces this
│  Browser, Mobile App, CLI       │
└────────────────┬────────────────┘
                 │ HTTP/HTTPS only
                 ▼
┌─────────────────────────────────┐
│      APPLICATION TIER           │  ← Business logic lives here
│  REST API, Go microservices,    │
│  Auth, Validation, Orchestration│
└────────────────┬────────────────┘
                 │ Internal network only
                 ▼
┌─────────────────────────────────┐
│         DATA TIER               │  ← Never exposed to clients
│  Postgres, Valkey, S3, Kafka    │
└─────────────────────────────────┘
```

### Each Tier's Responsibility

**Presentation Tier**
- Renders UI, captures user input
- Knows nothing about how data is stored
- Communicates with application tier only via APIs
- Examples: React SPA, iOS app, curl/CLI client

**Application Tier**
- Enforces all business rules and validation
- Authenticates and authorizes requests
- Orchestrates calls to data tier
- The only tier allowed to touch the database
- Examples: REST service, your Go JMS/JES/SAS microservices

**Data Tier**
- Stores and retrieves data
- Enforces data integrity (constraints, transactions)
- Never receives requests directly from clients
- Examples: Postgres, Valkey (Redis), S3, Kafka

### Why Separation Matters — Security Lens

**Without three-tier (bad):**
```
Browser → directly queries database
         ↑ SQL injection can destroy everything
         ↑ No auth layer
         ↑ DB credentials exposed
```

**With three-tier (good):**
```
Browser → App tier (validates, authenticates, authorizes) → DB
         ↑ App tier is the only one with DB credentials
         ↑ App tier rejects malformed input before it touches DB
         ↑ Even if presentation tier is compromised, DB is unreachable
```

**Security boundaries per tier:**

| Tier | Who can reach it | Auth mechanism |
|---|---|---|
| Presentation | Public internet | HTTPS, no auth |
| Application | Presentation tier only | JWT/OAuth tokens, API keys |
| Data | Application tier only | DB credentials, IAM roles, VPC rules |

**Principle of Least Privilege:** Each tier only has the permissions it needs. The app tier has no admin DB access. The presentation tier has no DB access at all.

### Why Separation Matters — Scalability Lens

Each tier scales **independently**:

```
High traffic → scale presentation tier (more CDN nodes, static hosting)
Heavy compute → scale application tier (more API pods on k3s)
Large data    → scale data tier (read replicas, sharding)
```

Contrast with a monolith: you scale the whole thing even if only one part is the bottleneck.

### Your System as an Example

```
Polaris/OME.Next maps to three-tier:

Presentation:  Dell management UI / API clients calling Job endpoints
Application:   JMS (Job Management Service) + JES (Job Execution Service)
               + SAS (Status Aggregation Service) — Go microservices on k3s
Data:          Postgres (job state), Valkey (cache/coordination), Kafka (event bus)
```

If an interviewer asks for a concrete example — that's yours.

### Common Interview Follow-ups

**Q: What's the difference between two-tier and three-tier?**
Two-tier = client talks directly to DB (client-server). No separate logic layer. Fine for small internal tools, breaks at scale and fails security requirements.

**Q: Can tiers be on the same machine?**
Yes — tier is a logical separation, not a physical one. But in production you separate them for security isolation and independent scaling.

**Q: What sits between tiers in practice?**
Load balancers, API gateways, firewalls, VPC subnets, security groups — all enforce the tier boundaries at the network level.

> **One-liner:** *"Three-tier separates presentation, logic, and data into independent layers — each with its own security boundary, scaling axis, and responsibility. The app tier is the only gatekeeper to the data tier."*

---

## 4. Synchronous vs. Asynchronous Communication

### The Core Distinction

**Synchronous:** Caller sends a request and **blocks** — it waits, doing nothing, until it gets a response.

**Asynchronous:** Caller sends a request and **continues** — it doesn't wait. The response (if any) arrives later via callback, event, or polling.

```
Synchronous:
  Caller ──── request ────► Service
  Caller ◄─── response ─── Service
         (caller blocked the whole time)

Asynchronous:
  Caller ──── message ────► Queue ──► Service (processes later)
  Caller continues immediately ↑
                      (decoupled in time)
```

### Deep Dive — Synchronous

**How it works:**
```
1. Caller makes HTTP request to Service B
2. Caller's thread is blocked — cannot do anything else
3. Service B processes the request (100ms, 2s, 5s — caller just waits)
4. Service B returns response
5. Caller unblocks, reads response, continues
```

**Implications:**

| Aspect | Impact |
|---|---|
| Latency | Caller's total latency = sum of all downstream call latencies |
| Coupling | Caller and callee must both be up at the same time |
| Failure | If Service B is down → caller fails immediately |
| Timeout | If Service B is slow → caller blocks until timeout |
| Simplicity | Easy to reason about — request/response is intuitive |
| Debugging | Simple — trace follows the call chain |

**When to use:**
- User needs the result **right now** to continue (login, checkout price calculation)
- Simple request/response where caller acts on the result immediately
- Low latency requirement with reliable downstream services

**Examples:** REST HTTP, gRPC, database queries, function calls

### Deep Dive — Asynchronous

**How it works:**
```
1. Caller publishes a message to a queue/topic (Kafka, SQS, SNS)
2. Caller returns immediately — not blocked
3. Consumer (separate service) picks up message from queue when ready
4. Consumer processes it independently
5. Caller may learn of result via: callback, another event, polling, or not at all
```

**Implications:**

| Aspect | Impact |
|---|---|
| Latency | Caller's latency = only time to enqueue (microseconds) |
| Coupling | Caller and callee don't need to be up simultaneously |
| Failure | Queue buffers messages — consumer can be down and catch up |
| Backpressure | Queue absorbs spikes — consumers process at their own pace |
| Complexity | Harder to debug — no linear trace, eventual consistency |
| Ordering | Not guaranteed without explicit mechanisms (Kafka partition key) |

**When to use:**
- Caller doesn't need the result immediately (email sending, report generation)
- High-throughput pipelines where producers are faster than consumers
- Decoupling services so they can evolve and fail independently
- Absorbing traffic spikes (queue buffers, consumers drain at steady rate)

**Examples:** Kafka, SQS, SNS, RabbitMQ, event-driven architectures

### Failure Mode Comparison — Critical for SysDE

```
Synchronous failure cascade:
  A → B → C → D
  D goes down → C times out → B times out → A fails
  One downstream failure can take down the whole chain

Asynchronous failure isolation:
  A → Queue → B
  B goes down → messages accumulate in queue
  B comes back up → processes backlog
  A never knew B was down
```

This is why async is preferred for **resilience** in distributed systems — it's a core SysDE topic.

### Your System as an Example

```
JMS → Kafka → JES is async:

JMS publishes a job event to Kafka and returns immediately.
JES subscribes to the Kafka topic and processes jobs independently.
If JES is down: Kafka retains the messages (retention policy).
If JES is slow: Kafka backlog grows, JMS is unaffected.
SAS reads job status events from Kafka asynchronously.

Contrast: if JES exposed a REST endpoint and JMS called it synchronously,
JES downtime = JMS downtime. Kafka breaks that coupling.
```

### Side-by-side

| | Synchronous | Asynchronous |
|---|---|---|
| Caller behavior | Blocks, waits | Fires and continues |
| Coupling | Tight | Loose |
| Latency contribution | Adds to caller's path | Out of critical path |
| Failure propagation | Cascades upstream | Isolated by queue |
| Consistency | Immediate | Eventual |
| Debugging | Simple (linear trace) | Hard (distributed trace) |
| Examples | REST, gRPC, DB query | Kafka, SQS, SNS, webhooks |
| Best for | Real-time results needed | Decoupling, throughput, resilience |

> **One-liner:** *"Sync blocks the caller and tightly couples services — simple but fragile. Async decouples services via a queue — resilient and scalable but eventual consistency and harder debugging. In distributed systems, async is preferred for resilience; sync for simplicity where real-time results are needed."*

---

## Quick Reference — All Four Topics

```
HASHING
  One-way, deterministic, fixed-size digest
  SHA-256 for integrity, bcrypt/Argon2 for passwords
  Cannot decrypt a hash — it's not encryption

SYMMETRIC CRYPTOGRAPHY
  One shared key, fast, bulk data
  Problem: key distribution
  AES-256 — data at rest, session data

ASYMMETRIC CRYPTOGRAPHY
  Key pair (public encrypts / private decrypts), slow
  Solves key distribution
  RSA, ECC — TLS handshake, digital signatures

TLS = ASYMMETRIC handshake + SYMMETRIC data transfer

THREE-TIER ARCHITECTURE
  Presentation → Application → Data
  Each tier: own security boundary, own scaling axis
  App tier is the only gatekeeper to the data tier
  Principle of least privilege per tier

SYNCHRONOUS vs ASYNCHRONOUS
  Sync: blocks caller, tight coupling, cascading failures
  Async: non-blocking, loose coupling, queue buffers failures
  Async preferred for resilience; sync for simplicity
  Your system: JMS → Kafka → JES = async
```

---

## The Bar-Raiser Test

Amazon doesn't just want *what* — they want *why*.

| Question | Weak answer | Strong answer |
|---|---|---|
| Why not store passwords as SHA-256? | "It's not safe" | "SHA-256 is too fast — GPUs can brute-force billions/sec. bcrypt adds cost factor and salt, making brute force impractical." |
| Why does TLS use both symmetric and asymmetric? | "For security" | "Asymmetric solves key distribution but is too slow for bulk data. Symmetric is fast but can't bootstrap securely. TLS uses asymmetric only for the handshake, then switches to AES for everything else." |
| Why three tiers instead of two? | "It's cleaner" | "Security boundary isolation — clients never reach the DB. Independent scaling axes. Single responsibility per tier. Principle of least privilege." |
| Why Kafka instead of REST between JMS and JES? | "It's more scalable" | "Async decoupling — JES downtime doesn't cascade to JMS. Kafka buffers traffic spikes. Consumers drain at their own pace. REST would tightly couple availability of both services." |
