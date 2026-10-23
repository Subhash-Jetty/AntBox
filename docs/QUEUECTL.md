# Tasko (QueueCTL) — Production-Grade Distributed Job Queue Engine

> **AntBox Relevance:** The exact infrastructure needed to run automated sprint submission evaluation — processing thousands of concurrent student submissions without losing, duplicating, or silently dropping a single one.

---

## The Problem

Build a production-grade distributed job queue from scratch. Not a wrapper around Celery or RabbitMQ — a ground-up implementation that handles the hard distributed systems problems: concurrent claiming, crash recovery, data integrity, and observability.

**Source:** [github.com/Subhash-Jetty/Tasko](https://github.com/Subhash-Jetty/Tasko)

**Why build from scratch?** Using Celery would demonstrate "I can pip install things." Building the queue from scratch demonstrates understanding of atomic transactions, concurrency primitives, failure modes, and state machine design — the skills that matter when AntBox's sprint evaluation pipeline needs debugging at 2 AM.

---

## Architecture

```
                    ┌─────────────────┐
                    │   queuectl CLI  │
                    │   (Enqueue /    │
                    │    Monitor /    │
                    │    Admin)       │
                    └────────┬────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │   SQLite DB     │
                    │   (WAL Mode)    │
                    │                 │
                    │ ┌─────────────┐ │
                    │ │   jobs      │ │
                    │ │ ┌─────────┐ │ │
                    │ │ │ status  │ │ │
                    │ │ │ worker  │ │ │
                    │ │ │ retries │ │ │
                    │ │ │ heartbt │ │ │
                    │ │ └─────────┘ │ │
                    │ └─────────────┘ │
                    └────────┬────────┘
                             │
              ┌──────────────┼──────────────┐
              │              │              │
         ┌────▼────┐   ┌────▼────┐   ┌────▼────┐
         │Worker 1 │   │Worker 2 │   │Worker 3 │
         │         │   │         │   │         │
         │ Heartbt │   │ Heartbt │   │ Heartbt │
         │ Monitor │   │ Monitor │   │ Monitor │
         └────┬────┘   └────┬────┘   └────┬────┘
              │              │              │
              ▼              X (crash)      ▼
         COMPLETED           │         COMPLETED
                             ▼
                    ┌─────────────────┐
                    │ Orphan Reaper   │
                    │ (Heartbeat      │
                    │  Timeout Check) │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │  Retry or DLQ   │
                    └─────────────────┘
```

---

## Core Design Decisions

### 1. Why SQLite with WAL Mode?

| Feature | WAL Mode Benefit |
|:---|:---|
| Concurrent reads during writes | Readers don't block writers; writers don't block readers |
| Crash safety | WAL provides atomic commits even during power loss |
| Zero deployment dependency | No external message broker to install/configure/monitor |
| Single-file portability | Database is one file — easy to backup, test, migrate |

For a CLI tool, eliminating the Redis/RabbitMQ dependency isn't laziness — it's a design choice that reduces operational surface area.

### 2. Atomic Job Claiming

The hardest problem in a job queue: **10 workers race to claim the next job. Exactly 1 must win. The other 9 must get nothing.**

```sql
-- job_repository.py — claim_next_job()

BEGIN IMMEDIATE;       -- Acquire exclusive write lock immediately

SELECT * FROM jobs
WHERE status = 'PENDING'
ORDER BY created_at ASC
LIMIT 1;

-- If a job is found:
UPDATE jobs
SET status = 'RUNNING',
    worker_id = :worker_identity,
    claimed_at = :now,
    heartbeat = :now
WHERE id = :job_id
  AND status = 'PENDING'    -- Double-check: prevent race condition
RETURNING *;

COMMIT;
```

**Key guarantees:**
- `BEGIN IMMEDIATE` acquires a write lock before the SELECT, preventing other transactions from modifying the same row
- The `AND status = 'PENDING'` in the UPDATE is a compare-and-swap that catches any race condition the lock might not prevent
- `RETURNING *` eliminates a second query — atomic claim-and-read

**Test proof:**
```python
# 10 threads, each with own DB connection, race to claim 1 job
# Result: Exactly 1 gets the job, other 9 get None
# Verified across 100+ runs — zero duplicates, zero losses
```

### 3. Crash Recovery & Orphan Reclamation

**The scenario:** Worker A claims a job, starts processing, then gets SIGKILL'd (no cleanup handler runs). The job is stuck in RUNNING state forever.

**The solution — Heartbeat Reaper:**

```
Every worker:
  - Updates its heartbeat timestamp every N seconds while processing
  
Reaper daemon:
  - Scans for jobs in RUNNING state with heartbeat > 30s stale
  - Marks them as ORPHANED
  - Decrements retry count
  - If retries > 0: re-enqueues as PENDING
  - If retries == 0: moves to Dead Letter Queue (DLQ)
```

**SIGKILL test:**
```python
# 1. Start worker, let it claim a job
# 2. Kill worker with .kill() — SIGKILL equivalent, no cleanup
# 3. Start new worker
# 4. Job reaches 'completed' via new worker
# 5. Execution file has exactly one line (no duplicate execution)
```

### 4. Worker Identity

Each worker has a globally unique identity:

```
Format: HOSTNAME-PID-THREAD_ID-UUID_PREFIX
Example: DESKTOP-ABC-12345-1-a3f7b2c1
```

This enables:
- Identifying exactly which worker orphaned a job
- Debugging production issues by correlating worker identity with logs
- Preventing a restarted worker from accidentally reclaiming its own orphaned job

### 5. Job State Machine

```
                    ┌──────────┐
                    │ PENDING  │ ← Initial state / re-enqueued
                    └────┬─────┘
                         │
                    ┌────▼─────┐
                    │ RUNNING  │ ← Claimed by a worker
                    └────┬─────┘
                         │
          ┌──────────────┼──────────────┐
          │              │              │
     ┌────▼─────┐  ┌────▼─────┐  ┌────▼──────────┐
     │COMPLETED │  │FAIL_RETRY│  │RECLAIM_ORPHAN │
     └──────────┘  └────┬─────┘  └────┬───────────┘
                        │              │
                        │    (retries > 0)
                        │              │
                   ┌────▼─────┐  ┌────▼─────┐
                   │FAIL_DEAD │  │ PENDING  │ (re-enqueued)
                   └────┬─────┘  └──────────┘
                        │
                   ┌────▼─────┐
                   │   DLQ    │ ← Dead Letter Queue
                   └────┬─────┘
                        │
                   ┌────▼──────┐
                   │DLQ_RETRY │ ← Manual admin retry from DLQ
                   └───────────┘
```

**State transitions are forward-only.** A COMPLETED job can never go back to RUNNING. This prevents metric corruption and ensures audit integrity.

---

## Testing Philosophy

This isn't a project where `pytest test_add_numbers.py` counts as testing. The tests validate the **hard distributed systems scenarios:**

| Test Scenario | What It Validates |
|:---|:---|
| 10-thread concurrent claim race | Exactly 1 winner, 9 get None, 0 duplicates |
| SIGKILL recovery | Orphaned job is reclaimed and completed by a new worker |
| DLQ exhaustion | Job that fails 3x lands in DLQ, never silently dropped |
| Idempotent re-claim | Re-enqueued job is processed exactly once |
| Heartbeat staleness | Stale workers are detected within the timeout window |

---

## File Structure

```
tasko/
├── queuectl.py              # CLI entry point (enqueue, start, monitor, admin)
├── job_repository.py         # Database operations, atomic claiming, state transitions
├── worker.py                 # Worker loop, heartbeat, signal handling
├── reaper.py                 # Orphan detection, DLQ routing
├── models.py                 # Job data model, state enum
├── db.py                     # SQLite connection management, WAL setup
├── identity.py               # Worker identity generation
├── tests/
│   ├── test_concurrent.py    # 10-thread race condition stress test
│   ├── test_sigkill.py       # Crash recovery verification
│   ├── test_dlq.py           # Dead letter queue exhaustion
│   └── test_e2e.py           # Full pipeline integration test
├── pyproject.toml             # Package metadata + pytest config
└── README.md                 # Setup, usage, architecture overview
```

---

## Why This Matters for AntBox

AntBox's Career Sprint evaluation pipeline is fundamentally a distributed job queue:

| AntBox Operation | QueueCTL Equivalent |
|:---|:---|
| Student submits sprint deliverable | Job enqueued |
| Evaluation worker processes submission | Worker claims and executes job |
| Evaluation worker crashes mid-grading | Orphan reclamation recovers the submission |
| Submission fails automated checks 3x | Dead Letter Queue — routed to manual review |
| 500 students submit at sprint deadline | Concurrent claiming handles the thundering herd |

This isn't theoretical. The code runs, the tests pass, and the failure modes are verified.

---

*Built from scratch to understand the fundamentals — not to avoid pip install, but to prove mastery of the concepts that matter when production systems fail.*
