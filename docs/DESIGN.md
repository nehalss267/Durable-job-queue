# Design Document
## PostgreSQL-Backed Reliable Job Queue

## 1. Design summary

The API durably inserts each accepted job and its idempotency record in one PostgreSQL transaction. Workers poll PostgreSQL for due jobs. A worker claims a job using a short transaction that locks eligible rows with `FOR UPDATE SKIP LOCKED`, assigns a random lease token and expiry, increments the attempt count, and commits. The handler then runs outside the transaction.

Workers heartbeat long-running jobs. Success/failure updates are fenced by the lease token. If a worker dies, its lease expires; a recovery loop makes the job eligible for another attempt or moves it to `dead` when the retry budget is exhausted. Retry timing, attempts, results, and operator actions are all persisted in PostgreSQL.

## 2. Architecture

```text
Client
  |
  | POST /jobs + Idempotency-Key
  v
FastAPI API
  |
  | transaction: INSERT job + idempotency mapping
  v
PostgreSQL 15  <-----------------------------+
  ^                                           |
  | claim due job / heartbeat / ack / reclaim |
  |                                           |
Worker A      Worker B      Worker C          |
  |             |             |               |
  +-------------+-------------+---------------+
                |
                | registered handler
                v
        Partner API adapter
                |
                v
        Mock partner API

Client ---- GET /jobs/{id}, GET /stats ----> FastAPI
Operator -- admin inspect/retry ----------> FastAPI
```

PostgreSQL is the only infrastructure dependency. API and worker processes are stateless with respect to job ownership; their local memory is never the source of truth.

## 3. Core data model

### `jobs`
- `id` UUID/ULID primary key
- `client_id` text, default `default`
- `job_type` text
- `payload` JSONB
- `priority` integer, default 0
- `run_at` timestamptz
- `status` enum/text constrained to `queued`, `running`, `succeeded`, `dead`
- `attempt_count`, `max_attempts`
- `lease_owner` nullable worker ID
- `lease_token` nullable random UUID
- `lease_expires_at` nullable timestamptz
- `started_at`, `finished_at`
- `result` JSONB nullable
- `last_error_code`, `last_error_message` nullable
- `created_at`, `updated_at`

### `idempotency_keys`
- `client_id`
- `idempotency_key`
- `request_hash`
- `job_id` foreign key
- `created_at`
- Unique constraint on `(client_id, idempotency_key)`

### `job_attempts`
- `id` primary key
- `job_id` foreign key
- `attempt_number`
- `worker_id`
- `lease_token_hash` (never expose raw token)
- `started_at`, `finished_at`
- `outcome` (`running`, `succeeded`, `retryable_failure`, `permanent_failure`, `lease_expired`)
- `error_code`, `error_message`
- `result_summary` JSONB
- Unique constraint on `(job_id, attempt_number)`

### `job_events` / `operator_actions`
Append-only audit trail for manual retry and important state transitions. Keep the initial implementation small; attempt history is mandatory, while a full event-sourcing model is not.

## 4. Indexes

Recommended indexes:
- Partial claim index on `(priority DESC, run_at ASC, created_at ASC)` for rows where `status = 'queued'`.
- Partial lease recovery index on `lease_expires_at` for `status = 'running'`.
- `(status, created_at DESC)` for operator dead-job listing.
- `(job_type, status)` for filtered operational views.
- Unique `(client_id, idempotency_key)` for submission deduplication.

Validate index order with `EXPLAIN (ANALYZE, BUFFERS)` against representative data. Do not assume an index is useful merely because it exists.

## 5. Atomic claim algorithm

Within a short transaction:
1. Select one or a bounded batch of rows with `status='queued' AND run_at <= now()`.
2. Order by priority descending, then creation time ascending.
3. Lock selected rows using `FOR UPDATE SKIP LOCKED`.
4. Update each selected row to `running`, set worker ID, fresh lease token, lease expiry, `started_at`, and increment attempt count.
5. Insert corresponding `job_attempts` rows.
6. Commit.
7. Execute handlers outside the transaction.

The exact SQLAlchemy implementation must preserve this atomic boundary. Never select candidates in one transaction and claim them later in another.

## 6. Lease, heartbeat and fencing

- Each claim receives a cryptographically random lease token.
- Lease duration must exceed normal heartbeat scheduling jitter; initial tuning values should be configuration, not constants buried in code.
- Worker heartbeat updates `lease_expires_at` only where job ID, status, worker ID and lease token all match.
- Completion/failure updates use the same predicate.
- If the update affects zero rows, the worker has lost ownership and must not overwrite the current attempt.
- Lease token is a fencing token for database state updates. It cannot stop an already-issued external HTTP request from completing.

A lease that expires does not prove the original worker is dead; it only proves the service no longer trusts its ownership. This is why handlers must be idempotent.

## 7. Crash windows and recovery

| Crash point | Durable state | Recovery behavior | Possible external duplicate |
|---|---|---|---|
| Before claim commit | `queued` | Another worker claims | No handler call |
| After claim commit, before handler | `running` with lease | Reclaim after expiry | Usually none |
| During partner request | `running` with lease | Reclaim after expiry | Partner may have received request |
| After partner success, before DB success commit | `running` with lease | Re-execute after expiry | Yes, unless partner idempotency deduplicates |
| After DB success commit | `succeeded` | No re-execution | No queue-driven duplicate |

## 8. Retry policy

Initial policy (configurable):
- `max_attempts = 5`, counting the first execution.
- Exponential delay: `min(base_delay * 2^(attempt_count - 1), max_delay)`.
- Apply bounded random jitter to avoid synchronized retries.
- Honor `Retry-After` for partner responses when valid and within configured limits.
- Retry network timeouts, connection errors, HTTP 429 and selected 5xx responses.
- Treat invalid payloads and known permanent business 4xx responses as permanent.
- Persist next eligible time in `run_at`.
- When attempts are exhausted, transition to `dead`.

The partner's contract may require different classification. Record the final mapping in the adapter tests.

## 9. Idempotency design

The API requires an `Idempotency-Key` header. The job and key mapping are inserted in one transaction. A unique constraint resolves concurrent submissions. The request body is canonicalized (stable key ordering and normalized timestamps) and hashed. A replay with the same hash returns the original job; a different hash returns `409`.

The partner request uses a stable key derived from the job ID, if supported by the mock API. This protects partner-side effects across worker retries. If the partner does not support idempotency or a receipt lookup, the queue cannot guarantee exactly-once external effects.

## 10. Stats and operational visibility

`GET /stats` uses aggregate queries for counts by status, due/future backlog, active/expired leases, attempts and retry totals. Oldest due job age is computed from the oldest eligible queued row. Worker heartbeat tracking may use a `workers` table with `worker_id`, `last_seen_at`, `started_at`, and `version`; it is diagnostic only and never the source of job ownership.

## 11. Alternatives considered and rejected

### In-memory queue
Rejected: jobs disappear when the API/worker process restarts; no cross-process coordination.

### PostgreSQL advisory locks only
Rejected as the primary ownership model: locks are connection/session-scoped and disappear on connection loss. They do not by themselves provide durable job state, attempt history, retry schedule, or a clear inspectable lease record. They may still be useful for singleton maintenance tasks, but are unnecessary for the core claim path.

### `SELECT` then `UPDATE` without row locking
Rejected: concurrent workers can select the same row before either updates it.

### Holding a DB transaction during handler execution
Rejected: long transactions retain locks/connections, increase contention and vacuum pressure, and still cannot atomically include an external HTTP side effect.

### Celery, RQ, Kafka, Redis, RabbitMQ
Rejected because the brief explicitly restricts infrastructure to PostgreSQL and forbids these queue/broker systems.

### Exactly-once execution
Rejected as an unachievable general guarantee across PostgreSQL and an independent partner API without a distributed transaction or partner idempotency. The chosen honest guarantee is durable acceptance plus at-least-once execution attempts.

## 12. Open questions for client

1. What exact mock partner endpoint, payload, authentication and receipt format will be supplied?
2. Does the partner honor an idempotency header? What is its retention window?
3. Can the partner expose a receipt lookup endpoint for reconciliation?
4. What are the expected throughput and maximum concurrent workers?
5. What payload/result size limits are expected?
6. Should priority be strict or should old low-priority work age upward?
7. Are job cancellation and hard execution timeouts required?
8. What operator authentication model is expected?
9. How long should successful and dead jobs be retained?
10. Should manual retry reuse the same job ID (chosen) or create a linked job?

## 13. Initial configuration proposal

| Setting | Initial value | Rationale |
|---|---:|---|
| Poll interval | 250 ms | Responsive without tight spin |
| Claim batch | 1–10 | Keep ownership bounded; tune by throughput |
| Lease duration | 60 s | Initial default; validate against partner latency |
| Heartbeat interval | 15 s | Four heartbeats per lease interval |
| Max attempts | 5 | Bounded retry budget |
| Backoff base | 1 s | Quick first retry |
| Backoff cap | 5 min | Avoid retry storms |
| Partner connect timeout | 3 s | Fail promptly on connection issues |
| Partner read timeout | 15 s | Must be adjusted to mock API behavior |

All values are starting assumptions, not performance claims. Tune from mock measurements and soak results.
