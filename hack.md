# Durable job queue: weekly execution plan

Project: Postgres-only job queue and worker
Repo: https://github.com/nehalss267/Task-3-Durable-job-queue
Plan date: 31 October 2026
Current state: core implementation, tests, Docker stack, docs and CI are in place. This plan covers the remaining four weeks: design review, reliability work, the mid-month requirement change, and the final walkthrough.

## 1. Goal

Ship a small service that accepts named jobs, runs them on disposable workers, retries failures, keeps dead letters, prevents duplicate submissions and exposes operator health. Postgres is the only infrastructure. The fixed contract is:

- `POST /jobs`
- `GET /jobs/{id}`
- `GET /stats`

Delivery is at-least-once. Handlers use `job.id` as the downstream idempotency key. No job is lost when a worker dies mid-job.

## 2. Working rules

1. One vertical slice at a time. Each slice includes code, test, docs and a way to run it.
2. Every guarantee in `DESIGN.md` maps to a named test.
3. No merge unless ruff, strict mypy and pytest pass.
4. Decisions go into `backend/DESIGN.md`; failure modes go into `backend/LIMITATIONS.md`.
5. Keep a demo script that starts from a clean database and runs in under 15 minutes.
6. Reserve time for the mid-month requirement change. Do not fill every day with feature work.

## 3. Week-by-week plan

### Week 1: design review and test baseline (2-8 November)

Focus: make sure the design is defensible before adding more code.

| Day | Work | Output |
| --- | --- | --- |
| Mon | Re-read the brief and `DESIGN.md`. List every assumption and unresolved question. | Short review notes |
| Tue | Review the guarantee-to-test map. Add a test wherever a claim has no direct evidence. | Updated map |
| Wed | Walk through the schema, claim query and state machine with the mentor. Record decisions. | Design sign-off notes |
| Thu | Run the full suite from a clean database and a clean Docker volume. | Verification log |
| Fri | Review the mock partner and chaos harness. Confirm the partner call log proves delivery. | Test gaps list |

Deliverables:

- Reviewed `backend/DESIGN.md`.
- Complete guarantee-to-test map.
- List of open questions with owners.
- One clean run of `docker compose up --build` plus `32 passed, 3 skipped`.

Exit criteria:

- The delivery guarantee, idempotency contract and dead-letter behavior are agreed.
- No guarantee depends only on opinion.

### Week 2: reliability and observability (9-15 November)

Focus: prove behavior under failure and make operator signals useful.

| Day | Work | Output |
| --- | --- | --- |
| Mon | Run chaos tests three times. Record kills, retries, duplicate partner calls and lost jobs. | Chaos report |
| Tue | Tune lease, heartbeat, reaper and backoff settings if recovery or duplicate work is too high. | Config notes |
| Wed | Run soak test with at least 1,000 jobs and 4 workers. Record throughput and queue age. | Soak report |
| Thu | Review `/stats` alerts against real failures. Add a missing signal only if it answers an operator question. | Stats review |
| Fri | Test migration from an empty database and from the previous schema. | Migration note |

Deliverables:

- Repeated chaos and soak results.
- Updated tuning values, if any.
- Operator checklist for backlog, expired leases, dead letters and worker health.

Exit criteria:

- Every submitted job is either in the database or has a documented reason it is not.
- Duplicate execution is visible and expected, not hidden.
- Recovery time after a worker kill is measured, not estimated.

### Week 3: requirement change and regression (16-22 November)

Focus: absorb the mid-month requirement change without weakening existing guarantees.

| Day | Work | Output |
| --- | --- | --- |
| Mon | Write the change as a short spec: behavior, edge cases, data changes, API impact. | Change note |
| Tue | Add failing tests for the new behavior before implementation. | Red tests |
| Wed | Implement the smallest change that passes the tests. Update the migration if needed. | Green tests |
| Thu | Run the full regression suite and the chaos test. Check old guarantees still hold. | Regression report |
| Fri | Update `README.md`, `DESIGN.md` and `LIMITATIONS.md`. Remove stale text. | Docs update |

Likely change cases to prepare for:

- A new job type or partner behavior.
- A change to retry policy or attempt limits.
- A new stats field or filter.
- A change to idempotency-key retention.
- A cancellation or scheduling rule.

Exit criteria:

- The change is covered by tests.
- The old contract still passes.
- The docs describe the new behavior in one place.

### Week 4: release and walkthrough (23-29 November)

Focus: make the project easy to run, explain and modify live.

| Day | Work | Output |
| --- | --- | --- |
| Mon | Freeze features. Re-run ruff, mypy and the default suite. | Green build |
| Tue | Run chaos and soak one final time. Save results in the release notes. | Final test report |
| Wed | Rehearse the live change: add a handler or change retry behavior in under 10 minutes. | Walkthrough script |
| Thu | Test from a fresh clone and an empty volume. Confirm `docker compose up --build` works. | Clean-start proof |
| Fri | Tag the release, update the plan and prepare the demo order. | Release candidate |

Walkthrough order:

1. Start the stack.
2. Submit a `noop` job and show success.
3. Submit a `partner_call` job and show retries.
4. Kill a worker mid-job and show recovery.
5. Show a dead letter and retry it.
6. Show `/stats`.
7. Make one live change, run the relevant test, and show the result.

Exit criteria:

- A fresh clone runs without manual database steps.
- The walkthrough fits in 15 minutes.
- The live change touches one clear area and has a test.

## 4. Progress tracker

| Week | Focus | Main deliverable | Status |
| --- | --- | --- | --- |
| 1 | Design review | Reviewed design and test map | Planned |
| 2 | Reliability | Chaos and soak reports | Planned |
| 3 | Requirement change | New behavior plus regression proof | Planned |
| 4 | Release | Clean run and walkthrough | Planned |

## 5. Risks and controls

| Risk | Control |
| --- | --- |
| Requirement change arrives late | Keep week 3 free of new feature work; write the change as tests first |
| Chaos test finds duplicates | Treat duplicates as expected; confirm the partner received every job |
| Lease tuning causes false expiry | Measure recovery time and duplicate rate before changing defaults |
| Docs drift from code | Update docs in the same change that alters behavior |
| Demo fails on a fresh machine | Test from a clean clone and empty volume in week 4 |
| Scope grows | Reject work that does not support the five guarantees or the fixed contract |

## 6. Definition of done

- `docker compose up --build` starts the API, worker, partner and database.
- A submitted job reaches `succeeded` or `failed`.
- A killed worker does not lose a job.
- A repeated submission with the same idempotency key returns one job.
- Dead letters can be inspected and retried.
- `/stats` reflects backlog, leases and worker health.
- Chaos and soak tests pass with `RUN_SLOW_TESTS=1`.
- `python -m ruff check .`, `python -m mypy app tests` and `python -m pytest` pass.
- `DESIGN.md`, `README.md` and `LIMITATIONS.md` match the code.
- The live walkthrough can be completed without notes beyond this plan.
