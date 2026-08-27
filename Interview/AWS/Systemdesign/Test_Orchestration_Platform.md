# System Design: Test Orchestration Platform
> Senior / Principal Engineer Level — Modeled on the Ticket Master framework (Requirements → Entities → API → HLD → Deep Dives)

---

## What Is a Test Orchestration System?

A **Test Orchestration System** is a platform that:
- Accepts test suites submitted by developers or CI/CD pipelines
- Schedules and distributes individual test cases across a pool of workers
- Executes tests in parallel, tracks their state, collects results
- Aggregates results, generates reports, and notifies stakeholders

Real-world examples: **CircleCI, GitHub Actions, Jenkins, BuildKite, AWS CodeBuild, Google Cloud Build, Meta's internal Sandcastle**.

This question is especially relevant at infrastructure-heavy companies (AWS, Meta, Google) and is often framed as: *"Design a distributed CI/CD test runner"* or *"Design a system like GitHub Actions."*

---

## Interview Roadmap

| Phase | Time (approx) | Goal |
|---|---|---|
| Requirements | 5–8 min | Align on scope; functional + non-functional |
| Core Entities | 2–3 min | Identify data model |
| APIs | 3–5 min | Map endpoints to requirements |
| High-Level Design | 15 min | Simple design satisfying all functional requirements |
| Deep Dives | 15 min | Go deep on 2–3 non-functional areas |

---

## 1. Requirements

### Functional Requirements

Walk through the user flow — who are the users and what do they need to do?

| # | Requirement |
|---|---|
| 1 | Users (or CI/CD pipelines) should be able to **submit a test suite** (a collection of test cases) |
| 2 | The system should **schedule and distribute** individual tests across available workers |
| 3 | Users should be able to **view real-time status** of a running test suite (which tests passed, failed, are running, pending) |
| 4 | Users should be able to **retrieve the final test report** with results, logs, timing, and pass/fail summary |
| 5 | The system should **retry failed tests** (configurable — e.g., flaky test detection) |

**Out of scope (below the line):** Building/compiling code, managing source control, artifact storage, billing, RBAC, multi-tenancy security isolation (acknowledge, don't design).

> **Tip:** Clarify with the interviewer — is this a general-purpose test runner (unit/integration/e2e)? Assume yes unless told otherwise. Knowing the test types affects worker isolation requirements.

---

### Non-Functional Requirements

> Frame each one *specifically* to this system — not generic buzzwords.

| Requirement | Why It Matters Here |
|---|---|
| **Scalability for bursty workloads** | CI pipelines don't run uniformly — there are spikes at business hours, before release cutoffs, when many PRs merge simultaneously. The system must elastically scale worker capacity. |
| **High Availability for job submission** | If engineers can't submit tests, development halts. Submission path must be highly available. |
| **Eventual consistency for status/results** | It's okay if a test status takes a second or two to reflect in the dashboard — this is not financial data. Availability over strict consistency here. |
| **Strong consistency for final results** | A test run must eventually report a definitive, accurate pass/fail. Partial or incorrect final reports are unacceptable. |
| **Low latency task dispatch** | A test sitting in a queue for minutes when workers are available is a broken experience. Dispatch latency should be sub-second. |
| **Fault tolerance — worker failures** | Workers (containers/VMs) will crash mid-test. The system must detect this and requeue the test without data loss. |
| **Idempotency** | Retried dispatches must not cause a test to run twice and report double results. |
| **Observability** | The system itself must be monitorable — queue depths, worker utilization, test throughput, failure rates. |

> **Senior/Principal signal:** Distinguishing where you need strong consistency (final report) vs. where eventual consistency is fine (live status dashboard) — just like the Ticket Master booking vs. search split.

---

## 2. Core Entities

### Entity Relationship Overview

```
Pipeline / CI System
        │
        │ triggers
        ▼
┌─────────────────────────────────┐
│           TestSuite              │  ← root aggregate; one per CI run
│  PK: id                          │
│      pipeline_id  (external ref) │
│      commit_sha                  │
│      repo                        │
│      config (JSONB)              │
│      status                      │
│      submitted_at                │
│      completed_at                │
└──────────────┬──────────────────┘
               │ 1
               │ one suite has many test cases
               │ N
               ▼
┌─────────────────────────────────┐
│            TestCase              │  ← one atomic unit of work
│  PK: id                          │
│  FK: suite_id → TestSuite.id     │  ← which suite it belongs to
│  FK: assigned_worker_id          │  ← which Worker is running it
│         → Worker.id (nullable)   │    NULL when pending
│      name                        │
│      file_path                   │
│      status                      │
│        (pending/running/         │
│         passed/failed/retrying)  │
│      retry_count                 │
└──────────────┬──────────────────┘
               │ 1
               │ one test case has one result per attempt
               │ N  (N is usually 1; >1 only on retries)
               ▼
┌─────────────────────────────────┐
│           TestResult             │  ← immutable record of one execution attempt
│  PK: id                          │
│  FK: test_case_id → TestCase.id  │  ← which test case was run
│  FK: worker_id → Worker.id       │  ← which worker ran it
│      task_attempt_id (UUID)      │  ← idempotency key; unique per enqueue
│      exit_code                   │
│      stdout (TEXT)               │
│      stderr (TEXT)               │
│      duration_ms                 │
│      started_at                  │
│      finished_at                 │
│  UNIQUE (test_case_id,           │  ← prevents double-write on retry
│           task_attempt_id)       │
└─────────────────────────────────┘

┌─────────────────────────────────┐
│             Worker               │  ← compute node; independent lifecycle
│  PK: id                          │
│      status (idle/busy/dead)     │
│      capacity                    │
│      tags  (string[])            │  ← e.g. ["gpu","linux"] for routing
│      last_heartbeat_at           │  ← used by Worker Monitor to detect death
└─────────────────────────────────┘
       ▲                  ▲
       │                  │
  referenced by      referenced by
  TestCase            TestResult
  .assigned_worker_id .worker_id

┌─────────────────────────────────┐
│             Report               │  ← generated once all TestCases settle
│  PK: id                          │
│  FK: suite_id → TestSuite.id     │  ← one report per suite (1:1)
│      total                       │
│      passed                      │
│      failed                      │
│      skipped                     │
│      duration_ms                 │
│      created_at                  │
└─────────────────────────────────┘
```

### Relationship Summary

| From | Field | Points To | Cardinality | Notes |
|---|---|---|---|---|
| `TestCase` | `suite_id` | `TestSuite.id` | N:1 | Many test cases belong to one suite |
| `TestCase` | `assigned_worker_id` | `Worker.id` | N:1 (nullable) | NULL while pending; set when worker picks up the task |
| `TestResult` | `test_case_id` | `TestCase.id` | N:1 | Usually 1 result per test case; more if retried |
| `TestResult` | `worker_id` | `Worker.id` | N:1 | Records which worker produced this result |
| `Report` | `suite_id` | `TestSuite.id` | 1:1 | One final report per suite, generated on completion |

### Why Worker Is a Standalone Entity (Not Embedded)

Worker has its **own independent lifecycle** — it exists before any suite is submitted and lives across many suite runs. It is not owned by a TestCase; a TestCase merely *references* which Worker is currently assigned to it. This also means:
- A single Worker can appear in thousands of `TestResult` rows across its lifetime
- When a worker dies, you query `TestCase WHERE assigned_worker_id = :dead_worker_id AND status = 'running'` to find orphaned tasks — the FK makes this efficient
- Worker health (heartbeat, status) is managed separately from test execution state

---

### Table & Column Details

---

#### `test_suites`

| Column | Type | Constraints |
|---|---|---|
| `id` | `UUID` | PK |
| `pipeline_id` | `VARCHAR(255)` | NOT NULL |
| `commit_sha` | `VARCHAR(40)` | NOT NULL |
| `repo` | `VARCHAR(512)` | NOT NULL |
| `config` | `JSONB` | NOT NULL |
| `status` | `VARCHAR(20)` | NOT NULL, DEFAULT `'pending'` |
| `submitted_at` | `TIMESTAMPTZ` | NOT NULL, DEFAULT NOW() |
| `completed_at` | `TIMESTAMPTZ` | NULLABLE |

**Why each column exists:**

**`id`** — The primary key and the stable identifier for this suite across every service. Every downstream record (TestCase, Report) references this ID. UUID is used instead of a sequential integer to avoid enumeration attacks and to make IDs safe to expose externally in API responses.

**`pipeline_id`** — This is the external system's identifier for the CI run that triggered this suite — e.g. a GitHub Actions `run_id` or a Jenkins build number. It is stored so the orchestrator can correlate a suite back to the originating pipeline. If the same pipeline retriggers a run, you can query by `pipeline_id` to find all historical suite runs for that pipeline without needing to know the internal `id` upfront.

**`commit_sha`** — The exact Git commit hash (40-character SHA-1) of the code under test. This is critical for traceability: if a test fails, developers need to know *which exact version of the code* was being tested. It also enables queries like "show me all suite runs for this commit" — useful for flaky test analysis across multiple attempts at the same commit.

**`repo`** — The repository that owns these tests, stored as `org/repo-name` (e.g. `acme/backend-api`). Needed for multi-repo platforms so you can filter, group, and report results per repository. Without this, all suites from all repos would be indistinguishable.

**`config`** — Stores the full test configuration as a JSONB blob: parallelism level, retry limits, timeout per test, worker tags required, and the list of test files. It is JSONB rather than normalized columns because configuration shape varies wildly between suite types — a unit test suite, a GPU benchmark suite, and an e2e browser test suite all have completely different parameters. Storing it as JSONB means the schema doesn't need to change as new config options are added. It is queried rarely (only when the Orchestrator reads it at fan-out time), so the flexibility outweighs the cost of not having typed columns.

**`status`** — The current lifecycle state of the entire suite. Drives the UI (show a spinner vs. a green checkmark vs. a red X) and controls business logic — the Report Service only generates a report once status is `completed` or `failed`. Valid transitions: `pending → running → completed | failed | cancelled`. Having this at the suite level means you can answer "is this suite done?" in O(1) without counting all its test cases.

**`submitted_at`** — Timestamp of when the suite arrived at the Orchestrator. Used for SLA tracking ("how long did it take from submission to completion?"), queue time analysis, and audit logs. Defaults to `NOW()` so the application doesn't need to pass it explicitly.

**`completed_at`** — NULL while the suite is in progress; set to the timestamp when the last test case reaches a terminal state. The delta between `submitted_at` and `completed_at` is the total end-to-end suite duration — a key operational metric. Being nullable is intentional: a non-null value signals definitively that the suite is done, which is safer than relying solely on `status`.

---

#### `test_cases`

| Column | Type | Constraints |
|---|---|---|
| `id` | `UUID` | PK |
| `suite_id` | `UUID` | FK → `test_suites.id`, NOT NULL |
| `name` | `VARCHAR(512)` | NOT NULL |
| `file_path` | `VARCHAR(1024)` | NOT NULL |
| `status` | `VARCHAR(20)` | NOT NULL, DEFAULT `'pending'` |
| `retry_count` | `INT` | NOT NULL, DEFAULT `0` |
| `assigned_worker_id` | `UUID` | FK → `workers.id`, NULLABLE |
| `created_at` | `TIMESTAMPTZ` | NOT NULL, DEFAULT NOW() |

**Why each column exists:**

**`id`** — Primary key and the unit of work identity. When the Orchestrator fans out a suite into individual tasks, each task message on the queue carries this `id`. Workers and the Result Collector reference it to know which test case a result belongs to.

**`suite_id`** — Foreign key back to `test_suites.id`. This is the join column that ties a test case to its parent suite. Every time you ask "show me all tests for suite X" — the dashboard, the progress counter, the report generator — this FK is what makes that query possible. Indexed for exactly this reason.

**`name`** — The human-readable identifier of the test, e.g. `AuthServiceTest::testLoginWithInvalidPassword`. Displayed in the UI, in failure notifications, and in the report. Without a name, the test case is just a UUID — meaningless to a developer debugging a failure.

**`file_path`** — The relative path to the test file in the repo, e.g. `src/test/java/com/acme/AuthServiceTest.java`. The worker uses this to know *what to execute* — it checks out the repo at `commit_sha` and runs this specific file. It's also used in the report to group failures by file, and by flaky test detection systems to track which files are consistently problematic.

**`status`** — The current execution state of this individual test case. This is what drives the live dashboard seat-map equivalent — each test case's color (grey/yellow/green/red) on the status page. It also drives retry logic: the Result Collector checks this after a failure to decide whether to requeue. Valid transitions: `pending → running → passed | failed | skipped`, and `failed → retrying → running` on retry.

**`retry_count`** — Tracks how many execution attempts have been made for this test case so far. The Result Collector compares this against `config.retry_limit` from the parent suite to decide whether another retry is allowed. Without this column you'd have to count `test_results` rows per test case on every failure — an extra query on the hot path. Storing it here makes the retry decision O(1).

**`assigned_worker_id`** — Foreign key to `workers.id`, nullable. NULL means the test is pending (not yet picked up). Set to the worker's ID the moment a worker claims the task from the queue. This is the critical column for dead worker recovery: the Worker Monitor queries `WHERE assigned_worker_id = :dead_worker_id AND status = 'running'` to find all tasks that were orphaned when that worker died. Without this column, you'd have no way to find stranded tasks efficiently.

**`created_at`** — When this test case record was created, which happens during fan-out immediately after suite submission. Used for queue-time analysis: the difference between `created_at` and the `started_at` on its first `test_result` tells you how long the test waited in the queue before a worker picked it up.

> **Index on `suite_id`:** most common query — "give me all test cases for suite X."
> **Compound index on `(assigned_worker_id, status)`:** Worker Monitor dead-task recovery query.

---

#### `workers`

| Column | Type | Constraints |
|---|---|---|
| `id` | `UUID` | PK |
| `status` | `VARCHAR(10)` | NOT NULL, DEFAULT `'idle'` |
| `capacity` | `INT` | NOT NULL, DEFAULT `1` |
| `tags` | `VARCHAR[]` | NOT NULL, DEFAULT `'{}'` |
| `last_heartbeat_at` | `TIMESTAMPTZ` | NOT NULL |

**Why each column exists:**

**`id`** — Primary key assigned when the worker process registers on startup. Every test case and test result that passes through this worker records this ID, creating a full audit trail: "which worker ran this test, and was that worker healthy at the time?"

**`status`** — The current lifecycle state of the worker node: `idle` (available to pick up work), `busy` (currently executing one or more test cases), `dead` (heartbeat missed — Worker Monitor has declared it gone). The Orchestrator and Worker Monitor both read this to make decisions. The queue-based pull model means workers don't strictly need to be `idle` in the DB before picking up work — but this status is used for capacity planning dashboards and for knowing not to assign new tasks to a worker that's already dead.

**`capacity`** — The maximum number of test cases this worker can run concurrently. Defaults to `1` because test isolation is safest when one worker runs one test at a time (no shared state, no port conflicts). Higher values can be set for lightweight unit tests where full isolation isn't necessary. This is **self-regulated by the worker itself** — the worker reads its own capacity on startup and pulls that many messages from the queue simultaneously. The Orchestrator does not push tasks or track which worker is free; workers pull when ready. Capacity `4` means the worker holds 4 queue messages at once, executes them in parallel threads/processes, and ACKs each message only when that test completes.

**`tags`** — A string array of capability labels, e.g. `{gpu, linux, cuda-12, high-memory}`. At fan-out time, the Orchestrator reads `required_tags` from the suite's `config` and routes each test case's task message to the queue that matches workers bearing those tags. Without tags, every test would have to run on a generic worker — GPU tests would fail because no CUDA drivers are present, Windows tests would fail because the OS is wrong. Tags are the routing mechanism that makes multi-environment testing possible.

**`last_heartbeat_at`** — The timestamp of the most recent heartbeat signal received from this worker. Every 15–30 seconds, the worker sends a `POST /internal/workers/{id}/heartbeat` request, and this column is updated. The Worker Monitor runs on a schedule and queries for workers where `last_heartbeat_at < NOW() - INTERVAL '30 seconds'` — those workers are presumed dead. This is the entire mechanism behind fault detection. Without this column, there is no way to distinguish a healthy idle worker from a crashed one.

> **Index on `status`:** Worker Monitor scans for non-dead workers with stale heartbeats.
> **`tags` queried with `@>` operator:** `WHERE tags @> ARRAY['gpu']` — no join table needed.

---

#### `test_results`

| Column | Type | Constraints |
|---|---|---|
| `id` | `UUID` | PK |
| `test_case_id` | `UUID` | FK → `test_cases.id`, NOT NULL |
| `worker_id` | `UUID` | FK → `workers.id`, NOT NULL |
| `task_attempt_id` | `UUID` | NOT NULL |
| `exit_code` | `INT` | NOT NULL |
| `stdout` | `TEXT` | NULLABLE |
| `stderr` | `TEXT` | NULLABLE |
| `duration_ms` | `INT` | NOT NULL |
| `started_at` | `TIMESTAMPTZ` | NOT NULL |
| `finished_at` | `TIMESTAMPTZ` | NOT NULL |

**Unique constraint:** `UNIQUE(test_case_id, task_attempt_id)`

**Why each column exists:**

**`id`** — Primary key for the result record itself. Rarely queried directly; mostly used as an opaque identifier in API responses.

**`test_case_id`** — Foreign key to `test_cases.id`. This is the join column that answers "what was the outcome of test case X?" — the most common query the Report Service makes. It also ties results back to the suite (via `test_cases.suite_id`), enabling suite-level aggregation. Indexed because the Report Service reads all results for all test cases in a suite on every report generation.

**`worker_id`** — Foreign key to `workers.id`. Records *which worker* ran this particular attempt. This serves two purposes: (1) operational debugging — if a worker is consistently producing failures, you can detect it by grouping results by `worker_id` and looking at failure rates; (2) audit trail — in the event of a hardware fault or flaky environment, you can trace every result that worker ever produced and invalidate them if needed.

**`task_attempt_id`** — A UUID generated fresh each time a test case is enqueued (including retries). This is the idempotency key. Because the queue guarantees at-least-once delivery, the same task message can be delivered twice — causing the worker to run the test twice and the Result Collector to receive two result messages for the same attempt. The `UNIQUE(test_case_id, task_attempt_id)` constraint means the second `INSERT` is a no-op (`ON CONFLICT DO NOTHING`). Without this column, double delivery would inflate pass/fail counts and corrupt the report. Note: `task_attempt_id` is NOT the same as `test_case_id` — a retry gets a new `task_attempt_id` but the same `test_case_id`, so you can see all attempts for a given test case as separate rows.

**`exit_code`** — The OS-level exit code returned by the test process. `0` means the test passed. Any non-zero value means failure. This is the ground truth for pass/fail determination — not a string "PASSED"/"FAILED" field, because exit codes are universal across all test frameworks (JUnit, pytest, Go test, etc.). The Result Collector reads this to update the parent `test_case.status`.

**`stdout`** — The full standard output captured from the test process during execution. Contains test framework output — e.g. pytest's dot notation, JUnit's XML summary, timing lines. Developers read this when investigating what happened during a passing or failing run. Nullable because some tests produce no output.

**`stderr`** — The full standard error captured from the test process. This is where stack traces, assertion failure messages, and error logs appear on failure. In practice this is the most important field for debugging — when a test fails, `stderr` is the first thing a developer looks at. Nullable because a passing test typically produces no stderr.

**`duration_ms`** — Wall-clock time in milliseconds from process start to process exit, as measured by the worker. Used for performance tracking (slow test detection, p95 execution time dashboards), for suite total duration in the Report, and for identifying tests that are approaching timeout limits. Integer milliseconds is sufficient precision for test execution; nanoseconds would be overkill.

**`started_at`** — The timestamp when the worker actually began executing the test (after picking it up from the queue and doing any setup). The delta between `test_cases.created_at` and this field gives you the *queue wait time* — how long the test sat waiting before a worker picked it up. This is a key SLA metric for the platform.

**`finished_at`** — The timestamp when the worker received the process exit signal. `finished_at - started_at` gives the execution duration from the *server's perspective* (as a cross-check against `duration_ms`). Also used to compute the suite's total elapsed time: `MAX(finished_at) - MIN(started_at)` across all results for a suite.

> **Trade-off:** `stdout`/`stderr` as `TEXT` in Postgres works for typical test output (kilobytes). For verbose integration tests or load tests producing megabytes of logs, store logs in S3 and keep a `log_url VARCHAR` here instead. Worth flagging in the interview as a known scaling trade-off.

---

#### `reports`

| Column | Type | Constraints |
|---|---|---|
| `id` | `UUID` | PK |
| `suite_id` | `UUID` | FK → `test_suites.id`, NOT NULL, UNIQUE |
| `total` | `INT` | NOT NULL |
| `passed` | `INT` | NOT NULL |
| `failed` | `INT` | NOT NULL |
| `skipped` | `INT` | NOT NULL |
| `duration_ms` | `INT` | NOT NULL |
| `created_at` | `TIMESTAMPTZ` | NOT NULL, DEFAULT NOW() |

**Why each column exists:**

**`id`** — Primary key. Reports are fetched by `suite_id` in practice (via `GET /suites/{id}/report`), but the PK exists for standard relational integrity and for any internal tooling that references reports by their own ID.

**`suite_id`** — Foreign key to `test_suites.id`, with a `UNIQUE` constraint. This enforces the 1:1 relationship between a suite and its report at the database level — not just in application logic. If the Report Service fires twice due to a duplicate event (a real failure mode in event-driven systems), the second `INSERT` hits the unique constraint and fails safely. The `UNIQUE` constraint also means `GET /suites/{id}/report` can be answered with a direct indexed lookup — O(log n), not a scan.

**`total`** — The total count of test cases in the suite. Stored here so the report is self-contained — you don't need to re-query `test_cases` to render "47 tests ran." Also serves as a sanity check: `passed + failed + skipped` should equal `total`. If it doesn't, something went wrong in result collection.

**`passed`** — Count of test cases that finished with `exit_code = 0`. The headline number developers care about — "how many tests passed?" Stored as a pre-computed aggregate so `GET /report` is a single row read, not a `COUNT(*)` aggregation across potentially thousands of `test_results` rows on every request.

**`failed`** — Count of test cases that finished with a non-zero `exit_code` after exhausting all retries. The number that triggers a red build badge and blocks a PR merge. Stored separately from `passed` so you can immediately answer "did anything fail?" without arithmetic.

**`skipped`** — Count of test cases that were intentionally not executed — e.g. tests marked `@Ignore` or excluded by config. Stored because `total - passed - failed` without `skipped` would give a misleading picture ("where did the other 3 tests go?"). Skipped tests are not failures, but they need to be accounted for in the total.

**`duration_ms`** — Total elapsed time of the suite run in milliseconds, computed as `MAX(test_result.finished_at) - MIN(test_result.started_at)` across all results. This is not the sum of individual test durations (which would give you CPU time, not wall-clock time). Because tests run in parallel, the wall-clock duration is determined by the longest parallel path. Stored here so it's immediately available in the report without recomputing from `test_results`.

**`created_at`** — When this report was generated. Useful for knowing how stale the report is (though reports are immutable once written), and for audit logs showing when the suite was declared complete.

---

### Full Schema (DDL)

```sql
CREATE TABLE test_suites (
  id              UUID          PRIMARY KEY DEFAULT gen_random_uuid(),
  pipeline_id     VARCHAR(255)  NOT NULL,
  commit_sha      VARCHAR(40)   NOT NULL,
  repo            VARCHAR(512)  NOT NULL,
  config          JSONB         NOT NULL,
  status          VARCHAR(20)   NOT NULL DEFAULT 'pending',
  submitted_at    TIMESTAMPTZ   NOT NULL DEFAULT NOW(),
  completed_at    TIMESTAMPTZ
);

CREATE TABLE workers (
  id                  UUID          PRIMARY KEY DEFAULT gen_random_uuid(),
  status              VARCHAR(10)   NOT NULL DEFAULT 'idle',
  capacity            INT           NOT NULL DEFAULT 1,
  tags                VARCHAR[]     NOT NULL DEFAULT '{}',
  last_heartbeat_at   TIMESTAMPTZ   NOT NULL
);

CREATE INDEX idx_workers_status ON workers(status);

CREATE TABLE test_cases (
  id                  UUID          PRIMARY KEY DEFAULT gen_random_uuid(),
  suite_id            UUID          NOT NULL REFERENCES test_suites(id),
  name                VARCHAR(512)  NOT NULL,
  file_path           VARCHAR(1024) NOT NULL,
  status              VARCHAR(20)   NOT NULL DEFAULT 'pending',
  retry_count         INT           NOT NULL DEFAULT 0,
  assigned_worker_id  UUID          REFERENCES workers(id),
  created_at          TIMESTAMPTZ   NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_test_cases_suite_id       ON test_cases(suite_id);
CREATE INDEX idx_test_cases_worker_status  ON test_cases(assigned_worker_id, status);

CREATE TABLE test_results (
  id                UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
  test_case_id      UUID        NOT NULL REFERENCES test_cases(id),
  worker_id         UUID        NOT NULL REFERENCES workers(id),
  task_attempt_id   UUID        NOT NULL,
  exit_code         INT         NOT NULL,
  stdout            TEXT,
  stderr            TEXT,
  duration_ms       INT         NOT NULL,
  started_at        TIMESTAMPTZ NOT NULL,
  finished_at       TIMESTAMPTZ NOT NULL,
  UNIQUE (test_case_id, task_attempt_id)
);

CREATE INDEX idx_test_results_test_case_id ON test_results(test_case_id);

CREATE TABLE reports (
  id            UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
  suite_id      UUID        NOT NULL UNIQUE REFERENCES test_suites(id),
  total         INT         NOT NULL,
  passed        INT         NOT NULL,
  failed        INT         NOT NULL,
  skipped       INT         NOT NULL,
  duration_ms   INT         NOT NULL,
  created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

---

## 3. APIs

### Submit a Test Suite
```
POST /suites
Body: {
  repo: string,
  commit_sha: string,
  test_config: {
    test_files: string[],       // which test files to run
    parallelism: number,        // max concurrent workers
    retry_limit: number,        // per-test retry attempts
    timeout_seconds: number,
    worker_tags: string[]       // e.g. ["gpu", "linux"]
  }
}
Header: Authorization: Bearer <JWT>

Response: {
  suite_id: string,
  status: "pending",
  estimated_queue_time_seconds: number
}
```

### Get Suite Status (Real-Time)
```
GET /suites/{suiteId}/status

Response: {
  suite_id: string,
  status: "running" | "completed" | "failed",
  progress: {
    total: number,
    pending: number,
    running: number,
    passed: number,
    failed: number
  },
  test_cases: TestCase[]
}
```

> For real-time updates, this can also be served via **SSE** or **WebSocket** — see Deep Dive 2.

### Get Final Report
```
GET /suites/{suiteId}/report

Response: Report + TestResult[] (with logs, timing per test)
```

### Retry a Failed Test (Manual)
```
POST /suites/{suiteId}/tests/{testCaseId}/retry
Header: Authorization: Bearer <JWT>

Response: { test_case_id: string, status: "retrying" }
```

### Worker Heartbeat (Internal — Worker → Orchestrator)
```
POST /internal/workers/{workerId}/heartbeat
Body: { status: "idle" | "busy", current_test_id: string | null }
```

> Internal APIs are worth mentioning at senior/principal level. The heartbeat is how the system detects dead workers.

---

## 4. High-Level Design

### Architecture Overview

```
Client / CI Pipeline
        │
        ▼
┌──────────────────────────────┐
│         API Gateway           │  validates JWT, rate limits,
│                               │  routes to correct service
└──────────────┬────────────────┘
               │  (only auth + routing happen here)
               ▼
┌──────────────────────────────────────────────┐
│           Orchestrator Service                │
│                                               │
│  1. Accepts suite submission (POST /suites)   │
│  2. Writes TestSuite + TestCases to DB        │
│  3. Fans out: 1 task message per TestCase     │
│  4. Tracks suite + test state                 │
└──────────────────┬────────────────────────────┘
                   │ enqueue (1 msg per TestCase)
                   ▼
    ┌──────────────────────────────┐
    │      Task Queue (Kafka/SQS)   │
    │  [queue:linux] [queue:gpu]... │
    └──────┬──────────────┬─────────┘
           │              │  workers pull when ready (pull model)
           ▼              ▼
    ┌────────────┐  ┌────────────┐   ... (auto-scaled pool)
    │  Worker 1  │  │  Worker 2  │
    │ (container)│  │ (container)│
    │ capacity:N │  │ capacity:N │
    └─────┬──────┘  └─────┬──────┘
          │               │
          └───────┬────────┘
                  │ publish result
                  ▼
    ┌──────────────────────────┐
    │    Result Collector       │  idempotent write via
    │       Service             │  (test_case_id, task_attempt_id)
    └──────────────┬────────────┘
                   │ writes results + updates test_case status
                   ▼
            ┌─────────────┐
            │  PostgreSQL  │  (primary data store)
            └──────┬───────┘
                   │
        ┌──────────┴──────────┐
        │                     │
        ▼                     ▼
┌──────────────┐    ┌──────────────────────┐
│ Report Service│    │    Status Service     │
│ (generates    │    │ (serves live status) │
│  final report)│    │ pushes via SSE       │
└──────┬────────┘    └──────────┬───────────┘
       │                        │
       ▼                        ▼
  Redis (cache)          Kafka topic
  GET /report            suite.{id}.events
                              │
                              ▼
                        Client Dashboard (SSE)

┌──────────────────────────┐
│     Worker Monitor        │  heartbeat checker; marks dead
│  (background process)     │  workers, requeues orphaned tasks
└──────────────────────────┘
```

> **Important:** The API Gateway does **not** break suites into tasks, enqueue anything, or touch the database. It is a thin validation and routing layer only. All business logic — fan-out, state tracking, retry decisions — lives in the Orchestrator and downstream services.

---

### Walking Through Each Functional Requirement

#### FR1: Submit a Test Suite

1. Client sends `POST /suites` → API Gateway → **Orchestrator Service**
2. Orchestrator writes a `TestSuite` record to **PostgreSQL** with `status = pending`
3. Orchestrator **parses the test config** → creates one `TestCase` record per test file, all with `status = pending`
4. Orchestrator **publishes one task message per TestCase** to a **Task Queue** (Kafka topic or SQS queue)
5. Returns `suite_id` to client immediately — submission is asynchronous

> The suite is broken into individual tasks here. This is the key architectural decision: **fan-out at submission time**, not at execution time. Each TestCase is an independent, atomic unit of work.

#### FR2: Schedule and Distribute Tests

- Workers are long-running containers that **pull / consume** from the Task Queue — this is a **pull model**, not push
- Each worker pulls up to `capacity` messages at once (default 1, higher for lightweight tests)
- For each task, the worker updates the TestCase `status = running` and `assigned_worker_id = self`
- Worker executes the test in isolation (either the container itself is the isolation boundary, or the worker spawns a child process per test)
- Worker publishes the result to the **Result Collector Service** when done, then immediately pulls the next task — no idle waiting

**Worker Assignment — Why a Queue?**
- Queue provides natural load distribution — workers pull when they're ready (pull model, not push)
- No central scheduler needed to track worker availability in real time
- Queue acts as a buffer during traffic spikes — tasks accumulate, workers drain them at their own rate

#### FR3: Real-Time Status

- **Orchestrator Service** (or a dedicated **Status Service**) maintains current state of each TestCase in PostgreSQL
- As results come in, `status` is updated: `running → passed / failed`
- Suite `status` is updated once all TestCases complete
- Clients poll `GET /suites/{id}/status` or receive push updates via SSE (see Deep Dive 2)

#### FR4: Final Report

- **Report Service** aggregates all `TestResult` records for a suite
- Generates the `Report` entity (totals, pass/fail counts, duration)
- Caches the report in **Redis** (reports are immutable once generated — perfect cache candidate)
- Returns full report including per-test logs

#### FR5: Retry Failed Tests

- On test failure, the **Result Collector** checks `retry_count < retry_limit`
- If yes: re-enqueue the task to the Task Queue with `retry_count + 1`, set TestCase `status = retrying`
- Idempotency key on each task message prevents double-execution if a message is re-delivered

---

### Database Design

**Why PostgreSQL?**
- Strong consistency for test results and suite state
- ACID transactions for state transitions (e.g., marking a suite complete only after all tests finish)
- Relational model fits perfectly: Suite → TestCase → TestResult
- `JSONB` support for flexible test config storage

> Full schema with all constraints, indexes, and column types is defined in Section 2 (Table & Column Details → Full Schema DDL). The key tables are `test_suites`, `test_cases`, `test_results`, `workers`, and `reports`.

---

## 5. Deep Dives

### Deep Dive 1: Fault Tolerance — Dead Worker Detection and Task Recovery

**Problem:** A worker picks up a test task, starts executing, then crashes (OOM, hardware failure, network partition). The TestCase is now `status = running` forever with no result coming.

#### Approach: Heartbeat + Visibility Timeout

**Heartbeat mechanism:**
- Every worker sends `POST /internal/workers/{id}/heartbeat` every 15–30 seconds
- Orchestrator tracks `last_heartbeat_at` per worker
- A **Worker Monitor** (background process) runs every N seconds, queries for workers where `last_heartbeat_at < NOW() - threshold`
- Those workers are marked `status = dead`

**Task recovery:**
- Queue-side: Use **message visibility timeout** (SQS) or **consumer group offset management** (Kafka)
  - SQS: If a worker doesn't ACK the message within the visibility timeout, SQS makes it visible again for another worker to pick up
  - Kafka: If the consumer crashes before committing the offset, another consumer in the group re-reads from the last committed offset
- DB-side: Worker Monitor also queries for TestCases with `status = running` and `assigned_worker_id` pointing to a dead worker → resets them to `status = pending` and re-enqueues

**Idempotency to prevent double results:**
- Each task message carries a `task_attempt_id` (UUID)
- Result Collector checks: if a result for this `test_case_id` + `task_attempt_id` already exists → discard (exactly-once semantics)

> **Principal-level nuance:** SQS gives you this for free (at-least-once delivery + visibility timeout). Kafka requires more careful offset commit management — commit only *after* writing the result to the DB, not before executing the test.

---

### Deep Dive 2: Real-Time Test Status — Scaling the Live Dashboard

**Problem:** During a large test suite run (10,000 test cases), developers watch the dashboard. Every test completion triggers a status update. How do you push these updates efficiently without hammering the DB or flooding clients?

#### Option 1 — Polling
- Client polls `GET /suites/{id}/status` every 2–5 seconds
- Simple, no persistent connection
- **Downside:** Thundering herd if 1,000 engineers are watching the same suite (monorepo release); DB takes a beating

#### Option 2 — Server-Sent Events (SSE) ✅ Recommended
- When a client opens the status page, it establishes an SSE connection to the **Status Service**
- The Status Service subscribes to a **Kafka topic** (or Redis Pub/Sub channel) keyed by `suite_id`
- When a TestCase result is written, the Result Collector publishes a status-change event to that topic
- Status Service pushes the delta (not the full state) to all subscribed clients
- Unidirectional (server → client only) — perfect fit; no bidirectional communication needed

```
Result Collector ──publish──► Kafka topic: suite.{suiteId}.events
                                      │
                               Status Service (subscriber)
                                      │
                         SSE push ──► Client Dashboard
```

**Why not WebSockets?**
- Bidirectional — unnecessary here. SSE is lighter, works over plain HTTP/2, and doesn't require special infrastructure.

**Scaling SSE connections:**
- SSE connections are long-lived and stateful — a single Status Service instance has connection limits
- Solution: horizontal scale the Status Service + use **sticky sessions** (load balancer routes a client to the same instance for the life of the connection) OR use a **fan-out pub/sub layer** (Redis Pub/Sub or Kafka consumer groups) so any instance can serve the event for any suite

---

### Deep Dive 3: Scalability — Handling Traffic Spikes (Bursty Workloads)

**Problem:** At 9 AM Monday, 500 teams merge PRs simultaneously. 50,000 test cases are enqueued in 30 seconds. Workers are overwhelmed. Queue depth skyrockets.

#### The Queue as a Buffer

The Task Queue (Kafka/SQS) inherently absorbs the spike. Tasks accumulate; workers drain at their pace. This is the right answer for the *queue* side — it handles spikes gracefully. The real question is: how do you scale the *workers*?

#### Auto-Scaling the Worker Pool

- Workers run as **containers on Kubernetes** (or ECS/EKS on AWS)
- Use **Horizontal Pod Autoscaler (HPA)** or a custom autoscaler
- Scaling metric: **Task Queue depth** (not CPU/memory — the queue depth is the true signal of demand)
  - SQS: `ApproximateNumberOfMessagesVisible`
  - Kafka: Consumer group lag
- Scale up aggressively when queue depth crosses a threshold; scale down slowly (drain in-flight tasks before terminating workers)

#### Worker Tagging and Routing

Different tests have different resource requirements:
- GPU tests → workers with GPU nodes
- Windows tests → Windows worker pool
- Heavy integration tests → high-memory workers

**Implementation:**
- TestCase has `required_tags: string[]`
- Task Queue has **separate queues (or Kafka partitions/topics) per worker type**
- Workers subscribe only to their relevant queue
- Orchestrator routes tasks to the appropriate queue at fan-out time

#### Preventing Worker Starvation

**Problem:** A large suite with 10,000 tests from one team monopolizes all workers, starving other teams.

**Solution: Fair scheduling**
- Use **weighted round-robin** across multiple queues — one queue per team/project, Orchestrator round-robins across them
- Or: implement **priority levels** — P0 (release branch) gets dedicated capacity, P1/P2 share the rest
- OR (simplest): **concurrency limits per suite** — configured at submission time via `parallelism` in the test config

---

### Deep Dive 4: Idempotency and Exactly-Once Test Execution

**Problem:** The queue delivers a task message twice (network retry, rebalance). The test runs twice. The result is reported twice. The suite shows 2x passed counts.

#### Solution: Idempotency Key Per Task Attempt

1. Each task message contains:
   - `test_case_id` (stable, identifies the logical test)
   - `task_attempt_id` (UUID, unique per enqueue attempt)

2. When the Result Collector receives a result:
   ```sql
   INSERT INTO test_results (test_case_id, task_attempt_id, ...)
   ON CONFLICT (test_case_id, task_attempt_id) DO NOTHING;
   ```
   Unique constraint on `(test_case_id, task_attempt_id)` — second write is a no-op.

3. Suite progress counters are updated **transactionally** — not via application-level increment but via:
   ```sql
   UPDATE test_suites
   SET passed_count = passed_count + 1
   WHERE id = :suite_id
   -- wrapped in a transaction that checks the test_result insert succeeded
   ```

4. Suite completion check: rather than trusting a counter, the Orchestrator queries:
   ```sql
   SELECT COUNT(*) FROM test_cases
   WHERE suite_id = :id AND status NOT IN ('passed', 'failed', 'skipped')
   ```
   Suite is marked `completed` only when this returns 0.

---

### Deep Dive 5: Observability (Principal-Level Signal)

A well-designed test orchestration system is useless if you can't see what it's doing internally.

**Key metrics to instrument:**

| Metric | Why |
|---|---|
| Queue depth per worker pool | Primary scaling signal; also indicates bottlenecks |
| Worker utilization (busy / total) | Detects over/under-provisioning |
| Task dispatch latency (enqueue → worker pickup) | SLA for how fast tests start |
| Test execution duration distribution (p50/p95/p99) | Detect slow tests, runaway tests |
| Retry rate per suite / per test file | Flaky test detection |
| Worker death rate | Infrastructure health |
| Suite completion time end-to-end | The metric developers care about most |

**Tooling:**
- **Metrics:** Prometheus + Grafana (or CloudWatch if AWS-native)
- **Tracing:** OpenTelemetry across Orchestrator → Queue → Worker → Result Collector (trace a single test case end-to-end)
- **Alerting:** Alert on queue depth > threshold for > 5 minutes (workers not keeping up), worker death rate spike, or p99 dispatch latency crossing SLO

> **This is a principal-level differentiator.** Most candidates stop at the happy path. Showing that you instrument and alert on the system's own health — not just the user-facing behavior — signals operational maturity.

---

## Architecture Diagram (Text)

```
┌─────────────────────────────────────────────────────────────────┐
│                      CLIENT / CI PIPELINE                        │
└────────────────────────────┬────────────────────────────────────┘
                             │ POST /suites
                             ▼
                      ┌─────────────┐
                      │ API Gateway │  (auth, rate limiting, routing)
                      └──────┬──────┘
                             │
                    ┌────────▼────────┐
                    │  Orchestrator   │  writes Suite + TestCases
                    │   Service       │──────────────────────► PostgreSQL
                    └────────┬────────┘
                             │ fan-out: 1 message per TestCase
                             ▼
              ┌──────────────────────────────┐
              │         Task Queue            │
              │    (Kafka / SQS)              │
              │  [queue:linux] [queue:gpu]... │
              └──────┬───────────┬────────────┘
                     │           │
            ┌────────▼──┐  ┌────▼────────┐
            │  Worker 1  │  │  Worker 2   │   ... (auto-scaled)
            │ (executes  │  │             │
            │  test in   │  │             │
            │ container) │  │             │
            └─────┬──────┘  └─────┬───────┘
                  │               │
                  └──────┬────────┘
                         │ publish result
                         ▼
              ┌──────────────────────┐
              │  Result Collector    │  idempotent write
              │     Service          │──────────────────► PostgreSQL
              └──────────┬───────────┘
                         │ publish status-change event
                         ▼
              ┌──────────────────────┐
              │   Kafka topic:        │
              │ suite.{id}.events    │
              └──────────┬───────────┘
                         │ subscribe
                         ▼
              ┌──────────────────────┐
              │    Status Service    │──── SSE push ──► Client
              └──────────────────────┘
                         
              ┌──────────────────────┐
              │    Report Service    │──── Redis cache ──► GET /report
              └──────────────────────┘

              ┌──────────────────────┐
              │   Worker Monitor     │  heartbeat checker, dead worker recovery
              └──────────────────────┘

              ┌──────────────────────┐
              │   Observability      │  Prometheus, OpenTelemetry, Grafana
              └──────────────────────┘
```

---

## Key Design Decisions Summary

| Decision | Choice | Rationale |
|---|---|---|
| Task distribution model | Pull (workers consume from queue) | Workers self-regulate; no central scheduler bottleneck |
| Queue technology | Kafka or SQS | Kafka for high throughput + replay; SQS for simplicity + managed visibility timeout |
| Fan-out strategy | At submission time (1 msg per TestCase) | Independent atomic units; enables max parallelism |
| Real-time status push | SSE over Kafka Pub/Sub | Unidirectional; no WebSocket overhead; scales horizontally |
| Dead worker recovery | Heartbeat + queue visibility timeout + DB requeue | Defense in depth — queue handles it automatically; DB handles edge cases |
| Idempotency | `(test_case_id, task_attempt_id)` unique constraint | Prevents double-counting from at-least-once delivery |
| Consistency model | Strong for final results; eventual for live status | Matches user expectations; avoids over-engineering the dashboard path |
| Worker scaling signal | Queue depth (lag) | Directly represents demand; more accurate than CPU/memory |
| Caching | Reports in Redis | Immutable after generation — perfect cache hit rate |

---

## Interview Tips for This Problem

| Tip | Detail |
|---|---|
| **Clarify test types early** | Unit tests (fast, lightweight) vs. integration/e2e tests (slow, may need DB, network) affects worker isolation requirements |
| **Fan-out is the key insight** | The suite → individual task fan-out is what enables parallelism. Say it explicitly. |
| **Pull vs. push dispatch** | Always choose pull (workers consume from queue) over push (scheduler assigns to workers). Pull is self-regulating; push requires knowing worker state. |
| **Mention idempotency unprompted** | At senior/principal level, you're expected to notice at-least-once delivery and handle it. |
| **Dead worker handling = your Redis lock analog** | Just like the Redis TTL in Ticket Master handles expired reservations, the heartbeat + visibility timeout handles dead workers. Same pattern, different domain. |
| **Queue depth as scaling metric** | Don't say "scale on CPU." Say "scale on queue depth / consumer lag — that's the actual demand signal." |
| **Strong vs. eventual consistency split** | Apply it here the same way as Ticket Master: eventual for dashboard, strong for the final report. Shows architectural maturity. |
| **Observability is a differentiator** | Most candidates skip it. Mentioning metrics, tracing, and alerting on the system's own health is a principal-level signal. |

---

## Comparison: Ticket Master vs. Test Orchestration

| Aspect | Ticket Master | Test Orchestration |
|---|---|---|
| **Core concurrency problem** | Two users booking the same seat | Two workers running the same test / reporting double results |
| **Solution** | Redis distributed lock with TTL | Idempotency key + unique constraint |
| **Task expiry** | Reservation TTL (10 min) | Heartbeat timeout + visibility timeout |
| **Bursty traffic** | Taylor Swift ticket release | Monday 9 AM PR merge flood |
| **Traffic spike solution** | Virtual waiting queue | Elastic worker auto-scaling via queue depth |
| **Real-time updates** | SSE for seat map changes | SSE for test status dashboard |
| **Consistency split** | Strong (booking) / Eventual (search) | Strong (final report) / Eventual (live status) |
| **Search optimization** | Elasticsearch inverted index | N/A — not a search problem |
| **Caching** | Events/venues in Redis | Final reports in Redis |

---

## Glossary

| Term | Definition |
|---|---|
| **ACID** | Atomicity, Consistency, Isolation, Durability — guarantees for reliable DB transactions |
| **API Gateway** | Entry point for all external requests; handles auth, rate limiting, routing |
| **At-Least-Once Delivery** | Messaging guarantee: a message is delivered one or more times; requires idempotent consumers |
| **Auto-Scaling** | Dynamically adjusting the number of worker instances based on observed demand |
| **Change Data Capture (CDC)** | Streaming DB mutations to downstream consumers in real time |
| **Consumer Group (Kafka)** | A set of consumers sharing a Kafka topic; each partition is consumed by exactly one member |
| **Dead Letter Queue (DLQ)** | Queue where messages go after exceeding max delivery attempts; used for debugging failures |
| **Exactly-Once Semantics** | Guarantee that a message is processed exactly once, despite retries; achieved via idempotency keys |
| **Fan-Out** | One input (suite submission) generating many downstream tasks (one per TestCase) |
| **Fault Tolerance** | System continues functioning correctly despite partial failures (worker crashes, network issues) |
| **Heartbeat** | Periodic signal from a worker to the orchestrator proving it is alive; absence triggers recovery |
| **HPA (Horizontal Pod Autoscaler)** | Kubernetes component that scales pod count based on metrics (e.g., custom queue depth metric) |
| **Idempotency** | Property of an operation such that applying it multiple times has the same effect as once |
| **Idempotency Key** | A unique identifier attached to a request/message to detect and discard duplicates |
| **Kafka** | Distributed event streaming platform; high-throughput, durable, supports consumer groups and replay |
| **Message Visibility Timeout** | SQS feature: after a consumer picks up a message, it becomes invisible to others for N seconds; reappears if not ACKed |
| **Offset (Kafka)** | A pointer to a specific position in a Kafka partition; consumers commit their offset after processing |
| **Orchestrator** | Service responsible for accepting suites, breaking them into tasks, and coordinating execution |
| **PostgreSQL** | Relational database with ACID compliance, used here as the primary data store |
| **Pull Model** | Workers actively consume tasks from a queue when ready — self-regulating load distribution |
| **Push Model** | A scheduler actively assigns tasks to specific workers — requires knowledge of worker state |
| **Queue Depth** | Number of messages waiting in a queue; primary scaling signal for worker auto-scaling |
| **Redis** | In-memory key-value store; used here for report caching and Pub/Sub |
| **Redis Pub/Sub** | Lightweight messaging layer in Redis for broadcasting events to subscribers |
| **Retry Limit** | Maximum number of times a failed test is re-executed before being marked permanently failed |
| **Server-Sent Events (SSE)** | Unidirectional persistent HTTP connection; server pushes updates to client |
| **SQS (Simple Queue Service)** | AWS managed message queue; provides at-least-once delivery and visibility timeout |
| **Sticky Sessions** | Load balancer routes a specific client to the same backend instance for the duration of a session |
| **Task Queue** | Intermediate buffer between the Orchestrator and Workers; decouples submission from execution |
| **Test Case** | A single atomic unit of test execution — one test file or one test function |
| **Test Suite** | A collection of test cases submitted together as one job |
| **Visibility Timeout** | See *Message Visibility Timeout* |
| **Worker** | A single container/process that pulls tasks from the queue and executes tests. One physical machine (node) runs many workers. Worker = container, not machine. |
| **Worker Monitor** | Background service that detects dead workers via missed heartbeats and triggers task recovery |
| **Worker Tagging** | Labeling workers with capabilities (e.g., `gpu`, `linux`) to route specialized tasks correctly |
