# Three-Week Engineering Roadmap
## PostgreSQL-Backed Reliable Job Queue

**Delivery principle:** Week 1 proves the design before implementation expands. Week 2 builds the vertical slice and core reliability. Week 3 proves behavior under concurrency and failure, hardens operations, and prepares the live change walkthrough.

## Week 1 — Design, contracts, and foundation

### Day 1: Requirement discovery and risk analysis
- Translate the brief into functional/non-functional requirements.
- Identify ambiguities: idempotency scope, retry policy, priority semantics, payload limits, partner API behavior, retention, operator access.
- Draft assumptions and client questions.
- Define the exact meaning of “no job is ever lost.”
- Write delivery guarantee and job-author obligations.

**Deliverables:** PRD draft, assumptions/questions, risk register.

### Day 2: Architecture and state machine
- Compare PostgreSQL polling, advisory locks, and external brokers.
- Choose row-level claim transactions with `FOR UPDATE SKIP LOCKED`.
- Define lease token, lease expiry, heartbeat, reclaim and fencing behavior.
- Define state transitions and attempt history.
- Define transaction boundaries: accept, claim, heartbeat, complete, fail, reclaim, manual retry.

**Deliverables:** architecture diagram in `DESIGN.md`, state transition table, sequence diagrams.

### Day 3: API and data contract
- Finalize request/response schemas for the three fixed endpoints.
- Specify idempotency-key semantics and conflict behavior.
- Define operator endpoints for dead-job inspection and retry.
- Define error envelope, UTC rules, pagination and payload limits.
- Obtain and inspect the mentor's mock partner API contract as soon as it is supplied.

**Deliverables:** `API.md`, OpenAPI schema, partner adapter contract.

### Day 4: Repository and database foundation
- Create FastAPI application structure and settings.
- Add SQLAlchemy 2 models and Alembic initial migration.
- Add Docker Compose with PostgreSQL 15 and API/worker service placeholders.
- Add ruff, mypy and pytest configuration.
- Add health/readiness checks and structured logging.

**Deliverables:** bootable skeleton, migration, CI baseline.

### Day 5: Design review and freeze
- Review `DESIGN.md` with mentor before expanding implementation.
- Resolve feedback and record rejected alternatives.
- Confirm retry/lease defaults and mock partner semantics.
- Turn review feedback into tracked backlog items.
- Freeze v1 API and state-machine contract.

**Deliverables:** reviewed `DESIGN.md`, Week 1 review notes, updated roadmap.

**Week 1 exit criteria**
- A reviewer can explain how jobs are accepted, claimed, recovered and retried.
- The delivery guarantee is precise and not described as exactly-once.
- Every ambiguous requirement has a written decision or open question.
- The project starts with `docker compose up` and migrations apply cleanly.

## Week 2 — Build the complete vertical slice

### Day 6: Durable submission and idempotency
- Implement `POST /jobs`.
- Persist job and idempotency mapping in one transaction.
- Canonicalize request body and store a cryptographic body hash.
- Add unique constraint for `(client_id, idempotency_key)`.
- Test duplicate same-body replay and different-body conflict.

### Day 7: Status and stats
- Implement `GET /jobs/{id}` with safe response models.
- Implement `GET /stats` using aggregate queries and bounded work.
- Add state and timestamp indexes.
- Add tests for missing IDs, state serialization and counter consistency.

### Day 8: Atomic worker claim and handler registry
- Implement worker process and typed handler interface.
- Claim due work in a short transaction using `SKIP LOCKED`.
- Assign worker ID, random lease token and lease expiry.
- Commit before invoking the handler.
- Implement graceful shutdown that stops claiming new work and allows bounded completion.

### Day 9: Leases, heartbeat and retry policy
- Implement heartbeat updates conditional on lease token.
- Implement conditional success/failure acknowledgement.
- Implement retryable/permanent error classification.
- Persist exponential backoff with jitter and max delay.
- Add expired-lease recovery and attempt history.

### Day 10: Partner job and dead-letter operations
- Implement `partner.request` adapter against the provided mock.
- Apply explicit HTTP timeouts and stable external idempotency key if supported.
- Implement dead-letter transition and operator inspection/manual retry.
- Add audit trail for manual retry.
- Add integration tests using a controllable fake partner if the mentor mock is not yet stable.

**Week 2 exit criteria**
- A submitted job can travel through API → database → worker → handler → persisted result.
- Multiple workers claim different jobs safely.
- Worker death is recoverable through lease expiry.
- Retries and dead-letter transitions are persisted and testable.
- Partner job handles success, timeout, transient failure and permanent failure.

## Week 3 — Prove, harden, document, and demonstrate

### Day 11: Guarantee-focused tests
- Test durable acceptance around API process restart.
- Test idempotency races with concurrent requests.
- Test two workers competing for the same due jobs.
- Test stale worker acknowledgement after lease reassignment.
- Test delayed jobs and priority ordering.

### Day 12: Chaos testing
- Run partner jobs and kill workers at controlled points:
  1. after claim, before handler;
  2. during partner request;
  3. after partner success, before DB success commit;
  4. during heartbeat;
  5. during retry scheduling.
- Compare accepted logical job IDs, attempt records, DB terminal state and partner receipts.
- Verify duplicate partner receipts are prevented only if partner idempotency is supported; otherwise demonstrate the expected at-least-once duplicate window.

### Day 13: Soak and performance testing
- Submit thousands of jobs with varied priorities, schedules and payload sizes.
- Run several workers with randomized process kills and restarts.
- Record throughput, queue latency, retry count, recovery latency and partner receipts.
- Check for stuck `running` rows, missing accepted jobs, duplicate idempotency rows and inconsistent counters.
- Tune claim batch size, indexes, poll interval and heartbeat interval based on evidence.

### Day 14: Hardening and operational docs
- Run full pytest suite, ruff, mypy and migration checks.
- Inspect logs for secret leakage and unbounded payload output.
- Validate graceful shutdown and restart behavior.
- Complete README, `LIMITATIONS.md`, runbooks and CI workflow.
- Ensure `docker compose up` starts the complete system from a clean volume.

### Day 15: Live walkthrough and change exercise
- Demonstrate submit → inspect → execute → result.
- Demonstrate delayed job and priority behavior.
- Kill a worker mid-job and show lease recovery.
- Show dead-letter inspection and manual retry.
- Explain at-least-once delivery and partner idempotency.
- Make one requested change live (recommended rehearsal: change retry backoff cap or add a new job type), run focused tests, then full checks.

**Week 3 exit criteria**
- Chaos and soak tests produce auditable evidence for the stated guarantees.
- No accepted job disappears from durable storage under the test assumptions.
- All jobs eventually become succeeded or dead when the database and workers recover and handlers return.
- Limitations and failure modes are explicit.
- The developer can explain and modify the implementation live.

## Milestones and dependencies

| Milestone | Target | Dependency |
|---|---|---|
| M1: Design review | End of Week 1 | Mentor review and mock API contract |
| M2: Vertical slice | Mid-Week 2 | Database/API foundation |
| M3: Reliability complete | End of Week 2 | Lease/retry implementation |
| M4: Chaos/soak evidence | Mid-Week 3 | Stable partner mock |
| M5: Final walkthrough | End of Week 3 | CI, docs, clean Compose setup |

## Change-request buffer

Reserve approximately 15–20% of Week 3 capacity for the expected mid-month requirement change. Do not absorb a change by weakening the delivery guarantee silently. Update PRD, API, user stories, tests and `DESIGN.md` together, and record any migration or compatibility impact.
