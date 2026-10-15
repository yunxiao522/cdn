# Sink Adapter Runtime Contract

This document defines output-side delivery requirements for HTTP APIs, Kafka,
database-like destinations, and shared file/object storage. These adapters are
not implemented in this repository. They consume durable output intents as
described in [`processing-lifecycle.md`](processing-lifecycle.md); they must
not treat a local send attempt as a durable delivery.

## Common sink contract

Every sink adapter must:

1. Receive a stable output-intent ID, original `event_id`, partition key,
   schema/processing version, and assignment epoch as metadata.
2. Derive or receive a deterministic idempotency key that remains identical
   across retries and replays. If one input produces multiple outputs, include
   a stable output identity/ordinal so distinct outputs do not collide.
3. Mark an output intent delivered only after an authoritative destination
   confirmation or a successful outcome reconciliation.
4. Treat timeouts, connection resets, and worker crashes after sending as
   **unknown outcomes**, not failures known to have had no effect.
5. Bound concurrent requests, batch size, local buffers, retry attempts, retry
   age, and total in-flight work. Apply backpressure to the output lane when
   limits are reached.
6. Preserve destination-specific response/receipt metadata needed for audit
   and reconciliation, but redact secrets and sensitive response bodies.
7. Validate the current assignment epoch before performing side effects when
   ownership can move between workers. Where the destination cannot fence old
   owners, idempotency alone does not protect against stale writes with
   different content.

If a destination supports neither durable idempotency nor querying/reconciling
the outcome of a write, exactly-once side effects cannot be guaranteed.
Document whether the approved policy accepts possible duplicates or possible
loss; never silently choose one.

## HTTP/API destination

- Require an API idempotency mechanism with a documented key scope, retention
  period, and behavior when the same key is used with a different payload.
- Reuse the same key and byte-equivalent logical request on every retry. Store
  the idempotency key for at least the maximum replay/recovery interval.
- Treat documented success responses as confirmation only if they mean the
  destination has durably accepted the effect. A `202 Accepted` response may
  require polling a status resource before marking delivery complete.
- On timeout after request transmission, query by idempotency key or safely
  repeat the same request. Do not generate a new key to “unstick” an unknown
  outcome.
- Retry only documented transient statuses and transport failures, with
  bounded exponential backoff, jitter, and `Retry-After` support. Handle
  permanent validation/authorization failures visibly and route them to the
  configured failure path.
- Bound request concurrency and honor destination quotas. Apply circuit
  breaking or pause egress when the service is unhealthy; avoid retry storms.
- Use TLS and secret-manager/workload identity credentials. Do not log
  authorization headers, URLs containing credentials, or unrestricted bodies.

## Kafka destination

- Publish with a deterministic record key derived from the required ordering
  key, and include output-intent/event identity in headers or the envelope.
- Require broker acknowledgements and producer idempotence according to the
  selected broker/client. Producer idempotence limits duplicate records for a
  producer session; it does not by itself deduplicate application retries
  across restarts or make input offset, state, and Kafka output atomic.
- Configure delivery timeout, retry policy, in-flight request limits, and
  buffer bounds explicitly. On backpressure, pause the output-lane consumer
  rather than dropping buffered records.
- Mark an output delivered only after the broker's durable publish
  acknowledgement. If a worker crashes after publish but before recording that
  acknowledgement, replay may publish again; consumers must deduplicate by
  stable output/event identity unless a verified transaction covers the full
  boundary.
- Claim transactional exactly-once only when the chosen system atomically
  commits input progress, state, and output and the failure tests demonstrate
  that exact boundary. A producer transaction limited to Kafka output does not
  include an external state store or arbitrary source.
- Version output topics and schemas using
  [`data-contracts-and-topics.md`](data-contracts-and-topics.md). Confirm
  partition-count changes do not violate ordering/state assumptions.

## Database or transactional API destination

- Require a unique constraint or equivalent atomic conditional write on the
  idempotency key. A “check then insert” sequence without a transaction or
  uniqueness guarantee is race-prone across workers.
- In one destination transaction, apply the business mutation and record the
  idempotency key/result. Repeating the key must return the original outcome or
  a documented equivalent, not apply the mutation again.
- Treat transaction commit acknowledgement loss as an unknown outcome and
  query by idempotency key before deciding to retry.
- Bound transaction size, lock wait, connection pool, and statement timeout.
  Keep batch transactions within the destination's documented atomicity and
  resource limits.
- Apply schema migrations through a separately reviewed compatibility and
  rollout process. Do not have event workers perform uncoordinated DDL.
- If the output is not transactional, state precisely which individual effect
  is idempotent and which multi-effect partial failures need reconciliation.

## Shared file/object-storage destination

- Write each output to a deterministic object identity derived from pipeline,
  partition, and output-intent identity. Avoid shared append-only files unless
  the storage system provides tested conditional append/locking semantics.
- Write to a temporary object/key, verify size/checksum, then publish through a
  conditional rename/manifest or equivalent atomic visibility mechanism.
  Object-store rename may be copy-and-delete rather than atomic; use the
  provider's conditional put/version/generation controls instead.
- Never overwrite an existing object blindly. On retry, compare its recorded
  intent identity and checksum; treat mismatches as conflict requiring
  operator review.
- Commit the durable completion manifest before marking the output intent
  delivered. The manifest should include output ID, event ID, schema version,
  destination key, checksum, creation time, and writer generation/epoch.
- Define retention, encryption, access control, partial-write cleanup, and
  duplicate-object reconciliation. Cleanup must not remove objects that are
  referenced by a committed manifest.
- For partitioned JSONL/columnar output, make file size/rotation and late-event
  behavior explicit. A file's existence alone is not proof that all expected
  records were written.

## Batch and ordering behavior

- A batch is a transport optimization, not a transaction, unless the
  destination explicitly guarantees atomic batch commit.
- Bound batch count, bytes, and maximum wait time independently. Flush on
  limits and orderly drain; preserve per-key order when splitting or retrying
  partial batches.
- Track per-item outcome where available. If a batch has mixed success, retry
  only unresolved items with their original idempotency keys.
- A batch-level timeout with unknown partial success must be reconciled before
  retry. Never assume all-or-none behavior without destination guarantees.
- Do not promise global order across partitions. If the destination requires
  per-key order, serialize or sequence writes for that key and make gap
  handling explicit.

## Delivery acceptance tests

For each sink and output contract, test:

- crash before send, after destination commit but before response, and after
  response but before local completion recording;
- duplicate intent delivery and same-key retry with identical payload;
- same idempotency key with changed payload is rejected or surfaced as a
  conflict;
- destination throttling/outage triggers bounded retry and backpressure;
- partial batch success, ambiguous batch timeout, and drain with in-flight
  writes;
- stale assignment owner cannot commit conflicting effects;
- recovery/replay after the idempotency retention window is handled safely.

Capture sink success ratio, ambiguous outcomes, retry/age distribution,
backpressure duration, latency percentiles, and dead-letter/conflict counts in
the service objectives and failure-drill evidence.
