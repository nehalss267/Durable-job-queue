# Postgres job queue

A job queue and worker that use only Postgres. Clients submit a named job with a JSON payload, an optional priority (0-9, higher first) and an optional `run_at`. Workers claim jobs with `FOR UPDATE SKIP LOCKED`, heartbeat a lease and recover work from dead workers. Failed jobs retry with capped exponential backoff. Jobs that exhaust their attempts, or fail permanently, stay in `failed` for inspection and manual retry.

Delivery is **at-least-once**. Handlers must be idempotent and use `job.id` as the downstream idempotency key. See [DESIGN.md](DESIGN.md) for the contract and [LIMITATIONS.md](LIMITATIONS.md) for failure modes.

## Quick start

Docker with Compose v2 is required.

```bash
cd backend
docker compose up --build
```

That starts:

| Service | URL | Purpose |
| --- | --- | --- |
| `api` | http://localhost:8000 | FastAPI and OpenAPI docs at `/docs` |
| `worker` | - | Job worker; add replicas with `docker compose up --scale worker=3` |
| `partner` | http://localhost:8001 | Mock flaky partner |
| `db` | `localhost:5432` | Postgres 15 (`queue`/`queue`, database `queue`) |
| `migrate` | - | One-shot `alembic upgrade head` |

If a port is taken, override it:

```bash
POSTGRES_PORT=5433 API_PORT=8080 PARTNER_PORT=8081 docker compose up --build
```

Submit a job:

```bash
curl -sS -X POST http://localhost:8000/jobs \
  -H 'Content-Type: application/json' \
  -H 'Idempotency-Key: checkout-42' \
  -d '{"name":"noop","payload":{"hello":"world"},"priority":5}'

curl -sS http://localhost:8000/jobs/<id>
curl -sS http://localhost:8000/stats
```

Call the mock partner. It fails randomly by default.

```bash
curl -sS -X POST http://localhost:8000/jobs \
  -H 'Content-Type: application/json' \
  -d '{"name":"partner_call","payload":{"account":"acct-1","amount":10},"max_attempts":10}'

curl -sS http://localhost:8001/_received
curl -sS -X POST http://localhost:8001/_config \
  -H 'Content-Type: application/json' \
  -d '{"failure_rate":0,"min_latency_ms":0,"max_latency_ms":0}'
```

`_received` lists every call the partner got, keyed by queue job id. Tests use it to check delivery.

## HTTP API

The first three endpoints are the fixed contract. The rest are for operators and tests.

| Method | Path | Description |
| --- | --- | --- |
| `POST` | `/jobs` | Submit. Returns 201 on insert, 200 on an idempotency replay. |
| `GET` | `/jobs/{id}` | Status, attempts, `last_error`, `result`, timestamps. |
| `GET` | `/stats` | Counts, backlog, leases, workers, throughput and alerts. |
| `GET` | `/jobs` | List and filter by `status`, `limit` (1-200), `offset`. |
| `POST` | `/jobs/{id}/retry` | Requeue a `failed` job. Returns 202 or 409. |
| `GET` | `/job-types` | Registered handler names. |
| `GET` | `/healthz` | Database health check. |
| `POST` | `/maintenance/reap` | Recover expired leases now. |

### Submission

```json
{
  "name": "partner_call",
  "payload": {"account": "acct-1", "amount": 10},
  "priority": 2,
  "run_at": "2030-01-01T00:00:00Z",
  "max_attempts": 5,
  "idempotency_key": "checkout-42"
}
```

- `name` is required. `payload` defaults to `{}`.
- `priority` defaults to `2` and must be 0-9.
- `run_at` must include a timezone. Naive values are treated as UTC.
- `max_attempts` defaults to `DEFAULT_MAX_ATTEMPTS` (5).
- `idempotency_key` can also be sent as the `Idempotency-Key` header. Different values in both places return 422.

A replay returns the original job with `Idempotent-Replayed: true`. Without a key, two identical requests create two jobs.

### Job statuses

- `pending`: waiting for a worker. Eligible when `run_at <= now()`.
- `running`: leased to a worker. A dead worker's lease expires and the job is requeued or dead-lettered.
- `succeeded`: terminal. The handler's JSON result is stored.
- `failed`: terminal dead letter. The row keeps `last_error` and can be requeued with `POST /jobs/{id}/retry`.

## Built-in handlers

| Name | Payload | Behavior |
| --- | --- | --- |
| `noop` | anything | Succeeds immediately. |
| `sleep` | `{"seconds": 0-120}` | Sleeps; useful for kill and drain tests. |
| `always_fail` | `{"message": "..."}` | Permanent failure for dead-letter demos. |
| `partner_call` | `{"account": "...", "amount": 10}` | Calls the mock partner `/charge` with `job.id` as the idempotency key. |

Add a handler with `@register("name")` in `app/handlers/`. Raise `TransientJobError` for retryable failures and `PermanentJobError` for failures that cannot succeed. Other exceptions are treated as transient.

## Configuration

Settings come from environment variables. See `.env.example`; Compose passes them to every service.

| Variable | Default | Notes |
| --- | --- | --- |
| `DATABASE_URL` | `postgresql+psycopg://queue:queue@localhost:5432/queue` | SQLAlchemy URL |
| `PARTNER_API_URL` | `http://localhost:8001` | Partner base URL |
| `PARTNER_TIMEOUT_SECONDS` | `10` | Per-request HTTP timeout |
| `DEFAULT_MAX_ATTEMPTS` | `5` | Used when a submission omits `max_attempts` |
| `RETRY_BASE_SECONDS` | `2` | Backoff base |
| `RETRY_MAX_SECONDS` | `300` | Backoff cap |
| `WORKER_CONCURRENCY` | `4` | Handler threads per worker |
| `WORKER_LEASE_SECONDS` | `30` | Lease length, extended by heartbeat |
| `WORKER_HEARTBEAT_INTERVAL_SECONDS` | `5` | Heartbeat interval |
| `WORKER_POLL_INTERVAL_SECONDS` | `0.25` | Idle claim poll |
| `WORKER_TIMEOUT_SECONDS` | `300` | Release the lease after this handler runtime |
| `WORKER_REAP_INTERVAL_SECONDS` | `5` | Reaper interval |
| `WORKER_SHUTDOWN_GRACE_SECONDS` | `25` | Drain window on SIGTERM |
| `WORKER_ALIVE_WINDOW_SECONDS` | `30` | A worker is alive if seen this recently |
| `STATS_BACKLOG_ALERT_SECONDS` | `60` | Alert when the oldest ready job is older |
| `STATS_THROUGHPUT_WINDOW_MINUTES` | `5` | Recent succeeded/failed window |
| `LOG_LEVEL` | `INFO` | JSON logs to stdout |

Mock partner settings are `PARTNER_FAILURE_RATE` (default `0.3`), `PARTNER_MIN_LATENCY_MS` (`150`), `PARTNER_MAX_LATENCY_MS` (`1200`) and `PARTNER_ALLOW_CONFIG` (`true`).

## Local development

You need Python 3.11 and Postgres. Tests use `postgresql+psycopg://queue:queue@localhost:5432/queue_test` by default; override with `TEST_DATABASE_URL`. The fixture creates the test database and runs migrations.

```bash
cd backend
python -m venv .venv
# Windows: .venv\Scripts\activate
# POSIX:   source .venv/bin/activate
pip install -e ".[dev]"

alembic upgrade head
uvicorn app.main:app --reload --port 8000
python -m app.worker
uvicorn app.mock_partner.main:app --port 8001
```

## Checks

```bash
cd backend
python -m ruff check .
python -m mypy app tests
python -m pytest
```

Default tests use real Postgres and worker subprocesses. Chaos and soak tests kill workers repeatedly and are opt-in:

```bash
# POSIX
RUN_SLOW_TESTS=1 pytest -m slow
# PowerShell
$env:RUN_SLOW_TESTS=1; pytest -m slow
```

Knobs: `CHAOS_JOBS` (default 60), `CHAOS_KILL_SECONDS` (6), `SOAK_JOBS` (1000), `SOAK_WORKERS` (4), `SOAK_KILL_SECONDS` (10).

The claim-to-test table is at the end of [DESIGN.md](DESIGN.md).

- `test_queue.py`: claiming, priority, leases, stale tokens, retries, dead letters.
- `test_idempotency.py`: duplicate submissions, including a concurrent race.
- `test_worker.py`: worker subprocesses, retries, dead letters, killed workers.
- `test_chaos.py`: random kills while every job reaches the partner.
- `test_soak.py`: thousands of jobs, several workers, random kills.
- `test_migrations.py`: Alembic schema matches the models.

## Migrations

```bash
alembic upgrade head
alembic revision --autogenerate -m "describe change"
```

`alembic/env.py` uses `DATABASE_URL` when set, otherwise `alembic.ini`. Compose runs migrations before the API, worker and partner start.

## CI

`.github/workflows/ci.yml` runs ruff, strict mypy and the default tests against Postgres 15, then builds the image and validates the compose file. Run the slow tests locally or in a release job.
