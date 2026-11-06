# Strengths, weaknesses, operations and scaling

Measure contention, request latency, stale-conflict rates, retry volume and uncertain-save incidents before redesigning the store. Pure rules scale as stateless code, but database transactions, indexes, external providers and audit durability remain shared constraints. Separate rule tests from real SQL concurrency and authorization integration tests.

Strength: rules are explicit, reusable and independently testable. Weakness: passing pure checks cannot establish transaction isolation or token validity. Tradeoff: conservative conflicts and unknown-state handling can require a reload, but avoid silent overwrites or discarding meaningful data.

When introducing queues/outbox patterns, define idempotency keys, deduplication ownership, retries, partial failures and reconciliation before distributing business effects. The public example does not implement a new queue/store or claim exactly-once delivery. A request timeout must not cause an automatic duplicate payment or approval.

Keep user errors descriptive without leaking SQL/provider internals. Preserve request correlation and audit context, but keep sensitive pay/person data out of logs. Frontend validation improves feedback; the backend remains authoritative.

No cloud capacity, schedule, rate limit or paid service was changed for this project. Manual-only reference CI avoids automatic hosted-minute use.
