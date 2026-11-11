# PostgreSQL-Backed Reliable Job Queue
## Execution Plan, Technical Approach & Delivery Commitments

**Stack:** Python 3.11 · FastAPI · SQLAlchemy 2 · Alembic · PostgreSQL 15 · pytest · ruff · mypy · Docker Compose  
**Infrastructure constraint:** PostgreSQL is the only infrastructure dependency. No Celery, RQ, Kafka, Redis, RabbitMQ, or external broker.  
---

## 1. Executive objective

Build a durable background job service that accepts named jobs over HTTP, stores them in PostgreSQL, and executes them through multiple independent worker processes. The system must recover work after worker crashes, retry transient failures, retain exhausted jobs for human inspection, prevent duplicate job creation when clients retry submissions, and expose useful operational health.

The project is not complete when the API returns `202`. It is complete when we can demonstrate—using worker termination, randomized failures, database inspection, and the mock partner's actual receipt records—that accepted work remains accounted for and that the delivery guarantee is accurately described.

The first real job handler will call the mentor-provided slow and randomly failing partner API.

## 2. What we are building

### 2.1 Main components

1. **FastAPI application**
   - Accepts jobs.
   - Validates job type and payload.
   - Implements the fixed API contract.
   - Provides job status and operational statistics.
   - Provides protected operator actions for dead-job inspection and retry.

2. **PostgreSQL database**
   - Stores job definitions, payloads, state, schedule, priority, attempts, leases, results, idempotency records, and operator audit history.
   - Coordinates concurrent workers.
   - Is the single source of truth.

3. **Worker process**
   - Polls PostgreSQL for due work.
   - Atomically claims jobs.
   - Executes registered handlers outside the claim transaction.
   - Renews leases while work is running.
   - Acknowledges success/failure only if it still owns the lease.

4. **Recovery loop**
   - Finds jobs whose worker lease expired.
   - Makes them eligible for another attempt or moves them to the dead-letter state.
   - Prevents abandoned `running` rows from remaining stuck forever.

5. **Partner API adapter**
   - Calls the mentor's mock partner API.
   - Uses explicit network timeouts.
   - Classifies retryable and permanent failures.
   - Reuses a stable partner idempotency key if the mock supports it.
   - Records a safe response summary.

6. **Test and verification harness**
   - Starts multiple workers.
   - Submits controlled workloads.
   - Kills workers at selected failure boundaries.
   - Reconciles accepted jobs, database states, attempts, and partner receipts.

### 2.2 High-level flow

```text
Client
  |
  | POST /jobs + Idempotency-Key
  v
FastAPI
  |
  | One DB transaction:
  | - insert job
  | - insert idempotency mapping
  v
PostgreSQL
  ^
  | atomic claim / heartbeat / acknowledgement / recovery
  |
Worker 1       Worker 2       Worker N
  |               |              |
  +---------------+--------------+
                  |
             Job handler
                  |
            Partner API

Client ---> GET /jobs/{id}
Client ---> GET /stats
Operator -> inspect dead jobs / request manual retry
```

## 3. Correctness contract: what we will and will not promise

### 3.1 Durable acceptance

Once the API returns `202 Accepted`, the job row and its idempotency mapping must have committed in PostgreSQL. If the database transaction fails, the API must not claim the job was accepted.

This is the boundary behind the phrase “no accepted job is lost.”

### 3.2 Execution guarantee: at-least-once

The system provides **at-least-once execution attempts**, not exactly-once execution.

A worker can:
1. call the partner successfully;
2. receive a successful response;
3. crash before committing `succeeded` to PostgreSQL.

After the lease expires, another worker cannot know with certainty whether the external operation happened, so it must retry. This can produce a duplicate external request.

### 3.3 Requirements on job authors

Every handler must be safe to execute more than once. For external effects, authors must use a stable idempotency key or a domain-level deduplication mechanism. For database effects, use unique constraints, upserts, or transactions that make repeated application harmless.

A lease token can fence stale database updates. It cannot retract a request already sent to an independent partner.

### 3.4 Assumptions behind “no job is lost”

- PostgreSQL is eventually available again.
- Committed database data remains durable.
- The database volume is not deleted or corrupted without a recoverable backup.
- Operators do not manually delete accepted jobs.
- Workers are eventually restarted and the handler is not permanently blocked.

A job may end in `dead`; that means it is retained and inspectable, not silently lost. It does not mean every business operation is guaranteed to succeed.

---

# 4. Week-by-week execution plan

## WEEK 1 — Understand, design, specify, and establish the foundation

**Weekly objective:** Before writing the core worker, make the system's behavior precise enough that a reviewer can challenge the assumptions and the implementation can be tested against an agreed contract.

**Expected outcome:** A reviewed `DESIGN.md`, a defined API and state machine, a bootable project skeleton, and a working PostgreSQL migration.

### Day 1 — Requirement decomposition and ambiguity review

**Work**
- Convert the brief into functional requirements, non-functional requirements, and explicit acceptance criteria.
- Identify every unspecified behavior:
  - idempotency-key scope and retention;
  - priority ordering and starvation;
  - default retry count and backoff;
  - what constitutes a permanent error;
  - lease duration and heartbeat interval;
  - maximum payload/result size;
  - partner idempotency support;
  - job retention;
  - operator authentication;
  - cancellation and execution timeout expectations.
- Define the meaning of “no job is ever lost.”
- Separate durable acceptance from execution delivery and external side-effect guarantees.
- Create a risk list and questions for the mentor.

**Output**
- PRD baseline.
- Assumptions and open questions.
- Initial acceptance-test checklist.

**Review checkpoint**
- Confirm that no requirement is being interpreted as exactly-once execution without evidence.

### Day 2 — Architecture and state machine

**Work**
- Compare PostgreSQL row locking, advisory locks, and an external broker.
- Select PostgreSQL row-level locking with `FOR UPDATE SKIP LOCKED` for the claim path.
- Define the job state model:
  - `queued`;
  - `running`;
  - `succeeded`;
  - `dead`.
- Treat future `run_at` jobs as queued but not claimable until due; expose future work separately in status/statistics.
- Define the lease lifecycle:
  - claim;
  - heartbeat;
  - lease expiry;
  - reclaim;
  - stale-worker fencing.
- Define all transaction boundaries.
- Draw sequence diagrams for submit, claim, execute, retry, and crash recovery.

**Output**
- Architecture section in `DESIGN.md`.
- State-transition table.
- Crash-window analysis.

**Review checkpoint**
- Reviewer can explain why no transaction remains open while an HTTP handler runs.

### Day 3 — API and partner contract

**Work**
- Specify `POST /jobs`, `GET /jobs/{id}`, and `GET /stats`.
- Define request/response models and stable error codes.
- Define required `Idempotency-Key` semantics:
  - same key + same body returns original job;
  - same key + different body returns `409`;
  - concurrent duplicate requests create one row.
- Define optional operator endpoints for listing dead jobs, attempt history, and manual retry.
- Obtain the mock partner API specification from the mentor.
- Inspect its method, endpoint, authentication, latency, failure modes, retry headers, idempotency support, and receipt lookup.
- Do not invent partner behavior before seeing its contract.

**Output**
- `API.md`.
- Partner adapter interface.
- OpenAPI contract draft.

**Review checkpoint**
- Fixed assessment endpoints remain unchanged.

### Day 4 — Repository and infrastructure foundation

**Work**
- Establish the application package layout.
- Configure FastAPI, settings, structured logging, and dependency injection.
- Add SQLAlchemy 2 models.
- Add Alembic initial migration.
- Add PostgreSQL 15 to Docker Compose.
- Add API and worker service definitions.
- Add pytest, ruff, and mypy configuration.
- Add health/readiness checks and environment-variable documentation.
- Verify startup from a clean checkout.

**Output**
- Bootable application skeleton.
- Initial migration.
- Compose configuration.
- CI baseline.

### Day 5 — Design review and Week 1 exit

**Work**
- Present `DESIGN.md` to the mentor.
- Walk through the state machine and each crash window.
- Explain why at-least-once is the honest guarantee.
- Review alternatives rejected and why.
- Incorporate mentor feedback.
- Freeze initial API/state-machine decisions.
- Convert review comments into explicit backlog items.

**Output**
- Reviewed `DESIGN.md`.
- Week 1 review notes.
- Updated risk and decision log.

**Week 1 exit criteria**
- [ ] Assumptions and unanswered questions are written down.
- [ ] Design review has happened before core implementation expands.
- [ ] Database schema and migrations are defined.
- [ ] API request/response contract is documented.
- [ ] Project starts with Docker Compose.
- [ ] Delivery guarantee and author obligations are unambiguous.

---

## WEEK 2 — Implement the end-to-end queue and worker

**Weekly objective:** Deliver a functioning vertical slice, then implement the reliability mechanisms around it.

**Expected outcome:** A client can submit a partner job, a worker can execute it, the result is persisted, and failed/crashed executions recover correctly.

### Day 6 — Durable submission and idempotency

**Work**
- Implement `POST /jobs`.
- Validate job type, payload, priority, and timezone-aware `run_at`.
- Canonicalize request JSON and calculate a stable body hash.
- Create the job row and idempotency mapping in one transaction.
- Add a unique constraint on `(client_id, idempotency_key)`.
- Handle concurrent requests that race on the same key.
- Return the original job ID on a valid replay.
- Return `409` if the same key is reused with a different request body.

**Tests**
- First submission persists and returns an ID.
- Same key/same body returns same ID.
- Same key/different body returns conflict.
- Concurrent duplicate submissions create one job.
- API restart does not erase idempotency records.
- Database failure never returns false acceptance.

### Day 7 — Status endpoint, statistics, and database query paths

**Work**
- Implement `GET /jobs/{id}`.
- Return safe job status, schedule, attempts, timestamps, result, and safe error summary.
- Implement `GET /stats`.
- Calculate due queue depth, future scheduled count, running count, succeeded count, dead count, expired leases, and oldest due job age.
- Add indexes for claim, lease recovery, and operator listing.
- Use bounded aggregate queries rather than loading every job into application memory.

**Tests**
- Unknown job returns `404`.
- Result appears only after success.
- Stats match known fixture rows.
- Future jobs are not counted as due work.
- Stats query remains bounded.

### Day 8 — Worker process and atomic claim

**Work**
- Implement a typed handler registry keyed by `job_type`.
- Implement worker identity and configurable concurrency.
- Implement a short claim transaction using `FOR UPDATE SKIP LOCKED`.
- Set status to `running`.
- Assign a fresh lease token and lease expiry.
- Increment attempt count and create an attempt-history row.
- Commit before invoking the handler.
- Add polling with a bounded interval and claim batch size.
- Add graceful shutdown that stops new claims first.

**Tests**
- Multiple workers claim different jobs.
- No two workers own the same valid lease.
- A job is not claimed before `run_at`.
- Priority and FIFO tie-breaking behave as documented.
- Handler execution occurs outside the claim transaction.

### Day 9 — Heartbeats, fencing, retries, and recovery

**Work**
- Implement heartbeat updates conditional on job ID, worker ID, and lease token.
- Implement success acknowledgement conditional on the same ownership fields.
- Implement failure acknowledgement with retry classification.
- Persist exponential backoff with jitter and a maximum delay.
- Respect partner `Retry-After` where appropriate and bounded.
- Implement lease-expiry recovery.
- Ensure stale workers cannot update a job after ownership changes.
- Move exhausted jobs to `dead`.

**Tests**
- Healthy long-running handler renews its lease.
- Stale lease token cannot heartbeat or acknowledge.
- Retry schedule is persisted.
- Retryable failure is requeued.
- Permanent failure is dead-lettered.
- Expired lease is recovered by another worker.
- Attempt history remains complete.

### Day 10 — Partner handler and operator workflows

**Work**
- Implement `partner.request` using the mentor-provided mock API.
- Configure connect/read timeouts.
- Classify network errors, timeouts, rate limits, selected 5xx, and permanent business errors.
- Send stable partner idempotency key if supported.
- Store partner receipt/reference and response summary.
- Implement paginated dead-job inspection.
- Implement manual retry with operator reason and audit event.
- Protect operator routes appropriately for the deployment context.

**Tests**
- Partner success persists result.
- Slow partner does not block a database transaction.
- Random transient failure retries.
- Permanent failure does not retry indefinitely.
- Dead job can be inspected and manually retried.
- Partner idempotency behavior matches the mock contract.

**Week 2 exit criteria**
- [ ] Submit → claim → execute → persist result works end to end.
- [ ] Multiple workers execute concurrently.
- [ ] Lease heartbeat and stale-worker fencing work.
- [ ] Crashed workers' jobs are recovered.
- [ ] Retry schedule is persisted and bounded.
- [ ] Exhausted jobs remain inspectable.
- [ ] Mock partner job is integrated.
- [ ] Fixed endpoints pass contract tests.

---

## WEEK 3 — Prove guarantees, harden, document, and demonstrate

**Weekly objective:** Move from “it appears to work” to repeatable evidence. Test the failure windows that can invalidate the delivery claim.

**Expected outcome:** Chaos/soak evidence, clean quality checks, complete operational documentation, and a successful live walkthrough including an on-the-spot change.

### Day 11 — Guarantee-focused integration tests

**Work**
- Test durable acceptance around API process restart.
- Test idempotency under concurrent client submissions.
- Test concurrent worker claiming.
- Test stale-worker acknowledgement after lease reassignment.
- Test delayed execution and priority ordering.
- Test retry exhaustion and manual retry.
- Use an injectable clock/random source for deterministic retry and scheduling tests.

**Output**
- Focused integration test suite.
- Test-to-guarantee traceability matrix.

### Day 12 — Worker-kill chaos tests

**Work**
Kill workers at controlled boundaries:

1. After claim commit but before handler invocation.
2. During the partner HTTP request.
3. After partner success but before success acknowledgement commits.
4. During heartbeat.
5. While retry scheduling is being persisted.

For each run, record:
- logical job ID;
- submission idempotency key;
- attempt number;
- worker ID;
- lease token identifier (never expose the secret token);
- partner request/receipt identifier;
- final database state;
- timestamps for claim, lease expiry, recovery, and completion.

**Reconciliation**
- Every accepted job ID must still exist in durable job state.
- Every job must be queued/running/succeeded/dead according to a valid transition.
- No stale worker may overwrite a newer attempt.
- Partner receipt count must be interpreted using the mock's actual idempotency behavior.
- If partner idempotency is absent, duplicate external effects are an expected at-least-once window, not a hidden test failure.

**Output**
- Chaos test report with counts and failure-boundary results.

### Day 13 — Multi-worker soak test

**Work**
- Submit thousands of jobs with varied payloads, priorities, and `run_at` values.
- Start several worker processes.
- Inject randomized partner latency and failures.
- Randomly kill and restart workers during the run.
- Observe throughput, queue wait time, execution duration, retries, lease recoveries, and dead-letter count.
- Reconcile accepted logical IDs, database terminal states, attempt rows, and partner receipts.
- Inspect for stuck running jobs, missing rows, duplicate idempotency records, and counter mismatches.
- Tune indexes, polling interval, claim batch size, heartbeat, and connection pool only when measurements justify the change.

**Output**
- Soak test script.
- Machine-readable run summary and human-readable report.
- Query-plan notes for hot SQL paths.

### Day 14 — Hardening, CI, and documentation

**Work**
- Run complete pytest suite.
- Run ruff formatting/lint checks.
- Run mypy with no inline suppressions.
- Validate Alembic upgrade from an empty database.
- Verify `docker compose up` from a clean environment.
- Check structured logs for secrets, payload leakage, and unbounded output.
- Document environment variables, commands, health checks, and recovery behavior.
- Complete README and `LIMITATIONS.md`.
- Add CI workflow for tests, lint, type checking, and migration validation.

**Output**
- CI workflow.
- Complete README.
- Final limitations and operational notes.
- Clean quality-check report.

### Day 15 — Live walkthrough and requirement-change rehearsal

**Demonstration sequence**
1. Start the full stack from Docker Compose.
2. Submit a job with an idempotency key.
3. Submit it again and show that the same job ID is returned.
4. Show that a future-scheduled job is not claimed early.
5. Start multiple workers and show concurrent processing.
6. Show successful partner execution and persisted result.
7. Kill a worker mid-job.
8. Show lease expiry, recovery, and another attempt.
9. Trigger retry exhaustion and inspect the dead job.
10. Manually retry after remediation.
11. Show `/stats` and explain what each metric means.
12. Explain the exact delivery guarantee and the external side-effect limitation.

**Live change exercise**
- Rehearse one small, bounded change such as adding a new registered job type or changing the retry cap.
- Explain the affected interface, database/API impact, and test.
- Implement the change live.
- Run focused tests and relevant full checks.
- Do not choose a change that requires pretending an untested migration or guarantee is safe.

**Output**
- Walkthrough checklist.
- Final test/soak/chaos evidence.
- Updated design decisions and known limitations.

**Week 3 exit criteria**
- [ ] Chaos tests cover all critical crash windows.
- [ ] Soak test runs thousands of jobs with several workers and randomized kills.
- [ ] Database state and partner receipts are reconciled.
- [ ] No accepted job silently disappears under stated assumptions.
- [ ] Delivery guarantee is stated accurately.
- [ ] README, CI, and limitations are complete.
- [ ] Compose startup works from a clean environment.
- [ ] Live change can be explained, implemented, and tested.

---

# 5. Initial technical decisions

| Concern | Decision | Why |
|---|---|---|
| Queue storage | PostgreSQL table | Only permitted infrastructure and durable source of truth |
| Worker concurrency | `FOR UPDATE SKIP LOCKED` | Concurrent workers can claim separate rows atomically |
| Handler transaction | Outside claim transaction | Avoid long-held locks and DB connections during network I/O |
| Ownership | Lease + random token | Recover abandoned work and fence stale DB acknowledgements |
| Delivery | At-least-once attempts | Honest guarantee across independent DB and partner systems |
| Client deduplication | Idempotency key + unique constraint + body hash | Safe client request retries |
| Retry | Exponential backoff + jitter + cap | Avoid synchronized retry storms |
| Exhausted work | Durable `dead` state | Human inspection and controlled retry |
| Scheduling | `run_at` in PostgreSQL | No separate scheduler infrastructure |
| Priority | Higher first, FIFO tie-break | Simple documented ordering |
| Metrics | Aggregate SQL via `/stats` | Operational visibility without scanning all records |

## 6. Proposed initial configuration

These are starting values to validate with the mock partner and soak test, not performance claims.

| Setting | Initial value |
|---|---:|
| Worker poll interval | 250 ms |
| Claim batch size | 1–10 |
| Lease duration | 60 seconds |
| Heartbeat interval | 15 seconds |
| Maximum attempts | 5 |
| Backoff base | 1 second |
| Backoff cap | 5 minutes |
| Partner connect timeout | 3 seconds |
| Partner read timeout | 15 seconds |

All settings should be environment-configurable and documented. Use PostgreSQL time for lease predicates and UTC for persisted timestamps.

## 7. Database entities

### `jobs`
Stores job identity, type, JSON payload, priority, `run_at`, status, attempt count, maximum attempts, lease owner/token/expiry, result, safe last error, and lifecycle timestamps.

### `idempotency_keys`
Stores client scope, idempotency key, canonical request hash, job ID, and creation time. Unique key: `(client_id, idempotency_key)`.

### `job_attempts`
Stores attempt number, worker ID, start/end time, outcome, safe error summary, and partner receipt/reference. Unique key: `(job_id, attempt_number)`.

### `operator_actions`
Stores who manually retried a job, when, and why. This provides an audit trail for operator-triggered re-execution.

## 8. API contract

The assessment's fixed routes are non-negotiable:

### `POST /jobs`
Accepts `job_type`, `payload`, optional `priority`, optional timezone-aware `run_at`, and required `Idempotency-Key`.

- First accepted submission: `202`.
- Same key and same body: return original job ID.
- Same key and different body: `409`.
- Invalid payload/type: `422`.
- Database cannot confirm commit: non-success response.

### `GET /jobs/{id}`
Returns status, job type, priority, schedule, timestamps, attempts, result on success, and safe error summary.

### `GET /stats`
Returns counts by state, due and future backlog, running jobs, expired leases, attempts/retries, oldest due job age, and database/worker health indicators.

Additional operator routes should be protected and paginated.

## 9. Test strategy and proof matrix

| Claim | Test | Evidence |
|---|---|---|
| Accepted jobs are durable | Submit, restart API/worker, query DB/API | Same job ID and persisted row |
| Submission retry is idempotent | Repeat and race same key | Exactly one logical job |
| Workers do not double-claim a live lease | Concurrent worker integration test | Unique active ownership |
| Stale workers cannot acknowledge | Expire/reassign lease, then stale ack | Conditional update affects zero rows |
| Crashes are recoverable | Kill after claim | Lease recovery and next attempt |
| Retry is bounded | Force repeated transient errors | Persisted delays, then `dead` |
| Dead jobs are inspectable | Exhaust attempts | Payload metadata and attempt history available to operator |
| Partner behavior is understood | Kill after partner success | Receipt reconciliation and idempotency outcome |
| System survives load/failure | Thousands of jobs + random kills | Reconciliation report and no unaccounted accepted jobs |

### Required soak-test reconciliation

At the end of each run, calculate:

- `accepted_logical_jobs`
- `jobs_present_in_database`
- `jobs_succeeded`
- `jobs_dead`
- `jobs_queued_or_running`
- `total_attempt_records`
- `unique_partner_receipts`
- `total_partner_requests`
- `duplicate_partner_effects` (only where the mock exposes enough information)
- `unaccounted_accepted_jobs`

The key invariant is:

`accepted_logical_jobs == jobs_present_in_database`

At a fully drained test end-state:

`accepted_logical_jobs == jobs_succeeded + jobs_dead`

If the test intentionally stops before draining, remaining queued/running jobs must still exist and be explainable.

Partner request count is not expected to equal logical job count under at-least-once execution. Partner *effects* should be deduplicated only when the partner actually supports the required idempotency contract.

## 10. Risks and mitigations

| Risk | Impact | Mitigation |
|---|---|---|
| Partner has no idempotency support | Duplicate external effects after crash | Document at-least-once; use domain deduplication/receipt reconciliation if available |
| Lease too short | Healthy work may be reclaimed | Measure latency; heartbeat; configurable lease |
| Lease too long | Crash recovery is delayed | Tune against handler duration and heartbeat |
| Priority starvation | Low-priority backlog grows | Monitor oldest age; consider aging only as an explicit later change |
| Polling load | PostgreSQL pressure | Index claim query; tune poll interval/batch size; soak test |
| Database volume loss | Accepted jobs may be lost | Persistent volume, backup/restore procedure, explicit assumption |
| Payload growth | Storage/WAL and API pressure | Payload size limits and references for large data |
| Retry storm | Partner overload | Exponential backoff, jitter, `Retry-After`, bounded attempts |

## 11. Change-request procedure

One requirement change is expected during the project. When it arrives:

1. Write the exact changed behavior and reason.
2. Identify impacted PRD requirement IDs, API schema, database model, worker lifecycle, and tests.
3. State backward-compatibility and migration implications.
4. Update `DESIGN.md` and decision log before implementation.
5. Add or update acceptance tests first.
6. Implement the smallest coherent change.
7. Re-run relevant reliability tests and full CI.
8. Update README and limitations if the guarantee or operational behavior changed.

Never silently weaken the durability or delivery contract to fit a deadline.

## 12. Definition of done

The project is done only when all of the following are true:

- [ ] The three fixed API routes work as documented.
- [ ] Job submission and idempotency mapping commit atomically.
- [ ] Multiple workers claim jobs safely.
- [ ] Handler execution happens outside claim transactions.
- [ ] Leases, heartbeats, fencing, and recovery work.
- [ ] Retry policy is bounded and persisted.
- [ ] Dead jobs are retained and can be manually retried.
- [ ] Mock partner handler is implemented against its actual contract.
- [ ] Chaos tests kill workers at critical boundaries.
- [ ] Soak tests use thousands of jobs, several workers, and random kills.
- [ ] Partner receipts are reconciled with database outcomes.
- [ ] README and `LIMITATIONS.md` are complete.
- [ ] CI runs pytest, ruff, mypy, and migration validation.
- [ ] `docker compose up` works from a clean environment.
- [ ] The engineer can explain and change the implementation live.

---

## 13. What I will explicitly avoid

- I will not claim exactly-once execution when the architecture cannot provide it.
- I will not treat a successful HTTP response from the partner as proof that the database recorded success.
- I will not run job handlers while holding row locks or long database transactions.
- I will not keep retry state only in worker memory.
- I will not allow a stale worker to overwrite a newer lease owner's database state.
- I will not call a job “lost” merely because it is dead-lettered; it remains durable and inspectable.
- I will not report chaos/soak tests as passing without recording the actual partner receipts and database reconciliation.
- I will not add infrastructure prohibited by the brief.
- I will not merge a requirement change without updating the design, tests, and relevant documentation.

