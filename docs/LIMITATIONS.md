# Limitations and Failure Modes

This service improves durability and recovery, but it cannot remove every failure mode. The guarantees are conditional and intentionally narrower than “exactly once.”

## 1. PostgreSQL is a critical dependency
If PostgreSQL is unavailable, new submissions cannot be durably accepted and workers cannot safely claim/acknowledge jobs. The API must return a failure rather than claim acceptance. Recovery depends on PostgreSQL becoming available again.

## 2. Database durability is an assumption
“No accepted job is lost” assumes committed transactions survive according to PostgreSQL's configured durability, the persistent volume is retained, backups are valid, and operators do not delete job rows. A destroyed volume without a recoverable backup can lose accepted work.

## 3. At-least-once execution can duplicate external effects
A worker may complete an external side effect and crash before recording success. After lease expiry, another worker can execute the job again. Exactly-once side effects require partner idempotency, a transactional outbox/inbox pattern on the receiving side, or a domain-specific deduplication mechanism. The queue alone cannot guarantee it.

## 4. Lease expiry is not proof of process death
A paused process, long GC pause, network partition or overloaded host can miss heartbeats while still running. The service may reassign the job while the old worker continues. Fencing prevents stale database acknowledgements but cannot retract an HTTP request already sent.

## 5. Long-running handlers need cooperation
Handlers must set realistic timeouts, heartbeat, and stop or avoid side effects when they detect lease loss. A handler that ignores cancellation can continue consuming resources after losing its lease.

## 6. Partner API behavior is outside the queue's control
The partner can be slow, unavailable, rate-limited, change response semantics, accept a request but drop its response, or fail to honor idempotency keys. Retry classification must be adapted to its actual contract.

## 7. Strict priority can starve low-priority work
If high-priority jobs continuously arrive, low-priority jobs may wait indefinitely. Priority aging or weighted fair scheduling is a future option, but changes the simple ordering contract and must be explicitly specified.

## 8. Polling creates database load
Workers periodically query PostgreSQL. Large worker fleets, short poll intervals, unbounded scans, or missing indexes can increase database load. Batch size, poll interval, indexes and connection pool limits require soak testing.

## 9. Stats are snapshots
`GET /stats` can observe counters at slightly different moments unless implemented in one consistent database snapshot. It is for operational visibility, not billing or correctness decisions.

## 10. Retention and storage growth
Job payloads, results and attempt history grow without retention/archival. Deleting rows too early can break client status lookups and idempotency replay behavior. Retention policy and idempotency-key expiry must be designed together.

## 11. Payload and result constraints
JSONB does not make arbitrarily large payloads safe. Large payloads/results increase row size, WAL volume, backups and query cost. Enforce size limits and store references rather than binary data.

## 12. No built-in multi-region high availability
The initial design uses one PostgreSQL primary. It does not implement multi-region consensus or automatic failover. A failover may introduce availability gaps and must preserve the database's durability guarantees.

## 13. Operator endpoints are sensitive
Dead-job payloads and manual retry operations can expose customer data or trigger repeated business actions. Any non-local deployment needs authentication, authorization, audit logs and payload redaction.

## 14. Clock and time assumptions
Scheduling and lease expiry depend on PostgreSQL time. Use database time for claim/lease predicates where possible, store timestamps in UTC, and avoid comparing worker-local clocks for ownership decisions.

## 15. The system cannot guarantee business success
A job can be durably accepted and eventually dead-lettered. “No job lost” means retained and inspectable, not that every external operation will succeed.
